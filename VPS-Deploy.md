## VPS Deployment Guide

### 1. Create SSH Keys

Generate an SSH key pair and copy the public key (`.pub`) to your server:
```bash
ssh-keygen -t ed25519 -C "your_email_or_name"
```

**Useful server info commands:**
```bash
lsb_release -a      # Ubuntu version
uname -m            # Architecture
free -h             # RAM / memory
nproc               # CPU cores
```

---

### 2. Update & Upgrade Packages
```bash
sudo apt update
sudo apt upgrade -y
```

---

### 3. Enable Firewall (UFW)
```bash
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw status verbose
```

---

### 4. Create a Non-Root User
```bash
sudo useradd -m -s /bin/bash <username>
sudo passwd <username>
# Enter new password when prompted
```

---

### 5. Install Docker (as Non-Root User)

Switch to the new user and follow the official Docker installation guide:
👉 https://docs.docker.com/engine/install/ubuntu/

> ⚠️ **Critical Warning:** When you expose a container port, Docker **bypasses UFW** and exposes it directly on the host machine.
> **Never publish ports in your `docker-compose.yml`** — not even by accident.

---

### 6. Grant Docker Access to Non-Root User

Docker installation automatically creates a `docker` group. Add your user to it:
```bash
sudo usermod -aG docker <username>
```

---

### 7. MongoDB — Create a Least-Privilege Admin User

**Goal:** Create a non-root MongoDB user with access scoped to a specific database only.
```bash
# Enter the running MongoDB container
docker exec -it <container_id_or_name> sh

# Connect to MongoDB (you'll be prompted for the password)
mongosh -u <admin_username> --authenticationDatabase admin -p
```

Once inside the shell:
```js
show dbs
use admin
show collections
db.system.users.find()

// Create a scoped user
db.createUser({
  user: "username",
  pwd: "password",       // Avoid special characters in the password
  roles: [
    { db: "databaseName", role: "readWrite" }
  ]
})
```

Update your connection string with the new credentials:
```
mongodb://username:password@<mongo_container_name>:27017/databasename?authSource=admin
```

> Replace `localhost` with the MongoDB **container name**.

---

### 8. Fix Docker + UFW Port Exposure

Docker can bypass UFW and expose container ports to the public internet. To prevent this, add custom rules to UFW's `after.rules`:

📖 Reference: https://stackoverflow.com/a/30383845
```bash
sudo vim /etc/ufw/after.rules
```

Append the following block at the **end of the file**:
```
# BEGIN UFW AND DOCKER
*filter
:ufw-user-forward - [0:0]
:DOCKER-USER - [0:0]
-A DOCKER-USER -j RETURN -s 10.0.0.0/8
-A DOCKER-USER -j RETURN -s 172.16.0.0/12
-A DOCKER-USER -j RETURN -s 192.168.0.0/16

-A DOCKER-USER -j ufw-user-forward

-A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 192.168.0.0/16
-A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 10.0.0.0/8
-A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 172.16.0.0/12
-A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 192.168.0.0/16
-A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 10.0.0.0/8
-A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 172.16.0.0/12

-A DOCKER-USER -j RETURN
COMMIT
# END UFW AND DOCKER
```

Then restart UFW:
```bash
sudo systemctl restart ufw
```

---

### 9. Deploy Traefik & Application

Push your code to the VPS and start the stack using Docker Compose along with all required `.env` files.

---

### 10. 🚨 NEVER Run This Command in Production
```bash
docker system prune -f
```

**What it does:** Removes unused Docker data to free disk space.

**Deletes:**
- Stopped containers
- Unused networks
- Dangling images (untagged images from old builds)
- Build cache

**Even more destructive variant — avoid entirely on production:**
```bash
docker system prune -af --volumes
```

> ☠️ **This deletes ALL volumes, including your database data. There is no undo.**

---

---

## Best Practices

---

### 🔒 Security Hardening

#### Disable Root SSH Login & Password Authentication

After your non-root user and SSH key are set up, lock down SSH access:
```bash
sudo vim /etc/ssh/sshd_config
```

Set (or confirm) the following values:
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Then restart SSH:
```bash
sudo systemctl restart ssh
```

> ✅ From this point on, only SSH key login is allowed. Password brute-force attacks are fully blocked.

---

#### Install Fail2Ban — Auto-ban Brute Force Attempts
```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Check banned IPs:
```bash
sudo fail2ban-client status sshd
```

---

#### Keep Packages Auto-Updated (Security Patches Only)
```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

This automatically applies security patches without touching other packages.

---

#### Secrets & Environment Variables

