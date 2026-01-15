# Flow-Next — Research (GPT)

**Date**: 2026-01-13  
**Plugin version (this repo)**: 0.6.1 (`.claude-plugin/plugin.json`)  
**Scope**: Tài liệu này mô tả **repo `flow-next/` hiện tại**: mục đích, nguyên lý thiết kế, cấu trúc code/prompt, và cách dùng trong thực tế (manual + autonomous).

---

## 0) Self-check / TODO (đã làm)

Các câu hỏi mình tự đặt ra khi đọc repo, và trạng thái:

- [x] Repo này thực sự “là gì”: code chạy ở đâu, entrypoint nào? (Claude Code plugin + CLI `flowctl`)
- [x] Dữ liệu và state nằm ở đâu, schema thế nào? (`.flow/` + JSON/MD, schema v2, tương thích v1)
- [x] Thuật toán chọn “việc tiếp theo” hoạt động ra sao? (`flowctl next`: plan-gate, epic deps, resume in-progress, priority+deps)
- [x] Review backend “rp” vs “codex” hoạt động thế nào? (wrappers + receipt + verdict parsing + session continuity)
- [x] Ralph autonomous loop chạy thế nào *theo đúng template*? (watch mode, worker timeout, receipts, attempts, branches.json)
- [x] Hooks bảo vệ Ralph thực sự enforce cái gì, chỉ bật khi nào? (`FLOW_RALPH=1`, receipt gating, anti-pattern blocks)
- [x] Memory system có “auto-capture” thật không? (**Không**: hook chỉ nhắc; `flowctl memory add` là manual)
- [x] So khớp docs/prompt với code để bắt mismatch quan trọng:
  - `flowctl config get review.backend` cần `--json` nếu parse bằng `jq`
  - README đang nhắc `flowctl task set ... --status` nhưng CLI không có command đó
  - docs/README nói “auto-capture memory” nhưng hook hiện tại không tự ghi file memory
- [x] Danh sách commands/skills/subagents và luồng nối giữa chúng (commands → skills → flowctl/subagents → hooks)
- [x] Setup/uninstall thực tế tạo/xoá gì? (`/flow-next:setup`, `/flow-next:ralph-init`, cleanup)
- [x] Receipt schema + validate gate check gì? (`type/id` bắt buộc; Codex receipt có `session_id` + `review`)
- [x] Repo có “self-tests” nào để verify flowctl/ralph templates? (smoke + e2e scripts trong `scripts/`)

Nếu bạn muốn mình tiếp tục “đào sâu” theo hướng khác (ví dụ: hardening CI, threat model chi tiết, hoặc chuẩn hoá prompt), nói rõ tiêu chí “mỹ mãn” mong muốn.

---

## 1) Flow-Next là gì (theo repo này)

Flow-Next là một **Claude Code plugin** tập trung vào “plan-first orchestration” cho coding agent:

- **Plan trước, work sau**: biến yêu cầu thô thành epic + tasks nhỏ, có deps rõ ràng.
- **State nằm ngoài context**: lưu ở `.flow/` để tránh drift khi context dài/compaction.
- **Re-anchoring bắt buộc**: trước mỗi task phải đọc lại spec + trạng thái từ `.flow/`.
- **Quality gates**: review “Carmack-level” bởi backend khác model (RepoPrompt hoặc Codex CLI).
- **Autonomous mode (Ralph)**: loop ngoài Claude bằng shell, mỗi iteration là phiên chạy mới → giảm “context pollution”.

Repo này chứa cả:

- Prompt/skill/agent definitions (Markdown) cho Claude Code.
- CLI `flowctl` (Python) để đọc/ghi `.flow/` + wrappers cho RepoPrompt/Codex.
- Templates để scaffold `scripts/ralph/` và `.flow/bin/` khi chạy setup.
- Hook guard để enforce quy tắc khi chạy Ralph.

---

## 2) Bản đồ repo (các phần quan trọng)

