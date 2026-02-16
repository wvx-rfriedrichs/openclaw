Secure OpenClaw - Claude Code Instructions (Windows 11 Pro)
You are hardening an existing OpenClaw installation running via Docker Desktop on Windows 11 Pro.

Prerequisites: Docker Desktop for Windows must be installed and running with WSL 2 backend.

Goal: Make minimal changes to secure the setup while keeping full functionality (onboarding wizard, dashboard, skills, API keys all still work).
What You Need From User
Cloudflare Tunnel Token - user must create tunnel first (instructions below)
Domain they configured in Cloudflare (e.g., ai.example.com)

Note: You MUST verify OPENCLAW_GATEWAY_BIND=lan is in the .env file (see Step 2.5). Hostinger's template omits this, causing tunnel 502 errors.
Tell User to Create Cloudflare Tunnel First
Create a Cloudflare Tunnel before we continue:

Go to Cloudflare Zero Trust Dashboard
Navigate to Networks → Tunnels → Create a tunnel
Choose Cloudflared, name it (e.g., openclaw)
Copy the tunnel token (the long eyJ... string)
Add a Public Hostname:
Subdomain: ai (or your choice)
Domain: your domain
Service Type: HTTP
URL: localhost:18789 (must match OPENCLAW_GATEWAY_PORT in your .env file)
Save the tunnel

Give me the token and domain.


Step 1: Find the OpenClaw Installation
The project is at C:\projects\docker\openclaw. Verify with PowerShell:

```powershell
# Check Docker containers
docker ps -a | Select-String "openclaw"

# Verify docker-compose.yml exists
Get-ChildItem C:\projects\docker\openclaw\docker-compose.yml
```

Note the container names for the following steps. The gateway service is named `openclaw-gateway`.


Step 2: Patch docker-compose.yml
The key security change is binding the ports to 127.0.0.1 only. This prevents direct access from the network.

IMPORTANT:

DO NOT change OPENCLAW_GATEWAY_BIND from lan - keep it as lan
DO NOT change allowInsecureAuth to false - keep it as true
The security comes from the port binding to 127.0.0.1, which restricts access to localhost only
Cloudflare Tunnel connects via HTTP internally, so allowInsecureAuth: true is required

The docker-compose.yml has two services: `openclaw-gateway` and `openclaw-cli`.
Edit the `openclaw-gateway` service in docker-compose.yml:

Change 1: Port binding (find the ports section in openclaw-gateway)

```yaml
# FROM:
    ports:
      - "${OPENCLAW_GATEWAY_PORT:-18789}:18789"
      - "${OPENCLAW_BRIDGE_PORT:-18790}:18790"

# TO:
    ports:
      - "127.0.0.1:${OPENCLAW_GATEWAY_PORT:-18789}:18789"
      - "127.0.0.1:${OPENCLAW_BRIDGE_PORT:-18790}:18790"
```

Change 2: Add Docker socket for sandboxing (find the volumes section in openclaw-gateway)

```yaml
# FROM:
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace

# TO:
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
      - /var/run/docker.sock:/var/run/docker.sock
```

Note: Docker Desktop for Windows automatically translates `/var/run/docker.sock` to the Windows named pipe. This path works as-is in docker-compose.yml.

Change 3: Convert the command to a shell script for sandbox setup

The original command is a JSON array:
```yaml
    command:
      [
        "node",
        "dist/index.js",
        "gateway",
        "--bind",
        "${OPENCLAW_GATEWAY_BIND:-lan}",
        "--port",
        "18789",
      ]
```

Replace it with a shell script that sets up the sandbox before starting the gateway:
```yaml
    command:
      - sh
      - -c
      - |
        # Add node user to docker group for sandbox support
        DOCKER_GID=$$(stat -c '%g' /var/run/docker.sock 2>/dev/null || echo "")
        if [ -n "$$DOCKER_GID" ]; then
          groupadd -g $$DOCKER_GID docker 2>/dev/null || true
          usermod -aG docker node 2>/dev/null || true
        fi

        # Install Docker CLI if not present
        if ! command -v docker >/dev/null 2>&1; then
          echo "Installing Docker CLI..."
          curl -fsSL https://download.docker.com/linux/static/stable/x86_64/docker-27.4.1.tgz -o /tmp/docker.tgz
          tar -xzf /tmp/docker.tgz -C /tmp
          cp /tmp/docker/docker /usr/local/bin/
          chmod +x /usr/local/bin/docker
          rm -rf /tmp/docker /tmp/docker.tgz
        fi

        # Build sandbox image if not present
        if ! docker image inspect openclaw-sandbox:bookworm-slim >/dev/null 2>&1; then
          echo "Building sandbox image..."
          cd /app && ./scripts/sandbox-setup.sh
        fi

        # Add node user to linuxbrew group for skill installation
        groupadd -g 1001 linuxbrew 2>/dev/null || true
        usermod -aG linuxbrew node 2>/dev/null || true

        # Start the gateway as node user
        exec su node -c "node dist/index.js gateway --bind $${OPENCLAW_GATEWAY_BIND:-lan} --port 18789"
```

