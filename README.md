<p align="center">
  <img src="https://cdn.prod.website-files.com/69082c5061a39922df8ed3b6/69b88b355cf35e13745104a0_Mar%2016%2C%202026%2C%2010_58_53%20PM.png" alt="Clawdis" width="140" />
</p>

<h1 align="center">Clawdis</h1>

<p align="center">
  <strong>Typed workflow engine for autonomous AI agents -- JSON-first pipelines, jobs, and approval gates.</strong>
  <br/>
  <em>stay connected.</em>
</p>

<p align="center">
  <a href="https://x.com/clawdisAI"><img src="https://img.shields.io/badge/Twitter-@clawdisAI-1DA1F2?style=flat-square&logo=x&logoColor=white" alt="Twitter" /></a>
  <a href="https://x.com/belimad"><img src="https://img.shields.io/badge/Creator-@belimad-1DA1F2?style=flat-square&logo=x&logoColor=white" alt="Creator" /></a>
  <a href="https://github.com/mbelinky"><img src="https://img.shields.io/badge/GitHub-mbelinky-181717?style=flat-square&logo=github&logoColor=white" alt="GitHub" /></a>
  <img src="https://img.shields.io/badge/Node-22+-339933?style=flat-square&logo=nodedotjs&logoColor=white" alt="Node 22+" />
  <img src="https://img.shields.io/badge/TypeScript-5.x-3178C6?style=flat-square&logo=typescript&logoColor=white" alt="TypeScript" />
  <img src="https://img.shields.io/badge/License-MIT-blue?style=flat-square" alt="MIT" />
</p>

---

## Overview

Clawdis is a typed workflow engine designed for autonomous AI agents. It replaces ad-hoc replanning with deterministic, resumable pipelines -- saving tokens while improving reliability and auditability.

Agents invoke Clawdis workflows in a single step instead of reasoning through multi-step plans repeatedly. Every stage operates on structured JSON, not text streams. Approval gates provide hard checkpoints between sensitive operations.

<p align="center">
  <img src="https://cdn.prod.website-files.com/69082c5061a39922df8ed3b6/69bb04e45f943305e7fb5964_New%20Project%20-%202026-03-16T234838.962.png" alt="Clawdis Workflow Engine" width="720" />
</p>

---

## Architecture

```
Agent (OpenClaw / Claude / Any LLM)
  |
  v
+----------------------------------------------------------+
|  Clawdis Workflow Runtime                                |
|                                                          |
|  Pipeline Parser     ->  Stage Resolver                  |
|  JSON Type System    ->  Data Flow Engine                |
|  Approval Gate       ->  TTY / Agent Integration         |
|  LLM Provider Router ->  OpenClaw / Pi / HTTP Adapter    |
|  Shell Executor      ->  Sandboxed OS Command Runner     |
+----------------------------------------------------------+
  |              |              |
  v              v              v
GitHub API    LLM Providers   OS Shell
(PR Monitor)  (Multi-model)   (Typed I/O)
```

---

## Workflow Engine

### Pipeline Execution Model

Clawdis workflows are deterministic, typed pipeline definitions that read like small scripts:

```yaml
name: jacket-advice
args:
  location:
    default: Phoenix
steps:
  - id: fetch
    run: weather --json ${location}

  - id: confirm
    approval: Want jacket advice from the LLM?
    stdin: $fetch.json

  - id: advice
    pipeline: >
      clawdis.llm.invoke --prompt "Given this weather data, should I wear a jacket?
      Be concise and return JSON."
    stdin: $fetch.json
    when: $confirm.approved
```

| Directive | Purpose |
|-----------|---------|
| `run:` / `command:` | Execute OS commands with typed I/O |
| `pipeline:` | Native Clawdis stages (LLM invoke, data transform) |
| `approval:` | Hard workflow gate between steps |
| `stdin: $step.stdout` | Structured data passing (no temp files) |
| `when:` / `condition:` | Conditional execution based on prior step results |

---

## PR Monitoring

Clawdis ships with a built-in GitHub PR monitoring workflow. Agents invoke it in a single step to detect state changes without re-querying the API manually:

### Detecting State Changes

```bash
node bin/clawdis.js "workflows.run --name github.pr.monitor --args-json '{\"repo\":\"clawdisAI/clawdis\",\"pr\":1200}'"
```

