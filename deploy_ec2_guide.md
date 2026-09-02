# Complete Guide: Deploying FastAPI App to AWS EC2

This guide will walk you through hosting your **Voice Banking Assistant API** on AWS EC2 using Docker.

---

## Step 1: Launch an AWS EC2 Instance

1. Log into your **AWS Management Console** and search for **EC2**.
2. Click **Launch Instance**.
3. **Name**: `voice-banking-app`
4. **AMI (OS)**: Select **Ubuntu** (Ubuntu 24.04 LTS or 22.04 LTS).
5. **Instance Type**:
   - `t3.small` (2 vCPU, 2 GiB RAM) — *Recommended for PyTorch & OpenCV*.
   - `t2.micro` or `t3.micro` — *Free Tier eligible* (Note: We will configure Swap Memory in Step 3 so PyTorch runs without memory exhaustion).
6. **Key Pair**: Click **Create new key pair**.
   - Name: `voice-banking-key`
   - Private key format: `.pem`
   - Save the downloaded `voice-banking-key.pem` file to your computer.
7. **Network Settings (Security Group)**:
   - Check **Allow SSH traffic from** -> Select `Anywhere` (0.0.0.0/0) or `My IP`.
   - Check **Allow HTTP traffic from the internet** (Port 80).
   - Check **Allow HTTPS traffic from the internet** (Port 443).
   - Add a custom rule (Optional): **Custom TCP Rule**, Port `8001`, Source `0.0.0.0/0`.
8. **Storage**: Keep default 8 GB or increase to 16 GB root storage.
9. Click **Launch Instance**.

---

## Step 2: Connect to your EC2 Instance via SSH

Open PowerShell or Command Prompt on your computer and navigate to where your `.pem` file was saved:

```powershell
# Navigate to the folder containing your key file
cd C:\path\to\your\keyfolder

# Connect to EC2 (Replace <EC2-PUBLIC-IP> with your instance's IPv4 address from AWS)
ssh -i "voice-banking-key.pem" ubuntu@<EC2-PUBLIC-IP>
```

---

## Step 3: Server Setup & Swap Memory Configuration

Once connected to your Ubuntu server, run the following updates and set up 2GB swap space:

```bash
# 1. Update system packages
sudo apt update && sudo apt upgrade -y

# 2. Add 2GB Swap space (Crucial for PyTorch CPU on t2.micro/t3.micro)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 3. Install Docker & Docker Compose plugin
sudo apt install -y docker.io docker-compose-v2 git
sudo usermod -aG docker $USER
newgrp docker
```

---

## Step 4: Transfer Code to EC2

### Option A: Via Git (Recommended)
1. Push your project to GitHub / GitLab (Make sure `.env` is listed in `.gitignore`).
2. On EC2, clone your repo:
   ```bash
   git clone <YOUR_GIT_REPO_URL> app
   cd app
   ```

### Option B: Via SCP (Direct file upload from local PC)
From your local PC terminal:
```powershell
# Zip or transfer files directly to EC2
scp -i "voice-banking-key.pem" -r "d:\final project 1" ubuntu@<EC2-PUBLIC-IP>:~/app
```

---

## Step 5: Create `.env` File on EC2

Inside the `app` directory on your EC2 instance, create your `.env` file:

```bash
cd ~/app
nano .env
```

Paste your environment variables:
```env
GEMINI_API_KEY=your_actual_gemini_api_key_here
DATABASE_URL=sqlite:///banking.db
```
Press `Ctrl + O`, then `Enter` to save, and `Ctrl + X` to exit.

---

## Step 6: Build & Run the Container

Build and start the application using Docker Compose:

```bash
# Build & start container in background
docker compose up -d --build
```

To verify the container status and inspect live logs:
```bash
# Check running containers
docker ps

# View application logs
docker compose logs -f
```

---

## Step 7: Access Your Deployed Web App

Open your browser and navigate to:
- **Web App**: `http://<EC2-PUBLIC-IP>/`
- **FastAPI Interactive Docs**: `http://<EC2-PUBLIC-IP>/docs` or `http://<EC2-PUBLIC-IP>:8001/docs`

---

## Quick Reference Commands

| Command | Action |
| --- | --- |
| `docker compose up -d --build` | Build & run container in background |
| `docker compose logs -f` | View live application logs |
| `docker compose stop` | Stop application |
| `docker compose restart` | Restart application |
