---
name: openstack-bug-fixer
description: "Apply a minimal source fix for an already-identified OpenStack (any project) bug, with regression test, release note, and a Gerrit-style local commit. Works for any OpenStack project under review.opendev.org (Nova, Cinder, Glance, Neutron, Keystone, Heat, Swift, ...); auto-detects the project from .gitreview and loads a project adapter for project-specific HACKING rules, config/policy paths, and idioms. Stage by default; commit on user request; push to Gerrit via `git review` only when the user explicitly says so (\"提交\", \"commit and push\", \"git review\", \"push to Gerrit\"). Validates preflight: `git-review` installed, `gerrit` remote URL matches `.gitreview`, working tree on a feature branch (not master), local master not diverged from Gerrit master. Runs a pre-submission check (subject ≤ 50 chars, Change-Id + Signed-off-by well-formed, body 72-col wrapped, no stale release notes from prior patch sets, unit tests + pre-commit clean) before pushing. Supports the re-submit loop: refined work for the same bug goes back through Phase 2/3/4 with `git commit --amend` (keep the same `Change-Id`) and `git review -y`, which lands as a new patch set on the existing Gerrit change. Input: a verified finding (file:line + description) — named explicitly by the user, or the top entry of a `*-bug-scanner` report. Output: source patch + unit test in the matching tests/unit/... file + release note under releasenotes/notes/ + staged + (optionally) committed + (optionally) pushed to a new or existing Gerrit change. Does NOT scan for new bugs (use the project's bug-scanner); does NOT push without explicit user consent; does NOT touch unrelated code. DO NOT use for non-OpenStack code, for repos that don't use Gerrit, for general refactors, for code-explanation requests, or for race conditions / security audits."
---

# openstack-bug-fixer

Apply a minimal fix for an **already-identified** OpenStack bug,
across any project (Nova, Cinder, Glance, Neutron, Keystone, Heat,
Swift, ...), with regression test, release note, and a Gerrit-style
**staged** local commit.

The user provides the finding (or a scanner report containing it);
the skill trusts the verification that already happened and proceeds
to fix.

For Nova specifically: project-specific HACKING rules (H307, H342,
H350, H362, H369, H373, `nova.utils.*` idioms, eventlet) are
auto-loaded from
`_shared/openstack-bug/references/project-adapters/nova.md` based on
`.gitreview` (`openstack/nova.git`). This skill handles Nova directly
— no wrapper skill is needed.

## Language — 默认中文

**Default to Chinese (简体中文) for all user-facing communication
when running this skill.** This includes:

- 状态行（"Detected project: ..."、"Phase 2 完成，5 个测试通过"）
- 各阶段之间的衔接说明（"现在进入 Phase 3，跑 pre-commit"）
- Pre-submission 检查结果摘要（"Subject 46 chars (≤ 50 ✓)..."）
- 推送完成消息（"Change ... 已打开，CI 可以开始跑"）
- 推送前的最终检查清单输出
- 最终的总结 / 交付说明

例外（保持英文）：

- **代码、文件路径、commit message、release note、命令输出、
  工具错误信息、日志字符串** —— 凡是要落到 repo 里或由工具渲染的，
  一律英文。
- **逐字引用项目约定**（HACKING.rst、
  doc/source/contributor/commit-messages.rst 等）—— 原文是英文就
  保持英文。
- **没有合适中文译名的技术术语**（Gerrit、Change-Id、patch set、
  staging area、`dst_numa_info` 之类的属性名等）—— 保持英文。

如果用户明确要求英文（"用英文"、"reply in English"、"speak English"），
在该对话中切到英文。否则中文是默认。

## When to trigger

Trigger when the user:

- Names a specific bug and asks to fix it
  ("修 BUG-001", "fix this", "fix A4-001",
  "在 nova/compute/monitors/cpu/virt_driver.py 修除零",
  "把那个 .find() 没检查 None 的 bug 修掉").
- Hands over a scanner report and says "fix the top finding" or
  "fix FINDING-3".
- Says "commit the fix" / "提交上次的修复" / "stage it" — usually
  referring to an earlier scanner+fix conversation.
- Asks to push to Gerrit ("git review push 代码", "推上去",
  "commit and push", "open the change"). The skill will run the
  full fix → test → pre-commit → release-note → `git review`
  pipeline, including the preflight checks.