```
flow-next/
├── .claude-plugin/plugin.json              # metadata plugin (name/version/desc)
├── commands/flow-next/*.md                 # slash commands → chỉ gọi skill tương ứng
├── skills/*/SKILL.md                       # “luật chơi” cho từng workflow (plan/work/review/setup/…)
│   ├── flow-next-plan/steps.md             # các bước plan chi tiết (subagents + ghi .flow)
│   ├── flow-next-work/phases.md            # các pha work chi tiết (re-anchor, start/done, tests, review)
│   ├── flow-next-setup/templates/*         # snippet chèn vào CLAUDE.md/AGENTS.md + .flow/usage.md
│   └── flow-next-ralph-init/templates/*    # scaffold scripts/ralph/ (ralph.sh, prompt_plan/work,…)
├── agents/*.md                             # subagents: repo-scout, context-scout, …
├── scripts/flowctl.py                      # core CLI (Python) quản lý .flow/ + review wrappers
├── scripts/hooks/ralph-guard.py            # hook guard (active khi FLOW_RALPH=1)
├── hooks/hooks.json                        # đăng ký hook gọi ralph-guard.py
└── docs/*.md                               # human docs (flowctl, ralph, CI example)
```

Ghi chú nhỏ: `.mx_hybrid/` và `.x-droid/` trông như artifacts nội bộ của môi trường dev (không phải core của plugin).

---

## 3) Model dữ liệu: `.flow/` (source of truth)

### 3.1 Cấu trúc thư mục

`flowctl init` tạo:

```
.flow/
├── meta.json            # {schema_version: 2, next_epic: 1} (counter không còn là source of truth)
├── config.json          # default: {"memory":{"enabled":false}}
├── epics/fn-N.json      # metadata epic
├── specs/fn-N.md        # epic spec (markdown)
├── tasks/fn-N.M.json    # metadata task
├── tasks/fn-N.M.md      # task spec (markdown, có headings bắt buộc)
└── memory/              # pitfalls/conventions/decisions (opt-in)
```

Nguyên tắc:
- **JSON = state machine** (status, deps, assignee, timestamps…)
- **Markdown = narrative/spec** (mô tả, acceptance, done summary, evidence…)

### 3.2 ID format (epic-first)

- Epic: `fn-N` (ví dụ `fn-1`)
- Task: `fn-N.M` (ví dụ `fn-1.2`)

Không có “standalone task” độc lập: mọi task thuộc về một epic.

### 3.3 Task spec headings (bắt buộc)

`flowctl validate` yêu cầu task markdown có đủ đúng 1 lần:

- `## Description`
- `## Acceptance`
- `## Done summary`
- `## Evidence`

`flowctl done` sẽ patch vào `Done summary` và `Evidence` (thay vì agent tự ghi rời rạc).

---

## 4) `flowctl` (Python CLI): mục đích và nguyên lý

`scripts/flowctl` là wrapper bash gọi `scripts/flowctl.py`.

### 4.1 Triết lý thiết kế

- **All writes qua flowctl**: hạn chế edit tay `.flow/*` để tránh lệch schema/headings.
- **Atomic write**: ghi file qua temp + `os.replace` để giảm corruption.
- **Merge-safe ID allocation**: không dùng counter tăng dần; thay vào đó **scan file hiện có** để chọn ID tiếp theo.
- **Multi-user soft-claim**: task có `assignee`, `claimed_at`, `claim_note` để tránh đụng nhau (không cần server).

### 4.2 Các hành vi quan trọng (đúng theo code)

**Commands chính (đúng theo argparse trong `flowctl.py`)**

```
init, detect, config, memory,
epic (create, set-plan, set-plan-review-status, set-branch, close),
task (create, set-description, set-acceptance),
dep (add),
show, epics, tasks, list, cat,
ready, next, start, done, block,
validate,
prep-chat (legacy),
rp (windows, pick-window, ensure-workspace, builder, prompt-get/set, select-get/add, chat-send, prompt-export, setup-review),
codex (check, impl-review, plan-review)
```

