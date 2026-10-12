# PyUpgrade Intelligence

**Know exactly what will break before you upgrade a Python framework but not from a changelog, from the actual code.**

A semantic graph-based upgrade risk checker for Python frameworks.

---
## The problem
"Is it safe to upgrade Django from 4.2 to 5.0?" is a question every Python developer eventually asks, and today the honest answer requires manually reading changelogs, guessing which of hundreds of changes actually touch your codebase, and hoping you didn't miss something. Every existing tool in this space answers the question by reading what maintainers *wrote* about a release. None of them read the actual code.

## What this does differently
PyUpgrade Intelligence builds a real semantic graph of a framework's source code - every function, class, and method, with real signatures, and how they call each other - for two versions, and structurally diffs them. It then cross-references that diff against your own codebase's actual usage, and produces a risk score with the exact files and lines that will break, and why.
- **Structural, not textual.** Catches undocumented breaking changes - a parameter renamed, a method quietly turned into a `@property`, a sync method becoming async, that never made it into a changelog.
- **Deterministic and explainable.** Every risk score traces back to a formula with a mathematical proof, not an LLM's best guess. See `docs/components/risk-scorer.md` and `Risk-Scoring-Mathematical-Model.md`.
- **Scoped to your code.** Out of hundreds of real changes in a framework release, you see only the ones that intersect with what you actually use.

## How it's delivered
- **CLI**: See [`docs/components/cli.md`](docs/components/cli.md).
- **MCP server**: usable with any MCP client (Claude Desktop, the MCP Inspector, etc.)

## Usage
The engine works entirely on its own.

Some example commands look like:
```bash
python engine/cli.py check django 4.2 5.0 --repo path/to/your/project
python engine/cli.py changes django 4.2 5.0
python engine/cli.py usage django 4.2 --repo path/to/your/project
python engine/cli.py warm django 5.0
```

Full command reference: [`CLI-Specification.md`](CLI-Specification.md)

It also runs as a plain MCP Server, connectable from any MCP Client (like, Claude Desktop, MCP Inspector, and even your own MCP Client)

To run a server with a debugging inspector, run:
```bash
pip install mcp[cli]
mcp dev engine/mcp_server.py
```

---
## Architecture

```mermaid
flowchart TD
    PyPI[(PyPI)] --> CGB[context_graph_builder]
    CGB --> Graph[(semantic graph<br/>cached to disk)]
 
    subgraph engine [Python engine]
        Graph --> DE[diff_engine]
        Graph --> UI[usage_indexer]
        Repo[(your codebase)] -.-> UI
        DE --> RS[risk_scorer]
        UI --> RS
        RS --> O[orchestrator]
    end
 
    O --> CLI[CLI]
    O --> MCP[MCP server]
```

- `context_graph_builder` parses a framework's real source into a graph - every function, class, and method, with signatures, async/property flags, and a structural fingerprint of each body - plus the calls, imports, and inheritance between them.
- `diff_engine` and `usage_indexer` both read from that graph independently: one compares two versions structurally, the other scans the user's code for real usage, entirely on their machine.
- `risk_scorer` joins the two, only symbols that are both changed and used matter, and scores them with a risk scoring model.
- `orchestrator` wires all of this into a handful of functions that the CLI and MCP server both call directly.

---
## License
MIT (see [`LICENSE`](LICENSE))

## Author
Disha Singh ([@dishasingh-21](https://github.com/dishasingh-21))
