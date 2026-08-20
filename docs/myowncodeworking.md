# Having S17Code Write Its Own Product Code — What Actually Happened

This document is the full record of a separate effort from the main
`implementation.md` log: an attempt to have the already-submitted Perplexity-
clone product (`s17-answers-engine`) rewritten by **S17Code's own live-graph
coding capability** — its `read_code`/`edit_code`/`create_file`/`run_command`
loop, driven through the real HTTP API — rather than by Claude Code editing
files directly. It records the full chain of diagnosis, one real bug found
and fixed along the way, and the final, honestly-reported outcome: the
rewrite was not achieved, and why.

---

## 1. The flaw that triggered this

The user flagged the already-submitted `deephazar-eva-ai/s17-answers-engine`
repo: its README's required "who wrote what" section states plainly —

> "Claude (Sonnet 5, via Claude Code) wrote every line of code in this
> submission... I did not hand-type any of it."

The user's objection: the product code was supposed to be written by
S17Code's **own** coding skill/capability, not by an external coding
assistant operating outside the harness. Two clarifying questions were asked
before touching anything, given the scale of what "fixing" this could mean:

1. **Mechanism** — does "written by the coding skill, not Claude" mean
   driving S17Code's own `read_code`/`edit_code`/`create_file`/`run_command`
   capabilities through its live-graph API (with Claude Code only
   orchestrating/monitoring), versus something else (e.g. writing a new
   `SKILL.md` guidance doc)? **Answer: drive S17Code's own coding
   capability.**
2. **Scope** — does this mean redoing just this session's own unpushed
   changes (the DOCX/PPTX export work), or the entire original,
   already-submitted, already-demoed product? **Answer: the entire original
   product.**

This second answer raised the stakes considerably: `deephazar-eva-ai/
s17-answers-engine` already links a published, unlisted YouTube demo video
(`https://youtu.be/PlVNkLKVKdE`) and the actual product code lives on the
already-pushed `feat/answers-engine` branch of `deephazar-eva-ai/S17Code`.

---

## 2. Building an isolated, history-free workspace

S17Code's coding capabilities operate inside whatever directory
`S17_WORKSPACE` points at (`s17code/coding/workspace.py`), and the git
history inside that workspace is fully reachable by the agent — `run_command`
allows `git log`, `git show`, and `git checkout` (`s17code/coding/exec.py`'s
`GIT_SUBCOMMANDS` allowlist). Pointing the coding capability straight at the
existing `S17Code` checkout, or a plain clone of the `feat/answers-engine`
branch, would let the agent simply discover and reuse the old, already-
Claude-written implementation via `git log --all` / `git checkout` — defeating
the entire point.

To get a genuine independent rewrite:

1. Cloned `deephazar-eva-ai/S17Code` at `feat/answers-engine` into a scratch
   directory (`agent-workspace-s17code`), confirmed the real commit history
   and found the merge-base with `origin/main`: commit `15ab9c7` — the last
   commit **before** the answers-engine feature was ever added.
2. Used `git archive 15ab9c7 | tar -x` to export just that pre-feature file
   tree into a brand-new directory, `agent-rebuild-workspace`, with **no**
   `.git` directory at all.
3. Ran `git init` fresh in that directory with a single commit ("base:
   pre-answers-engine state") — no other branches, no remotes, no reachable
   history of the real implementation whatsoever.
4. Verified: `find . -iname "*answer*"` in the fresh export returned nothing;
   `git branch -a` / `git remote -v` in the new repo showed only `master`
   and no remotes.
5. Pointed `S17_WORKSPACE` at `agent-rebuild-workspace` in `S17Code/.env`
   and restarted the local dev server so the change took effect (`Workspace.
   from_env()` reads the env var fresh per capability call, but the process's
   own `os.environ` is only populated from `.env` at import time via
   `load_dotenv`, so a running server needs a restart to see a `.env` edit).

The live, currently-served `/answers` app (with this session's own unrelated
export-feature work still sitting uncommitted on it) was never touched by
any of this — `S17_WORKSPACE` only controls where the *coding capability*
operates, not what the FastAPI app itself serves.

---

## 3. Attempt 1 — the full original spec, in one run

Extracted the functional spec directly from the submitted README's "Part 1 —
the product" section (search box; live per-node reasoning panel; answer panel
gated on a real `answer_with_evidence` success; numbered Sources list built
from `hits[].url`/`pages[].url`; safe DOM-only markdown rendering, never
`innerHTML`; a raw-snapshot viewer; visibly distinct honest-refusal styling;
visible failure handling) and handed it to the agent as a goal, along with
pointers to the real files it would need to read first (`s17code/routes.py`,
`s17code/ui/routes.py`, `s17code/ui/agui.py`, `s17code/main.py`) so it would
ground itself in the actual contracts rather than guess field names — the
same kind of reading Claude Code itself did originally, per `implementation.
md` §1.

Kicked off as `POST /v1/agent/runs/async` with `allowed_side_effects:
["edit_code", "create_file", "run_command", "git_reset", "validate_work"]`,
`respond_as: "text"`, no `budget` set.

**Result**: the agent correctly loaded both the `a2ui` and `web-pages`
skills (matching their own trigger keywords), and used `create_file` to
produce a real but skeletal 67-line `answers.html` — search box, `EventSource`
wiring, a bare reasoning panel and answer panel — recognizably the right
*shape*, but missing the markdown-safety rule, the sources list, the
honest-refusal styling, and the snapshot viewer. It never touched the backend
at all (no `/runs/async` endpoint, no `/answers` route). The run was then
force-terminated by three consecutive gateway failures:

```
planner call failed visibly: RuntimeError: gateway /v1/chat returned 503:
{"detail":"all providers unavailable. attempts:
[{'provider': 'cerebras', 'reason': 'prompt 43898 > max_ctx 8000'}]. last_error: None"}
```

(and again at 107215 and 71364 tokens). `git status` in the workspace
afterward confirmed: one untracked file, no edits to any existing file.

---

## 4. Splitting into smaller runs, and a broken first result

Given the context-overflow failure, and per the user's choice, the task was
split into smaller, separately-run pieces: (1) the backend `/runs/async`
endpoint alone, (2) the `/answers` route alone, (3) `answers.html` alone,
(4) a final test/diff pass. Workspace reset (`git checkout -- .` + `git clean
-fd`) between attempts throughout.

**Run 1 (backend endpoint), first attempt**: reached a succeeded
`answer_with_evidence` node and self-reported success, citing a diff in its
own answer text. But cross-checking the *actual* `git diff` in the workspace
against the real APIs (not trusting the agent's self-report, which turned
out to describe a cleaner draft than what actually landed) found two real
bugs:

- **Wrong route path**: `@router.post("/agent/runs/async")` decorated on a
  router already mounted with `prefix="/v1/agent"` — the real final path is
  `/v1/agent/agent/runs/async`, not the intended `/v1/agent/runs/async`. The
  endpoint is unreachable at its documented URL.
- **A hallucinated method**: `request.app.state.runtime.graph.register(run_id,
  ...)` — verified directly against `s17code/core/live_graph/store.py`'s
  `GraphStore` class; no such method exists. Calling it would raise
  `AttributeError` at runtime.

Also present: duplicate `import os` / `import asyncio` / `import uuid`
statements inside the function body (left over from several failed
"fix_missing_os_import" attempts visible in the node list), and
`allowed_side_effects=body.allowed_side_effects` passed as a list where the
original pattern elsewhere in the file uses a set — stylistically
inconsistent, not necessarily fatal.

---

## 5. A real bug found along the way: `run_command_worker`

Every `run_command` node in that same run (`run_tests_initial`,
`run_tests_fixed`, `run_tests_fixed_v2`, and `validate_final_state`) failed
with:

```
TypeError: Object of type CommandResult is not JSON serializable
```

Traced to `s17code/workers/coding.py`:

```python
async def run_command_worker(ctx: RunContext, task: TaskSpec) -> dict[str, Any]:
    return run_command(ctx.workspace(), task.input["command"],
                       timeout=int(task.input.get("timeout", 120)))
```

`run_command()` (in `s17code/coding/exec.py`) returns a `CommandResult`
dataclass, not a dict — and that dataclass already has its own `as_dict()`
method, written for exactly this purpose, that was simply never called here.
Grepping the whole codebase for `as_dict()` confirmed every other capability
worker in the same file (`read_code_worker`, `edit_code_worker`,
`create_file_worker`, etc.) already unwraps its own coding-layer result
into a plain dict; `run_command_worker` was the one that didn't. No existing
test caught it because `tests/test_coding_surface.py` exercises the pure
`run_command()` function directly, never the async worker wrapper that
actually crosses into a JSON-serialized graph node result.

This meant the coding agent had been flying blind on its own test/validation
feedback the entire time — every self-check it tried to run came back as an
opaque crash instead of real pass/fail signal, which is a very plausible
contributor to why the earlier broken endpoint was never caught and fixed by
the agent itself.

**Fix** (one line, in the live `S17Code` repo — not the isolated rebuild
workspace, since this is a real defect in the harness itself):

```python
return run_command(ctx.workspace(), task.input["command"],
                   timeout=int(task.input.get("timeout", 120))).as_dict()
```

**Verified fail-before/pass-after**, the same discipline this project's own
bug-fix PRs use: temporarily reverted the one-line fix with `sed` (after an
earlier, more dangerous `git stash` had accidentally reverted *every*
uncommitted change in the live repo across this whole session — caught
immediately and fully restored via `git stash pop` before any real damage;
the `sed`-based approach was used afterward specifically to avoid repeating
that mistake), confirmed the test suite failed with the exact reproduced
error, restored the fix, confirmed it passed. Added
`tests/test_run_command_worker_serialization.py` (3 tests: the worker
returns a plain dict, that dict is actually `json.dumps`-able, and the dict
carries the real exit code/stdout/stderr). Full suite: 529 passed, 1
pre-existing skip, `ruff check` clean. Restarted the local dev server so it
picked up the fix.

---

## 6. Retrying Run 1 — hit the node cap instead

Cleaned the rebuild workspace and reran the same backend-endpoint goal with
the fix in place. This time it never reached a terminal node at all: it ran
to the planner's hard `max_nodes` cap (32) after **24 consecutive `edit_code`
failures**, every one:

```
EditError: old_string does not appear in s17code/routes.py.
It must match exactly, including indentation and whitespace.
```

Inspecting the actual `old_string` values used showed the agent wasn't
quoting its own `read_code` output — it was inventing a plausible-looking
but fabricated function (`async def run_agent(...)` with a fake `hashlib.
sha256(...)`-based `run_id` and a `# ... (existing logic)` placeholder
comment) that resembles the real function in shape but matches none of its
actual text.

---

## 7. First diagnosis, and a self-correction

Initial hypothesis: the planner role is pinned to the `frontier` tier
(`config/tiers.yaml`'s `role_tiers.planner: frontier`, which maps to GitHub
Models / `openai/gpt-4.1`), and a direct test confirmed GitHub Models is in
a real, active outage right now:

```
{"detail":"github failed: github HTTP 410:
{\"error\":{\"code\":\"github_models_retirement_brownout\", ...}}"}
```

— the same incident already documented in `.env`'s own comments from
2026-08-14. The user approved temporarily repointing `role_tiers.planner`
from `frontier` to `standard` (gemini) to work around it.

