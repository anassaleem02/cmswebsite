# FM's Power Solar CMS — Server Deployment Guide

## Server Details

| Item | Value |
|------|-------|
| **Provider** | Hostinger VPS |
| **OS** | Ubuntu 24.04 LTS |
| **IP** | `187.77.146.102` |
| **Domains** | `thefmspower.com`, `thefmstrading.com` |
| **Domain Provider** | hostndomain |
| **Stack** | .NET 8 + Angular + PostgreSQL + Nginx |

---

## Pre-Requisite: Point DNS

Contact **hostndomain support** or add these DNS records yourself:

### For thefmspower.com:
| Type | Host | Value |
|------|------|-------|
| A | `@` | `187.77.146.102` |
| A | `www` | `187.77.146.102` |

### For thefmstrading.com:
| Type | Host | Value |
|------|------|-------|
| A | `@` | `187.77.146.102` |
| A | `www` | `187.77.146.102` |

> DNS takes 5 min to 48 hours to propagate. Verify with: `nslookup thefmspower.com`

---

## Step 1: Connect to Server

```bash
ssh root@187.77.146.102
```

---

## Step 2: Update System

```bash
apt update && apt upgrade -y
```

---

## Step 3: Install .NET 8

```bash
wget https://packages.microsoft.com/config/ubuntu/24.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
dpkg -i packages-microsoft-prod.deb
apt update && apt install dotnet-sdk-8.0 -y
```

Verify:
```bash
dotnet --version
```

---

## Step 4: Install PostgreSQL

```bash
apt install postgresql postgresql-contrib -y
systemctl enable postgresql && systemctl start postgresql
```

Verify:
```bash
systemctl status postgresql
```

---

## Step 5: Install Node.js 20

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install nodejs -y
```

Verify:
```bash
node --version
npm --version
```

---

## Step 6: Install Nginx & Certbot

```bash
apt install nginx certbot python3-certbot-nginx -y
systemctl enable nginx
```

---

## Step 7: Setup Database

```bash
sudo -u postgres psql -c "CREATE USER solarcms_user WITH PASSWORD 'FmsPower@2026!';"
sudo -u postgres psql -c "CREATE DATABASE \"SolarCMS\" OWNER solarcms_user;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE \"SolarCMS\" TO solarcms_user;"
```

Verify:
```bash
sudo -u postgres psql -c "\l" | grep SolarCMS
```

---

## Step 8: Clone Project

```bash
git clone https://github.com/anassaleem02/CMS_Website.git /var/www/solarcms
```

---

## Step 9: Create Production Config

```bash
cat > /var/www/solarcms/phase-2-backend/SolarCMS.API/appsettings.Production.json << 'EOF'
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=SolarCMS;Username=solarcms_user;Password=FmsPower@2026!"
  },
  "Jwt": {
    "Key": "Fmspower2026SolarCMSJwtSecretKeyThatIsAtLeast64CharsLongForSecurity!!",
    "Issuer": "SolarCMS",
    "Audience": "SolarCMSClient",
    "ExpiryHours": 24
  },
  "UploadSettings": {
    "Path": "/var/www/solarcms/uploads"
  },
  "Serilog": {
    "MinimumLevel": "Warning"
  }
}
EOF
```

---

## Step 10: Build & Publish Backend

```bash
cd /var/www/solarcms/phase-2-backend/SolarCMS.API
dotnet publish -c Release -o /var/www/solarcms-api
mkdir -p /var/www/solarcms/uploads
chown -R www-data:www-data /var/www/solarcms/uploads
```

---

## Step 11: Create Backend Service

```bash
cat > /etc/systemd/system/solarcms.service << 'EOF'
[Unit]
Description=FM's Power Solar CMS API
After=network.target postgresql.service

[Service]
WorkingDirectory=/var/www/solarcms-api
ExecStart=/usr/bin/dotnet SolarCMS.API.dll
Restart=always
RestartSec=10
SyslogIdentifier=solarcms
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=ASPNETCORE_URLS=http://localhost:5000

