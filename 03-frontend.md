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
  https://raw.githubusercontent.com/90s-org/expense-obs-documentation/main/artifacts/expense-frontend-v1.tar.gz
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
> `backend.daws86s.fun` if Route 53 is set up (see `expense-infra`).

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
from the internet (0.0.0.0/0). Port 8080 on the backend must **not** be open
to the internet — only to the frontend EC2's security group.
