# Flow-Next (repo `flow-next/`) — Research (GPT)

**Last updated**: 2026-01-25  
**Plugin version**: 0.18.16 (`.claude-plugin/plugin.json`)  
**Phạm vi**: Repo plugin `flow-next/` hiện tại: mục đích, cấu trúc, dữ liệu `.flow/`, CLI `flowctl`, chế độ Ralph, và cách dùng trong dự án.

## 1) Mục tiêu của tài liệu

Tài liệu này giúp bạn trả lời nhanh:

- Repo này làm gì, chạy ở đâu, và gồm phần nào?
- `.flow/` lưu gì? Trạng thái runtime lưu ở đâu? Dùng lệnh nào để đọc đúng?
- Quy trình plan → work → review hoạt động ra sao?
- Ralph chạy thế nào, và hook guard chặn lỗi gì?
- Docs/prompt/code có chỗ nào đang lệch nhau?

## 2) Flow-Next là gì (theo repo này)

Flow-Next là một plugin cho Claude Code để điều phối công việc theo hướng “plan trước, làm sau”.

Giá trị cốt lõi:

- Biến yêu cầu thành **epic + các task nhỏ**, có deps rõ.
- Bắt agent **đọc lại spec và repo** trước khi làm (re-anchor).
- Lưu state ra file để sống qua reset/compaction.
- Dùng **review gate** (RepoPrompt hoặc Codex CLI) trước khi “SHIP”.
- Có thể chạy **Ralph**: vòng lặp bash gọi Claude theo lượt, tự chọn việc tiếp theo và tự ép review/receipt.

Repo này chứa cả “prompt layer” (commands/skills/agents) lẫn “tooling layer” (`scripts/flowctl.py`, hook guard, template Ralph).

## 3) Các khối chính trong repo

### 3.1 Prompt layer (Claude Code plugin)

- `commands/flow-next/*.md`: slash commands, mỗi file gọi 1 skill.
- `skills/*/`: workflow prompts (plan, work, review, prime, ralph-init, …).
- `agents/*.md`: subagents (worker, plan-sync, scouts, …).

### 3.2 Tooling layer

- `scripts/flowctl.py`: CLI quản lý `.flow/`, runtime state, checkpoint, review wrappers (rp/codex), và control Ralph.
- `scripts/hooks/ralph-guard.py`: hook enforce luật trong Ralph mode.
- `hooks/hooks.json`: đăng ký hook (PreToolUse/PostToolUse/Stop/SubagentStop).

### 3.3 Ralph scaffold (dùng trong repo dự án)

- `/flow-next:ralph-init` scaffold `scripts/ralph/` vào repo dự án bạn đang làm việc.

## 4) Dữ liệu: `.flow/` (tracked) và state-dir (runtime)

Flow-Next tách “định nghĩa” và “trạng thái runtime” để giảm xung đột git và để share giữa worktrees.

### 4.1 `.flow/` (tracked): kế hoạch + spec

`flowctl init` tạo `.flow/` theo mẫu:

```
.flow/
├── meta.json
├── config.json
├── epics/     # epic definition (JSON)
├── specs/     # epic spec (Markdown)
├── tasks/     # task definition (JSON) + task spec (Markdown)
└── memory/    # ghi nhớ (opt-in)
```

Nguyên tắc:

- File `.md` là nơi viết nội dung: mô tả, tiêu chí đạt, tóm tắt đã làm, bằng chứng.
- File `.json` trong `.flow/epics/` và `.flow/tasks/` là “definition” ổn định: id, title, deps, priority, …

### 4.2 state-dir (không track): status/assignee/evidence/…

Từ v0.18.x, các field runtime được lưu riêng:

- status (`todo/in_progress/blocked/done`)
- assignee/claimed_at/claim_note
- evidence/blocked_reason/updated_at

Flow-Next resolve state-dir theo thứ tự:

1. `FLOW_STATE_DIR` (env override)
2. `git rev-parse --git-common-dir` + `/flow-state` (share giữa worktrees)
3. `.flow/state` (fallback nếu không phải repo git)

