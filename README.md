# agent-docs

Shared docs that coding agents fetch on demand. A repository links the raw URL of the doc it needs from its `AGENTS.md`, so an agent reads the latest version only when a task calls for it.

| Doc | For |
|---|---|
| [plugin-releasing.md](plugin-releasing.md) | Claude Code plugin repositories whose marketplace entry pins each plugin to a release tag |
| [plugin-releasing-dist.md](plugin-releasing-dist.md) | Claude Code plugin repositories whose plugin installs from a `dist` branch that CI builds |

Each doc holds no repository-specific values: it reads what it needs from the repository it runs in.

Fetch a doc with `curl -fsSL https://raw.githubusercontent.com/JustAzul/agent-docs/main/<doc>`. The raw file server caches for up to five minutes, so an edit on `main` reaches agents within that time.
