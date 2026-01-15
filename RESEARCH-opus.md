# Flow-Next: Research Document

**Version**: 2.0  
**Date**: 2026-01-13  
**Plugin**: 0.6.1 by Gordon Mickel ([@gmickel](https://twitter.com/gmickel))

---

## Contents

1. [What Is Flow-Next](#1-what-is-flow-next)
2. [Core Principles](#2-core-principles)
3. [System Architecture](#3-system-architecture)
4. [Components](#4-components)
5. [Workflows](#5-workflows)
6. [Ralph: Autonomous Mode](#6-ralph-autonomous-mode)
7. [Context Engineering](#7-context-engineering)
8. [Practical Use](#8-practical-use)
9. [Limits and Trade-offs](#9-limits-and-trade-offs)
10. [Technical Deep Dives](#10-technical-deep-dives)
11. [Conclusion](#11-conclusion)
12. [Reference](#12-reference)

---

## 1. What Is Flow-Next

Flow-Next is a Claude Code plugin for **plan-first orchestration**. It solves four problems AI coding agents face:

| Problem | Solution |
|---------|----------|
| Context drift | Re-read specs before each task |
| Scope creep | Epic-first model with dependencies |
| Quality issues | Cross-model reviews |
| Long tasks | State tracking in `.flow/` |

### Design Philosophy

```
Plan first. Work second. Zero dependencies.
```

Five principles guide the design:

1. **Zero dependencies** — Needs only Python 3.8+
2. **Non-invasive** — No CLAUDE.md edits, no daemons
3. **Clean uninstall** — Delete `.flow/` and done
4. **Multi-user safe** — Scan-based IDs, soft claims
5. **Epic-first** — Every task belongs to an epic

### Repository Structure

```
flow-next/
├── agents/         # 7 subagents for parallel research
├── commands/       # 8 command triggers
├── docs/           # CLI and Ralph guides
├── hooks/          # Ralph workflow guards
├── scripts/        # flowctl CLI (3961 lines Python)
└── skills/         # 11 skill definitions
```

---

## 2. Core Principles

### 2.1 Re-anchoring

The most important principle. Before every task:

```bash
$FLOWCTL show <epic-id> --json    # Read epic
$FLOWCTL cat <epic-id>            # Read spec
$FLOWCTL show <task-id> --json    # Read task
$FLOWCTL cat <task-id>            # Read task spec
git status && git log -5          # Check git state
```

Why? Context windows compress. LLMs forget. The spec is truth. Reading is cheap. Drift is expensive.

### 2.2 Multi-Model Review

Two models catch what one misses.

| Backend | Platform | Method |
|---------|----------|--------|
| RepoPrompt | macOS | Visual builder, full file context |
| Codex CLI | Cross-platform | Terminal, context hints |

Both use the same 7 review criteria:

**Plan Review**: Completeness, Feasibility, Clarity, Architecture, Risks, Scope, Testability

**Impl Review**: Correctness, Simplicity, DRY, Architecture, Edge Cases, Tests, Security

Reviews block until verdict:
```xml
<verdict>SHIP</verdict>
<verdict>NEEDS_WORK</verdict>
<verdict>MAJOR_RETHINK</verdict>
```

### 2.3 Receipt-Based Gating

Treat the agent as untrusted. Receipts prove work was done.

```json
{
  "type": "impl_review",
  "id": "fn-1.1",
  "mode": "codex",
  "verdict": "SHIP",
  "timestamp": "2026-01-13T10:30:00Z"
}
```

No receipt = no progress. Ralph retries until receipt exists.

### 2.4 Fresh Context Per Iteration

Unlike agents that accumulate context (and pollution), Flow-Next Ralph:
- Runs each iteration in a new Claude session
- Starts each task with clean context
- Failed attempts vanish with the session

---

## 3. System Architecture

### 3.1 Data Model

All state lives in `.flow/`:

```
.flow/
├── meta.json           # Schema version
├── config.json         # Settings
├── epics/fn-N.json     # Epic metadata
├── specs/fn-N.md       # Epic spec
├── tasks/fn-N.M.json   # Task metadata
├── tasks/fn-N.M.md     # Task spec
└── memory/             # Persistent learnings
    ├── pitfalls.md
    ├── conventions.md
    └── decisions.md
```

### 3.2 ID Format

- Epic: `fn-N` (e.g., `fn-1`, `fn-42`)
- Task: `fn-N.M` (e.g., `fn-1.1`, `fn-42.7`)

No standalone tasks. Even a single task needs an epic.

### 3.3 Separation of Concerns

| File Type | Content |
|-----------|---------|
| JSON | Metadata (IDs, status, deps, assignee) |
| Markdown | Narrative (specs, descriptions, summaries) |

### 3.4 flowctl CLI

Bundled with the plugin. Not installed globally.

```bash
FLOWCTL="${CLAUDE_PLUGIN_ROOT}/scripts/flowctl"
$FLOWCTL <command>
```

Core commands:
- `init`, `detect` — Setup
- `epic create/set-plan/close` — Epic management
- `task create/set-description/set-acceptance` — Task management
- `ready`, `next`, `start`, `done`, `block` — Workflow
- `validate` — CI/CD validation
- `rp`, `codex` — Review backend wrappers

---

## 4. Components

### 4.1 Subagents (7)

Run in parallel to gather context fast:

| Agent | Purpose | Tools |
|-------|---------|-------|
| context-scout | Token-efficient exploration | rp-cli |
| repo-scout | Find patterns, conventions | Grep, Glob, Read |
| docs-scout | Find documentation | WebSearch, WebFetch |
| practice-scout | Gather best practices | WebSearch |
| flow-gap-analyst | Find edge cases, gaps | Analysis |
| memory-scout | Search project memory | Read |
| quality-auditor | Code review | Grep, Read |

### 4.2 Skills (11)

| Skill | Purpose |
|-------|---------|
| flow-next | Quick task operations |
| flow-next-plan | Turn idea into epic with tasks |
| flow-next-work | Execute tasks systematically |
| flow-next-interview | Deep spec interview (40+ questions) |
| flow-next-plan-review | Carmack-level plan review |
| flow-next-impl-review | Carmack-level impl review |
| flow-next-ralph-init | Setup Ralph harness |
| flow-next-setup | Local flowctl install |
| flow-next-export-context | Export for external LLMs |
| flow-next-rp-explorer | RepoPrompt exploration |
| flow-next-worktree-kit | Git worktree management |

### 4.3 Hooks

Only active when `FLOW_RALPH=1`. Enforce workflow rules:

| Hook | Purpose |
|------|---------|
| PreToolUse | Block dangerous patterns |
| PostToolUse | Track outcomes |
| Stop | Ensure receipt exists |
| SubagentStop | Ensure subagent completes |

Blocked patterns:
- `--json` on chat-send (suppresses review text)
- `--new-chat` on re-reviews (loses context)
- Direct codex calls (must use flowctl wrappers)
- Missing flags on setup-review, select-add
- Receipt writes before review completes

---

## 5. Workflows

### 5.1 Human-in-the-Loop

```
Idea → [Interview] → Plan → [Plan Review] → Work → [Impl Review] → Ship
```

Commands:

| Command | Purpose |
|---------|---------|
| `/flow-next:plan <idea>` | Research and create epic |
| `/flow-next:work <id>` | Execute epic or task |
| `/flow-next:interview <id>` | Deep spec interview |
| `/flow-next:plan-review <id>` | Review plan |
| `/flow-next:impl-review` | Review implementation |

### 5.2 Task States

```
todo → in_progress → done
         ↘ blocked ↗
```

### 5.3 Planning Workflow (steps.md)

1. **Initialize** — `$FLOWCTL init`
2. **Research** — Parallel subagents (context-scout or repo-scout + practice-scout + docs-scout + memory-scout)
3. **Gap analysis** — flow-gap-analyst finds edge cases
4. **Pick depth** — SHORT / STANDARD / DEEP
5. **Write to .flow** — epic create → set-plan → tasks create
6. **Validate** — `$FLOWCTL validate`
7. **Review** — If chosen, run plan-review

### 5.4 Work Workflow (phases.md)

1. **Resolve input** — Epic ID, task ID, or idea text
2. **Branch choice** — Current, new, or worktree
3. **Re-anchor** — EVERY task (mandatory)
4. **Execute** — start → implement → test → commit → done → validate
5. **Quality** — Tests, lint, optional quality-auditor
6. **Ship** — Verify all tasks done
7. **Review** — If chosen, run impl-review

Hard requirements:
- Run `flowctl done` for each task
- Use `git add -A` (never list files)
- Verify task status is `done` before claiming completion

---

## 6. Ralph: Autonomous Mode

### 6.1 Overview

Ralph is the repo-local autonomous loop. Runs overnight with quality gates.

```bash
/flow-next:ralph-init           # Setup
./scripts/ralph/ralph_once.sh   # One iteration (observe)
./scripts/ralph/ralph.sh        # Full loop (AFK)
```

### 6.2 Architecture

```
ralph.sh loop
├── flowctl next
│   ├── status=plan → plan-review
│   └── status=work → work + impl-review
├── Check receipts
├── If missing → retry
├── If SHIP → next task
└── If done → close epics
```

### 6.3 Ralph vs ralph-wiggum (Anthropic)

| Aspect | ralph-wiggum | Flow-Next Ralph |
|--------|--------------|-----------------|
| Session | Single, accumulating | Fresh per iteration |
| Context | Grows, fills up | Clean slate |
| Failed attempts | Pollute future | Gone with session |
| Re-anchoring | None | Every task |
| Quality gates | Tests only | Tests + reviews + receipts |

### 6.4 Configuration (config.env)

```bash
PLAN_REVIEW=codex      # rp, codex, none
WORK_REVIEW=codex      # rp, codex, none
BRANCH_MODE=new        # new, current, worktree
MAX_ITERATIONS=25
MAX_ATTEMPTS_PER_TASK=5
YOLO=1                 # Skip permission prompts
```

### 6.5 Run Artifacts

```
scripts/ralph/runs/<run-id>/
├── iter-001.log        # Raw Claude output
├── progress.txt        # Append-only log
├── attempts.json       # Per-task retry counts
├── receipts/           # Review receipts
└── block-fn-1.2.md     # Auto-blocked task
```

### 6.6 Security

YOLO mode (`--dangerously-skip-permissions`) lets Claude:
- Run any command
- Read/write any file
- Access network

Mitigations:
1. Docker sandbox
2. No secrets in environment
3. Isolated workspace (worktrees)
4. Morning review before merge

---

## 7. Context Engineering

### 7.1 Anthropic's Framework

Context engineering manages tokens across multiple inference steps. Key concepts:

| Concept | Meaning |
|---------|---------|
| Context Rot | Recall drops as tokens grow |
| Attention Budget | Treat context as finite |
| Progressive Disclosure | Load on-demand |

### 7.2 Flow-Next's Strategies

| Strategy | Implementation |
|----------|----------------|
| Re-anchoring | Mandatory spec re-read |
| Subagent isolation | Each agent has own context |
| Token efficiency | `structure` command (10x reduction) |
| Memory system | Learnings outside context |
| Receipt system | Proof prevents pollution |

### 7.3 Token Comparison

| Approach | Tokens |
|----------|--------|
| Full file | ~5000 |
| `structure` | ~500 |
| `read --limit 50` | ~300 |

---

## 8. Practical Use

### 8.1 Single Developer

```bash
/flow-next:plan Add OAuth login
/flow-next:interview fn-1        # Optional
/flow-next:plan-review fn-1      # Optional
/flow-next:work fn-1
/flow-next:impl-review           # Optional
```

### 8.2 Team Workflow

```bash
# Actor A claims task
$FLOWCTL start fn-1.1

# Actor B tries same task
$FLOWCTL start fn-1.1   # Fails: claimed by actor-a@example.com
$FLOWCTL start fn-1.2   # Works: different task
```

### 8.3 Overnight Autonomous

```bash
/flow-next:ralph-init
vim scripts/ralph/config.env
./scripts/ralph/ralph_once.sh    # Observe first
./scripts/ralph/ralph.sh         # Full run
cat scripts/ralph/runs/*/progress.txt  # Morning review
```

### 8.4 CI/CD Integration

```yaml
- run: python3 flowctl.py validate --all --json
```

---

## 9. Limits and Trade-offs

### 9.1 Current Limits

| Limit | Impact |
|-------|--------|
| RepoPrompt macOS only | Cross-platform users need Codex |
| No web UI | CLI/terminal only |
| Python required | Need Python 3.8+ |
| Single repo | No monorepo support |
| No real-time sync | Soft claims only |

### 9.2 Memory System

Optional persistent learnings:

```bash
$FLOWCTL config set memory.enabled true
$FLOWCTL memory init
$FLOWCTL memory add --type pitfall "Always use flowctl rp wrappers"
```

Types: `pitfall`, `convention`, `decision`

Auto-capture: When review returns NEEDS_WORK, Ralph captures issues to `pitfalls.md`.

---

## 10. Technical Deep Dives

### 10.1 Hook System (ralph-guard.py)

566-line Python script handling all hook events:

**PreToolUse**: Block dangerous patterns before execution
```python
def handle_pre_tool_use(data: dict) -> None:
    # Block --json on chat-send
    # Block --new-chat on re-reviews
    # Block direct codex calls
    # Validate required flags
    # Block receipt writes before review
```

**PostToolUse**: Track outcomes
```python
def handle_post_tool_use(data: dict) -> None:
    # Count chats_sent
    # Track chat_send_succeeded
    # Track flowctl_done_called
    # Parse verdict
```

**Stop**: Ensure receipt exists before Claude stops

State stored in `/tmp/ralph-guard-{session_id}.json`

### 10.2 flowctl CLI (flowctl.py)

3961-line Python implementation. Key features:

**Atomic writes**:
```python
def atomic_write(path: Path, content: str) -> None:
    fd, tmp_path = tempfile.mkstemp(dir=path.parent)
    # Write to temp, then rename (atomic on POSIX)
    os.replace(tmp_path, path)
```

**Symbol extraction** for context hints:
- Python: `def`, `class`, `__all__`
- JS/TS: `export function/class/const`
- Go: `func`, `type`
- Rust: `pub fn`, `struct`, `enum`, `trait`, `impl`
- C/C++: function definitions, struct, typedef
- Java: class, interface, method

**Session continuity**:
```python
def run_codex_exec(prompt: str, session_id: Optional[str] = None):
    # If session_id exists, resume conversation
    # Thread ID stored in receipt for next review
```

### 10.3 RepoPrompt Integration

**rp-cli commands used**:

| Command | Purpose |
|---------|---------|
| `windows` | List windows |
| `structure` | Code signatures (10x fewer tokens) |
| `builder` | AI-powered file selection |
| `chat` | Send to AI |
| `select` | Manage file selection |
| `context` | Export context |

**Atomic setup-review**:
```bash
eval "$(flowctl rp setup-review --repo-root "$REPO_ROOT" --summary "...")"
# Outputs: W=1 T=abc123
```

Combines pick-window + builder in one atomic operation. Prevents Claude from skipping steps.

### 10.4 Claude Code Plugin Architecture

Five pillars of extensibility:

1. **Skills** — SKILL.md files with instructions
2. **Commands** — Slash command triggers
3. **Subagents** — Parallel isolated agents
4. **Hooks** — Lifecycle event handlers
5. **Marketplace** — Plugin distribution

SKILL.md format:
```yaml
---
name: flow-next-plan
description: "Create structured build plans..."
---

# Flow plan
[Instructions in markdown]
```

Key variables:
- `$ARGUMENTS` — User input
- `$CLAUDE_PLUGIN_ROOT` — Plugin path

### 10.5 Interview Question Categories

10 categories, 40+ questions typical:

1. **Technical** — Data structures, edge cases, state, concurrency
2. **Architecture** — Boundaries, integration, dependencies, APIs
3. **Error Handling** — What fails? Recovery? Timeouts?
4. **Performance** — Load, latency, memory, caching
5. **Security** — Auth, validation, sensitivity, attacks
6. **UX** — Loading, errors, offline, accessibility
7. **Testing** — Unit focus, integration, E2E, mocking
8. **Migration** — Breaking changes, data migration, rollback
9. **Acceptance** — What's done? How verify? Benchmarks
10. **Risks** — Uncertainty, blockers, research, external deps

---

## 11. Conclusion

### What Flow-Next Does Well

| Problem | Solution |
|---------|----------|
| Context drift | Re-anchor from .flow/ specs |
| Scope creep | Epic-first with dependencies |
| Quality issues | Cross-model Carmack reviews |
| Autonomous reliability | Receipt gating + fresh context |
| Long task management | flowctl state tracking |

### Alignment with Anthropic Guidelines

- ✓ Context curation — Subagents filter, memory-scout retrieves relevant only
- ✓ Re-anchoring — Mandatory spec re-read
- ✓ Attention budget — Token-efficient `structure` (10x reduction)
- ✓ Progressive disclosure — Skills load on-demand
- ✓ Structured notes — .flow/memory/ system

### Strengths

1. Zero dependencies (Python only)
2. Non-invasive (no CLAUDE.md edits)
3. Multi-user safe (scan-based IDs)
4. CI/CD ready (`validate --all`)
5. Cross-platform reviews (Codex CLI)

### Limits

1. RepoPrompt macOS only
2. No web UI
3. Single repo focus
4. Python runtime required

### When to Use

| Scenario | Approach |
|----------|----------|
| Quick feature | plan + work |
| Complex spec | Add interview |
| Quality-critical | Enable reviews |
| Overnight autonomous | Ralph + Docker sandbox |
| Team work | Soft claims + CI validation |

---

## 12. Reference

### Primary Sources
- [README.md](README.md)
- [flowctl CLI Reference](docs/flowctl.md)
- [Ralph Guide](docs/ralph.md)

### Anthropic Documentation
- [Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- [Model Context Protocol](https://docs.claude.com/en/docs/mcp)

### External Tools
- [RepoPrompt](https://repoprompt.com/) — Visual context engineering (macOS)
- [OpenAI Codex CLI](https://github.com/openai/codex) — Cross-platform review

### Source Code Analyzed
- `scripts/flowctl.py` — 3961 lines, core CLI
- `scripts/hooks/ralph-guard.py` — 566 lines, hook enforcement
- `skills/*/SKILL.md` — 11 skill definitions
- `agents/*.md` — 7 subagent definitions

---

## Appendix A: Quick Reference

### Commands

```bash
# Planning
/flow-next:plan <idea>
/flow-next:interview fn-1
/flow-next:plan-review fn-1

# Working
/flow-next:work fn-1
/flow-next:impl-review

# Ralph
/flow-next:ralph-init
./scripts/ralph/ralph.sh
```

### flowctl

```bash
$FLOWCTL init
$FLOWCTL epic create --title "..."
$FLOWCTL task create --epic fn-1 --title "..."
$FLOWCTL start fn-1.1
$FLOWCTL done fn-1.1 --summary-file s.md --evidence-json e.json
$FLOWCTL validate --all
```

---

## Appendix B: File Formats

### Epic JSON (fn-1.json)
```json
{
  "id": "fn-1",
  "title": "Add OAuth login",
  "status": "open",
  "plan_review_status": "ship",
  "branch_name": "fn-1"
}
```

### Task JSON (fn-1.1.json)
```json
{
  "id": "fn-1.1",
  "epic": "fn-1",
  "title": "Create OAuth routes",
  "status": "done",
  "depends_on": [],
  "assignee": "user@example.com"
}
```

### Review Receipt
```json
{
  "type": "impl_review",
  "id": "fn-1.1",
  "mode": "codex",
  "verdict": "SHIP",
  "session_id": "thread_abc123",
  "timestamp": "2026-01-13T12:30:00Z"
}
```

---

## Appendix C: Glossary

| Term | Meaning |
|------|---------|
| Epic | Container for tasks (fn-N) |
| Task | Single work unit (fn-N.M) |
| Re-anchoring | Reading specs before each task |
| Receipt | JSON proof of review completion |
| Verdict | SHIP, NEEDS_WORK, MAJOR_RETHINK |
| Ralph | Autonomous loop harness |
| Context Rot | Loss of recall as tokens grow |
| Builder | RepoPrompt's AI file selector |
| flowctl | Bundled CLI for .flow/ |
| Carmack-level | Thorough review (ref: John Carmack) |

---

## Impressive Points

After researching the entire codebase, five aspects stand out:

1. **Re-anchoring as Core Principle**  
   The mandatory spec re-read before every task is simple but powerful. Most autonomous agents accumulate context until they drift. Flow-Next forces fresh context from the source of truth. This one principle prevents most failure modes.

2. **Cross-Model Review Gates**  
   Using a different model (GPT via Codex or RepoPrompt) to review Claude's work catches blind spots self-review misses. The receipt-based gating ensures reviews actually run—Claude can't skip them and claim completion.

3. **Fresh Context Per Iteration**  
   Ralph's architecture inverts the typical autonomous loop. Instead of accumulating context in one session (where failed attempts pollute future work), each iteration starts clean. Failed attempts vanish with the session.

4. **Token Efficiency Through Structure**  
   The `structure` command extracts code signatures at 10x fewer tokens than reading full files. Combined with targeted `read --limit`, this makes large codebase exploration practical within context limits.

5. **Epic-First Model**  
   No standalone tasks. Every piece of work belongs to an epic. This sounds restrictive but simplifies everything: dependencies always exist within an epic, re-anchoring always has parent context, and Ralph always knows the scope.

---

## Summary

Flow-Next is a Claude Code plugin that solves the core problems of AI coding agents: context drift, scope creep, quality assurance, and long-running task management.

The plugin uses five principles: plan-first methodology, mandatory re-anchoring before every task, cross-model reviews (GPT reviews Claude's work), receipt-based gating (proof-of-work for autonomous operations), and fresh context per iteration (each task starts clean).

Architecture centers on `.flow/` directory for state, `flowctl` CLI for operations, 7 subagents for parallel research, and 11 skills for different workflows. Ralph provides autonomous overnight runs with quality gates.

Key technical innovations include atomic operations for data integrity, symbol extraction for context hints, hook-based enforcement for Ralph mode, and RepoPrompt/Codex integration for cross-model reviews.

The plugin directly implements Anthropic's context engineering recommendations: context curation, re-anchoring, attention budgets, progressive disclosure, and structured note-taking.

Flow-Next is production-ready for solo developers (plan + work), teams (soft claims + CI validation), quality-critical work (Carmack-level reviews), and overnight autonomous runs (Ralph + Docker sandbox).

---

*Research completed 2026-01-13 by AI Research Agent*
