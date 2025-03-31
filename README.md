# VPS Deployment Guide

## Steps

### Step 1: Login and Update VPS
For login use 
``` ssh root@your_vps_ip ```
then fill your password 

If you use private key then 
``` ssh -i "path_to_your_key" root@your_vps_ip ```

```sh
sudo apt update
sudo apt upgrade -y
sudo apt install curl git unzip ufw -y
```

### Step 2: Install Node.js and PM2
```sh
# Download and install nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash

# Load nvm without restarting the shell
\. "$HOME/.nvm/nvm.sh"

# Install Node.js
nvm install 22

# Verify installations
node -v  # Should print "v22.14.0"
nvm current  # Should print "v22.14.0"
npm -v  # Should print "10.9.2"
```

### Step 3: Install and Setup MongoDB
```sh
sudo apt-get install gnupg curl

curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# Create MongoDB list file
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Install MongoDB
sudo apt-get update
sudo apt-get install -y mongodb-org

# Start and enable MongoDB service
sudo systemctl daemon-reload
sudo systemctl start mongod
sudo systemctl enable mongod

# Check MongoDB status
sudo systemctl status mongod
```

#### MongoDB Connection URI:
```
mongodb://127.0.0.1:27017
```

### step 4: install Firewall
```sh
sudo apt update
sudo apt install ufw -y
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp   # Allow HTTP
sudo ufw allow 443/tcp  # Allow HTTPS
sudo ufw allow 27017/tcp  # MongoDB
``` 

### Step 5: Copy Code
```sh
mkdir /var/www
```
Clone the repository using Git or use FileZilla to copy files.
```sh
cd /var/www/your_repo
```

### Step 6: Deploy Backend
```sh
cd Backend
npm install
npm install -g pm2
pm2 start server.js --name Backend
pm2 startup 
pm2 save
```
allow firewall
```sudo ufw allow 4000/tcp   # Backend```
use pm2 list to check status of backend and pm2 logs to check logs
``` pm2 list ```
``` pm2 logs pm2_process-name/nbr ```

### Step 7: Deploy Frontend
```sh
cd Frontend
npm run build
```

### Step 8: Install and Configure Nginx
Configure DNS Records
Go to your domain registrar (e.g., Hostinger, Namecheap, GoDaddy, Cloudflare) and add the following DNS records:

```sh
1. A Record (For Main Domain)
Type: A

Host: @

Value (IPv4 Address): Your VPS/Public Server IP

TTL: Default (Auto/Automatic)
```
```sh
2. A Record (For WWW Subdomain)
Type: A

Host: www

Value (IPv4 Address): Your VPS/Public Server IP

TTL: Default (Auto/Automatic)
```

```sh
3. CNAME Record (Optional, for API Subdomain)
If you want to forward API requests to a subdomain like api.yourdomain.com, add:

Type: CNAME

Host: api

Value: yourdomain.com

TTL: Default (Auto/Automatic)
```
Now configure nginx so it can handle traffic
```sh
sudo apt install -y nginx
sudo nano /etc/nginx/sites-available/your_domain.com.conf
```

Replace the contents with the following configuration:

```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com www.yourdomain.com;
    
    # Serve Frontend
    root /var/www/your_project/Frontend;
    index index.html index.htm;

    # Proxy API requests to the backend
    location /api/ {
        proxy_pass http://localhost:PORT;  # Ensure this matches your backend service
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Serve static files from specific directories
    location /Photos {
        alias /var/www/your_project/path_to_Photos/;
        autoindex on;
    }

    location /aboutPhotos {
        alias /var/www/your_project/path_to_aboutPhotos/;
        autoindex on;
    }

    location /uploads {
        alias /var/www/your_project/Path_to_uploads/;
        autoindex on;
    }

    }

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name your_domain.com www.your_domain.com;
    return 301 https://$host$request_uri;
}

```

Create a symbolic link to enable the site:
```sh
ln -s /etc/nginx/sites-available/your_domain.com.conf /etc/nginx/sites-enabled/
```

