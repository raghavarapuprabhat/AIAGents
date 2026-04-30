# GitHub Copilot Chat — Custom Chat Modes

Five `.chatmode.md` files that port the AI Agent Platform's agents into
**GitHub Copilot Chat (VS Code)** as native custom chat modes. No backend
needed — Copilot's built-in tools (and any MCP servers you have configured)
do the work.

| File | Mode picker name | What it does |
|---|---|---|
| `code-doc-agent.chatmode.md`   | Code Documentation Agent | Generate `.docs/` for the active workspace (Java + React) |
| `sre-agent.chatmode.md`        | SRE Triage Agent | Triage a pasted issue or CSV against the codebase + docs |
| `sre-fixer-agent.chatmode.md`  | SRE Fixer Agent | Plan -> patch -> test -> open Azure Repos PR (no auto-merge) |
| `ado-md-agent.chatmode.md`     | ADO MD Personal Assistant | Portfolio snapshot + drill-down (needs ADO MCP) |
| `ado-dev-agent.chatmode.md`    | ADO Developer Personal Assistant | Status report + consent-gated workitem updates (needs ADO MCP) |

## Install (per workspace)

1. Create a `.github/chatmodes/` folder in the repo where you want the agents available:
   ```bash
   mkdir -p .github/chatmodes
   ```
2. Copy the chat mode files you want to use:
   ```bash
   cp /Users/prabhat/Documents/AIAgentPlatform/CopilotChatmodes/*.chatmode.md .github/chatmodes/
   ```
3. Reload VS Code. The modes appear in the chat-mode picker (the small dropdown
   above the Copilot Chat input — "Ask", "Edit", "Agent", and now your custom
   modes).

## Install (user-global)

Put them in your VS Code user data directory so they're available in every workspace:

- macOS:   `~/Library/Application Support/Code/User/prompts/`
- Linux:   `~/.config/Code/User/prompts/`
- Windows: `%APPDATA%\Code\User\prompts\`

VS Code picks up `*.chatmode.md` files from that directory automatically.

## Required MCP servers

Two of the five agents depend on external MCP servers. Configure them in your
VS Code settings (`mcp.json`) before using:

### Azure DevOps MCP (for `ado-md-agent` and `ado-dev-agent`)
```json
{
  "servers": {
    "azure-devops": {
      "command": "npx",
      "args": ["-y", "@azure-devops/mcp"],
      "env": {
        "AZURE_DEVOPS_ORG": "https://dev.azure.com/<your-org>",
        "AZURE_DEVOPS_PAT": "${input:azure_devops_pat}"
      }
    }
  }
}
```

The `sre-fixer-agent` uses Copilot's built-in `runCommands` tool to call
`git` and `curl` — it does NOT need an MCP server, but you must have a
PAT in your environment when it runs the Azure Repos REST call.

## Tool permissions

When you select a chat mode for the first time, Copilot asks you to confirm
the tool list declared in the file's frontmatter. The agents below request
exactly the tools they need — nothing more:

- `code-doc-agent`     -> `codebase`, `editFiles`, `search`, `usages`, `findTestFiles`, `runCommands`
- `sre-agent`          -> `codebase`, `search`, `usages`, `findTestFiles`, `editFiles`
- `sre-fixer-agent`    -> `codebase`, `editFiles`, `runCommands`, `runTests`, `usages`, `findTestFiles`
- `ado-md-agent`       -> `codebase`, `editFiles`, `fetch` (+ ADO MCP tools)
- `ado-dev-agent`      -> `codebase`, `editFiles` (+ ADO MCP tools)

## Model

Each file declares `model: Claude Sonnet 4` in the frontmatter. Override
in the file or in Copilot's mode picker if you prefer a different model.

## Differences vs the full website implementation

These chat modes encode the **behavior** of each agent into a Copilot system
prompt + tool-use plan. The full website implementation (`Code/`) ships:
- Postgres-backed conversation summarisation (cross-session memory)
- Chroma vector store for SRE RAG over `.docs/` embeddings
- APScheduler for the daily MD ETL
- Hard safety rails enforced in Python (path traversal, branch protection,
  test command whitelist) — Copilot's tool sandbox provides a *softer* version
  of the same protection (you have to approve each command), but the chat
  mode body still spells the rules out so the model refuses unsafe requests.

If you want the full pipeline (durable memory, vector RAG, scheduled ETL,
hard-coded safety rails), use the website. If you want a single-developer
"agent in my IDE" experience, use these chat modes.

## Versioning

These files are intentionally self-contained — no imports, no shared library.
You can copy any one of them into another project's `.github/chatmodes/`
without bringing the rest of the platform.
