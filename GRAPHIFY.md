# Graphify — local playbook

KaDeep Studios keeps a **machine-local knowledge graph** of the codebase at `graphify-out/`. That directory is gitignored. This is not the product memory sidecar (`services/memory-graph` / Neo4j). That graph is about *what the agent learned in a project*. Graphify is about *how this repository is wired*.

| Artifact | Use it for |
|---|---|
| [graphify-out/graph.json](../graphify-out/graph.json) | `query` / `path` / `explain` / `affected` |
| [graphify-out/wiki/index.md](../graphify-out/wiki/index.md) | Browse communities before opening source |
| [graphify-out/GRAPH_REPORT.md](../graphify-out/GRAPH_REPORT.md) | Hubs, import cycles, cohesion |
| [graphify-out/graph.html](../graphify-out/graph.html) | Interactive graph |
| [graphify-out/GRAPH_TREE.html](../graphify-out/GRAPH_TREE.html) | Folder tree of symbols |
| [graphify-out/teststudios.ai-callflow.html](../graphify-out/teststudios.ai-callflow.html) | Call-flow diagrams |
| [docs/ARCHITECTURE.md](ARCHITECTURE.md) | Written architecture from the graph |

Current map (code-only, built 2026-10-02 from `6ee9d696`): **9023 nodes · 29866 edges · 253 communities**.

## One-time setup

Needs [uv](https://docs.astral.sh/uv/) and Python 3.10+. The PyPI package is `graphifyy` (two y’s); the command is `graphify`.

```bash
uv tool install 'graphifyy==0.9.55'
./scripts/setup-graphify.sh
```

The script pins `graphifyy==0.9.55`, writes this machine’s Python at `graphify-out/.graphify_python`, installs the post-commit / post-checkout hooks, and builds `graphify-out/graph.json` when it is missing. The Cursor rule is already in git — do not let a raw `graphify cursor install` overwrite [`.cursor/rules/graphify.mdc`](../.cursor/rules/graphify.mdc).

After that, `graphify query "how does BrowserSession get used?"` works with no API key.

## Daily commands

```bash
graphify query "how does runChatTurn start a browser"
graphify path "registry.ts" "runner.ts"
graphify explain "BrowserSession"
graphify affected "BrowserSession"
graphify god-nodes
```

| Question shape | Command |
|---|---|
| How does X work? What touches Y? | `graphify query "..."` |
| How does A reach B? | `graphify path "A" "B"` |
| What is this symbol? | `graphify explain "Concept"` |
| What breaks if I change X? | `graphify affected "X"` |

Prefer wiki + these commands before grepping. After you change code:

```bash
graphify update .
```

AST-only, no API key. Then refresh the browsable pages:

```bash
graphify export wiki
graphify tree
```

The post-commit hook rebuilds `graph.json` after `git commit` and writes only under `graphify-out/`. Do not commit that directory.

## Question cookbook

Ask these (or paste them to Cursor). Each line is a starting symbol the graph actually has.

### Browser and agent

```bash
graphify query "why does BrowserSession bridge runner QA tools artifacts and budget gates"
graphify explain "BrowserSession"
graphify affected "BrowserSession"
graphify path "runChatTurn" "ensureBrowser"
graphify query "how does runChatTurn reach a real browser"
```

`BrowserSession` (`packages/browser-driver/src/index.ts`) is the strongest product hub (~79 edges). It sits in [Browser Session Manager](../graphify-out/wiki/Browser_Session_Manager.md) and reaches runner tools, budget gates (`overBudget()` in `apps/api/src/agent/tools/browser.ts`), snapshots, and screenshots. Do not fold browser-driver into the API.

### Tool registry (import cycles)

```bash
graphify affected "apps/api/src/agent/registry.ts"
graphify path "registry.ts" "runner.ts"
graphify explain "allTools"
graphify query "why does agent registry import control tools subagents and runner"
```

Two cycles, both through the registry:

- `registry.ts` → `control.ts` → `subagents.ts` → `registry.ts`
- `registry.ts` → `qa.ts` → `runner.ts` → `registry.ts`

See [HITL Tool Registry](../graphify-out/wiki/HITL_Tool_Registry.md). Treat a registry change as a wide blast radius, not a local edit.

### Memory (three layers + graph sidecar)

```bash
graphify query "where do memory store curator retrieve and graph meet"
graphify path "writeFact" "graphSearch"
graphify explain "MemoryLayer"
graphify affected "collection"
```

Code communities: [Memory Store and Graph](../graphify-out/wiki/Memory_Store_and_Graph.md), [Memory Curation](../graphify-out/wiki/Memory_Curation.md) (includes `guardMemory()` / PII), [Memory Retrieval](../graphify-out/wiki/Memory_Retrieval.md), [Memory Graph Service](../graphify-out/wiki/Memory_Graph_Service.md). The product sidecar is `services/memory-graph` — query Graphify for *callers*, not for Zep/Neo4j schema.

### Artifacts

```bash
graphify query "what connects UploadResult TurnArtifacts SaveInput to Artifact"
graphify path "TurnArtifacts" "Artifact"
graphify affected "saveUploadedArtifact"
```

Artifacts are split across [Artifact Processing](../graphify-out/wiki/Artifact_Processing.md), [Artifact Spec Builder](../graphify-out/wiki/Artifact_Spec_Builder.md), [Artifact Context Mentions](../graphify-out/wiki/Artifact_Context_Mentions.md), and [Artifacts View UI](../graphify-out/wiki/Artifacts_View_UI.md). Shared type: `Artifact` in `packages/shared`.

### Data and auth

```bash
graphify affected "collection"
graphify explain "collection"
graphify query "how do projects teams and vault secrets relate"
graphify path "login" "issueTokens"
```

`collection()` (`apps/api/src/db/client.ts`) is a utility hub (~38 edges). Changing it hits most Mongo-backed services. Vault crypto lives in [Secrets Vault and Projects](../graphify-out/wiki/Secrets_Vault_and_Projects.md), separate from [Auth and Passwords](../graphify-out/wiki/Auth_and_Passwords.md).

## What stays local

Nothing under `graphify-out/` is committed. That includes `graph.json`, `GRAPH_REPORT.md`, `wiki/`, the HTML views, `manifest.json`, labels, `.graphify_python`, `.graphify_root`, `cache/`, and dated backup folders.

`./scripts/setup-graphify.sh` builds the map the first time (`graphify update .`, code-only via `.graphifyignore`). Later `graphify update .` is a no-op unless code topology changed. `PYTHONHASHSEED=0` is set by the setup script and the git hook so community ids do not churn between runs.

Rebuild from scratch (rare — after changing `.graphifyignore`):

```bash
PYTHONHASHSEED=0 GRAPHIFY_FORCE=1 graphify update .
```

A later `/graphify .` with a Gemini key can add docs and images. That is optional. The local code graph is enough for Cursor on this machine.
