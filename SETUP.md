# Setup Guide

This guide walks through setting up the Daily Briefing dashboard from scratch on a Windows PC with an NVIDIA GPU.

## Prerequisites

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| GPU VRAM | 16GB | 32GB (RTX 5090) |
| System RAM | 32GB | 64GB |
| OS | Windows 10 64-bit | Windows 11 |
| Storage | 50GB free | 100GB free |

---

## Part 1 — Install Ollama

Ollama is the local LLM inference backend.

**1. Download and install Ollama for Windows:**

Go to [ollama.com/download](https://ollama.com/download) and run the installer. Ollama adds itself to PATH and starts as a background service.

**2. Enable Flash Attention for better performance:**

Open PowerShell as Administrator:

```powershell
[System.Environment]::SetEnvironmentVariable("OLLAMA_FLASH_ATTENTION","1","Machine")
[System.Environment]::SetEnvironmentVariable("OLLAMA_ORIGINS","*","Machine")
[System.Environment]::SetEnvironmentVariable("OLLAMA_HOST","0.0.0.0:11434","Machine")
```

Restart Ollama from the system tray (right-click → Quit, then relaunch).

**3. Pull your model:**

```powershell
# Best all-round model for 32GB VRAM
ollama pull qwen3:32b

# Faster MoE variant (optional)
ollama pull qwen3:30b-a3b

# Reasoning specialist (optional)
ollama pull deepseek-r1:32b
```

Each model is roughly 18–20GB. Downloads resume automatically if interrupted.

**4. Verify GPU acceleration:**

```powershell
ollama run qwen3:32b "Hello"
```

Open Task Manager → Performance → GPU while it responds. GPU memory should jump to ~20GB. If it stays at zero, check your NVIDIA drivers (550+ required).

**5. Set Ollama to start automatically on boot:**

```powershell
Set-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "Ollama" -Value "$env:LOCALAPPDATA\Programs\Ollama\ollama.exe"
```

---

## Part 2 — Install Docker Desktop

Docker runs SearXNG, Open WebUI, and the nginx web server as containers.

**1. Download Docker Desktop** from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop) and install it.

**2. Enable auto-start:**

Open Docker Desktop → Settings → General → check **Start Docker Desktop when you log in**.

**3. Verify Docker is running:**

```powershell
docker --version
docker ps
```

---

## Part 3 — Set Up SearXNG (Private Search Engine)

SearXNG is the self-hosted search engine the dashboard uses to fetch live news.

**1. Create a config file:**

```powershell
New-Item -ItemType Directory -Path "$HOME\searxng-config" -Force

@"
use_default_settings: true

server:
  secret_key: "replacethiswithalongrandomstring"
  limiter: false
  image_proxy: true

search:
  safe_search: 0
  autocomplete: ""
  default_lang: "auto"
  formats:
    - html
    - json
"@ | Out-File -FilePath "$HOME\searxng-config\settings.yml" -Encoding utf8
```

**2. Start SearXNG:**

```powershell
docker run -d `
  --name searxng `
  -p 8080:8080 `
  --add-host=host.docker.internal:host-gateway `
  --restart unless-stopped `
  -v "${HOME}\searxng-config:/etc/searxng" `
  searxng/searxng:latest
```

**3. Verify it's running:**

Visit `http://localhost:8080` in your browser. You should see the SearXNG search page.

---

## Part 4 — Set Up Open WebUI (Chat Interface)

Open WebUI gives you a ChatGPT-style interface for your local models.

```powershell
docker run -d `
  -p 3000:8080 `
  --add-host=host.docker.internal:host-gateway `
  -v open-webui:/app/backend/data `
  --name open-webui `
  --restart always `
  ghcr.io/open-webui/open-webui:main
```

Visit `http://localhost:3000` and create your admin account. Your Ollama models will appear in the model dropdown automatically.

**Connect SearXNG for web search:**

In Open WebUI: Admin Panel → Settings → Web Search → Enable → set engine to `searxng` → set URL to `http://host.docker.internal:8080/search?q=<query>` → Save.

---

## Part 5 — Set Up the Dashboard Web Server

The dashboard is served by nginx, which also acts as a reverse proxy for SearXNG and Ollama to solve browser CORS restrictions.

**1. Create the nginx config:**

```powershell
$config = "server {`n    listen 80;`n`n    resolver 127.0.0.11 ipv4=on;`n`n    location / {`n        root /usr/share/nginx/html;`n        index index.html;`n        add_header Access-Control-Allow-Origin *;`n    }`n`n    location /searxng/ {`n        proxy_pass http://host.docker.internal:8080/;`n        proxy_set_header Host host.docker.internal:8080;`n        add_header Access-Control-Allow-Origin *;`n    }`n`n    location /ollama/ {`n        proxy_pass http://host.docker.internal:11434/;`n        add_header Access-Control-Allow-Origin *;`n    }`n}`n"
[System.IO.File]::WriteAllText("$HOME\nginx.conf", $config, [System.Text.UTF8Encoding]::new($false))
```

**2. Copy the dashboard HTML to your home folder:**

