# Frontend

The frontend is a React (Vite) SPA, compiled to static files and served by
Nginx. Nginx also acts as a reverse proxy, forwarding `/api/` requests to the
backend privately — the browser never talks to the backend directly.

---

## Install Node.js and Nginx

```bash
dnf module disable nodejs -y
dnf module enable nodejs:20 -y
dnf install nodejs nginx -y
```

Enable and start nginx:

```bash
systemctl enable nginx
systemctl start nginx
```

Verify the default Nginx page is accessible at `http://<FRONTEND-PUBLIC-IP>`.

---

## Build the App

This tier ships as source, not a pre-built bundle — Vite bakes
`VITE_API_URL` into the compiled JS at build time, so the build has to
happen where you know the final config.

```bash
curl -o /tmp/frontend.tar.gz \
  https://raw.githubusercontent.com/daws-90s/expense-prometheus-grafana-docs/main/artifacts/expense-frontend-v1.tar.gz
mkdir -p /tmp/expense-frontend && cd /tmp/expense-frontend
tar -xzf /tmp/frontend.tar.gz
npm install
```

**Do not set `VITE_API_URL`** for a production build — leaving it unset
makes the app call relative `/api/...` paths, which nginx proxies to the
backend. `VITE_API_URL` is only for local dev without nginx in front. If a
leftover `.env` sets it, `npm run build` will bake that absolute URL into
the bundle and the reverse proxy will be bypassed (and likely unreachable,
since the backend's SG only accepts the frontend's SG).

```bash
npm run build      # outputs dist/
```

---

## Deploy Frontend Content

```bash
rm -rf /usr/share/nginx/html/*
cp -r dist/* /usr/share/nginx/html/
```

---

## Configure Nginx Reverse Proxy

```bash
vim /etc/nginx/default.d/expense.conf
```

```nginx
# No `root` directive here -- the parent server block in /etc/nginx/nginx.conf
# already sets root /usr/share/nginx/html, and nginx errors on a duplicate
# directive in the same context even when the value matches.

index index.html;

location / {
    try_files $uri $uri/ /index.html;
}

location /api/ {
    proxy_pass         http://<BACKEND-PRIVATE-IP>:8080/api/;
    proxy_http_version 1.1;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_connect_timeout 5s;
    proxy_read_timeout    30s;
}

location /health {
    stub_status on;
    access_log off;
}
```

> **Replace `<BACKEND-PRIVATE-IP>`** with the backend EC2's private IP, or
> `backend.daws86s.fun` if Route 53 is set up (see `expense-prometheus-grafana`).

Test and reload:

```bash
nginx -t
systemctl reload nginx
```

---

## Verification

Open `http://<FRONTEND-PUBLIC-IP>` and confirm the dashboard loads with
categories and expenses.

Test the reverse proxy directly on the box:

```bash
curl http://localhost/api/categories
```

You should get a JSON array back from the backend, not a connection error.

If the UI loads but data doesn't, open DevTools → Network and check which
URL the failing requests hit:
- `backend...:8080/api/...` directly → the build baked in `VITE_API_URL`;
  clear `.env`, rebuild, redeploy (see above).
- relative `/api/...` failing → nginx can't reach the backend; check the
  backend is running and the security groups allow frontend → backend on
  8080.

---

## Security Group

Ensure the frontend EC2 security group allows **inbound TCP on port 80**
(and **443**, if you run the HTTPS demo below) from the internet
(0.0.0.0/0). Port 8080 on the backend must **not** be open to the internet
— only to the frontend EC2's security group.

---

## HTTPS (self-signed 1-hour demo cert)

Not part of the automatic `terraform apply` flow — this is an on-demand,
manual step you run once nginx is already up, to demo a cert-expiry alert
end to end. The artifact tarball includes two files for this: `nginx-tls.conf`
and `cert-demo.sh` (both extracted alongside `nginx.conf` in the **Build the
App** step above, into `/tmp/expense-frontend`).

```bash
cd /tmp/expense-frontend
sudo ./cert-demo.sh                 # or: sudo ./cert-demo.sh <public-ip-or-hostname>
```

What it does:

1. Generates a **1-hour** self-signed key/cert into `/etc/nginx/tls/`. Plain
   `openssl req -x509 -days N` can't express an interval shorter than 24h, so
   the script picks between two approaches automatically based on the box's
   OpenSSL version:
   - **OpenSSL ≥ 3.2** — uses `-not_after <timestamp>` directly.
   - **Older OpenSSL** — temporarily shifts the system clock forward 23
     hours, issues a normal `-days 1` cert (which computes `notAfter` as
     real-now + 1 hour), then force-resyncs the clock (`chronyc -a makestep`,
     falling back to `systemctl restart systemd-timesyncd`).
2. If you don't pass an IP/hostname argument, it looks up the box's own
   public IPv4 via IMDSv2 and uses that as the cert's `subjectAltName`.
3. `chmod 600` on the private key.
4. Copies `nginx-tls.conf` to **`/etc/nginx/conf.d/expense-tls.conf`** —
   deliberately `conf.d/`, not `default.d/` alongside `nginx.conf`.
   `default.d/*.conf` fragments are included *inside* the AMI's existing
   port-80 `server {}` block, so they can't contain a second `server {}`;
   `conf.d/*.conf` files are included at the `http` level and can hold a
   full standalone `server { listen 443 ssl; ... }` block, which is what
   this needs.
5. `nginx -t` then `systemctl reload nginx` to pick it up. Port 80 keeps
   serving via the original config throughout — this only adds 443, it
   doesn't touch or replace anything.

Verify:

```bash
curl -kI https://<FRONTEND-PUBLIC-IP>
openssl x509 -enddate -noout -in /etc/nginx/tls/expense.crt
```

`-k` is required — it's a self-signed cert, so curl won't trust it without
that flag (same reason blackbox_exporter's `http_2xx_tls` module needs
`insecure_skip_verify: true` — see the monitoring doc). Because the cert is
only good for an hour, `probe_ssl_earliest_cert_expiry` in Prometheus counts
down for real, which is the whole point of the demo — it's what
`SSLCertExpiringSoon`/`SSLCertExpiringCritical` in
`prometheus/alerting-rules.yaml` actually watch.

Re-run `./cert-demo.sh` any time to reissue a fresh 1-hour cert and repeat
the drill.
