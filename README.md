# Self-Hosting Supabase on Oracle Cloud with Docker

A complete guide to set up your own Supabase instance on Oracle Cloud using Docker, Nginx, SSL certificates, and automatic updates.

## 📋 Prerequisites

- Oracle Cloud account (free tier works)
- Domain name with DNS management access
- SSH access to your Oracle Cloud VM
- Basic command line knowledge

## 🚀 Step 1: Oracle Cloud VM Setup

### 1.1 Create VM Instance
1. Login to Oracle Cloud Console
2. Create a new VM instance:
   - **Shape**: VM.Standard.E2.1.Micro (Always Free)
   - **Image**: Ubuntu 22.04 LTS
   - **SSH Keys**: Generate and download your key pair
   - **Networking**: Allow HTTP (80) and HTTPS (443) traffic

### 1.2 Reserve Static IP
1. Go to **Networking → Virtual Cloud Networks → Public IPs**
2. Click on your VM's external IP → **Reserve Static IP**
3. Note down your static IP address (e.g., `123.456.78.90`)

### 1.3 Configure Security Rules
1. **Networking → Virtual Cloud Networks → Security Lists**
2. Add ingress rules:
   - **Port 80** (HTTP): Source `0.0.0.0/0`
   - **Port 443** (HTTPS): Source `0.0.0.0/0`

## 🌐 Step 2: DNS Configuration

Set up your subdomain to point to your Oracle Cloud VM:

| Record Type | Host | Value | TTL |
|-------------|------|--------|-----|
| A | `sb` | `123.456.78.90` | 14400 |

**Example**: If your domain is `example.com`, create `sb.example.com` pointing to your VM's IP.

## 🖥️ Step 3: Server Setup

### 3.1 Connect to Your VM
```bash
ssh -i ~/path/to/your-ssh-key.key ubuntu@123.456.78.90
```

### 3.2 Update System & Install Dependencies
```bash
sudo apt update
sudo apt install nginx git -y
```

### 3.3 Install Docker
```bash
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
```

### 3.4 Install Docker Compose
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

### 3.5 Install Certbot
```bash
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
```

## 📦 Step 4: Supabase Installation

### 4.1 Clone Supabase Repository
```bash
git clone --depth 1 https://github.com/supabase/supabase.git
cd supabase/docker
cp .env.example .env
```

### 4.2 Configure Environment Variables

Edit the `.env` file:
```bash
nano .env
```

**Required Changes:**

```env
# Database Password (25+ characters, all characters allowed)
POSTGRES_PASSWORD=SecureDatabasePassword123!@#

# JWT Secret (32+ characters, letters and numbers only)
JWT_SECRET=MyJWTSecretKey1234567890ABCDEFGHIJ

# External URLs (replace with your domain)
API_EXTERNAL_URL=https://sb.example.com
SUPABASE_PUBLIC_URL=https://sb.example.com

# Dashboard Credentials
DASHBOARD_USERNAME=admin
DASHBOARD_PASSWORD=SecureAdminPassword123

# Security Keys
SECRET_KEY_BASE=SecretKey64CharactersLong123456789012345678901234567890!@#$
VAULT_ENC_KEY=VaultEncryptionKey32CharsLong123456

# File Upload Limit (0 = no limit, value in bytes)
FILE_SIZE_LIMIT=0

# Keep existing ANON_KEY and SERVICE_ROLE_KEY as they are
```

**Password Requirements:**
- **POSTGRES_PASSWORD**: Minimum 25 characters, any characters allowed
- **JWT_SECRET**: Minimum 32 characters, **letters and numbers only**
- **SECRET_KEY_BASE**: Minimum 64 characters, any characters allowed  
- **VAULT_ENC_KEY**: Minimum 32 characters, letters and numbers only

### 4.3 Fix Docker Compose Port Mapping

Edit `docker-compose.yml` to expose Studio port:
```bash
nano docker-compose.yml
```

Find the `studio:` section (around line 12) and add ports:

```yaml
  studio:
    container_name: supabase-studio
    image: supabase/studio:2025.05.19-sha-3487831
    restart: unless-stopped
    ports:
      - "3000:3000"  # ADD THIS LINE
    healthcheck:
      test:
        [
          "CMD",
          "node",
          "-e",
          "fetch('http://studio:3000/api/platform/profile').then((r) => {if (r.status !== 200) throw new Error(r.status)})"
        ]
```

**⚠️ Important**: Add the `ports:` section exactly between `restart: unless-stopped` and `healthcheck:`.

### 4.4 Start Supabase
```bash
sudo docker-compose up -d
```

**This will take 10-15 minutes on the first run.**