**Scan-based allocation**

- `flowctl epic create`: lấy `max(fn-*.json) + 1`
- `flowctl task create`: lấy `max(fn-N.*.json) + 1`
- `meta.json.next_epic` vẫn tồn tại nhưng **không còn là source of truth** (để giảm conflict khi merge).

**Soft-claim**

- `flowctl start fn-N.M`:
  - kiểm deps đã `done` (trừ khi `--force`)
  - set `status=in_progress`
  - set `assignee` (theo `FLOW_ACTOR` → git email → git name → `$USER` → `unknown`)
- `flowctl done fn-N.M`:
  - mặc định yêu cầu task đang `in_progress` và `assignee` khớp actor (trừ `--force`)
  - patch task markdown (summary + evidence), rồi set `status=done` trong JSON.

**Chọn task “ready”**

- `flowctl ready --epic fn-N`:
  - `ready`: task `todo` và mọi deps `done`
  - `in_progress`: tách riêng để dễ thấy ai đang làm
  - `blocked`: task `blocked` hoặc bị deps chưa done/missing
  - sort theo `(priority asc, task_num asc, title)`

### 4.3 `flowctl next`: thuật toán chọn “plan hay work”

Đây là điểm “động cơ” cho Ralph:

1. Lấy danh sách epics (từ `--epics-file` hoặc scan `.flow/epics/`).
2. Bỏ epic đã `status=done`.
3. Epic-level deps:
   - nếu `depends_on_epics` có epic chưa done → epic bị block.
4. Nếu `--require-plan-review` và epic `plan_review_status != ship` → trả về `status=plan`.
5. Với epic hợp lệ:
   - ưu tiên “resume” task `in_progress` mà `assignee == current_actor`
   - nếu không có, chọn task `todo` mà deps done, sort theo `(priority, task_num)`
6. Nếu không còn gì:
   - trả `status=none`
   - nếu bị block bởi epic deps, trả thêm `blocked_epics`.

Output JSON (khi `--json`) có dạng:

```json
{"status":"work|plan|none","epic":"fn-1","task":"fn-1.2","reason":"ready_task|resume_in_progress|needs_plan_review|..."}
```

### 4.4 `validate`: những invariant nào được check

`flowctl validate` có 2 mode:

- `validate --epic fn-N`: validate 1 epic
- `validate --all`: validate root + mọi epics

Các check quan trọng:
- `.flow/meta.json` tồn tại và `schema_version ∈ {1,2}`
- thư mục con `epics/ specs/ tasks/ memory/` tồn tại
- epic spec `.flow/specs/fn-N.md` tồn tại
- mỗi task có spec `.flow/tasks/fn-N.M.md` và đủ headings bắt buộc
- task deps phải tồn tại và **không được trỏ ra ngoài epic**
- detect dependency cycle (DFS)
- epic deps (`depends_on_epics`) phải là list epic IDs hợp lệ và không self-reference
- nếu epic `status=done` thì mọi task phải `done`

---

## 5) Review: RepoPrompt (rp) vs Codex CLI (codex)

Repo này implement **hai backend review**; mục tiêu là reviewer model có đủ context để đưa verdict chuẩn hoá:

```
<verdict>SHIP</verdict>
<verdict>NEEDS_WORK</verdict>
<verdict>MAJOR_RETHINK</verdict>
```

### 5.1 RepoPrompt backend (`flowctl rp ...`)

`flowctl` bọc `rp-cli` để giảm lỗi “agent quên bước”:

- `flowctl rp windows` → raw windows JSON
- `flowctl rp setup-review --repo-root ... --summary ...`
  - tìm window phù hợp theo repo root
  - gọi `builder "<summary>"`
  - in ra `W=<id> T=<tab>`
- `flowctl rp select-add/select-get/prompt-get/prompt-set/chat-send/prompt-export`

