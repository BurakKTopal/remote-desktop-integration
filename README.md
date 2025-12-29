
Full setup to securely code on your Windows 10 Home machine from an Android phone without exposing RDP to the internet.

---

## 1. Prerequisites

- Windows 10 with WSL2 installed and a Linux distribution (Ubuntu recommended)
- Android phone
- Tailscale account (for secure private network)
- GitHub account (for OAuth authentication)
- Administrative access on Windows / WSL

---

## 2. Install Tailscale

### On Windows

Download and install Tailscale: https://tailscale.com/download

Sign in with your account.

### On WSL (optional)
```bash
sudo snap install tailscale
sudo tailscale up
```

### On Android

Install Tailscale from Play Store.

Sign in with the same account.

Make sure your phone and WSL machine can ping each other's Tailscale IP:
```bash
ping 
```

---

## 3. Install code-server in WSL
```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Install code-server
curl -fsSL https://code-server.dev/install.sh | sh

# Run code-server (bind to localhost)
code-server --bind-addr 127.0.0.1:8080
```

Test locally:
```bash
curl http://127.0.0.1:8080
```

You should see `HTTP server listening on http://127.0.0.1:8080/`

> **Note:** code-server does not serve HTTPS directly — Nginx will handle HTTPS.

---

## 4. Install OAuth2 Proxy

### Download binary
```bash
curl -L -o /usr/local/bin/oauth2-proxy https://github.com/oauth2-proxy/oauth2-proxy/releases/latest/download/oauth2-proxy-linux-amd64
sudo chmod +x /usr/local/bin/oauth2-proxy
```

### Create config file
```bash
sudo nano /etc/oauth2-proxy.cfg
```

Example config:
```ini
provider = "github"
client_id = "YOUR_GITHUB_CLIENT_ID"
client_secret = "YOUR_GITHUB_CLIENT_SECRET"

http_address = "127.0.0.1:4180"
cookie_secret = "GENERATE_A_VALID_32_BYTE_SECRET"

upstreams = [ "http://127.0.0.1:8080" ]
redirect_url = "https:///oauth2/callback"
email_domains = [ "*" ]
reverse_proxy = true
skip_provider_button = true
cookie_secure = true
cookie_expire = "8h"
cookie_refresh = "1h"
```

Replace `<TAILSCALE-IP>` with your WSL/Tailscale IP.

Generate a valid 32-byte `cookie_secret`:
```bash
head -c 32 /dev/urandom | base64 | tr -d '\n'
```

### Run OAuth2 Proxy
```bash
sudo oauth2-proxy --config /etc/oauth2-proxy.cfg
```

Test:
```bash
curl http://127.0.0.1:4180
```

Should return HTML of GitHub login page.

---

## 5. Install and configure Nginx
```bash
sudo apt install nginx -y
```

### Create Nginx site
```bash
sudo nano /etc/nginx/sites-available/code-server
```

Example configuration:
```nginx
server {
    listen 443 ssl;
    server_name <TAILSCALE-IP>;  # Tailscale IP of your WSL

    ssl_certificate     /etc/ssl/localca/server.crt;
    ssl_certificate_key /etc/ssl/localca/server.key;

    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:4180;  # OAuth2 Proxy
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Real-IP $remote_addr;

	# WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400s;
    }
}
```

### Enable site
```bash
sudo ln -s /etc/nginx/sites-available/code-server /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 6. Create self-signed SSL certificates (for local HTTPS)
```bash
sudo mkdir -p /etc/ssl/localca
cd /etc/ssl/localca

# Create local CA
sudo openssl genrsa -out ca.key 2048
sudo openssl req -x509 -new -nodes -key ca.key -days 3650 -out ca.crt -subj "/C=BE/O=YourName/CN=LocalCA"

# Create server key and CSR
sudo openssl genrsa -out server.key 2048
sudo openssl req -new -key server.key -out server.csr -subj "/C=BE/O=YourName/CN="

# Sign server cert with local CA
sudo openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 825 -sha256
```

Use `/etc/ssl/localca/server.crt` and `server.key` in Nginx config.

---

## 7. Configure GitHub OAuth App

**Homepage URL:**
```
https://<TAILSCALE-IP>/
```

**Authorization callback URL:**
```
https://<TAILSCALE-IP>/oauth2/callback
```

Must match `redirect_url` in `oauth2-proxy.cfg`.

---

## 8. Start all services

### Step 1 — code-server
```bash
code-server --bind-addr 127.0.0.1:8080
```

### Step 2 — OAuth2 Proxy
```bash
sudo oauth2-proxy --config /etc/oauth2-proxy.cfg
```

### Step 3 — Nginx
```bash
sudo systemctl restart nginx
```

---

## 9. Connect from Android

1. Make sure Tailscale is active on your phone.
2. Open a browser and go:
```
https://<TAILSCALE-IP>/
```

3. Log in via GitHub (OAuth2 Proxy).
4. After login, you will see code-server IDE.

---

## 10. Security Notes

- code-server is only HTTP on localhost; HTTPS is handled by Nginx.
- OAuth2 Proxy ensures 2FA/GitHub login protection.
- All traffic is encrypted over Tailscale VPN.
- **Do not expose code-server directly to the internet.**

---

## Optional Enhancements

- Create systemd services for code-server and OAuth2 Proxy to auto-start.
- Use a domain name instead of Tailscale IP for cleaner URLs.
- Configure persistent `cookie_secret` and secure it:
```bash
sudo chmod 600 /etc/oauth2-proxy.cfg
```

- Enable HTTP → HTTPS redirect in Nginx (add to /etc/nginx/sites-available/code-server):
```nginx
server {
    listen 80;
    server_name default_name; # So that settings as server setting above 
    return 301 https://$host$request_uri;
}
```