Save `morning-news.html` to `C:\Users\<yourname>\morning-news.html`.

**3. Add your wallpaper (optional):**

Save any landscape image as `singaporeskyline.jpg` in `C:\Users\<yourname>\`. The dashboard uses it as a background. If you use a different filename, update the CSS in `morning-news.html` accordingly.

**4. Start the nginx server:**

```powershell
docker run -d `
  --name news-server `
  -p 5500:80 `
  --add-host=host.docker.internal:host-gateway `
  -v "${HOME}:/usr/share/nginx/html:ro" `
  -v "${HOME}\nginx.conf:/etc/nginx/conf.d/default.conf:ro" `
  --restart unless-stopped `
  nginx:alpine
```

**5. Open the dashboard:**

Visit `http://localhost:5500/morning-news.html` in Firefox.

**6. Set as Firefox homepage:**

Firefox → Settings → Home → Homepage and new windows → Custom URLs → paste `http://localhost:5500/morning-news.html`.

---

## Part 6 — Open Firewall Ports (for remote access)

If you want to access the dashboard from other devices on the same network:

```powershell
New-NetFirewallRule -DisplayName "Daily Briefing" -Direction Inbound -Protocol TCP -LocalPort 5500 -Action Allow
New-NetFirewallRule -DisplayName "SearXNG" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow
New-NetFirewallRule -DisplayName "Ollama" -Direction Inbound -Protocol TCP -LocalPort 11434 -Action Allow
```

---

## Part 7 — Remote Access with Tailscale (Optional)

Tailscale lets you access the dashboard from your phone or laptop securely from anywhere.

**1. Install Tailscale** from [tailscale.com/download](https://tailscale.com/download) on your PC and any other devices. Sign in with the same account on all devices.

**2. Fix Tailscale startup issues on Windows:**

```powershell
sc.exe config Tailscale start= delayed-auto
```

**3. Get your Tailscale IP:**

```powershell
tailscale ip -4
```

**4. Access from other devices:**

On any device connected to your Tailscale network, open:
```
http://<your-tailscale-ip>:5500/morning-news.html
```

> **Note:** If your Tailscale IP changes after a reboot, the nginx config using `host.docker.internal` still works for local PC access. Only the remote URL needs updating, which you can check with `tailscale ip -4`.

---

## Part 8 — Keep the Model Warm on Startup

After a reboot, Ollama needs to load the model into VRAM before the dashboard can use it. Add a startup script to pre-load it automatically:

```powershell
@"
Start-Sleep -Seconds 45
try {
    Invoke-RestMethod -Uri 'http://localhost:11434/api/generate' -Method POST -ContentType 'application/json' -Body '{"model":"qwen3:32b","prompt":"hi","stream":false}' -TimeoutSec 120 | Out-Null
} catch {}
"@ | Out-File -FilePath "$HOME\warmup-ollama.ps1" -Encoding utf8

$shell = New-Object -ComObject WScript.Shell
$shortcut = $shell.CreateShortcut("$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\WarmupOllama.lnk")
$shortcut.TargetPath = "powershell.exe"
$shortcut.Arguments = "-WindowStyle Hidden -ExecutionPolicy Bypass -File `"$HOME\warmup-ollama.ps1`""
$shortcut.WindowStyle = 7
$shortcut.Save()
```

This runs silently 45 seconds after login and pre-loads the model so your dashboard is ready immediately when you open it.

---

## Verifying Everything Works

Run this checklist after setup or after a reboot:

```powershell
# Check all containers are running
docker ps

# Check Ollama is responding
Invoke-WebRequest -Uri "http://localhost:11434" -UseBasicParsing | Select-Object StatusCode

# Check SearXNG is responding
Invoke-WebRequest -Uri "http://localhost:8080/search?q=test&format=json" -UseBasicParsing | Select-Object StatusCode

# Check the dashboard is serving
Invoke-WebRequest -Uri "http://localhost:5500/morning-news.html" -UseBasicParsing | Select-Object StatusCode
```

All three should return `StatusCode: 200`.

---

## Troubleshooting

**Dashboard shows "Could not load stories"**
- Run `docker ps` and confirm all containers show `Up`
- Open Firefox console (F12 → Console) and check for red errors
- Run `ollama ps` to confirm the model is loaded

**GPU not being used (Task Manager shows 0% GPU)**
- Run `ollama run qwen3:32b "hello"` to manually warm up the model
- Check NVIDIA driver version with `nvidia-smi` — must be 550 or newer

**Tailscale stuck on "Starting..."**
- Run `sc.exe config Tailscale start= delayed-auto` as Administrator
- If that fails: `Stop-Service Tailscale`, delete `C:\ProgramData\Tailscale\server-state.conf`, then `Start-Service Tailscale` and run `tailscale up` to re-authenticate

**Prayer times unavailable**
- The aladhan.com API occasionally goes down briefly — hit Refresh to retry
- Check your internet connection is active

**News loads but articles are old**
- SearXNG's `time_range=day` filter requires search engines to have indexed recent content
- Hit Refresh — different search engine results are returned each time
