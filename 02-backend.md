# Backend

The backend service handles the REST API and persists data to MySQL. It is
written in Node.js/Express, with Prometheus metrics and structured JSON logs.

> **This setup requires Node.js >= 18** (uses `crypto.randomUUID`, ships with
> `express`, `mysql2`, `pino`, `prom-client`).

---

## Install Node.js

```bash
dnf module disable nodejs -y
dnf module enable nodejs:20 -y
dnf install nodejs -y
```

Verify:

```bash
node -v
```

---

## Create Application User and Directory

```bash
useradd --system --home /app --shell /sbin/nologin --comment "expense system user" expense
mkdir -p /app
chown expense:expense /app
```

**Why a system user?** UID < 1000, no login shell, no password — a
compromised backend process can't be used to log in, and `ps aux` clearly
attributes the process to `expense` for auditing.

---

## Download and Extract Application

```bash
curl -o /tmp/backend.tar.gz \
  https://raw.githubusercontent.com/daws-90s/expense-prometheus-grafana-docs/main/artifacts/expense-backend-v1.tar.gz
cd /app
tar -xzf /tmp/backend.tar.gz
chown -R expense:expense /app
```

---

## Install Dependencies

```bash
cd /app
npm install
```

---

## Configure Environment

Create `/app/.env`:

```bash
PORT=8080
SERVICE_NAME=expense-backend
DB_HOST=<MYSQL-SERVER-PRIVATE-IP>
DB_PORT=3306
DB_USER=expense_app
DB_PASSWORD=<same password as create_user.sql on the mysql tier>
DB_NAME=expense_db
LOG_LEVEL=info
ENABLE_DEBUG_ROUTES=false
```

> **Replace `<MYSQL-SERVER-PRIVATE-IP>`** with the mysql EC2's private IP
> (or `mysql.daws86s.fun` if Route 53 is set up — see `expense-infra`).

---

## Configure SystemD Service

```bash
vim /etc/systemd/system/backend.service
```

```ini
[Unit]
Description=Expense Tracker backend (Node/Express)
After=network.target

[Service]
Type=simple
User=expense
Group=expense
WorkingDirectory=/app
EnvironmentFile=/app/.env
ExecStart=/usr/bin/node src/server.js
Restart=on-failure
RestartSec=5

NoNewPrivileges=true
ProtectSystem=full
PrivateTmp=true

[Install]
WantedBy=multi-user.target
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

```bash
systemctl status backend
journalctl -u backend -f
curl http://localhost:8080/health
```

Expected:

```json
{ "status": "ok" }
```

`/health` returns `503 {"status":"degraded"}` if the MySQL connection pool
can't reach the database — check that first if the frontend shows "Couldn't
reach the server".

Full endpoint reference: [backend-api.md](backend-api.md).

---

## Security Group

Ensure the backend EC2 security group allows **inbound TCP on port 8080**
from the frontend EC2's security group only — not from the internet. The
frontend's nginx reverse-proxies `/api/` to this port privately, so the
browser never talks to the backend directly.