Trong state-dir:

```
<state-dir>/
├── tasks/   # *.state.json
└── locks/   # *.lock (fcntl trên Unix)
```

Điểm cần nhớ:

- `flowctl show|tasks|ready|next` đọc “merged view” = definition + runtime state.
- Đừng parse `.flow/tasks/*.json` để lấy status. Status có thể nằm ở state-dir.
- Nếu bạn cần biết state-dir thật sự đang dùng, chạy `flowctl state-path`.

### 4.3 Invariant của task spec

`flowctl validate` yêu cầu mỗi task spec có đúng một lần các heading:

- `## Description`
- `## Acceptance`
- `## Done summary`
- `## Evidence`

`flowctl done` sẽ patch `Done summary` và `Evidence`.

## 5) ID format: `fn-N-xxx` (mới) và legacy

Format hiện tại:

- Epic: `fn-N-xxx` (suffix 3 ký tự `[a-z0-9]`)
- Task: `fn-N-xxx.M`

Legacy vẫn chạy:

- Epic: `fn-N`
- Task: `fn-N.M`

Lý do có suffix: tránh collision khi 2 branch tạo epic cùng số N rồi merge.

## 6) `flowctl` làm gì (và vì sao phải dùng nó)

`scripts/flowctl.py` là đường đi chuẩn cho các thao tác ghi/đọc `.flow/`.

Nguyên tắc thiết kế:

- Agent không nên edit tay `.flow/*` (dễ lệch schema/heading).
- Ghi file theo kiểu atomic (temp + replace).
- Trạng thái runtime tách ra state-dir để giảm conflict merge.
- Có lock theo task để tránh race khi nhiều tiến trình cùng start/done.

### 6.1 Nhóm lệnh quan trọng

Top-level commands (rút gọn theo thứ tự hay dùng):

- `init`, `detect`, `status`
- `epic ...` (create/set-plan/set-plan-review-status/set-branch/close/add-dep/rm-dep)
- `task ...` (create/set-description/set-acceptance/set-spec/reset)
- `show`, `epics`, `tasks`, `ready`, `next`, `start`, `done`, `block`
- `validate`
- `state-path`, `migrate-state`
- `checkpoint ...` (save/restore/delete)
- `review-backend`, `rp ...`, `codex ...`
- `ralph ...` (pause/resume/stop/status)

### 6.2 Task lifecycle (luật chính)

- `flowctl start <task>`: đặt runtime status sang `in_progress`, gắn assignee (từ `FLOW_ACTOR` hoặc git user).
- `flowctl done <task> --summary... --evidence...`: patch spec markdown và ghi runtime status `done` + evidence vào state-dir.
- `flowctl block <task> --reason-file ...`: patch `Done summary` và ghi runtime status `blocked`.
- `flowctl task reset <task> [--cascade]`: reset runtime về `todo` và dọn evidence; lệnh này cũng dọn vài “legacy fields” trong definition để giữ tương thích.

## 7) `flowctl next`: chọn “plan” hay “work”

`flowctl next` là selector mà Ralph dùng để quyết định lượt tiếp theo.

Logic (tóm gọn):

1. Lấy danh sách epics (từ `--epics-file` hoặc scan `.flow/epics/`).
2. Bỏ epic `done`.
3. Bỏ epic bị chặn bởi `depends_on_epics`.
4. Nếu bật `--require-plan-review` và epic `plan_review_status != ship` → trả `status=plan`.
5. Trong epic đó:
   - Nếu có task `in_progress` do đúng actor đang claim → resume.
   - Nếu không, chọn task `todo` mà deps đã done, ưu tiên theo `priority` rồi theo số thứ tự task.
6. Nếu không còn gì → `status=none` (có thể kèm `blocked_epics`).

## 8) Review backends và receipts

Flow-Next chuẩn hóa verdict để máy có thể quyết định:

```
<verdict>SHIP</verdict>
<verdict>NEEDS_WORK</verdict>
<verdict>MAJOR_RETHINK</verdict>
```

