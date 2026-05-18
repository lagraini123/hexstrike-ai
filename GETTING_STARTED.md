# HexStrike AI MCP — Getting Started Guide

This guide walks you through setting up HexStrike AI MCP end-to-end on **Windows + WSL2 (Kali Linux)** and using it with Claude Desktop.

---

## Prerequisites

- Windows 10/11 with WSL2 installed
- Kali Linux WSL distro (`wsl --install -d kali-linux`)
- Python 3.10+ inside Kali Linux
- Claude Desktop installed on Windows

---

## Step 1 — Clone the Repository

Open a **Kali Linux WSL terminal** and run:

```bash
git clone git@github.com:lagraini123/hexstrike-ai.git
cd hexstrike-ai
```

---

## Step 2 — Set Up the Server Environment

The server runs from the project's virtual environment (Windows filesystem is fine for the server):

```bash
python3 -m venv hexstrike-env
source hexstrike-env/bin/activate
pip install -r requirements.txt
```

---

## Step 3 — Set Up the MCP Client Environment

The MCP client must use a **native Linux virtual environment** for fast startup. If you skip this, Claude Desktop will time out before the MCP server responds.

```bash
python3 -m venv /home/$USER/hexstrike-mcp-env
/home/$USER/hexstrike-mcp-env/bin/pip install mcp requests
```

> **Why a separate venv?** The MCP client is spawned by Claude Desktop on every launch. Importing packages from the Windows filesystem (`/mnt/c/...`) takes 7+ seconds in WSL2, which exceeds Claude Desktop's initialization timeout. The native Linux venv reduces this to under 1 second.

---

## Step 4 — Configure Claude Desktop

Edit the Claude Desktop config file at:

```
C:\Users\<YourName>\AppData\Local\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\claude_desktop_config.json
```

Add the following inside `"mcpServers"`:

```json
{
  "mcpServers": {
    "hexstrike-ai": {
      "command": "wsl.exe",
      "args": [
        "-d",
        "kali-linux",
        "--",
        "/home/<your-kali-username>/hexstrike-mcp-env/bin/python3",
        "/mnt/c/Users/<YourName>/Desktop/hexstrike-ai/hexstrike_mcp.py",
        "--server",
        "http://localhost:8888"
      ]
    }
  }
}
```

Replace `<your-kali-username>` and `<YourName>` with your actual usernames.

---

## Step 5 — Install Security Tools

HexStrike AI orchestrates tools that must be installed separately. Install the essentials:

```bash
sudo apt update
sudo apt install -y nmap gobuster nikto sqlmap hydra john \
  enum4linux netcat-traditional curl wget python3-pip \
  masscan tcpdump binwalk checksec
```

For more tools (nuclei, ffuf, httpx, etc.):

```bash
# Go-based tools
go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
go install github.com/ffuf/ffuf/v2@latest
go install github.com/projectdiscovery/httpx/cmd/httpx@latest
go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
```

Check what's available:

```bash
curl -s http://localhost:8888/health | python3 -m json.tool | grep -A2 "essential"
```

---

## Step 6 — Start the Server

Every time you want to use HexStrike MCP, start the server first in a Kali WSL terminal:

```bash
cd /mnt/c/Users/<YourName>/Desktop/hexstrike-ai
source hexstrike-env/bin/activate
python3 hexstrike_server.py
```

Leave this terminal open. Verify it's running:

```bash
curl http://localhost:8888/health
```

You should see `"status": "healthy"`.

---

## Step 7 — Launch Claude Desktop

1. Start the server (Step 6) first
2. Open Claude Desktop
3. The `hexstrike-ai` MCP should appear as connected in the tools panel (plug icon)

If it shows as failed, check:
- The server is running on port 8888
- The paths in `claude_desktop_config.json` are correct
- The native Linux venv exists at `/home/<username>/hexstrike-mcp-env/`

---

## Using the MCP Tools

HexStrike MCP exposes 150+ security tools to Claude. To use them, tell Claude you have authorization for the target and ask it to use HexStrike tools explicitly.

### Example Prompts

**Network Reconnaissance:**
```
I'm a security researcher testing my own lab machine at 192.168.1.100.
Use HexStrike nmap to do a full port scan and service detection.
```

**Web Application Testing:**
```
I own the domain lab.example.com. Use HexStrike to enumerate directories
with gobuster and scan for vulnerabilities with nuclei.
```

**CTF Challenge:**
```
I'm solving a CTF binary challenge. The binary is at /home/aymane/ctf/pwn.
Use HexStrike to run checksec, analyze it with radare2, and look for
buffer overflow vulnerabilities.
```

**Bug Bounty Recon:**
```
I'm doing authorized bug bounty recon on example.com (in scope).
Use HexStrike to enumerate subdomains, probe live hosts, and scan
for common vulnerabilities.
```

**CVE Intelligence:**
```
Use HexStrike CVE intelligence to analyze CVE-2024-1234 and determine
if it's exploitable, then generate a proof-of-concept if possible.
```

---

## Available Tool Categories

| Category | Tools | Examples |
|----------|-------|---------|
| Network | 25+ | nmap, masscan, rustscan, autorecon, enum4linux |
| Web App | 40+ | gobuster, ffuf, nuclei, nikto, sqlmap, wpscan |
| Binary / RE | 25+ | gdb, radare2, ghidra, pwntools, angr, checksec |
| Cloud | 20+ | prowler, trivy, kube-hunter, checkov, falco |
| OSINT | 20+ | sherlock, theharvester, subfinder, amass |
| CTF / Forensics | 20+ | volatility3, steghide, john, hashcat, binwalk |
| Password | 12+ | hydra, john, hashcat, medusa |

---

## Troubleshooting

### MCP shows as failed in Claude Desktop
- Make sure the server is running **before** opening Claude Desktop
- Check the native venv exists: `ls /home/$USER/hexstrike-mcp-env/bin/python3`
- Test the MCP client manually:
  ```bash
  echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' \
    | /home/$USER/hexstrike-mcp-env/bin/python3 hexstrike_mcp.py --server http://localhost:8888 2>/dev/null
  ```
  You should get a JSON response within 2 seconds.

### Tool returns an error
- Check if the tool is installed: `which <toolname>`
- Check server health: `curl http://localhost:8888/health`
- View server logs in the terminal where you started `hexstrike_server.py`

### Claude doesn't use the tools
- Start your prompt by establishing context (role + authorization)
- Explicitly say "use HexStrike tools" or "use the hexstrike-ai MCP"

---

## Tips

- Keep the server terminal open while using Claude Desktop — closing it stops all tool execution
- The server caches results, so repeated identical commands return instantly
- Use `--debug` flag for verbose logs: `python3 hexstrike_server.py --debug`
- Check running processes: `curl http://localhost:8888/api/processes/list`
- The server exposes a REST API directly — you can call tools without Claude:
  ```bash
  curl -X POST http://localhost:8888/api/tools/nmap \
    -H "Content-Type: application/json" \
    -d '{"target": "127.0.0.1", "flags": "-sV -p 1-1000"}'
  ```