[Install]
WantedBy=multi-user.target
EOF
```

Start the service:
```bash
systemctl daemon-reload
systemctl enable solarcms
systemctl start solarcms
```

Verify:
```bash
systemctl status solarcms
```

If there's an error, check logs:
```bash
journalctl -u solarcms -f
```

---

## Step 12: Build & Deploy Frontend

```bash
cd /var/www/solarcms/phase-3-angular

# Fix production API URL
sed -i "s|http://localhost:5000/api|/api|g" src/environments/environment.prod.ts

# Install dependencies and build
npm install
npm run build -- --configuration production

# Copy to web root
cp -r dist/phase-3-angular/browser/* /var/www/html/
```

---

## Step 13: Configure Nginx

```bash
cat > /etc/nginx/sites-available/solarcms << 'EOF'
server {
    listen 80;
    server_name thefmspower.com www.thefmspower.com thefmstrading.com www.thefmstrading.com;

    root /var/www/html;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    # Static files cache
    location ~* \.(js|css|png|jpg|jpeg|gif|svg|ico|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Angular SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API to .NET backend
    location /api/ {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        client_max_body_size 50M;
    }

    # Serve uploaded files
    # NOTE: ^~ is required so this prefix match takes priority over the
    # regex `location ~* \.(png|jpg|...)` block above — otherwise nginx
    # routes /uploads/*.png to the static-cache regex and returns 404.
    location ^~ /uploads/ {
        alias /var/www/solarcms/uploads/;
        expires 30d;
    }
}
EOF
```

Enable the config:
```bash
rm -f /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/solarcms /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

---

## Step 14: Setup Firewall

```bash
ufw allow 'Nginx Full'
ufw allow OpenSSH
ufw --force enable
```

---

## Step 15: Install SSL Certificate (After DNS is pointed!)

> Only run this AFTER `nslookup thefmspower.com` returns `187.77.146.102`

```bash
certbot --nginx -d thefmspower.com -d www.thefmspower.com -d thefmstrading.com -d www.thefmstrading.com
```

It will ask for your email — enter `thefmstrading@gmail.com` and agree to terms.

SSL auto-renews. Verify with:
```bash
certbot renew --dry-run
```

---

## Verify Deployment

```bash
# Check API is running
curl http://localhost:5000/api/settings

# Check website
curl http://localhost

# Check service status
systemctl status solarcms
systemctl status nginx
systemctl status postgresql
```

---

## After Deployment

| Item | Value |
|------|-------|
| **Website** | `https://thefmspower.com` |
| **Admin Panel** | `https://thefmspower.com/admin` |
| **Admin Email** | `admin@fmspower.com` |
| **Admin Password** | `Admin@1234` |
| **DB Password** | `FmsPower@2026!` |
| **JWT Key** | `Fmspower2026SolarCMSJwtSecretKeyThatIsAtLeast64CharsLongForSecurity!!` |

> **IMPORTANT:** Change the admin password immediately after first login!

---

## Troubleshooting

### Backend not starting?
```bash
journalctl -u solarcms -n 50 --no-pager
```

### Nginx error?
```bash
nginx -t
cat /var/log/nginx/error.log
```

### Database connection issue?
```bash
sudo -u postgres psql -c "\l"
sudo -u postgres psql -d SolarCMS -c "\dt"
```

### Frontend not loading?
```bash
ls -la /var/www/html/index.html
```

### Check all services at once:
```bash
systemctl status solarcms nginx postgresql
```

---

## Future Updates / Redeployment

```bash
cd /var/www/solarcms
git pull origin master

# Rebuild backend
cd phase-2-backend/SolarCMS.API
dotnet publish -c Release -o /var/www/solarcms-api
systemctl restart solarcms

# Rebuild frontend
cd /var/www/solarcms/phase-3-angular
npm install
npm run build -- --configuration production
cp -r dist/phase-3-angular/browser/* /var/www/html/
```

---

## Security Reminders

- [ ] Change admin password after first login
- [ ] Change DB password to something stronger
- [ ] Change JWT key to a random 64+ character string
- [ ] Keep system updated: `apt update && apt upgrade -y`
- [ ] Monitor logs regularly: `journalctl -u solarcms -f`
