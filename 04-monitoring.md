# Monitoring (Prometheus, Grafana, blackbox_exporter)

The fourth EC2 instance. Runs Prometheus (scraping the other three tiers via
**EC2 service discovery**, not static IPs), Grafana (dashboards + datasource
auto-provisioned), and blackbox_exporter (HTTP/HTTPS synthetic probes of the
frontend and backend). node_exporter also runs here so the monitoring box
monitors itself.

> Everything in this doc is what `userdata/prometheus.sh` already automates
> on `terraform apply` — this is the manual-runbook version, useful if you're
> building the box by hand or debugging a step that failed.

---

## Install Prometheus

```bash
useradd --system --no-create-home --shell /sbin/nologin prometheus

cd /opt
curl -sL -o prometheus-3.13.2.linux-amd64.tar.gz \
  "https://github.com/prometheus/prometheus/releases/download/v3.13.2/prometheus-3.13.2.linux-amd64.tar.gz"
tar -xzf prometheus-3.13.2.linux-amd64.tar.gz
ln -sfn prometheus-3.13.2.linux-amd64 /opt/prometheus

mkdir -p /var/lib/prometheus
```

**Config lives in `/opt/prometheus`**, not `/etc/prometheus` — deliberately
alongside the binary (the `/opt/prometheus` symlink target), so
`recording_rules.yaml`/`alerting_rules.yaml` and `prometheus.yml` all sit in
one place. Fetch the pre-built rules + Grafana dashboards from this repo:

```bash
curl -o /tmp/expense-prometheus.tar.gz \
  https://raw.githubusercontent.com/daws-90s/expense-prometheus-grafana-docs/main/artifacts/expense-prometheus-v1.tar.gz
mkdir -p /tmp/expense-prometheus && cd /tmp/expense-prometheus
tar -xzf /tmp/expense-prometheus.tar.gz

cp recording_rules.yaml alerting_rules.yaml /opt/prometheus/
```

Write `/opt/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "/opt/prometheus/recording_rules.yaml"
  - "/opt/prometheus/alerting_rules.yaml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    ec2_sd_configs:
      - region: us-east-1
        port: 9100
        filters:
          - name: "tag:Project"
            values: ["expense"]
    relabel_configs:
      - source_labels: [__meta_ec2_instance_state]
        regex: running
        action: keep
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance
      - source_labels: [__meta_ec2_tag_Tier]
        target_label: tier

  - job_name: "mysqld_exporter"
    ec2_sd_configs:
      - region: us-east-1
        port: 9104
        filters:
          - name: "tag:Project"
            values: ["expense"]
          - name: "tag:Tier"
            values: ["database"]
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance
      - source_labels: [__meta_ec2_tag_Tier]
        target_label: tier

  - job_name: "expense-backend"
    ec2_sd_configs:
      - region: us-east-1
        port: 8080
        filters:
          - name: "tag:Project"
            values: ["expense"]
          - name: "tag:Tier"
            values: ["backend"]
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance
      - source_labels: [__meta_ec2_tag_Tier]
        target_label: tier

  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - http://backend.daws86s.fun:8080/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115

  - job_name: 'blackbox-https'
    metrics_path: /probe
    params:
      module: [http_2xx_tls]
    static_configs:
      - targets:
          - https://daws86s.fun
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115
```

Notes on the two exporter-style SD jobs vs. the two blackbox jobs:
- `node_exporter`/`mysqld_exporter`/`expense-backend` use `ec2_sd_configs` —
  Prometheus calls `DescribeInstances` (via the instance's IAM role, no
  static AWS keys anywhere) and discovers **instances**, filtered by the
  `Project`/`Tier` tags every EC2 instance in this stack carries.
- `blackbox-http`/`blackbox-https` use `static_configs` instead — blackbox
  probes **endpoints** (URLs), not instances, so there's nothing for EC2 SD
  to discover here. The `relabel_configs` block is what actually makes the
  probe work: it takes the URL you listed under `targets`, moves it into the
  `target` query param blackbox_exporter expects, and points `__address__`
  at blackbox_exporter's own `127.0.0.1:9115` — this is the standard
  Prometheus blackbox-probing pattern, not specific to this repo.
- `blackbox-http` probes the **backend's** `/health` directly (bypassing
  nginx) with the plain `http_2xx` module. `blackbox-https` probes the
  **frontend's** public HTTPS endpoint with `http_2xx_tls` — see the
  blackbox section below for why that module needs
  `insecure_skip_verify: true`.

```bash
chown -R prometheus:prometheus /opt/prometheus-3.13.2.linux-amd64 /var/lib/prometheus
```

Systemd unit (`/etc/systemd/system/prometheus.service`):

```ini
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
User=prometheus
Group=prometheus
Restart=on-failure
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

`--web.enable-lifecycle` lets you `curl -X POST http://localhost:9090/-/reload`
after editing `prometheus.yml`, without a full restart.

```bash
systemctl daemon-reload
systemctl enable --now prometheus
```

---

## Install node_exporter

Same block as every other tier — see `02-backend.md`'s equivalent section.
Runs on port `9100`.

---

## Install blackbox_exporter

```bash
useradd --system --no-create-home --shell /sbin/nologin blackbox

cd /opt
curl -sL -o blackbox_exporter-0.25.0.linux-amd64.tar.gz \
  "https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz"
tar -xzf blackbox_exporter-0.25.0.linux-amd64.tar.gz
ln -sfn blackbox_exporter-0.25.0.linux-amd64 /opt/blackbox_exporter
chown -R blackbox:blackbox /opt/blackbox_exporter-0.25.0.linux-amd64
```