### 4.5 Verify Installation
```bash
sudo docker ps
```

All containers should show "Up" status. Some may show "Restarting" or "Unhealthy" - this is normal for optional services like pooler and realtime.

## 🔒 Step 5: SSL Certificate Setup

### 5.1 Get SSL Certificate
```bash
sudo certbot certonly --nginx -d sb.example.com
```

Follow the prompts:
- Enter your email address
- Accept terms of service
- Choose whether to share email with EFF

**Certificate files will be stored at:**
- Certificate: `/etc/letsencrypt/live/sb.example.com/fullchain.pem`
- Private Key: `/etc/letsencrypt/live/sb.example.com/privkey.pem`

## 🌐 Step 6: Nginx Configuration

### 6.1 Create Nginx Configuration
```bash
sudo nano /etc/nginx/sites-available/supabase.conf
```

**Add this configuration** (replace `sb.example.com` with your subdomain):

```nginx
server {
    server_name sb.example.com;
    
    gzip on;

    # REST API
    location ~ ^/rest/v1/(.*)$ {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://localhost:8000;
        proxy_redirect off;
    }

    # Authentication
    location ~ ^/auth/v1/(.*)$ {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://localhost:8000;
        proxy_redirect off;
    }

    # Realtime
    location ~ ^/realtime/v1/(.*)$ {
        proxy_redirect off;
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Supabase Studio (Dashboard)
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/sb.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sb.example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    if ($host = sb.example.com) {
        return 301 https://$host$request_uri;
    }

    listen 80;
    server_name sb.example.com;
    return 404;
}
```

### 6.2 Enable Configuration
```bash
sudo ln -s /etc/nginx/sites-available/supabase.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## ✅ Step 7: Access Your Supabase Instance

Open your browser and navigate to: `https://sb.example.com`

You should see the Supabase Studio login page.

**Login with:**
- Username: `admin` (or whatever you set as DASHBOARD_USERNAME)
- Password: Your DASHBOARD_PASSWORD

## 🔄 Step 8: Automatic Updates & Auto-Start

### 8.1 Create Auto-Update Script

Create an optimized update script that only backs up configurations:
```bash
cd ~
nano update_supabase.sh
```

**Add this content:**
```bash
#!/bin/bash

LOG_FILE="/var/log/supabase_update.log"
BACKUP_DATE=$(date +'%Y-%m-%d_%H-%M-%S')

echo "=== Supabase Update Started: $(date) ===" >> $LOG_FILE

# Backup only important configs (NOT the database!)
echo "Creating config backup..." >> $LOG_FILE
mkdir -p ~/supabase_config_backup_$BACKUP_DATE
cp ~/supabase/docker/.env ~/supabase_config_backup_$BACKUP_DATE/
cp ~/supabase/docker/docker-compose.yml ~/supabase_config_backup_$BACKUP_DATE/
cp -r ~/supabase/docker/volumes/api ~/supabase_config_backup_$BACKUP_DATE/

# Navigate to supabase directory
cd ~/supabase/docker

# Stop current services
echo "Stopping Supabase services..." >> $LOG_FILE
sudo docker-compose down >> $LOG_FILE 2>&1

# Pull latest changes from repository
echo "Pulling latest Supabase updates..." >> $LOG_FILE
git pull origin master >> $LOG_FILE 2>&1

# Pull latest Docker images
echo "Pulling latest Docker images..." >> $LOG_FILE
sudo docker-compose pull >> $LOG_FILE 2>&1

# Start services with updated images
echo "Starting updated Supabase services..." >> $LOG_FILE
sudo docker-compose up -d >> $LOG_FILE 2>&1

# Wait for services to start
sleep 60

# Check if services are running
echo "Checking service status..." >> $LOG_FILE
sudo docker ps >> $LOG_FILE 2>&1

# Clean up old Docker images to save space
echo "Cleaning up old Docker images..." >> $LOG_FILE
sudo docker image prune -af >> $LOG_FILE 2>&1

# Remove old config backups (keep only last 2)
echo "Cleaning up old config backups..." >> $LOG_FILE
find ~/supabase_config_backup_* -maxdepth 0 -type d | sort | head -n -2 | xargs rm -rf

echo "=== Supabase Update Completed: $(date) ===" >> $LOG_FILE
echo "" >> $LOG_FILE
```

### 8.2 Make Script Executable & Setup Log
```bash
chmod +x ~/update_supabase.sh
sudo touch /var/log/supabase_update.log
sudo chmod 666 /var/log/supabase_update.log
```

### 8.3 Set Up Cronjobs

Open crontab:
```bash
sudo crontab -e
```