Receipt là file JSON lưu kết quả review để Ralph kiểm và để bạn trace.

### 8.1 RepoPrompt (rp)

`flowctl rp setup-review --repo-root ... --summary ...` là đường đi chuẩn:

- Pick đúng RepoPrompt window theo repo root.
- Chạy builder và trả `W=<id> T=<tab>`.
- Ghi `/tmp/.ralph-pick-window-<hash>` để hook guard kiểm tra.

Trong Ralph mode, tuyệt đối tránh `flowctl rp chat-send --json`. Hook guard sẽ chặn vì nó làm mất review text.

### 8.2 Codex CLI (codex)

`flowctl codex impl-review` và `flowctl codex plan-review` bọc `codex exec` để:

- dùng model mặc định `gpt-5.2` (override: `FLOW_CODEX_MODEL`)
- ép `model_reasoning_effort="high"`
- chạy `--sandbox read-only` và `--skip-git-repo-check`
- viết receipt qua `--receipt <path>` (và dùng `session_id` trong receipt để resume)

## 9) Ralph: chạy tự động theo vòng lặp

`/flow-next:ralph-init` scaffold `scripts/ralph/` vào repo dự án.

Các file quan trọng trong scaffold:

- `scripts/ralph/ralph.sh`: vòng lặp chính (select → run Claude → verify receipt → retry/reset/block).
- `scripts/ralph/config.env`: cấu hình (review backend, plan gate, branch mode, limits).
- `scripts/ralph/prompt_plan.md`, `scripts/ralph/prompt_work.md`: contract cho mỗi lượt.
- `scripts/ralph/runs/<run-id>/`: log, receipts, progress, attempts, branches.

Các điểm thiết kế đáng chú ý:

- Ralph chạy Claude ở `--output-format stream-json` để log có cấu trúc.
- Ralph dùng `REVIEW_RECEIPT_PATH` để ép receipt có thật trước khi kết thúc review.
- Ralph có `PAUSE`/`STOP` sentinel files. `flowctl ralph pause|resume|stop` chỉ là wrapper tạo/xóa các file này.

## 10) Hook guard: vì sao Ralph khó “lách”

Hook `scripts/hooks/ralph-guard.py` chỉ chạy khi `FLOW_RALPH=1`.

Nó chặn các lỗi hay gặp làm Ralph “tự tưởng xong”:

- Dùng `--new-chat` khi re-review (mất ngữ cảnh reviewer).
- Gọi `codex exec` trực tiếp (bỏ qua receipt/continuity).
- Dùng `--last` với codex (làm hỏng continuity; Ralph dùng `session_id` trong receipt).
- Chạy `flowctl done` thiếu summary/evidence (trong Ralph mode hook bắt buộc phải có).
- Ghi receipt trước khi review thật sự chạy xong, hoặc ghi impl receipt khi chưa thấy `flowctl done <task>`.

## 11) Skills và subagents: ghép thành workflow

Slash commands chỉ gọi skill tương ứng:

- `/flow-next:plan`, `/flow-next:work`
- `/flow-next:plan-review`, `/flow-next:impl-review`
- `/flow-next:interview`
- `/flow-next:prime`
- `/flow-next:sync`
- `/flow-next:ralph-init`
- `/flow-next:setup`, `/flow-next:uninstall`

Các vai chính:

- Skill `flow-next-plan`: tạo epic + tasks trong `.flow/` (thường có nghiên cứu trước).
- Skill `flow-next-work`: làm task theo `flowctl ready/next`; mỗi task chạy bằng subagent `worker` để giữ context nhỏ.
- Skill review: ép prompt format verdict + receipt; lặp đến khi SHIP.
- Subagent `plan-sync`: cập nhật spec downstream khi implementation lệch plan (thường hạn chế tool).

## 12) Cách dùng trong dự án

### 12.1 Manual (người theo dõi)

1. Chạy `/flow-next:plan ...`
2. (tùy) `/flow-next:plan-review <epic> --review=rp|codex`
3. Chạy `/flow-next:work <epic>`
4. (tùy) `/flow-next:impl-review ...` để review theo branch