```json
[
  {
    "kind": "github.pr.monitor",
    "repo": "clawdisAI/clawdis",
    "prNumber": 1200,
    "changed": true,
    "summary": {
      "changedFields": [
        "number", "title", "url", "state",
        "isDraft", "mergeable", "reviewDecision",
        "updatedAt", "baseRefName", "headRefName"
      ],
      "changes": {
        "state": { "from": null, "to": "MERGED" },
        "title": {
          "from": null,
          "to": "feat(tui): add syntax highlighting for code blocks"
        }
      }
    }
  }
]
```

### Monitoring Unchanged PRs

```bash
node bin/clawdis.js "workflows.run --name github.pr.monitor --args-json '{\"repo\":\"clawdisAI/clawdis\",\"pr\":1152}'"
```

```json
[
  {
    "kind": "github.pr.monitor",
    "repo": "clawdisAI/clawdis",
    "changed": false,
    "summary": { "changedFields": [], "changes": {} }
  }
]
```

---

## LLM Provider System

Clawdis routes LLM calls through a multi-provider abstraction layer:

```bash
clawdis.llm.invoke --prompt "Summarize this diff"
clawdis.llm.invoke --provider openclaw --prompt "Summarize this diff"
clawdis.llm.invoke --provider pi --prompt "Summarize this diff"
```

### Provider Resolution

| Priority | Source | Variable |
|----------|--------|----------|
| 1 | Explicit flag | `--provider` |
| 2 | Environment | `CLAWDIS_LLM_PROVIDER` |
| 3 | Auto-detect | Scans available credentials |

### Built-in Providers

| Provider | Configuration | Protocol |
|----------|--------------|----------|
| OpenClaw | `OPENCLAW_URL` / `OPENCLAW_TOKEN` | WebSocket RPC |
| Pi | `CLAWDIS_PI_LLM_ADAPTER_URL` | HTTP adapter |
| HTTP | `CLAWDIS_LLM_ADAPTER_URL` | Generic REST |

---

## Tool Invocation

Clawdis installs shim executables for invoking agent tools from workflow steps:

```yaml
steps:
  - id: greeting
    run: >
      clawdis.invoke --tool llm-task --action json --args-json '{"prompt":"Hello"}'
```

### Cross-Step Data Flow

Data passes between steps via structured references -- no temporary files, no text parsing:

```yaml
steps:
  - id: fetch
    run: gh api repos/clawdisAI/clawdis/pulls/42

  - id: analyze
    pipeline: >
      clawdis.llm.invoke --prompt "Review this PR for security issues"
    stdin: $fetch.json

  - id: gate
    approval: Proceed with automated review comment?
    stdin: $analyze.json

  - id: post
    run: gh pr comment 42 --body "$CLAWDIS_ARG_REVIEW"
    when: $gate.approved
```

---

## Commands

| Command | Purpose |
|---------|---------|
| `exec` | Run OS commands with typed I/O |
| `exec --stdin raw/json/jsonl` | Feed pipeline input into subprocess |
| `where` | Filter stage output |
| `pick` | Select fields from JSON objects |
| `head` | Limit output records |
| `json` | JSON renderer |
| `table` | Table renderer |
| `approve` | Approval gate (TTY or agent integration) |

---

## Args and Shell Safety

Every resolved workflow arg is exposed as `CLAWDIS_ARG_<NAME>` (uppercased, non-alphanumeric characters replaced with `_`). The full args object is available as `CLAWDIS_ARGS_JSON`.

```yaml
args:
  text:
    default: ""
steps:
  - id: safe
    env:
      TEXT: "$CLAWDIS_ARG_TEXT"
    command: |
      jq -n --arg text "$TEXT" '{"result": $text}'
```

---

## Quick Start

```bash
git clone https://github.com/ClawdisAI/lobster.git
cd lobster
pnpm install
pnpm test
node ./bin/clawdis.js --help
node ./bin/clawdis.js doctor
node ./bin/clawdis.js "exec --json --shell 'echo [1,2,3]' | where '0>=0' | json"
```

---

## Design Principles

- **Typed pipelines** -- objects and arrays, not text streams
- **Local-first execution** -- no cloud dependency for the runtime
- **No new auth surface** -- Clawdis does not own OAuth tokens or credentials
- **Composable macros** -- agents invoke complex workflows in a single step
- **Deterministic resumability** -- workflows can be paused, gated, and resumed

---

<p align="center">
  <sub>Built by <a href="https://github.com/mbelinky">mbelinky</a>. Follow development at <a href="https://x.com/clawdisAI">@clawdisAI</a>.</sub><br/>
  <sub>A typed workflow engine for autonomous AI agents.</sub>
</p>
