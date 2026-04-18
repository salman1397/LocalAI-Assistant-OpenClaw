# 🆘 Troubleshooting & FAQ

## Common Issues & Solutions

### SSH Connection Issues

**Problem**: "Connection refused" or "timeout"
```
ssh: connect to host 10.15.55.118 port 22: Connection refused
```

**Solutions**:
```bash
# 1. Check if SSH is enabled on Pi
ssh pi@raspberrypi.local "sudo systemctl status ssh"

# 2. Check if you're on the same network
ping 10.15.55.118

# 3. Try with verbose output
ssh -vvv salman@10.15.55.118

# 4. Restart SSH service on Pi
sudo systemctl restart ssh

# 5. Check firewall
sudo ufw status
sudo ufw allow 22/tcp
```

---

**Problem**: "Permission denied (password)" after key setup
```
Permission denied (publickey,password).
```

**Solutions**:
```bash
# 1. Verify SSH keys exist
ls ~/.ssh/id_ed25519*

# 2. Check permissions
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub

# 3. Verify key is on Pi
ssh salman@10.15.55.118 "cat ~/.ssh/authorized_keys"

# 4. Try password auth temporarily
ssh -o PubkeyAuthentication=no salman@10.15.55.118

# 5. Copy key again
ssh-copy-id -i ~/.ssh/id_ed25519.pub salman@10.15.55.118
```

---

### OpenClaw Gateway Issues

**Problem**: Gateway won't start
```
Failed to start OpenClaw gateway
```

**Solutions**:
```bash
# 1. Check service status
sudo systemctl status openclaw.service

# 2. View detailed logs
journalctl -u openclaw.service -n 50 -e

# 3. Check if port 18789 is in use
lsof -i :18789

# 4. Kill existing process
pkill -9 -f "openclaw"

# 5. Verify config syntax
jq . ~/.openclaw/openclaw.json > /dev/null && echo "Valid" || echo "Invalid JSON"

# 6. Restart fresh
sudo systemctl restart openclaw.service
```

---

**Problem**: Gateway running but not accessible
```
curl: (7) Failed to connect to 127.0.0.1 port 18789
```

**Solutions**:
```bash
# 1. Check service is actually running
ps aux | grep -i openclaw

# 2. Check port binding
netstat -tulpn | grep 18789

# 3. Check logs
journalctl -u openclaw.service -f

# 4. Try accessing from localhost
curl http://127.0.0.1:18789

# 5. If network access needed, update config
nano ~/.openclaw/openclaw.json
# Change: "bind": "0.0.0.0" (from "127.0.0.1")

# 6. Restart service
sudo systemctl restart openclaw.service

# 7. Test from another machine
curl http://10.15.55.118:18789
```

---

### Ollama Model Issues

**Problem**: Ollama service not responding
```
curl: (7) Failed to connect to 127.0.0.1 port 11434
```

**Solutions**:
```bash
# 1. Check Ollama is running
ps aux | grep -i ollama

# 2. Start Ollama if stopped
sudo systemctl start ollama

# 3. Check Ollama logs
journalctl -u ollama -f

# 4. Verify port binding
lsof -i :11434

# 5. Try direct test
ollama run llama3.2:1b "test"

# 6. Restart service
sudo systemctl restart ollama.service
```

---

**Problem**: "Model not found" error
```
Error: model "llama3.2:1b" not found
```

**Solutions**:
```bash
# 1. Check available models
ollama list

# 2. Check if model exists
curl http://127.0.0.1:11434/api/tags | jq '.models[] | .name'

# 3. Pull model again
ollama pull llama3.2:1b

# 4. Check model directory
ls -la ~/.ollama/models/

# 5. If pull fails, check disk space
df -h

# 6. If low on space, delete unused models
ollama rm qwen2.5:0.5b
ollama rm other:unused
```

---

**Problem**: Model pull extremely slow or stuck
```
pulling manifest
[stuck for hours]
```

**Solutions**:
```bash
# 1. Check download speed
speedtest-cli

# 2. Use background pull with timeout
timeout 300 ollama pull llama3.2:1b &

# 3. Monitor file size
watch -n 2 'du -sh ~/.ollama/models/'

# 4. Kill stuck pull
pkill -f "ollama pull"

# 5. Try smaller model
ollama pull qwen2.5:0.5b  # Faster, uses less space

# 6. Check internet connectivity
ping google.com -c 5
```

---

### Memory & Performance Issues

**Problem**: Out of Memory (OOM) error
```
Cannot allocate memory
Killed process
```

