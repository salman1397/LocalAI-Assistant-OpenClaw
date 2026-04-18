# ⚙️ Configuration Examples & Architecture

## Complete OpenClaw Configuration File

Full example of `~/.openclaw/openclaw.json` with all sections:

```json
{
  "version": "2026.4.12",
  "gateway": {
    "bind": "127.0.0.1",
    "port": 18789,
    "trustedProxies": [],
    "requireAuth": false,
    "authToken": "",
    "memoryLimit": "512MB"
  },
  "agents": {
    "main": {
      "description": "Main conversational agent",
      "initialized": true
    },
    "defaults": {
      "model": {
        "primary": "ollama/llama3.2:1b",
        "fallback": "ollama/qwen2.5:1.5b",
        "temperature": 0.7,
        "topP": 0.9,
        "topK": 40,
        "contextWindow": 8192,
        "maxTokens": 2048,
        "timeout": 60000
      },
      "sandbox": {
        "mode": "off",
        "restricted": false
      }
    }
  },
  "models": {
    "providers": {
      "ollama": {
        "enabled": true,
        "baseUrl": "http://127.0.0.1:11434",
        "api": "ollama",
        "apiKey": "OLLAMA_API_KEY",
        "timeout": 120000,
        "retries": 3,
        "models": [
          {
            "id": "llama3.2:1b",
            "name": "Llama 3.2 1B",
            "provider": "ollama",
            "reasoning": false,
            "multimodal": false,
            "input": ["text"],
            "output": ["text"],
            "cost": {
              "input": 0,
              "output": 0,
              "cached": 0
            },
            "contextWindow": 8192,
            "maxTokens": 4096,
            "description": "Lightweight 1.2B parameter model, fast inference"
          },
          {
            "id": "qwen2.5:1.5b",
            "name": "Qwen 2.5 1.5B",
            "provider": "ollama",
            "reasoning": false,
            "input": ["text"],
            "output": ["text"],
            "cost": {
              "input": 0,
              "output": 0
            },
            "contextWindow": 32768,
            "maxTokens": 8192,
            "description": "Versatile 1.5B model with larger context window"
          },
          {
            "id": "qwen2.5:0.5b",
            "name": "Qwen 2.5 0.5B",
            "provider": "ollama",
            "reasoning": false,
            "input": ["text"],
            "output": ["text"],
            "cost": {
              "input": 0,
              "output": 0
            },
            "contextWindow": 2048,
            "maxTokens": 1024,
            "description": "Ultra-lightweight 494M model for resource-constrained systems"
          }
        ]
      }
    }
  },
  "channels": {
    "whatsapp": {
      "enabled": false,
      "provider": "twilio",
      "config": {
        "accountSid": "ACxxxxxxxxxxxxxxxxxxxxxx",
        "authToken": "your_auth_token_here",
        "phoneNumber": "+1234567890",
        "webhookUrl": "http://10.15.55.118:18789/webhooks/whatsapp"
      }
    },
    "web": {
      "enabled": true,
      "endpoint": "/chat",
      "publicUrl": "http://127.0.0.1:18789"
    }
  },
  "logging": {
    "level": "info",
    "format": "json",
    "outputs": [
      "console",
      "file"
    ],
    "fileRotation": {
      "maxFiles": 10,
      "maxSize": "10MB"
    }
  },
  "security": {
    "cors": {
      "enabled": true,
      "origin": ["http://127.0.0.1:18789", "http://localhost:18789"]
    },
    "rateLimiting": {
      "enabled": true,
      "windowMs": 60000,
      "maxRequests": 100
    }
  },
  "tools": {
    "allow": ["all"],
    "deny": []
  },
  "memory": {
    "plugin": "memory-core",
    "storage": "~/.openclaw/memory",
    "enabled": true
  }
}
```

---

## Configuration Scenarios

### Scenario 1: Ultra-Light Setup (4GB RAM)

For Raspberry Pi 4 with 4GB RAM, minimal overhead:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "ollama/qwen2.5:0.5b",
        "contextWindow": 2048,
        "maxTokens": 512
      }
    }
  },
  "gateway": {
    "bind": "127.0.0.1",
    "port": 18789
  }
}
```

**Benefits**: Minimal memory usage (~1.5GB), fast responses (~100ms/token)  
**Trade-off**: Lower quality responses, smaller context window

---

### Scenario 2: Balanced Setup (4GB RAM, Recommended)

Good balance of quality and performance:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "ollama/llama3.2:1b",
        "temperature": 0.7,
        "contextWindow": 4096,
        "maxTokens": 2048
      }
    }
  },
  "gateway": {
    "bind": "127.0.0.1",
    "port": 18789
  }
}
```

