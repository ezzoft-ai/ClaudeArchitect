# Claude Desktop Setup

This repo was set up and verified on **Windows 11**, but today's demo runs on a **Mac**.
The two platforms need slightly different commands — use the section that matches your
machine. The actual "point Claude Desktop at the server" step is the same UI flow on
both.

---

## macOS 

### 1. Prerequisites

```bash
python3 --version     # need 3.12+
git --version
node --version         # only needed later for MCP Inspector, optional
```

If `uv` isn't installed:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Close and reopen your terminal afterward so `uv` is on your `PATH`.

### 2. Clone, install, initialize

```bash
git clone https://github.com/abhijithezzoft/analytics-mcp-demo.git
cd analytics-mcp-demo
uv sync
uv run analytics-mcp init
```

On Mac there's no equivalent of the Windows Application Control issue we hit — the
`analytics-mcp` command (an entry point `uv` generates) should just work directly. If for
any reason it doesn't, the fallback that always works on any platform is:

```bash
uv run python -m analytics_mcp.cli init
```

### 3. Point Claude Desktop at the server

The exact config file location **varies by app version** — don't assume a fixed path.
Instead, use the in-app flow, which always shows you the real file:

1. Open Claude Desktop → **Settings**.
2. Go to **Desktop app → Extensions** (or **Developer**, depending on your version).
3. Find **"Local MCP servers"** and click **"Edit config"**. This opens the actual file
   the app reads — note its path, since it may not be the classic
   `~/Library/Application Support/Claude/claude_desktop_config.json` location.
4. Add (or merge into existing content — **don't delete anything already there**) an
   `mcpServers` key:

```json
{
  "mcpServers": {
    "analytics": {
      "command": "uv",
      "args": [
        "run",
        "--directory",
        "/absolute/path/to/analytics-mcp-demo",
        "analytics-mcp",
        "serve"
      ]
    }
  }
}
```

Replace `/absolute/path/to/analytics-mcp-demo` with wherever you cloned it (run `pwd` inside
the folder to get it). If the file already has other keys (app preferences, other MCP
servers), only add this `mcpServers` block alongside them — don't overwrite the file.

**Back up the file before editing it**, in case the edit needs to be undone:

```bash
cp "<path-to-config>" "<path-to-config>.bak"
```

5. **Fully quit** Claude Desktop (not just close the window — Cmd+Q, or Activity Monitor
   if it lingers) and reopen it. MCP servers are only loaded at startup.
6. Back in **Settings → Desktop app → Extensions**, "analytics" should now appear under
   Local MCP servers, going from "Starting" to a running/connected state.
7. Test it with a real message in a new chat:
   > *"Show me the top 5 products by revenue and analyze the monthly sales trend"*

   The first time each tool runs, Claude Desktop will ask for permission to use it —
   that's normal, expected behavior the first time, not a bug.

---

## Windows 11 (how this was verified — reference only)

Two Windows-specific issues came up that **don't apply on Mac**:

1. **Application Control policy** blocked the `analytics-mcp.exe` entry-point shim `uv`
   generates. Workaround: use `uv run python -m analytics_mcp.cli <command>` everywhere
   instead of `uv run analytics-mcp <command>`, including inside the Claude Desktop
   config's `args`.
2. The config file on this machine's app build lived at:
   ```
   C:\Users\<user>\AppData\Local\Claude-Data\<session-id>\claude_desktop_config.json
   ```
   not the classic `%APPDATA%\Claude\claude_desktop_config.json` — confirming the config
   location really does vary by app build/version, hence step 3's advice above to always
   find it via the in-app "Edit config" button rather than assuming a path.

Windows config block used here (for reference, don't copy directly — paths won't match
your Mac):

```json
{
  "mcpServers": {
    "analytics": {
      "command": "C:\\Users\\<user>\\.local\\bin\\uv.exe",
      "args": [
        "run",
        "--directory",
        "D:\\Local MCP demo\\analytics-mcp",
        "python",
        "-m",
        "analytics_mcp.cli",
        "serve"
      ]
    }
  }
}
```
