# PureCure Pharmacy

PureCure is a Django web application for pharmacy staff. It provides a role-based workspace for viewing pharmacy records, maintaining catalogue data, receiving purchases, preparing sales, taking payment, and printing receipts.

## Technology and structure

- **Web framework:** Django 6.0.
- **Database:** MySQL-compatible database, accessed through PyMySQL.
- **Application:** `purecure/` contains the database models, staff views, URL routes, templates, and static CSS.
- **Project configuration:** `pharmacy/` contains Django settings and the root URL configuration.
- **Data model:** `purecure/models.py` maps the existing pharmacy tables, including customers, employees, medicines, suppliers, purchases, batches, sales, payments, prescriptions, and stock movements.
- **Database ownership:** the models use `managed = False`. Django reads and writes the existing schema; it does not create or migrate these tables. Back up the database before changing its schema.

## Main workflows

1. Staff sign in with an employee account. The app stores the employee ID in Django's signed-cookie session and checks that the employee is active on each protected request.
2. The dashboard shows medicine, customer, sales, and stock totals, plus recent sales.
3. Staff can search and view records allowed for their role. Managers can maintain employee accounts; managers and pharmacists can maintain medicine and supplier records; customer records can be maintained by all three roles.
4. Managers and pharmacists can record received purchases. A purchase, its line item, the new batch, and its stock movement are saved as one database transaction.
5. Managers and pharmacists can prepare a sale and issue a pending receipt. A cashier or manager completes payment. Payment, stock deductions, stock movements, and the sale status change are committed together.
6. Staff can view and print sale receipts.

## Run locally

1. Create and activate a Python virtual environment.
2. Install dependencies with `pip install -r requirements.txt`.
3. Configure the database connection in `pharmacy/settings.py` for the MySQL-compatible database that contains the pharmacy schema.
4. Confirm that the transaction tables use InnoDB (or another engine with real row-level locks and transactional rollback). SQLite and MySQL's non-transactional engines do not provide the guarantees used by payment processing.
5. Start the development server with `python manage.py runserver` and open `/login/`.

The current settings load `.env` for `ALLOWED_HOSTS`, but the database connection, `SECRET_KEY`, and `DEBUG` setting are declared directly in `pharmacy/settings.py`. Configure those values safely before deployment; do not commit production credentials or run with `DEBUG=True` in production. The `.env.example` file is a sample and does not currently configure the database connection by itself.

## Roles

| Role | Main access |
| --- | --- |
| Manager | All record sections, employee administration, purchase and sale recording, and payment. |
| Pharmacist | Medicine, supplier, purchase, stock, prescription, customer, and sale workflows; can prepare sales. |
| Cashier | Customer and sales access; can complete payments. |

The exact table-level read, create, and edit permissions are defined by `TABLES`, `CREATE_ROLES`, and `EDIT_ROLES` in `purecure/views.py`.

## Transactions and concurrency

The purchase and sale workflows use database transactions and row locks so concurrent staff actions do not silently produce duplicate payments or oversell stock. The design and database requirements are described in [docs/TRANSACTIONS_AND_CONCURRENCY.md](docs/TRANSACTIONS_AND_CONCURRENCY.md).

For a detailed system overview, data model, role matrix, workflows, security observations, limitations, and deployment considerations, see the [PureCure Pharmacy System Report](docs/SYSTEM_REPORT.md).

## Important operational notes

- The database schema already exists and is externally managed. Model changes alone do not alter the schema.
- Transactions protect operations made through these application workflows. Direct SQL or other applications that update the same tables must follow the same locking and audit rules.
- Pending sale receipts do not reserve stock. Stock is checked and deducted when the cashier completes payment. A receipt can therefore be declined at payment time if another completed sale consumed the remaining stock.
- The current app has no automated test suite in `purecure/tests.py`.
