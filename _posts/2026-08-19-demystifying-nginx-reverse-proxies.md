---
layout: post
title: "Demystifying Nginx Reverse Proxies - Exposing Homelabs and Routing Docker Containers Like a Pro"
date: 2026-08-19 08:00:00 +0300
categories: [homelab, networking]
tags: [nginx, docker, reverse-proxy, devops, selfhosted]
math: true
---

A couple of years ago, my homelab was an absolute dumpster fire. I had an old Lenovo ThinkCentre sitting under my desk running about a dozen Docker containers: Jellyfin for movies, Nextcloud for files, Gitea for my code, Vaultwarden for passwords, and a couple of random Go and Node.js side projects. 

Whenever I wanted to access a service from my phone or show something to my friends, I had to memorize random port numbers: `192.168.1.120:8096`, `192.168.1.120:3000`, `192.168.1.120:8080`. Worse, when I first decided to access them outside my local network, my initial "solution" was to log into my ISP router and blindly port-forward every single one of those ports to the public internet without SSL, without security headers, and without rate limiting. 

Yeah... don't do that.

Enter the **Nginx Reverse Proxy**. 

Once you understand how Layer 7 proxying works and how Nginx hooks directly into Docker internal bridge networks, you can expose fifty different self-hosted applications through standard ports (`80` and `443`) with automatic TLS certificates, clean subdomains (`git.mydomain.dev`, `media.mydomain.dev`), and hardened security policies.

Let's break down how this works under the hood, dive into low-level networking and event models, and build a production-grade homelab reverse proxy architecture.

---

## 1. How a Reverse Proxy Actually Works (Under the Hood)

To understand why Nginx is so effective, we have to look at the OSI model and differentiate between **Forward Proxies** and **Reverse Proxies**.

```
[ Client / Browser ] 
       │
       ▼ (Public WAN: Port 443 / HTTPS)
[ Router / Firewall / Cloudflare ]
       │
       ▼ (LAN / Host Port 443)
[ Nginx Reverse Proxy (Layer 7 Dispatcher) ]
       │
       ├─── (Docker Bridge: http://vaultwarden:80) ────► [ Vaultwarden Container ]
       ├─── (Docker Bridge: http://jellyfin:8096)  ────► [ Jellyfin Media Server ]
       └─── (Docker Bridge: http://nextcloud:80)   ────► [ Nextcloud App Container ]
```

A **Forward Proxy** sits in front of clients (like inside a school or corporate network) and intercepts outgoing requests to external web servers. The server thinks it is talking to the proxy, not the client.

A **Reverse Proxy** sits in front of servers and intercepts incoming requests from clients. The client believes it is communicating directly with `https://media.mydomain.dev`, but behind the scenes, Nginx receives the HTTP request, terminates TLS, inspects headers (like the `Host:` header), routes the payload through a private internal network, and shuttles the response back.

### Layer 4 vs. Layer 7 Proxying

* **Layer 4 (Transport Layer - TCP/UDP):** Routes raw packets based purely on IP addresses and ports using standard NAT or stream modules. It does not inspect HTTP headers or URLs.
* **Layer 7 (Application Layer - HTTP/HTTPS/WebSockets):** Nginx parses the entire HTTP protocol stream. It can read headers, rewrite query strings, terminate SSL certificates using Server Name Indication (SNI), inject authentication tokens, compress data on the fly with gzip/brotli, and cache static assets.

---

## 2. Low-Level Performance: Why Nginx Handles Thousands of Connections

Unlike traditional web servers (like Apache MPM prefork) that spin up an entire thread or process per incoming connection, Nginx relies on an **asynchronous, event-driven, non-blocking architecture**.

In Linux systems, Nginx leverages the `epoll` kernel system call. A small number of single-threaded worker processes (typically matched to the number of CPU cores) manage thousands of file descriptors concurrently without context-switching overhead.

To illustrate how an event-driven non-blocking socket listener functions at the systems programming level, here is a minimal demonstration written in C:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <fcntl.h>