Điểm quan trọng:
- `chat-send --json` **sẽ mất review text** (chỉ còn chat id) → Ralph guard sẽ block pattern này.
- Re-review phải **giữ nguyên chat** (không `--new-chat`) để reviewer có context.

### 5.2 Codex backend (`flowctl codex ...`)

`flowctl codex impl-review` và `flowctl codex plan-review`:

- Chạy `codex exec` với:
  - model mặc định `gpt-5.2`
  - `model_reasoning_effort="high"`
  - `--sandbox read-only`
  - `--json` để lấy `thread_id` và lưu session continuity.
- Tự parse verdict bằng regex từ output.
- Nếu có `--receipt <path>`:
  - đọc `session_id` cũ để `codex exec resume <session_id>`
  - ghi receipt JSON (kèm `review` text đầy đủ).

Chi tiết hữu ích:
- `impl-review` hỗ trợ **standalone** (không task id): review toàn branch vs base branch (dùng `--focus` + diff summary).

### 5.3 Context hints cho Codex review

Vì Codex reviewer không có “builder” như RepoPrompt, `flowctl` tạo “context_hints”:

- lấy danh sách changed files vs base branch
- extract symbols (JS/TS exports, Python def/class, Go/Rust/…)
- `git grep` các symbol đó ở file khác để gợi ý reviewer đọc thêm

Đây là heuristic để giảm nguy cơ reviewer chỉ nhìn diff mà thiếu phụ thuộc/caller.

---

## 6) Hooks: `ralph-guard.py` (enforcement khi chạy Ralph)

Hook được đăng ký tại `hooks/hooks.json` và **chỉ active khi** `FLOW_RALPH=1` (script tự early-exit nếu không).

Mục tiêu: xem agent như “untrusted worker” và chặn các lỗi quy trình gây phá automation.

### 6.1 PreToolUse: chặn trước khi chạy command

Ví dụ các rule:

- Block `chat-send --json` (vì mất review text → không parse được verdict).
- Block `--new-chat` ở re-review (giữ continuity).
- Block gọi `codex exec`/`codex review` trực tiếp (phải dùng wrappers của flowctl).
- Enforce `flowctl done` phải có `--summary-file` và `--evidence-json`.
- Block việc “ghi receipt” trước khi review thực sự trả về.

### 6.2 PostToolUse + Stop

- Track số lần chat-send, last verdict, tasks đã `flowctl done`.
- Nếu thấy SHIP verdict nhưng chưa có receipt (rp mode) → hook nhắc command ghi receipt.
- Khi Stop, nếu `REVIEW_RECEIPT_PATH` set mà file chưa tồn tại → **block stop** và đưa command để ghi receipt.

### 6.3 Memory reminder (không auto-write)

Khi verdict là `NEEDS_WORK/MAJOR_RETHINK` và memory bật, hook chỉ:
- nhắc “nếu có lesson generalizable thì ghi `flowctl memory add ...`”

Không có đoạn code tự động patch `.flow/memory/*` nếu agent không làm (dù một số docs/README đang mô tả “auto-capture”).

---

## 7) Ralph autonomous harness (scaffold từ templates)

`/flow-next:ralph-init` (skill) sẽ tạo `scripts/ralph/` trong repo bạn đang làm việc bằng cách copy templates:

- `scripts/ralph/ralph.sh`: loop chính (gọi `flowctl next` → chạy plan/work prompt)
- `scripts/ralph/ralph_once.sh`: chạy đúng 1 iteration để quan sát
- `scripts/ralph/prompt_plan.md`, `scripts/ralph/prompt_work.md`: prompt cho từng iteration
- `scripts/ralph/flowctl`, `scripts/ralph/flowctl.py`: copy CLI vào run harness
- `scripts/ralph/config.env`: config review backend/branch mode/limits
- `scripts/ralph/runs/<run-id>/...`: logs, receipts, attempts, progress

### 7.0 Config knobs quan trọng (template `config.env`)

