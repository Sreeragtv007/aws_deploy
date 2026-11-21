# ERPNext on AWS EC2 — Corrected Production Deployment Guide

**Frappe / ERPNext Version 16 · Ubuntu 24.04 LTS**

## What This Guide Covers

* Creating an EC2 instance and configuring its security group firewall
* Connecting to the instance, updating Ubuntu, setting the timezone, and creating the `frappe` user
* Installing system dependencies, MariaDB 11.8, Python 3.14 (`uv`), Node.js 24 (`nvm`), Yarn, and Bench
* Initializing a `version-16` bench, creating a site, installing ERPNext, and enabling the scheduler
* Configuring Supervisor and Nginx (including the `log_format main` and `node` `/usr/local/bin` fixes), building assets, and verifying the deployment
* Associating an Elastic IP, configuring DNS, enabling HTTPS with Certbot, and setting up backups

> **Starting server reference:** 2 vCPU, 8 GiB RAM (for example, `t3.large` or `m6i.large`), and 40–50 GB gp3 SSD on **Ubuntu Server 24.04 LTS** (64-bit x86) with a static/Elastic IP. This is a starting point for a small deployment, not a capacity guarantee.

---

## Before You Begin

* **Target versions:** This walkthrough uses **Ubuntu Server 24.04 LTS**, **Python 3.14**, **Node.js 24**, and **MariaDB 11.8** for Frappe / ERPNext `version-16`. Confirm current requirements against official documentation before deploying.
* **User privileges:** Run `sudo` commands for system packages, MariaDB, Nginx, and Supervisor; run Bench and site commands as the dedicated `frappe` Linux user (`sudo su - frappe`), never as `root`.
* **Security group rules:** Allow **SSH (22)** from your IP only, **HTTP (80)** from `0.0.0.0/0`, and **HTTPS (443)** from `0.0.0.0/0`. Do **not** expose MariaDB (`3306`), Redis (`6379`), or the development server (`8000`) to the public internet.

---

## Deployment Architecture

| Component | Purpose |
| :--- | :--- |
| AWS EC2 / Ubuntu | Hosts the application and database services |
| Nginx | Receives web traffic on ports 80 and 443 and serves static assets |
| Frappe / ERPNext | Application framework and ERP software |
| MariaDB | Stores ERPNext site data |
| Redis | Cache, queues, and real-time services |
| Supervisor | Keeps web, scheduler, worker, and Socket.IO processes running |
| Certbot | Obtains and renews HTTPS certificates |

---

## Step-by-Step Deployment

### 1. Create EC2 Instance & Configure Security Group

1. In the AWS EC2 console, launch an instance with **Ubuntu Server 24.04 LTS** (64-bit x86), **8 GiB RAM** (such as `t3.large`), **40–50 GB gp3 SSD**, and a saved key pair (`erpnext-key.pem`).
2. Configure security group `erpnext-sg` with inbound rules for **SSH (`22`)** from your IP, **HTTP (`80`)** from `0.0.0.0/0`, and **HTTPS (`443`)** from `0.0.0.0/0`. Keep ports `3306`, `6379`, and `8000` closed to the public internet.

### 2. Connect to the Instance, Set Timezone, and Create `frappe` User

Connect from PowerShell or your terminal:

```bash
ssh -i "erpnext-key.pem" ubuntu@YOUR_PUBLIC_IP
```

Update packages and set the timezone (`Asia/Kolkata` in the guide; adjust to your required timezone):

```bash
sudo apt update && sudo apt upgrade -y
sudo timedatectl set-timezone Asia/Kolkata
date
```

Create the dedicated `frappe` Linux user and switch to it using `sudo su - frappe`:

```bash
sudo adduser frappe
sudo usermod -aG sudo frappe
sudo su - frappe
```

* **Verification:** Your prompt changes to `frappe@...` and `whoami` returns `frappe`.

### 3. Install System Packages

Install required Linux packages:

```bash
sudo apt install -y \
  git curl wget build-essential pkg-config \
  libffi-dev libssl-dev python3-dev python3-venv \
  xvfb libfontconfig redis-server nginx supervisor \
  software-properties-common
```

* **Verification:** `apt` completes with exit code `0` and `redis-server`, `nginx`, and `supervisor` are installed.

### 4. Install and Configure MariaDB 11.8

Add the MariaDB 11.8 repository, install MariaDB server/client/headers, and start the service:

```bash
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | \
  sudo bash -s -- --mariadb-server-version="mariadb-11.8"
sudo apt update
sudo apt install -y mariadb-server mariadb-client libmariadb-dev
sudo systemctl enable --now mariadb
mariadb --version
```

Write `/etc/mysql/mariadb.conf.d/99-frappe.cnf` with proper `=` assignments and underscores:

```bash
sudo tee /etc/mysql/mariadb.conf.d/99-frappe.cnf >/dev/null <<'EOF'
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
EOF

sudo systemctl restart mariadb
sudo mariadb-secure-installation
```

* **Verification:** `mariadb --version` reports `11.8.x` and `sudo systemctl status mariadb` is `active (running)`.

### 5. Install Python 3.14 (`uv`), Node.js 24 (`nvm`), and Yarn

