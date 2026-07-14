# Self-Hosting n8n on Oracle Cloud — Complete Guide

> A step-by-step guide covering every scenario: with or without a domain, with or without a PAYG upgrade.

---

## Who Is This Guide For?

Before you start, identify which scenario applies to you:

| Scenario | PAYG Upgrade | Domain | Path |
|---|---|---|---|
| A | ✅ Yes | ✅ Yes | Coolify + n8n + Custom Domain + HTTPS |
| B | ✅ Yes | ❌ No | Coolify + n8n + sslip.io URL |
| C | ❌ No | ❌ No | Direct Docker + n8n + ngrok or sslip.io for webhooks |
| D | ❌ No | ✅ Yes | Direct Docker + n8n + Caddy + Custom Domain + HTTPS |

> **Important:** People without a PAYG upgrade cannot access the ARM A1.Flex shape. They are limited to the VM.Standard.E2.1.Micro (1 OCPU, 1GB RAM) which is too small for Coolify. They must install n8n directly via Docker. However, if they have a domain (Scenario D), they can still get a professional HTTPS URL using Caddy as a reverse proxy — completely free.

---

## Part 1 — Create Your Oracle Cloud Account (All Scenarios)

### Step 1 — Sign Up

Go to https://signup.cloud.oracle.com and fill in your details:

- Country: your location
- Account type: Individual
- Home Region: choose the closest region to you (cannot be changed later)
- Card: real Mastercard or Visa (no virtual cards)

Oracle places a small verification hold (~€0.93) on your card during signup. This is refunded.

### Step 2 — Verify Your Email

Check your inbox and click the verification link Oracle sends you.

### Step 3 — Enable MFA

Oracle will prompt you to set up multi-factor authentication. Use Google Authenticator or Microsoft Authenticator — tick "Offline Mode or Use Another Authenticator App", scan the QR code with your authenticator app, then enter the 6-digit code to confirm.

---

## Part 2 — Upgrade to Pay As You Go (Scenarios A and B only)

> Skip this section if you are on Scenario C.

The PAYG upgrade is required to access the ARM A1.Flex shape. You will not be charged for Always Free resources. Oracle only charges if you manually create paid resources outside the free tier.

**What happens during upgrade:**
- Oracle places a temporary authorization hold of ~€100–125 on your card
- This hold is reversed immediately on Oracle's end (your bank may take a few days)
- Always Free resources (ARM VM) remain at €0/month forever

### Steps to Upgrade

1. Log in to https://cloud.oracle.com
2. Click the **"Upgrade"** link in the yellow Free Tier banner at the top
3. Go to **Upgrade and Manage Payment** → click **"Change Payment Method"** and add your card
4. Select **"Individual"** as account type
5. Under Tax Details, tick **"Tax information is not available"** and select **"Not registered"** as the reason
6. Tick the terms and conditions checkbox
7. Click **"Upgrade your account"**
8. Wait for the confirmation email (usually within a few hours)

---

## Part 3 — Create the VM Instance

### Scenario A & B — ARM A1.Flex (2 OCPU, 12GB RAM)

1. Log in → click **☰ Menu** → **Compute** → **Instances** → **Create Instance**
2. Give it a name (e.g. `n8n-server`)
3. Click **"Change Image"** → select **Ubuntu** → choose **"Canonical Ubuntu 22.04 Minimal aarch64"** → click **"Select image"**
4. Click **"Change Shape"** → select **Ampere** tab → choose **VM.Standard.A1.Flex**
   - Set OCPUs to **2**
   - Set Memory to **12 GB**
   - Click **"Select shape"**
5. Under Networking → select **"Create new virtual cloud network"** and **"Create new public subnet"**
6. Under SSH Keys → select **"Generate a key pair for me"** → click **"Download private key"** and **"Download public key"** — save both files safely
7. Click **"Next"** through Security and Storage (leave defaults)
8. Click **"Create"**

### Scenarios C and D — AMD Micro (1 OCPU, 1GB RAM)

Follow the same steps above but at the Shape selection step:
- Choose **"Specialty and previous generation"** tab
- Select **VM.Standard.E2.1.Micro** (marked Always Free eligible)
- Leave OCPUs and memory at default (1 OCPU, 1GB)

