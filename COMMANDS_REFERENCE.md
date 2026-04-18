# 🎯 Quick Command Reference

## SSH Connection Cheat Sheet

### Connect to Pi

```bash
# Basic connection
ssh salman@10.15.55.118

# With specific port (if changed to 2222)
ssh -p 2222 salman@10.15.55.118

# Verbose mode (for debugging)
ssh -vvv salman@10.15.55.118
```

### Passwordless SSH Setup

**Windows PowerShell:**
```powershell
# Generate key
ssh-keygen -t ed25519 -C "your_email@example.com"

# Copy to Pi
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh salman@10.15.55.118 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

**Mac/Linux Bash:**
```bash
# Generate and copy
ssh-keygen -t ed25519 -C "your_email@example.com"
ssh-copy-id -i ~/.ssh/id_ed25519.pub salman@10.15.55.118
```

### File Transfer

```bash
# Pi → Local
scp salman@10.15.55.118:~/file.txt ~/Downloads/

# Local → Pi
scp ~/file.txt salman@10.15.55.118:~/

# Directory (recursive)
scp -r salman@10.15.55.118:~/.openclaw ~/ 

# Compressed transfer
scp -C salman@10.15.55.118:~/large.zip ~/
```

### SSH Tunneling

```bash
# Port forward (local 18789 → Pi 18789)
ssh -L 18789:127.0.0.1:18789 salman@10.15.55.118

# Then access: http://localhost:18789/chat

# Keep connection alive
ssh -L 18789:127.0.0.1:18789 salman@10.15.55.118 -N
```

---

## OpenClaw Commands

### Status & Health

```bash
openclaw status                              # Full system status
openclaw status --deep                       # Detailed diagnostics
openclaw logs                                # View logs
openclaw logs --follow                       # Stream logs
openclaw logs --filter "error"               # Errors only
openclaw security audit                      # Security check
openclaw debug info                          # System info
```

### Gateway Control

```bash
openclaw gateway run                         # Start gateway
systemctl start openclaw.service             # Start (systemd)
systemctl stop openclaw.service              # Stop
systemctl restart openclaw.service           # Restart
systemctl status openclaw.service            # Check status
```

### Configuration

```bash
openclaw init                                # Initialize
nano ~/.openclaw/openclaw.json               # Edit config
openclaw config get agents.defaults.model.primary  # Get model
jq '.agents.defaults.model.primary' ~/.openclaw/openclaw.json  # View model
```

---

## Ollama Commands

### Installation & Startup

```bash
# Install
curl -fsSL https://ollama.ai/install.sh | sh

# Start
ollama serve &

# Start as service
systemctl start ollama

# Check status
systemctl status ollama
journalctl -u ollama -f
```

### Model Management

```bash
ollama pull llama3.2:1b              # Download model
ollama pull qwen2.5:1.5b             # Alternative model
ollama pull qwen2.5:0.5b             # Ultra-light

ollama list                          # Show all models
ollama show llama3.2:1b              # Model details
ollama rm llama3.2:1b                # Delete model
ollama cp llama3.2:1b mymodel        # Copy/rename

ollama run llama3.2:1b "test"        # Run once
```

### API Testing

```bash
# Get model tags
curl -s http://127.0.0.1:11434/api/tags | jq '.models[] | {name, size}'

# Generate response
curl -X POST http://127.0.0.1:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.2:1b",
    "prompt": "Hello!",
    "stream": false
  }' | jq '.response'
```

---

## System Monitoring

### Quick Checks

```bash
# Memory
free -h

# Disk
df -h

# Processes
ps aux | grep -E "ollama|openclaw"

# Ports
lsof -i :11434    # Ollama
lsof -i :18789    # OpenClaw

# Temperature (Pi)
vcgencmd measure_temp
```

### Live Monitoring

```bash
# htop (interactive)
htop

# Watch memory
watch -n 1 'free -h'

# Watch disk
watch -n 5 'df -h'

# Follow logs
journalctl -u openclaw.service -f
journalctl -u ollama -f
tail -f ~/.openclaw/openclaw.log
```

---

## Switch Active Model

### Quick Method

```bash
# Edit config
nano ~/.openclaw/openclaw.json

# Find line: "primary": "ollama/llama3.2:1b"
# Change to: "primary": "ollama/qwen2.5:1.5b"