- **Never commit `.env` files to Git.** Always add them to `.gitignore`.
- Use a secrets manager like [Doppler](https://www.doppler.com/) or [Infisical](https://infisical.com/) for team environments.
- Rotate secrets immediately if they are ever accidentally exposed.
- Use different `.env` files per environment: `.env.development`, `.env.production`.
```bash
# Example .gitignore entry
.env
.env.*
!.env.example
```

---

#### Set Resource Limits on Docker Containers

Prevent a single container from consuming all server memory and CPU:
```yaml
# docker-compose.yml
services:
  app:
    image: your-app
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
```

---

### 📦 Backups & Recovery

#### Automated MongoDB Backups

Run a daily compressed backup using `mongodump` inside a cron job:
```bash
# Create backup script
sudo vim /usr/local/bin/mongo-backup.sh
```
```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d_%H-%M)
BACKUP_DIR="/backups/mongodb/$DATE"
mkdir -p "$BACKUP_DIR"

docker exec <mongo_container> mongodump \
  -u <username> -p <password> \
  --authenticationDatabase admin \
  --out /backup

docker cp <mongo_container>:/backup "$BACKUP_DIR"
echo "Backup completed: $BACKUP_DIR"
```
```bash
chmod +x /usr/local/bin/mongo-backup.sh
```

Schedule it as a daily cron job:
```bash
crontab -e

# Add this line — runs every day at 2 AM
0 2 * * * /usr/local/bin/mongo-backup.sh
```

---

#### Offload Backups to Remote Storage

Never store backups only on the same VPS — if the server dies, your backups die too.

Options:
- **AWS S3** — `aws s3 cp /backups/ s3://your-bucket/ --recursive`
- **Backblaze B2** — cheap, S3-compatible
- **Rclone** — supports almost any cloud storage provider
```bash
# Example: sync backups to S3 after dump
aws s3 sync /backups/mongodb/ s3://your-bucket/mongodb-backups/
```

---

#### Test Your Backups

A backup you've never restored is not a backup.
```bash
# Restore a backup to verify it works
docker exec -i <mongo_container> mongorestore \
  -u <username> -p <password> \
  --authenticationDatabase admin \
  /backup/path
```

Schedule a monthly restore drill to a staging server.

---

### 📊 Monitoring & Logging

#### Centralize Docker Logs

By default Docker logs are scattered. Set a log rotation policy globally to prevent disk exhaustion:
```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```
```bash
sudo systemctl restart docker
```

---

#### Monitor Server Resources — Netdata (Lightweight)

[Netdata](https://www.netdata.cloud/) gives you a real-time dashboard for CPU, RAM, disk, and network with zero configuration:
```bash
bash <(curl -Ss https://my-netdata.io/kickstart.sh)
```

Access at: `http://<your-vps-ip>:19999`

> 🔒 Protect this port with UFW — only allow access from your own IP:
> ```bash
> sudo ufw allow from <your_ip> to any port 19999
> ```

---

#### Set Up Uptime Monitoring

Use a free external monitor to alert you when your app goes down:

- [UptimeRobot](https://uptimerobot.com/) — free, 5-minute checks, email/Slack alerts
- [Betterstack](https://betterstack.com/) — more detailed, has a status page feature

---

#### Structured Application Logging

Log in JSON format so logs are parseable by monitoring tools:
```js
// Node.js — use Winston or Pino
import pino from "pino"
const logger = pino({ level: "info" })

logger.info({ userId: "123", action: "login" }, "User logged in")
```

Never use `console.log` in production. Use log levels: `debug`, `info`, `warn`, `error`.

---

### ⚡ Performance & Scaling

#### Enable Docker BuildKit for Faster Builds
```bash
export DOCKER_BUILDKIT=1
```

Or set it permanently in `/etc/docker/daemon.json`:
```json
{
  "features": { "buildkit": true }
}
```

---

#### Use a `.dockerignore` File

Avoid copying unnecessary files into your Docker image — this reduces build time and image size:
```
# .dockerignore
node_modules
.git
.env
.env.*
*.log
dist
coverage
```

---

#### Multi-Stage Docker Builds

Keep production images small and free of dev dependencies:
```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

#### Enable Swap Space (Low-RAM VPS)

If your VPS has less than 2GB RAM, add swap to prevent OOM crashes:
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make it permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verify:
```bash
free -h
```

---

#### Set Up Traefik Health Checks

Add health checks in your `docker-compose.yml` so Traefik only routes to healthy containers:
```yaml
services:
  app:
    image: your-app
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
```

Expose a simple `/health` endpoint in your app that returns `200 OK`.

---

#### Regularly Check Disk Usage

Docker build cache and logs accumulate over time. Schedule a weekly safe cleanup:
```bash
# Safe — only removes truly unused/dangling resources (NOT volumes)
docker system prune -f

# Check disk usage
df -h
du -sh /var/lib/docker
```

> This is the only safe version of `docker system prune`. Never add `-a` or `--volumes` on production.