IMPORTANT: Because we're replacing the JSON array command with a shell script, the container will now start as root (to run setup), then drop to the `node` user via `exec su node`. The `user:` directive should NOT be set in docker-compose.yml for this service.

Why this is needed:

The OpenClaw container doesn't include Docker CLI by default
The node user needs to be in the docker group to access the socket
The sandbox requires the openclaw-sandbox:bookworm-slim image to be built
These steps run at container startup and only execute if needed (idempotent)


Step 2.5: Configure .env File (CRITICAL)
This step is essential for Cloudflare Tunnel to work. The Hostinger template does NOT include OPENCLAW_GATEWAY_BIND by default, so it defaults to loopback mode which rejects HTTP connections from the tunnel.

Check the .env file in the OpenClaw directory:

```powershell
Get-Content C:\projects\docker\openclaw\.env
```

Ensure these settings exist:

```
OPENCLAW_GATEWAY_PORT=18789             # (or whatever port you chose)
OPENCLAW_GATEWAY_BIND=lan               # CRITICAL - must be "lan", NOT "loopback"
OPENCLAW_GATEWAY_TOKEN=xxx              # Your gateway token
OPENCLAW_CONFIG_DIR=/c/Users/<username>/.openclaw
OPENCLAW_WORKSPACE_DIR=/c/Users/<username>/.openclaw/workspace
```

Note: The config directory paths use Docker-format paths (forward slashes, `/c/` instead of `C:\`). The actual Windows path is `C:\Users\<username>\.openclaw`.

If OPENCLAW_GATEWAY_BIND is missing, add it:

```powershell
Add-Content C:\projects\docker\openclaw\.env "OPENCLAW_GATEWAY_BIND=lan"
```

Why this matters:

loopback mode (the default when not set) causes OpenClaw to reject HTTP connections, even from localhost
The Cloudflare Tunnel connects via HTTP internally (even though external access is HTTPS)
Without OPENCLAW_GATEWAY_BIND=lan, you'll see "connection reset by peer" errors in tunnel logs and 502 errors in the browser
This is the #1 cause of "tunnel is running but dashboard won't load" issues


Step 3: Update openclaw.json Security Settings
On Windows, the config file is directly accessible on the host filesystem. The path is whatever OPENCLAW_CONFIG_DIR maps to in Windows format.

For example, if `.env` has `OPENCLAW_CONFIG_DIR=/c/Users/Ralf Friedrichs/.openclaw`, the Windows path is:

```
C:\Users\Ralf Friedrichs\.openclaw\openclaw.json
```

Read the current config:

```powershell
Get-Content "$env:USERPROFILE\.openclaw\openclaw.json" | ConvertFrom-Json | ConvertTo-Json -Depth 10
```

Merge these security settings into openclaw.json:

```json
{
  "discovery": {
    "mdns": {
      "mode": "off"
    }
  },
  "logging": {
    "redactSensitive": "tools"
  },
  "channels": {
    "whatsapp": {
      "dmPolicy": "allowlist",
      "groups": {
        "*": {
          "requireMention": true
        }
      }
    },
    "telegram": {
      "dmPolicy": "allowlist"
    }
  },
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main",
        "scope": "session",
        "workspaceAccess": "rw",
        "docker": {
          "network": "bridge"
        }
      }
    }
  },
  "tools": {
    "elevated": {
      "enabled": false
    }
  }
}
```

IMPORTANT:

Only WhatsApp and Telegram support dmPolicy - DO NOT add it for Discord or Slack (causes config validation errors)
dmPolicy: "allowlist" means only the user can message the bot (strangers are completely ignored)
User accounts are automatically added to the allowlist when connected via onboarding
The sandbox section requires the Docker socket mount from Step 2


Step 4: Install Cloudflare Tunnel
On Windows, install cloudflared using winget (recommended) or download the MSI:

Option A - winget (recommended):
```powershell
winget install cloudflare.cloudflared
```

Option B - Direct download:
Download the latest Windows installer from the Cloudflare GitHub releases page and run it.

Install as a Windows service (run PowerShell as Administrator):

```powershell
cloudflared service install <TUNNEL_TOKEN>
```

Verify the service is running:

```powershell
Get-Service cloudflared
```

If the service isn't started:
```powershell
Start-Service cloudflared
```


Step 4.5: Install Homebrew for Skills
OpenClaw uses Homebrew to install skills via the onboarding wizard. Without Homebrew, skill installation will fail.

On Windows with Docker Desktop, you cannot directly mount a host Homebrew directory into the Linux container. Instead, use a named Docker volume and install Homebrew at container startup.

Step 1: Add a named volume for Homebrew in docker-compose.yml

Add to the `openclaw-gateway` volumes section:
```yaml
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
      - /var/run/docker.sock:/var/run/docker.sock
      - homebrew_data:/home/linuxbrew/.linuxbrew    # <-- Add this
