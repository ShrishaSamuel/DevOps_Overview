# Nginx — Zero to Hero

> A complete, beginner-to-advanced guide to Nginx as a web server, reverse proxy, and load balancer.

---

## Table of Contents

1. [Introduction & Theory](#1-introduction--theory)
2. [Installation & Setup](#2-installation--setup)
3. [Core Concepts](#3-core-concepts)
4. [Hands-on Tasks](#4-hands-on-tasks)
5. [Projects](#5-projects)
6. [Best Practices & Common Pitfalls](#6-best-practices--common-pitfalls)
7. [Interview Questions](#7-interview-questions)
8. [Quizzes](#8-quizzes)
9. [Further Resources](#9-further-resources)

---

## 1. Introduction & Theory

### 1.1 What is Nginx?

**Nginx** (pronounced "engine-x") is a high-performance, open-source **web server, reverse proxy, load balancer, and HTTP cache**. Created by Igor Sysoev in 2004 to solve the **C10k problem** (handling 10,000+ concurrent connections), it has become one of the most widely deployed web servers in the world, powering a large share of the busiest sites. Its event-driven architecture makes it exceptionally efficient at serving static content and proxying traffic.

### 1.2 What Nginx does

| Role | Description |
|------|-------------|
| **Web server** | Serve static files (HTML, CSS, JS, images) fast |
| **Reverse proxy** | Forward client requests to backend app servers |
| **Load balancer** | Distribute traffic across multiple backends |
| **HTTP cache** | Cache responses to reduce backend load |
| **TLS terminator** | Handle HTTPS/SSL so backends don't have to |
| **API gateway** | Route, rate-limit, and secure API traffic |

### 1.3 The C10k problem and event-driven architecture

Older servers (like Apache's prefork model) spawn a **thread/process per connection**, which consumes lots of memory and context-switching under high concurrency. Nginx instead uses an **asynchronous, event-driven** model: a small number of **worker processes** each handle thousands of connections using a non-blocking event loop.

```
   Apache (process/thread per connection)     Nginx (event-driven workers)
   ─────────────────────────────────────     ────────────────────────────
   conn1 → thread1                            conn1 ┐
   conn2 → thread2                            conn2 ┤
   conn3 → thread3                            conn3 ├─► worker (event loop)
   ...    (memory grows linearly)             ...   ┘   handles all async
```

This is why Nginx uses **less memory** and **scales** to massive concurrency.

### 1.4 Master/worker process model

```
   Master process (root)
     │ reads config, binds ports, manages workers
     ├── Worker process 1  (event loop, handles connections)
     ├── Worker process 2
     └── Worker process N   (typically one per CPU core)
```

- **Master:** Reads/validates config, binds privileged ports, and spawns/manages workers (does not handle requests itself).
- **Workers:** Handle actual connections via an efficient event loop (epoll/kqueue). Usually `worker_processes auto;` = one per core.

### 1.5 Forward proxy vs. reverse proxy

```
   Forward proxy (on behalf of clients)
   Client ──► Forward Proxy ──► Internet/Server

   Reverse proxy (on behalf of servers) ← Nginx's job
   Client ──► Reverse Proxy (Nginx) ──► Backend app servers
```

A **reverse proxy** sits in front of backend servers, accepting client requests and forwarding them — providing load balancing, TLS termination, caching, and a single entry point.

### 1.6 Request processing flow

```
   Request ──► Nginx
     1. Match server block (by host/port)        [virtual host]
     2. Match location block (by URI)            [routing]
     3. Apply directives (root/proxy_pass/...)   [action]
     4. Serve file OR proxy to backend
     5. Apply caching, headers, logging
   ──► Response
```

### 1.7 When to use Nginx

- Serving static websites/assets and SPAs.
- Reverse-proxying application servers (Node, Python, Java, PHP-FPM).
- Load balancing across app instances.
- TLS termination and HTTP/2/HTTP/3.
- API gateway, rate limiting, and caching layer.
- Kubernetes Ingress (via the ingress-nginx controller).

When **not** ideal: as the application runtime itself (it proxies, not executes app code, except via modules); extremely dynamic config needs may favor service meshes/Envoy.

---

## 2. Installation & Setup

### 2.1 Install Nginx

```bash
# Debian/Ubuntu
sudo apt update && sudo apt install -y nginx

# RHEL/CentOS/Fedora
sudo dnf install -y nginx

# macOS
brew install nginx

# Docker
docker run -d -p 8080:80 --name web nginx:stable
```

### 2.2 Start and verify

```bash
sudo systemctl enable --now nginx     # start + enable at boot
sudo systemctl status nginx
curl -I http://localhost               # check it responds
```

**Expected:** `HTTP/1.1 200 OK` and the default "Welcome to nginx!" page.

### 2.3 Key file locations (Debian/Ubuntu)

```
/etc/nginx/nginx.conf            # main configuration
/etc/nginx/sites-available/      # available site configs
/etc/nginx/sites-enabled/        # enabled sites (symlinks)
/etc/nginx/conf.d/               # additional config snippets
/var/www/html/                   # default web root
/var/log/nginx/access.log        # access log
/var/log/nginx/error.log         # error log
```

### 2.4 Essential commands

```bash
sudo nginx -t                    # TEST config (always before reload!)
sudo systemctl reload nginx      # apply config with zero downtime
sudo systemctl restart nginx     # full restart
sudo nginx -s reload             # alternative reload signal
nginx -v                         # version
```

> **Always run `nginx -t` before reloading** — it validates the config and prevents taking the server down with a syntax error.

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Configuration structure: contexts and directives

Nginx config is made of **directives** (settings) organized into **contexts** (blocks):

```nginx
# main context (global)
user www-data;
worker_processes auto;

events {                      # events context
  worker_connections 1024;
}

http {                        # http context
  include mime.types;
  default_type application/octet-stream;
  sendfile on;

  server {                    # server context (virtual host)
    listen 80;
    server_name example.com;

    location / {              # location context (routing)
      root /var/www/html;
      index index.html;
    }
  }
}
```

- **Directives** end with `;`.
- **Contexts** are `{ }` blocks that scope directives.
- Inner contexts inherit from outer ones.

#### 3.1.2 Serving static content

```nginx
server {
  listen 80;
  server_name mysite.com;
  root /var/www/mysite;
  index index.html;

  location / {
    try_files $uri $uri/ =404;     # serve file, dir, or 404
  }
}
```

`root` sets the document root; `try_files` attempts each path in order.

#### 3.1.3 Server blocks (virtual hosts)

Multiple sites on one Nginx via `server_name`:

```nginx
server {
  listen 80;
  server_name site-a.com;
  root /var/www/site-a;
}
server {
  listen 80;
  server_name site-b.com;
  root /var/www/site-b;
}
```

Nginx routes by the `Host` header to the matching server block.

#### 3.1.4 Enabling a site (Debian pattern)

```bash
sudo nano /etc/nginx/sites-available/mysite
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

#### 3.1.5 Logs

```nginx
access_log /var/log/nginx/mysite-access.log;
error_log  /var/log/nginx/mysite-error.log warn;
```

```bash
sudo tail -f /var/log/nginx/access.log     # watch traffic live
```

### 3.2 INTERMEDIATE

#### 3.2.1 Location matching

```nginx
location = /exact { }          # exact match (highest priority)
location ^~ /images/ { }       # prefix, stop regex search
location ~ \.php$ { }          # case-sensitive regex
location ~* \.(jpg|png)$ { }   # case-insensitive regex
location /docs/ { }            # prefix match
location / { }                 # catch-all (lowest priority)
```

**Priority order:** exact (`=`) → `^~` prefix → regex (`~`/`~*`, first match) → longest prefix → `/`.

#### 3.2.2 Reverse proxy

```nginx
server {
  listen 80;
  server_name app.example.com;

  location / {
    proxy_pass http://127.0.0.1:3000;          # forward to backend
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

The `proxy_set_header` directives pass real client info to the backend (otherwise the app sees Nginx's IP).

#### 3.2.3 Load balancing

```nginx
upstream backend {
  # default: round-robin
  server 10.0.0.1:3000;
  server 10.0.0.2:3000;
  server 10.0.0.3:3000 weight=2;     # gets twice the traffic
  server 10.0.0.4:3000 backup;       # only used if others fail
}

server {
  location / {
    proxy_pass http://backend;
  }
}
```

**Algorithms:**
- **Round-robin** (default) — rotate through servers.
- **least_conn** — send to the server with fewest active connections.
- **ip_hash** — same client IP → same server (session stickiness).
- **hash $request_uri** — consistent hashing by key.

```nginx
upstream backend {
  least_conn;
  server 10.0.0.1:3000;
  server 10.0.0.2:3000;
}
```

#### 3.2.4 TLS/HTTPS

```nginx
server {
  listen 443 ssl;
  http2 on;
  server_name example.com;

  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5;

  location / { proxy_pass http://backend; }
}

# Redirect HTTP → HTTPS
server {
  listen 80;
  server_name example.com;
  return 301 https://$host$request_uri;
}
```

Use **Let's Encrypt / Certbot** for free, auto-renewing certificates:

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d example.com
```

#### 3.2.5 Caching

```nginx
# Define a cache zone (in http context)
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mycache:10m max_size=1g inactive=60m;

server {
  location / {
    proxy_cache mycache;
    proxy_cache_valid 200 10m;
    proxy_cache_valid 404 1m;
    add_header X-Cache-Status $upstream_cache_status;
    proxy_pass http://backend;
  }
}

# Cache static assets in the browser
location ~* \.(jpg|css|js)$ {
  expires 30d;
  add_header Cache-Control "public, immutable";
}
```

#### 3.2.6 Rewrites and redirects

```nginx
return 301 https://newsite.com$request_uri;     # permanent redirect

rewrite ^/old/(.*)$ /new/$1 permanent;          # regex rewrite

location /api/ {
  rewrite ^/api/(.*)$ /$1 break;                 # strip /api prefix
  proxy_pass http://backend;
}
```

### 3.3 ADVANCED

#### 3.3.1 Performance tuning

```nginx
worker_processes auto;                  # one per CPU core
worker_rlimit_nofile 65535;             # file descriptor limit

events {
  worker_connections 4096;              # per worker
  multi_accept on;
  use epoll;                            # Linux event method
}

http {
  sendfile on;                          # efficient file serving
  tcp_nopush on;
  tcp_nodelay on;
  keepalive_timeout 65;
  gzip on;                              # compress responses
  gzip_types text/css application/javascript application/json;
}
```

Max connections ≈ `worker_processes × worker_connections`.

#### 3.3.2 Rate limiting and connection limits

```nginx
# Limit request rate per client IP (http context)
limit_req_zone $binary_remote_addr zone=reqlimit:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=connlimit:10m;

server {
  location /api/ {
    limit_req zone=reqlimit burst=20 nodelay;     # allow bursts
    limit_conn connlimit 10;                        # max concurrent
    proxy_pass http://backend;
  }
}
```

Protects against abuse, scraping, and basic DoS.

#### 3.3.3 Security hardening

```nginx
server_tokens off;                      # hide Nginx version

# Security headers
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header Content-Security-Policy "default-src 'self'" always;

# Block hidden files
location ~ /\.(?!well-known) { deny all; }

# Restrict by IP
location /admin/ {
  allow 10.0.0.0/8;
  deny all;
}
```

Combine with TLS best practices, WAF (e.g., ModSecurity), and fail2ban.

#### 3.3.4 FastCGI / app server integration

```nginx
# PHP-FPM
location ~ \.php$ {
  include fastcgi_params;
  fastcgi_pass unix:/run/php/php8.2-fpm.sock;
  fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

Nginx proxies to PHP-FPM (FastCGI), uWSGI (Python), or HTTP backends (Node, Java, Go).

#### 3.3.5 WebSockets and HTTP upgrades

```nginx
location /ws/ {
  proxy_pass http://backend;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  proxy_read_timeout 3600s;
}
```

#### 3.3.6 Health checks and upstream resilience

```nginx
upstream backend {
  server 10.0.0.1:3000 max_fails=3 fail_timeout=30s;
  server 10.0.0.2:3000 max_fails=3 fail_timeout=30s;
  keepalive 32;                         # reuse upstream connections
}
```

Open-source Nginx uses **passive** health checks (`max_fails`/`fail_timeout`); **active** health checks are an Nginx Plus feature (or use a separate probe).

#### 3.3.7 Nginx as Kubernetes Ingress

The **ingress-nginx** controller runs Nginx inside Kubernetes, translating `Ingress` resources into Nginx config to route external traffic to services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-svc
                port: { number: 80 }
```

#### 3.3.8 Logging, monitoring, and debugging

```nginx
log_format detailed '$remote_addr - $request "$status" '
                    '$body_bytes_sent rt=$request_time '
                    'uct=$upstream_connect_time urt=$upstream_response_time';
access_log /var/log/nginx/access.log detailed;
```

- Use **stub_status** or the Prometheus **nginx-exporter** for metrics.
- Tail `error.log` for diagnostics; raise `error_log` to `debug` temporarily.
- Custom `log_format` captures latency/upstream timing for performance analysis.

---

## 4. Hands-on Tasks

### Task 1: Install and serve the default page

Install Nginx, start it, and `curl -I http://localhost`.

**Expected:** `200 OK` and the welcome page.

### Task 2: Serve a custom static site

Create a server block with a custom `root` and an `index.html`; enable it.

**Expected:** Your page loads at the configured `server_name`/port.

### Task 3: Validate and reload safely

Edit config, run `sudo nginx -t`, then `systemctl reload nginx`.

**Expected:** Test passes; changes apply without downtime.

### Task 4: Host two virtual hosts

Define two server blocks with different `server_name`s and roots.

**Expected:** Each hostname serves its own site.

### Task 5: Set up a reverse proxy

Proxy `/` to a local app on port 3000 with proper `proxy_set_header`s.

**Expected:** Requests reach the backend; it sees the real client IP.

### Task 6: Load balance two backends

Create an `upstream` with two servers and `proxy_pass` to it.

**Expected:** Requests alternate across backends (round-robin).

### Task 7: Enable HTTPS with Certbot

Run `certbot --nginx -d <domain>`; verify the HTTP→HTTPS redirect.

**Expected:** Site served over TLS; HTTP redirects to HTTPS.

### Task 8: Add caching

Configure `proxy_cache` and observe `X-Cache-Status` headers.

**Expected:** Repeat requests show `HIT`.

### Task 9: Add rate limiting

Apply `limit_req` to `/api/` and hammer it with requests.

**Expected:** Excess requests get `503`/`429` after the burst.

### Task 10: Add security headers and hide version

Set `server_tokens off;` and add security headers; verify with `curl -I`.

**Expected:** Version hidden; security headers present.

---

## 5. Projects

### Project 1 (Beginner): Static Website with Custom Config

**Goal:** Host a multi-page static site properly.

Steps:
1. Create a server block with `root`, `index`, and `try_files`.
2. Add custom error pages (404/50x).
3. Cache static assets with `expires`/`Cache-Control`.
4. Enable `gzip`. Validate with `nginx -t` and reload.
5. Tail access logs to verify requests.

**Skills:** server/location blocks, static serving, caching, logging.

### Project 2 (Intermediate): Reverse Proxy + Load Balancer for an App

**Goal:** Front a multi-instance app with TLS.

Steps:
1. Run an app on 2–3 backend ports; define an `upstream` (least_conn).
2. Reverse proxy with correct forwarded headers and WebSocket support.
3. Terminate **TLS** with Let's Encrypt; redirect HTTP→HTTPS.
4. Add **proxy caching** and `X-Cache-Status`.
5. Add **rate limiting** and **security headers**; hide the version.
6. Configure passive health checks (`max_fails`/`fail_timeout`).

**Skills:** reverse proxy, load balancing, TLS, caching, rate limiting, resilience.

### Project 3 (Advanced): Production API Gateway / Ingress

**Goal:** A hardened, observable edge for microservices.

Steps:
1. Route multiple services by path/host (API gateway) with rewrites.
2. Implement **rate limiting**, **connection limits**, and **auth** (e.g., `auth_request`).
3. Tune performance (workers, keepalive, `gzip`, buffers) for high concurrency.
4. Harden security (TLS 1.2/1.3, HSTS, CSP, IP allowlists, `server_tokens off`).
5. Add **detailed logging** + Prometheus **nginx-exporter** dashboards.
6. (Optional) Deploy as **ingress-nginx** in Kubernetes with annotations.
7. Document blue-green/canary routing.

**Skills:** API gateway patterns, security/rate limiting, performance tuning, observability, Kubernetes ingress.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Always run `nginx -t` before reloading**; prefer `reload` over `restart`.
- **Set `worker_processes auto;`** and tune `worker_connections` for your load.
- **Pass real client info** with `proxy_set_header` (Host, X-Real-IP, X-Forwarded-*).
- **Terminate TLS** with modern protocols (1.2/1.3) and automate certs (Certbot).
- **Enable gzip** and **browser/proxy caching** to cut bandwidth and load.
- **Hide the version** (`server_tokens off;`) and add **security headers**.
- **Use rate/connection limiting** to mitigate abuse.
- **Use `try_files`** instead of fragile `if` blocks (avoid "if is evil").
- **Organize config** with `sites-available`/`sites-enabled` and includes.
- **Monitor** access/error logs and export metrics.
- **Set sensible timeouts** and upstream `keepalive`.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Reloading without `nginx -t` | Outage from bad config | Always test first |
| Missing `proxy_set_header Host` | Backend gets wrong host | Set forwarding headers |
| Overusing `if` in config | Subtle bugs ("if is evil") | Use `try_files`/maps |
| Wrong location priority assumptions | Unexpected routing | Learn match order |
| No TLS / weak ciphers | Insecure traffic | TLS 1.2/1.3, strong ciphers |
| Exposing version/banner | Easier fingerprinting | `server_tokens off;` |
| No rate limiting on APIs | Abuse/DoS | `limit_req`/`limit_conn` |
| Forgetting WebSocket upgrade headers | Broken WS connections | `Upgrade`/`Connection` headers |
| Serving large files without `sendfile` | Poor performance | Enable `sendfile`, tune buffers |
| Ignoring error.log | Hard debugging | Tail and raise log level |

---

## 7. Interview Questions

### Beginner

1. **What is Nginx?**
   *A high-performance web server, reverse proxy, load balancer, and HTTP cache known for its event-driven efficiency.*

2. **What is a reverse proxy?**
   *A server that sits in front of backends, forwarding client requests to them and returning responses — providing load balancing, TLS termination, and caching.*

3. **What is a server block?**
   *A virtual host configuration that defines how Nginx handles requests for a particular `server_name`/port.*

4. **Why run `nginx -t`?**
   *It validates the configuration before applying it, preventing a syntax error from taking the server down.*

5. **What's the difference between reload and restart?**
   *Reload applies new config gracefully with no downtime; restart fully stops and starts the service.*

### Intermediate

6. **How does Nginx's architecture differ from Apache's prefork?**
   *Nginx uses an event-driven model with a few worker processes handling many connections asynchronously, while prefork Apache uses a process/thread per connection — so Nginx scales with far less memory.*

7. **Explain location matching priority.**
   *Exact (`=`) first, then `^~` prefix, then regex (`~`/`~*`) in order, then the longest prefix match, then `/`.*

8. **What load balancing algorithms does Nginx support?**
   *Round-robin (default), least_conn, ip_hash (sticky), and generic hash/consistent hashing.*

9. **How do you pass the real client IP to a backend?**
   *Set `proxy_set_header X-Real-IP $remote_addr;` and `X-Forwarded-For $proxy_add_x_forwarded_for;` (and have the app trust these).*

10. **How does Nginx caching work?**
    *Define a `proxy_cache_path` zone and enable `proxy_cache` with `proxy_cache_valid`; Nginx stores backend responses and serves cached copies, reducing backend load.*

### Advanced

11. **What is the master/worker process model?**
    *The master reads config and binds ports while worker processes (typically one per core) handle connections via an event loop; the master manages workers and enables zero-downtime reloads.*

12. **How do you rate-limit requests in Nginx?**
    *Define a `limit_req_zone` keyed on client IP and apply `limit_req` with a rate and burst in the relevant location, optionally with `limit_conn` for concurrent connections.*

13. **How do you support WebSockets through Nginx?**
    *Use HTTP/1.1 and forward upgrade headers: `proxy_http_version 1.1`, `proxy_set_header Upgrade $http_upgrade`, and `Connection "upgrade"`, plus longer read timeouts.*

14. **Why is "if" considered evil in Nginx, and what do you use instead?**
    *`if` inside location blocks has surprising, often buggy behavior; prefer `try_files`, `map`, and proper location matching for predictable routing.*

15. **How is Nginx used as a Kubernetes Ingress controller?**
    *The ingress-nginx controller watches `Ingress` resources and dynamically generates Nginx config to route external traffic to cluster services, handling TLS, rewrites, and annotations.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Nginx was created to solve the:
- A) C10k problem  B) Y2K problem  C) DNS problem  D) TLS problem

**Q2.** Which process handles actual connections?
- A) Master  B) Worker  C) Init  D) Cache manager

**Q3.** A virtual host is configured with a:
- A) location block  B) server block  C) upstream block  D) events block

**Q4.** Which command tests config validity?
- A) `nginx -s`  B) `nginx -t`  C) `nginx -v`  D) `nginx -r`