**Benefits**: Good quality, reasonable speed (~200ms/token), stable  
**Trade-off**: Moderate memory usage (~2.5-3GB)

---

### Scenario 3: Performance Setup (8GB RAM+)

Maximum quality with larger models:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "ollama/llama2:7b-chat-q4",
        "temperature": 0.7,
        "contextWindow": 4096,
        "maxTokens": 2048,
        "timeout": 120000
      }
    }
  },
  "gateway": {
    "bind": "0.0.0.0",
    "port": 18789,
    "trustedProxies": ["192.168.1.0/24"]
  }
}
```

**Benefits**: Excellent response quality, large context window  
**Trade-off**: Requires 8GB+ RAM, slower inference (~1-2s/token)

---

### Scenario 4: Network-Accessible Setup

Expose dashboard to local network:

```json
{
  "gateway": {
    "bind": "0.0.0.0",
    "port": 18789,
    "requireAuth": true,
    "authToken": "your-secret-token-12345"
  },
  "security": {
    "cors": {
      "enabled": true,
      "origin": ["http://10.15.55.118:18789", "http://192.168.1.*"]
    },
    "rateLimiting": {
      "enabled": true,
      "maxRequests": 50
    }
  }
}
```

**Access**: `http://10.15.55.118:18789` from any machine on network  
**Security**: Token-based auth required

---

### Scenario 5: Production Deployment

Hardened configuration for production:

```json
{
  "version": "2026.4.12",
  "gateway": {
    "bind": "127.0.0.1",
    "port": 18789,
    "requireAuth": true,
    "authToken": "$(OPENCLAW_API_TOKEN)",
    "tlsEnabled": true,
    "tlsCert": "/etc/openclaw/cert.pem",
    "tlsKey": "/etc/openclaw/key.pem"
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "ollama/llama3.2:1b",
        "contextWindow": 4096,
        "maxTokens": 2048
      },
      "sandbox": {
        "mode": "all"
      }
    }
  },
  "logging": {
    "level": "warn",
    "outputs": ["file"],
    "fileRotation": {
      "maxFiles": 30,
      "maxSize": "50MB"
    }
  },
  "security": {
    "cors": {
      "enabled": false
    },
    "rateLimiting": {
      "enabled": true,
      "maxRequests": 30
    }
  },
  "tools": {
    "deny": ["group:web", "browser"]
  }
}
```

**Features**: TLS encryption, token auth, comprehensive logging, tool restrictions

---

## SSH Configuration Examples

### ~/.ssh/config

Create `~/.ssh/config` on your local machine for easy Pi access:

```bash
# Windows: C:\Users\YourUsername\.ssh\config
# Mac/Linux: ~/.ssh/config

Host pi
    HostName 10.15.55.118
    User salman
    IdentityFile ~/.ssh/id_ed25519
    Port 22
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel QUIET

Host pi-tunnel
    HostName 10.15.55.118
    User salman
    IdentityFile ~/.ssh/id_ed25519
    LocalForward 18789 127.0.0.1:18789
    LocalForward 11434 127.0.0.1:11434
    -N -f -q
```

**Usage**:
```bash
# Simple SSH
ssh pi

# With tunnel
ssh pi-tunnel

# SCP with shortname
scp pi:~/file.txt ~/
```

---

## System Architecture

### Data Flow

```
User Input (WhatsApp/Web)
    ↓
OpenClaw Gateway (Port 18789)
    ├─ Request validation
    ├─ Authentication check
    ├─ Rate limiting
    ├─ Session management
    ↓
Agent Router
    ├─ Model selection (ollama/llama3.2:1b)
    ├─ Context assembly
    ├─ Tool availability check
    ↓
Ollama Runtime (Port 11434)
    ├─ Model loading (if not cached)
    ├─ Tokenization
    ├─ Inference (GPU/CPU)
    ├─ Output decoding
    ↓
Post-Processing
    ├─ Safety checks
    ├─ Formatting
    ├─ Response assembly
    ↓
Output (WhatsApp/Web)
```

### Storage Structure

