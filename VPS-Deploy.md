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