### 12.2 Team + worktrees

- State runtime nằm ở git common-dir nên share giữa worktrees theo mặc định.
- Dùng `flowctl start` để claim task, `--force` để takeover khi cần.
- CI gate hợp lý: `flowctl validate --all`.

### 12.3 Ralph (chạy tự động)

1. Chạy `/flow-next:ralph-init` trong repo dự án.
2. Sửa `scripts/ralph/config.env`.
3. Chạy `scripts/ralph/ralph_once.sh` để quan sát 1 lượt.
4. Chạy `scripts/ralph/ralph.sh` để chạy lâu.

## 13) Drift và pitfall đã xác nhận

Các điểm này dễ làm bạn mất thời gian vì “tài liệu nói một đằng, code làm một nẻo”.

### 13.1 Memory “auto-capture”

Một số mô tả nói Ralph sẽ tự ghi learnings vào `.flow/memory/…` khi review trả NEEDS_WORK.

Hook guard hiện chỉ nhắc bạn chạy `flowctl memory add ...`. Nó không tự ghi memory file.

### 13.2 Ví dụ `jq` sai schema

Trong `skills/flow-next-work/phases.md` có ví dụ parse `flowctl tasks --json` không đúng.

`flowctl tasks --json` trả object dạng `{"success":..., "tasks":[...], "count":...}` nên `jq` phải đọc `.tasks`.

### 13.3 Progress UI của Ralph đọc sai nguồn status

Template `scripts/ralph/ralph.sh` có đoạn tính tiến độ bằng cách đọc `.flow/tasks/*.json`.

Nhưng `flowctl start/done/block` ghi status vào state-dir, không cập nhật status trong definition JSON.

Kết quả: progress có thể sai dù `flowctl show --json` vẫn đúng.

## 14) Test scripts (repo plugin)

Các script này cho biết “cái gì quan trọng” trong repo:

- `scripts/ci_test.sh`: check toàn diện (id format, state-dir, checkpoint, ralph control, prompt invariants).
- `scripts/smoke_test.sh`: smoke cho flowctl core.
- `scripts/ralph_smoke_test.sh`: smoke Ralph với stub Claude.
- `scripts/ralph_e2e_test.sh`: e2e với Claude CLI thật.

## 15) Tài liệu nên đọc kèm (trong repo)

- `README.md`
- `docs/flowctl.md`
- `docs/ralph.md`
- `scripts/flowctl.py`
- `scripts/hooks/ralph-guard.py`
- `skills/flow-next-ralph-init/templates/ralph.sh`

## 16) Ấn tượng nhất (3–5 điểm)

1. Repo tách definition và runtime state theo kiểu worktree-aware (`git-common-dir/flow-state`) để giảm xung đột merge và vẫn share trạng thái.
2. Ralph dùng receipt + hook guard để biến review thành một gate có thể tự động hóa, thay vì “review cho vui”.
3. `flowctl next` ưu tiên resume task đúng assignee trước, giúp chạy dài không bị nhảy task.
4. Worker subagent theo từng task giúp giữ context nhỏ và giảm drift giữa các task.
5. Checkpoint giúp bạn khôi phục epic/task khi review loop bị compaction hoặc spec bị lệch.

## 17) Tóm tắt

Flow-Next là một plugin cho Claude Code để biến yêu cầu thành epic/tasks trong `.flow/`, rồi thực thi từng task theo một vòng lặp có kiểm soát. `flowctl` là trung tâm: nó quản lý file layout, merge runtime state từ state-dir, chọn task tiếp theo (`next`), và bọc review backends (RepoPrompt/Codex) theo một giao thức verdict thống nhất.

Ralph là chế độ tự chạy: nó gọi Claude theo lượt, dựa vào `flowctl next`, và ép mọi review phải có receipt. Hook guard làm Ralph “khó sai” bằng cách chặn các thao tác phá continuity hoặc làm mất review text. Tài liệu cũng chỉ ra vài điểm drift giữa docs/prompt/code để bạn tránh debug vòng lặp vô ích.