```

And add the named volume at the bottom of docker-compose.yml:
```yaml
volumes:
  homebrew_data:
```

Step 2: Add Homebrew installation to the command section

Add these lines to the shell script command (before the `exec su node` line, after the linuxbrew group setup):

```sh
        # Install Homebrew if not present
        if [ ! -f /home/linuxbrew/.linuxbrew/bin/brew ]; then
          echo "Installing Homebrew..."
          # Create linuxbrew user inside container
          useradd -m -s /bin/bash -u 1001 -g 1001 linuxbrew 2>/dev/null || true
          chown -R linuxbrew:linuxbrew /home/linuxbrew/.linuxbrew
          su linuxbrew -c 'NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"'
        fi
        # Ensure linuxbrew user exists (for subsequent starts)
        useradd -m -s /bin/bash -u 1001 -g 1001 linuxbrew 2>/dev/null || true
        chown -R linuxbrew:linuxbrew /home/linuxbrew/.linuxbrew 2>/dev/null || true
```

The container's PATH already includes /home/linuxbrew/.linuxbrew/bin, so skills will work after installation.

Why this is needed:

The onboarding wizard uses brew install to add skills
Without Homebrew, you'll see "brew: command not found" errors
Skills provide additional capabilities (web search, file handling, etc.)
A named Docker volume persists Homebrew across container restarts


Step 5: Restart OpenClaw
```powershell
cd C:\projects\docker\openclaw
docker compose down
docker compose up -d

# Wait for startup
Start-Sleep -Seconds 15

# Check it's running
docker ps | Select-String "openclaw"
```


Step 6: Verify Security
Run these checks in PowerShell:

```powershell
Write-Host "=== Security Check ==="

# Port should be localhost only
$containerId = docker ps -q --filter "name=openclaw-gateway"
$portOutput = docker port $containerId
if ($portOutput -match "127\.0\.0\.1") {
    Write-Host "Port bound to localhost: YES"
} else {
    Write-Host "Port bound to localhost: NO"
}

# Tunnel should be running
$svc = Get-Service cloudflared -ErrorAction SilentlyContinue
if ($svc -and $svc.Status -eq 'Running') {
    Write-Host "Cloudflare Tunnel: Running"
} else {
    Write-Host "Cloudflare Tunnel: NOT running"
}

# Test public access via tunnel (replace with your domain)
# curl -sf -o NUL "https://YOUR_DOMAIN/"

