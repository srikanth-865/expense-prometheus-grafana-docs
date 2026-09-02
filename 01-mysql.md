# MySQL

The database tier. Stores categories and expenses for the Expense Tracker.

> **This setup uses MySQL 8.0.x** (the schema's `CHECK` constraint requires 8.0.16+).

---

## Install

```bash
dnf install mysql-server -y
```

## Configure

Enable and start the MySQL service:

```bash
systemctl enable mysqld
systemctl start mysqld
```

Set the root password (use `ExpenseApp@1` or a password of your choice):

```bash
mysql_secure_installation --set-root-pass ExpenseApp@1
```

---

## Load Schema

Download the mysql package and extract it on the DB server itself:

```bash
curl -o /tmp/mysql.tar.gz \
  https://raw.githubusercontent.com/90s-org/expense-obs-documentation/main/artifacts/expense-mysql-v1.tar.gz
mkdir -p /tmp/expense-mysql && cd /tmp/expense-mysql
tar -xzf /tmp/mysql.tar.gz
```

Load the schema. You're running as root on the DB box itself, so no `-u
root -p` is needed — the local root Linux user reaches MySQL over the local
socket without a password:

```bash
mysql < schema.sql
```

This creates the `expense_db` database plus the `categories` and `expenses`
tables.

Create the application user the backend connects as:

```bash
mysql < create_user.sql
```

> **`create_user.sql` ships with a placeholder password (`ChangeMe123!`).
> Change it before using this anywhere but a lab/demo environment**, and keep
> it in sync with `DB_PASSWORD` in the backend's `.env`.

Optionally load sample data:

```bash
mysql < seed.sql
```

If this box also runs `mysqld_exporter` (Prometheus MySQL metrics), create
its monitoring user too:

```bash
mysql < monitor.mysql
```

This creates `exporter`@`localhost` with just `PROCESS`, `REPLICATION
CLIENT`, and `SELECT` — enough for the exporter, nothing more.

---

## Schema

```sql
CREATE TABLE categories (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  name       VARCHAR(100) NOT NULL UNIQUE,
  icon       VARCHAR(50)  NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE expenses (
  id            INT AUTO_INCREMENT PRIMARY KEY,
  category_id   INT NOT NULL REFERENCES categories(id) ON DELETE RESTRICT,
  amount        DECIMAL(10,2) NOT NULL CHECK (amount > 0),
  expense_date  DATE NOT NULL,
  notes         VARCHAR(500) NULL,
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

`ON DELETE RESTRICT` means a category with existing expenses can't be
deleted — the backend surfaces this as a `409 Conflict` rather than letting
the FK error bubble up raw.

---

## Verification

Connect locally:

```bash
mysql -u root -pExpenseApp@1
```

Connect remotely from the backend server (replace with this box's private IP):

```bash
mysql -h <MYSQL-SERVER-PRIVATE-IP> -u expense_app -p expense_db
```

Check tables:

```sql
USE expense_db;
SHOW TABLES;
DESCRIBE categories;
DESCRIBE expenses;
SELECT * FROM categories;
```

---

## Security Group

Ensure the DB EC2 security group allows **inbound TCP on port 3306** from
the backend EC2's security group only — not from the internet.