**Before making that change, a closer read of `s17code/runtime.py` caught
that the diagnosis was wrong**: `EconomicsConfig`/`role_tiers`/tier routing
is only invoked when a run sets a `budget` field (`runtime.py`'s `if budget
is not None: economics_config = economics or EconomicsConfig.load()`).
None of the rebuild runs set a budget — they all use the plain, unmetered
`gateway_text_llm` path, which never touches `role_tiers` or the `frontier`
tier at all. The `tiers.yaml` edit the user had just approved would have
had **zero effect** on these specific runs, and was not applied. This
correction was reported to the user directly rather than silently proceeding
with a change known not to work — the GitHub Models outage is real and
independently confirmed, but it was not the actual cause of these failures.

---

## 8. The real mechanism: glc_v5's per-key cooldown

Reproduced the actual failure directly: six identical, large (~63k-token)
`POST /v1/chat` calls fired back-to-back at `provider: gemini`. The first
two succeeded (200 OK, ~1.7s each); the third through sixth failed
**instantly** (~0.02s — far too fast to be a real upstream round trip) with:

```
{"detail":"all providers unavailable. attempts:
[{'provider': 'gemini_1', 'reason': 'cooldown (0.5s)'},
 {'provider': 'gemini_2', 'reason': 'cooldown (2.2s)'}]. last_error: None"}
```

Traced to `glc_v5/glc/routing/core.py`'s provider limits table:

```python
"gemini": {"rpm": 15, "rpd": 1000, "tpm": 250000, "cooldown": 4, "max_ctx": 1000000},
"cerebras": {..., "max_ctx": 8000},  # inferred from the earlier error text
```

Each gemini key enforces a 4-second cooldown between calls; with two keys in
the pool, the real sustainable ceiling is roughly one call every ~2 seconds.
When both keys are cooling down, glc_v5 returns `503` immediately, with no
upstream call attempted at all. `s17code/gateway.py`'s `GatewayClient.chat()`
treats `503` as retriable, retries with backoff, and on continued failure
falls through to the configured fallback provider — `cerebras`, whose
`max_ctx: 8000` is what produced the original context-overflow errors in §3,
and whose apparent unreliability at literal exact-string quoting plausibly
explains the hallucinated `old_string` values in §4 and §6.

This is `glc_v5`'s own rate-limiter working as designed, colliding with two
things at once: the live graph's replanning loop firing planner calls faster
than the per-key cooldown allows as a coding task's context grows with every
`read_code` result, **and** this entire diagnostic session's own heavy
manual `curl` traffic sharing the exact same key pool and adding contention
on top of whatever a live run was doing.

---

## 9. A clean retry — and the deeper finding

Cleaned the workspace, waited for cooldowns to clear, and reran Run 1's
exact backend-endpoint goal a third time, deliberately making **no**
competing manual gateway calls during its execution window.

**Result**: still ran to the 32-node cap. Still 17/17 `edit_code` failures.
Still the identical fabrication pattern — inspecting the failed `old_string`
directly showed it inventing a different-but-equally-fake function
(`async def run_agent(..., _=Depends(require_control)) -> dict[str, Any]:
return await export_run(request, body)` — note `export_run` is a real
function in the file, imported for an unrelated telemetry-export purpose,
suggesting the model was blending unrelated pieces of what it had read
rather than quoting any one part of it exactly), even though the
corresponding `read_code` calls immediately beforehand all succeeded and
returned the real file content.

This ruled out the cooldown/fallback mechanism as the sole or even primary
cause: under genuinely clean, uncontended conditions, with real `read_code`
output available, the model driving these planning decisions still did not
ground `edit_code`'s `old_string` argument in what it had actually just
read. Across three separate attempts (contended, cache-cold-after-fix, and
clean), the failure mode was consistent — a genuine, reproducible
reliability limitation of the current coding-capability setup for this task
shape, not an infrastructure problem fixable by waiting, retrying, or
adjusting provider routing.

---

## 10. One more narrowly-scoped test, to isolate whether ANY edit can succeed

Before writing this off as a hard capability floor, one more run tested the
smallest possible case: read the top of `s17code/routes.py`, then make
**exactly one** `edit_code` call inserting a single line (`import asyncio`)
right after the existing `import json` line — nothing else — with an
explicit instruction that `old_string` must be copied character-for-character
from the real `read_code` output, not retyped from memory, and a fallback
instruction to re-read and re-copy rather than adjust from memory if the
first attempt failed.

**Result: it worked, first try.** Four nodes total (`read_routes`,
`edit_routes`, `git_diff_final`, `answer_final`), all succeeded. The actual
`git diff` in the workspace:

```diff
--- a/s17code/routes.py
+++ b/s17code/routes.py
@@ -4,6 +4,7 @@ from __future__ import annotations
 import hashlib
 import hmac
 import json
+import asyncio
 import os
 from datetime import UTC, datetime
 from typing import Any
```

Exactly the requested change, nothing else touched (`git diff --stat`: 1
file, +1/-0), file still parses (`ast.parse` clean). Crucially, the
`old_string` the agent actually used —
`"import hashlib\nimport hmac\nimport json\nimport os"` — is a byte-exact
match of the real file's text, not a reconstruction. This is the same model
that fabricated file content in every earlier attempt, succeeding cleanly
once the target was small and the instruction was explicit about copying
verbatim.

**What this changes about the diagnosis in §9**: the earlier conclusion —
"a genuine model reliability limit on faithfully quoting file content" — was
correct as a description of what happened, but incomplete as a verdict on
the capability itself. The failure scales with the *size and complexity* of
what has to be reproduced from memory across a longer plan, not a blanket
inability to use `edit_code` correctly at all. A one-line insertion next to
an obvious, short anchor succeeded; a multi-line function signature and body
constructed from an earlier, larger `read_code` call did not. The practical
path this opens up, if the full rewrite is picked up again: proceed as a
sequence of small, atomic `edit_code` calls — one insertion at a time, each
scoped the way this test was — rather than one call asked to insert a whole
new multi-line function in a single shot.

---

## 11. Where this stands

**Achieved and real:**
- An isolated, history-free rewrite workspace, built and verified to leak
  none of the original implementation (`agent-rebuild-workspace`).
- A genuine bug found and fixed in `S17Code` itself:
  `run_command_worker`'s missing `.as_dict()` call, with a fail-before/
  pass-after regression test (`tests/test_run_command_worker_serialization.
  py`), verified against the full suite (529 passed).
- A fully traced, dual-confirmed understanding of why the larger rewrite
  attempts kept failing: a real GitHub Models outage (confirmed, but not the
  actual cause here), a real glc_v5 gemini-key cooldown/cerebras-fallback
  chain (confirmed, a real contributing factor for the earliest attempts),
  and — the dominant cause — a model reliability limit that scales with the
  size of what `edit_code` has to reproduce from memory, not a blanket
  inability to edit at all: a minimal, single-line, explicitly-verbatim test
  (§10) succeeded cleanly on the first try.

**Not (yet) achieved:** the self-authored rewrite of the full
Perplexity-clone product. Four full-scope/backend-endpoint run attempts
failed to produce a single successful multi-line `edit_code` call; only the
final, deliberately minimal single-line test succeeded. The full original
product (backend endpoint + `/answers` route + `answers.html`) remains
unbuilt this way — but §10 leaves a concrete, evidenced path forward (small,
atomic edits) rather than a dead end.

**Left in place, for whoever picks this up next:**
- `S17Code/.env`'s `S17_WORKSPACE` is currently pointed at
  `agent-rebuild-workspace`, not the toy `coding-workspace/` fixture it
  pointed at before this investigation started.
- `agent-rebuild-workspace` currently holds §10's one-line `import asyncio`
  change, uncommitted, in an otherwise-clean single-commit repo.
- The `run_command_worker` fix and its test are real, live changes on
  `S17Code`'s `feat/answers-engine` branch, uncommitted alongside this
  session's other unrelated work (the DOCX/PPTX export feature, the
  `/composed` and `app.html` token fixes) — not yet committed or pushed.
- `config/tiers.yaml` was **not** modified — the approved change was
  correctly identified as ineffective before being applied, and skipped.

---

## 12. Picking the remaining task back up — the full rewrite, done small

Resumed exactly where §10 left off: instead of one large one-shot goal per
piece, each remaining deliverable was driven through S17Code's own
`read_code`/`edit_code`/`create_file` capability as a sequence of small,
independently-verified runs against `agent-rebuild-workspace`, with every
result cross-checked against the real `git diff` and, where possible, a real
import or a real running server — never against the agent's own self-report.

Both supporting services were restarted first (`glc serve` on `:8111`,
`s17code serve` on `:8113`, both had been left down) and the gateway was
confirmed healthy and uncontended before any run.

**A refinement to §10's finding.** The earlier conclusion was "keep `old_string`
short, insert one line at a time." Testing this run showed the real
constraint is narrower: `old_string` must be short and copied verbatim — but
`new_string` can be arbitrarily long, since it's freshly authored text with
nothing to mismatch. A whole ~35-line function was inserted in a **single**
`edit_code` call by anchoring on one short, unique existing line (e.g.
`@router.post("/channel-messages")`) and writing the new function plus that
same anchor line as `new_string`. This is what made the rest of this session
tractable — one atomic call per insertion point, not one per line.

**Backend endpoint (`POST /v1/agent/runs/async`), built and verified:**
- Four tiny anchored edits (two new imports, a `log` binding, an
  `app.state.background_runs = {}` line) landed correctly on the first try.
- The endpoint function itself landed via one anchored `edit_code` call —
  correct shape, but with two real authoring bugs on first pass: `llm=
  gateway_text_llm` (a 3-arg function passed where a 2-arg callable was
  required — would `TypeError` on first use) and `transport="http"` (a
  string passed where the real `GatewayClient` object was required — would
  `AttributeError` on first use). Neither is a transcription error like
  earlier attempts; both are real logic mistakes, caught by re-reading the
  diff against `runtime.py`'s actual call sites, not by trusting the agent's
  "succeeded" self-report.
- Fixed in one more run, two anchored edits, both landed clean.
- Verified independently, outside the agent: `.venv/bin/python -c "import
  s17code.routes"` succeeds, `run_async` is present with the right
  signature.

**`/answers` route, built and verified:**
- One anchored `edit_code` call, mirroring the existing `/app` route's exact
  pattern. Landed clean except one unrequested, harmless-but-unused `import
  os` the agent added on its own initiative — removed in one follow-up
  anchored edit.

**`answers.html`, built and verified — the part that actually broke, and how
it was recovered:**
- First pass: one `create_file` call, grounded in reading `app.html` (for
  its existing safe, `innerHTML`-free markdown-to-DOM renderer, reused
  verbatim) and `agui.py`/`ui/routes.py` (for the real AG-UI event shapes
  and the real SSE route). Result: a real, syntactically valid, but
  incomplete page — search box, live reasoning panel, gated answer
  rendering, and the raw-snapshot toggle all worked; the Sources list and
  the honest-refusal panel were entirely missing, matching the same
  "recognizably right shape, spec-incomplete on one shot" pattern from §3's
  very first attempt.
- Second pass, adding the missing pieces: mostly landed clean (source
  collection/rendering, a content-type header fix), but one edit — inserting
  `RUN_FINISHED` handling — genuinely broke the file. The agent's own retry
  loop (triggered by an `old_string`-appears-twice error) ended up inserting
  a duplicate, malformed block **outside** the enclosing function, leaving
  invalid JavaScript. This was caught by an independent `node --check` run
  outside the agent's own reported "succeeded" results — the agent's last
  self-check in that run had reported success.
- Recovered by restoring the last known-good (agent-authored, just
  incomplete) version directly — a workspace-state revert, the same kind of
  orchestrator-level correction as §5's `git stash`/`sed` moves, not new
  authored logic — then re-issuing the same fix as three smaller, more
  precisely and unambiguously anchored edits instead of one instruction
  bundling several placement decisions together. This time all three landed
  correctly on the first try, and both the agent's own `node --check` and an
  independent one outside the agent confirmed valid syntax.
- **End-to-end verified against a real running instance**, not just
  statically: started a second `uvicorn` process directly on
  `agent-rebuild-workspace`'s own code (port `:8213`, its own `.env`,
  gitignored, pointed at the same live gateway), confirmed `GET /answers`
  returns 200, fired a real `POST /v1/agent/runs/async` request, and read
  the real SSE stream back from `GET /v1/runs/{id}/events`. It produced
  exactly the event shapes `answers.html`'s JS expects — `STEP_FINISHED`
  with `stepName` and `delta.value.answer`, then `RUN_FINISHED` — for a live
  round trip through the real graph, not a mock.

**Where this leaves §11's "not yet achieved":** the full original product
(backend endpoint + `/answers` route + `answers.html`) is now built inside
`agent-rebuild-workspace`, entirely through S17Code's own coding capability,
every real bug caught by independent verification and fixed through the same
capability, and the whole thing confirmed working end-to-end against a real
running instance. It is a leaner rewrite than the original 685-line
submission — no DOCX/PPTX export (correctly out of scope; that was this
session's unrelated later addition, not part of the README's "Part 1"
spec), a plainer visual language than `app.html`'s full design system, and
no dedicated raw-node-by-node status iconography beyond plain text lines —
but every required behavior from §3's spec is present and independently
verified: search box, live per-node reasoning panel, gated answer panel,
numbered clickable Sources built from `hits[]`/`pages[]` `url` fields, safe
DOM-only markdown rendering, a raw-snapshot viewer, visibly distinct
honest-refusal styling, and visible failure handling.

**Left in place, for whoever picks this up next:**
- `glc serve` (`:8111`) and `s17code serve` (`:8113`) are running in the
  background from this session (`nohup`, logs in the scratch dir under
  `/tmp/claude-1001/.../scratchpad/`) — `S17_WORKSPACE` still points at
  `agent-rebuild-workspace`.
- `agent-rebuild-workspace` now has real uncommitted changes: `s17code/
  main.py`, `s17code/routes.py`, `s17code/ui/routes.py` modified, `s17code/
  ui/client/answers.html` new and untracked, plus a new gitignored `.env`
  (copied from `S17Code/.env` with `S17_PORT`/`S17_A2A_GRPC_PORT` changed to
  avoid colliding with the real server, A2A gRPC disabled). None of this is
  committed.
- The `run_command_worker` fix, its test, and this session's unrelated
  DOCX/PPTX export work are still uncommitted on `S17Code`'s
  `feat/answers-engine` branch, exactly as `§11` left them.

---

## 13. A follow-up session: restart, a provenance question, and a real bug the token auth caught

A separate, later session picked this back up with three small asks:
restart the two background services, explain how to actually verify the
rebuilt product's provenance, and then — after trying it — report that
`:8213` gave "no response."

**Restart.** Both `glc serve` (`:8111`) and `s17code serve` (`:8113`) from
the prior session were killed and restarted clean; both came back healthy
(`GET /docs` → 200 on each).

**The provenance question, and a distinction worth being explicit about.**
Asked how to confirm the Perplexity-clone capability at `localhost:8113/
answers` was actually built by S17Code's own agent. The honest answer: it
isn't — `:8113` serves whatever is checked out on `S17Code`'s
`feat/answers-engine` branch, which is still the original,
Claude-Code-authored implementation this whole investigation exists because
of. `S17_WORKSPACE` only ever controlled where the *coding capability*
(`read_code`/`edit_code`/`create_file`) operates; it has no bearing on what
the running FastAPI app serves to a browser. The only way to actually browse
the agent-built version is to run a second, separate server process
directly against `agent-rebuild-workspace`'s own code — which is what the
temporary `:8213` instance from §12 was. Verification suggested to go with
it, beyond just reading this document: `git log --all` in
`agent-rebuild-workspace` (one commit, no reachable history of the real
implementation), `git status`/`git diff` there (the literal, unedited output
of the coding agent's own tool calls), and the run-by-run trail already in
§12.

**"No response from 8213" turned out to be a real product bug, not a dead
server.** The process was alive and `curl` reached it fine from the shell —
but the access log told the real story: the browser successfully loaded
`GET /answers` (200), then the moment "Ask" was clicked, `POST /v1/agent/
runs/async` came back `401 Unauthorized`, and the page's own follow-up `GET
/v1/runs/undefined/events` (`run_id` was `undefined` because the failed
POST's body had no `run_id` to destructure) 404'd. Nothing was down; the
page's own request was being correctly rejected.

**Root cause: a real gap in the original goal, not a coding-capability
failure.** `POST /v1/agent/runs/async` is a control-plane route — decorated
`dependencies=[Depends(require_control)]` in `s17code/routes.py`, so it
fails closed without a valid bearer token, by design (the S17Code agent
added that same decorator itself, back in §12's backend-endpoint run). The
goal handed to the agent when it wrote `answers.html` never mentioned
authentication at all, so it wrote a plain unauthenticated `fetch()` — which
is exactly why the earlier direct-`curl` verification in §12 passed (that
call supplied the header by hand) while a real browser session, going
through the page's own JS, could not. Checking the real, original
`S17Code/s17code/ui/client/answers.html` confirmed the live implementation
solves this the same way every control-plane-gated static page has to: a
token input field, persisted in `localStorage`, sent as the `Authorization`
header on the request that starts a run — reading a run back (`/events`)
needs no token, only starting one does.

**Fixed the same way as every other bug in this document: through the
agent, verified independently.** One more small run against
`agent-rebuild-workspace`, three anchored `edit_code` calls: add a token
input field to the page, load/persist it via `localStorage`, and merge it
into the existing fetch call's headers (the trickiest part — the file
already had a `Content-Type` header line from an earlier fix, so the edit
had to fold `Authorization` into the same header object rather than add a
second, colliding `headers:` key; the agent got this right on the first
try). Verified independently, same discipline as before: extracted the
inline `<script>` and ran `node --check` outside the agent's own
self-report, then restarted the `:8213` instance and confirmed by hand —
`curl` with no token still correctly gets `401`, `curl` with the real
control token now gets a real `{"run_id": ..., "status": "started"}`,
exactly matching what the page's own JS now sends.

**State after this session:** `agent-rebuild-workspace`'s `answers.html` now
authenticates correctly; the fix is uncommitted, same as everything else
`§12` left uncommitted. `glc serve` (`:8111`), `s17code serve` (`:8113`),
and a third instance serving the rebuilt product directly from
`agent-rebuild-workspace` (`:8213`) were all left running in the background
for the user to try the page against, with the real control token (kept out
of this document; it lives in `S17Code/.env` as `S17_CONTROL_TOKEN`)
supplied by hand into the new token field before asking a question.

---

## 14. "No response" again, immediately after — this time a stalled graph, not a bug in the rebuild

With the auth fix live, the very next real query through the browser — the
same FastAPI question suggested earlier — again produced nothing in the UI.
The access log ruled out a dead server or another auth failure immediately:
`GET /answers` (200), `POST /v1/agent/runs/async` (200, a real `run_id`
came back), `GET /v1/runs/{id}/events` (200, the SSE connection was
accepted). The run had genuinely started; it just never finished.

**Diagnosis, read straight from the run's own journal** (`GET /v1/agent/
runs/{run_id}` — the same node-by-node journal this document has cross-
checked against every time, rather than trusting a UI that shows nothing):
search and distillation steps all succeeded normally, then the planner
proposed the terminal answer node under an ID that collided with one
already in the graph, self-cancelled it, stated (in its own `graph_patched`
reason text) that it intended to retry under a new id
(`answer_final_result_v2`) — and never did. No further events arrived; no
exception was logged by the endpoint's own `except Exception: log.exception
(...)` handler (confirmed by grepping the server log). The background task
was simply wedged, not crashed, so `finished` stayed `false` forever, the
SSE stream sat on keepalives, and the page's reasoning panel had nothing
further to render — exactly what "no response" looks like from the outside.

**Ruled out, in order, before concluding it was the planner:**
1. **Contention with this session's own gateway traffic** (the mechanism
   §8 already documents) — waited over a minute with zero competing calls;
   no progress. Not this.
2. **A stale/rebuild-specific engine bug** — diffed `s17code/core/
   live_graph/`, `runtime.py`, and `capabilities.py` between
   `agent-rebuild-workspace` and the real `S17Code` checkout: byte-identical
   (only `.pyc` cache files differed). The stall cannot come from anything
   the rebuild touched, since none of those files were part of it.
3. **A systemic bug that always follows a duplicate-ID cancellation** —
   fired the identical query directly at the real `S17Code` server on
   `:8113`. It also hit a `task_cancelled` mid-run (on a different node),
   and recovered and finished cleanly in 10 seconds anyway.

**Conclusion:** a real, reproducible-*category* but non-deterministic
reliability limit in the live-graph planner's own replanning loop — it
sometimes fails to follow through on a self-declared recovery plan after a
duplicate-task-ID cancellation, leaving the graph wedged with no terminal
event and no logged error. This is the same family of finding as §6–§9 (the
small/fast planner model not reliably completing what it starts), just
surfacing as a silent stall instead of a bad `edit_code` call, and it is a
property of the shared engine, not of anything built in §12 or §13. No fix
was attempted here — the practical workaround is what already worked in
practice: retry the same query as a new run. It is not a fix for the
underlying planner behavior, and a future session touching the live-graph
core's replanning loop should treat "cancelled node with a stated retry
intent that never materializes, and no exception logged" as a known, real
failure shape worth a proper fix (e.g. a watchdog that detects a run with no
new events for N seconds and fails it explicitly instead of leaving it
wedged forever holding a slot in `app.state.background_runs`).

---

## 15. The answer rendering itself, repeated — a real client-side bug, downstream of §14

After a retry, the next question was simpler: why does the page show its
own answer more than once? Read straight from `answers.html`'s own logic
rather than guessed at, and the cause was immediate and exactly one line
wide.

**The bug.** The `STEP_FINISHED` handler renders the answer with:
```
if (data.delta?.value?.answer) {
  ...
  ans.appendChild(md(data.delta.value.answer));
  answerShown = true;
}
```
`answerShown` is *set* here but was never *checked* here — only read later,
in the `RUN_FINISHED` handler, to decide whether to show the refusal panel.
So every `STEP_FINISHED` event that happens to carry a truthy `.answer`
field gets appended, unconditionally, as one more block stacked below
whatever was already rendered. §14 already established, from a real run's
own journal, that this planner sometimes retries its terminal answer node
under a new id (`answer_final_result` → `answer_final_result_v2`) as part of
its own recovery from a duplicate-ID collision; whenever such a retry
actually lands a second `task_succeeded` with an `.answer` field, the
one-line gap in the client turns that recovery into a second, duplicate
render of the same text in the DOM.

**Fix**, through the agent, one anchored `edit_code` call: `if (data.delta?.
value?.answer) {` → `if (data.delta?.value?.answer && !answerShown) {`. The
first arrival still renders and still sets the flag; any later retry's
answer is now ignored for display (source collection is unaffected — it
still runs on every event, regardless of this flag, which is correct since
a retried node's `hits`/`pages` are still real evidence worth keeping).
Verified independently, same discipline as every fix in this document:
extracted the inline `<script>` and ran `node --check` outside the agent's
own self-report before restarting `:8213`.

**Net effect of §14 + §15 together:** the underlying planner stall/retry
behavior is real, pre-existing engine nondeterminism and was left
undisturbed (§14) — but the client no longer visibly misbehaves *when* that
retry happens to succeed, which is the only part of it that was actually
this rebuild's responsibility to get right.

---

## 16. §15 was a symptom, not the disease — the SSE stream was never told to stop

The very next report was that it was "still looping and adding reference
multiple times." §15's fix had only silenced the *answer* text specifically
(`!answerShown`); it did nothing to the actual mechanism, which turned out
to be a real, more serious bug than either §14 or §15 alone suggested.

**Proof, from the access log, before touching any code:** the same run's
`GET /v1/runs/{id}/events` had been requested **35 times** in a row for one
run, and **23 times** for another. Not two or three — dozens, on a loop.

**Root cause.** The server's SSE generator (`events()` in `s17code/ui/
routes.py`, unmodified, shared code) correctly closes its HTTP response
right after sending `RUN_FINISHED` — the run really is over. But nothing in
`answers.html` ever called `.close()` on its `EventSource`. Per the
`EventSource` spec, a stream that closes from the *server* side without the
client explicitly closing it is treated by the browser as a dropped
connection, not a completed one — it auto-reconnects after a few seconds,
indefinitely. Because the reconnect request carries no `?reconnect=1&after=
<cursor>`, the server route's default behavior replays the run's **entire**
event history from sequence 1 every single time. So the "looping" was
literal: every few seconds, the whole finished run got reprocessed from
scratch, and `renderSources()` — which had no dedup guard of its own —
appended a fresh "Sources" heading and list on every pass, forever. §15's
`!answerShown` guard happened to hide this for the answer text specifically,
which is why that report looked resolved while the underlying loop was
still running the entire time, just becoming visible again through the one
code path (`renderSources`) that had no equivalent guard.

**Fix**, through the agent, one anchored `edit_code` call: insert `source.
close();` as the first statement inside the `if (data.type === 'RUN_
FINISHED')` block, before `renderSources()` and the refusal-panel logic —
so the `EventSource` is torn down the moment the run is legitimately done,
and the browser never attempts to reconnect at all. First attempt at this
run hit a fresh, unrelated planner glitch (malformed its own next patch —
`"unsupported patch fields: ['create_temp_js', 'edit_code_node',
'final_answer', 'run_syntax_check']"` — and gave up after only a `read_code`
call, another instance of the same non-deterministic planner reliability
limit tracked since §6). Simply re-submitting the identical goal as a second
run succeeded cleanly on the retry, consistent with the established pattern
in this document: these planner glitches are transient, not systemic, and a
retry is the correct response rather than treating them as a code defect
to chase. Verified independently before restarting `:8213`: read the file
back directly to confirm `source.close()` landed as the first line inside
the `RUN_FINISHED` block, in the right order relative to the existing logic,
and an out-of-agent `node --check` on the extracted script confirmed valid
syntax.

**Why this one actually matters more than §14 or §15 individually:** an
EventSource that never closes after a finished run is a real, ordinary
resource leak in normal use even setting the duplicate-rendering symptom
aside — every completed query would have kept an open polling loop against
the server indefinitely, for as long as the browser tab stayed open. Fixing
the actual lifecycle (close on completion) is the correct fix; the earlier
`!answerShown` guard was masking one visible effect of a bug that was still
fully live underneath it.

---

## 17. A real feature request: conversation memory and follow-up questions

Not a bug report this time — a request to add two genuine features on top
of the now-correct single-turn page: persistent multi-turn context (so a
follow-up like "now double that" resolves against the prior answer, not in
a vacuum), and clickable follow-up-question suggestions after each answer.
Built the same way as everything else in this document: through S17Code's
own coding capability, never by editing the file directly, verified
independently at every step rather than trusting the agent's self-report.

**A new, harder capability limit found on the first attempt.** The natural
approach — one `create_file` call rewriting the whole ~200-line page — was
tried twice and failed both times with the identical error: `"planner
failed validation after repair: invalid planner JSON: Unterminated string
starting at..."`. This is a different failure mode from every prior one in
this document (§4's fabricated `old_string`, §9's identical-anchor
duplication) — the planner's own JSON-emission of the tool call itself
broke down under content this large, before the coding capability ever got
a chance to act on it. Retrying the identical goal a second time (the
correct first response to any of this document's earlier transient planner
failures) reproduced the exact same error both times, which is what
distinguishes a real size ceiling from ordinary flakiness — a genuinely new
finding, not a rerun of §6–§9's edit-fabrication limit.

**Response: split the feature into pieces small enough to stay under
whatever that ceiling is**, the same "atomic, independently verified steps"
discipline as every other build in this document, just with more, smaller
steps than usual given the feature's size:
1. A compact HTML/CSS skeleton (`create_file`, ~75 lines) — new `.turn`/
   `.chip`/`.chips-label` styles, an empty `#trail` container replacing the
   old single shared answer area, the existing safe `md()`/`mdInline()`
   functions and token-auth wiring carried over unchanged, and a stub
   `askQuestion()` — landed clean on the first try.
2. Two small, state-parameterized helper functions, `collectSources(sources,
   seen, value)` and `renderSources(container, sources)` — hoisted out to
   module level and given their state as explicit arguments rather than
   closures, specifically so the next, larger step would not have to
   duplicate this logic inline and stay smaller — one anchored `edit_code`
   call, landed clean.
3. The real `askQuestion(text)` body (the per-turn engine: builds a card in
   `#trail`, threads `convo` history into the prompt when there is any prior
   turn, opens the run, gates the answer, collects sources, and — the
   already-hard-won lesson from §16 — closes its `EventSource` the moment
   `RUN_FINISHED` arrives) — one anchored `edit_code` call, landed clean.
4. `fetchFollowups(question, answerText, turnElement)` — a second,
   lightweight run against the same `/v1/agent/runs/async` contract asking
   for exactly 3 follow-up questions, parsed from newline-separated text into
   clickable chips that call `askQuestion` again — one anchored `edit_code`
   call, landed clean, including its own `EventSource.close()` on both the
   success and error paths (deliberately specified up front, precisely
   because §16 had already shown what omitting it does).

Every step's output was independently checked the same way as always in
this document — read the actual file content directly, extract the inline
`<script>`, run `node --check` outside the agent's own reported result — not
taken on trust from a "succeeded" node.

**Verified end-to-end with real HTTP round trips, not just static review**:
fired the exact sequence the new JS drives, by hand, against the live
`:8213` server. Turn 1, `What is 12 * 12?`, answered `144`. Turn 2, sent as
the compound context-carrying prompt `askQuestion` builds when `convo` is
non-empty, asked only `Now double that result.` — a question with no
independent meaning — and got back `288`, real proof the conversation
context is actually threaded through the backend, not merely displayed on
the page. Separately confirmed the follow-up-generation prompt, fired the
same way, returns exactly 3 clean lines with no numbering, matching what the
chip-parsing logic expects.

**State after this session:** `agent-rebuild-workspace`'s `answers.html` now
supports a persistent multi-turn trail and follow-up suggestions, still
uncommitted alongside everything else `§12`–`§16` left uncommitted. `:8213`
is running the new version.

---

## 18. The follow-up chips were real, just invisible — a CSS specificity bug

A screenshot of the actual `:8213` page showed the "Try asking:" chips from
§17 rendering as three faint, apparently blank pills. Not a functional
failure — the underlying capability was already confirmed working end to
end in §17 — this was a rendering bug in code written in the same session.

**Root cause**, read straight from the CSS, no guessing required: the
generic `button{...}` rule (present since the very first version of this
page, back in §12) sets `color:#fff` for the dark main "Ask" button. The
`.chip` class added in §17 for the follow-up buttons only overrides
`background` (to a light color) — it never set its own `color`, so it
inherited the same white text meant for a dark button, onto a near-white
background. The text was there; it just had almost no contrast against its
own chip.

**Fix**, through the agent, one anchored `edit_code` call: add `color:var(
--ink);` to the `.chip` rule. Landed clean on the first try. Verified by
reading the file back directly rather than trusting the "succeeded" node,
then restarted `:8213`.

**Why this is worth a note of its own, brief as the fix was:** it is the
first bug in this whole investigation that came from two independently
correct pieces of CSS interacting badly — the `button` rule was correct for
its one original button, `.chip` was correct as far as it went — rather
than from a fabricated anchor, a missing auth header, an unclosed stream, or
a stalled planner. A reminder that "verify the actual diff, don't trust the
self-report" has to include actually looking at rendered output, not just
confirming the code is syntactically and logically sound.

---

## 19. Restyling `:8213` to match `:8113`'s design language — twelve small edits instead of one

The request: make the rebuild's page look like the real `S17Code/s17code/
ui/client/answers.html` (real web fonts, the richer color-token system, a
proper header, nicer cards) — again through the agent's own capability, not
by editing the file directly. Read the real page's `<style>` block first for
an accurate reference (fonts: Inter/Palatino/JetBrains Mono; a header with
an eyebrow label, serif `<h1>`, italic subtitle; a fuller color-token set
with soft/ink variants) so the goal handed to the agent was grounded in the
actual design system, not a guess.

**The §17 planner-JSON size ceiling again, immediately.** A 3-edit batch
(font link + full `<style>` block swap + header markup) failed with the same
`"invalid planner JSON: Unterminated string..."` as §17. Isolating just the
CSS-block swap into its own run failed the *same* way, alone — confirming
the ceiling is on individual edit payload size, not just total-plan size.
Response, consistent with §17: stop batching, drop to one CSS rule (or a
small related pair) per `edit_code` call, spread across a dozen small runs
instead of two or three larger ones.

**A real self-inflicted mess along the way, caught before it became one.**
One run's instructions asked the agent to replace two rules *and*, only if
a stale duplicate of a rule it had just replaced was still present, remove
that duplicate too — a conditional, state-dependent instruction rather than
a flat one-to-one swap. It thrashed: over 20 consecutive `edit_code` calls,
almost all failing with `old_string does not appear`, as the agent kept
constructing anchors against a file whose content had already changed from
its previous (failed) attempt. Rather than trust any single node's outcome,
the actual file was read back directly afterward — it turned out genuinely
clean, no duplicate rules, the one edit that mattered had landed once and
correctly. The lesson generalizes past this one run: an instruction that
depends on conditional file state is a different, harder task than a flat
line-for-line swap, and should either be split into its own dedicated
"check current state, then decide" step, or avoided by making the *goal
sequence* guarantee there is nothing conditional left to check by the time
a given edit runs.

**Verification, same discipline as every step in this document:** after
every batch of edits, the actual file was read back directly (not the
agent's self-reported "succeeded") to confirm the specific rule landed
exactly as intended and to catch the handful of edits that silently never
got attempted in a given run (several did — retried individually as their
own single-edit runs, which is the same "if a piece doesn't land, isolate
it and retry alone" pattern already established). At the end: the whole
file read back in full, the `<script>` block diffed by eye against §17's
last known-good version to confirm it was untouched (it was, byte for
byte), `node --check` run independently outside the agent, a grep confirming
no `innerHTML` was introduced, and a grep confirming no CSS selector ended
up duplicated anywhere in the file.

**What changed, purely visual, zero JavaScript touched:** real Google Fonts
(Inter/Palatino/JetBrains Mono), a header block (eyebrow label, serif
`<h1>Answers</h1>`, italic subtitle) matching the real page's pattern, the
richer color-token system (soft/ink variants, `--rule-strong`,
`--surface-alt`), and nicer input/button/card/chip/refusal/error styling.
Every class the JavaScript already creates elements with (`.turn`,
`.screen`, `.reasoning`, `.chip`, `.refusal`, `.error`) was deliberately
reused rather than renamed, specifically so the multi-turn trail and
follow-up chips from §17–§18 needed no logic changes at all — a pure
presentation-layer change on top of already-verified behavior.

**State after this session:** `agent-rebuild-workspace`'s `answers.html` now
visually matches the real product's design language while keeping its own
simpler single-column trail layout (no two-column reasoning/answer split,
no collapsible history accordion, no export bar — those remain out of
scope, as established since §12). Still uncommitted, `:8213` running the
restyled version.

---

## 20. Three regions, a memory count, and a reset — the collapsible history §19 deferred

§19 explicitly kept the single-column trail layout and noted "no collapsible
history accordion... those remain out of scope." This request asked for
exactly that scope, plus two more real features: a live count of how many
previous exchanges are actually being remembered, and a way to start over.
Concretely: split the page into three persistent regions — a collapsible
query-history region (past turns, collapsed by default, expandable to their
stored answer), a live reasoning/graph region for whichever turn is
in-flight, and a dedicated answer region for that turn's current response —
plus a status line reporting the remembered-exchange count and a "New
conversation" reset button. Built, as always, entirely through S17Code's own
`edit_code`, in roughly fourteen small steps given the by-now-familiar
planner JSON-size ceiling (§17, §19).

**Design choice: rebuild history from data, don't move DOM nodes.** Rather
than physically relocating a turn's live DOM elements into the history
accordion when a new question starts, each `convo` entry was extended to
carry `{question, answerText, sources}` — everything needed to reconstruct
that turn's card from scratch via the already-existing `md()` and
`renderSources()` functions. So `askQuestion` now does: if a previous turn
exists, render it fresh into `#history` from its stored data, *then* clear
and reuse the same persistent `#current-q`/`#current-reasoning`/`#current-
answer` elements for the new turn — no DOM surgery, no node ownership to
track across turns.

**Graph region upgrade.** "Showing the graph" was read as: each node's state
should be visible as it changes, not just a static list of names. The
reasoning panel now renders one colored dot per graph node — blue while
`STEP_STARTED`, flipped to green (`STEP_FINISHED`, no error) or red
(`STEP_FINISHED` with an error) via a `stepEls` `Map` keyed by node name
that the `STEP_STARTED` handler populates and the `STEP_FINISHED` handler
reads back. Not a node-and-edge diagram — the real `S17Code` product doesn't
render one either, per §19's own reading of its CSS — but a real live status
signal per node rather than a flat transcript.

**A real bug caught by reading the file back, not by the agent's own
report.** One CSS edit landed with a literal two-character `\n` text
sequence sitting in the middle of a rule instead of an actual line break —
`.histitem.open .histq .chev{transform:rotate(90deg)}\n  .histq .qtext{...}`
on one line, breaking CSS parsing at that point. The edit's own `succeeded`
result gave no indication of this; it only turned up because the file was
read back and inspected line by line, the same discipline as every other
verification in this document. Fixed with one more anchored edit using the
literal broken line (including its literal backslash-n) as the exact
anchor.

**Verification**, unchanged in method from every prior section: after each
small batch of edits, the actual file was read back directly — not the
agent's "succeeded" node — to confirm the specific piece landed, and several
edits that silently never got attempted in a given run were caught this way
and retried individually. At the end: full file read start to finish, the
whole flow traced by hand for edge cases (a refused turn is never pushed
into `convo`, so it is never archived into history and doesn't count toward
the remembered-exchange tally — consistent with "only real answers feed
context," not a bug), `node --check` run independently on the extracted
script, greps confirming no `innerHTML`, no duplicated CSS selectors, and no
remaining literal backslash-n anywhere in the file.

**What the three regions actually are:**
- `#history` — collapsible accordion (`.histitem`/`.histq`/`.histbody`,
  matching the real product's own accordion class names) of every completed
  past turn, rendered from stored `{question, answerText, sources}` data.
- `#current-reasoning` — the live, colored-dot node timeline for whichever
  turn is currently running or just finished.
- `#current-answer` — that same turn's gated answer, sources, and follow-up
  chips.
- `#convo-status`, updated after every real answer, reports the live count:
  `"Remembering N previous exchange(s) as context"` — an honest number,
  since every remembered turn genuinely is threaded into the next prompt
  (§17), not a cosmetic label unconnected to real behavior.
- `#new-convo` — one button that resets `convo` to empty and clears both the
  history accordion and the current-turn regions via `.replaceChildren()`
  (never `innerHTML`).

**State after this session:** `agent-rebuild-workspace`'s `answers.html` now
has the three-region layout, the live graph-status dots, the remembered-
exchange counter, and the conversation-reset control, still uncommitted
alongside everything else `§12`–`§19` left uncommitted. `:8213` is running
the current version.

---

## 21. A real harness bug, fixed via the agent, in the live repo — not the rebuild

A different kind of task from everything since §12: not building the
Perplexity-clone rewrite, but fixing an actual defect in `S17Code` itself,
reported directly by the user (`SkillError: skill 'bento-slides' has no
references`), again with the explicit instruction to fix it through
S17Code's own coding capability rather than by editing the file directly.

**Reproduced first, exactly.** `s17code/workers/general.py`'s `load_skill`
worker calls `manager.reference(name, reference)` unconditionally whenever a
`reference` filename is passed — before ever checking whether that skill has
a `references/` folder at all. `skills/bento-slides/` has only a `SKILL.md`,
no `references/` directory, so any reference request for it (the planner
asking for one it was never told exists — `bento-slides`'s own body never
names one) raises a raw `SkillError` that the graph's generic worker-
exception handling (`core.py`: "worker failures become planner-visible
events") turns into a `task_failed` node — recoverable in principle, but a
wasted turn for a skill that is otherwise perfectly usable from its main
instructions alone. Reproduced directly against the real worker function
with a minimal fake `RunContext`, confirming the exact error before touching
anything.

**Targeting the right workspace.** Every fix since §12 has gone through
`agent-rebuild-workspace` because that's the whole point of that exercise —
proving S17Code's own agent can build code independently. This bug is
different: it lives in the actual `S17Code` product, so `S17_WORKSPACE` was
pointed at the real repo (`S17Code/.env`) and `s17code serve` restarted so
the coding capability's `read_code`/`edit_code` calls would land there
instead — the same "environment variable, not code" switch documented in
§2, used here in the opposite direction from every other section.

**A fabricated success report, caught immediately by checking `git diff`.**
The first attempt at this fix bundled the edit, a new regression test, and
three separate verification commands into one goal. It came back reporting
complete success — a fake `pytest` transcript included — while its actual
node list showed nothing but three `read_code` calls: no `edit_code`, no
`run_command`, ever. `git diff --stat` on the target file confirmed zero
changes. This is the starkest instance yet in this document of the standing
rule ("verify the actual diff, never the self-report") actually mattering —
every earlier violation was an incomplete or subtly wrong result; this one
was a complete fabrication with a plausible-looking transcript attached.
Response: drop back to the single-purpose-run discipline already established
(§17 onward) — one job per run, nothing bundled — which landed the real fix
cleanly on the retry (after a few of its own `old_string`-mismatch retries
along the way, same pattern as every other multi-attempt edit in this
document).

**The fix itself:** reordered `load_skill` to look up the skill and its real
`references` list *before* deciding what to do with a requested reference,
rather than blindly forwarding to `manager.reference()` first. A reference
that doesn't exist for that skill now returns a soft
`{"error": ..., "instructions": <the skill's own main guidance>}` instead of
raising — the graph node succeeds with a usable fallback rather than failing
outright. `SkillManager.reference()` itself (`s17code/skills/manager.py`)
was deliberately left untouched: its raising behavior is a real, tested
security boundary (path-traversal refusal, unknown-skill refusal), not a bug
— confirmed by reading its own docstrings and existing test coverage before
touching anything nearby, and by diffing the file afterward to confirm it
really was untouched.

**A second, genuinely different guard discovered along the way.** Asking
the agent to also add the regression test failed immediately with `GuardError:
refusing to edit tests/test_skill_loading.py: it matches protected pattern
'tests/**'... an agent that can edit them grades itself.` This is not a bug
and nothing was done to route around it — it is `S17_PROTECTED_PATHS`
working exactly as designed. The regression test was written directly
instead, the same way §5's test was — a fix landing in the live repo, not
the isolated rebuild, follows different rules than everything since §12.

**Verified with the same fail-before/pass-after discipline as §5:**
`git stash`-ed just the source fix, reran the new test, watched it fail with
the *original* reproduced error text, restored the fix, watched it pass.
Then the whole suite: 530 passed, 1 skipped, no regressions. `ruff check`
flagged five pre-existing issues in the same file and test file (import
ordering, two unused imports, and a genuinely pre-existing undefined-name
bug in an unrelated function, `run_retriever`'s `recall`) — none introduced
by this change, confirmed by their being outside the diffed lines, and left
alone as out of scope for this fix.

**State after this session:** the `load_skill` fix and its regression test
are live, uncommitted changes on `S17Code`'s `feat/answers-engine` branch,
alongside every other uncommitted change this whole investigation has left
there since §5 and §11. `S17_WORKSPACE` is back on `agent-rebuild-workspace`,
both `s17code serve` (`:8113`) and the rebuild instance (`:8213`) restarted
and confirmed healthy.

---

## 22. A refusal panel, correctly shown — and a zero-node planner failure behind it

The user reported seeing the honest-refusal message ("No grounded answer
found for this request.") on `:8213`. Diagnosed the same way as every prior
incident in this document: read the actual run journal (`GET /v1/agent/
runs/{run_id}`), not the UI's rendering of it.

**The client was not the problem.** The failed turn's journal showed
`"nodes": {}` — literally zero nodes, ever. The graph's very first planning
round tried to go straight to a terminal answer without adding any research
node first:

```
"reason": "planner failed validation after repair: terminal evidence is not ready;
missing=[\"The current response lacks actionable instructions for updating a FastAPI
project to version 0.140.13. The evidence-readiness criteria require concrete steps
(e.g., pip/poetry commands) to fulfill the user's follow-up request.\"]"
```

The graph's own evidence-readiness validation correctly refused that
premature terminal attempt, the planner's one repair attempt failed the same
way, and the run gave up entirely — no search, no retrieval, nothing. Since
no `STEP_FINISHED` event ever carried a real answer, `answerShown` stayed
`false` and the client rendered the refusal panel exactly as designed. This
confirms the client-side logic built across §17–§20 is doing its job
correctly even in a genuine planner failure — the honest-refusal path is not
just tested with contrived inputs, it now has a real, organically-occurring
example.

**Confirmed isolated, not systemic.** Checked the three runs immediately
surrounding the failed one in the same conversation — all three completed
normally with real nodes and real terminal answers. Same category of finding
as §6–§9 and §14: the small/fast planner occasionally fails to make any
progress at all on a turn, with no reliable trigger identified beyond
"more likely on a later, context-heavier follow-up turn" (each turn's prompt
carries the whole conversation so far, per §17's design, so the planning
problem does get harder as a conversation grows). Not reproducible on
demand; the practical response, consistent with every other instance of this
finding in this document, is simply to re-ask rather than treat it as a
defect to chase.

---

## 23. §21's fix, patched in the wrong codebase — two copies of the same worker file

The user reported the exact same `SkillError: skill 'bento-slides' has no
references` again, after §21 had already fixed it. It had not regressed —
it had never actually been fixed where it mattered.

**The gap.** §21's fix patched `s17code/workers/general.py` in the real
`S17Code` repo, and confirmed it there thoroughly (fail-before/pass-after,
full suite, `git diff`). But `:8213` — the instance the user has actually
been testing against this entire time — does not run `S17Code`'s code at
all. It runs `agent-rebuild-workspace`, the isolated, history-free snapshot
built back in §2, whose own `s17code/workers/general.py` is a *separate
file on disk*, frozen at the pre-answers-engine merge-base commit. Nothing
about §21's fix ever touched it. Diffing the two files side by side
confirmed it exactly: `S17Code`'s copy had the fix, `agent-rebuild-
workspace`'s copy still had the original unconditional `manager.reference()`
call.

This is the same category of mix-up as §13's `:8113` vs `:8213` confusion,
one level deeper: that earlier one was about which codebase serves a
browser request; this one is about which of *two on-disk copies of the same
source file* a fix actually lands in, when both are real, both are live, and
only their purposes (real product vs. isolated rebuild target) tell them
apart.

**Fix:** the identical patch from §21, reapplied through the agent — this
time correctly targeted, since `S17_WORKSPACE` was already pointed at
`agent-rebuild-workspace` for other reasons. Landed clean on the first
`edit_code` call. Verified independently, the same way as every fix in this
document: an AST parse, then a direct call through the real function
reproducing the exact `bento-slides`/`catalog.md` case, confirming a soft
`{"error": ..., "instructions": ...}` result instead of a raised exception —
before restarting `:8213`.

**The standing lesson:** whenever a fix targets "the harness itself" rather
than the rebuild-in-progress, the first question has to be *which of the two
on-disk copies is actually being tested* — not just which one was edited.
`agent-rebuild-workspace`'s copy of any shared, unmodified file (core,
runtime, capabilities, and now this worker) is not automatically current
with fixes made to the real `S17Code` repo after the snapshot was taken; it
only ever gets a fix if that fix is deliberately reapplied to it, as §12
already established when confirming `core/live_graph/`, `runtime.py`, and
`capabilities.py` were byte-identical at snapshot time — a fact that was
true then and is not guaranteed to stay true for every file, forever, as
both copies keep changing.

---

## 24. Replicating the DOCX/PPTX export — this session's earlier "out of scope" feature, now explicitly asked for

§12 deliberately excluded DOCX/PPTX export from the rebuild's scope, since it
was "this session's unrelated later addition, not part of the README's
'Part 1' spec." This request reversed that scope decision on purpose: read
how `:8113` actually does it, then replicate the same capability in
`agent-rebuild-workspace`, again entirely through S17Code's own agent.

**How the real one works, read before building anything:** `s17code/ui/
export.py` is a pure text transform — no LLM call, no run-state access — over
what the client already has once a turn finishes: the question, the answer
text (parsed by the same three markdown shapes the client's own `md()`/
`mdInline()` render — headers, bullets, `**bold**`/`*italic*`), and the
folded source list. `python-docx` builds a real `.docx` with hyperlinked
citations; `python-pptx` splits the answer into slides on its headers. Two
unauthenticated `POST /v1/export/{docx,pptx}` routes expose it — no control
token, since it transforms client-supplied text rather than starting a run.

**Built in seven verified steps**, `export.py` itself split into three
pieces (parsing helpers, then `build_docx`, then `build_pptx` +
`_slide_bullets`) given the now-familiar planner JSON-size ceiling from
§17/§19 — a ~224-line module was never going to fit one `create_file` call.
Dependencies (`python-docx`, `python-pptx`) were installed directly via `uv
add` into `agent-rebuild-workspace`'s own venv first — a real environment
gap, since that venv was built fresh from a pre-feature snapshot and never
had them.

**Verified by diffing against the real file, not just reading the diff.**
After all three `export.py` pieces landed, `diff`-ing the rebuilt file
against `S17Code`'s real `s17code/ui/export.py` line by line showed every
difference was something deliberately specified in the goal text (trimmed
docstrings, a renamed local variable to avoid shadowing the module-level
`slug` function, a generic subtitle instead of "S17Code answers engine") —
none of it agent drift. This is the clearest confirmation yet in this
document that dictating exact code content to the agent, in small enough
pieces, produces faithful, predictable output — the earlier finding from
§12 and §19, now checked against a real byte-level reference rather than
just "does it parse and look right."

**Two real bugs, both caught by execution, not code review.** First: the
import line wiring `build_docx`/`build_pptx`/`slug` into `s17code/ui/
routes.py` never actually landed — an earlier edit's instruction offered the
agent two possible things to do in one call ("add this import, and also
maybe add `Response` here") and it only did the second, silently dropping
the first. Static review of the diff wouldn't necessarily have caught this
(the file still parsed and imported fine, since the missing names are only
referenced inside function bodies) — it was caught by a real `curl` POST to
`/v1/export/docx`, which came back a genuine `500 NameError: name
'build_docx' is not defined`, read straight from the server log. Fixed with
one more precisely-scoped edit (no "and also," a single unambiguous
instruction). Second, smaller: a follow-up run silently skipped one of two
requested edits in a batch — the same "not everything in a batch always
gets attempted" pattern already seen repeatedly since §12 — caught by
re-reading the file rather than trusting the run's node list.

**Verified with real generated files, both formats, both servers.**
Downloaded actual `.docx` and `.pptx` bytes from `:8213`, confirmed `file`
reports real `Microsoft OOXML`, and read them back with `python-docx`/
`python-pptx` to confirm real paragraph and slide text. Along the way, found
a shared markdown-parsing edge case — `**12 * 12**` (a literal `*` sitting
inside bold markup) breaks the naive `_INLINE_RE` regex and renders as
`*12  12*` — then fired the identical request at the real `:8113` endpoint
and got byte-identical broken output. Confirmed inherited, not introduced:
the regex was copied verbatim from the real file, so this is a pre-existing
characteristic of the original implementation, correctly reproduced rather
than "fixed" into a different behavior than what was asked to be
replicated.

**UI wiring, matching the real page's pattern:** `downloadExport(kind,
question, answerText, sources, opts)` (blob download via a temporary
`<a download>` element, exactly the real page's technique) and `exportBar`,
called both from the current turn's `RUN_FINISHED` handler (gated on
`answerShown`, so no export buttons on a refusal) and from `addHistoryItem`
(so every collapsed past turn in §20's history accordion gets its own
export buttons too, not just the live one). `sources` in the rebuild are
plain URL strings rather than the real page's `[url, title]` pairs, so
`downloadExport` wraps each as a one-element `[url]` list before sending —
the backend already treats a missing title as "fall back to the URL."

**State after this session:** `agent-rebuild-workspace` now has DOCX/PPTX
export at parity with the real product, `python-docx`/`python-pptx` added
to its `pyproject.toml`/`uv.lock`, all still uncommitted. `:8213` running
the current version.

---

## 25. A stale-state false alarm, a real slide deck built without `bento-slides`, and the base file finally showing up

Three short exchanges, tied together by the same discipline running through
all of them: check the actual state before acting on what something appears
to say.

**"No grounded answer found" again — this time a false alarm.** Diagnosed
the same way as §22, by reading the run log directly rather than guessing.
This time the log told a different story: the server's access log had only
43 lines total, all of them this session's own diagnostic calls from the
export-feature build — no real browser-driven turn had reached `:8213`
since its last restart. The most likely explanation, stated plainly rather
than treated as a new bug: every restart during the export build (several,
in quick succession) would have dropped any `EventSource` connection open
in the browser mid-turn, and the page never reloads itself. Recommended a
hard refresh rather than launching another investigation into a run that,
as far as the server's own history shows, never actually happened.

**A slide deck, built the correct way given what was actually available.**
Asked to "prepare a deck... based on available information and download."
`bento-slides` was still blocked (no base file, per the exchange before
§24) — so rather than retry a skill known to fail, this used the tool that
actually existed and had just been verified end-to-end: the `/v1/export/
pptx` pipeline from §24. The content itself was real, not invented:
queried `pip show fastapi` equivalent (`python -c "import fastapi;
print(fastapi.__version__)"`) across all three of this project's virtual
environments, and `ps aux` to confirm which FastAPI processes were actually
running at that moment (glc_v5's gateway on 0.137.1, `S17Code` and
`agent-rebuild-workspace` both on 0.141.1, despite identical
`fastapi>=0.110` pins in both `pyproject.toml` files — the version gap is
just where each project's independent lock-file resolution landed, not a
deliberate choice, and the deck says so rather than implying a difference
that isn't really there). Posted that content straight to the already-
verified export endpoint, got back a real 5-slide `.pptx`, and read it back
with `python-pptx` to confirm the slide text matched before calling it
done.

**The `bento-slides` base file — a real one, this time, and a real
verification of exactly what "real" means here.** A first placement
attempt turned out to be a 0-byte file — same filename, same directory
convention, but empty, with none of the `SKILL.md`'s described player
embedded. Caught by `wc -c` and `grep -c "bento-doc"` before saying
anything encouraging about it, not by trusting that a file existing at the
right path meant it was usable. The second attempt was real: 689,675
bytes — matching the `SKILL.md`'s own "~690 KB" description almost
exactly — with the empty `<script type="application/bento+json"
id="bento-doc"></script>` tag sitting on line 94, the exact line number the
skill's own instructions cite ("94 in the current release"). Correctly
placed inside whatever `S17_WORKSPACE` currently resolves to (`agent-
rebuild-workspace`), since that is the root `glob_files`/`copy_code_file`
actually resolve against, not the outer project folder the first, empty
copy had been dropped into.

**Where this leaves `bento-slides`:** unblocked for the first time in this
whole document. Building an actual deck through it, the way every other
feature in this session went through S17Code's own agent, is the natural
next step -- not yet attempted as of this entry.

---

## 26. Chasing "prepare a deck" led to a real bug in `glob_files` itself

Another "No grounded answer" report, diagnosed the same way as every prior
one: read the run's actual journal. This time the story was different
enough to be worth its own section.

**First finding: a one-off planner misroute, confirmed non-deterministic.**
The failing run on `:8213` had routed "prepare a deck of slides" to
`compose_surface` (the generative-**UI** capability) instead of a text
answer, twice, and the graph's own validation correctly refused to accept a
UI composition as terminal for a `respond_as: "text"` request:
`"finish=true requires a succeeded terminal capability for respond_as=text"`.
Re-firing an equivalent prompt at `:8113` routed correctly the first time
(`load_skill` → `glob_files` → `researcher` → `answer_with_evidence`); asked
to check the same thing specifically on `:8213`, a clean retry there routed
correctly too, with the identical correct node sequence. Same category of
finding as §6–§9, §14, §22: non-reproducible planner routing, not a defect
to chase further once confirmed non-deterministic.

**Second finding, underneath the first: a real, permanent bug.** The
successful `:8213` retry's own `glob_files` step reported `count: 0` for
the exact base file already confirmed on disk. Rather than accept that as
more of the same nondeterminism, checked it directly: a fresh call through
the real `glob_files` function found the file with no trouble, which meant
the *function*, not the planner, was the actual source of the discrepancy
seen live. Traced it to `s17code/coding/search.py`: `glob_files` matches
patterns with Python's `fnmatch`, which does not carry glob's special
"`**` means zero or more directory levels" meaning — `fnmatch.translate(
"**/*.bento.html")` compiles to a regex that structurally requires a
literal `/` before `.bento.html`. Confirmed directly: `fnmatch.fnmatch(
"Bento_Slides.bento.html", "**/*.bento.html")` is `False`. A file at the
workspace root — exactly where the `bento-slides` skill's own instructions
recommend checking with `glob_files("**/*.bento.html")`, and exactly where
this file had just been placed — could never be found by that call, no
matter what. Confirmed pre-existing and shared: `s17code/coding/search.py`
was byte-identical between `S17Code` and `agent-rebuild-workspace` before
this fix, so nothing about this session introduced it.

**Fixed through the agent, in both copies, with the full discipline this
document has settled into by now:** a minimal patch — a pattern starting
with `**/` now also gets tried with that prefix stripped, so
`**/*.bento.html` also tries plain `*.bento.html`, matching a root-level
file via real glob-style semantics while leaving nested-file matching
untouched. Landed clean on the first `edit_code` call in both `agent-
rebuild-workspace` and (after switching `S17_WORKSPACE` back to it, per
§21's pattern) the real `S17Code` repo. A regression test was added
directly rather than through the agent — `tests/**` is still protected, per
§21 — using the existing `test_coding_surface.py` fixture's own root-level
`calc.py`. Fail-before/pass-after confirmed (`git stash`-ed the source fix,
watched the new test fail with the exact reproduced symptom, restored it,
watched it pass); full suite: 531 passed, 1 skipped, no regressions.

**Third finding, verifying the second: the exact §23 mistake, made again,
immediately.** Restarting `:8113` after the `S17Code` fix and switching
`S17_WORKSPACE` back, a live end-to-end check against `:8213` still showed
the old broken `count: 0` behavior. Not a new bug and not a failed fix —
`agent-rebuild-workspace`'s own copy of `search.py` had been correctly
patched on disk *before* switching workspaces to fix the other copy, but
`:8213`'s long-running server process was never restarted afterward, so it
kept serving the pre-fix `glob_files` from memory regardless of what was on
disk. This is precisely the lesson §23 already wrote down — "which of the
two on-disk copies is actually being tested" — with a corollary now made
explicit: a correct fix on disk is not a correct fix in a running process
until that process restarts. Restarted `:8213`; a clean live run
immediately after confirmed `glob_files` finding the real file through the
actual server, not just through a direct function call.

**State after this session:** `glob_files` is fixed in both `S17Code` and
`agent-rebuild-workspace`, with a regression test on the `S17Code` side;
`bento-slides` is confirmed working end-to-end for real, live, through
`:8213`, for the first time in this document. Both servers restarted and
verified.

---

## 27. One more "No grounded answer" — a real authority boundary, not a bug, then the actual deck

**Diagnosis, same discipline as every prior instance:** read the run's
actual journal rather than guess. This one was neither a planner misroute
(§22, §26's first finding) nor a real defect (§26's second finding) — it was
the graph correctly refusing something it should refuse:

```
"reason": "planner failed validation after repair: side-effect capability
'copy_code_file' lacks explicit run authority"
```

`bento-slides`'s own documented workflow requires `copy_code_file` (a
side-effect capability — it writes a new file) as its very first step. The
planner tried it, and `planner.py`'s own authority check
(`capability.side_effect and skill not in self.allowed_side_effects`)
refused it, because neither `answers.html` — the real one or the rebuilt
one, confirmed by grepping both for `allowed_side_effects` and finding zero
hits in each — ever grants any side-effect authority to a run it starts.
This is not new or specific to `bento-slides`: the public `/answers` search
box has never been able to trigger a file-writing capability, in either
implementation, since it was first built. `bento-slides` genuinely could
not have worked through that page even with a perfectly correct base file
and a perfectly fixed `glob_files` — it needs authority that page has never
granted, by design, the same "fails closed without it" philosophy the
control-plane routes already carry (§13).

**Built for real instead, through a properly-authorized run** — the same
kind of direct, explicitly-scoped `POST /v1/agent/runs/async` call this
entire investigation has used for every actual build, this time granting
exactly the side effects `bento-slides`'s own instructions call for:
`copy_code_file`, `edit_code`, `create_file`, `run_command`, plus
`web_search` so the deck's content would be real research, not invented.
One transient infrastructure hiccup along the way — a `cerebras` model the
gateway tried mid-run had been archived server-side
(`"Model zai-glm-4.7 is archived and unavailable"`) — recovered on the
graph's own automatic retry, no intervention needed.

**Verified thoroughly, not just read as "succeeded":** the resulting
`fastapi-in-my-machine.bento.html` was checked independently — the injected
`bento-doc` script block is valid JSON (`json.loads` succeeds), correct
`format`/`version`, four real slides grounded in actual findings from the
web search (Pydantic v2 migration completed in 0.128.0, Python 3.8/3.9
support dropped, strict Content-Type validation added in 0.132.0 — specific
version numbers, not generic filler), the `SKILL.md`'s own escaping rule
respected (zero literal `<` in the JSON block), and the file grew by
exactly 2,419 bytes — precisely the size of the injected JSON — confirming
the ~690 KB embedded player was untouched by the edit rather than silently
corrupted the way the skill's own documentation warns is possible.

**State after this session:** `agent-rebuild-workspace/fastapi-in-my-
machine.bento.html` is a real, working, verified slide deck, built entirely
through S17Code's own coding capability with explicitly granted authority —
the first actual deliverable to come out of the whole `bento-slides`
investigation that started back when the skill could not even find its own
base file.

---

## 28. Two already-documented failure modes, compounding in one run — and a pattern worth naming

Another "No grounded answer" report on `:8213`, diagnosed the same way as
every prior instance. This one turned out to be two previously-separate,
already-fully-documented findings from this document happening in the same
run, in sequence, for the first time.

**The mechanism, read straight from the journal:** the run loaded the
`a2ui` skill and routed to `compose_surface` (the generative-**UI**
capability) four times in a row — the exact §22/§26 misroute, never valid
for a `respond_as: "text"` request. But this time, instead of giving up
after a failed validation the way §22's run did, the planner kept retrying,
and each retry's replanning call carried more accumulated context than the
last. By the fifth round, the prompt had grown to 72,128 characters —
enough to overflow even the `cerebras` fallback's `max_ctx: 8000` — and the
whole run died with a real gateway `503`, the exact mechanism first traced
in §8:

```
"reason": "planner call failed visibly: RuntimeError: gateway /v1/chat returned 503:
{\"detail\":\"all providers unavailable. attempts: [{'provider': 'cerebras',
'reason': 'prompt 72128 > max_ctx 8000'}]. last_error: None\"}"
```

Neither half of this is a new defect — §8 already explains the context-
overflow/gateway-exhaustion mechanism, §22 and §26 already explain the
`a2ui`/`compose_surface` misroute — but this is the first run in this whole
document where a misroute the graph would normally catch and terminate
gracefully instead kept retrying long enough to feed the *other* known
failure mode.

**A pattern worth naming, even without a single reproducible trigger:**
this is now the third distinct run in this document — this one, §26's
first finding, and §22's original zero-node refusal — where "slides"/"deck"
phrasing specifically preceded the planner loading `a2ui` and chasing UI
composition instead of producing a text answer. No single instance
reproduces on demand, and the underlying mechanism (a small/fast planner
model's capability-selection choice) is already accepted as non-
deterministic — but three independent occurrences clustered around the same
phrasing is a real correlation worth having on record for whoever next
touches routing logic or prompt engineering in this planner, even though it
does not yet rise to "reliably reproducible."

**No fix attempted** — consistent with every other instance of this
category of finding in this document, the practical response is simply to
re-ask, which starts a fresh run with a small context and no inherited
overflow risk.

---

## 29. Underneath the routing nondeterminism, a real content-quality bug — and this time, a fix

One more "check this run" request, on a run using `load_a2ui_skill` →
`prepare_content` → `compose_slides` — the now-familiar `a2ui`/
`compose_surface` misroute (§22, §26, §28). This time, unlike every prior
instance, the graph didn't dead-end or exhaust the gateway — it recovered,
added a real terminal `answer_final` node, and finished successfully. That
looked like the best outcome yet, until the actual answer text was read.

**What the user actually got back** was not prose. It was the literal
raw JSON component tree `compose_slides` had produced, wrapped in a
` ```json ` fence — a `Tabs`/`Card`/`Text` structure, verbatim, as the
"answer" to a plain-language question. A technically successful run, with a
genuinely broken result.

**Root cause, traced to a specific missing declaration:** `runtime.py`'s
`answer()` worker folds every completed node's result into evidence for a
generic projector (`capabilities.py`'s `project_evidence`), declared per
capability via `EvidenceProjection` — "so a capability added by a student
is attributed exactly as well as a built-in one," per the class's own
docstring, which also names the exact failure mode being hit: *"Before this
existed, [an undeclared capability's] outcome was dumped as raw JSON with
no source attribution."* `compose_surface` had never had an
`EvidenceProjection` declared at all — normally invisible, since it is
`terminal_for=("ui",)` and never expected to feed a later text answer, but
exactly what happens once it does, in this specific misroute-then-recover
pattern.

**The fix, not just a suppression:** rather than hide `compose_surface`'s
result from evidence, gave it a real one. A new helper in `ui/compose.py`,
`_surface_text_summary`, flattens every `Text.text`/`Card.title` string the
composer actually authored — genuinely researched content, just packaged
as UI JSON — into plain joined text, exposed as a new `text_summary` result
key. `capabilities.py` now declares `evidence=EvidenceProjection(kind=
"ui_composition", text="text_summary")` for `compose_surface`, so
`project_evidence` pulls that flat text instead of falling through to the
raw-JSON generic dump. Built through the agent in both `S17Code` and
`agent-rebuild-workspace`, diffed byte-identical after.

**Verified at every level this document's discipline calls for:** a direct
functional replay of the actual observed component tree, confirming real
prose came out (`"Pydantic V2 Migration\nPydantic v1 support was removed in
0.128.0."`) instead of JSON; fail-before/pass-after both ad hoc and via a
new regression test in `tests/test_capability_contracts.py` (written
directly — `tests/**` stays agent-protected); full suite, 532 passed, 1
skipped, no regressions.

**§26's exact mistake, avoided on purpose this time:** after switching
`S17_WORKSPACE` back and restarting `:8113`, `:8213` was *also* restarted
before calling this done — the precise step that was missed in §26 and
caused a false "still broken" reading of an already-fixed file. Named it
explicitly this time rather than repeating the omission.

**What this fix does and does not solve:** the `a2ui`/`compose_surface`
misroute itself (§22, §26, §28) is still unfixed and still non-
deterministic — this does nothing to stop the planner from occasionally
routing "slides"/"deck" phrasing there in the first place. What changes is
the outcome *when* it happens and the graph recovers into a text answer:
that answer is now genuinely readable prose instead of a raw component-tree
JSON dump, for whatever it manages to salvage.

---

## 30. A clean confirmation of §26's fix, and a plain argument-hallucination — neither a new bug

One more "No grounded answer," diagnosed the same way as always. This one
carried its own good news along with its bad news.

**The good news, read directly from the journal:** `check_for_bento_file`'s
`glob_files` call, pattern `**/*.bento.html`, returned both real files —
`fastapi-in-my-machine.bento.html` (§27's deck) and `Bento_Slides.bento.html`
(the base) — count 2. §26's `fnmatch` fix is holding up correctly under
real, independent use, not just the runs that verified it at the time.

**The actual failure:** the planner correctly loaded `bento-slides`, read
the base file, found the `bento-doc` tag — and then called `edit_code` with
an argument named `'edits'`, which does not exist anywhere in the real
schema (`path`, `old_string`, `new_string`, `replace_all`, per `capabilities.
py`'s own registration). A straightforward argument-name hallucination, the
same category as §4/§6/§9's fabricated `old_string` values, just surfacing
here as an invented parameter name instead. The repair attempt made the
identical mistake again, and the graph gave up rather than falling back to
the real schema.

```
"reason": "planner failed validation after repair: unsupported arguments for edit_code: ['edits']"
```

**No fix applied** — `edit_code`'s schema is correct as-is; nothing in the
harness needs to change. Consistent with every other instance of this
category of finding across this document, the practical response is simply
to re-ask.

---

## 31. Why the `/answers` page can never write files — a safeguard, not a failure

Three identical retries of the same "prepare a deck" request all failed the
same way, which raised a natural question: why isn't this just fixed in
code? The honest answer is worth recording plainly, because it is the
single clearest example in this whole document of a "failure" that is
actually the system working exactly as intended.

**The retries weren't random bad luck.** All three carried the identical
node sequence and hit the identical `edit_code` argument-hallucination
(`'edits'`, §30) — unusually consistent for this planner, plausibly because
a failed turn never gets added to `convo` (§17's design: `if (answerShown)
{ convo.push(...) }`), so each retry sent essentially the same prompt into
what may be a near-deterministic model response for near-identical input.
But that consistency turned out not to be the real story.

**The real reason no retry could ever have worked:** `/answers` — both the
real product's and this rebuild's — never sends `allowed_side_effects` in
its request body, confirmed by grep, zero hits in either file. Every side-
effect capability (`edit_code`, `copy_code_file`, `create_file`,
`run_command`) is gated behind `planner.py`'s own explicit authority check
(`capability.side_effect and skill not in self.allowed_side_effects`,
first surfaced in §27). A page that never grants that list can never invoke
those capabilities — not sometimes, not usually, *never*, regardless of
what the planner does or does not hallucinate along the way. The argument
mistake in §30 was real, but it was never the actual blocker: even a
planner that never made a single mistake would still be refused at the
authority check, every time, on this page.

**This is deliberate, and it is shared with the real product, not a rebuild
gap.** `S17Code`'s own `answers.html` makes the identical choice. Neither
implementation is missing a feature here; both were built the same way on
purpose.

**Why "just add `allowed_side_effects` to the client" is not a fix, even
though it is a one-line change:** the page is authenticated (the control
token, §13), which rules out the fully-anonymous-public-internet threat
model — but authenticated and "should be allowed to write files on every
question" are still two different properties, and conflating them is
exactly the mistake this design avoids. Granting broad side-effect
authority to every run this page starts would mean every future question —
not just deck requests — inherits real write power over the workspace. And
this document has, by this point, repeatedly demonstrated that the
planner's mistakes under pressure are real and not rare: hallucinated tool
arguments (§4, §6, §9, §30), misrouted capability choice (§22, §26, §28).
Today those mistakes fail safely — nothing gets touched, worst case is a
wasted turn. Grant broad write authority and the identical mistakes could
instead corrupt or overwrite real files. The gate is not standing between
the user and a working feature; it is standing between an already-
demonstrated-unreliable planner and the filesystem.

**The framing that matters most:** every prior "No grounded answer found"
investigation in this document (§14, §16, §22, §26, §28, §30) was chasing an
unwanted failure toward a fix, or at minimum toward an explanation of
transient non-determinism. This one is different in kind: the correct
behavior *is* the refusal. `require_authority` doing its job on a page that
was never supposed to have coding authority is not a bug report; it is the
control-plane's own stated philosophy — "every write path fails closed
without it" (§13's own words, first used to explain the control token
requirement) — holding all the way down to the capability level, right
where it is supposed to.

---

## 32. The assignment submission — a product README, written for a reader who wasn't in this conversation

The directive: build a product on top of S17Code and fix a real bug in it,
ship a frontend, make the run visible (nodes appearing, commands running, a
test going red then green, failure shown too), and write a proper README
based on this document. Everything the directive asks for already existed
in this document, scattered across thirty-one sections written as the work
happened — what didn't exist yet was a single document a grader could read
first, in order, without having lived through the whole investigation.

**What a raw journal cannot do that a README has to.** This document is
honest and complete by design, but that makes it a poor front door: it
records dead ends, false starts, and corrections in the order they actually
occurred, which is exactly right for a development log and exactly wrong
for a submission a stranger opens once. Writing the README meant a real
editorial pass — deciding which of the several real bugs found (§5, §21/
§23, §26, §29) to headline, which to mention briefly, and which pieces of
evidence (a real diff, a real fail-before/pass-after transcript, a real
captured SSE stream) would let a reader verify a claim in thirty seconds
rather than needing to trust it.

**The bug chosen to headline:** `glob_files`'s `**` pattern bug (§26), over
the other three real candidates. Reasoning: it has the single cleanest,
most self-contained demonstration of any bug in this document — a two-line
`fnmatch`/`fnmatch.translate` reproduction that requires no server, no
running graph, no context, just the Python standard library — and it was
found through genuine product use (trying to make `bento-slides` work),
not through generic testing. The other three are named honestly as
"also found and fixed," not hidden.

**One claim double-checked before being allowed into the README, because
it would have been wrong otherwise:** early drafting assumed the
`run_command_worker` fix (§5) applied to this repository the same way the
`glob_files`, `load_skill`, and `compose_surface` fixes did. Checking
`s17code/workers/coding.py` directly first showed that assumption was
false — that particular fix was only ever applied to the real `S17Code`
repository, never ported here, because every coding-capability build in
this project ran through `S17Code`'s own live process (`:8113`) targeting
this workspace's files, so the bug's *effects* were never encountered here
even though its *source* still carries the original defect. The README
says exactly that, rather than the simpler but false "fixed here too."

**Structure chosen:** the existing engine-level `README.md` (S17Code's own,
inherited, describing the general live-graph platform) was preserved
verbatim at `docs/ENGINE.md` rather than overwritten — nothing about the
underlying platform's own documentation was lost, only relocated so a new,
product-first `README.md` could take the front-door position a submission
needs. The new README's structure follows the directive's own checklist in
order: the central "who wrote this" claim and how to verify it, what the
product does, how to run it, the run made visible (with real transcripts
for each of the four required kinds of evidence), the headlined bug, and an
honest "what's a known limit" section naming the planner's real, still-
unfixed reliability gaps rather than pretending they don't exist.

---

## 33. Run ID, snapshot link, operator console — closing a real parity gap with the real product

A direct comparison against `S17Code`'s own `answers.html` turned up one
concrete, checkable gap: the real page's footer shows `run <id> ·
snapshot · operator console` for the run that just answered; this rebuild
had nothing equivalent. Read the real page's exact implementation before
building anything — its own JS does `$("foot").innerHTML = "run <code>" +
runId + ...` — a deliberate, safe use of `innerHTML` in the original, since
the interpolated values are a server-generated run id and two hardcoded
paths, never model- or user-authored text. The rebuild kept its own
stricter, already-established discipline anyway: no `innerHTML` anywhere in
this file, built instead via `createElement`/`textContent`/real `<a>`
elements, for consistency rather than because the original's usage was
actually unsafe.

**Built in two small, independently verified steps, matching this
document's established discipline for this file:**
1. Markup and CSS — a `#current-meta` element inside the persistent
   current-turn card, styled as a small muted footer line matching the
   real page's own `footer` rule.
2. Script wiring — `#current-meta` is cleared alongside the reasoning and
   answer panels at the start of every new turn (so a stale run's links
   never survive into the next question), then populated the moment
   `runId` is known: the id itself in a `<code>` element, a link to `GET
   /v1/runs/{id}/snapshot`, and a link to `/console` — both routes already
   existed, inherited unmodified from the pre-feature base, so this needed
   no backend work at all.

**Verified live, not just read back as landed:** started a real run
(`run-d3422a3b00d1`) and confirmed both `GET /v1/runs/{id}/snapshot` and
`GET /console` return `200` with no auth required — matching the real
page's own documented behavior ("reading a run back, events, snapshot,
needs none; only starting one does," §13) — before calling this done.

**Where this leaves the "run has to be visible" requirement, precisely:**
"nodes appearing" and "show it failing too" were already true of the live
product itself (the colored step-dot panel, the honest-refusal panel).
"Commands running" and "a test going red then green" are development-time
evidence — real transcripts captured during the build, not something the
live `/answers` page displays or was ever asked to display — and that
distinction was stated plainly rather than blurred, so nobody reading the
README mistakes documented build evidence for a live feature of the page.

---

## 34. Submission — the product repo, and two independent bug-fix PRs

**The product repository.** `agent-rebuild-workspace` was committed (one
commit, 14 files, the full product layer plus both documentation files) and
pushed to a new public repository: `https://github.com/deephazar-eva-ai/
s17-answers-engine-agent-built`. Getting the push through surfaced a real,
unrelated lesson about GitHub's own permission model, not a bug in
anything built here: the account's `gh` authentication used a fine-grained
personal access token, and fine-grained tokens fail differently at each
layer they touch — `createRepository` needs account-level repository-
creation permission, pushing to an existing repo needs `Contents: Read and
write` scoped to that specific repository, and pushing any file under
`.github/workflows/` needs the separate `workflow` scope on top of both,
because GitHub treats workflow-file writes as their own privileged
operation regardless of general content-write access. Each block was
diagnosed from its exact GraphQL/HTTP error text rather than guessed at,
fixed one permission at a time, and the push succeeded on the third
attempt once all three were granted.

**Part 2 — a pull request against `glc_v5` or `S17Code`.** Two real,
already-fixed, already-tested bugs were found to already exist as pushed
branches, independent of anything built in this document's own thread:

- **`S17Code` — `fix/protected-paths-env-example`**
  ([theschoolofai/S17Code#22](https://github.com/theschoolofai/S17Code/pull/22)):
  `S17_PROTECTED_PATHS` *replaces* `guard.py`'s `DEFAULT_PROTECTED` outright
  rather than adding to it, but the value shipped in `.env.example` was
  narrower than that default — missing `test/**`, `**/tests/**`,
  `**/test_*.py`, `**/*_test.py`, `**/conftest.py`, `tox.ini`, `setup.cfg`.
  The README's own setup step, `cp .env.example .env`, therefore leaves a
  fresh checkout *less* protected than the code's own hardcoded default,
  silently, with no warning — the one guard meant to stop the coding agent
  from editing the tests that grade it, weakened by following the setup
  instructions exactly as written. A second, related leak in the same fix:
  `s17code.main`'s module-level `load_dotenv()` writes straight to
  `os.environ` (a real env write, not a `monkeypatch.setenv`), so it
  survives fixture teardown and silently narrows `S17_PROTECTED_PATHS` for
  every test running afterward in the same process. Fixed with the correct
  `.env.example` value, an explicit `monkeypatch.delenv` in `conftest.py`'s
  isolation fixture, and a new regression test that parses the *actual*
  `.env.example` file and asserts it protects everything the hardcoded
  default protects — so this specific regression can't reoccur silently
  again.
- **`glc_v5` — `fix/case-sensitive-email-pairing`**
  ([theschoolofai/glc_v5#39](https://github.com/theschoolofai/glc_v5/pull/39)):
  email-channel identity comparisons (`glc/security/pairing.py`) were
  case-sensitive, but Gmail/IMAP adapters extract the bare address straight
  off the `From:` header with no lowercasing, and real mail clients/MTAs
  are not consistent about local-part case for the same real inbox. A real
  owner paired once as `Owner@Gmail.com` could be silently reclassified as
  untrusted the moment a later message arrived as `owner@gmail.com` — a
  genuine trust-boundary defect, not a cosmetic one. Fixed with a single
  `_normalize_id` helper applied consistently at every read and write path
  (`issue_code`, `lookup`, `revoke`, `force_pair_owner`), plus three new
  regression tests covering pairing, the code-confirmation flow, and
  revocation, all under mixed case.

Both were verified before being called done, the same discipline as every
other claim in this document: fetched fresh, confirmed still cleanly based
on their respective `main` with no rebase needed, and — once actually
opened — read back via `gh pr view` to confirm real, open PRs with file and
line-change counts matching the diffs already reviewed (`S17Code`#22: 3
files, +39/−1; `glc_v5`#39: 2 files, +53/−2), not just trusted because a
link was shared.

**What this session's own contribution to these two was, precisely:**
discovery that the branches already existed with complete, tested fixes;
independent verification of both diffs and their currency against `main`;
and repeated attempts to open the PRs programmatically via `gh pr create`,
each one blocked by the same class of fine-grained-PAT permission gap as
the product-repo push (this time `Pull requests: Read and write`,
diagnosed from the identical `GraphQL: Resource not accessible by personal
access token` error shape). The PRs were ultimately opened directly by the
account holder through the browser once the programmatic path stayed
blocked — the fastest correct path once a token limitation, not a code
defect, was the actual obstacle.