# Direct access should fail from outside
Write-Host "Direct port not exposed: YES (bound to 127.0.0.1 only)"
```


What Changed (Tell User)
| Before | After |
|--------|-------|
| Port exposed to network (0.0.0.0) | Port only on localhost (127.0.0.1) |
| Anyone can connect directly | Access only via Cloudflare Tunnel |
| No sandbox | Tools run in isolated Docker containers |
| Anyone can DM | Only you can DM (allowlist policy for WhatsApp/Telegram) |
| mDNS broadcasts server info | Discovery disabled |
| Sensitive data in logs | Logs redacted |


Note on allowInsecureAuth: This must stay true because Cloudflare Tunnel connects via HTTP internally (even though HTTPS is used externally). This is still secure because the port is bound to 127.0.0.1 - only the tunnel can reach it.

What still works:

Dashboard (via your Cloudflare domain)
Onboarding wizard (see command below)
Adding skills via Homebrew
API key management
All channels (WhatsApp, Telegram, etc.)
All tools and features

Access OpenClaw at: https://YOUR_DOMAIN (replace with your Cloudflare domain)

Run onboarding (IMPORTANT: use -u node to avoid permission issues):

```powershell
$containerId = docker ps -q --filter "name=openclaw-gateway"
docker exec -it -u node $containerId node /app/dist/index.js onboard
```

WARNING: The onboarding wizard may set gateway.bind to "loopback", breaking tunnel access. After running onboard, verify the setting:
```powershell
Select-String '"bind"' "$env:USERPROFILE\.openclaw\openclaw.json"
```
If it shows "loopback", change it back to "lan" in openclaw.json.


Troubleshooting
If dashboard requires "pairing" after onboarding wizard:

The wizard was run as root, causing permission issues. Fix:
1. Fix file ownership inside the container:
```powershell
$containerId = docker ps -q --filter "name=openclaw-gateway"
docker exec $containerId chown -R 1000:1000 /home/node/.openclaw/
```
2. Check gateway.bind in openclaw.json - wizard may have set it to "loopback", change to "lan":
```powershell
Get-Content "$env:USERPROFILE\.openclaw\openclaw.json" | Select-String "bind"
```
3. Restart container:
```powershell
cd C:\projects\docker\openclaw
docker compose down && docker compose up -d
```
Prevention: Always run onboard with -u node flag

If tunnel shows 502 errors or "connection reset by peer" (MOST COMMON ISSUE):

First, check the .env file for OPENCLAW_GATEWAY_BIND=lan:
```powershell
Select-String "GATEWAY_BIND" C:\projects\docker\openclaw\.env
```
Hostinger's template does NOT include this setting by default
Without it, OpenClaw defaults to loopback mode which rejects tunnel connections
Fix:
```powershell
Add-Content C:\projects\docker\openclaw\.env "OPENCLAW_GATEWAY_BIND=lan"
```
Then restart the container.
Check tunnel logs: Open Windows Event Viewer > Application and Services Logs > cloudflared, or run `cloudflared tunnel run <TUNNEL_NAME>` manually for debug output.

If OpenClaw config validation fails:

Don't add dmPolicy for Discord or Slack channels - only WhatsApp and Telegram support it
Check logs:
```powershell
$containerId = docker ps -q --filter "name=openclaw-gateway"
docker logs $containerId
```

If OpenClaw won't start:

```powershell
$containerId = docker ps -aq --filter "name=openclaw-gateway"
docker logs $containerId
```

If dashboard won't load:

Make sure tunnel points to correct port (check .env for OPENCLAW_GATEWAY_PORT, default 18789)
Verify cloudflared is running: `Get-Service cloudflared`
Test local connection: `curl http://127.0.0.1:18789/`

If sandbox shows "Docker not available":

Verify Docker socket is mounted:
```powershell
$containerId = docker ps -q --filter "name=openclaw-gateway"
docker exec $containerId ls -la /var/run/docker.sock
```
Check node user is in docker group:
```powershell
docker exec $containerId id node
```
Check Docker CLI is installed:
```powershell
docker exec $containerId docker version
```
If Docker CLI missing, the startup script should install it on next restart.

If sandbox image is missing:

Run manually inside container:
```powershell
$containerId = docker ps -q --filter "name=openclaw-gateway"
docker exec $containerId sh -c 'cd /app && ./scripts/sandbox-setup.sh'
```
Or restart the container - the startup script will build it automatically.

If skill installation fails in onboarding wizard ("brew: command not found"):

Homebrew is not installed or not mounted into the container.
Verify the `homebrew_data` named volume exists: `docker volume ls | Select-String "homebrew"`
Check inside container:
```powershell
$containerId = docker ps -q --filter "name=openclaw-gateway"
docker exec $containerId /home/linuxbrew/.linuxbrew/bin/brew --version
```
If Homebrew is missing, restart the container - the startup script will install it.
Ensure docker-compose.yml has the volume mount: `homebrew_data:/home/linuxbrew/.linuxbrew`