**Solutions**:
```bash
# 1. Check current memory
free -h

# 2. Check what's using memory
ps aux --sort=-%mem | head -10

# 3. Reduce model context window
nano ~/.openclaw/openclaw.json
# Change: "contextWindow": 2048 (from 8192)

# 4. Switch to smaller model
node -e "
const fs = require('fs');
const p = process.env.HOME + '/.openclaw/openclaw.json';
const j = JSON.parse(fs.readFileSync(p, 'utf8'));
j.agents.defaults.model.primary = 'ollama/qwen2.5:0.5b';
fs.writeFileSync(p, JSON.stringify(j, null, 2));
"

# 5. Kill background processes
sudo systemctl stop bluetooth
sudo systemctl stop avahi-daemon

# 6. Enable swap (risky on SD card)
sudo dphys-swapfile swapon

# 7. Restart services
sudo systemctl restart openclaw.service
```

---

**Problem**: Slow response time (>5 seconds per token)
```
Model inference taking too long
```

**Solutions**:
```bash
# 1. Monitor inference speed
journalctl -u openclaw.service -f | grep -i "duration\|token"

# 2. Check CPU temperature
vcgencmd measure_temp

# 3. If temp > 80°C, improve cooling
# - Add heatsink
# - Add case fan
# - Improve ventilation

# 4. Check free resources
htop  # Press 'q' to exit

# 5. Reduce model size
ollama pull qwen2.5:0.5b
openclaw config update agents.defaults.model.primary ollama/qwen2.5:0.5b

# 6. Reduce context window
nano ~/.openclaw/openclaw.json
# "contextWindow": 2048

# 7. Consider USB SSD for models
# Faster than SD card I/O
```

---

### WhatsApp Integration Issues

**Problem**: WhatsApp channel shows "stopped"
```
WhatsApp │ ON │ WARN │ gateway: Linked but stopped
```

**Solutions**:
```bash
# 1. Check WhatsApp config
cat ~/.openclaw/openclaw.json | jq '.channels.whatsapp'

# 2. Verify Twilio credentials
grep -E "accountSid|authToken" ~/.openclaw/openclaw.json

# 3. Test Twilio API
curl -u "ACCOUNT_SID:AUTH_TOKEN" \
  https://api.twilio.com/2010-04-01/Accounts

# 4. Check service logs for errors
journalctl -u openclaw.service -f | grep -i whatsapp

# 5. Rejoin Twilio sandbox (expires after 72 hours)
# Send "join [sandbox-code]" from WhatsApp again

# 6. Restart OpenClaw
sudo systemctl restart openclaw.service
```

---

**Problem**: Messages not being received
```
Send message on WhatsApp - no response
```

**Solutions**:
```bash
# 1. Check OpenClaw is processing messages
journalctl -u openclaw.service -f | grep -i "message\|whatsapp"

# 2. Check agent is working
curl http://127.0.0.1:18789/agent/status

# 3. Test model directly
ollama run llama3.2:1b "Hello!"

# 4. Check Twilio webhook
# Dashboard → Phone Numbers → WhatsApp Sandbox
# Verify webhook URL points to your Pi

# 5. Monitor message flow
journalctl -u openclaw.service -f --grep="whatsapp\|incoming\|outgoing"

# 6. Full restart sequence
sudo systemctl restart ollama.service
sleep 3
sudo systemctl restart openclaw.service
sleep 3
openclaw status
```

---

## FAQ

### Q: What's the minimum Raspberry Pi for this?

**A**: 
- **Minimum**: Pi 4 with 4GB RAM, microSD 64GB
- **Recommended**: Pi 4 with 8GB RAM, SSD 128GB
- **Not suitable**: Pi Zero, Pi 3B (insufficient RAM & CPU)

---

### Q: How much does this cost to run?

**A**:
- **Hardware**: ~$80-150 for Pi 4 + accessories (one-time)
- **Electricity**: ~$10-15/year (low power consumption)
- **Models**: FREE (all open-source)
- **Cloud APIs**: $0/month (runs locally)

**vs Cloud APIs**: Save $200-500/year compared to API calls

---

### Q: Can I run larger models (7B, 13B)?

**A**: Technically yes, but not practical on Pi 4:
```
7B model needs: 8GB+ RAM + slow inference (>2s/token)
13B model needs: 16GB+ RAM + very slow (>10s/token)

Better option: Use Pi 4 with 1-2B models for good balance
```

---

### Q: How do I update the models?