Add these lines:
```bash
# Auto-start Supabase after VM reboot
@reboot sleep 30 && cd /home/ubuntu/supabase/docker && /usr/local/bin/docker-compose up -d

# Supabase auto-update every Sunday at 3 AM German time
0 1 * * 0 /bin/bash /home/ubuntu/update_supabase.sh >/dev/null 2>&1
```

**Note**: `0 1 * * 0` = 1 AM UTC = 3 AM German time (CEST)

### 8.4 Verify Cronjobs
```bash
sudo crontab -l
```

## 🔧 Port Overview

Your Supabase instance uses these ports:

- **Port 3000**: Supabase Studio (Dashboard)
- **Port 8000**: Kong API Gateway (REST API, Auth, Realtime)
- **Port 5432**: PostgreSQL Database (internal)
- **Port 4000**: Analytics/Logflare (internal)

## 🛠️ Managing Your Instance

### Manual Commands
```bash
# Stop Supabase
cd ~/supabase/docker
sudo docker-compose down

# Start Supabase
cd ~/supabase/docker
sudo docker-compose up -d

# View Logs
cd ~/supabase/docker
sudo docker-compose logs -f

# Manual Update
sudo bash ~/update_supabase.sh
```

### Monitor Updates
```bash
# Check update logs
tail -n 50 /var/log/supabase_update.log

# List config backups
ls -la ~/supabase_config_backup_*

# Check running containers
sudo docker ps
```

## 📊 Auto-Update Features

- **Automatic updates** every Sunday at 3 AM German time
- **Auto-start** after VM reboot (30 second delay)
- **Config-only backups** (fast, space-efficient)
- **Keeps only 2 latest backups** (prevents disk filling)
- **Database data preserved** (never copied, always persists)
- **Detailed logging** of all operations
- **Automatic cleanup** of old Docker images

## 🔐 Data Safety

### What Gets Backed Up (Fast & Small)
- ✅ `.env` file (passwords, keys, configuration)
- ✅ `docker-compose.yml` (port mappings, service config)
- ✅ `volumes/api/kong.yml` (API gateway configuration)

### What Stays Safe (Never Copied)
- 💾 **Database data**: `./volumes/db/data/` (persists on disk)
- 💾 **File storage**: `./volumes/storage/` (your uploaded files)
- 💾 **All user data**: Tables, users, API keys remain intact

## 🛠️ Troubleshooting

### 502 Bad Gateway
1. Check if containers are running: `sudo docker ps`
2. Verify ports are accessible: `sudo netstat -tlnp | grep :3000` and `sudo netstat -tlnp | grep :8000`
3. Check Nginx logs: `sudo tail -f /var/log/nginx/error.log`

### Container Restart Issues
Some containers (pooler, realtime) may show "Restarting" or "Unhealthy" status. This is normal and doesn't affect core functionality.

### SSL Certificate Renewal
Certificates auto-renew. To manually renew:
```bash
sudo certbot renew
sudo systemctl reload nginx
```

### Restore from Config Backup (if needed)
```bash
cd ~/supabase/docker
sudo docker-compose down
cp ~/supabase_config_backup_YYYY-MM-DD_HH-MM-SS/.env .
cp ~/supabase_config_backup_YYYY-MM-DD_HH-MM-SS/docker-compose.yml .
cp -r ~/supabase_config_backup_YYYY-MM-DD_HH-MM-SS/api ./volumes/
sudo docker-compose up -d
```

## 🔐 Security Considerations

1. **Change default passwords** in `.env` file
2. **Use strong, unique passwords** for all services
3. **Regularly monitor update logs**
4. **Keep your domain DNS secure**
5. **Monitor Oracle Cloud billing** (shouldn't exceed free tier)

## 📚 API Usage

Once installed, your Supabase API will be available at:

- **REST API**: `https://sb.example.com/rest/v1/`
- **Auth API**: `https://sb.example.com/auth/v1/`
- **Realtime**: `https://sb.example.com/realtime/v1/`

Use your `ANON_KEY` and `SERVICE_ROLE_KEY` from the `.env` file for API authentication.

## 🎉 Success!

You now have a fully automated, self-hosted Supabase instance that:

- ✅ **Runs 24/7** on Oracle Cloud free tier
- ✅ **Auto-updates** every Sunday at 3 AM German time
- ✅ **Auto-starts** after VM reboots
- ✅ **Keeps your data safe** during updates
- ✅ **Manages disk space** automatically
- ✅ **Uses SSL encryption** with automatic renewal
- ✅ **Accessible via custom domain**

---

**Need help?** Check the [official Supabase documentation](https://supabase.com/docs) or [Docker self-hosting guide](https://supabase.com/docs/guides/self-hosting/docker).
