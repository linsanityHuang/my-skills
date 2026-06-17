---
name: openstack-bug-scanner
description: Parallel bug scan of an OpenStack project (nova, cinder, glance, neutron, keystone, heat, swift, ...) for simple, easily-verifiable bugs only. Use this skill when the user names an area of any OpenStack project and asks to scan/find/hunt bugs — e.g. "在 cinder/volume/manager.py 找 unchecked None", "hunt bugs in neutron/agent/linux/", "扫一遍 glance/api/v2/images.py", "find division-by-zero in nova/scheduler/". Auto-detects the project from .gitreview/setup.cfg, loads a project adapter for project-specific idioms (i18n path, config/policy locations, eventlet use), fans out 1-5 parallel agents that write findings to disk as they go (resumable), then verifies each finding against the current source tree, and produces a final report of HIGH/MEDIUM findings with file:line + suggested fix. Does NOT modify any source files, write tests, add release notes, commit, or push. If the user also wants the bug fixed afterwards, hand the report to `openstack-bug-fixer`. DO NOT use for non-OpenStack code, for general refactors, for code-explanation requests, for race conditions, or for security audits.
---

# openstack-bug-scanner

Parallel simple-bug scan for **any OpenStack project** under
`review.opendev.org` (Nova, Cinder, Glance, Neutron, Keystone, Heat,
Swift, ...). The user nominates a **scope** (file or directory) and
optionally a **bug class** (unchecked `d[key]`, unchecked `None`,
division by zero, mutable default, broad `except`, etc.). The skill
produces a **findings report** — it does **not** modify source code,
write tests, add release notes, commit, or push.

If the user then wants the bug fixed, hand the report to
`openstack-bug-fixer`.

## Language — 默认中文

**Default to Chinese (简体中文) for all user-facing communication
when running this skill.** This includes:

- 状态行（"Detected project: openstack/cinder ..."、"Phase 2 完成,
  3 个 agent 已 fan out"）
- 各阶段之间的衔接说明
- 最终 report 的用户摘要
- 与用户的对话回复

例外（保持英文）：

- **代码、文件路径、命令输出、工具错误信息、commit message** 等
  落 repo 的内容一律英文。
- **逐字引用项目约定**（HACKING.rst、commit-messages.rst 等）
  原文是英文就保持英文。
- **没有合适中文译名的技术术语**（Gerrit、Change-Id、patch set、
  `dst_numa_info` 之类的属性名等）—— 保持英文。

如果用户明确要求英文（"用英文"、"reply in English"、"speak English"），
在该对话中切到英文。否则中文是默认。

## When to trigger

Trigger when the user:

- Names a specific area of an OpenStack project and asks for a bug
  scan ("scan nova/api/openstack/compute/ for unchecked `d[key]`",
  "在 cinder/volume/manager.py 找未检查 None 的 bug",
  "hunt bugs in neutron/agent/linux/", "扫 glance/api/v2/").
- Asks to **find / hunt / scan** for a particular bug class
  ("在 nova/ 找除零错误", "find unchecked `None` in cinder/objects/").
- Says "扫一遍 <project>/<dir>" or any of:
  - 找 bug / 扫 bug / 看有没有 bug
  - find / hunt / scan for bugs in <project>/...

## When NOT to trigger

- **The user already has a specific bug and wants it fixed** —
  use `openstack-bug-fixer` instead.
