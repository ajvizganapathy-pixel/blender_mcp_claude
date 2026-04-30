# Blender MCP × Claude Desktop — Windows Setup

![Windows](https://img.shields.io/badge/Windows-0078D6?style=flat&logo=windows&logoColor=white)
![Claude Desktop](https://img.shields.io/badge/Claude_Desktop-CC785C?style=flat&logo=anthropic&logoColor=white)
![Blender](https://img.shields.io/badge/Blender_4.x-F5792A?style=flat&logo=blender&logoColor=white)
![Python](https://img.shields.io/badge/Python_3.12-3776AB?style=flat&logo=python&logoColor=white)
![MCP](https://img.shields.io/badge/MCP_2025--11--25-22C55E?style=flat)
![blender-mcp](https://img.shields.io/badge/blender--mcp-v1.5.6-orange?style=flat)

Complete guide to connect **Blender MCP** with **Claude Desktop** on Windows — covers `uv` installation, PATH configuration, JSON config, Python version pinning, and dependency troubleshooting.

---

## Architecture

```
Claude Desktop  →  uvx (process launcher)  →  blender-mcp server  →  Blender (port 9000)
```

| Component | Role | Requirement |
|---|---|---|
| `uv` / `uvx` | Python tool runner — installs & runs blender-mcp in an isolated env | Must be on PATH visible to Claude Desktop |
| `blender-mcp` | MCP server — translates Claude tool calls to Blender Python API | Installed automatically by uvx |
| Blender addon | Socket server inside Blender listening on port 9000 | Must be enabled in Blender preferences |
| `claude_desktop_config.json` | Tells Claude Desktop how to spawn the MCP server | Absolute paths required on Windows |

---

## Setup

### Step 1 — Install uv

Run in an **Administrator PowerShell**:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Binaries are installed to:
```
C:\Users\<YourName>\.local\bin\uv.exe
C:\Users\<YourName>\.local\bin\uvx.exe
```

> **Note:** The installer modifies your PowerShell profile. Restart Claude Desktop after this step — it caches the environment at launch.

---

### Step 2 — Fix PowerShell Execution Policy

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

`RemoteSigned` allows locally created scripts (like the uv profile entry) to run freely, while still requiring internet-downloaded scripts to be signed.

---

### Step 3 — Install Python 3.12 via uv

> **Critical:** `blender-mcp v1.5.6` depends on `supabase → storage3 → pyiceberg`. The `pyiceberg` package has no pre-built wheel for Python 3.14 on Windows, causing a C-extension compile failure that requires MSVC. **Pin to Python 3.12.**

| Python Version | pyiceberg wheel | blender-mcp installs |
|---|---|---|
| 3.14 (system default) | No — build from source | Fails (needs MSVC) |
| **3.12 (via uv)** | **Pre-built wheel** | **Works** |
| 3.11 | Pre-built wheel | Works |

```powershell
uv python install 3.12
uv cache clean
```

---

### Step 4 — Configure `claude_desktop_config.json`

Open the config file:

```powershell
notepad "$env:APPDATA\Claude\claude_desktop_config.json"
```

Paste the following (replace `ANJAN GANAPATHY K` with your Windows username):

```json
{
  "mcpServers": {
    "blender": {
      "command": "C:\\Users\\<YourName>\\.local\\bin\\uvx.exe",
      "args": ["--python", "3.12", "blender-mcp"]
    }
  }
}
```

| Key | Value | Why |
|---|---|---|
| `"command"` | Full absolute path to `uvx.exe` | Claude Desktop inherits a limited PATH |
| `"args"[0]` | `"--python"` | Tells uvx to use a specific Python version |
| `"args"[1]` | `"3.12"` | Pins Python to avoid pyiceberg build failure on 3.14 |
| `"args"[2]` | `"blender-mcp"` | The PyPI package name of the MCP server |

> **Note:** Backslashes in JSON **must be doubled** (`\\`). Use `where.exe uvx` in PowerShell to get the exact path.

---

### Step 5 — Enable Blender MCP Addon

1. Open Blender → **Edit → Preferences → Add-ons**
2. Search for **"MCP"** or **"blender-mcp"**
3. Enable the addon — a panel appears with the socket port (default: `9000`)
4. Confirm the server is listening (green status in the addon panel)
5. Keep Blender open before restarting Claude Desktop

When connected successfully, the log shows:
```
Server started and connected successfully
```
followed by a `ListToolsRequest` listing all available Blender tools.

---

## Troubleshooting

| Error in Log | Root Cause | Fix |
|---|---|---|
| `spawn uvx ENOENT` | Claude Desktop can't find `uvx` — PATH not inherited | Use absolute path to `uvx.exe` in config |
| `'uvx' is not recognized` | Same — cmd.exe fallback also fails | Absolute path in `command` field |
| `No module named blender_mcp` | Config using `python -m blender_mcp` instead of `uvx` | Switch back to `uvx`-based config |
| `Microsoft Visual C++ 14.0 required` | Python 3.14 — pyiceberg has no wheel, tries to compile | Add `--python 3.12` to args |
| `Could not connect to Blender` | Blender not running or addon not enabled | Open Blender, enable MCP addon |
| `PSSecurityException` on profile | PowerShell execution policy blocks uv profile script | `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` |
| Log file not found | Server never launched — ENOENT before any file was written | Fix PATH/command first, then check `%APPDATA%\Claude\logs\` |

### View live logs

```powershell
Get-Content "$env:APPDATA\Claude\logs\mcp-server-blender.log" -Tail 30
```

---

## Verification Checklist

- [ ] `uv --version` returns `uv 0.11.7` or later
- [ ] `where.exe uvx` returns path inside `\.local\bin\`
- [ ] `claude_desktop_config.json` uses absolute path to `uvx.exe` with `--python 3.12`
- [ ] Blender is open with MCP addon enabled on port `9000`
- [ ] Claude Desktop fully quit (system tray → Quit) and relaunched
- [ ] Settings → Developer shows `blender` with a **connected** status
- [ ] Log shows `ListToolsRequest` with a tool list — no ENOENT, no build errors

---

## Quick Reference

```powershell
# 1. Install uv
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# 2. Fix execution policy (new Admin PowerShell)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# 3. Install Python 3.12
uv python install 3.12

# 4. Clear uv cache
uv cache clean

# 5. Confirm uvx path
where.exe uvx

# 6. Open Claude Desktop config
notepad "$env:APPDATA\Claude\claude_desktop_config.json"

# 7. View logs after restart
Get-Content "$env:APPDATA\Claude\logs\mcp-server-blender.log" -Tail 30
```

---

*blender-mcp × Claude Desktop × uv — Windows Setup | Python 3.12 · MCP 2025-11-25*
