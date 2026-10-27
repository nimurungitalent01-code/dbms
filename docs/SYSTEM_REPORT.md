# PureCure Pharmacy System Report

**Prepared:** 24 October 2026
**Repository reviewed:** PureCure Pharmacy Django application

## 1. Executive summary

PureCure is a staff-facing web application for maintaining pharmacy records and processing medicine receipts, purchases, sales, and payments. It is built with Django and maps to an existing MySQL-compatible pharmacy database. Its central operational workflow is to record a received purchase into batch stock, prepare a pending sale receipt, and deduct stock when a cashier or manager records payment.

The application implements role checks, password hashing for newly created employee accounts, CSRF protection on forms, and database transactions with row locks for key inventory/payment operations. The database schema is externally managed: all application models are unmanaged, so Django migrations do not create or update the pharmacy tables.

This report describes the code in the repository. It does not certify the live database schema, production configuration, deployment, or operational controls. Several security and operational settings need attention before deployment, particularly secrets and database credentials stored directly in settings, `DEBUG=True`, and the absence of a repository test suite.

## 2. Purpose and scope

The application provides an internal workspace for pharmacy employees. It covers catalogue and reference data, stock receipt, sale preparation, payment, receipts, and browsing existing records. It is not a public customer storefront, payment gateway, accounting package, or clinical decision-support system.

The codebase does not implement prescription creation or dispensing workflows, despite mapping prescription tables and exposing prescription records for reading. The sale form accepts one medicine batch per sale and has no prescription linkage. No supplier payment, returns, refunds, stock adjustment, automated reorder, or expiry-removal workflow is present in the reviewed views.

## 3. Technology and architecture

| Area | Implementation |
| --- | --- |
| Web framework | Django 6.0.4 |
| Database adapter | Django MySQL backend with PyMySQL 1.2.3 |
| Configuration helper | python-dotenv 1.2.3 loads `.env` |
| Application package | `purecure/` — models, views, routes, templates, CSS |
| Project package | `pharmacy/` — settings, root routes, WSGI and ASGI entry points |
| Session storage | Django signed-cookie session backend |
| UI | Server-rendered Django templates and a local stylesheet; no separate frontend framework |
| Database schema | Existing external schema; every mapped model declares `managed = False` |

The browser submits requests to Django views. Views authenticate staff, enforce role checks, validate forms, read or write ORM models, and render templates. The ORM maps to the existing database tables; Django does not own their lifecycle. Root routes include the application at `/` and Django admin at `/admin/`.

## 4. Functional capabilities

### 4.1 Sign-in and staff workspace

Employees sign in with a username and password stored in the `employees` table. On success, the session stores the employee ID. Each protected request reloads the employee and role and checks that the employee is still active; inactive or missing accounts are logged out. Login accepts Django-supported encoded password formats and also has a constant-time plaintext comparison fallback for legacy values. Newly created employees receive Django-hashed passwords.

The dashboard displays the count of active medicines, all customers, all sales, total remaining batch units, and up to six recent sales. The total stock figure includes all batches and does not exclude expired batches or inactive medicines.

### 4.2 Records and catalogue maintenance

Staff can browse role-authorized record sections and search the displayed fields. The records view is capped at 100 rows and orders by primary key ascending, including when the interface describes the list as newest first. Search is server-side and limited to the configured display fields.

Managers and pharmacists can add or edit medicines, categories, and suppliers. All three roles can add or edit customer records. Only managers can add employees and edit employee records. Employee creation requires a temporary password of at least eight characters and stores it through Django's password hasher. Transaction and audit tables are browsable but do not have generic create/edit controls.

### 4.3 Receiving purchases

Managers and pharmacists can record one medicine batch per purchase. The form requires an active supplier and medicine, a positive quantity, a positive unit cost, a unique medicine/batch number, and a non-expired expiry date. An invoice number is optional.

Within one database transaction, the application creates a purchase header, purchase item, medicine batch with received and remaining quantities initialized equally, and a stock movement. A database error rolls the writes back together. The form performs duplicate checks for convenience; database unique indexes must provide the final concurrency-safe protection.

### 4.4 Preparing sales and issuing receipts

Managers and pharmacists can prepare a sale for a customer or walk-in. The selected batch must have positive remaining quantity, be unexpired, and belong to an active medicine. The application locks the batch while it rechecks available quantity, then creates a pending sale and a sale item priced from the medicine's current unit price.