```
Raspberry Pi Filesystem
├── /home/salman/
│   ├── .openclaw/                 ← OpenClaw config & state
│   │   ├── openclaw.json          ← Main configuration
│   │   ├── agents/
│   │   │   ├── main/
│   │   │   │   ├── sessions/      ← Chat session data
│   │   │   │   └── plugins/       ← Agent plugins
│   │   ├── memory/                ← Persistent memory
│   │   └── logs/                  ← OpenClaw logs
│   ├── .ollama/                   ← Ollama data
│   │   ├── models/                ← Downloaded models
│   │   │   ├── llama3.2_1b/
│   │   │   ├── qwen2.5_1.5b/
│   │   │   └── ...
│   │   └── cache/
│   └── .npm-global/
│       └── lib/node_modules/      ← npm packages
├── /var/log/
│   ├── syslog
│   ├── auth.log
│   └── openclaw.log               ← OpenClaw logs (if enabled)
└── /etc/systemd/system/
    ├── openclaw.service
    └── ollama.service
```

### Memory Usage Profile

#### Llama 3.2 1B

```
System baseline:          ~200-300 MB
Ollama process:           ~500-700 MB
Model loading:            +1200 MB (one-time)
Inference context:        +200-400 MB (per request)
OpenClaw process:         ~100-200 MB
─────────────────────────────────────
Total at idle:            ~1.5 GB
Peak during inference:    ~2.5-3 GB
```

#### Qwen 2.5 0.5B

```
System baseline:          ~200-300 MB
Ollama process:           ~400-500 MB
Model loading:            +500 MB
Inference context:        +100-200 MB
OpenClaw process:         ~100-200 MB
─────────────────────────────────────
Total at idle:            ~1.0 GB
Peak during inference:    ~1.5-2 GB
```

---

## Environment Variables

### OpenClaw

```bash
# Set custom OpenClaw home
export OPENCLAW_HOME=/custom/path/.openclaw

# Set API token for security
export OPENCLAW_API_TOKEN=my-secret-token

# Set log level
export OPENCLAW_LOG_LEVEL=debug

# Enable verbose output
export OPENCLAW_VERBOSE=1
```

### Ollama

```bash
# Set custom Ollama models directory
export OLLAMA_MODELS=/mnt/ssd/models

# Set CPU/GPU usage
export OLLAMA_CPU_THREADS=4
export OLLAMA_GPU=0  # Disable GPU if having issues

# Set keep-alive timeout
export OLLAMA_KEEP_ALIVE=5m

# Bind to specific address
export OLLAMA_HOST=127.0.0.1:11434
```

---

## Network Diagram

```
                    Internet
                        ↓
            ┌───────────────────────┐
            │   Your Local Network   │
            │    (192.168.1.0/24)    │
            └───────────────────────┘
                        ↓
        ┌───────────────────────────────┐
        │  Raspberry Pi 4 (10.15.55.118) │
        ├───────────────────────────────┤
        │  ┌─────────────────────────┐  │
        │  │   OpenClaw Gateway      │  │
        │  │   Port 18789            │  │
        │  │  Bind: 127.0.0.1 (safe) │  │
        │  └───────────┬─────────────┘  │
        │              ↓                 │
        │  ┌─────────────────────────┐  │
        │  │  Ollama Runtime         │  │
        │  │  Port 11434             │  │
        │  │  (Local models)         │  │
        │  └─────────────────────────┘  │
        │              ↓                 │
        │  ┌─────────────────────────┐  │
        │  │  Local Models (1GB-2GB) │  │
        │  │  - llama3.2:1b          │  │
        │  │  - qwen2.5:1.5b         │  │
        │  └─────────────────────────┘  │
        └───────────────────────────────┘
                        ↑
        ┌───────────────────────────────┐
        │    Twilio WhatsApp API        │
        │  (Optional, for messaging)    │
        └───────────────────────────────┘
```

---

## Comparison: Local vs Cloud Models

| Aspect | Local (Ollama) | Cloud (API) |
|--------|----------------|-----------|
| **Cost** | Free | $0.01-0.10 per req |
| **Latency** | 100-500ms | 500-2000ms |
| **Privacy** | 100% local | Data sent to cloud |
| **Availability** | Always on | Depends on internet |
| **Customization** | Full control | Limited |
| **Quality** | Good (1-2B) | Excellent (Large) |
| **Setup** | Complex | Simple |

---

**Configuration Guide v1.0** | April 2026
