# Matrix-Livekit-Selfhosted
Complete guide to self-hosted Matrix + LiveKit voice/video calling — including the undocumented fixes that cause most deployments to fail" This is for Element X and all the errors from IPV6 to Open_ID_Errors handshake with JWT process.

======================================================================================================================================================================================================================================

# Self-Hosted Matrix + LiveKit Voice/Video Calling

A complete guide to setting up a **fully self-hosted Matrix homeserver (Synapse) with LiveKit-powered voice and video calling** using Element X. This guide covers everything from initial setup to troubleshooting the issues that cause most people to give up.

> **Note:** This guide was battle-tested over a real deployment. Most existing guides skip critical details around OpenID authentication, TLS configuration, IPv6, CORS, and Docker networking that cause calls to silently fail. All the problem I ran through and their solutions until a working Element X web, mobile to work.  One final note, not in the docuemntaiton, if an Android and Apple phone use Element X together, and user has sessions not verified, the emoji might not work until all sessions are either verified or signed out.  

By Mar Roble
---

## What You'll Build

| Component | Purpose |
|-----------|---------|
| **Synapse** | Matrix homeserver |
| **Caddy** | Reverse proxy with automatic TLS |
| **LiveKit** | WebRTC media server (audio/video) |
| **lk-jwt-service** | JWT auth bridge between Matrix and LiveKit |
| **Element Web** | Web-based Matrix client |
| **ddclient** | Dynamic DNS (for home servers with dynamic IPs) |

### Subdomains Required

| Subdomain | Purpose |
|-----------|---------|
| `domain.com` | Matrix homeserver |
| `element.domain.com` | Element Web client |
| `rtc.domain.com` | JWT authentication service |
| `livekit.domain.com` | LiveKit media server |

---

## Prerequisites

- A Linux server (this guide uses Debian/Ubuntu — tested on Proxmox LXC)
- A domain name with DNS control
- Docker and Docker Compose installed
- Ports 80, 443, 8448, 7881 TCP and 50000-50100 UDP open/forwarded
- Basic Linux command line knowledge

---

## Table of Contents