Test the configuration and restart Nginx:
```sh
nginx -t
systemctl restart nginx
sudo ufw allow "Nginx Full"
```

### Step 9: Configure SSL Certificates
Use Certbot to enable SSL:
```sh
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d your_domain.com -d www.your_domain.com
sudo systemctl restart nginx
```

### Step 10:(Optional Step) Forward Backend API to a Subdomain
if your backend is directly serving static files (frontend) you dont need this step. skip it.

Edit Nginx config to add a subdomain for API hosting.
```sh
sudo nano /etc/nginx/sites-available/api.your_domain.com.conf
```
replace these following content 
```sh
server {
    listen 80;
    server_name api.your_domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name api.your_domain.com;

    location / {
        proxy_pass http://127.0.0.1:PORT;  # Ensure this is your backend port
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        }
    }
}
```
Create a symbolic link to enable the site:
```sh
ln -s /etc/nginx/sites-available/api.your_domain.com /etc/nginx/sites-enabled/
sudo certbot --nginx -d api.your_domain.com
```
### Step 11: Install Jenkins
```sh
sudo apt install openjdk-17-jdk -y
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
``` sudo ufw enable ```

```
after installing check status of jenkins and get initial admin password
```sudo cat /var/lib/jenkins/secrets/initialAdminPassword ```
now go to your browser ```http://your-server-ip:8080```
fill admin password and register 
then install necessary plugins ``` dashboard -> manage jenkins -> plugins``` ie. docker, nodejs,git etc 
now create new project choose pipeline paste follow code in jenkisfile after editing
```sh
pipeline {
    agent any
    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', credentialsId: 'create_yours_credentions_only_if_yours_repo_is_private', url: 'https://github.com/your_git_link.git'
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building the application...'
                sh '''
                # Copy files from Jenkins workspace to /var/www/your_project
                cp -r "/var/lib/jenkins/workspace/your_project/"* /var/www/your_project/
                # Navigate to backend
                cd "/var/www/your_project/backend" || exit 1
                
                # Install dependencies
                rm -rf node_modules package-lock.json
                npm cache clean --force
                npm install
                '''
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying the application...'
                sh '''
                cd "/var/www/your_project/frontend"
                npm i
                CI=false npm run build #CI=false because it wont treat warnings as errors
                sudo -S systemctl restart nginx < /dev/null

                
                # Navigate to deployment directory
                cd "/var/www/your_project/backend" || exit 1
                
                npm i
                # Restart PM2 process (uncomment the appropriate line)
                sudo pm2 restart your_project || sudo pm2 start index.js --name your_project
                pm2 save
                '''
            }
        }
        
    }
}
```
```sudo ufw allow 8080/tcp   # jenkins```
### Step 12: Setup Monitoring System
install docker and docker compose 
```sudo apt install -y docker.io docker-compose```
Install Prometheus, Grafana, and Loki for system monitoring in docker 
save this below code in ``` docker-compose.yml``` .
```sh 
version: '3.8'

services:
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    restart: unless-stopped
    volumes:
      - ./promtail-config.yml:/etc/promtail/config.yml
    networks:
      - monitoring

  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:latest
    container_name: nginx-exporter
    restart: unless-stopped
    networks:
      - monitoring

  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: unless-stopped
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/config.yml
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "9100:9100"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3001:3000"
    volumes:
      - ./grafana-data:/var/lib/grafana
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```
after saving docker-compose.yml run 
``` docker-compose up -d ```
change the config file inside container 
first save the below files with their respective name ie promtail.conf
change congig file of promtail
```sh
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://193.203.160.6:3100/loki/api/v1/push

scrape_configs:
  - job_name: nginx
    pipeline_stages:
      - regex:
          expression: '.*namespace=(?P<namespace>\S+).*pod=(?P<pod>\S+).*'
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx
          namespace: namespace  # Replace with actual namespace
          pod: pod           # Replace with actual pod name
          __path__: /var/log/nginx/*log
```
save above content inside promtail.conf, local-config.yml and prometheus.yml respectedly file