- **Repo is not an OpenStack project** (or doesn't use Gerrit) —
  wrong workflow.
- General refactor, design review, or "explain this code" requests.
- Security audits — use a dedicated security review skill.
- Race conditions, threading, complex architectural issues — the
  scope here is *simple* bugs only (single-file, < ~20 line patches,
  objectively verifiable).

## Inputs the user must provide

1. **Scope** — path under the detected `<PACKAGE>/` (e.g.
   `nova/compute/manager.py`, `cinder/volume/`, `neutron/agent/linux/`,
   `glance/api/v2/images.py`). Accept either:
   - A path relative to the repo root: `cinder/volume/manager.py`.
   - A path relative to `<PACKAGE>/`: `volume/manager.py` (the skill
     resolves it to `<repo>/<PACKAGE>/volume/manager.py`).
2. **Bug class** — optional narrowing. Default: all 8 classes
   (see `_shared/openstack-bug/references/bug-classes.md`).
3. **Severity bar** — optional. Default: MEDIUM and above. The
   skill will discard LOW-only stylistic findings.

## Output

A **findings report** (markdown), one per run, written to:

```
.tmp/bug-scan/<YYYY-MM-DD>-<short-tag>/
├── progress.md        # high-level tracker
├── A1-<sub-scope>.md  # per-agent findings (one file each)
├── A2-…
└── final-report.md    # synthesized, verified, severity-sorted
```

The `final-report.md` is the user-facing deliverable. It contains,
per finding, the verified `file:line`, the trigger, a suggested
fix, and a `VERIFIED` / `DOWNGRADED` / `REJECTED` status with a
one-line reason. **No source files are touched.**

`<short-tag>` includes the project prefix so concurrent scans in
different repos don't collide — e.g.
`cin-12-dict-access-cinder-volume` (cinder),
`nov-12-dict-access-nova-api` (nova),
`neu-03-zero-div-neutron-agent`.

## Pipeline

Run these 4 phases in order. **Each phase's output is written to
disk so the run is resumable** if interrupted (a `kill` between
phases should not lose any work).

### Phase 1 — Detect project, resolve scope

1. **Run project detection** (see
   `_shared/openstack-bug/references/project-detection.md`):
   - Read `.gitreview` → `PROJECT_NAME`, `GERRIT_PROJECT`,
     `GERRIT_HOST`, `GERRIT_PORT`, `DEFAULT_BRANCH`.
   - Read `setup.cfg` / `setup.py` → `PACKAGE`.
   - Infer `CONFIG_PATH`, `POLICY_PATH`, `TEST_ROOT`,
     `RELEASE_NOTE_DIR`.
   - Load the adapter at
     `_shared/openstack-bug/references/project-adapters/<PROJECT_NAME>.md`,
     or fall back to `default.md`.
   - Print one line to the user:
     > "Detected project: openstack/nova (package=nova). Loading
     > adapter `nova.md`."

2. **Resolve the scope** to an absolute path:
   - If the user gave a path that starts with `<PACKAGE>/`,
     resolve it directly.
   - If the user gave a path that doesn't start with `<PACKAGE>/`,
     try `<repo>/<PACKAGE>/<scope>` first; if that doesn't exist,
     try `<repo>/<scope>` directly (so a stray "volume/manager.py"
     gets resolved correctly).
   - If the user named a file, scope = that file. If a directory,
     scope = the directory.
   - The rest of the repo is **out of scope** for this run (it can
     be a separate invocation).

3. **Create the run workspace**:
   ```
   .tmp/bug-scan/<YYYY-MM-DD>-<short-tag>/
   ├── progress.md
   ├── A1-<sub-scope>.md
   ├── A2-…
   └── final-report.md    (written in Phase 4)
   ```
   `<short-tag>` = `<PROJECT_NAME>-prefix><NN>-<bug-class>-<scope-slug>`
   (e.g. `cin-12-dict-access-cinder-volume`).

4. **Write the scan plan** to `progress.md`:
   - detected project + adapter loaded
   - bug class list and severity bar
   - one row per parallel agent: scope, agent prompt summary
   - resume protocol: how to re-launch only incomplete agents

### Phase 2 — Fan out (parallel)

- Decide how many agents to launch. Rules of thumb:
  - Single file → 1 agent, scope = that file
  - 2–5 files in one directory → 1 agent
  - 6+ files or a top-level subdirectory → 2–5 agents split by
    sub-directory or by file-group
- Spawn the agents in **one message** (parallel). Each agent gets:
  - Absolute scope
  - `PROJECT_CONTEXT` (PACKAGE, ADAPTER_PATH, TEST_ROOT,
    RELEASE_NOTE_DIR, …)
  - Bug class list and severity bar
  - Output file: `.tmp/bug-scan/<run>/A<N>-<sub-scope>.md`
  - Hard cap on findings (default 25; quality > quantity)
  - Instruction: "**Write each finding to the output file as soon as
    you confirm it. Do not buffer findings to the end.**" — this is
    the key resilience rule; if the agent is killed mid-scan,
    partial findings are already on disk.
- The agent prompt template is at
  `_shared/openstack-bug/agents/scanner.md` — Read it, fill in
  `<REPO_ROOT>`, `<SCOPE>`, `<OUTPUT_FILE>`, `<MAX_FINDINGS>`,
  `<PROJECT_CONTEXT_JSON>`.
- Wait for all to complete or for the user to interrupt.

### Phase 3 — Verify

For each finding in each `A*.md` file:

- **Re-read** the cited `file:line` in the working tree. Confirm the
  code still looks like the snippet. If the code has been edited
  since the agent saw it, drop or re-evaluate the finding.
- **Reproduce the trigger** in your head: what input or state would
  make the bug fire? If the trigger is theoretical-only (would
  require the user to do something very weird), downgrade to LOW.
- **Check test coverage** for the file/line (look in
  `<PROJECT_CONTEXT.TEST_ROOT>`). If the existing test suite would
  have caught the bug, downgrade the finding ("known and tested") —
  do not re-fix.
- **Apply project-specific anti-patterns** from the loaded adapter
  to the suggested fix — e.g. Nova adapter forbids `from nova.privsep
  import path`; Cinder adapter forbids direct DB access from
  `cinder/volume/drivers/`. A suggested fix that violates an
  adapter rule must be rewritten before it goes in the report.
- Mark each finding in the agent's file as `VERIFIED`, `DOWNGRADED`,
  or `REJECTED` with a one-line reason.

Discard LOW findings unless the user explicitly asked for them.

### Phase 4 — Final report

Synthesize the verified findings into `final-report.md`, sorted by
severity (HIGH first) and then by file path. Each entry:

```
### FINDING <N> — <severity> — <class>
- **File:** <abs path>:<line>
- **Status:** VERIFIED | DOWNGRADED | REJECTED
- **Trigger:** <1-2 sentence reproduction recipe>
- **Suggested fix:** <minimal patch sketch — do NOT apply it;
  must respect the loaded project adapter's anti-patterns>
- **Next:** hand to `openstack-bug-fixer` with the finding ID.
```

Update `progress.md` to mark the run as DONE. Tell the user:

> Scan complete. N findings (H HIGH, M MEDIUM). Top finding:
> `<file:line>`. Run `openstack-bug-fixer` to apply the fix, or
> pick a different finding ID.

## Resume protocol

If a previous run is found in `.tmp/bug-scan/`:

1. Read its `progress.md`. Note which agents finished
   (`Status: done` in the table) and which are missing
   (`Status: pending` or `Status: running`).
2. Read the existing `A*.md` files — those are the findings already
   in hand.
3. Re-launch **only the missing agents**, passing the same prompt
   and explicitly telling them to dedup against the existing
   `A*.md` files.
4. Continue from the appropriate phase.

If `progress.md` exists but no `A*.md` files, treat all agents as
pending and re-launch them all.

## Bundled resources

- `_shared/openstack-bug/agents/scanner.md` — the prompt template
  for parallel scan agents. Read in Phase 2.
- `_shared/openstack-bug/references/bug-classes.md` — the canonical
  8 bug classes and severity bar. Read in Phase 1.
- `_shared/openstack-bug/references/openstack-conventions.md` —
  common (project-agnostic) HACKING.rst rules. Loaded in Phase 1
  via the adapter chain.
- `_shared/openstack-bug/references/project-detection.md` — the
  project detection steps; run once at the top of Phase 1.
- `_shared/openstack-bug/references/project-adapters/<project>.md`
  — project-specific HACKING rules, code patterns, and
  anti-patterns. Auto-loaded based on `.gitreview`.

## Anti-patterns (do NOT do these)

- ❌ **Modifying any source file** — that's `openstack-bug-fixer`'s
  job.
- ❌ **Writing tests, release notes, or running `git add`** —
  not in this skill.
- ❌ Skipping the verification phase and trusting the agent's claim.
- ❌ Padding the report with LOW-only or stylistic findings to
  look thorough.
- ❌ Triggering on "fix this bug" — that's `openstack-bug-fixer`.
- ❌ **Skipping project detection** — without `PROJECT_CONTEXT`,
  the suggested fix may violate the loaded project adapter's
  anti-patterns (e.g. importing `nova.db` from `nova/virt/*`,
  using `neutron.i18n` instead of `neutron._i18n`).
- ❌ **Hardcoding project-specific paths in the orchestrator** —
  the agent prompt template must not name `nova/`, `cinder/`,
  etc. directly. Project-specific examples belong in the project
  adapter, not in the shared template.
