# VPS Deployment Guide

## Steps

### Step 1: Login and Update VPS
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

### Step 4: Copy Code
```sh
mkdir /var/www
```
Clone the repository using Git or use FileZilla to copy files.
```sh
cd /var/www/your_repo
```

### Step 5: Deploy Backend
```sh
cd Backend
npm install
npm install -g pm2
pm2 start server.js --name AnantDrishti
pm2 startup
pm2 save
```

### Step 6: Deploy Frontend
```sh
cd Frontend
npm run build
```

### Step 7: Install and Configure Nginx
```sh
sudo apt install -y nginx
sudo nano /etc/nginx/sites-available/default
```

Replace the contents with the following configuration:

```nginx
log_format upstreamlog '$server_name to: $upstream_addr [$request] '
  'upstream_response_time $upstream_response_time '
  'msec $msec request_time $request_time';

server {
    listen 443 ssl;
    server_name indraq.tech www.indraq.tech;

    ssl_certificate /etc/letsencrypt/live/indraq.tech/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/indraq.tech/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    access_log /var/log/nginx/access.log upstreamlog;

    # Serve Frontend
    root /var/www/AnantDrishti/Frontend/;
    index index.html index.htm;

# Proxy Requests to Backend (Fixed)
    location api/ {  # Adjust this path if needed
        proxy_pass http://localhost:4000; # Use HTTP unless backend is explicitly running HTTPS
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;



    }
    # Fix for /products API route

    # Static file serving
    location /Photos {
        alias /var/www/AnantDrishti/Photos/;
        autoindex on;
    }

    location /aboutPhotos {
        alias /var/www/AnantDrishti/aboutPhotos/;
        autoindex on;
    }

    location /uploads {
        alias /var/www/AnantDrishti/Backend/uploads/;
        autoindex on;
    }

location = /nav.html {
    add_header 'Access-Control-Allow-Origin' 'https://blog.anantdrristi.com';
    add_header 'Access-Control-Allow-Methods' 'GET, OPTIONS';
    add_header 'Access-Control-Allow-Headers' 'Content-Type';
}


    # API Routing
    #location /api/ {
       # proxy_pass https://127.0.0.1:4000;
       # proxy_ssl_verify off;
       # proxy_http_version 1.1;
        #proxy_set_header Upgrade $http_upgrade;
        #proxy_set_header Connection 'upgrade';
       # proxy_set_header Host $host;
      #  proxy_set_header X-Real-IP $remote_addr;
     #   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #}
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name indraq.tech www.indraq.tech;
    return 301 https://$host$request_uri;
}
```

Create a symbolic link to enable the site:
```sh
ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/
```

Test the configuration and restart Nginx:
```sh
nginx -t
systemctl restart nginx
```

### Step 8: Configure SSL Certificates
Use Certbot to enable SSL:
```sh
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d indraq.tech -d www.indraq.tech
sudo systemctl restart nginx
```

### Step 9: Forward Backend API to a Subdomain
Edit Nginx config to add a subdomain for API hosting.
```sh
sudo nano /etc/nginx/sites-available/api.indraq.com
```
replace these following content 
```sh
cat api.indraq.tech
server {
    listen 80;
    server_name api.indraq.tech;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name api.indraq.tech;

    ssl_certificate /etc/letsencrypt/live/api.indraq.tech/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/api.indraq.tech/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    location / {
        proxy_pass http://127.0.0.1:4000;  # Ensure this is your backend port
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
ln -s /etc/nginx/sites-available/api.indraq.tech /etc/nginx/sites-enabled/
sudo certbot --nginx -d api.indraq.tech
```

### Step 10: Setup Monitoring System
install docker and docker compose 
```sudo apt install -y docker.io docker-compose```
Install Prometheus, Grafana, and Loki for system monitoring in docker 
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
Start monitoring
``` docker-compose up -d ```


### Step 11: Install Jenkins
```sh
sudo apt install openjdk-17-jdk -y
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

### Step 12: Enable Firewall
```sh
sudo ufw enable
sudo ufw allow OpenSSH

# Allow Nginx Full (Both HTTP and HTTPS)
sudo ufw allow "Nginx Full"

# Allow specific ports
sudo ufw allow 4000/tcp   # Backend
sudo ufw allow 27017/tcp  # MongoDB
sudo ufw allow 3100/tcp   # Loki
sudo ufw allow 9090/tcp   # Prometheus
sudo ufw allow 9100/tcp   # Node Exporter
sudo ufw allow 3001/tcp   # Grafana
sudo ufw allow 9080/tcp   # Promtail
sudo ufw allow 8080/tcp   #jenkins
```

### Step 13: Copy Backup Scripts
Ensure automated backups are set up by copying necessary scripts to the VPS.

---

## Conclusion
This guide covers all steps needed to deploy a full-stack application on a VPS with security, monitoring, and CI/CD setup. Adjust configurations as per your requirements.