The sale form accepts a single batch and quantity. Preparing a sale does not reserve or deduct stock. A pending receipt can become unfulfillable if other completed sales consume those units before payment. Authenticated staff can view and print a sale receipt; it shows the sale lines, total, status, and payment details when present.

### 4.5 Taking payment and releasing stock

Managers and cashiers can record payment by cash, card, mobile money, or insurance. The received amount must be at least the sale total; excess appears as change on the receipt. Before processing, the view checks that the database backend supports `SELECT ... FOR UPDATE`.

Inside one transaction, the application locks the sale, verifies it is still pending, rechecks the amount against the locked sale total, loads the sale items, and locks their batches in ascending primary-key order. It verifies that each batch still exists, is not expired, and has enough units. It then deducts stock, records stock-movement rows, creates a paid payment with a generated reference, and marks the sale complete. These changes commit together. A second payment attempt for the same sale waits on the sale lock and then sees its completed status.

## 5. Roles and access

Role names are compared in application code and are expected to match `Manager`, `Pharmacist`, and `Cashier` exactly. Access is enforced in views as well as navigation construction.

| Capability | Manager | Pharmacist | Cashier |
| --- | --- | --- | --- |
| Dashboard and customers | Yes | Yes | Yes |
| Medicines | Read, add, edit | Read, add, edit | Read |
| Categories, batches, suppliers, purchases, stock movements | Read | Read | No |
| Prepare sales | Yes | Yes | No |
| View sales, sale items, payments, receipts | Yes | Yes | Yes |
| Complete payment | Yes | No | Yes |
| Prescriptions and prescription items | Read | Read | No |
| Employees and roles | Read, add/edit employees | No | No |

Role rules are defined by `TABLES`, `CREATE_ROLES`, and `EDIT_ROLES` in `purecure/views.py`. In the present code, role records can be read by managers, but no generic role create/edit form is exposed.

## 6. Data model

The ORM maps the following existing tables:

| Domain | Tables and principal relationships |
| --- | --- |
| Staff and authorization | `roles` → `employees`; employees are referenced by purchases, sales, prescriptions, and stock movements |
| Customers | `customers`; optionally referenced by sales and referenced by prescriptions |
| Product catalogue | `medicine_categories` → `medicines`; medicines are referenced by batches, transaction lines, prescriptions, and movements |
| Suppliers and receiving | `suppliers` → `purchases` → `purchase_items`; a purchase creates `medicine_batches` |
| Inventory history | `medicine_batches` hold received/remaining quantity, expiry, and cost; `stock_movements` record purchase and sale changes with employee and source references |
| Sales | `sales` → `sale_items`; each line references a medicine and batch; sales optionally reference a customer and prescription |
| Payments | `payments` references a sale and stores amount, method, status, timestamp, and unique reference |
| Prescriptions | `prescriptions` reference customer and employee; `prescription_items` reference prescription and medicine and store dispensing instructions/quantities |

Foreign keys use `DO_NOTHING` in the Django mapping. Referential integrity and delete behavior therefore depend on the actual database schema. Several unique declarations appear in model metadata, including employee username/email, role and category names, purchase invoice, payment reference, prescription reference, supplier registration number, and medicine/batch number. Because models are unmanaged, the actual constraints must be checked in the target database; model metadata alone does not install them.

## 7. Security and privacy controls observed

- Protected application views use a decorator that requires an active employee session.
- Role checks are performed server-side for record sections, editing, purchase creation, sale preparation, and payment.
- Django CSRF middleware is enabled, and the provided POST forms include CSRF tokens.
- New employee passwords are hashed with Django's password hasher. Login also supports legacy plaintext values through a constant-time comparison path.
- Django clickjacking middleware is enabled.
- Sessions use Django's signed-cookie backend, so the employee ID is stored in a signed client-side session cookie rather than the database session table.
- Record lists render selected fields and are limited to 100 rows per request.

## 8. Operational and security gaps

The following items are visible in the source and should be addressed or verified before production use:

1. **Secrets and database credentials:** `pharmacy/settings.py` contains a hard-coded Django secret key and database credentials. `.env.example` documents database environment variables, but the active database configuration uses literal values instead. Treat any committed credential as exposed: rotate it, move secrets into the deployment environment/secret manager, and ensure real `.env` files remain excluded from version control.
2. **Debug mode:** `DEBUG` is enabled in settings. Disable it in production and configure a production secret key, allowed hosts, HTTPS, secure cookies, and other Django deployment settings.
3. **Password migration:** the plaintext compatibility path allows legacy plaintext credentials to remain valid. Migrate those account values to Django hashes and remove the plaintext comparison once migration is complete. The custom login flow does not use Django's built-in authentication backend or apply the configured password validators at login.
4. **Login abuse controls:** no rate limiting, lockout, or login-attempt audit is present in the reviewed application code.
5. **Cookie/session policy:** signed-cookie sessions provide integrity but are not server-side revocable records. Production should use HTTPS and secure cookie settings, and review session expiry and key rotation policies.
6. **Database guarantees:** verify that all transaction tables use a transactional engine with row-level locking and that required unique indexes exist. External SQL writers must follow compatible transaction and lock rules.
7. **Schema and migrations:** models are unmanaged, so deployment must provision and evolve the schema separately. Backups and reviewed schema migrations are operational responsibilities.
8. **Testing and recovery:** `purecure/tests.py` contains only the default placeholder. No automated tests, backup procedure, point-in-time recovery configuration, monitoring, or deadlock retry policy is supplied in this repository.
9. **Data minimization:** the mapped customer and prescription records contain personal and health-related information. The repository does not document retention, privacy notice/consent, access audit, or data export/deletion processes. Confirm applicable local privacy and pharmacy requirements before deployment.
10. **Admin and permissions:** `/admin/` is routed, but `purecure/admin.py` does not register the pharmacy models. The normal application role system is separate from Django admin permissions; restrict the admin route and review whether it is needed.

## 9. Transactions, consistency, and recovery

Purchase receiving and payment use `transaction.atomic()`. Sale preparation also uses a transaction and locks the selected batch while creating a pending sale. Payment locks the sale and all referenced batches in stable primary-key order, then updates stock and writes payment and audit records before commit. This is a pessimistic concurrency approach; locks are held through the transaction. These guarantees depend on the configured database engine and constraints being present in the live schema.

The application does not automatically retry deadlocks. An aborted database transaction should be reported/retried by staff as appropriate. Pending sales do not reserve inventory. Stock movements provide an application audit trail for purchase receipts and completed sales, but do not replace database backups or transaction logs. The repository does not supply disaster recovery configuration.

For a fuller locking analysis and sample database inspection queries, see [Transactions and Concurrency](TRANSACTIONS_AND_CONCURRENCY.md).

## 10. Configuration and deployment outline

The repository README describes installing requirements, configuring the MySQL-compatible database, confirming transactional table engines, and running `python manage.py runserver` for local use. The current checked-in settings do not consume the database values in `.env.example`; deployment configuration should be corrected before relying on that sample.

Before deployment, the responsible operator should:

1. Provision the existing schema and verify columns, foreign keys, storage engines, and required unique indexes.
2. Rotate credentials currently present in source and configure secrets through the deployment environment.
3. Set production `DEBUG=False`, an appropriate `ALLOWED_HOSTS`, HTTPS, secure cookie settings, and a production secret key.
4. Apply and verify a plan for backups, restore tests, monitoring, and operational incident handling.
5. Migrate any legacy plaintext employee passwords and create manager accounts using a controlled process.
6. Exercise purchase, sale, payment, permission, and concurrent-payment workflows against a non-production database.

## 11. Known limitations

- No automated test coverage is currently provided in the repository.
- The sales UI records one medicine/batch per sale; multi-line sale entry is not implemented.
- Pending sales do not reserve stock.
- No refund, reversal, return, stock adjustment, or manual disposal/expiry workflow is implemented.
- Prescription tables are mapped and visible to authorized staff, but the application does not provide prescription management or connect prescription dispensing to sale entry.
- The dashboard's stock total is not filtered by expiry or active status.
- Record browsing is capped at 100 rows and sorted by primary key ascending; there is no pagination.
- Database schema changes, backups, operational monitoring, and disaster recovery are outside this repository.

## 12. Repository reference

- `README.md` — brief project overview and local setup notes.
- `docs/TRANSACTIONS_AND_CONCURRENCY.md` — transaction and locking design details.
- `purecure/views.py` — login, role checks, record management, purchase, sale, receipt, and payment workflows.
- `purecure/models.py` — mappings to the external pharmacy schema.
- `pharmacy/settings.py` — Django settings, database configuration, middleware, and session backend.
- `purecure/urls.py` and `pharmacy/urls.py` — application and project URL routes.
- `purecure/templates/purecure/` — staff-facing pages and printable receipts.
- `requirements.txt` — pinned Python dependencies.

---

**Review boundary:** This is a source-code documentation report, not a live-system audit. Confirm findings against the deployed settings, database schema and indexes, infrastructure, and operating procedures before using it as a compliance or production-readiness statement.
