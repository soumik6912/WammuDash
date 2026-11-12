# Hosting Bot (Invites + Boosts + Pterodactyl Provisioning)

This project tracks real invites (no rejoin/double count, filters fast-leaves as fake, optionally requires a verified role),
tracks server boosts for your guild, and provisions 1 server per user via the Pterodactyl Application API.

## Security notice
If you ever pasted real tokens/API keys into chat or code, rotate them immediately.

## Features
- Invite attribution on join (compares invite uses vs cached values)
- Tracks member invite status: active / left / fake / unverified / rejoined
- Calculates:
  - Total invites (raw joins credited)
  - Actual invites (active + unique + not fake + optionally verified)
- Tracks Nitro boosts (premium_since)
- `/invites` shows dashboard (actual + total + left + fake + boosts)
- `/claim` lets eligible users submit email + username and provisions server
- Automatic enforcement:
  - If user falls below criteria -> DM warning + grace period
  - After grace period -> deletes server (12-hour audit job)

## User Tutorial

### How to Claim Your First Server
1. **Meet Requirements**: Ensure you have enough invites, boosts, or coins to qualify for a plan
2. **Use `/claim` Command**: In Discord, type `/claim` and fill out the form with your email and desired username
3. **Wait for Provisioning**: Your server will be automatically created (may take 1-2 minutes)
4. **Access Your Server**: Use the "Access Panel" button in the dashboard to log into your Pterodactyl control panel

### How to Start/Stop/Restart Your Server
1. **Go to Dashboard**: Visit your dashboard at the web URL
2. **Find Your Server Card**: Look for your server in the "Services" section
3. **Use Power Controls**:
   - **Start**: Click "Start" to boot up your server
   - **Stop**: Click "Stop" for a graceful shutdown
   - **Restart**: Click "Restart" to reboot your server
   - **Kill**: Click "Kill" for immediate shutdown (use only if stuck)

### How to Upgrade Your Server
1. **Check Available Plans**: Click "Store" in the navigation menu
2. **Choose New Plan**: Select a plan with better specs (more RAM, CPU, storage)
3. **Click "Upgrade"**: On your current server card, click the "Upgrade" button
4. **Complete Purchase**: Pay with coins if required
5. **Automatic Migration**: Your server will be automatically migrated to the new plan

### How to Delete Server & Create New One
**⚠️ WARNING: This permanently deletes your server. Backup important data first!**

1. **Stop Your Server**: Click "Stop" to gracefully shut down before deletion
2. **Cancel Server**: 
   - Click "Cancel Server" button on your server card
   - Read the warning carefully
   - Click "Confirm Delete Server"
3. **Wait for Deletion**: Server deletion takes 1-2 minutes
4. **Claim New Server**: 
   - Use `/claim` command again in Discord
   - Choose new username/email if desired
   - Wait for new server provisioning

### Server Management Tips
- **Auto-Shutdown**: Some plans automatically stop servers after a set time to save resources
- **Suspension**: Servers suspend if you run out of coins or lose eligibility
- **Unsuspend**: Click "Unsuspend" button when you meet requirements again
- **Backup Data**: Regularly backup your server files before making changes
- **Check Status**: Use the dashboard to monitor server CPU, RAM, and disk usage

### Troubleshooting
- **Server Won't Start**: Check if you have enough coins or meet plan requirements
- **Can't Access Panel**: Try "Auto Login to Panel" button or clear browser cache
- **Server Suspended**: Check notifications for reason and fix eligibility issues
- **Slow Performance**: Consider upgrading to a higher plan with more resources

