# 🎯 LocalAI Assistant with OpenClaw - Complete Setup Guide

A comprehensive guide to building a Personal AI Assistant using OpenClaw with local Ollama models on Raspberry Pi 4. Run a free, private WhatsApp chatbot without cloud dependencies.

## 📚 Table of Contents

1. [Architecture & Overview](#architecture)
2. [Hardware & Prerequisites](#hardware)
3. [Step-by-Step Installation](#installation)
4. [SSH Setup & Remote Management](#ssh)
5. [Model Configuration](#models)
6. [WhatsApp Integration](#whatsapp)
7. [All Essential Commands](#commands)
8. [Troubleshooting](#troubleshooting)

---

## 🏗️ Architecture

```
Your Windows/Mac Machine
        ↓ (SSH: port 22)
Raspberry Pi 4 (10.15.55.118)
├── OpenClaw Gateway (port 18789)
│   ├── Ollama Runtime (port 11434)
│   │   ├── llama3.2:1b (1.2B params) ✅ RECOMMENDED
│   │   ├── qwen2.5:1.5b (1.5B params)
│   │   └── qwen2.5:0.5b (494M params) [Ultra-light]
│   └── Web Dashboard (http://127.0.0.1:18789/chat)
└── WhatsApp Integration (via Twilio)
    └── Incoming messages → Ollama inference → WhatsApp response
```

---

## 💻 Hardware Requirements

| Component | Recommended | Minimum |
|-----------|-------------|---------|
| **Device** | Raspberry Pi 4 Model B | Raspberry Pi 4 Model B |
| **RAM** | 8GB | 4GB ⚠️ Tight |
| **Storage** | 128GB SSD | 64GB microSD |
| **CPU** | ARM Cortex-A72 4-core | Same |
| **Power** | 5V/3A USB-C | 5V/2.5A (risky) |
| **Network** | Gigabit Ethernet | WiFi 802.11ac |

### Model Compatibility

| Model | Size | RAM Needed | Speed | Quality |
|-------|------|-----------|-------|---------|
| **Llama 3.2 1B** ⭐ | 1.2B | 2-3GB | ~200ms/token | Excellent |
| **Qwen 2.5 1.5B** | 1.5B | 3-4GB | ~250ms/token | Very Good |
| **Qwen 2.5 0.5B** | 494M | 1-2GB | ~100ms/token | Good |
| **Llama 2 7B** | 7B | 8GB+ | 1000+ms/token | Very Good |

---

## 🚀 Step-by-Step Installation

### STEP 1: Prepare Raspberry Pi

#### 1.1 Flash Raspberry Pi OS

1. Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
2. Select: **Raspberry Pi OS (64-bit, Lite)**
3. Choose your microSD card
4. Write image (takes 5-10 minutes)

#### 1.2 First Boot

```bash
# Plug in microSD, power on, wait 2-3 minutes for boot

# From your Windows/Mac machine, connect via SSH
ssh pi@raspberrypi.local
# Password: raspberry (default)

# Change password IMMEDIATELY
passwd
```

#### 1.3 Enable Required Services

```bash
# Run configuration menu
sudo raspi-config

# Navigate: Interface Options → SSH → Enable
# Navigate: Interface Options → I2C → Enable (optional)
# Exit and reboot
sudo reboot
```

#### 1.4 Update Everything

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl wget htop vim build-essential
```

---

### STEP 2: Install Node.js v20

```bash
# Add NodeSource repository
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

# Install Node.js
sudo apt install -y nodejs

# Verify installation
node --version   # Should output: v20.x.x
npm --version    # Should output: 10.x.x
```

---

### STEP 3: Install and Configure Ollama

#### 3.1 Install Ollama

```bash
# Download and install (one-line installer)
curl -fsSL https://ollama.ai/install.sh | sh

# Verify
ollama --version
which ollama  # Should output: /usr/local/bin/ollama
```

#### 3.2 Enable Ollama as Service

```bash
# Ollama installs as systemd service automatically
sudo systemctl enable ollama
sudo systemctl start ollama

# Check status
sudo systemctl status ollama

# View live logs
journalctl -u ollama -f
```

#### 3.3 Pull Your Model

```bash
# Recommended: Llama 3.2 1B (1.2B params)
ollama pull llama3.2:1b
# Takes 2-5 minutes depending on internet speed

# OR Alternative: Qwen 2.5 1.5B
ollama pull qwen2.5:1.5b

# OR Ultra-light: Qwen 2.5 0.5B
ollama pull qwen2.5:0.5b

# List installed models
ollama list

# Test interactively
ollama run llama3.2:1b
# Type: "Hello, what is 2+2?"
# Exit with: /bye
```

#### 3.4 Verify Ollama API

```bash
# Check if models are accessible via API
curl -s http://127.0.0.1:11434/api/tags | jq '.models[] | {name, size}'

# Should output:
# {
#   "name": "llama3.2:1b",
#   "size": 1321098329
# }
```

---

### STEP 4: Install OpenClaw

#### 4.1 Global Installation

```bash
# Install globally
npm install -g openclaw

# Verify
openclaw --version
openclaw status

# Initialize config
openclaw init
```

#### 4.2 Configure Ollama Provider

```bash
# Edit the config file
nano ~/.openclaw/openclaw.json
```

**Find and update the `models.providers` section:**

```json
{
  "models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://127.0.0.1:11434",
        "api": "ollama",
        "apiKey": "OLLAMA_API_KEY",
        "models": [
          {
            "id": "llama3.2:1b",
            "name": "llama3.2:1b",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0 },
            "contextWindow": 8192,
            "maxTokens": 4096
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "ollama/llama3.2:1b"
      }
    }
  }
}
```

**Save**: `Ctrl+O` → Enter → `Ctrl+X`

#### 4.3 Test OpenClaw Gateway

```bash
# Start gateway (foreground for testing)
openclaw gateway run

# Expected output:
# OpenClaw gateway listening at http://127.0.0.1:18789
# Agent: main
# Ready

# Stop with: Ctrl+C
```

#### 4.4 Create Systemd Service for Auto-Start

```bash
# Create service file
sudo tee /etc/systemd/system/openclaw.service > /dev/null <<EOF
[Unit]
Description=OpenClaw AI Gateway
After=network.target ollama.service

[Service]
Type=simple
User=$USER
Group=$USER
ExecStart=$(which openclaw) gateway run
Restart=on-failure
RestartSec=10s
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable openclaw.service
sudo systemctl start openclaw.service

# Verify it's running
sudo systemctl status openclaw.service
```

---

### STEP 5: Enable Dashboard Access (Optional)

To access OpenClaw dashboard from your local machine:

#### 5.1 Setup SSH Tunnel

```bash
# From your Windows/Mac machine
ssh -L 18789:127.0.0.1:18789 salman@10.15.55.118

# Leave this terminal open, then open browser:
# http://localhost:18789/chat
```

#### 5.2 OR Allow Network Access

**Edit `~/.openclaw/openclaw.json`:**

```json
{
  "gateway": {
    "bind": "0.0.0.0",
    "port": 18789
  }
}
```

Then restart:
```bash
sudo systemctl restart openclaw.service
```

Access from any machine:
```
http://10.15.55.118:18789/chat
```

⚠️ **Warning**: This exposes API to network. Use firewall rules to restrict access.

---

### STEP 6: (Optional) WhatsApp Integration

#### 6.1 Setup Twilio Account

1. Go to [twilio.com/try-twilio](https://twilio.com/try-twilio)
2. Sign up with email and phone
3. Get **Account SID** and **Auth Token** from Dashboard
4. Enable WhatsApp Sandbox: [Twilio Console](https://www.twilio.com/console)
5. Join sandbox: Send `join [sandbox-code]` from WhatsApp to Twilio number

#### 6.2 Configure in OpenClaw

Edit `~/.openclaw/openclaw.json`:

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true,
      "config": {
        "accountSid": "ACxxxxxxxxxxxxxxxxxxxxxx",
        "authToken": "your_auth_token_here",
        "phoneNumber": "+1234567890"
      }
    }
  }
}
```

#### 6.3 Test Connection

```bash
# Restart OpenClaw
sudo systemctl restart openclaw.service

# Check status
openclaw status | grep -A 5 WhatsApp

# Send test message from WhatsApp
# You should get AI response!
```

---

## 🔌 SSH: Connect & Manage Your Pi

### Basic SSH Connection

```bash
# From your Windows/Mac terminal
ssh salman@10.15.55.118
# Enter password when prompted

# Test connection without login
ssh -o ConnectTimeout=5 salman@10.15.55.118 "echo Connected!"
```

### Setup Passwordless SSH (Recommended)

#### Windows (PowerShell):

```powershell
# Generate SSH key pair
ssh-keygen -t ed25519 -C "your_email@example.com"
# Press Enter for defaults, optionally set passphrase

# Copy public key to Pi
$content = Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
ssh salman@10.15.55.118 "mkdir -p ~/.ssh && echo '$content' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# Test passwordless login
ssh salman@10.15.55.118
# Should login without password!
```

#### Mac/Linux (Bash):

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "your_email@example.com"

# Copy to Pi (one command)
ssh-copy-id -i ~/.ssh/id_ed25519.pub salman@10.15.55.118

# Test
ssh salman@10.15.55.118  # No password needed
```

### Useful SSH Commands

```bash
# Copy file from Pi to your machine
scp salman@10.15.55.118:~/some_file.txt ~/Downloads/

# Copy directory from Pi
scp -r salman@10.15.55.118:~/.openclaw ~/Downloads/openclaw_backup/

# Run command on Pi without interactive shell
ssh salman@10.15.55.118 "openclaw status | head -20"

# Create secure tunnel for dashboard
ssh -L 18789:127.0.0.1:18789 salman@10.15.55.118 -N
# Then: Open http://localhost:18789

# SCP with compression
scp -C salman@10.15.55.118:~/large_file.zip ~/
```

---

## 🔄 Model Management

### Switch Active Model

```bash
# Option 1: Direct edit
nano ~/.openclaw/openclaw.json
# Change: "primary": "ollama/llama3.2:1b" → "ollama/qwen2.5:1.5b"
# Save and restart: sudo systemctl restart openclaw.service

# Option 2: Programmatic (one-liner)
node -e "
const fs = require('fs');
const p = process.env.HOME + '/.openclaw/openclaw.json';
const j = JSON.parse(fs.readFileSync(p, 'utf8'));
j.agents.defaults.model.primary = 'ollama/qwen2.5:1.5b';
fs.writeFileSync(p, JSON.stringify(j, null, 2));
console.log('Updated to: ' + j.agents.defaults.model.primary);
"

# Restart service
sudo systemctl restart openclaw.service
```

### Download New Models

```bash
# List available models
ollama list

# Pull new model (background-safe)
nohup ollama pull llama2:7b-chat > ~/ollama_pull.log 2>&1 &

# Monitor pull progress
tail -f ~/ollama_pull.log

# List all pulled models
ollama list

# Delete unused model
ollama rm qwen2.5:0.5b
```

---

## 📋 Essential Commands Reference

### OpenClaw Commands

```bash
# Status & Diagnostics
openclaw status                              # Full system health
openclaw status --deep                       # Deep diagnostics
openclaw logs                                # View logs
openclaw logs --follow                       # Stream logs (Ctrl+C to exit)
openclaw logs --filter "error"               # Show only errors
openclaw security audit                      # Security check

# Gateway Control
openclaw gateway run                         # Start (foreground)
systemctl start openclaw.service             # Start (background)
systemctl stop openclaw.service              # Stop service
systemctl restart openclaw.service           # Restart service
systemctl status openclaw.service            # Check status

# Config & Management
openclaw init                                # Initialize config
nano ~/.openclaw/openclaw.json               # Edit config
openclaw config get agents.defaults.model.primary  # Get current model
jq '.agents.defaults.model.primary' ~/.openclaw/openclaw.json  # View model
```

### Ollama Commands

```bash
# Server
ollama serve                    # Start server
systemctl status ollama         # Check service status
journalctl -u ollama -f         # View logs

# Models
ollama pull llama3.2:1b         # Download model
ollama list                     # Show installed models
ollama show llama3.2:1b         # Model details
ollama rm llama3.2:1b           # Delete model
ollama run llama3.2:1b "test"   # Run model once
ollama cp llama3.2:1b mymodel   # Rename/copy

# API Testing
curl http://127.0.0.1:11434/api/tags  # List models (API)
curl -X POST http://127.0.0.1:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{"model": "llama3.2:1b", "prompt": "Hello!", "stream": false}'
```

### System Monitoring

```bash
# System Health
free -h                         # RAM usage
df -h                          # Disk usage
ps aux | grep -E "ollama|openclaw|node"  # Process check
lsof -i :11434                 # Check Ollama port
lsof -i :18789                 # Check OpenClaw port
vcgencmd measure_temp          # CPU temperature (Pi-specific)

# Logs
journalctl -u openclaw.service -n 100  # Last 100 lines
journalctl -u openclaw.service -f      # Follow logs
journalctl -u ollama -f                # Ollama logs
tail -f ~/.openclaw/openclaw.log       # OpenClaw log file

# Network
ip addr                        # Get IP address
curl http://127.0.0.1:18789   # Test gateway
ping 8.8.8.8                  # Test internet
netstat -tulpn | grep -E "11434|18789"  # Port status

# htop (interactive dashboard)
htop                          # Press 'q' to exit
```

### Maintenance

```bash
# Restart all services
sudo systemctl restart ollama.service
sudo systemctl restart openclaw.service

# View recent system log
dmesg | tail -20

# Check disk for errors
sudo fsck.ext4 -n /dev/mmcblk0p2

# Update all packages
sudo apt update && sudo apt upgrade -y

# Clean up disk space
sudo apt autoremove -y
sudo apt clean

# Create config backup
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup

# Restore from backup
cp ~/.openclaw/openclaw.json.backup ~/.openclaw/openclaw.json
```

---

## 🔧 Troubleshooting

### Gateway Won't Start

```bash
# Check service status
sudo systemctl status openclaw.service

# View detailed logs
journalctl -u openclaw.service -n 50

# Kill lingering processes
pkill -9 -f "openclaw"
pkill -9 -f "node"

# Check if ports are in use
lsof -i :18789

# Try starting manually
openclaw gateway run

# If error: Port in use
# Change config port to 18790, or kill process on 18789
```

### Ollama Models Not Found

```bash
# Check Ollama is running
systemctl status ollama

# Check API endpoint
curl http://127.0.0.1:11434/api/tags

# If no response, start Ollama
ollama serve &

# Re-pull models
ollama pull llama3.2:1b
```

### Out of Memory (OOM Killer)

```bash
# Check RAM usage
free -h

# Monitor memory in real-time
watch -n 1 'free -h'

# Check which process uses most memory
ps aux --sort=-%mem | head -5

# Solution options:
# 1. Switch to smaller model
#    ollama pull qwen2.5:0.5b
# 2. Reduce context window in config (8192 → 2048)
# 3. Add swap (risky on SD card)
```

### Slow Response Time (>5 seconds)

```bash
# Check what's consuming resources
htop

# Monitor inference speed
journalctl -u openclaw.service -f | grep -i "duration\|inference"

# Causes:
# - Model too large → use smaller model
# - Low free RAM → close apps
# - SD card bottleneck → use USB SSD
# - CPU throttling → check temp: vcgencmd measure_temp

# If temp > 80°C, improve cooling (heatsink + fan)
```

### WhatsApp Messages Not Received

```bash
# Check channel status
openclaw status | grep -A 5 WhatsApp

# Verify Twilio credentials in config
cat ~/.openclaw/openclaw.json | jq '.channels.whatsapp'

# Test Twilio API
curl -u "ACCOUNT_SID:AUTH_TOKEN" \
  https://api.twilio.com/2010-04-01/Accounts

# View message logs
journalctl -u openclaw.service -f | grep -i "whatsapp"

# Rejoin sandbox (sandbox expires after 72 hours of inactivity)
# Send: "join [sandbox-code]" from WhatsApp again
```

### SSH Connection Refused

```bash
# Check SSH service is running
sudo systemctl status ssh

# Check SSH is listening on port 22
lsof -i :22

# Check SSH config syntax
sudo sshd -t

# Restart SSH
sudo systemctl restart ssh

# From local machine, use verbose mode to debug
ssh -vvv salman@10.15.55.118
```

---

## 📊 Performance Benchmarks

### On Raspberry Pi 4 (4GB RAM, Llama 3.2 1B)

| Operation | Duration | Notes |
|-----------|----------|-------|
| Model load | 500-800ms | First request only |
| First token | 800-1200ms | Prompt processing |
| Token generation | 150-250ms/token | Steady state |
| Full response | 2-5 seconds | Typical message |
| Max batch | 1-2 tokens | Limited by RAM |

### Model Comparison

| Model | Startup | Token Speed | Quality | Recommended |
|-------|---------|-------------|---------|-------------|
| Llama 3.2 1B | 700ms | 200ms | Excellent | ✅ YES |
| Qwen 2.5 1.5B | 900ms | 250ms | Very Good | ✅ YES |
| Qwen 2.5 0.5B | 400ms | 100ms | Good | ✅ For speed |

---

## 🔒 Security Hardening

### SSH Security

```bash
# Edit SSH config
sudo nano /etc/ssh/sshd_config

# Set these lines:
# PasswordAuthentication no     # Keys only
# PubkeyAuthentication yes
# PermitRootLogin no
# Port 2222                     # Change from 22 (optional)
# AllowUsers salman             # Limit users

# Restart SSH
sudo systemctl restart sshd

# Then connect with: ssh -p 2222 salman@10.15.55.118
```

### Firewall

```bash
# Enable UFW
sudo ufw enable

# Allow SSH
sudo ufw allow 22/tcp

# Block remote access to gateway (recommended)
sudo ufw deny 18789
sudo ufw deny 11434

# OR: Allow specific IP
sudo ufw allow from 192.168.1.100 to any port 18789

# Check status
sudo ufw status
```

### Data Privacy

✅ **Safe**:
- All data processed locally (no cloud)
- Models run on device only
- No API calls to external services

⚠️ **Considerations**:
- Backup `~/.openclaw/` regularly
- Secure SSH with keys, not passwords
- Keep system updated for security patches
- Use firewall to restrict network access

---

## 📞 Support

**If you encounter issues:**

1. **Check logs:**
   ```bash
   journalctl -u openclaw.service -f
   journalctl -u ollama -f
   ```

2. **Check system health:**
   ```bash
   openclaw status --deep
   free -h
   df -h
   ```

3. **Verify services:**
   ```bash
   curl http://127.0.0.1:11434/api/tags
   curl http://127.0.0.1:18789
   ```

4. **Restart fresh:**
   ```bash
   sudo systemctl restart ollama.service
   sudo systemctl restart openclaw.service
   sleep 5
   openclaw status
   ```

---

**Version**: 1.0.0  
**Last Updated**: April 2026  
**Tested On**: Raspberry Pi 4, 4GB RAM, Bullseye 64-bit