### Assign a Public IP Address (All Scenarios)

After the instance is running:

1. Click on the instance → **Networking** tab → click the VNIC name under **Attached VNICs**
2. Click **"IP administration"** tab
3. Click the **three dots (...)** next to the private IP → click **"Edit"**
4. Select **"Ephemeral public IP"** → click **"Update"**
5. Copy and save your Public IP address — you will need it throughout this guide

### Open Firewall Ports (Oracle Security List)

1. Go to **☰ Menu** → **Networking** → **Virtual Cloud Networks** → click your VCN
2. Click **"Security"** tab → click **"Default Security List"**
3. Click **"Add Ingress Rules"** and add these rules (one session, click "+ Another Ingress Rule" between each):

**Rule 1 — HTTP**
```
Source CIDR: 0.0.0.0/0
Protocol: TCP
Destination Port: 80
Description: HTTP
```

**Rule 2 — HTTPS**
```
Source CIDR: 0.0.0.0/0
Protocol: TCP
Destination Port: 443
Description: HTTPS
```

**Rule 3 — Coolify (Scenarios A & B only)**
```
Source CIDR: 0.0.0.0/0
Protocol: TCP
Destination Port: 8000
Description: Coolify
```

**Rule 4 — n8n**
```
Source CIDR: 0.0.0.0/0
Protocol: TCP
Destination Port: 5678
Description: n8n
```

4. Click **"Add Ingress Rules"** to save

---

## Part 4 — Connect via SSH (All Scenarios)

### Windows (PowerShell)

```powershell
ssh -i "$env:USERPROFILE\Downloads\ssh-key-YYYY-MM-DD.key" ubuntu@YOUR_SERVER_IP
```

Replace `YYYY-MM-DD` with your key file date and `YOUR_SERVER_IP` with your public IP.

Type `yes` when asked about the fingerprint.

### Mac / Linux (Terminal)

```bash
chmod 400 ~/Downloads/ssh-key-YYYY-MM-DD.key
ssh -i ~/Downloads/ssh-key-YYYY-MM-DD.key ubuntu@YOUR_SERVER_IP
```

You are connected when you see `ubuntu@your-server-name:~$`

---

## Part 5 — Server Setup (All Scenarios)

Run these commands after connecting via SSH:

### Update the server

```bash
sudo apt update && sudo apt upgrade -y
```

When asked which services to restart, type `12` (none of the above) and press Enter.

### Open Ubuntu firewall ports

```bash
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 8000 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 5678 -j ACCEPT
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

### Reboot the server

```bash
sudo reboot
```

Wait 1 minute, then reconnect via SSH using the same command as before.

---

## Part 6A — Install Coolify (Scenarios A and B)

> Skip to Part 6C if you are on Scenario C.

```bash
sudo curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

This takes 5–10 minutes. When complete you will see:

```
Your instance is ready to use!
You can access Coolify through your Public IPV4: http://YOUR_IP:8000
```

### Access Coolify

Open your browser and go to `http://YOUR_SERVER_IP:8000`

Register your admin account with your email and a strong password.

### Initial Setup

Click **"Let's go!"** → select **"This Machine"** → click **"Create My First Project"** → click **"Go to Dashboard"**

---

## Part 6B — Deploy n8n via Coolify (Scenarios A and B)

1. In Coolify dashboard → click **"Projects"** → click your project → click **"+ Add a new resource"**
2. Search for **n8n** → select **"N8N With Postgresql"**
3. Click **"Edit Compose File"** and replace the entire content with the following:

```yaml
services:
  n8n:
    image: 'n8nio/n8n:latest'
    environment:
      - SERVICE_URL_N8N_5678
      - 'N8N_EDITOR_BASE_URL=${SERVICE_URL_N8N}'
      - 'WEBHOOK_URL=${SERVICE_URL_N8N}'
      - 'N8N_HOST=${SERVICE_URL_N8N}'
      - 'N8N_PROTOCOL=${N8N_PROTOCOL:-https}'
      - 'GENERIC_TIMEZONE=${GENERIC_TIMEZONE:-Europe/Berlin}'
      - 'TZ=${TZ:-Europe/Berlin}'
      - N8N_SECURE_COOKIE=false
      - DB_TYPE=postgresdb
      - 'DB_POSTGRESDB_DATABASE=${POSTGRES_DB:-n8n}'
      - DB_POSTGRESDB_HOST=postgresql
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_USER=$SERVICE_USER_POSTGRES
      - DB_POSTGRESDB_SCHEMA=public
      - DB_POSTGRESDB_PASSWORD=$SERVICE_PASSWORD_POSTGRES
      - N8N_RUNNERS_ENABLED=true
      - N8N_RUNNERS_MODE=external
      - 'N8N_RUNNERS_BROKER_LISTEN_ADDRESS=${N8N_RUNNERS_BROKER_LISTEN_ADDRESS:-0.0.0.0}'
      - 'N8N_RUNNERS_BROKER_PORT=${N8N_RUNNERS_BROKER_PORT:-5679}'
      - N8N_RUNNERS_AUTH_TOKEN=$SERVICE_PASSWORD_N8N
      - 'N8N_RUNNERS_MAX_CONCURRENCY=${N8N_RUNNERS_MAX_CONCURRENCY:-5}'
      - 'N8N_PROXY_HOPS=${N8N_PROXY_HOPS:-1}'
    volumes:
      - 'n8n-data:/home/node/.n8n'
    depends_on:
      postgresql:
        condition: service_healthy
    healthcheck:
      test:
        - CMD-SHELL
        - 'wget -qO- http://127.0.0.1:5678/healthz'
      interval: 5s
      timeout: 20s
      retries: 10
  task-runners:
    image: 'n8nio/runners:latest'
    environment:
      - 'N8N_RUNNERS_TASK_BROKER_URI=${N8N_RUNNERS_TASK_BROKER_URI:-http://n8n:5679}'
      - N8N_RUNNERS_AUTH_TOKEN=$SERVICE_PASSWORD_N8N
      - 'N8N_RUNNERS_AUTO_SHUTDOWN_TIMEOUT=${N8N_RUNNERS_AUTO_SHUTDOWN_TIMEOUT:-15}'
      - 'N8N_RUNNERS_MAX_CONCURRENCY=${N8N_RUNNERS_MAX_CONCURRENCY:-5}'
    depends_on:
      - n8n
    healthcheck:
      test:
        - CMD-SHELL
        - 'wget -qO- http://127.0.0.1:5680/healthz'
      interval: 5s
      timeout: 20s
      retries: 10
  postgresql:
    image: 'postgres:16-alpine'
    volumes:
      - 'postgresql-data:/var/lib/postgresql/data'
    environment:
      - POSTGRES_USER=$SERVICE_USER_POSTGRES
      - POSTGRES_PASSWORD=$SERVICE_PASSWORD_POSTGRES
      - 'POSTGRES_DB=${POSTGRES_DB:-n8n}'
    healthcheck:
      test:
        - CMD-SHELL
        - 'pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}'
      interval: 5s
      timeout: 20s
      retries: 10
```

> Change `Europe/Berlin` to your own timezone if needed (e.g. `Africa/Lagos` for Nigeria).

4. Click **"Save"**

### Set Your Domain (Choose one)

**Scenario A — Custom Domain:**

In the N8N service Settings → Domains field, enter:
```
https://n8n.yourdomain.com
```

Then go to your domain registrar (Namecheap, etc.) and add an A record:
- Type: A
- Host: n8n
- Value: YOUR_SERVER_IP
- TTL: Automatic

**Scenario B — No Domain (use sslip.io):**

Leave the Domains field as the auto-generated sslip.io URL Coolify provides, e.g.:
```
http://n8n-xxxxx.YOUR_IP.sslip.io
```

This URL works immediately with no DNS setup required. Coolify generates it automatically.

5. Click **"Deploy"**

Wait 2–3 minutes. When status shows **"Running (healthy)"** your n8n is live.

### Enable HTTPS (Scenario A — Custom Domain)