Install `uv` and set Python 3.14 as the default:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source "$HOME/.local/bin/env"
uv python install 3.14 --default
python --version
uv --version
```

Install `nvm`, install Node.js 24 and Yarn, and symlink `node` and `npm` into `/usr/local/bin` so Supervisor's `node-socketio` process can locate `node`:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install 24
nvm use 24
npm install -g yarn
node -v
npm -v
yarn -v

sudo ln -sf "$(which node)" /usr/local/bin/node
sudo ln -sf "$(which npm)" /usr/local/bin/npm
/usr/local/bin/node -v
```

* **Verification:** `python --version` (`3.14.x`), `uv --version`, `node -v` (`v24.x.x`), `/usr/local/bin/node -v`, `npm -v`, and `yarn -v` all succeed.

### 6. Install Bench, Initialize v16, and Set Up Site & ERPNext

Install `frappe-bench` via `uv`:

```bash
uv tool install frappe-bench
export PATH="$HOME/.local/bin:$PATH"
bench --version
```

Initialize a `version-16` bench, download ERPNext `version-16`, create your site, install ERPNext, and enable the scheduler (replace `erp.example.com` with your site name/domain):

```bash
bench init --frappe-branch version-16 frappe-bench
cd ~/frappe-bench
bench version

bench get-app --branch version-16 erpnext
bench new-site erp.example.com
bench --site erp.example.com install-app erpnext
bench --site erp.example.com list-apps
bench --site erp.example.com enable-scheduler
```

* **Verification:** `bench --site erp.example.com list-apps` shows both `frappe` and `erpnext` on `version-16`.

### 7. Configure Production Services, Nginx, and Frontend Assets

Install Ansible and run `bench setup production frappe`:

```bash
cd ~/frappe-bench
sudo apt install -y ansible
sudo env "PATH=$PATH" bench setup production frappe
```

Ensure `log_format main` is defined inside the `http` block of `/etc/nginx/nginx.conf` (before `include /etc/nginx/conf.d/*.conf;`) so `sudo nginx -t` does not fail with `unknown log format "main"`, then build assets, set asset permissions, sync Nginx config, and start Supervisor services:

```bash
if ! sudo grep -q "log_format main" /etc/nginx/nginx.conf; then
  sudo sed -i '/include \/etc\/nginx\/conf\.d\/\*\.conf;/i \        log_format main '\''$remote_addr - $remote_user [$time_local] "$request" '\''\n                        '\''$status $body_bytes_sent "$http_referer" '\''\n                        '\''"$http_user_agent" "$http_x_forwarded_for"'\'';' /etc/nginx/nginx.conf
fi

cd ~/frappe-bench
bench build --force
bench --site erp.example.com clear-cache
sudo chmod 755 /home/frappe /home/frappe/frappe-bench /home/frappe/frappe-bench/sites
sudo chmod -R o+rX /home/frappe/frappe-bench/sites/assets

bench setup nginx
sudo cp ~/frappe-bench/config/nginx.conf /etc/nginx/conf.d/frappe-bench.conf
sudo nginx -t && sudo systemctl reload nginx

sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart all
sudo supervisorctl status
```

* **Verification:**
  * `sudo nginx -t` outputs `syntax is ok` and `test is successful`.
  * `sudo supervisorctl status` shows web, `node-socketio`, Redis, scheduler, and worker processes as `RUNNING`.
  * `curl -I -H "Host: erp.example.com" http://127.0.0.1/assets/frappe/dist/css/$(ls ~/frappe-bench/sites/assets/frappe/dist/css/ | head -n 1)` returns `HTTP/1.1 200 OK` with `Content-Type: text/css`.

### 8. Elastic IP, DNS A Record, HTTPS (Certbot), and Backups

1. In the AWS EC2 console, go to **Network & Security → Elastic IPs**, allocate an Elastic IP, and associate it with your EC2 instance.
2. At your DNS provider, create an **A** record (for example, `erp` -> `YOUR_ELASTIC_IP`) and wait until DNS resolves to your instance before requesting a certificate.

Install Certbot and enable HTTPS:

```bash
sudo apt install -y certbot python3-certbot-nginx
cd ~/frappe-bench
bench setup nginx
sudo systemctl reload nginx
sudo certbot --nginx -d erp.example.com
sudo certbot renew --dry-run
```

Create backups using `--with-files` and copy backups off-instance (such as to a private S3 bucket):

```bash
bench --site erp.example.com backup --with-files
```

---

## Verification & Maintenance Commands

```bash
cd ~/frappe-bench
bench version
bench --site erp.example.com list-apps
bench --site erp.example.com enable-scheduler
bench --site erp.example.com clear-cache
bench --site erp.example.com backup --with-files
sudo supervisorctl status
sudo supervisorctl restart all
sudo systemctl status nginx
sudo systemctl status mariadb
```

---

## Production Notes & Official References

* **Capacity & monitoring:** Monitor memory, CPU, disk space, and worker queues as usage grows.
* **Backups & security:** Keep encrypted off-instance backups, test restores periodically, restrict SSH to your IP, use strong passwords, and apply least-privilege AWS IAM permissions.
* **Updates & compatibility:** Do not run `bench update` blindly on production—take a backup, review release notes, and test first, keeping Frappe and ERPNext on compatible branches.
* **Official references:**
  * Frappe Installation: `https://docs.frappe.io/framework/user/en/installation`
  * Frappe Production Setup: `https://docs.frappe.io/framework/user/en/bench/guides/setup-production`
  * AWS EC2 Getting Started: `https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html`
  * AWS Security Groups: `https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html`