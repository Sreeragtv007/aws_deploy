# ERPNext on AWS EC2 — Corrected Production Deployment Guide

**Frappe / ERPNext Version 16 · Ubuntu 24.04 LTS**

## What This Guide Covers

* Creating an EC2 instance and configuring its security group firewall
* Connecting to the instance, updating Ubuntu, setting the timezone, and creating the `frappe` user
* Installing system dependencies, MariaDB 11.8, Python 3.14 (`uv`), Node.js 24 (`nvm`), Yarn, and Bench
* Initializing a `version-16` bench, creating a site, installing ERPNext, and enabling the scheduler
* Configuring Supervisor and Nginx (including the `log_format main` and `node` `/usr/local/bin` fixes), building assets, and verifying the deployment
* Associating an Elastic IP, configuring DNS, enabling HTTPS with Certbot, and setting up backups[cite: 18, 19, 20, 21, 22, 23]

> **Starting server reference:** 2 vCPU, 8 GiB RAM (for example, `t3.large` or `m6i.large`), and 40–50 GB gp3 SSD on **Ubuntu Server 24.04 LTS** (64-bit x86) with a static/Elastic IP. This is a starting point for a small deployment, not a capacity guarantee[cite: 18, 19].

---

## Before You Begin

* **Target versions:** This walkthrough uses **Ubuntu Server 24.04 LTS**, **Python 3.14**, **Node.js 24**, and **MariaDB 11.8** for Frappe / ERPNext `version-16`. Confirm current requirements against official documentation before deploying[cite: 18, 19].
* **User privileges:** Run `sudo` commands for system packages, MariaDB, Nginx, and Supervisor; run Bench and site commands as the dedicated `frappe` Linux user (`sudo su - frappe`), never as `root`[cite: 19, 20].
* **Security group rules:** Allow **SSH (22)** from your IP only, **HTTP (80)** from `0.0.0.0/0`, and **HTTPS (443)** from `0.0.0.0/0`. Do **not** expose MariaDB (`3306`), Redis (`6379`), or the development server (`8000`) to the public internet[cite: 19].

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
| Certbot | Obtains and renews HTTPS certificates[cite: 19] |

---

## Step-by-Step Deployment

### 1. Create EC2 Instance & Configure Security Group

1. In the AWS EC2 console, launch an instance with **Ubuntu Server 24.04 LTS** (64-bit x86), **8 GiB RAM** (such as `t3.large`), **40–50 GB gp3 SSD**, and a saved key pair (`erpnext-key.pem`)[cite: 19, 20].
2. Configure security group `erpnext-sg` with inbound rules for **SSH (`22`)** from your IP, **HTTP (`80`)** from `0.0.0.0/0`, and **HTTPS (`443`)** from `0.0.0.0/0`. Keep ports `3306`, `6379`, and `8000` closed to the public internet[cite: 19].

### 2. Connect to the Instance, Set Timezone, and Create `frappe` User

Connect from PowerShell or your terminal:

```bash
ssh -i "erpnext-key.pem" ubuntu@YOUR_PUBLIC_IP
```[cite: 20]

Update packages and set the timezone (`Asia/Kolkata` in the guide; adjust to your required timezone):

```bash
sudo apt update && sudo apt upgrade -y
sudo timedatectl set-timezone Asia/Kolkata
date
```[cite: 20]

Create the dedicated `frappe` Linux user and switch to it using `sudo su - frappe` (fixing the broken two-line `sudo su` + `frappe` command in the original guide):

```bash
sudo adduser frappe
sudo usermod -aG sudo frappe
sudo su - frappe
```[cite: 20]

* **Verification:** Your prompt changes to `frappe@...` and `whoami` returns `frappe`.

### 3. Install System Packages

Install required Linux packages (with proper spacing on `python3-venv \`):

```bash
sudo apt install -y \
  git curl wget build-essential pkg-config \
  libffi-dev libssl-dev python3-dev python3-venv \
  xvfb libfontconfig redis-server nginx supervisor \
  software-properties-common
```[cite: 20]

* **Verification:** `apt` completes with exit code `0` and `redis-server`, `nginx`, and `supervisor` are installed[cite: 20].

### 4. Install and Configure MariaDB 11.8

Add the MariaDB 11.8 repository, install MariaDB server/client/headers, and start the service:

```bash
curl -LsS [https://r.mariadb.com/downloads/mariadb_repo_setup](https://r.mariadb.com/downloads/mariadb_repo_setup) | \
  sudo bash -s -- --mariadb-server-version="mariadb-11.8"
sudo apt update
sudo apt install -y mariadb-server mariadb-client libmariadb-dev
sudo systemctl enable --now mariadb
mariadb --version
```[cite: 20]

Write `/etc/mysql/mariadb.conf.d/99-frappe.cnf` with proper `=` assignments and underscores (fixing the broken lines in the original guide):

```bash
sudo tee /etc/mysql/mariadb.conf.d/99-frappe.cnf >/dev/null <<'EOF'
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
EOF

sudo systemctl restart mariadb
sudo mariadb-secure-installation
```[cite: 20, 21]

* **Verification:** `mariadb --version` reports `11.8.x` and `sudo systemctl status mariadb` is `active (running)`[cite: 20, 21, 23].

### 5. Install Python 3.14 (`uv`), Node.js 24 (`nvm`), and Yarn

Install `uv` and set Python 3.14 as the default:

```bash
curl -LsSf [https://astral.sh/uv/install.sh](https://astral.sh/uv/install.sh) | sh
source "$HOME/.local/bin/env"
uv python install 3.14 --default
python --version
uv --version
```[cite: 21]

Install `nvm` (piping to `| bash` so it actually installs), install Node.js 24 and Yarn, and symlink `node` and `npm` into `/usr/local/bin` so Supervisor's `node-socketio` process can find `node` without failing with `node not found` / `BACKOFF`:

```bash
curl -o- [https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh](https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh) | bash
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
```[cite: 21]

* **Verification:** `python --version` (`3.14.x`), `uv --version`, `node -v` (`v24.x.x`), `/usr/local/bin/node -v`, `npm -v`, and `yarn -v` all succeed[cite: 21].

### 6. Install Bench, Initialize v16, and Set Up Site & ERPNext

Install `frappe-bench` via `uv`:

```bash
uv tool install frappe-bench
export PATH="$HOME/.local/bin:$PATH"
bench --version
```[cite: 21]

Initialize a `version-16` bench (fixing `bench init --frappe-branch`), download ERPNext `version-16`, create your site, install ERPNext, and enable the scheduler (replace `erp.example.com` with your site name/domain):

```bash
bench init --frappe-branch version-16 frappe-bench
cd ~/frappe-
