# Transactions, concurrency, and locking

## Textbook basis

This implementation follows the transaction-processing concepts in Ramez Elmasri and Shamkant B. Navathe, *Fundamentals of Database Systems*, 7th edition. The attached copy's outline identifies Chapter 20, “Introduction to Transaction Processing Concepts and Theory” (book pages 776–810); Chapter 21, “Concurrency Control Techniques” (pages 812–841); and Chapter 22, “Database Recovery Techniques” (pages 844–869).

The application applies the book's core ideas:

- Treat a business operation that changes related rows as one transaction.
- Commit the operation only when every required write succeeds; otherwise roll it back.
- Use exclusive row locks to serialize conflicting inventory and payment operations.
- Keep locks until commit or rollback (strict two-phase locking behavior).
- Use a consistent order when locking multiple inventory rows to reduce deadlocks.
- Back the application checks with database uniqueness constraints where the existing schema provides them.

## What the application does

### Receiving a purchase

`record_purchase` in `purecure/views.py` wraps the purchase header, purchase item, medicine batch, and stock movement inserts in `transaction.atomic()`.

All four records commit together. If an insert fails, the transaction rolls back so there is no purchase header without its matching batch or audit movement. The form checks invoice and batch-number duplicates for helpful feedback. These checks can race with another request, so the database's unique indexes on purchase invoice numbers and on `(medicine_id, batch_number)` are the final duplicate guards; an `IntegrityError` is caught and returned to the form.

### Preparing a sale

`record_sale` opens an atomic transaction and uses `select_for_update()` on the selected medicine-batch row before checking available quantity. It then creates the pending sale and sale item in the same transaction. The lock is limited to the batch row, avoiding unnecessary locks on joined medicine rows.

Preparing a sale does **not** deduct or reserve stock. The availability check gives staff an up-to-date view when the receipt is prepared, but another pending receipt may use the same units. Payment is the point at which inventory is committed. If stock is no longer available by then, the cashier is told the sale cannot be completed.

### Taking payment and releasing stock

`take_payment` performs these steps inside a single `transaction.atomic()` block:

1. Lock the sale row with `select_for_update()` and confirm it is still pending. This serializes two cashiers attempting to complete the same sale. The second request waits for the first transaction, then sees the committed sale status and cannot charge again.
2. Load the sale items and lock all referenced medicine batches in ascending primary-key order.
3. Recheck that each batch exists, is not expired, and has enough units.
4. Decrement each batch's remaining quantity and create its `StockMovements` audit row.
5. Create the payment and its reference number.
6. Mark the sale completed.

The sale lock, batch locks, stock updates, audit rows, payment, and status change are held until the database commits. If a validation branch rejects the sale before writes, or an exception aborts the transaction, all writes in that transaction are rolled back. The form checks the amount before entering the transaction; the amount is checked again against the locked sale total so a concurrent total change cannot allow an underpayment.

## Concurrency model

The payment workflow uses pessimistic locking: it locks the sale and stock rows before changing them. The database grants conflicting writers one at a time. For a given sale, the sale row is the payment serialization point; for inventory, the batch rows are the stock serialization points.

The application locks rows only when it needs to make a decision that must remain true through the write. A normal list or receipt view does not lock rows. Purchase inserts rely on database unique indexes to resolve concurrent duplicate requests. The payment path also fails closed when the configured Django database backend does not advertise `SELECT ... FOR UPDATE` support.

Locking multiple batches in a stable order helps prevent the classic deadlock where two transactions each hold one batch and wait for the other. Database deadlocks can still occur because of other queries or external writers; the database will abort one transaction, and the staff member may need to retry. The application does not currently retry deadlocked transactions automatically.

## Database requirements and limits

These guarantees depend on the existing database schema and engine:

- Use MySQL/InnoDB (or a compatible engine that supports transactions and row-level `SELECT ... FOR UPDATE`). A backend that ignores row locks cannot safely complete payments; the application rejects payment POSTs when Django reports no row-lock support.
- The `purchases.invoice_number` uniqueness and `medicine_batches(medicine_id, batch_number)` uniqueness must exist as real database indexes. Django models use `managed = False`, so Django migrations do not create these constraints.
- All application instances must use the same transactional database. Process-local locks would not coordinate separate web workers and are not used here.
- Manual SQL, admin scripts, or other applications that alter sales or stock can bypass these workflow locks and checks. Those writers must use transactions and compatible locks too.
- Pending sale receipts are not stock reservations. If the business later requires reservation/expiry behavior, it needs an explicit reservation state and a release policy rather than treating a pending receipt as stock already deducted.
- The existing schema does not declare a one-payment-per-sale unique constraint in the Django model. Application requests are serialized by locking the sale row; a database-level unique index on `payments.sale_id` would add protection against writers that bypass this application, provided the business rules continue to allow only one payment per sale.

Because the tables are unmanaged and the live database is not changed by this code, confirm the storage engine and indexes on the target database before deployment. Example MySQL inspection queries:

```sql
SHOW TABLE STATUS
WHERE Name IN ('sales', 'sale_items', 'payments', 'medicine_batches', 'stock_movements', 'purchases', 'purchase_items');

SHOW INDEX FROM purchases;
SHOW INDEX FROM medicine_batches;
```

The engine column should show `InnoDB` for every table participating in a transaction. Confirm that the indexes include the unique purchase invoice key and the composite medicine/batch-number key. Back up the database and review existing duplicates before adding or changing any production constraint.

## Recovery and audit trail

Atomic commit and rollback ensure the application does not intentionally leave a half-written purchase or payment workflow after a database error. Durability after commit is provided by the database engine and its configured storage/recovery policy. This repository does not configure database backups, point-in-time recovery, or disaster recovery; those remain operational responsibilities.

Every received purchase and completed sale records a `StockMovements` row with the related batch, employee, quantity, movement type, and source reference. These rows make stock changes traceable; they are not a substitute for backups or transaction logs.

## Relevant implementation locations

- `purecure/views.py`: purchase, sale preparation, payment, and lock checks.
- `purecure/models.py`: mappings to the existing schema and uniqueness declarations.
- `pharmacy/settings.py`: active MySQL connection configuration.
- `README.md`: project setup, features, roles, and operations overview.
