## User Guide
A walkthrough for using PyUpgrade Intelligence.

---
## 1.Setup

### Prerequisites
- Python 3.11+
- For the BOSS plugin: JDK 17+, and BOSS itself installed

### Install the Python engine
```bash
git clone https://github.com/dishasingh-21/boss-plugin-PyUpgrade-Intelligence.git
cd boss-plugin-PyUpgrade-Intelligence/engine
python -m venv .venv
.venv\Scripts\activate          # Windows
# source .venv/bin/activate     # macOS/Linux
pip install -r requirements.txt
```

---
## 2. Via the CLI

Below is an example command:
```bash
python engine/cli.py check django 4.2 5.0 --repo path/to/a/django/project
```

The first run against a new framework/version pair takes a minute or two, it's downloading and parsing the framework's real source. Every run after that hits a local cache and finishes in under a second.

You'll see something like:
```
Risk score: 89/100 — HOLD
Relevant changes: 3 / 865
Worst symbol: django.middleware.locale.LocaleMiddleware.get_fallback_language
 
Breakdown:
  REMOVED: 1 x 1.0 = 1.00
  SIGNATURE_CHANGED_BREAKING: 1 x 0.85 = 0.85
  SIGNATURE_CHANGED_ADDITIVE: 1 x 0.15 = 0.15
 
Affected files:
  xyz_project/services.py:20 -> django.middleware.locale.LocaleMiddleware.get_fallback_language (REMOVED)
  xyz_project/services.py:13 -> django.core.paginator.Paginator.__init__ (SIGNATURE_CHANGED)
  xyz_project/services.py:23 -> django.forms.models.BaseModelFormSet.save_existing (SIGNATURE_CHANGED)
```

**More commands to try:**
```bash
# just the flagged file/line hits, faster than a full check
python engine/cli.py affected django 4.2 5.0 --repo path/to/project
 
# same, with real source code snippets and before/after signatures
python engine/cli.py affected django 4.2 5.0 --repo path/to/project --enriched
 
# everything the project uses, regardless of what changed
python engine/cli.py usage django 4.2 --repo path/to/project
 
# structural diff on its own, user's repo not needed
python engine/cli.py changes django 4.2 5.0
 
# widen to include lower-confidence, body-only implementation changes
python engine/cli.py changes django 4.2 5.0 --type BODY_CHANGED
```

`--enriched` costs more (it reads real source files for code snippets and diff text). Use plain `affected` for a quick pass, `--enriched` when you actually need to see what changed.

---
## 3. As an MCP server
The same engine runs as an MCP server, usable from any MCP client.

**Test it with the MCP Inspector** (a browser-based tool for calling MCP servers directly, useful for verifying it works before wiring it into any client):
```bash
pip install "mcp[cli]"
mcp dev engine/mcp_server.py
```

**Connect it to Claude Desktop:** add an entry to Claude Desktop's config file (Claude menu → Settings → Developer → Edit Config):
```json
{
  "mcpServers": {
    "pyupgrade-intelligence": {
      "command": "path/to/.venv/Scripts/python.exe",
      "args": ["path/to/engine/mcp_server.py"]
    }
  }
}
```
Use absolute paths. Restart Claude Desktop fully (quit from the system tray, not just close the window) for it to pick up the change.

Once connected, it'll have 7 MCP tools defined in mcp_server.py. Ask claude some questions related to each of the mcp tools to test them.

---
## 4. Managing the cache
Framework graphs are built once and cached — every check after the first is fast. Build one ahead of time if you want:

```bash
python engine/cli.py warm django 5.0
python engine/cli.py warm django 4.2
```
 
```bash
python engine/cli.py cache list           # see what's already cached
python engine/cli.py cache clear          # clear everything
python engine/cli.py cache clear django   # clear just one framework
```

---
## 5. A framework whose import name differs from its PyPI name
Some packages install under one name but import under a completely different one like, `pillow` imports as `PIL`, `livekit-agents` imports as `livekit`, `scikit-learn` imports as `sklearn`. The tool detects this automatically in almost all cases. If it can't (you'll get a clear error listing what it found instead), tell it explicitly:

for e.g.,
```bash
python engine/cli.py check livekit-agents 1.5.1 1.8.1 --repo path/to/project --package livekit
```