2. Loki
```sh
auth_enabled: false

server:
  http_listen_port: 3100

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

# By default, Loki will send anonymous, but uniquely-identifiable usage and configuration
# analytics to Grafana Labs. These statistics are sent to https://stats.grafana.org/
#
# Statistics help us better understand how Loki is used, and they show us performance
# levels for most users. This helps us prioritize features and documentation.
# For more information on what's sent, look at
# https://github.com/grafana/loki/blob/main/pkg/usagestats/stats.go
# Refer to the buildReport method to see what goes into a report.
#
# If you would like to disable reporting, uncomment the following lines:
#analytics:
#  reporting_enabled: false
```
3. Prometheus
```sh 
 Global configurations
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["localhost:9093"]

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "/etc/prometheus/alert.rules.yml"

# Scrape configurations
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets: ["193.203.160.6:9100"]

  # Scraping NGINX Plus metrics using Prometheus Exporter
  - job_name: "nginx"
    static_configs:
      - targets: ["193.203.160.6:9113"]

  # Loki logs scraping
  - job_name: "loki"
    static_configs:
      - targets: ["193.203.160.6:3100"]  # Use VPS IP if Prometheus and Loki are on different hosts
```

then copy these file inside container
```sh
docker cp promtail-config.yml promtail_container:/etc/promtail/config.yml
docker cp loki-config.yml loki_container:/etc/loki/local-config.yml
docker cp prometheus.yml prometheus_container:/etc/prometheus/prometheus.yml
```
othervise you can paste it manually also 
 Open a Shell in the Container ```docker exec -it container_name sh ``` and manually edit config file in all container in monitoring 
Start monitoring
``` docker-compose restart ```
Enable Firewall
```sh
sudo ufw allow 3100/tcp   # Loki
sudo ufw allow 9090/tcp   # Prometheus
sudo ufw allow 9100/tcp   # Node Exporter
sudo ufw allow 3001/tcp   # Grafana
sudo ufw allow 9080/tcp   # Promtail
```
``` sudo ufw enable ```