After DNS propagates (check at https://www.whatsmydns.net):

1. Go to N8N service Settings → change domain from `http://` to `https://`
2. Click **"Save"** → click **"Restart"**
3. Coolify's Traefik proxy will automatically request a free Let's Encrypt SSL certificate

### Register Your n8n Account

Open your n8n URL in the browser → fill in your name, email and password → click **"Next"** → enter the free license key sent to your email in Settings → Usage and Plan.

---

## Part 6C — Direct Docker Install (Scenario C — No Upgrade, No Coolify)

> This is for users on the free AMD Micro VM (1GB RAM). Coolify is too heavy for this VM.

### Add Swap Memory (Critical — prevents crashes)

```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verify: `free -h` — should show Swap: 1.0G

### Install Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu
newgrp docker
```

### Create n8n Data Folder

```bash
sudo mkdir -p /home/ubuntu/.n8n
sudo chown -R 1000:1000 /home/ubuntu/.n8n
```

### Deploy n8n

```bash
sudo docker run -d \
  --name n8n \
  --restart always \
  -p 5678:5678 \
  -e N8N_SECURE_COOKIE=false \
  -e GENERIC_TIMEZONE=Africa/Lagos \
  -e TZ=Africa/Lagos \
  -v /home/ubuntu/.n8n:/home/node/.n8n \
  n8nio/n8n
```

Change `Africa/Lagos` to your timezone.

Verify it's running:
```bash
sudo docker ps
```

You should see n8n with status `Up XX seconds`.

### Access n8n

Open your browser and go to:
```
http://YOUR_SERVER_IP:5678
```

Register your account. Your n8n is running — but webhooks from the internet won't reach it yet. See Part 7 below.

---

## Part 6D — Direct Docker + Custom Domain + HTTPS (Scenario D — No Upgrade, Has Domain)

> This is for users who have a domain but cannot upgrade to PAYG. You get a professional HTTPS URL using Caddy as a free reverse proxy. No Coolify needed.

Follow Part 6C first (Add Swap, Install Docker, Create n8n folder, Deploy n8n). Then continue here.

### Point Your Domain DNS to Your Server IP

Go to your domain registrar (Namecheap, etc.) → Advanced DNS → add an A record:

```
Type:  A Record
Host:  n8n  (or @ for root domain)
Value: YOUR_SERVER_IP
TTL:   Automatic
```

Wait for DNS to propagate — check at https://www.whatsmydns.net before proceeding.

### Install Caddy (Free Reverse Proxy with Auto SSL)

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy -y
```

### Configure Caddy

```bash
sudo nano /etc/caddy/Caddyfile
```

Replace the entire file content with (replace `n8n.yourdomain.com` with your actual subdomain):

```
n8n.yourdomain.com {
    reverse_proxy localhost:5678
}
```

Save and exit (Ctrl+X, Y, Enter), then restart Caddy:

```bash
sudo systemctl restart caddy
sudo systemctl enable caddy
```

Caddy automatically requests and renews a free Let's Encrypt SSL certificate for your domain. No extra steps needed.

### Update n8n to Use Your Domain

Stop the existing n8n container and recreate it with the correct webhook URL:

```bash
sudo docker stop n8n
sudo docker rm n8n
sudo docker run -d \
  --name n8n \
  --restart always \
  -p 5678:5678 \
  -e N8N_SECURE_COOKIE=true \
  -e GENERIC_TIMEZONE=Africa/Lagos \
  -e TZ=Africa/Lagos \
  -e WEBHOOK_URL=https://n8n.yourdomain.com \
  -e N8N_EDITOR_BASE_URL=https://n8n.yourdomain.com \
  -e N8N_HOST=n8n.yourdomain.com \
  -e N8N_PROTOCOL=https \
  -v /home/ubuntu/.n8n:/home/node/.n8n \
  n8nio/n8n
```

Change `Africa/Lagos` to your timezone and `n8n.yourdomain.com` to your actual domain.

### Access n8n

Open your browser and go to:
```
https://n8n.yourdomain.com
```

You will see the n8n setup page with a valid SSL certificate. Your webhook URL in workflows will be:
```
https://n8n.yourdomain.com/webhook/YOUR_WORKFLOW_ID
```

**Pros:** Free HTTPS, professional custom domain, stable webhook URL, no account needed beyond your domain registrar
**Cons:** 1GB RAM (add swap from Part 6C to avoid crashes), no Coolify management UI, SQLite database (sufficient for personal/small team use)

---

## Part 7 — Webhook URLs for Scenario C (No Domain, No Coolify)

When n8n runs directly on Docker without a domain or reverse proxy, it only has a raw IP and port (`http://YOUR_IP:5678`). This works for accessing the editor, but **webhooks from external services (Stripe, WhatsApp, Lovable, GitHub etc.) require a stable public HTTPS URL**. Here are your options:

> ⚠️ **Critical note:** Every time you change your `WEBHOOK_URL` environment variable and restart n8n, you must **deactivate and reactivate all existing workflows** in the n8n editor. This forces n8n to re-register the new webhook URLs with the correct address.

---

### Option 1 — Cloudflare Tunnel (Best Free Option — No Account Limit)

Cloudflare Tunnel is completely free, creates a permanent stable URL, and is more reliable than ngrok on the free plan. It works by running a lightweight `cloudflared` agent on your server that connects outbound to Cloudflare's network — no port forwarding needed.

**Setup:**

1. Install cloudflared on your server:

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64 -o cloudflared
sudo mv cloudflared /usr/local/bin/
sudo chmod +x /usr/local/bin/cloudflared
```

2. Create a quick tunnel (gives you an instant URL to test with):

```bash
cloudflared tunnel --url http://localhost:5678
```

Cloudflare will print a URL like `https://random-words.trycloudflare.com`. This URL works immediately but changes every time you restart it.

3. For a **permanent stable URL**, log in to https://dash.cloudflare.com → go to **Zero Trust** → **Networks** → **Tunnels** → create a named tunnel. Follow the setup wizard and Cloudflare will give you a permanent subdomain.

4. Run the named tunnel in the background:

```bash
nohup cloudflared tunnel run YOUR_TUNNEL_NAME &
```

5. Update n8n with the Cloudflare URL:

```bash
sudo docker stop n8n && sudo docker rm n8n
sudo docker run -d \
  --name n8n \
  --restart always \
  -p 5678:5678 \
  -e N8N_SECURE_COOKIE=false \
  -e GENERIC_TIMEZONE=Africa/Lagos \
  -e TZ=Africa/Lagos \
  -e WEBHOOK_URL=https://your-tunnel.trycloudflare.com \
  -e N8N_EDITOR_BASE_URL=https://your-tunnel.trycloudflare.com \
  -v /home/ubuntu/.n8n:/home/node/.n8n \
  n8nio/n8n
```

**Pros:** Completely free, no request limits, HTTPS, most secure option
**Cons:** Named tunnel requires a Cloudflare account (free), quick tunnel URL changes on restart

---

### Option 2 — ngrok Free Static Domain

ngrok gives every free account one permanent static domain that never changes or expires, even if you restart your server.

**Setup:**

1. Create a free account at https://ngrok.com
2. In the ngrok dashboard → click **Cloud Edge** → **Domains** → click **"Create Domain"**
3. ngrok assigns you a permanent domain e.g. `whale-suited-regularly.ngrok-free.app`
4. Copy your authtoken from the ngrok dashboard
5. Install ngrok on your server:

```bash
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update && sudo apt install ngrok -y
ngrok config add-authtoken YOUR_AUTHTOKEN
```

6. Start ngrok with your permanent static domain:

```bash
nohup ngrok http --url=whale-suited-regularly.ngrok-free.app 5678 &
```

7. Update n8n with the ngrok URL:

```bash
sudo docker stop n8n && sudo docker rm n8n
sudo docker run -d \
  --name n8n \
  --restart always \
  -p 5678:5678 \
  -e N8N_SECURE_COOKIE=false \
  -e GENERIC_TIMEZONE=Africa/Lagos \
  -e TZ=Africa/Lagos \
  -e WEBHOOK_URL=https://whale-suited-regularly.ngrok-free.app \
  -e N8N_EDITOR_BASE_URL=https://whale-suited-regularly.ngrok-free.app \
  -v /home/ubuntu/.n8n:/home/node/.n8n \
  n8nio/n8n
```

Your webhook URL in n8n workflows will be:
```
https://whale-suited-regularly.ngrok-free.app/webhook/YOUR_WORKFLOW_ID
```

External services like Stripe, WhatsApp, Lovable, and GitHub can all reach this URL without issues.

**Pros:** Free, HTTPS, one permanent static domain per free account
**Cons:** 1 static domain limit and 20k requests/month on free plan

---

### Option 3 — sslip.io + Caddy (No Account Needed)

sslip.io turns your server IP into a free hostname. Combined with Caddy (which handles SSL automatically), you get HTTPS with zero accounts required.

**Install Caddy:**

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy -y
```

**Configure Caddy** (replace YOUR_SERVER_IP):

```bash
sudo nano /etc/caddy/Caddyfile
```

```
YOUR_SERVER_IP.sslip.io {
    reverse_proxy localhost:5678
}
```

```bash
sudo systemctl restart caddy
sudo systemctl enable caddy
```

**Update n8n:**

```bash
sudo docker stop n8n && sudo docker rm n8n
sudo docker run -d \
  --name n8n \
  --restart always \
  -p 5678:5678 \
  -e N8N_SECURE_COOKIE=false \
  -e GENERIC_TIMEZONE=Africa/Lagos \
  -e TZ=Africa/Lagos \
  -e WEBHOOK_URL=https://YOUR_SERVER_IP.sslip.io \
  -e N8N_EDITOR_BASE_URL=https://YOUR_SERVER_IP.sslip.io \
  -v /home/ubuntu/.n8n:/home/node/.n8n \
  n8nio/n8n
```

**Pros:** Free, HTTPS via Let's Encrypt (Caddy handles it automatically), no account needed
**Cons:** URL contains raw IP (less professional), URL changes if Oracle reassigns your IP

---

### Option 4 — Buy a Cheap Domain (~$1–2/year)

Even without a PAYG upgrade, you can buy a `.xyz` domain for ~$1–2/year from Namecheap. Then follow Part 6D — you get a professional custom domain with full HTTPS via Caddy, stable webhook URLs, and no ongoing costs beyond the annual domain renewal.

---

## Part 8 — Useful Management Commands

### Check running containers
```bash
sudo docker ps
```

### View n8n logs
```bash
sudo docker logs n8n --tail 50
```

### Restart n8n
```bash
sudo docker restart n8n
```

### Check RAM usage
```bash
free -h
```

### Reconnect to your server
```bash
ssh -i /path/to/your-key.key ubuntu@YOUR_SERVER_IP
```

---

## Part 9 — Important Information to Save

Save all of this in a password manager:

| Item | Value |
|---|---|
| Server Public IP | YOUR_IP |
| SSH Key File | ssh-key-YYYY-MM-DD.key |
| SSH Command | `ssh -i key-file ubuntu@YOUR_IP` |
| n8n URL | http(s)://your-domain-or-ip |
| Coolify URL | http://YOUR_IP:8000 |
| Oracle Login | https://cloud.oracle.com |
| n8n Email | your email |
| n8n Password | your password |
| Coolify Email | your email |
| Coolify Password | your password |

---

## Scenario Comparison Summary

| Feature | Scenario A (PAYG + Domain) | Scenario B (PAYG, No Domain) | Scenario C (No Upgrade, No Domain) | Scenario D (No Upgrade, Has Domain) |
|---|---|---|---|---|
| VM Size | 2 OCPU, 12GB | 2 OCPU, 12GB | 1 OCPU, 1GB | 1 OCPU, 1GB |
| Coolify | ✅ Yes | ✅ Yes | ❌ No | ❌ No |
| PostgreSQL | ✅ Via Coolify | ✅ Via Coolify | ❌ SQLite only | ❌ SQLite only |
| Custom Domain | ✅ Yes | ❌ sslip.io | ❌ ngrok or sslip.io | ✅ Yes |
| HTTPS | ✅ Auto via Traefik | ✅ Auto via Traefik | ✅ Via ngrok or Caddy | ✅ Auto via Caddy |
| Webhook URL | ✅ Stable custom | ✅ Stable sslip.io | ✅ ngrok static domain | ✅ Stable custom |
| Cost | €0/month | €0/month | €0/month | €0/month |
| Best for | Production | Production | Testing / Learning | Small team / Personal |

---

*Guide version: July 2026 · Ubuntu 22.04 · Docker · n8n latest · Coolify 4.1.2*
