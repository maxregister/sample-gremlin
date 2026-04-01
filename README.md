# Malware Analysis Pipeline

Static-analysis-only pipeline: Discord bot → DigitalOcean → Ghidra (MCP) → Claude.
The binary is **never executed**. Ghidra disassembles and decompiles it statically.

---

## Services (all on one Droplet)

```
bot/          Discord bot          — receives ZIP, extracts in memory, triggers API
api/          FastAPI orchestrator — pulls from Spaces, runs Ghidra, calls Claude via MCP
mcp-server/   Ghidra MCP bridge   — exposes Ghidra tools (list_functions, decompile, etc.)
```

All three communicate over an **internal Docker network** — nothing is exposed to the internet
except port 22 (SSH, your IP only). Access the API via SSH tunnel.

---

## Quick Start

### 1. Provision a Droplet
- Ubuntu 22.04, Basic, 2 vCPU / 4 GB RAM (~$24/mo)
- Note the public IP

### 2. Create a DO Space
- Region: nyc3 (or match your droplet region)
- Name: `malware-samples`
- Access: Restricted (private)
- Generate a Spaces API key pair

### 3. Create a Discord Bot
- https://discord.com/developers/applications → New Application → Bot
- Enable: Message Content Intent
- Invite to your private server with permissions: Read Messages, Send Messages, Attach Files
- Create a channel called `malware-analysis` (or set ANALYSIS_CHANNEL in .env)

### 4. Deploy
```bash
scp -r . root@<droplet-ip>:/opt/malware-pipeline
ssh root@<droplet-ip>
chmod +x /opt/malware-pipeline/droplet_setup.sh
/opt/malware-pipeline/droplet_setup.sh <your-home-ip>
```

### 5. Fill in .env
```bash
nano /opt/malware-pipeline/.env
docker compose restart
```

---

## Usage

1. Zip your sample with password `infected`:
   ```bash
   zip --password infected sample.zip malware.exe
   ```
2. Drop `sample.zip` into `#malware-analysis` on Discord
3. The bot extracts in memory → streams raw binary to Spaces → triggers the API
4. Ghidra runs headless static analysis; Claude queries it via MCP
5. Report appears in Discord as an embed + downloadable `.txt`

---

## Accessing the API directly (SSH tunnel)

The API port is never opened publicly. Use a tunnel:
```bash
ssh -L 8080:localhost:8080 root@<droplet-ip>
```
Then open: http://localhost:8080/docs

---

## Container layout

```
┌─────────────────────────────────────────────┐
│  Droplet                                    │
│                                             │
│  ┌─────────┐   http   ┌─────────────────┐  │
│  │   bot   │ ───────► │  api  :8080     │  │
│  └─────────┘          └────────┬────────┘  │
│                                │ http       │
│                       ┌────────▼────────┐  │
│                       │  mcp  :9000     │  │
│                       │  (Ghidra inside)│  │
│                       └─────────────────┘  │
│                                             │
│  All containers on internal Docker network  │
│  No ports exposed to internet               │
└─────────────────────────────────────────────┘
```

---

## MCP Tools available to Claude

| Tool               | Description                              |
|--------------------|------------------------------------------|
| `list_functions`   | All functions with addresses             |
| `decompile_function` | Pseudocode for a named function        |
| `get_strings`      | All extracted strings (min length)       |
| `get_imports`      | Imported libraries and symbols           |
| `get_xrefs`        | Cross-references to a symbol             |
| `search_bytes`     | Hex byte pattern search                  |

---

## Security notes

- Binary only exists on disk during analysis; deleted in `finally` block regardless of outcome
- MCP server is localhost-only (internal Docker network, no host port binding)
- Spaces bucket is private; all objects use server-side encryption
- Bot never writes the raw binary to disk — extraction is in-memory only
- Rate limit: 5 analyses per user per hour (configurable in bot/bot.py)