### Step 13: Copy Backup Scripts
Ensure automated backups are set up by copying necessary scripts to the VPS.
you need to install these dependency
```sh
curl https://rclone.org/install.sh | sudo bash
```
```sh
sudo apt update
sudo apt install mailutils -y
```
```sh
sudo apt install postfix -y
```
now configure rclone and mail service with your mail
for realtime mongodb backups . it creates a new backup inside backupdir everytime changes are made to mongodb
```sh
cat backup_system.js
const { MongoClient } = require('mongodb');
const { exec } = require('child_process');
const path = require('path');
const fs = require('fs');

// Configuration
const config = {
    mongoUri: 'mongodb://localhost:27017',
    dbName: 'AnantDrishti',
    backupDir: '/root/backups/mongo_backups',
    maxBackups: 3  // Keep only last 3 backups
};

// Ensure backup directory exists
if (!fs.existsSync(config.backupDir)) {
    fs.mkdirSync(config.backupDir, { recursive: true });
}

// Function to create backup using mongodump
async function createBackup() {
    function getISTTimestamp() {
    const now = new Date();
    now.setMinutes(now.getMinutes() + 330); // Convert UTC to IST (UTC+5:30)
    return now.toISOString().replace(/[:.]/g, '-');
}
const timestamp = getISTTimestamp();

    const backupPath = path.join(config.backupDir, `backup-${timestamp}`);

    return new Promise((resolve, reject) => {
        const command = `mongodump --uri="${config.mongoUri}" --db=${config.dbName} --out=${backupPath}`;

        exec(command, (error, stdout, stderr) => {
            if (error) {
                console.error(`Backup failed: ${error}`);
                reject(error);
                return;
            }
            console.log(`Backup created successfully at ${backupPath}`);
            cleanOldBackups();
            resolve(backupPath);
        });
    });
}

// Enhanced function to clean old backups
function cleanOldBackups() {
    fs.readdir(config.backupDir, (err, files) => {
        if (err) {
            console.error(`Error reading backup directory: ${err}`);
            return;
        }

        // Get all backup directories with their creation times
        const backups = files
            .filter(file => file.startsWith('backup-'))
            .map(file => ({
                name: file,
                path: path.join(config.backupDir, file),
                time: fs.statSync(path.join(config.backupDir, file)).birthtime
            }))
            .sort((a, b) => b.time - a.time); // Sort newest to oldest

        // Keep only the last 3 backups, delete the rest
        if (backups.length > config.maxBackups) {
            console.log(`Found ${backups.length} backups, keeping last ${config.maxBackups}`);

            // Get the backups to delete (all except the last 3)
            const backupsToDelete = backups.slice(config.maxBackups);

            // Delete each old backup
            backupsToDelete.forEach(backup => {
                try {
                    fs.rmSync(backup.path, { recursive: true, force: true });
                    console.log(`Deleted old backup: ${backup.name} from ${backup.time.toISOString()}`);
                } catch (error) {
                    console.error(`Failed to delete backup ${backup.name}:`, error);
                }
            });

            console.log(`Cleanup complete. ${backupsToDelete.length} old backups removed.`);
        } else {
            console.log(`Current backup count (${backups.length}) is within limit (${config.maxBackups}). No cleanup needed.`);
        }
    });
}

// Function to watch for database changes
async function watchDatabaseChanges() {
    try {
        const client = await MongoClient.connect(config.mongoUri);
        const db = client.db(config.dbName);

        console.log('Connected to MongoDB, watching for changes...');

        // Watch all collections in the database
        const changeStream = db.watch([], {
            fullDocument: 'updateLookup'
        });

        // Handle change events
        changeStream.on('change', async (change) => {
            console.log(`Detected change in database: ${change.operationType}`);
            try {
                await createBackup();
            } catch (error) {
                console.error('Failed to create backup:', error);
            }
        });

        // Handle errors
        changeStream.on('error', (error) => {
            console.error('Change stream error:', error);
            // Attempt to reconnect after error
            setTimeout(watchDatabaseChanges, 5000);
        });

        // Handle stream closure
        changeStream.on('close', () => {
            console.log('Change stream closed, attempting to reconnect...');
            setTimeout(watchDatabaseChanges, 5000);
        });

    } catch (error) {
        console.error('Failed to connect to MongoDB:', error);
        // Attempt to reconnect
        setTimeout(watchDatabaseChanges, 5000);
    }
}

// Start the backup system
watchDatabaseChanges().catch(console.error);

// Graceful shutdown
process.on('SIGINT', async () => {
    console.log('Shutting down backup system...');
    process.exit(0);
});
```
```sh
pm2 start backup_system.js --name backup
```

