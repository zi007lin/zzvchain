# implw — implement from a GitHub issue (portable)

Implements a scored spec from a GitHub Issue number and drives it to a PR
against `main` without stopping. **Repo-agnostic**: this command makes no
assumption about which repo it runs in or what sibling repos exist. It works
in any HTU repo that carries this command plus `.github/workflows/implw.yml`.

Spec acquisition is the contract defined in [`docs/IMPLW_FLOW.md`](../../docs/IMPLW_FLOW.md)
— that document is the source of truth. This command implements it; if the
two ever diverge, IMPLW_FLOW.md wins.

## Usage

```
implw <issue-number> [--repo <owner/repo>]
```

- **Repo** defaults to the current repo, resolved from
  `git remote get-url origin` (do not hardcode an owner or name).
- **Repo root** is resolved from `git rev-parse --show-toplevel`.
- No hardcoded `~/dev/*` paths and no sibling-repo sync. The runner checks
  the repo out fresh; locally a single `git pull --ff-only` of the current
  repo is enough.

## Steps

### 0. Sync the working repo

Resolve `REPO` from the origin remote and `ROOT` from the git toplevel.
Run `git pull --ff-only` on the current repo (a no-op under a clean CI
checkout; keeps a local clone current). Do **not** touch other repos.

### 1. Fetch the issue

```
gh issue view <N> --repo <REPO> --json title,body,labels,createdAt
```

Determine `spec_type` from the title prefix: `BUG:` / `FEAT:` / `CHORE:` /
`REFACTOR:` / `SPEC:` / `HOTFIX:` → lowercased `bug` / `feat` / `chore` /
`refactor` / `spec` / `hotfix`.

### 2. Acquire the scored spec (IMPLW_FLOW.md §2)

Acquisition is **content-based, not transport-based**: accept the spec from
either path as long as a verifiable PASS score is present. Try Path A first,
then Path B.

**Path A — file-uploaded (legacy / TUI).** Read the `_Source: <file>.md_`
footer in the issue body, then locate the matching
`issues/<name>.scored.md` (also check `issues/parked/`). If the file exists
and contains a `## ZAI Spec Score` block with `**Passed:** YES`, treat the
file body as authoritative.

**Path B — inline-scored issue body (MCP).** When no file resolves, detect
the inline-scored format by **all three** signals (§3):

1. a `## ZAI Spec Score` heading,
2. `**Score:** N/N` where `N` equals the required section count for
   `spec_type`,
3. `**Passed:** YES`.

If present, the issue body is authoritative. **Auto-materialize** it to
`issues/<name>.scored.md` per §4 (filename derivation §4.1, atomic write
with `-v2`/`-v3` collision suffixing §4.2, `## Provenance (auto-materialized)`
footer §4.3) and commit it with the implementation PR (§4.4). Strip the
score block from the working copy used for implementation; keep it in the
materialized file.

**Precedence (§7):** if both paths resolve, **Path A wins** (the local file
is authoritative); do not materialize. Note any content divergence in the
run digest.

**HALT — informational, not a numbered global gate (§3, §8):**

- `**Passed:** NO` → scored but rejected; stop and ask the operator to
  re-score.
- Partial inline signals → stop and name the missing signal.
- Neither path resolves → stop: spec not found; identify what is missing.

### 3. Integrity re-score

Re-run scoring on the acquired spec content via the HTU Skills MCP
`score_spec` tool (pass the correct `spec_type`). Compare the result with
the stored score block. On mismatch → stop (informational): the spec
content and its score block disagree; re-score before proceeding.

### 4. Gate 1 classifier

- **AUTO (proceed):** `chore` (full pass) and `hotfix` (full pass) with no
  `gates[]` declared.
- **HOLD:** `feat` / `bug` / `refactor` / `spec` / research specs, any spec
  declaring `gates[]`, or an issue carrying the `needs-approval` label.