- Scope: `EPICS=` (rỗng = scan tất cả epic open)
- Plan gate: `PLAN_REVIEW=rp|codex|none`, `REQUIRE_PLAN_REVIEW=0|1`
- Work gate: `WORK_REVIEW=rp|codex|none`
- Loop limits: `MAX_ITERATIONS`, `MAX_ATTEMPTS_PER_TASK`, (optional) `MAX_TURNS`
- Permissions: `YOLO=1` (unattended)
- Branching: `BRANCH_MODE=new|current|worktree`
- Observability: `RALPH_UI=0` (disable pretty output), `--watch` / `--watch verbose`
- Timeout: `WORKER_TIMEOUT` (mặc định 1800s trong `ralph.sh`; nếu có `timeout/gtimeout` thì enforce)

### 7.1 Loop logic (đúng theo template `ralph.sh`)

- Mỗi iteration:
  - chạy `flowctl next --json` (optionally scoped epics + require plan review)
  - nếu `status=plan`: chạy prompt plan gate
  - nếu `status=work`: chạy prompt work gate
- Gọi Claude bằng `claude -p "<rendered prompt>"` với `FLOW_RALPH=1`.
  - mặc định mỗi iteration là session mới (giảm context pollution); có thể override qua `FLOW_RALPH_CLAUDE_SESSION_ID` trong `config.env`.
  - luôn append system prompt “AUTONOMOUS MODE ACTIVE …” để ép tuân thủ receipts/verify/skills.
- Sau khi Claude chạy xong:
  - verify receipt JSON tồn tại và `type/id` khớp (nếu review != none)
  - verify `flowctl show <task>` là `status=done` (với work)
  - nếu fail → force retry
  - bump attempts theo task; quá `MAX_ATTEMPTS_PER_TASK` → auto-block task bằng `flowctl block`.
- Khi `status=none`: có thể auto-close epics trong scope (nếu mọi task done), rồi `<promise>COMPLETE</promise>`.

### 7.1.1 Watch mode (TUI) + stream-json

Template hỗ trợ 2 chế độ watch:

- `./scripts/ralph/ralph.sh --watch`: stream tool calls (lọc bằng `scripts/ralph/watch-filter.py`)
- `./scripts/ralph/ralph.sh --watch verbose`: stream tool calls + text/thinking

Kỹ thuật:
- dùng `claude --output-format stream-json` rồi `tee` ra `iter-XXX.log`
- `watch-filter.py` “fail open” (drain stdin khi pipe lỗi) để tránh SIGPIPE làm chết whole pipeline

### 7.2 Branch modes

Trong config:
- `BRANCH_MODE=new`: tạo 1 run branch `ralph-<run-id>` dùng cho toàn run; khi prompt work chạy, branch mode effective là `current`.
- `BRANCH_MODE=current`: làm luôn trên branch hiện tại.
- `BRANCH_MODE=worktree`: workflow khác (kết hợp worktree kit).

Ngoài ra, run meta được ghi vào `scripts/ralph/runs/<run-id>/branches.json` (`base_branch`, `run_branch`) để resume/debug.

### 7.3 Security / safety notes

- `YOLO=1` thêm `--dangerously-skip-permissions` cho Claude để chạy unattended.
- Thực tế an toàn phụ thuộc sandbox/container và việc repo có/không có secrets.
- Review gate + receipt chỉ đảm bảo “quy trình” chứ không tự động ngăn exfiltration nếu environment đã lộ secrets.

---

## 8) Plugin architecture (commands/skills/agents)

### 8.1 Commands

Các file trong `commands/flow-next/*.md` chỉ có 1 nhiệm vụ: **invoke skill tương ứng**.

Ví dụ:
- `/flow-next:plan` → `skill flow-next-plan`
- `/flow-next:work` → `skill flow-next-work`
- `/flow-next:plan-review` → `skill flow-next-plan-review`
- `/flow-next:impl-review` → `skill flow-next-impl-review`

### 8.2 Skills

