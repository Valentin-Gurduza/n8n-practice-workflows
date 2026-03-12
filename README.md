# n8n-practice-workflows
Repository dedicated for n8n workflows, self learning about n8n automation, tests

---

# Comprehensive Guide: Setting up n8n on Oracle Cloud Free Tier (1GB RAM) with Tailscale

This guide walks you through the step-by-step process of installing and running **n8n** (a powerful workflow automation tool) on an **Oracle Cloud Free Tier VM** with only **1GB of RAM**. 

Because 1GB of RAM is below n8n's recommended specifications, this guide includes crucial optimizations (like Swap memory and NodeJS limits) to prevent crashes. Furthermore, we will secure the instance using **Tailscale** to hide your n8n server from the public internet.

---

## Step 1: Create the Oracle VM
1. Go to your Oracle Cloud Dashboard and create a new **Compute Instance**.
2. **Image:** Choose `Ubuntu 22.04 LTS`.
3. **Shape:** Choose the Always Free `VM.Standard.E2.1.Micro` (1 OCPU, 1GB RAM) or an ARM Ampere shape if available in your region.
4. Add your SSH keys and create the instance.

## Step 2: Open Oracle Cloud Firewalls (VCN)
We need to temporarily open ports to allow inbound web traffic.
1. In Oracle Cloud, go to **Compute > Instances** and click your network VM.
2. Click the link next to **Subnet** under the Primary VNIC section.
3. Click on the **Security List** (e.g., `Default Security List...`).
4. Click **Add Ingress Rules** and add rules for the following Destination Ports:
   - `5678` (n8n default port)
   - `80` (HTTP)
   - `443` (HTTPS)
   *(Set Source CIDR to `0.0.0.0/0` for all of them).*

## Step 3: Connect to VM & Set Up Swap Memory
Since your VM only has 1GB of RAM, we must create a Swap file ("fake RAM" on the hard drive). Without this, n8n will run out of memory and crash.

Connect to your VM using PowerShell/Terminal:
```bash
ssh -i /path/to/your/key.key ubuntu@<YOUR_VM_PUBLIC_IP>
```

Run the following commands one by one to update the system, open the OS firewall, and create a 2GB swap file:
```bash
# Update the system
sudo apt update && sudo apt upgrade -y

# Open the internal Ubuntu firewall for n8n
sudo iptables -I INPUT -m state --state NEW -p tcp --dport 5678 -j ACCEPT
sudo netfilter-persistent save

# Create a 2GB Swap file to prevent Out of Memory crashes
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make the swap file permanent after reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Step 4: Install Docker
Docker is the easiest and cleanest way to run n8n.

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose plugin
sudo apt-get install docker-compose-plugin -y

# Allow your user to run docker commands without sudo
sudo usermod -aG docker $USER
newgrp docker
```

## Step 5: Install and Configure Tailscale
Tailscale creates a secure, private network (Tailnet). It also provides built-in reverse proxying and automatically handles HTTPS certificates.

**1. Install Tailscale:**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```
*(Click the link provided in the terminal to authenticate the VM to your Tailscale account).*

**2. Enable HTTPS in Tailscale:**
- Go to the [Tailscale Admin Console](https://login.tailscale.com/admin/dns) in your browser.
- Go to the **DNS** tab and ensure **MagicDNS** is enabled.
- Under HTTPS Certificates, click **Enable HTTPS**.

**3. Get your VM's Tailscale Domain Name:**
```bash
tailscale status
```
*(Find your VM's full domain name, e.g., `oracle-vm.coral-blue.ts.net`).*

**4. Start the Tailscale Reverse Proxy (`serve`):**
Tell Tailscale to securely route external HTTPS traffic to n8n's local port:
```bash
sudo tailscale serve --bg http://127.0.0.1:5678
```

## Step 6: Configure and Run n8n
Create a folder for n8n and set up the optimized configurations.

```bash
mkdir ~/n8n-docker
cd ~/n8n-docker
nano docker-compose.yml
```

Paste the following `docker-compose.yml` config. 
**IMPORTANT:** Replace `YOUR_TAILSCALE_DOMAIN` with your actual domain (e.g., `oracle-vm.coral-blue.ts.net`).

```yaml
version: "3"

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      # Bind to localhost only so it's inaccessible without Tailscale
      - "127.0.0.1:5678:5678"
    environment:
      # General Configuration
      - N8N_HOST=YOUR_TAILSCALE_DOMAIN
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://YOUR_TAILSCALE_DOMAIN/
      
      # ⚠️ CRITICAL 1GB RAM OPTIMIZATIONS ⚠️
      # Limits NodeJS RAM so it doesn't get killed by the OS
      - NODE_OPTIONS=--max-old-space-size=512
      
      # Clean up executions so SQLite DB stays small and fast
      - EXECUTIONS_DATA_SAVE_ON_ERROR=all
      - EXECUTIONS_DATA_SAVE_ON_SUCCESS=none
      - EXECUTIONS_DATA_SAVE_ON_PROGRESS=false
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=168 # Keep logs for max 7 days
      
      # Database and Core optimization
      - DB_TYPE=sqlite
      - N8N_ENFORCEMENT_MEMORY_LIMIT=true
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```
Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

Start n8n:
```bash
docker compose up -d
```

## Step 7: Access n8n Securely
Ensure the Tailscale app is running on your personal computer/phone and you are logged in.
Open your browser and navigate to: **`https://<your-tailscale-domain>`**

Set up your Owner Account and immediately enable **Two-Factor Authentication (2FA)** in your n8n profile settings.

## Step 8: Completely Hide the Server (Recommended)
Now that Tailscale is handling the connection, your server no longer needs open public ports.
Go back to your Oracle Cloud Dashboard -> **Compute > Instances > Subnet > Security List**.
**Delete** the Ingress rules for ports `80`, `443`, and `5678`.

Your server is now entirely invisible to the public internet.

## Step 9: Webhooks (Optional)
If you need to receive Webhooks from external public services (like Stripe or GitHub), those services won't be able to reach your private Tailscale network. 

To safely punch a hole for webhooks, use **Tailscale Funnel**:

1. Go to the [Tailscale Access Controls](https://login.tailscale.com/admin/acls/file).
2. Add `{"target": ["autogroup:member"], "attr": ["funnel"]},` to your `"nodeAttrs"`.
3. Run this command on your Oracle VM:
```bash
sudo tailscale funnel --bg http://127.0.0.1:5678
```

*Note: Enabling Funnel makes your login screen visible to the public internet. With a strong password and 2FA enabled, your server remains extremely secure against brute-force attacks.*