- Asks to apply review feedback to an existing change
  ("按 sean mooney 的意见改", "address the second review
  comment", "再修一下另外两个 issue", "把旧的 release note
  删掉", "amend 到当前 commit"). This is the **re-submit
  loop** (Phase 4e) — refine the source/test/note on the
  existing branch, `git rm` any stale artifacts from the
  prior patch set, `git commit --amend` (keep `Change-Id`),
  then `git review -y` for a new patch set on the same
  Gerrit change.

## When NOT to trigger

- **The user wants to find new bugs** — use the project's
  bug-scanner first. `openstack-bug-fixer` trusts the input
  finding is already verified; it does not re-scan.
- **Repo is not an OpenStack project** (or doesn't use Gerrit) —
  wrong workflow. Use a different fixer skill.
- General refactor, design review, or "explain this code" requests.
- Security audits — use a dedicated security review skill.
- Race conditions, threading, complex architectural issues — out
  of scope. The scope here is *simple* bugs only (single-file,
  < ~20 line patches, objectively verifiable).

## Inputs the user must provide

1. **Finding** — at minimum a `file:line` and a one-line
   description. Either named explicitly
   ("BUG-001", "A4-001", "FINDING-3") or implicit (a
   `final-report.md` produced by a `*-bug-scanner`).
2. **Optional: scanner report path** — if the user just got a
   report, default to "fix the top finding" (or whatever the user
   specified).
3. **Optional: scope** — usually inferred from the finding's
   file:line. The skill reads the surrounding code, not the rest
   of the project tree.

## Output

A clean working tree, plus (depending on user intent):

- The **source file** patched in place (minimal fix).
- A **new test** in the matching `tests/unit/.../test_*.py`,
  covering the exact trigger.
- A **release note** under `releasenotes/notes/` with the
  `fix-<short-name>-<12-char-hex>.yaml` naming convention.
- `git add` of the three files (source + test + release note)
  — always.
- If user asked to commit: a local commit in Gerrit style
  (subject ≤ 50 chars, body ≤ 72 cols, `Change-Id:` placeholder,
  `Signed-off-by:` from git config). A first push to a new
  Gerrit change is **PS1**; a refined push for the same bug
  (after review feedback) is **PS2/PSn** of the same change —
  re-push via `git commit --amend` + `git review -y`, see
  Phase 4e.
- If user asked to push: a new Gerrit change (URL surfaced in
  the response).

The skill never auto-commits or auto-pushes. The user's wording
("修" / "提交" / "git review" / etc.) decides what runs.

## Pipeline

Run these 4 phases in order. **Each phase's output is written
to disk** so a kill between phases loses no work.

A single bug fix usually goes through this pipeline **more than
once**: the first iteration lands a change in Gerrit as PS1,
then the reviewer asks for adjustments, and the skill loops
back to Phase 2 to refine the source/test/note and amend the
commit. The amended commit goes to Gerrit as PS2 of the same
change (same `Change-Id`). See **Phase 4e — Re-submitting
refined work for the same bug** below.

### Phase 1 — Detect project, read the finding, design the fix

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

2. **Identify the input finding.** If the user named it
   ("BUG-001", "A4-001", "FINDING-3"), find it in the
   project's `final-report.md` (e.g. `.tmp/bug-scan/*/A*.md`
   for nova; for cinder, a similar `cinder-bug-scanner`
   layout). If the user just said "fix the top finding", open
   the most recent scanner report and take the first finding.

3. **Re-read** the cited `file:line` in the working tree
   (the source may have changed since the scanner ran). Resolve
   the path against the detected `TEST_ROOT` / `RELEASE_NOTE_DIR`.

4. **Reproduce the trigger** in your head (or, for non-trivial
   triggers, in a scratch test script under `.tmp/`). The fix
   must address the actual trigger, not a related-looking one.

5. **Design the minimal fix**:
   - Match the surrounding code's comment density, naming, and
     idiom.
   - If a sibling site already uses a particular guard pattern
     (e.g. `.get('key', default)` two lines up), use the same
     one.
   - Do not reformat unrelated code.

6. **Check style rules** in
   `_shared/openstack-bug/references/openstack-conventions.md`
   for project-agnostic HACKING rules, and in the project
   adapter for project-specific ones (config / policy paths,
   privsep, locks, etc.). The minimum HACKING rules to consult
   per project are listed in the adapter's "HACKING rules"
   section.

### Phase 1.5 — Switch to the default branch, create a fresh branch

**Always, even when the user did not ask to push.** This rule
applies to every fix, because:

- Each bug fix is its own Gerrit change → needs its own branch.
- Prior bug-fix commits accumulated on master locally; master
  is the integration tip. Branching from any other branch
  (e.g. the previous bug's `bug-001/...` branch) makes the new
  commit carry the previous fix's history.
- It makes `git review -y` deterministic: the push target is
  the same `refs/for/<defaultbranch>`, regardless of which
  branch you started on.

```bash
git checkout <defaultbranch>          # from PROJECT_CONTEXT
git checkout -b bug-<NNN>/<short-slug>
```

Branch-name convention (user's rule, applies across all
projects):

- `bug-001/zero-cputime-delta` — example for a Nova CPU monitor fix.
- `bug-<NNN>/<short-slug>` — `<short-slug>` is ≤ 30 chars,
  kebab-case, summarizes the trigger
  (e.g. `nvdimm-missing-path`, `flavor-extra-specs-reinsert`,
  `aggregate-add-host-empty-nodelist`).

Edge cases:

- If the user is **already** on a fresh branch for this bug
  (created earlier in the conversation), skip the
  re-creation. Don't `git checkout master` mid-fix.
- If the user explicitly says "use branch X" or "stay on
  master", respect that — the rule is the default, not
  absolute.
- If `git checkout <defaultbranch>` fails (local default-branch
  is behind upstream and the user hasn't asked to rebase),
  surface the divergence and ask before proceeding.

**Multi-bug in the same source file.** When the user asks to fix
multiple bugs in the same session and both bugs touch the same
source file (e.g. two unrelated functions in
`nova/virt/libvirt/migration.py`):

1. Each bug gets its own branch and its own commit (per the
   one-branch-per-fix rule).
2. The git index is per-repository, not per-branch. Once you
   `git add` a file on branch A, switching to branch B
   carries the staged content into B's working tree, where
   it may collide with B's HEAD.
3. The clean workflow is **commit each branch in sequence**,
   not stage-and-switch:
   - On bug-A branch: apply source/test/note → `git add` →
     `git commit` (with proper Gerrit-style message).
   - `git checkout bug-B` (now branch A is durable on its own
     branch; the index is empty again).
   - On bug-B branch: apply source/test/note → `git add` →
     `git commit`.
4. If you only stage (don't commit) both branches, the second
   `git add` after switching will overwrite the first
   branch's staging with the second branch's file content —
   the first branch silently loses its index. Always commit
   before switching if you want both staged AND ready.

### Phase 2 — Apply the fix, add the test

- **Source patch** — apply the minimal change to the working
  tree. Use Edit, not Write (preserve everything else).
- **Regression test** — add a test in the corresponding
  `tests/unit/.../test_*.py` next to the existing tests for
  the module. The test should drive the **exact trigger**
  from Phase 1, not a contrived one. If the trigger is hard to
  set up in a unit test (e.g. needs a real hypervisor / real
  storage backend), explain that and add a test that covers
  as much as is practical in-process.
- **Run the test** with the project's standard runner:
  ```
  # standard (OpenStack convention)
  .tox/py3/bin/python -m unittest <dotted.test.path> -v

  # or via tox
  tox -e py3 -- <dotted.test.path>
  ```
  If the project uses eventlet (check the adapter), also set
  `OS_<PACKAGE_UPPER>_DISABLE_EVENTLET_PATCHING=False`
  (most projects default this; only override if a test
  specifically requires the native-threading mode).
  **All tests in the file must pass** — not just the new
  one. The fix must not break existing coverage.

### Phase 3 — Pre-commit, release note

- **Pre-commit** on the changed files (the actual gate
  OpenStack uses):
  ```
  .tox/pep8/bin/pre-commit run --files <changed files…>
  ```
  All hooks must pass. If autopep8 re-formats, run it, then
  re-run the unit tests.
- **Release note** under the project's `releasenotes/notes/`
  (resolved from `RELEASE_NOTE_DIR` in `PROJECT_CONTEXT`),
  with a random 12-char hex suffix in the filename (mirror
  the existing convention in that dir):
  ```yaml
  ---
  fixes:
    - |
      [Plain-English description, 1-3 sentences.
      Mention the observable user impact.]
  ```
- **Do NOT add a Python test** that imports the release
  note (release notes are tested separately by
  `tox -e releasenotes`).

### Phase 4 — Stage, commit, push (last step only on user request)

The user's wording controls what this phase does:

| User said | Stage? | Commit? | Push? |
|---|---|---|---|
| "修 BUG-NNN" / "fix this" (no commit verb) | ✅ | ❌ | ❌ |
| "提交" / "commit it" / "stage it" | ✅ | ✅ | ❌ |
| "git review" / "push to Gerrit" / "提交并推" / "commit and push" | ✅ | ✅ | ✅ |

Default to **stage only**; commit/push are opt-in. Per the
project's CLAUDE.md / AGENTS.md
("Do not run `git add` / `commit` / `push` … unless the user
explicitly asks"), the skill must NOT commit or push without
an explicit verb in the request.

**4a — Always: stage the three files**

- `git add <source> <test> <release note>`.
- Never stage the untracked session artifacts
  (`.ai-auto-collection.state.json`, `.codegraph/`, `.mcp.json`,
  anything under `.tmp/`) — these are session bookkeeping, not
  part of the fix.

**4b — If committing: write the commit message first, then commit**

- Gerrit-style message per the project's
  `doc/source/contributor/commit-messages.rst` (and the
  generic rules in
  `_shared/openstack-bug/references/openstack-conventions.md`):
  ```
  <scope>: <one-line summary under 72 chars>

  <optional body — wrap at 72 cols>

  Change-Id: I<placeholder>
  Signed-off-by: <name> <email>
  ```
  Body should be ≤ 50 chars in the subject (Gerrit emits a
  `warning: subject >50 characters; use shorter first paragraph`
  otherwise — the push still succeeds, but the warning is noise
  in the change review and easy to silence by amending the
  subject; see the `-s`-on-amend note below). The 72-char hard
  limit is for the body and any non-subject lines. Be concrete
  about the trigger + the fix in the subject (a vague subject
  gets reviewers to skip the change).
- `git commit -s -m "<subject>" -m "<body>" -m "Change-Id: I…"` —
  the `-s` adds `Signed-off-by` from `git config user.name` /
  `user.email`. The commit-msg hook (installed by `git-review`)
  will auto-generate the `Change-Id` if the commit message
  doesn't already have one. Do **not** handcraft a `Change-Id`
  placeholder; just omit it from the message and let the hook
  add it.
- **If you used `-c user.name=… -c user.email=…` on the
  original commit** to override a wrong default git config,
  you're propagating a bad value — fix the underlying
  config first:
  ```
  git config user.name "<name>"
  git config user.email "<addr>"
  ```
  Then on amend, pass `--reset-author` so author and
  committer both come from git config (not from the
  previous commit's stale `-c` override):
  ```
  git commit --amend --reset-author -s -F <message-file>
  ```
  `--reset-author` rewrites `Author:` and `Committer:` to
  match `git config user.{name,email}`; the `-s` then
  regenerates `Signed-off-by:` from the same config. Do
  **not** keep using `-c user.email=…` on the amend —
  the inline override hides a wrong git config and the
  fix only lives in the commit message, not in your
  environment.
- **If amending** (e.g. to shorten a >50-char subject, to apply
  review feedback as a refined source change, or to fix any
  commit-message issue): always pass `-s` to `--amend` too.
  The commit-msg hook only auto-adds `Change-Id`; it does NOT
  re-add `Signed-off-by` if the new message omits it. Gerrit
  will reject the push with `not Signed-off-by author/committer/
  uploader` and you have to amend again. So:
  `git commit --amend -s -m "<new subject>" -m "<body>"`. Keep
  the **same** `Change-Id` so Gerrit updates the existing
  change as a new patch set (PS2) rather than creating a new
  change. For the full review-feedback re-submit loop (source
  + test + release note refinements, stale-note cleanup, etc.),
  see **Phase 4e**.
- **Use `-F <message-file>` (not `-m`) when amending.** The
  commit-msg hook treats `-m "…"` as a fresh message and
  regenerates `Change-Id` even if the previous commit already
  had one. To preserve the original `Change-Id`, write the
  full new message (including the existing `Change-Id:` and
  `Signed-off-by:` trailer) to a file and pass `-F <file>`:
  ```
  cat > /tmp/msg.txt << 'EOF'
  <new subject>

  <new body>

  Change-Id: I<original 40-char hex>
  Signed-off-by: <name> <email>
  EOF
  git -c user.name="…" -c user.email="…" commit --amend -s -F /tmp/msg.txt
  ```
  The `-s` is still required (it adds nothing if `Signed-off-by`
  is already present, but it's defensive).
- After any amend or new commit, run
  `git log -1 --format=fuller` and confirm both `Change-Id:` and
  `Signed-off-by:` are present and match what you expect
  **before** `git review -y`. Specifically verify:
  - `Change-Id:` value is the same as before the amend
    (compare `git log -1 --format='%B'` before and after).
  - `Signed-off-by:` email matches the user's preferred
    address, not the default git config.
  - Subject length is ≤ 50 chars (Gerrit lint warning threshold).
- Never include `Co-Authored-By` unless the user asks (Gerrit
  rejects commits that contain it).
- Show the user the final commit log (`git log -1 --format=fuller`)
  before proceeding to 4c.

**4c — If pushing: Gerrit preflight + `git review`**

OpenStack projects use Gerrit. Pushing is **not** a raw
`git push` — it's `git review` (which under the hood does
`git push <remote> HEAD:refs/for/<defaultbranch>`). Skipping
the preflight below leads to silent misroutes (wrong project),
mysterious `Missing tree` errors (default-branch divergence),
or accidentally pushing default-branch tip to
`refs/for/<defaultbranch>`.

Preflight (fail fast, before any remote call):

1. **`git-review` installed?** `which git-review` → exit 0?
   - If no: ask the user before installing (project conventions
     typically forbid unprompted `pip install`). Suggested
     install:
     `pip install --user --break-system-packages git-review`
     (PEP 668 override needed on externally-managed systems).
   - On a successful install,
     `export PATH=$HOME/.local/bin:$PATH` so the user-shell
     can also call it.

2. **Remote configured for the right project?** Read `.gitreview`
   (`project=…`) and compare to the `gerrit` remote URL.
   - If `gerrit` is missing or points to a different project
     (e.g. `.gitreview` says `openstack/nova.git` but the
     remote says `openstack/cinder.git`): stop and ask before
     fixing. `git remote set-url gerrit <correct-url>` is a
     one-line fix but the user must consent.
   - If the project is not on `review.opendev.org` (e.g.
     `host=...` is something else), this skill is out of scope.

3. **On a feature branch, not the default branch?**
   `git branch --show-current`
   - If `<defaultbranch>` (or `main`, `master`): create a
     feature branch from the current commit.
     `git checkout -b bug-<NNN>/<short-slug>` (or whatever
     slug the user prefers — ask if unclear).
   - **Why**: `git review` from `<defaultbranch>` pushes
     `<defaultbranch>` tip to `refs/for/<defaultbranch>`, which
     can carry unrelated commits from local-only history.
     Gerrit also rejects (or warns) when the pushed commit's
     parent tree doesn't match Gerrit's current
     `refs/heads/<defaultbranch>`.

4. **Local default branch not diverged from Gerrit default
   branch?** `git fetch gerrit <defaultbranch>` then
   `git rev-list --left-right --count gerrit/<defaultbranch>...<defaultbranch>`.
   - If local is **behind** (Gerrit has commits you don't):
     warn the user, suggest `git rebase gerrit/<defaultbranch>`
     or merging, and stop. Pushing a stale parent to
     `refs/for/<defaultbranch>` gives the "Missing tree …"
     unpack error.
   - If local is **ahead** with **unrelated** commits (a true
     fork), warn and ask before pushing.

5. **SSH auth works?** `ssh -T -o ConnectTimeout=10
   <user>@<host> -p <port> echo test` — Gerrit's reply will be
   `fatal: Gerrit Code Review: echo: not found`, which is the
   expected "auth OK" signal.
   - **Disambiguation.** A bare
     `Disconnected from … port <port>:2: Protocol error or
     corrupt packet` (right after the SSH `KEXINIT` exchange,
     before authentication completes) is **not** a credential
     problem. It's a server-side or network issue:
     - Gerrit's Apache MINA SSHD is sometimes flaky — the
       same simple interactive `ssh -p <port> user@host` can
       succeed a moment later. Retry `git review -y` once or
       twice before concluding the connection is broken.
     - If the failure is persistent from the user's normal
       shell, it's a WSL/firewall/proxy/MITM issue on the
       client side — surface to the user; don't try to fix
       their environment (no `~/.ssh/config` rewrites, no
       copying keys across environments) without consent.
     - If the failure is persistent from a sandbox shell but
       the user confirms the same command works in their
       normal shell, it's a sandbox networking issue; have
       the user run the push from their own shell.

Then run the push:

```
export PATH=$HOME/.local/bin:$PATH
git review -y    # -y skips the "does not appear in gerrit" prompt
```

Capture and surface the change URL from the remote output:

```
remote: https://review.opendev.org/c/<GERRIT_PROJECT>/+/<NNN> <subject> [NEW]
```

Hand off: "Change
https://review.opendev.org/c/openstack/nova/+/<NNN> opened.
CI and reviewers can now pick it up. `git review` will update
the same change on subsequent pushes."

**4d — If Gerrit warns or rejects on push**

Common push outcomes and how to recover without creating a
stray new change:

| Gerrit message | Cause | Fix |
|---|---|---|
| `warning: subject >50 characters; use shorter first paragraph` | Subject too long; push still succeeds | `git commit --amend -s -m "<shorter subject>"` (keep the same body and `Change-Id`), then `git review -y`. Gerrit updates the same change as a new patch set (PS2). |
| `commit <sha>: not Signed-off-by author/committer/uploader` | `--amend` was used without `-s`, dropping the trailer | `git commit --amend -s` (no other changes) and re-push. The commit-msg hook only re-adds `Change-Id`; you must pass `-s` explicitly. |
| `commit <sha>: missing Change-Id in message footer` | Same hook-not-installed issue, but for Change-Id | `git commit --amend -m "<existing body>"` and re-push (the hook will add `Change-Id`). |
| `remote unpack failed: error Missing tree <tree-sha>` | Local commit references a tree Gerrit doesn't have (usually because the parent of the commit isn't yet in Gerrit, or you pushed from a branch whose tip diverged) | Run preflight #4 again (`git fetch gerrit <defaultbranch>` + `git rev-list --left-right --count gerrit/<defaultbranch>...<defaultbranch>`). If behind, rebase the feature branch onto `gerrit/<defaultbranch>`. If ahead with unrelated commits, ask the user. |
| `! [remote rejected] HEAD -> refs/for/<defaultbranch> (no new changes)` | Local commit is identical to what's already on Gerrit (the previous push already landed) | Nothing to do. The change is already in Gerrit. |

For all of the above, the recovery is a fresh `git review -y` —
Gerrit updates the same change as a new patch set, not a new
change, as long as the `Change-Id` in the amended commit
matches.

**4e — Re-submitting refined work for the same bug
(review-feedback iterations)**

Real bug fixes almost never land on the first push. The
common loop is:

```
PS1 pushed  →  reviewer comments  →  refine source/test/note
            →  git commit --amend (keep Change-Id)  →  PS2
            →  reviewer comments  →  ...  →  PSn  →  merge
```

When the user says something like
"apply sean mooney's review feedback",
"address the second review comment",
"我按你提的意见改了", or
"再修一下另外两个 issue", they want a **new patch set on the
existing Gerrit change**, not a new commit/branch/change.

Workflow:

1. **Re-read the review comments** in the Gerrit change page
   (or the local `feedback.json` if available). Confirm what
   needs to change — don't guess, don't over-refactor.
2. **Stay on the existing feature branch** (e.g.
   `bug-001/zero-cputime-delta`). Do **not** create a new
   branch; do **not** `git checkout <defaultbranch>`. The
   branch already carries the right `Change-Id`.
3. **Edit the source / test / release note** in the working
   tree as in Phase 2/Phase 3. The previous patch set's
   files are already in the index, so any further edits
   accumulate on top of them.
4. **Re-run pre-commit + unit tests** (Phase 3) on the
   newly-changed files. Stale `.pyc` or lint regressions
   will re-appear.
5. **Audit for stale artifacts from the previous patch set**:
   - **Stale release note.** The previous release note (if
     any) is still part of the commit. If the refined
     behavior contradicts it, the old note is now wrong.
     Compare the new release note's content against the
     current code; if the old note describes a behavior the
     code no longer has, `git rm` the old note and amend
     without it. The most common symptom: two notes under
     `releasenotes/notes/` describing the same fix in
     incompatible ways.
   - **Stale test.** If the previous patch set's test
     asserted the old behavior (e.g. "expects 0% for
     percent metrics"), update the assertions to match the
     refined behavior. Don't leave an old test failing or
     asserting something that's no longer true.
   - **Stale body text in the commit message.** The
     previous body may describe the old behavior. Rewrite
     the body to reflect the refined behavior; the diff
     between the old and new commit messages is the
     "what changed" story the reviewer will read.
6. **Amend with the updated commit message**, preserving
   the existing `Change-Id`. The commit-msg hook keeps the
   `Change-Id` if the previous commit already has one:
   ```
   git commit --amend -s -F <new-message-file>
   ```
   `<new-message-file>` is a complete new message. The
   commit-msg hook validates the format but does not
   regenerate `Change-Id` when one is already present, so
   the same `Change-Id: I…` stays in the amended commit
   and Gerrit treats the push as a new patch set on the
   same change.
   - **Use `-F` not `-m`.** This is the only way to keep
     `Change-Id`; `-m "…"` triggers hook regeneration.
     Worked example and rationale in **Phase 4b** above.
   - **If the original commit's `Author:` or
     `Signed-off-by:` has the wrong email** (often from a
     hand-written `-c user.email=…` that hid a wrong git
     config), fix `git config user.email` first, then
     `git commit --amend --reset-author -s -F
     <message-file>` — `--reset-author` rewrites both
     `Author:` and `Committer:` from the (now-correct) git
     config, and `-s` regenerates `Signed-off-by:` from
     the same source. Don't just patch the email in the
     message file; the bad config will keep biting future
     commits.
7. **Pre-submission check** (see **4f** below). Pay extra
   attention to subject length and `Signed-off-by` —
   refined work tends to grow the subject past 50 chars,
   and a typo in the amend flags tends to drop the
   trailer.
8. **Re-push**:
   ```
   git review -y
   ```
   Gerrit will respond with
   `remote: SUCCESS ... updated: 1` (not `NEW`), confirming
   it's a new patch set on the existing change. The change
   URL is the same as the previous push.

Edge case — **subject length crept over 50 chars**:
A common reason to amend is that the refined subject is
longer than the original (e.g.
"cpu monitor: refine handling of zero/negative cputime
delta" → 59 chars, Gerrit lint warning). Shorten before
amending: pick a more concise phrasing, ideally one that
captures the refinement's *shape* (e.g. "split", "refine",
"handle") rather than restating the full description. See
**4f — Pre-submission check** below for the ≤ 50 char rule.

Edge case — **the refined change drops a file** (e.g.
`git rm releasenotes/notes/fix-…-zero-delta-….yaml`):
`git rm` stages the deletion. The amend picks it up
automatically. Verify with
`git diff --cached --stat` before amending.

**4f — Pre-submission check (run before `git review -y`)**

Before pushing, run a quick local audit so the push is
accepted cleanly the first time. Gerrit's lint warnings
don't reject the push, but they pile up review noise; the
hook check (`not Signed-off-by author/committer/uploader`)
*does* reject, and a fresh push costs a review round-trip.

Checklist (all should pass):

```
# 1. working tree
git status --short
#   expect: only the three changed files modified, no
#   untracked session artifacts (.ai-auto-collection.*,
#   .codegraph/, .mcp.json, .tmp/)

# 2. branch
git branch --show-current
#   expect: bug-NNN/...-slug, NOT <defaultbranch>

# 3. subject length (Gerrit lint)
SUBJ="$(git log -1 --format='%s')"
[ ${#SUBJ} -le 50 ] || echo "subject > 50 chars: shorten"
#   expect: subject ≤ 50 chars

# 4. body 72-col wrap
git log -1 --format='%b' | awk '{ if (length > 72) print NR": "$0 }'
#   expect: no output

# 5. Change-Id and Signed-off-by present and well-formed
git log -1 --format='%B' | grep -E '^Change-Id: I[a-f0-9]{40}$'
git log -1 --format='%B' | grep -E '^Signed-off-by: [^<]+ <[^>]+>$'
#   expect: each line matches

# 6. diff vs the commit's parent — no stale release notes
#    describing behavior that contradicts the current code
git diff HEAD~1 --name-only -- releasenotes/notes/
#   expect: only the current change's release note(s).
#   If you see two notes with overlapping but conflicting
#   descriptions, the older one is stale — git rm it and
#   amend.

# 7. unit tests still pass on the amended file
.tox/py3/bin/python -m unittest <dotted.test.path> -v
#   (or with OS_<PKG>_DISABLE_EVENTLET_PATCHING=False if the
#    adapter says the project uses eventlet)

# 8. pre-commit clean on the amended files
.tox/pep8/bin/pre-commit run --files <changed files…>
```

If any of (3)–(5) fail, fix before pushing (typically one
amend: `git commit --amend -s -m "<shorter subject>" -m
"<body>"` for a subject shorten; the commit-msg hook will
re-add `Change-Id`; `git commit --amend -s` for a SOB
recovery). If (6) fails, `git rm` the stale note and amend
with `--no-edit`, then re-run the checklist.

Surface a one-line summary to the user before pushing:
"Subject 46 chars (≤ 50 ✓), Change-Id + SOB present,
unit tests + pre-commit clean, no stale release notes —
pushing."

## Bundled resources

- `_shared/openstack-bug/references/openstack-conventions.md` —
  HACKING.rst rules cheat sheet (project-agnostic). Always
  loaded.
- `_shared/openstack-bug/references/bug-classes.md` — the
  canonical 8 bug classes; useful for picking the right guard
  pattern.
- `_shared/openstack-bug/references/project-detection.md` —
  the project detection steps; run once at the top of
  Phase 1.
- `_shared/openstack-bug/references/project-adapters/<project>.md` —
  project-specific HACKING rules, code patterns, and
  anti-patterns. Loaded based on `.gitreview`.
- `_shared/openstack-bug/references/project-adapters/default.md` —
  fallback for projects without a per-project adapter.

## Anti-patterns (do NOT do these)

- ❌ **Re-scanning the project for new bugs** — use the
  project's `*-bug-scanner` for that.
- ❌ **Modifying unrelated code** while fixing the bug.
- ❌ **Running `git commit` or `git push` without
  explicit user consent** — project's conventions require
  the user to ask. Default to stage-only.
- ❌ **Stage-then-push silently** — always surface the
  Gerrit change URL after a successful push.
- ❌ **Raw `git push` to Gerrit** — use `git review`, which
  picks up `.gitreview` and the correct target branch. Raw
  `git push` is for GitHub-style remotes.
- ❌ **Pushing from `<defaultbranch>`** — create a feature
  branch first. `<defaultbranch>` → `refs/for/<defaultbranch>`
  re-pushes history and gets rejected with "Missing tree".
- ❌ **Silently fixing a wrong `gerrit` remote URL** — if
  `.gitreview` says `openstack/nova.git` but the remote says
  `openstack/cinder.git`, stop and ask. A wrong-project push
  is hard to retract.
- ❌ **`pip install` without user consent** — project
  conventions forbid unprompted installs. If `git-review`
  is missing, ask first.
- ❌ **Squashing the fix + test + release note into
  one monstrous commit** — the project's Gerrit
  convention is **unsquashed** series (but a single
  bug fix is fine as one commit when the user
  prefers).
- ❌ **Handcrafting a `Change-Id` placeholder** — the
  commit-msg hook auto-generates it. Just omit it from the
  message body and let the hook add it.
- ❌ **Including `Co-Authored-By`** — Gerrit rejects commits
  that contain it.
- ❌ **Adding a Python test that imports the new
  release note**.
- ❌ **Trusting the user's claim without re-reading the
  source** — the source may have changed since the scan;
  always re-verify the trigger.
- ❌ **HACKING rules with project-specific paths** (H307,
  H342, H350, H362, H369, H373, etc.) — don't put these in
  the common `openstack-conventions.md`; they belong in the
  project adapter.