Mongo Sync Script
this script sync these realtime backups to remote gdrive . set cronjob to set syncing frequncy
``` sh
cat sync_mongo_backups.sh
#!/bin/bash

# Define local backup directory and Google Drive remote path
LOCAL_BACKUP_DIR=~/backups/mongo_backups
GDRIVE_REMOTE="gdrive:Backups/AnantDrishti/Realtime Backup/MongoDB"

# Log file for debugging
LOG_FILE="$LOCAL_BACKUP_DIR/rclone_sync.log"

echo "Starting sync: $(date)" | tee -a "$LOG_FILE"

# Run rclone sync
rclone sync -v "$LOCAL_BACKUP_DIR" "$GDRIVE_REMOTE" --log-file="$LOG_FILE"

if [ $? -eq 0 ]; then
    echo "Sync completed successfully: $(date)" | tee -a "$LOG_FILE"
else
    echo "Sync failed! Check the log at $LOG_FILE" | tee -a "$LOG_FILE"
fi
```
daily backup (cronjob) with email 
this script creates complete backup of code and mongodb daily set cronjob to set time 
```sh
cat daily_backup.sh
export TZ="Asia/Kolkata"
DATE=$(date +"%Y-%m-%d")
TIME=$(date +"%H:%M:%S")
SERVER_NAME="AnantDrishti"
BACKUP_ROOT="/root/backups/daily_backups"
LOG_FILE="$BACKUP_ROOT/backup_log.log"
RETENTION_DAYS=7

# Email settings
EMAIL_TO="harkaran@indraq.com,bharat@indraq.com"
EMAIL_FROM="info@indraQ.com"
EMAIL_SIGNATURE="<br><br>
<strong><span style='color: #073763; font-size: 18px;'>IndraQ Innovations</span></strong><br>
<strong><span style='color: #073763;'>Email :-</span></strong> <span style='color: #FF8000;'>info@indraQ.com</span><br>
<strong><span style='color: #073763;'>Phone :-</span></strong> <span style='color: #FF8000;'>+1 (929) 581-8345 / +91 7888818668</span><br>
<strong><span style='color: #073763;'>Website :-</span></strong> <a href='https://www.indraQ.com' style='color: #FF8000;'>www.indraQ.com</a><br>
<strong><span style='color: #073763;'>Address :-</span></strong> <span style='color: #FF8000;'>USA | Canada | UK | UAE | India | Australia</span><br>
<br>
<img src='https://ci3.googleusercontent.com/mail-sig/AIorK4zkfoQFzAyNk-scwXuNU7BiBFbb-KU14C--n0RdKR3AhUDhyv5wUYKQtZxRc-RBG9KcY7Ow9KNmns6b' alt='' width='200' height='50' />"

EMAIL_FAILURE_SUBJECT="❌ Backup Failed - $SERVER_NAME - $DATE"

# Ensure backup directory exists
BACKUP_DIR="$BACKUP_ROOT/$DATE"
mkdir -p "$BACKUP_DIR"

# MongoDB backup configuration
MONGO_BACKUP_DIR="$BACKUP_DIR/mongo"
code_backup_dir="$BACKUP_DIR/static_files"
mkdir -p "$code_backup_dir"
CODE_BACKUP_NAME="$code_backup_dir/code_backup.tar.gz"
GDRIVE_BACKUP_ROOT="gdrive:Backups/AnantDrishti/Daily Backups"

# Function to log messages
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Check if MongoDB service is running
if ! pgrep -x "mongod" > /dev/null; then
    log_message "ERROR: MongoDB service is not running!"
    EMAIL_FAILURE_BODY="<html><body>
    <p>Dear <strong>Harkaran Singh</strong>,</p>
    <p>⚠️ <strong>ALERT:</strong> The automated backup process for $SERVER_NAME failed on $DATE at $TIME.</p>
    <p><strong>📂 Backup Details:</strong></p>
    <ul>
        <li><strong>Backup Status :-</strong> ❌ Failed</li>
        <li><strong>Backup Date :-</strong> $DATE</li>
        <li><strong>Error Message :-</strong> MongoDB service is not running</li>
    </ul>
    <p>Please <a href='https://indraq.com/' target='_blank'>raise a ticket</a> for assistance.</p>
    <p>Thanks, Best Regards $EMAIL_SIGNATURE</body></html>"

    echo "$EMAIL_FAILURE_BODY" | mutt -e "set content_type=text/html" -s "$EMAIL_FAILURE_SUBJECT" -a "$LOG_FILE" -- $EMAIL_TO
    exit 1
fi

# MongoDB Backup (Only AnantDrishti DB)
log_message "Starting MongoDB backup for AnantDrishti..."
if mongodump --db AnantDrishti --out "$MONGO_BACKUP_DIR"; then
    log_message "MongoDB backup completed successfully"
else
    log_message "ERROR: MongoDB backup failed"
    EMAIL_FAILURE_BODY="<html><body>
    <p>Dear <strong>Harkaran Singh</strong>,</p>
    <p>⚠️ <strong>ALERT:</strong> The automated backup process for $SERVER_NAME failed on $DATE at $TIME.</p>
    <p><strong>📂 Backup Details:</strong></p>
    <ul>
        <li><strong>Backup Status :-</strong> ❌ Failed</li>
        <li><strong>Backup Date :-</strong> $DATE</li>
        <li><strong>Error Message :-</strong> MongoDB backup failed</li>
    </ul>
    <p>Please <a href='https://indraq.com/' target='_blank'>raise a ticket</a> for assistance.</p>
    <p>Thanks, Best Regards $EMAIL_SIGNATURE</body></html>"

    echo "$EMAIL_FAILURE_BODY" | mutt -e "set content_type=text/html" -s "$EMAIL_FAILURE_SUBJECT" -a "$LOG_FILE" -- $EMAIL_TO
    exit 1
fi

# Code Backup
log_message "Starting code backup..."
if tar -czf "$CODE_BACKUP_NAME" -C "/var/www/AnantDrishti" .; then
    log_message "Code backup completed successfully"
else
    log_message "ERROR: Code backup failed"
    EMAIL_FAILURE_BODY="<html><body>
    <p>Dear <strong>Harkaran Singh</strong>,</p>
    <p>⚠️ <strong>ALERT:</strong> The automated backup process for $SERVER_NAME failed on $DATE at $TIME.</p>
    <p><strong>📂 Backup Details:</strong></p>
    <ul>
        <li><strong>Backup Status :-</strong> ❌ Failed</li>
        <li><strong>Backup Date :-</strong> $DATE</li>
        <li><strong>Error Message :-</strong> Code backup failed</li>
    </ul>
    <p>Please <a href='https://indraq.com/' target='_blank'>raise a ticket</a> for assistance.</p>
    <p>Thanks, Best Regards $EMAIL_SIGNATURE</body></html>"

    echo "$EMAIL_FAILURE_BODY" | mutt -e "set content_type=text/html" -s "$EMAIL_FAILURE_SUBJECT" -a "$LOG_FILE" -- $EMAIL_TO
    exit 1
fi

# Calculate Backup Size
BACKUP_SIZE=$(du -sh "$BACKUP_DIR" | awk '{print $1}')

# Cleanup old local backups (older than retention period)
log_message "Cleaning up old local backups..."
find "$BACKUP_ROOT" -mindepth 1 -maxdepth 1 -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \;
log_message "Local cleanup completed."
```
remote backup with email notification
this script sync all daily backup to the remote google drive.
```sh 
 cat backup_to_gdrive.sh
#!/bin/bash

export TZ="Asia/Kolkata"
DATE=$(date +"%Y-%m-%d")
TIME=$(date +"%H:%M:%S")
SERVER_NAME="AnantDrishti"
BACKUP_ROOT="/root/backups/daily_backups"
BACKUP_DIR="$BACKUP_ROOT/$DATE"
LOG_FILE="$BACKUP_ROOT/gdrive_sync_log.log"
RETENTION_DAYS=7
GDRIVE_BACKUP_ROOT="gdrive:Backups/AnantDrishti/Daily Backups"
GDRIVE_BACKUP_DIR="$GDRIVE_BACKUP_ROOT/$DATE"
EMAIL_TO="harkaran@indraq.com,bharat@indraq.com"
EMAIL_FROM="info@indraQ.com"
EMAIL_SIGNATURE="Best regards,<br><br>
<strong><span style='color: #073763; font-size: 18px;'>IndraQ Innovations</span></strong><br>
<strong><span style='color: #073763;'>Email :-</span></strong> <span style='color: #FF8000;'>info@indraQ.com</span><br>
<strong><span style='color: #073763;'>Phone :-</span></strong> <span style='color: #FF8000;'>+1 (929) 581-8345 / +91 7888818668</span><br>
<strong><span style='color: #073763;'>Website :-</span></strong> <a href='https://www.indraQ.com' style='color: #FF8000;'>www.indraQ.com</a><br>
<strong><span style='color: #073763;'>Address :-</span></strong> <span style='color: #FF8000;'>USA | Canada | UK | UAE | India | Australia</span><br>"
EMAIL_FAILURE_SUBJECT="❌ Backup Sync Failed - $SERVER_NAME - $DATE"

log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

if [ ! -d "$BACKUP_DIR" ]; then
    log_message "ERROR: Backup directory $BACKUP_DIR does not exist!"
    EMAIL_FAILURE_BODY="<html><body>
    <p>Dear <strong>Harkaran Singh</strong>,</p>
    <p>⚠️ <strong>ALERT:</strong> The automated backup process for $SERVER_NAME failed on $DATE at $TIME.</p>
    <p><strong>📂 Backup Details:</strong></p>
    <ul>
        <li><strong>Backup Status :-</strong> ❌ Failed</li>
        <li><strong>Backup Date :-</strong> $DATE</li>
        <li><strong>Error Message :-</strong> Backup directory missing</li>
        <li><strong>Backup Location :-</strong> $GDRIVE_BACKUP_ROOT</li>
    </ul>
    <br>$EMAIL_SIGNATURE</body></html>"

    echo "$EMAIL_FAILURE_BODY" | mutt -e "set content_type=text/html" -s "$EMAIL_FAILURE_SUBJECT" -a "$LOG_FILE" -- $EMAIL_TO
    exit 1
fi

log_message "Starting sync to Google Drive..."
if rclone sync "$BACKUP_DIR" "$GDRIVE_BACKUP_DIR" --log-file="$LOG_FILE"; then
    log_message "Backup successfully synced to Google Drive"
    BACKUP_SIZE=$(du -sh "$BACKUP_DIR" | awk '{print $1}')

    log_message "Cleaning up old backups from Google Drive..."
    rclone delete "$GDRIVE_BACKUP_ROOT" --min-age ${RETENTION_DAYS}d --log-file="$LOG_FILE"
    rclone rmdirs "$GDRIVE_BACKUP_ROOT" --min-age ${RETENTION_DAYS}d --leave-root --log-file="$LOG_FILE"
    log_message "Google Drive cleanup completed."

    EMAIL_SUCCESS_SUBJECT="✅ Backup Successfully Completed - $SERVER_NAME - $DATE"
    EMAIL_SUCCESS_BODY="<html><body>
    <p>Dear <strong>Harkaran Singh</strong>,</p>
    <p>The automated backup process for <strong>$SERVER_NAME</strong> was successfully completed on <strong>$DATE</strong> at <strong>$TIME</strong>.</p>
    <p><strong>📂 Backup Details:</strong></p>
    <ul>
        <li><strong>Backup Status :-</strong> ✅ Success</li>
        <li><strong>Backup Date :-</strong> $DATE</li>
        <li><strong>Backup Location :-</strong> Google Drive [<a href='https://drive.google.com/drive/folders/1krbMCip0EsFTRl1zvR5yaqWHjerWX7ZI' target='_blank'>Link to backup Folder</a>]</li>
        <li><strong>Size :-</strong> $BACKUP_SIZE</li>
    </ul>
    <br>$EMAIL_SIGNATURE</body></html>"

    echo "$EMAIL_SUCCESS_BODY" | mutt -e "set content_type=text/html" -s "$EMAIL_SUCCESS_SUBJECT" -a "$LOG_FILE" -- $EMAIL_TO
    log_message "Success email notification sent to $EMAIL_TO"
else
    log_message "ERROR: Failed to sync backup to Google Drive"
    EMAIL_FAILURE_BODY="<html><body>
    <p>Dear <strong>Harkaran Singh</strong>,</p>
    <p>⚠️ <strong>ALERT:</strong> The automated backup process for $SERVER_NAME failed on $DATE at $TIME.</p>
    <p><strong>📂 Backup Details:</strong></p>
    <ul>
        <li><strong>Backup Status :-</strong> ❌ Failed</li>
        <li><strong>Backup Date :-</strong> $DATE</li>
        <li><strong>Error Message :-</strong> Sync failed</li>
        <li><strong>Backup Location :-</strong> Google Drive ($GDRIVE_BACKUP_ROOT)</li>
    </ul>
    <br>$EMAIL_SIGNATURE</body></html>"

    echo "$EMAIL_FAILURE_BODY" | mutt -e "set content_type=text/html" -s "$EMAIL_FAILURE_SUBJECT" -a "$LOG_FILE" -- $EMAIL_TO
    exit 1
fi
```



---