#define MAX_EVENTS 64
#define BUFFER_SIZE 4096

static int set_nonblocking(int fd) {
  int flags = fcntl(fd, F_GETFL, 0);
  if (flags == -1) {
    return -1;
  }
  return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

void handle_client_traffic(int client_fd) {
  // Avoid heap allocations inside hot loops: use a fixed primitive buffer
  char raw_buffer[BUFFER_SIZE];
  ssize_t bytes_read = read(client_fd, raw_buffer, sizeof(raw_buffer) - 1);
  
  if (bytes_read > 0) {
    raw_buffer[bytes_read] = '\0';
    if (strncmp(raw_buffer, "GET", 3) == 0) {
      const char response[] = "HTTP/1.1 200 OK\r\nContent-Length: 13\r\n\r\nHomelab Alive";
      write(client_fd, response, sizeof(response) - 1);
    }
  }
  close(client_fd);
}
```

Because Nginx keeps raw memory allocations minimal and monitors all sockets through an `epoll_wait()` event loop, its RAM footprint remains tiny (often under 20MB in idle Docker containers).

---

## 3. Docker Internal Networking & DNS Resolution (`127.0.0.11`)

When you run Docker containers, Docker automatically creates isolated network namespaces. By default, containers are assigned to a default `bridge` network, but default bridge networks **do not** support automatic DNS resolution by container name.

To make Nginx route traffic seamlessly, we create a user-defined custom bridge network:

```bash
docker network create homelab_network
```

Inside user-defined bridge networks, Docker runs an embedded DNS server at `127.0.0.11`. 

When Nginx sees:
```nginx
proxy_pass http://jellyfin_service:8096;
```
It queries `127.0.0.11:53`, which resolves `jellyfin_service` to its current internal IP address (e.g., `172.20.0.4`). This means you **never** need to hardcode dynamic private IP addresses or expose host ports on your backend services.

---

## 4. Building the Modular Nginx Configuration

Instead of cramming 500 lines of messy configuration into one single `nginx.conf`, we structure our setup cleanly with reusable snippet files.

### Directory Structure

```text
/opt/homelab/nginx/
├── docker-compose.yml
├── nginx.conf
├── snippets/
│   ├── proxy_params.conf
│   ├── ssl_params.conf
│   └── security_headers.conf
└── conf.d/
    ├── vaultwarden.conf
    ├── jellyfin.conf
    └── nextcloud.conf
```

### The Master `nginx.conf`

```nginx
user nginx;
worker_processes auto;
pid /var/run/nginx.pid;
worker_rlimit_nofile 65535;

events {
  worker_connections 4096;
  use epoll;
  multi_accept on;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  # Performance Tuning
  sendfile on;
  tcp_nopush on;
  tcp_nodelay on;
  keepalive_timeout 65;
  types_hash_max_size 2048;
  server_tokens off;

  # Buffer Sizes for Reverse Proxying
  client_body_buffer_size 128k;
  client_max_body_size 500M;
  proxy_buffer_size 128k;
  proxy_buffers 4 256k;
  proxy_busy_buffers_size 256k;

  # Logging format with upstream timing
  log_format homelab_combined '$remote_addr - $remote_user [$time_local] '
                              '"$request" $status $body_bytes_sent '
                              '"$http_referer" "$http_user_agent" '
                              'rt=$request_time uct="$upstream_connect_time" '
                              'uht="$upstream_header_time" urt="$upstream_response_time"';

  access_log /var/log/nginx/access.log homelab_combined;
  error_log /var/log/nginx/error.log warn;

  # Gzip Compression
  gzip on;
  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 6;
  gzip_types text/plain text/css text/xml application/json application/javascript application/xml+rss;

  # Dynamic Docker DNS Resolver
  resolver 127.0.0.11 valid=30s ipv6=off;

  # Map for WebSocket Connection upgrades
  map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
  }

  include /etc/nginx/conf.d/*.conf;
}
```

### Reusable Snippet: `snippets/proxy_params.conf`

When proxying, backend containers need to know who the real client is, what protocol was used, and what host was requested:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Port $server_port;

# HTTP 1.1 support and WebSocket headers
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;

# Timeouts
proxy_connect_timeout 60s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;
```

### Reusable Snippet: `snippets/security_headers.conf`

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "no-referrer-when-downgrade" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
```

---

## 5. Configuring Virtual Hosts for Specific Services

### Example 1: Jellyfin Media Server (Streaming & WebSockets)

Jellyfin needs custom buffering settings so high-bitrate 4K streams don't stutter, plus active WebSocket proxying for live sync.

Create `/opt/homelab/nginx/conf.d/jellyfin.conf`:

```nginx
server {
  listen 80;
  listen [::]:80;
  server_name media.mydomain.dev;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name media.mydomain.dev;

  ssl_certificate /etc/letsencrypt/live/mydomain.dev/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/mydomain.dev/privkey.pem;
  include /etc/nginx/snippets/security_headers.conf;

  client_max_body_size 20M;

  location / {
    include /etc/nginx/snippets/proxy_params.conf;
    
    # Direct upstream proxy pass to container name
    proxy_pass http://jellyfin:8096;
    
    # Disable buffering for smooth video streaming
    proxy_buffering off;
  }

  location /socket {
    include /etc/nginx/snippets/proxy_params.conf;
    proxy_pass http://jellyfin:8096;
  }
}
```

### Example 2: Vaultwarden (Bitwarden in Rust - WebSockets & Crypto Sync)

Create `/opt/homelab/nginx/conf.d/vaultwarden.conf`:

```nginx
server {
  listen 80;
  server_name vault.mydomain.dev;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl http2;
  server_name vault.mydomain.dev;

  ssl_certificate /etc/letsencrypt/live/mydomain.dev/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/mydomain.dev/privkey.pem;
  include /etc/nginx/snippets/security_headers.conf;

  client_max_body_size 128M;

  location / {
    include /etc/nginx/snippets/proxy_params.conf;
    proxy_pass http://vaultwarden:80;
  }

  # Vaultwarden notifications WebSocket
  location /notifications/hub {
    include /etc/nginx/snippets/proxy_params.conf;
    proxy_pass http://vaultwarden:3012;
  }

  location /notifications/hub/negotiate {
    include /etc/nginx/snippets/proxy_params.conf;
    proxy_pass http://vaultwarden:80;
  }
}
```

---

## 6. Rate Limiting Mathematics: Protecting the Homelab

When you open services to the internet, bots will immediately start brute-forcing login endpoints. Nginx handles rate limiting via the **Leaky Bucket (or Token Bucket)** algorithm.

We configure rate limits inside the `http` context:

```nginx
limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/s;
```

Mathematically, the token bucket enforces an average arrival rate $r$ and allows burst capacity $B$. 

If a client sends $N$ requests at time $t = 0$, the initial burst consumes available tokens. If requests exceed $B$, the excess requests are either delayed or rejected immediately with HTTP 429.

The delay calculation $\Delta t$ for request $k$ inside the burst window can be expressed as:

$$\Delta t = \max\left(0, \frac{k - B}{r}\right)$$

In terms of queueing theory, if the incoming request rate $\lambda$ exceeds the processing rate $\mu$, the probability $P_{\text{reject}}$ of a request being dropped when the queue capacity $Q = B$ fills up is:

$$P_{\text{reject}} = \frac{(1 - \rho)\rho^B}{1 - \rho^{B+1}} \quad \text{where } \rho = \frac{\lambda}{\mu}$$

To apply this to an authentication endpoint:

```nginx
location /api/v1/auth {
  limit_req zone=login_limit burst=10 nodelay;
  include /etc/nginx/snippets/proxy_params.conf;
  proxy_pass http://auth_service:8080;
}
```

---

## 7. The Full `docker-compose.yml` Stack

Here is the complete unified Docker Compose stack that links Nginx, Certbot (for automatic SSL), Vaultwarden, and Jellyfin over an isolated bridge network:

```yaml
version: '3.8'

networks:
  homelab_net:
    name: homelab_network
    driver: bridge

services:
  nginx:
    image: nginx:alpine
    container_name: reverse_proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /opt/homelab/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - /opt/homelab/nginx/conf.d:/etc/nginx/conf.d:ro
      - /opt/homelab/nginx/snippets:/etc/nginx/snippets:ro
      - /opt/homelab/certbot/conf:/etc/letsencrypt:ro
      - /opt/homelab/certbot/www:/var/www/certbot:ro
    networks:
      - homelab_net
    depends_on:
      - vaultwarden
      - jellyfin

  certbot:
    image: certbot/certbot:latest
    container_name: certbot
    volumes:
      - /opt/homelab/certbot/conf:/etc/letsencrypt:rw
      - /opt/homelab/certbot/www:/var/www/certbot:rw
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12d & wait $${!}; done;'"

  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - WEBSOCKET_ENABLED=true
    volumes:
      - /opt/homelab/data/vaultwarden:/data
    networks:
      - homelab_net

  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    volumes:
      - /opt/homelab/data/jellyfin/config:/config
      - /opt/homelab/data/jellyfin/cache:/cache
      - /mnt/storage/media:/media
    networks:
      - homelab_net
```

---

## 8. Common Gotchas That Cost Me Hours of Sleep

### 1. The 502 Bad Gateway Loop of Death
If Nginx starts *before* a backend container is fully initialized, or if a backend container restarts and changes its internal Docker IP, Nginx can get stuck caching the old IP address. 

**Fix:** Use a variable in `proxy_pass` combined with Docker's embedded DNS resolver:

```nginx
location / {
  resolver 127.0.0.11 valid=10s;
  set $upstream_app http://my_container:3000;
  proxy_pass $upstream_app;
}
```

### 2. The Cloudflare Real-IP Blindspot
If you route your domain through Cloudflare's proxy (the orange cloud icon), every single visitor's IP address will appear in your logs as Cloudflare's IP range (`173.245.48.0/20`, etc.).

**Fix:** Add `set_real_ip_from` directives for Cloudflare's IP blocks inside `/etc/nginx/conf.d/cloudflare_real_ip.conf`:

```nginx
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 103.21.244.0/22;
set_real_ip_from 103.22.200.0/22;
set_real_ip_from 103.31.4.0/22;
set_real_ip_from 141.101.64.0/18;
set_real_ip_from 108.162.192.0/18;
set_real_ip_from 190.93.240.0/20;
set_real_ip_from 188.114.96.0/20;
set_real_ip_from 197.234.240.0/22;
set_real_ip_from 198.41.128.0/17;
set_real_ip_from 162.158.0.0/15;
set_real_ip_from 104.16.0.0/13;
set_real_ip_from 104.24.0.0/14;
set_real_ip_from 172.64.0.0/13;
set_real_ip_from 131.0.72.0/22;

real_ip_header CF-Connecting-IP;
```

### 3. File Uploads Failing with HTTP 413
By default, Nginx has a tiny `client_max_body_size` limit of **1MB**. If you try to upload a video to Nextcloud or sync a database backup, Nginx drops the connection immediately.

Always explicitly set:
```nginx
client_max_body_size 512M;
```

---

## 9. Wrapping Up

Setting up a proper Nginx reverse proxy transformed my homelab from an insecure mesh of open ports into a clean, unified, enterprise-grade development environment. 

You get:
1. **Centralized TLS Termination:** Automatic certificate renewals with zero downtime.
2. **Zero Open Ports on Services:** Only ports `80` and `443` on Nginx touch the outside world.
3. **Internal Container Isolation:** All communication stays within encrypted TLS sessions externally and high-speed Docker virtual bridge networks internally.

If you are still typing IP addresses and port numbers into your browser, stop what you are doing, spin up this compose file, and take control of your local infrastructure.