## Setup (Local / VPS)
1. Create virtualenv and install dependencies:
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install --upgrade pip
   pip install -r requirements.txt
   ```
2. Update `.env` with your settings.
3. Enable Server Members Intent in Discord Developer Portal.
4. Ensure bot has permissions to View Server Invites (Manage Guild) + send embeds/DMs.

## Run options

### Option A: Web + Bot together (recommended)

```bash
source .venv/bin/activate
python wammu.py
```

## Plans
Edit `app/constants.py` to configure invite/boost plans, or use `/plan_add` to store plans in the database.

## PM2 (production)

Install PM2 once:
```bash
sudo apt update
sudo apt install -y nodejs npm
sudo npm install -g pm2
```

Start separated web and bot processes:
```bash
cd /var/www/dash
pm2 delete wammu || true
pm2 start deploy/pm2/ecosystem.config.cjs
pm2 save
pm2 startup
```

Useful commands:
```bash
pm2 status
pm2 logs wammu-web
pm2 logs wammu-bot
pm2 restart wammu-web
pm2 restart wammu-bot
pm2 stop wammu-web
pm2 stop wammu-bot
pm2 resurrect
```

Hardening for long uptime:
- Use `deploy/pm2/ecosystem.config.cjs` (includes backoff restarts + memory restart).
- Process auto-restarts if memory crosses limit (`max_memory_restart`).
- Keep app on one fixed port (no auto port hopping), so Nginx/Cloudflare upstream stays stable after restarts.
- The web app now runs separately from the bot with multiple Uvicorn workers (`WEB_CONCURRENCY`) for better burst handling.
- The web app also uses short in-process request queues and concurrency caps (`WEB_REQUEST_*`, `WEB_POLL_*`, `WEB_HEAVY_*`) to shed load cleanly during spikes.

After VPS reboot, verify:
```bash
pm2 status
ss -ltnp | grep 8000
curl -I http://127.0.0.1:8000/
```

## Nginx + SSL (localhost bound, Cloudflare Tunnel)

If you want tunnel -> Nginx on localhost with SSL:

Flow:
`Cloudflare Tunnel -> https://localhost:8343 -> Nginx -> http://127.0.0.1:8000 (FastAPI/Uvicorn)`

### 1) Create Nginx site
Also install the shared traffic zones file from `deploy/nginx/50-wammu-traffic.conf` into `/etc/nginx/conf.d/`.

Create `/etc/nginx/sites-available/dash.wammuhost.com`:

```nginx
server {
    listen 127.0.0.1:8343 ssl http2;
    server_name dash.wammuhost.com;

    access_log /var/log/nginx/dash.access.log;
    error_log  /var/log/nginx/dash.error.log warn;

    # Replace with your cert/key paths (Let's Encrypt or Cloudflare Origin cert)
    ssl_certificate     /etc/ssl/certs/dash.wammuhost.com.crt;
    ssl_certificate_key /etc/ssl/private/dash.wammuhost.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_prefer_server_ciphers off;

    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 5;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml image/svg+xml;

    client_max_body_size 50m;
    client_body_timeout 120s;
    sendfile off;

    # Trust cloudflared local connector
    real_ip_header CF-Connecting-IP;
    set_real_ip_from 127.0.0.1;
    set_real_ip_from ::1;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host $host;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_connect_timeout 60s;
        proxy_send_timeout 180s;
        proxy_read_timeout 180s;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### 2) Enable with symlink + reload
```bash
sudo ln -s /etc/nginx/sites-available/dash.wammuhost.com /etc/nginx/sites-enabled/dash.wammuhost.com
sudo nginx -t
sudo systemctl reload nginx
```

### 3) Cloudflared config
`/etc/cloudflared/config.yml`:

```yaml
tunnel: <TUNNEL_ID>
credentials-file: /etc/cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: dash.wammuhost.com
    service: https://localhost:8343
    originRequest:
      noTLSVerify: true
  - service: http_status:404
```

Then restart tunnel:
```bash
sudo systemctl restart cloudflared
sudo systemctl status cloudflared
```

### 4) Start app with PM2 and venv
```bash
cd /var/www/dash
python3 -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -r requirements.txt
deactivate

pm2 start /var/www/dash/.venv/bin/python --name wammu -- wammu.py
pm2 save
```

## Notes
- You can use SQLite or MySQL for bot data. Set `DB_DRIVER=sqlite` or `DB_DRIVER=mysql` in `.env`.
- The bot does not write to Pterodactyl's panel database directly.
- Pterodactyl API: Use an Application API key with permissions to create users/servers and delete servers.