Skills định nghĩa “protocol” để agent làm việc ổn định:

- **flow-next (task mgmt)**: thao tác nhanh `.flow/` (list/show/create task/validate/…) khi user hỏi “show me my tasks”, không thay cho `/flow-next:plan` hay `/flow-next:work`.
- **Plan**: init `.flow` → research subagents → gap analysis → ghi epic/tasks bằng flowctl → validate → optional plan review loop.
- **Work**: hỏi branch mode → re-anchor mỗi task → start → implement+tests → commit → `flowctl done` (kèm evidence) → amend commit để include `.flow/` → validate → optional impl review loop.
- **Setup**: copy flowctl vào `.flow/bin`, tạo `.flow/usage.md`, chèn snippet vào CLAUDE.md/AGENTS.md theo markers.
- **Ralph-init**: scaffold `scripts/ralph/`.
- **Worktree-kit**: script an toàn tạo/list/switch/cleanup worktrees trong `.worktrees/`.
- **Export-context**: dùng RepoPrompt builder + prompt export để review bằng LLM bên ngoài (ChatGPT/Claude web).
- **RP-explorer**: “use rp to …” kiểu token-efficient exploration (đi thẳng rp-cli).
- **Interview**: hỏi sâu (40+ câu) và ghi lại vào epic/task spec (skill này yêu cầu dùng AskUserQuestion tool trong Claude).

### 8.3 Subagents

`agents/*.md` mô tả các “vai phụ” dùng để thu thập context nhanh:

- `repo-scout`: grep/read nhanh, tìm conventions và reuse points
- `context-scout`: dùng RepoPrompt builder để khám phá token-efficient
- `docs-scout`, `practice-scout`: tìm docs/best practices
- `flow-gap-analyst`: edge cases, câu hỏi còn thiếu
- `memory-scout`: tra `.flow/memory` (khi bật)
- `quality-auditor`: soi diff về correctness/security/tests trước ship

---

## 9) Cách dùng trong thực tế (gợi ý workflow)

### 9.0 Cài plugin (Claude Code)

Theo `README.md`, luồng cài phổ biến (từ marketplace repo của tác giả) là:

```bash
/plugin marketplace add https://github.com/gmickel/gmickel-claude-marketplace
/plugin install flow-next
```

### 9.1 Manual (human-in-the-loop)

1. `/flow-next:plan <idea>`
2. (tuỳ) `/flow-next:interview fn-N`
3. (tuỳ) `/flow-next:plan-review fn-N`
4. `/flow-next:work fn-N` (hoặc từng task `fn-N.M`)
5. (tuỳ) `/flow-next:impl-review`

### 9.2 Team (merge-safe + soft-claim)

- Mỗi người `flowctl start fn-N.M` để claim.
- Nếu đụng nhau, người sau sẽ bị chặn (trừ khi `--force` takeover).
- Thêm CI gate: `flowctl validate --all` để PR không làm bể `.flow/`.

### 9.3 Autonomous (Ralph)

1. `/flow-next:ralph-init`
2. chỉnh `scripts/ralph/config.env`
3. chạy `scripts/ralph/ralph_once.sh` để quan sát
4. chạy `scripts/ralph/ralph.sh` để loop
5. xem `scripts/ralph/runs/*/progress.txt` vào sáng hôm sau

---

## 10) Pitfalls / mismatches đáng nhớ

- `flowctl config get review.backend` muốn parse bằng `jq` thì phải dùng `--json`; nếu không output là text (không phải JSON).
- `.claude-plugin/plugin.json` đang ghi “6 subagents” nhưng thư mục `agents/` trong repo hiện có 7 file (có thêm `memory-scout`).
- README đang nhắc `flowctl task set fn-1.2 --status pending` (reset task) nhưng trong `scripts/flowctl.py` hiện **không có** subcommand `task set`/`--status` (chỉ có `task set-description` và `task set-acceptance`).
- docs/README có đoạn nói “NEEDS_WORK reviews auto-capture learnings to `.flow/memory/pitfalls.md`”, nhưng hook `scripts/hooks/ralph-guard.py` hiện chỉ **nhắc** người/agent chạy `flowctl memory add` (không tự ghi).
- Không nên đặt “artifact files” (evidence/summary tạm) vào `.flow/tasks/`:
  - một số command đã cố skip (GH-21), nhưng không phải command nào cũng immune (ví dụ: `epics`/`epic close` có thể bị ảnh hưởng nếu file JSON không giống task schema).