# Save and restart
sudo systemctl restart openclaw.service
```

### Programmatic Method

```bash
# One-liner model switch
node -e "
const fs = require('fs');
const p = process.env.HOME + '/.openclaw/openclaw.json';
const j = JSON.parse(fs.readFileSync(p, 'utf8'));
j.agents.defaults.model.primary = 'ollama/qwen2.5:1.5b';
fs.writeFileSync(p, JSON.stringify(j, null, 2));
console.log('Model changed to: ' + j.agents.defaults.model.primary);
"

# Restart
sudo systemctl restart openclaw.service
```

---

## Emergency Procedures

### Restart Everything

```bash
sudo systemctl restart ollama.service
sudo systemctl restart openclaw.service
sleep 3
openclaw status
```

### Kill Hanging Processes

```bash
pkill -9 -f "openclaw"
pkill -9 -f "ollama"
pkill -9 -f "node"

# Restart services
sudo systemctl restart ollama.service
sudo systemctl restart openclaw.service
```

### Check All Ports

```bash
netstat -tulpn | grep -E "11434|18789|3000"

# OR (newer systems)
ss -tulpn | grep -E "11434|18789|3000"
```

### Clear Logs

```bash
# Clear OpenClaw logs
sudo journalctl -u openclaw.service --vacuum-time=1d

# Clear Ollama logs
sudo journalctl -u ollama --vacuum-time=1d

# Clear all system logs
sudo journalctl --vacuum-time=1w
```

---

## Backup & Restore

### Backup Config

```bash
# Backup OpenClaw config
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup.$(date +%Y%m%d)

# Backup entire OpenClaw directory
tar -czf ~/openclaw_backup.tar.gz ~/.openclaw/

# SCP backup to local machine
scp ~/openclaw_backup.tar.gz salman@10.15.55.118:~/backups/
```

### Restore Config

```bash
# From backup file
cp ~/.openclaw/openclaw.json.backup ~/.openclaw/openclaw.json

# From archive
tar -xzf ~/openclaw_backup.tar.gz -C ~/
```

---

## Performance Tuning

### Reduce Memory Usage

```bash
# Edit config to reduce context window
nano ~/.openclaw/openclaw.json

# Change:
# "contextWindow": 8192  →  "contextWindow": 2048
# "maxTokens": 4096      →  "maxTokens": 1024

# Restart
sudo systemctl restart openclaw.service
```

### Stop Unnecessary Services

```bash
sudo systemctl stop bluetooth
sudo systemctl stop avahi-daemon
sudo systemctl disable bluetooth
sudo systemctl disable avahi-daemon
```

### Monitor Temperature

```bash
# Single check
vcgencmd measure_temp

# Continuous monitoring
watch -n 1 'vcgencmd measure_temp'

# Safe temperature: < 80°C
# Warning: > 80°C (throttling)
# Critical: > 85°C (thermal shutdown)
```

---

## Networking

### Get IP Address

```bash
hostname -I
ip addr | grep "inet "
ifconfig  # if installed
```

### Test Connectivity

```bash
# Internet test
ping 8.8.8.8 -c 4

# Local gateway
ping 192.168.1.1

# OpenClaw API
curl http://127.0.0.1:18789

# Ollama API
curl http://127.0.0.1:11434/api/tags
```

### Network Speed Test

```bash
# Install speedtest
curl -Lo speedtest-cli https://raw.githubusercontent.com/sivel/speedtest-cli/master/speedtest.py
chmod +x speedtest-cli

# Run test
python3 speedtest-cli
```

---

## Useful One-Liners

### Get system info
```bash
echo "OS: $(uname -a)"; echo "RAM: $(free -h | grep Mem)"; echo "Disk: $(df -h | grep root)"
```

### Check all listening ports
```bash
sudo netstat -tulpn 2>/dev/null | grep LISTEN
```

### Find large files
```bash
du -sh ~/* | sort -h | tail -10
```

### Monitor for OOM kills
```bash
grep -i "out of memory" /var/log/syslog
```

### See service startup time
```bash
systemd-analyze blame | head -10
```

---

## Configuration File Locations

```
~/.openclaw/openclaw.json          # Main config
/etc/systemd/system/openclaw.service   # OpenClaw service file
/etc/systemd/system/ollama.service     # Ollama service file
/var/log/syslog                   # System logs
/var/log/auth.log                 # SSH/Auth logs
~/.ssh/authorized_keys            # SSH public keys
~/.bashrc                          # Bash configuration
```

---

**Quick Reference v1.0** | Last Updated: April 2026
