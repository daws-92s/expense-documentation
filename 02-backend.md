# Backend

The backend service is responsible for handling API requests and persisting data to the database. It is written in Node.js.

> **Check with the developer for the exact version required. This setup requires Node.js >= 24.**

---

## Install Node.js

By default, Node.js 16 is available on the system. Enable and install version 24:

```bash
dnf module disable nodejs -y
dnf module enable nodejs:24 -y
dnf install nodejs -y
```

Verify:

```bash
node -v
```

---

## Set Up Application Directory

```bash
mkdir /app
```

---

## Create Application User

Add a system user to run the application:

```bash
useradd --system --home /app --shell /sbin/nologin --comment "expense system user" expense
```

**Why a system user?**

System users (UID 1–999) are created exclusively to run services — not for human login. Compared to normal users they provide:

- **No login access** — `/sbin/nologin` shell blocks any interactive login, reducing attack surface.
- **No password** — cannot be brute-forced via SSH.
- **Least privilege** — only owns the files it needs; a compromised service can't touch the rest of the system.
- **Process accountability** — `ps aux` clearly shows `expense` owns the backend process, making auditing easy.

Normal users start at UID 1000. You can verify this user was created as a system user:

```bash
id expense
grep expense /etc/passwd
```
---

## Download and Extract Application

```bash
curl -o /tmp/backend.tar.gz https://raw.githubusercontent.com/daws-92s/expense-documentation/refs/heads/main/artifacts/expense-backend-v5.tar.gz
```

Extract files directly into `/app` (no subfolder):

```bash
cd /app
tar -xzf /tmp/backend.tar.gz
```

---

## Install Dependencies

```bash
cd /app
npm install
```

---

## Configure SystemD Service

Create the service file:

```bash
vim /etc/systemd/system/backend.service
```

```bash
[Unit]
Description=Expense Backend Service
After=network.target

[Service]
User=expense
Environment=DB_HOST=<MYSQL-SERVER-IPADDRESS>
Environment=DB_USER=expense
Environment=DB_PWD=ExpenseApp@1
Environment=DB_DATABASE=transactions
Environment=ADMIN_TOKEN=admin123
ExecStart=/bin/node /app/index.js
SyslogIdentifier=backend
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **Replace `<MYSQL-SERVER-IPADDRESS>` with the private IP of the MySQL EC2 instance.**

`ADMIN_TOKEN` protects **Delete All**. The UI asks for it; a missing token gives `401`, a wrong one `403`. Optional: `RATE_LIMIT_MAX` (default `10`) and `RATE_LIMIT_WINDOW_SEC` (default `60`) control when write requests start getting `429`.

---

## Load Database Schema

```bash
dnf install mysql -y
mysql -h <MYSQL-SERVER-IPADDRESS> -u root -pExpenseApp@1 < /app/schema/backend.sql
```

---

## Start the Service

```bash
systemctl daemon-reload
systemctl enable backend
systemctl start backend
```

---

## Verification

Check service status:

```bash
systemctl status backend
```

Check application logs:

```bash
journalctl -u backend -f
```

Test the health endpoint (from the same server):

```bash
curl http://localhost:8080/health
```

Expected: `{"status":"ok","db":"up"}`. If MySQL is unreachable you get `503` with `"db":"down"`.

Every request is logged with its status code, e.g. `PUT /transaction/3 200 12ms`. Watch them live with `journalctl -u backend -f` while you click around the UI.

---

## Security Group

Ensure the backend EC2 security group allows **inbound TCP on port 8080** from the frontend EC2's security group only (not from the internet).