Write `/etc/blackbox/blackbox.yml`:

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      method: GET
      valid_status_codes: [200]
      preferred_ip_protocol: "ip4"

  http_2xx_tls:
    prober: http
    timeout: 5s
    http:
      method: GET
      valid_status_codes: [200]
      preferred_ip_protocol: "ip4"
      tls_config:
        insecure_skip_verify: true
```

`insecure_skip_verify: true` on `http_2xx_tls` is required because the
frontend's HTTPS cert is the 1-hour **self-signed** demo cert from
`03-frontend.md` — without it, the TLS handshake fails and `probe_success`
is always `0`, regardless of whether the cert is expired. The metric that
actually matters for the expiry demo is `probe_ssl_earliest_cert_expiry`,
not `probe_success` — skipping verification just lets the probe get far
enough to read the cert's `notAfter` at all.

```bash
chown -R blackbox:blackbox /etc/blackbox
```

Systemd unit (`/etc/systemd/system/blackbox_exporter.service`):

```ini
[Unit]
Description=Blackbox Exporter
After=network-online.target
Wants=network-online.target

[Service]
User=blackbox
Group=blackbox
Restart=on-failure
ExecStart=/opt/blackbox_exporter/blackbox_exporter \
  --config.file=/etc/blackbox/blackbox.yml \
  --web.listen-address=:9115

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now blackbox_exporter
```

### Testing a probe directly (don't wait for Prometheus's scrape cycle)

```bash
curl "http://localhost:9115/probe?target=https://daws86s.fun&module=http_2xx_tls"
curl "http://localhost:9115/probe?target=http://backend.daws86s.fun:8080/health&module=http_2xx"
```

Both return raw Prometheus-exposition text immediately. Look for
`probe_success 1` and, for the HTTPS one, `probe_ssl_earliest_cert_expiry`
(a Unix timestamp — subtract `time()` and divide by 3600 for hours
remaining, which is exactly what
`instance:probe_ssl_earliest_cert_expiry:hours_remaining` in
`recording_rules.yaml` does). If `probe_success` comes back `0`, check the
target is actually reachable and listening before suspecting the blackbox
config — port 443 has nothing on it until you run `cert-demo.sh`
(`03-frontend.md`).

---

## Install Grafana

```bash
dnf install -y grafana
```

No separate repo file needed — the `Redhat-9-DevOps-Practice` AMI's
existing repos already carry the `grafana` package.

### Datasource provisioning

Must exist **before** `grafana-server` first starts, so Prometheus is
auto-wired with a fixed `uid` that the dashboard JSON files also reference
(`"uid": "prometheus"` — check any dashboard's panel `datasource` block if
you want to confirm).

`/etc/grafana/provisioning/datasources/prometheus.yml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    uid: prometheus
    url: http://localhost:9090
    isDefault: true
    editable: false
```

### Dashboard provisioning

`/etc/grafana/provisioning/dashboards/dashboards.yml`:

```yaml
apiVersion: 1
providers:
  - name: expense-app-dashboards
    orgId: 1
    folder: "Expense App"
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: false
    options:
      path: /etc/grafana/dashboards
      foldersFromFilesStructure: false
```

Install the actual dashboard JSON files (from the same
`expense-prometheus-v1.tar.gz` you extracted for the rules files above):

```bash
mkdir -p /etc/grafana/dashboards
cp /tmp/expense-prometheus/dashboards/*.json /etc/grafana/dashboards/
chown -R grafana:grafana /etc/grafana/provisioning /etc/grafana/dashboards
```

Seven dashboards ship in the "Expense App" folder: fleet overview,
node/host resource use, MySQL, application RED metrics, business metrics,
blackbox synthetic checks, and an SLO error-budget view.

```bash
systemctl daemon-reload
systemctl enable --now grafana-server
```

---

## Verification

```bash
curl -s localhost:9090/-/ready               # prometheus
curl -s localhost:9115/metrics | head        # blackbox_exporter is up
curl -s -o /dev/null -w '%{http_code}\n' localhost:3000   # grafana (200/302)
```

In the Prometheus UI (**Status → Targets**), you should see six jobs, all
`UP`: `prometheus`, `node_exporter` (4 targets), `mysqld_exporter` (1),
`expense-backend` (1), `blackbox-http` (1), `blackbox-https` (1 — only
after `cert-demo.sh` has been run on the frontend).

Useful PromQL to sanity-check the whole stack end to end:

```promql
up == 0                                          # anything down, across every job
mysql_up == 1
up{job="expense-backend"} == 1
probe_success{job="blackbox-http"} == 1
probe_success{job="blackbox-https"} == 1
instance:probe_ssl_earliest_cert_expiry:hours_remaining
```

Log into Grafana at `http://<PROMETHEUS-PUBLIC-IP>:3000` (default
`admin`/`admin`, it'll prompt you to change it on first login) and open the
**Expense App** folder — all seven dashboards should already be there with
the Prometheus datasource pre-wired, no manual setup needed.

For triggering the alerts these dashboards/rules watch for (on purpose, to
demo them), see **"Inducing faults"** in the main
`expense-prometheus-grafana` repo's `README.md`.

---

## Security Group

Ensure the Prometheus EC2 security group allows:
- **inbound TCP `9090`** (Prometheus UI) and **`3000`** (Grafana UI) from
  wherever you're browsing from,
- **inbound TCP `22`** for SSH,
- and **all outbound** (it needs to reach the AWS API for `DescribeInstances`
  plus every scrape target).

The app-tier security groups (mysql/backend/frontend) each need matching
inbound rules **sourced from the Prometheus security group** — `9100` on
all three, `9104` on mysql, `8080` (backend_port) on backend, and `443` on
frontend for `blackbox-https` — never opened to `0.0.0.0/0`. See
`security_groups.tf` in the main repo for the exact rules.