#### Resolving a HOLD — the approval channel depends on execution mode

A HOLD requires solo-operator approval before implementation starts. **How
that approval is signalled differs by execution mode; the HOLD semantics
themselves are identical in both.** This is **channel-correctness, not gate
loosening** — a labeled issue is held in every mode, and the approval act
(daniel-silvers removing the `needs-approval` label, or never attaching it) is
unchanged. The only thing that varies is whether the approval is read from an
in-session reply or from the issue's label state. In both modes, **PR review
by daniel-silvers is still required to merge** — Gate 1 governs whether
implementation starts, never whether the PR merges.

**Mode detection.** Treat the run as **headless** whenever Claude Code is
invoked in print mode (`-p`) or stdin is not a TTY (e.g. the implw runner);
otherwise treat it as **interactive**.

**Interactive (TTY / a human is in the session).** The reply path is
authoritative. Stop and emit:

```
needs_input: non-trivial spec — issue #<N>. Reply "approved" to proceed as
solo operator, or have daniel-silvers remove the needs-approval label.
PR review by daniel-silvers is still required to merge.
```

Proceed only when a solo-operator `"approved"` reply is present in the
session. Approval channel for the run digest: `reply`.

**Headless (`-p` / print mode, or stdin not a TTY).** No in-session reply can
ever arrive (`claude -p < /dev/null` has no reply channel), so the
`needs-approval` label on the target issue is the **sole** approval signal:

- label **present** → HALT `needs_input`; the HOLD stands. daniel-silvers must
  remove the label to approve.
- label **absent** → proceed as **solo-operator approved**. The approval act
  is daniel-silvers having removed or never attached the label — recorded
  out-of-band of the session, not via an in-session reply. Approval channel
  for the run digest: `label-absent` (attributed to daniel-silvers).

**Run-digest record.** On Gate 1 clearance, log which approval channel cleared
the gate: `reply` (interactive) or `label-absent` (headless, attributed to
daniel-silvers). AUTO-classified specs clear without a HOLD and need no
channel record.

### 5. Implement (no-stop)

- **Branch:** `htu/<slug>` cut from `origin/main`, where `<slug>` is the
  kebab-cased issue title (optionally suffixed `-<N>`). The `htu/` prefix is
  the HTU-wide convention (see CLAUDE.md). _Grandfather:_ repos with legacy
  `stt/` or `<type>/` branches in flight may finish them under their existing
  names, but new branches use `htu/<slug>`.
- Implement the spec exactly. Run `npm run lint` / `npm run build` / tests
  where the repo configures them; abort on failure (do not commit).
- Commit with the single required trailer `Co-authored-by: ZiLin
  <noreply@zzv.io>`. **Never** add a Claude / Anthropic / AI co-author.
- Include the Path B materialized spec file in the diff (§4.4).

### 6. Open the PR

```
gh pr create --base main --title "<type>: <description> (#<N>)"
```

Request `daniel-silvers` as reviewer, embed the `## ZAI Spec Score` block in
the PR body, and reference `Closes #<N>`. Do not stop between steps.

### 7. Report

Output:

```
PR #<n> opened: <url>
Branch: <branch>
Acquisition: <file-uploaded | inline-scored> · Spec: issues/<name>.scored.md
Score: <score> · Type: <spec_type> · Rubric: <rubric_version>
Gate 1: <AUTO | reply | label-absent>
```

`Gate 1` records how the gate cleared (§4 run-digest record): `AUTO` for an
AUTO-classified spec, `reply` when an interactive HOLD was cleared by an
in-session approval, or `label-absent` when a headless HOLD cleared on the
absence of the `needs-approval` label (attributed to daniel-silvers).

---

_Canonical, repo-agnostic command. Ported to participating HTU repos under
CHORE htu-foundation#168 (EPIC #156). Behavioral contract: see
[`docs/IMPLW_FLOW.md`](../../docs/IMPLW_FLOW.md)._
