# MySQL

The developer has chosen MySQL as the database. We need to install, configure, and load the schema.

> **Check with the developer for the exact version required. This setup uses MySQL 8.0.x.**

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

## Verification

The schema is loaded from the backend server (see [02-backend.md](02-backend.md)). After that, connect to MySQL from the same server:

```bash
mysql -u root -pExpenseApp@1
```

Connect remotely from another server:

```bash
mysql -h <MYSQL-SERVER-IPADDRESS> -u root -pExpenseApp@1
```

Check databases and tables:

```sql
SHOW DATABASES;
USE transactions;
SHOW TABLES;
DESCRIBE transactions;
SELECT * FROM transactions;
```

---

## Security Group

Ensure the DB EC2 security group allows **inbound TCP on port 3306** from the backend EC2's security group (not from the internet).