- RepoPrompt review:
  - đừng dùng `chat-send --json` (mất review text)
  - re-review phải cùng chat (không `--new-chat`)
- Codex review:
  - dùng wrappers `flowctl codex ...` để giữ receipt + session continuity; đừng gọi `codex exec` trực tiếp trong Ralph.

---

## 11) Setup / uninstall (repo-local)

### 11.1 `/flow-next:setup` (optional)

Theo `skills/flow-next-setup/workflow.md`, setup sẽ (trong repo bạn đang làm):

- tạo `.flow/bin/flowctl` và `.flow/bin/flowctl.py` (copy từ plugin)
- tạo `.flow/usage.md` (từ template)
- ghi `setup_version` + `setup_date` vào `.flow/meta.json`
- tuỳ chọn chèn snippet giữa markers `<!-- BEGIN FLOW-NEXT --> ... <!-- END FLOW-NEXT -->` vào `CLAUDE.md`/`AGENTS.md`

### 11.2 `/flow-next:ralph-init`

Tạo `scripts/ralph/` (nếu chưa tồn tại) bằng cách copy templates + copy flowctl vào harness.

### 11.3 Uninstall

- Xoá state flow: `rm -rf .flow/`
- Xoá autonomous harness: `rm -rf scripts/ralph/`
- Gỡ snippet khỏi `CLAUDE.md`/`AGENTS.md` (xoá block markers)

---

## 12) Repo self-tests (smoke/e2e) cho chính plugin này

Repo có sẵn một số script “tự kiểm” (không phải unit tests) để verify `flowctl` và templates Ralph:

- `scripts/smoke_test.sh`: smoke test cho `flowctl` (next/ready/validate/blocked/artifact files/memory/config…).
- `scripts/ralph_smoke_test.sh`: chạy Ralph với stub `claude` để test receipts + loop logic + artifacts (nhanh, không cần LLM thật).
- `scripts/ralph_e2e_test.sh`: e2e với `claude` thật (yêu cầu Claude Code CLI + plugin dir).
- `scripts/ralph_smoke_rp.sh`: smoke Ralph (rp backend) nếu môi trường có `rp-cli`.
- `scripts/ralph_e2e_rp_test.sh` / `scripts/ralph_e2e_short_rp_test.sh`: e2e Ralph với rp backend (chạy thật, chậm hơn).
- `scripts/plan_review_prompt_smoke.sh`: chuẩn bị repo + prompt file để smoke plan-review prompt trong rp mode (cần `rp-cli`).

Các script này có “safety check” để không chạy nhầm ngay trong marketplace repo (tìm `.claude-plugin/marketplace.json` / `plugins/flow-next/...`).

---

## 13) Reference

- `README.md`
- `docs/flowctl.md`
- `docs/ralph.md`
- `scripts/flowctl.py`
- `scripts/hooks/ralph-guard.py`
- `skills/flow-next-ralph-init/templates/ralph.sh`
- `skills/flow-next-ralph-init/templates/watch-filter.py`
- `scripts/smoke_test.sh`
- `scripts/ralph_smoke_test.sh`
- `scripts/ralph_smoke_rp.sh`
- `scripts/ralph_e2e_test.sh`
- `scripts/ralph_e2e_rp_test.sh`

### External references (liên quan chủ đề)

- RepoPrompt: https://repoprompt.com/
- OpenAI Codex CLI: https://github.com/openai/codex
- Effective Context Engineering (Anthropic): https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