**Q5.** Default load balancing algorithm:
- A) least_conn  B) ip_hash  C) round-robin  D) random

**Q6.** Which provides session stickiness by client IP?
- A) round-robin  B) ip_hash  C) least_conn  D) weight

**Q7.** Which directive forwards requests to a backend?
- A) `root`  B) `proxy_pass`  C) `try_files`  D) `return`

**Q8.** Which hides the Nginx version?
- A) `server_tokens off`  B) `autoindex off`  C) `gzip off`  D) `sendfile off`

**Q9.** Highest-priority location match is:
- A) regex  B) prefix  C) exact `=`  D) `/`

**Q10.** Which is used to rate-limit by IP?
- A) `limit_req_zone`  B) `proxy_cache`  C) `upstream`  D) `expires`

### Short Answer

**S1.** Which directive sets the document root for static files?

**S2.** Name the block used to define a group of backend servers.

**S3.** What two headers are required to proxy WebSockets?

**S4.** Which tool gets free auto-renewing TLS certs for Nginx?

**S5.** What header reveals cache hits/misses (commonly added)?

### Answer Key

**Multiple Choice:** Q1-A, Q2-B, Q3-B, Q4-B, Q5-C, Q6-B, Q7-B, Q8-A, Q9-C, Q10-A

**Short Answer:**
- **S1.** `root`.
- **S2.** `upstream`.
- **S3.** `Upgrade` and `Connection` (with `proxy_http_version 1.1`).
- **S4.** Certbot (Let's Encrypt).
- **S5.** `X-Cache-Status` (from `$upstream_cache_status`).

---

## 9. Further Resources

### Official Documentation
- Nginx docs — https://nginx.org/en/docs/
- Beginner's guide — https://nginx.org/en/docs/beginners_guide.html
- Directive reference — https://nginx.org/en/docs/dirindex.html
- Nginx admin guide — https://docs.nginx.com/nginx/admin-guide/

### Learning
- DigitalOcean Nginx tutorials — https://www.digitalocean.com/community/tags/nginx
- Mozilla SSL Config Generator — https://ssl-config.mozilla.org/
- ingress-nginx docs — https://kubernetes.github.io/ingress-nginx/

### Tools
- Certbot (Let's Encrypt), nginxconfig.io (config generator)
- nginx-prometheus-exporter, GoAccess (log analysis)
- ModSecurity (WAF)

### Books
- *Nginx HTTP Server* — Clément Nedelcu
- *Nginx Cookbook* — Derek DeJonghe (O'Reilly)
- *Mastering Nginx* — Dimitri Aivaliotis

---

*End of Nginx — Zero to Hero.*
