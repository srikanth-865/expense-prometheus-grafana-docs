# expense-prometheus-grafana-docs

Documentation and build artifacts for the **v1** Expense Tracker stack, plus
its Prometheus + Grafana + blackbox_exporter monitoring tier. Companion to
the `expense-mysql-v1`, `expense-backend-v1`, and `expense-frontend-v1`
repos — each of those carries its own detailed README; this repo is the
index and the place other EC2 instances (via `expense-prometheus-grafana`'s
`user_data`) pull deployable artifacts from.

---

## Architecture

```
                 nginx (frontend EC2, port 80)
  Browser  --->  static React build + reverse proxy
                       |
                       |  /api/*  (proxy_pass, private)
                       v
              Node/Express backend (private IP, port 8080)
                       |
                       v
                  MySQL (private IP, port 3306)
```

- The browser only ever talks to the frontend EC2 on port 80.
- nginx serves the built React app and reverse-proxies `/api/` to the
  backend's **private** IP/DNS — the backend's security group never needs to
  accept traffic from the internet, only from the frontend's SG (+ SSH).
- The backend talks to MySQL over its private IP; MySQL's SG only accepts the
  backend's SG (+ SSH).
- DNS is managed in Route 53 (`daws86s.fun` zone, see
  `expense-prometheus-grafana/r53.tf`): `daws86s.fun` -> frontend public IP,
  `backend.daws86s.fun` -> backend private IP, `mysql.daws86s.fun` -> mysql
  private IP, `prometheus.daws86s.fun` -> prometheus public IP.
- A fourth EC2 instance runs Prometheus (EC2 service discovery),
  node_exporter (all four boxes), mysqld_exporter (mysql), blackbox_exporter
  (HTTP/HTTPS probes of the frontend + backend `/health`), and Grafana with
  dashboards + datasource provisioned automatically from this repo's
  artifacts.

## Artifacts

| File | Source repo | Contents |
|---|---|---|
| `artifacts/expense-mysql-v1.tar.gz` | `expense-mysql-v1` | schema, seed data, app-user creation SQL, `mysqld_exporter` monitoring-user SQL |
| `artifacts/expense-backend-v1.tar.gz` | `expense-backend-v1` | Express app source + `backend.service` systemd unit (no `node_modules`, no `.env`) |
| `artifacts/expense-frontend-v1.tar.gz` | `expense-frontend-v1` | React app source + `nginx.conf` + `nginx-tls.conf` + `cert-demo.sh` (no `node_modules`, no build output, no `.env`) |
| `artifacts/expense-prometheus-v1.tar.gz` | `expense-prometheus-grafana`'s `grafana/`, `prometheus/` | Grafana dashboard JSON files + `recording_rules.yaml`/`alerting_rules.yaml`, installed into `/etc/grafana/dashboards` and `/opt/prometheus` |

None of the archives include `.git`, `node_modules`, build output, or `.env`
files — pull dependencies fresh with `npm install` and supply your own
environment config on the target box.

To fetch an artifact directly onto an EC2 instance once this repo is pushed
to GitHub:

```bash
curl -o /tmp/backend.tar.gz \
  https://raw.githubusercontent.com/daws-90s/expense-prometheus-grafana-docs/main/artifacts/expense-backend-v1.tar.gz
mkdir -p /opt/expense-backend && cd /opt/expense-backend
tar -xzf /tmp/backend.tar.gz
npm install
```

## Deploy order

Full step-by-step install docs, one per tier:

1. [01-mysql.md](01-mysql.md) — install MySQL, load `schema.sql` +
   `create_user.sql` + `seed.sql`.
2. [02-backend.md](02-backend.md) — install Node.js, extract the backend
   artifact, configure `.env`, install and start `backend.service`.
3. [03-frontend.md](03-frontend.md) — install Node.js + nginx, build the
   React app (`VITE_API_URL` must stay unset), deploy `dist/` + `nginx.conf`,
   plus the on-demand 1-hour self-signed HTTPS cert demo (`cert-demo.sh`).
4. [04-monitoring.md](04-monitoring.md) — install Prometheus (EC2 service
   discovery), node_exporter, mysqld_exporter, blackbox_exporter (HTTP/HTTPS
   synthetic probes), and Grafana (dashboards + datasource auto-provisioned).

API reference: [backend-api.md](backend-api.md).

Each source repo (`expense-mysql-v1`, `expense-backend-v1`,
`expense-frontend-v1`) also carries its own `README.md` with the same
commands plus local-dev instructions.