**A**:
```bash
# Update Ollama
sudo systemctl stop ollama
sudo apt update && sudo apt upgrade

# Pull newer model versions
ollama pull llama3.2:1b

# Switch to new version
nano ~/.openclaw/openclaw.json
# Update model ID if version changed
```

---

### Q: Can I access the dashboard from mobile?

**A**: Yes, if you:

**Option 1**: SSH Tunnel (safest)
```bash
# Establish tunnel from a desktop/laptop machine
ssh -L 18789:127.0.0.1:18789 salman@10.15.55.118 -N

# Then on mobile, connect to laptop's IP:
http://[laptop-ip]:18789
```

**Option 2**: Open Network Access (less safe)
```json
// In openclaw.json
"gateway": {
  "bind": "0.0.0.0",
  "requireAuth": true,
  "authToken": "strong-password-here"
}
```

Then access: `http://10.15.55.118:18789` from mobile on same WiFi

---

### Q: How do I backup my settings?

**A**:
```bash
# Backup local config
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup

# Full directory backup
tar -czf openclaw_backup.tar.gz ~/.openclaw/

# Copy to another machine
scp openclaw_backup.tar.gz user@desktop:/backups/

# Scheduled backup (cron)
# crontab -e
# 0 2 * * * tar -czf ~/openclaw_$(date +\%Y\%m\%d).tar.gz ~/.openclaw/
```

---

### Q: Can I run multiple agents?

**A**: Yes, OpenClaw supports multiple agents:

```json
{
  "agents": {
    "main": { "initialized": true },
    "assistant": { "initialized": true },
    "support": { "initialized": true }
  }
}
```

Each can have different models and configurations.

---

### Q: How do I secure my Pi?

**A**:

1. **SSH Key Auth**:
```bash
ssh-keygen -t ed25519
ssh-copy-id -i ~/.ssh/id_ed25519.pub salman@10.15.55.118

# Disable password auth on Pi
sudo nano /etc/ssh/sshd_config
# PasswordAuthentication no
sudo systemctl restart sshd
```

2. **Firewall**:
```bash
sudo ufw enable
sudo ufw allow 22/tcp  # SSH only
sudo ufw deny 18789    # Block remote gateway access
```

3. **Keep Updated**:
```bash
sudo apt update && sudo apt upgrade -y
```

---

### Q: Can I use this in production?

**A**: Yes, but with precautions:

```json
{
  "gateway": {
    "bind": "127.0.0.1",
    "requireAuth": true,
    "authToken": "$(STRONG_TOKEN_FROM_ENV)"
  },
  "agents": {
    "defaults": {
      "sandbox": { "mode": "all" }
    }
  },
  "tools": {
    "deny": ["group:web", "browser"]
  }
}
```

Plus:
- Use external reverse proxy (nginx/caddy)
- Enable TLS/SSL
- Set up monitoring
- Regular backups
- Log aggregation

---

### Q: What about data privacy?

**A**: Excellent:
- ✅ All inference happens locally
- ✅ No data leaves your network
- ✅ WhatsApp messages processed locally only
- ✅ No cloud API calls for AI
- ⚠️ Still use HTTPS/SSH for security

---

### Q: How do I monitor system health?

**A**:
```bash
# All-in-one status check
echo "=== Memory ===" && free -h && \
echo "=== Disk ===" && df -h && \
echo "=== Temp ===" && vcgencmd measure_temp && \
echo "=== CPU ===" && ps aux | head -5

# Continuous monitoring
watch -n 5 'echo "=== OpenClaw ==="; systemctl status openclaw.service; echo "=== Ollama ==="; systemctl status ollama'

# Log monitoring
journalctl -u openclaw.service -u ollama -f --tail=50
```

---

### Q: Can I move this to another Pi?

**A**: Yes:

```bash
# On old Pi
tar -czf openclaw_backup.tar.gz ~/.openclaw/
scp openclaw_backup.tar.gz new_pi:/home/salman/

# On new Pi
cd ~
tar -xzf openclaw_backup.tar.gz
sudo systemctl restart openclaw.service
```

Models still need to be re-pulled (too large to backup easily).

---

## Support Resources

- **OpenClaw Docs**: https://docs.openclaw.ai
- **Ollama Library**: https://ollama.ai/library
- **Twilio WhatsApp**: https://www.twilio.com/whatsapp
- **Raspberry Pi**: https://www.raspberrypi.com/documentation/
- **GitHub Issues**: Report problems in the repo

---

**Troubleshooting Guide v1.0** | Last Updated: April 2026