1. [DNS Setup](#1-dns-setup)
2. [Install Synapse](#2-install-synapse)
3. [Install Caddy](#3-install-caddy)
4. [Configure Files](#4-configure-files)
5. [Docker Compose Setup](#5-docker-compose-setup)
6. [Start Services](#6-start-services)
7. [Dynamic DNS (Optional)](#7-dynamic-dns-optional)
8. [Verify Everything Works](#8-verify-everything-works)
9. [Common Issues & Fixes](#9-common-issues--fixes)
10. [Firewall Ports Reference](#10-firewall-ports-reference)

---

## 1. DNS Setup

Create these A records pointing to your server's public IP:

```
domain.com          A    YOUR_PUBLIC_IP
element.domain.com  A    YOUR_PUBLIC_IP
rtc.domain.com      A    YOUR_PUBLIC_IP
livekit.domain.com  A    YOUR_PUBLIC_IP
```

Wait for DNS to propagate before continuing (use `dig domain.com A` to verify).

---

## 2. Install Synapse

```bash
# Add Matrix repository
apt install -y lsb-release wget apt-transport-https
wget -O /usr/share/keyrings/matrix-org-archive-keyring.gpg \
  https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/matrix-org-archive-keyring.gpg] \
  https://packages.matrix.org/debian/ $(lsb_release -cs) main" \
  > /etc/apt/sources.list.d/matrix-org.list

# Install
apt update
apt install matrix-synapse-py3 -y
```

---

## 3. Install Caddy

```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | tee /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install caddy -y
```

---

## 4. Configure Files

### 4a. Synapse — `/etc/matrix-synapse/homeserver.yaml`

> **Critical settings highlighted with comments**

```yaml
report_stats: false
pid_file: "/var/run/matrix-synapse.pid"

# CRITICAL: TLS certs for port 8448 (federation/OpenID)
# Copy these from Caddy after first Caddy start
tls_certificate_path: /etc/matrix-synapse/tls/domain.com.crt
tls_private_key_path: /etc/matrix-synapse/tls/domain.com.key

listeners:
  - bind_addresses:
      - "0.0.0.0"
    port: 8008
    resources:
      - names:
          - client
          - federation
        compress: false
    tls: false
    type: http
    x_forwarded: true

  # CRITICAL: Must be 0.0.0.0 (not 127.0.0.1) and tls: true
  # The lk-jwt-service uses matrix:// scheme which requires HTTPS on 8448
  - bind_addresses:
      - "0.0.0.0"
    port: 8448
    type: http
    tls: true
    tls_certificate_file: /etc/matrix-synapse/tls/domain.com.crt
    tls_private_key_file: /etc/matrix-synapse/tls/domain.com.key
    resources:
      - names: [federation]

media_store_path: "/var/lib/matrix-synapse/media_store"
database:
  name: sqlite3
  args:
    database: /var/lib/matrix-synapse/homeserver.db
log_config: "/etc/matrix-synapse/log.yaml"
signing_key_path: "/etc/matrix-synapse/homeserver.signing.key"
trusted_key_servers:
  - server_name: "matrix.org"
serve_server_wellknown: true
public_baseurl: "https://domain.com"
enable_registration_without_verification: true
registration_shared_secret: "YOUR_REGISTRATION_SECRET"
password_config:
  enabled: true
  localdb_enabled: true

experimental_features:
  msc3266_enabled: true
  msc3575_enabled: true
  msc4222_enabled: true
  msc4143_enabled: true
  msc2965_enabled: true
  msc4140_enabled: true

max_event_delay_duration: 24h

# Keep these if using coturn for classic Element calls
turn_uris:
  - "turn:domain.com:3478?transport=udp"
  - "turn:domain.com:3478?transport=tcp"
turn_shared_secret: "YOUR_TURN_SECRET"
turn_user_lifetime: 86400000
turn_allow_guests: true

# LiveKit MatrixRTC transport
matrix_rtc:
  livekit:
    livekit_service_url: "https://rtc.domain.com"

media_retention:
  local_media_lifetime: "60d"
  remote_media_lifetime: "30d"

allow_public_openid_requests: true
```

---

### 4b. Caddyfile — `/etc/caddy/Caddyfile`

> **Critical: rtc.domain.com must NOT add CORS headers — the JWT service handles its own**

```caddy
# 1. Main Matrix homeserver
domain.com {
    handle /.well-known/matrix/client {
        header Access-Control-Allow-Origin "*"
        header Content-Type "application/json"
        respond `{"m.homeserver":{"base_url":"https://domain.com"},"org.matrix.msc4143.rtc_foci":[{"type":"livekit","livekit_service_url":"https://rtc.domain.com"}]}` 200
    }

    handle /.well-known/matrix/server {
        root * /var/www/matrix
        file_server
        header Access-Control-Allow-Origin "*"
        header Content-Type "application/json"
    }

    handle /_matrix/client/unstable/org.matrix.msc4143/rtc/transports {
        header Access-Control-Allow-Origin "*"
        header Content-Type "application/json"
        respond `{"rtc_transports":[{"type":"livekit","livekit_service_url":"https://rtc.domain.com"}]}` 200
    }

    handle /_matrix/federation/* {
        reverse_proxy 127.0.0.1:8008
    }

    reverse_proxy /* 127.0.0.1:8008
}

# 2. LiveKit media server
# IMPORTANT: Use h1 transport for WebSocket support
livekit.domain.com {
    reverse_proxy 127.0.0.1:7880 {
        transport http {
            versions h1
        }
        header_up Host {host}
        header_up X-Real-IP {remote_host}
        header_up Connection {http.request.header.Connection}
        header_up Upgrade {http.request.header.Upgrade}
    }
}

# 3. JWT auth service
# CRITICAL: Do NOT add CORS headers here!
# The lk-jwt-service sends its own CORS headers.
# Adding them in Caddy creates duplicate headers which browsers reject.
rtc.domain.com {
    reverse_proxy 127.0.0.1:8082
}

# 4. Element Web client
element.domain.com {
    root * /var/www/element
    file_server
}
```

---

### 4c. Well-known — `/var/www/matrix/client`

```json
{
  "m.homeserver": {
    "base_url": "https://domain.com"
  },
  "org.matrix.msc4143.rtc_foci": [
    {
      "type": "livekit",
      "livekit_service_url": "https://rtc.domain.com"
    }
  ]
}
```

Create the directory first:
```bash
mkdir -p /var/www/matrix
```

---

### 4d. LiveKit — `/root/matrix-rtc/livekit.yaml`

```yaml
port: 7880
rtc:
  tcp_port: 7881
  port_range_start: 50000
  port_range_end: 50100
  use_external_ip: true    # Auto-detect public IP via STUN
  stun_servers:
    - "stun.l.google.com:19302"    # Note: no stun: prefix
keys:
  your_api_key: "your_api_secret"
```

> **Common mistake:** Using `"stun:stun.l.google.com:19302"` (with `stun:` prefix) will cause LiveKit to crash with "too many colons" error. Use `"stun.l.google.com:19302"` instead.

---

### 4e. Docker Compose — `/root/matrix-rtc/docker-compose.yml`

```yaml
services:
  livekit:
    image: livekit/livekit-server:latest
    command: --config /etc/livekit.yaml
    restart: unless-stopped
    network_mode: "host"
    volumes:
      - ./livekit.yaml:/etc/livekit.yaml:ro

  auth-service:
    image: ghcr.io/element-hq/lk-jwt-service:latest
    container_name: element-call-jwt
    restart: unless-stopped
    network_mode: "host"
    environment:
      - LOG_LEVEL=debug
      - LIVEKIT_URL=wss://livekit.domain.com
      - LIVEKIT_KEY=your_api_key
      - LIVEKIT_SECRET=your_api_secret
      - LIVEKIT_FULL_ACCESS_HOMESERVERS=domain.com
      - HOMESERVER_URL=http://127.0.0.1:8008
      - LIVEKIT_JWT_BIND=:8082
```

> **Critical notes:**
> - Both services must use `network_mode: "host"`
> - `LIVEKIT_URL` must be `wss://livekit.domain.com` (not `ws://127.0.0.1:7880`)
> - `LIVEKIT_JWT_BIND` must not conflict with other services on your host

---

### 4f. Element Web — `/var/www/element/config.json`

Download Element Web first:
```bash
cd /var/www
wget https://github.com/element-hq/element-web/releases/download/v1.11.99/element-v1.11.99.tar.gz
tar -xzf element-v1.11.99.tar.gz
mv element-v1.11.99 element
cp /var/www/element/config.sample.json /var/www/element/config.json
```

Edit `config.json` and set:
```json
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://domain.com",
            "server_name": "domain.com"
        }
    }
}
```

---

## 5. Docker Compose Setup

```bash
mkdir -p /root/matrix-rtc
cd /root/matrix-rtc
# Create livekit.yaml and docker-compose.yml as shown above
```

---

## 6. Start Services

**Always start in this order:**

```bash
# Step 1 - Copy TLS certs from Caddy to Synapse
# (Run this AFTER Caddy has obtained certificates)
mkdir -p /etc/matrix-synapse/tls
cp /var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/domain.com/domain.com.crt \
   /etc/matrix-synapse/tls/
cp /var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/domain.com/domain.com.key \
   /etc/matrix-synapse/tls/
chown -R matrix-synapse:matrix-synapse /etc/matrix-synapse/tls/
chmod 640 /etc/matrix-synapse/tls/*

# Step 2 - Start Synapse
systemctl restart matrix-synapse
systemctl status matrix-synapse | grep Active

# Step 3 - Start Caddy
systemctl restart caddy
systemctl status caddy | grep Active

# Step 4 - Start Docker services
cd /root/matrix-rtc
docker compose up -d
docker ps
```

**Create your first user:**
```bash
register_new_matrix_user -c /etc/matrix-synapse/homeserver.yaml \
  -u yourusername -p yourpassword -a http://localhost:8008
```

---

## 7. Dynamic DNS (Optional)

If your server has a dynamic public IP (home server), install ddclient:

```bash
apt install ddclient -y
nano /etc/ddclient.conf
```

For Namecheap:
```
daemon=300
syslog=yes
pid=/var/run/ddclient.pid
ssl=yes
use=web, web=checkip.dyndns.com
protocol=namecheap
server=dynamicdns.park-your-domain.com
login=domain.com
password=YOUR_NAMECHEAP_DDNS_PASSWORD

@
element
rtc
livekit
```

```bash
systemctl enable ddclient
systemctl start ddclient

# Test
ddclient -daemon=0 -debug -verbose -noquiet
```

> Get your Namecheap or other registrars/DNS providers DDNS password from: Mamecheap was used for this for DNS provider > site → Domain List → Manage → Advanced DNS → Dynamic DNS

---

## 8. Verify Everything Works

### Full auth chain test:
```bash
# Get access token
ACCESS=$(curl -s -X POST http://127.0.0.1:8008/_matrix/client/v3/login \
  -H 'Content-Type: application/json' \
  -d '{"type":"m.login.password","user":"@yourusername:domain.com","password":"yourpassword"}' \
  | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)

# Get OpenID token
OPENID=$(curl -s -X POST \
  "http://127.0.0.1:8008/_matrix/client/v3/user/@yourusername:domain.com/openid/request_token" \
  -H "Authorization: Bearer $ACCESS" \
  -H "Content-Type: application/json" \
  -d '{}' | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)

# Test JWT service - should return LiveKit token in under 1 second
time curl -X POST http://127.0.0.1:8082/sfu/get \
  -H "Content-Type: application/json" \
  -d "{\"room\":\"test\",\"openid_token\":{\"access_token\":\"$OPENID\",\"token_type\":\"Bearer\",\"matrix_server_name\":\"domain.com\"}}"
```

**Successful response looks like:**
```json
{"url":"wss://livekit.domain.com","jwt":"eyJhbGci..."}
```

### Individual service checks:
```bash
# Well-known
curl https://domain.com/.well-known/matrix/client

# Synapse health
curl http://127.0.0.1:8008/_matrix/client/versions

# Federation/OpenID endpoint
curl -k "https://domain.com:8448/_matrix/federation/v1/openid/userinfo?access_token=test"
# Should return M_MISSING_TOKEN (not connection refused)

# LiveKit WebSocket
curl -i --http1.1 \
  -H "Upgrade: websocket" -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://livekit.domain.com/rtc?access_token=test
# Should return 401 (not connection refused)
```

---

## 9. Common Issues & Fixes

### ❌ OpenID Error on calls

**Symptom:** `OPEN_ID_ERROR` in Element X, JWT logs show `context deadline exceeded`

**Root causes and fixes:**

**1. IPv6 causing 30-second timeouts**
```bash
# Check if IPv6 is causing slow DNS resolution
cat > /etc/sysctl.d/99-disable-ipv6.conf << 'EOF'
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
sysctl -p /etc/sysctl.d/99-disable-ipv6.conf
```

**2. Synapse port 8448 not using TLS**

The `lk-jwt-service` uses `matrix://` scheme which always connects via HTTPS to port 8448. Synapse must have TLS enabled on 8448 — see homeserver.yaml config above.

**3. Stuck TCP connections**
```bash
# Check for zombie connections
ss -tnp | grep 8448
# If many ESTABLISHED connections from lk-jwt-service:
cd /root/matrix-rtc && docker compose down && docker compose up -d
```

**4. NAT hairpin issue**

Add to `/etc/hosts` so JWT service resolves domain internally:
```
127.0.0.1 domain.com
```

---

### ❌ CORS Error — duplicate Access-Control-Allow-Origin headers

**Symptom:** Browser console shows `'Access-Control-Allow-Origin' header contains multiple values '*, *'`

**Fix:** Remove ALL CORS headers from the `rtc.domain.com` Caddy block. The JWT service sends its own correct headers. Caddy adding them again creates duplicates that browsers reject.

```caddy
# CORRECT - no headers:
rtc.domain.com {
    reverse_proxy 127.0.0.1:8082
}

# WRONG - causes duplicate CORS headers:
rtc.domain.com {
    header Access-Control-Allow-Origin "*"   # DELETE THIS
    reverse_proxy 127.0.0.1:8082
}
```

---

### ❌ LiveKit crashes on startup

**Symptom:** `could not resolve external IP: too many colons in address`

**Fix:** Remove `stun:` prefix from STUN server URL in `livekit.yaml`:
```yaml
# CORRECT:
stun_servers:
  - "stun.l.google.com:19302"

# WRONG:
stun_servers:
  - "stun:stun.l.google.com:19302"
```

---

### ❌ No audio — participants can't hear each other

**Symptom:** Call connects, JWT tokens issued, but no audio

**Check 1:** Verify `LIVEKIT_URL` in docker-compose.yml uses public domain:
```bash
docker inspect element-call-jwt | grep LIVEKIT_URL
# Must show: wss://livekit.domain.com
# NOT: ws://127.0.0.1:7880
```

**Check 2:** Verify LiveKit WebSocket works:
```bash
curl -i --http1.1 \
  -H "Upgrade: websocket" -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://livekit.domain.com/rtc?access_token=test
# Must return 401, not 200 or connection refused
```

**Check 3:** UDP ports open in firewall (50000-50100)

---

### ❌ Port 8080 already in use for JWT service

**Symptom:** `listen tcp :8080: bind: address already in use`

**Fix:** Bind the JWT service to a different port using `LIVEKIT_JWT_BIND`:
```yaml
environment:
  - LIVEKIT_JWT_BIND=:8082    # Use any free port
```
Update Caddyfile `rtc.domain.com` to proxy to the same port.

---

### ❌ Caddy can't get TLS certificate

**Symptom:** `Timeout during connect (likely firewall problem)` in Caddy logs

**Fixes:**
- Ensure port 80 TCP is open and forwarded to your server
- Ensure DNS A records are pointing to correct IP
- Check for Let's Encrypt rate limits (max 5 failures per hour per domain)
- Wait for DNS propagation before starting Caddy

---

### ❌ "Cannot find own membership event"

**Symptom:** Call connects but shows this error, no audio

**Fix:** Add `msc4140_enabled: true` to `experimental_features` in homeserver.yaml and restart Synapse.

---

### ❌ Everything breaks after reboot / IP change

1. Check current public IP: `curl ifconfig.me`
2. Update DNS records if IP changed
3. Restart in order: Docker down → Synapse → Caddy → Docker up
4. Consider setting up ddclient (see Section 7)

---

## 10. Firewall Ports Reference

| Port | Protocol | Required | Purpose |
|------|----------|----------|---------|
| 80 | TCP | ✅ Yes | Caddy HTTP + Let's Encrypt ACME |
| 443 | TCP | ✅ Yes | Caddy HTTPS (all web traffic) |
| 8448 | TCP | ✅ Yes | Matrix federation + OpenID auth |
| 7881 | TCP | ✅ Yes | LiveKit ICE/TCP fallback |
| 50000-50100 | UDP | ✅ Yes | LiveKit WebRTC media |
| 3478 | UDP+TCP | Optional | coturn STUN/TURN |
| 5349 | UDP+TCP | Optional | coturn TURN-TLS |
| 49152-49999 | UDP | Optional | coturn relay range |

---

## Certificate Renewal

Caddy auto-renews its own certificates. Synapse needs the certs copied manually (or automate with a cron job):

```bash
# Add to crontab (runs monthly)
# crontab -e
0 3 1 * * cp /var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/domain.com/domain.com.crt /etc/matrix-synapse/tls/ && cp /var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/domain.com/domain.com.key /etc/matrix-synapse/tls/ && chown -R matrix-synapse:matrix-synapse /etc/matrix-synapse/tls/ && chmod 640 /etc/matrix-synapse/tls/* && systemctl restart matrix-synapse
```

---

## Acknowledgements

This guide was developed through extensive real-world debugging of a complete self-hosted Matrix + LiveKit deployment. The issues documented here — particularly the OpenID TLS requirement, IPv6 timeouts, CORS duplication, and STUN URL format — are the exact problems that cause most self-hosted deployments to fail silently.

Prior to the error of Open_ID_Error, these were the original closest helps to setup after Synapse tempalte automated deployment via Proxmox.
Read both of their documentation and it might futher help.  It was not enough for me for authentication, call error for audio and other issues resolved above.

https://www.cleveradmin.de/blog/2025/04/matrixrtc-element-call-backend-einrichten/

https://willlewis.co.uk/blog/posts/deploy-element-call-backend-with-synapse-and-docker-compose/

---

## Contributing

If you found additional issues or fixes not covered here, please open a PR or issue. The Matrix self-hosting community needs more complete documentation like this.

---

## License

MIT — feel free to use, modify and share.

