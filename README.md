# S17 Answers

A Perplexity-shaped answer engine — search box, live per-node reasoning panel,
gated grounded answer, numbered clickable sources, multi-turn conversation
memory, follow-up suggestions, DOCX/PPTX export — built **on top of S17Code**,
where the engine is S17Code's own live-graph agent and the frontend/product
layer was written by that same agent, not by an external coding assistant
editing files directly.

The full, unabridged, run-by-run record of how this was built — every
`edit_code` call, every real bug hit and fixed, every failed attempt shown
honestly rather than edited out — is in
[`docs/myowncodeworking.md`](docs/myowncodeworking.md). This README is the
short version.

---

## 1. The central claim, and how to check it yourself

Every line of the product layer in this repository — `s17code/routes.py`'s
`/v1/agent/runs/async` endpoint, `s17code/ui/routes.py`'s `/answers` route
and DOCX/PPTX export endpoints, `s17code/ui/export.py`, and
`s17code/ui/client/answers.html` — was written by **S17Code's own
`read_code`/`edit_code`/`create_file`/`run_command` coding capability**,
driven through its real HTTP API (`POST /v1/agent/runs/async` against a
separately-running S17Code process), never by hand-editing these files
directly.

To make that claim checkable rather than just asserted, this repository was
built as an isolated, history-free workspace *before any of that work
started*:

```
$ git log --all --oneline
e1762be base: pre-answers-engine state (from S17Code@15ab9c7)

$ git branch -a
* master

$ git remote -v
(none)
```

One commit. No other branches. No remotes. The commit is `git archive`'d
from the real `S17Code` repository's merge-base commit — the last commit
*before* the answers-engine feature existed anywhere. The coding agent that
built everything in this repo could not have discovered or copied the
original, human-authored implementation; there was nothing here to find.

```
$ git status --short
 M pyproject.toml
 M s17code/capabilities.py
 M s17code/coding/search.py
 M s17code/main.py
 M s17code/routes.py
 M s17code/ui/compose.py
 M s17code/ui/routes.py
 M s17code/workers/general.py
 M uv.lock
?? s17code/ui/client/answers.html
?? s17code/ui/export.py
```

That diff — modified files plus two new ones — is the literal, complete
output of every coding-agent run this project used. Nothing here was typed
by hand except this README and the workspace's own `.env`.

---

## 2. What the product does

Open `http://localhost:8213/answers`:

- **Search box** — ask anything.
- **Live reasoning / graph panel** — one colored dot per graph node as it
  runs: blue while in flight, green on success, red on failure. Not a
  static log line — an actual reflection of the live graph's own event
  stream (`GET /v1/runs/{id}/events`, Server-Sent Events).
- **Gated answer panel** — only renders once a real `answer_with_evidence`
  result exists for that turn. Rendered through a hand-rolled, DOM-only
  markdown renderer (`createElement`/`textContent`, never `innerHTML`
  anywhere in the file) so nothing the model writes can execute as markup.
- **Numbered sources** — folded from every node's `hits[]`/`pages[]` result
  fields as they arrive, deduplicated, rendered as real links.
- **Honest refusal** — if a run finishes with no real answer, the page says
  so plainly (amber panel) instead of fabricating one.
- **Multi-turn conversation** — a collapsible history of past turns, a live
  counter ("Remembering N previous exchanges as context"), and a **New
  conversation** reset button. Each new question is sent with the full
  prior Q&A folded into the prompt, so a bare follow-up like *"now double
  that"* resolves against the previous answer.
- **Follow-up suggestions** — after a real answer, a second lightweight run
  asks the same engine for 3 follow-up questions, rendered as clickable
  chips.
- **DOCX / PPTX export** — turns the rendered answer into a real Office
  file server-side (`python-docx` / `python-pptx`), not a print-to-PDF
  hack.
- **Run metadata** — every answer is followed by `run <id> · snapshot ·
  operator console`: the real run id, a link to `GET /v1/runs/{id}/
  snapshot` (the complete raw graph state), and a link to `/console` (the
  read-only operator page — the durable event history, reconnect-with-
  `after`, the liveness beat). Matches the real product's own pattern,
  built with safe DOM construction rather than its `innerHTML` shortcut.

Every one of these is backed by a real, working `POST`/`GET` route — none
of it is client-side mock data.

---

## 3. Run it

Three processes, in order:

```bash
# 1. the gateway (provider keys, rate limiting)
cd ../glc_v5 && uv run glc serve            # :8111

# 2. S17Code itself — the orchestrating agent that will WRITE and RUN code
#    in this workspace (S17_WORKSPACE must point here)
cd ../S17Code && uv run s17code serve       # :8113

# 3. THIS product, served from its own code
cd . && uv run uvicorn s17code.main:app --host 127.0.0.1 --port 8213
```

Then open `http://localhost:8213/answers`. `POST /v1/agent/runs/async` is a
control-plane route and fails closed without `S17_CONTROL_TOKEN` (see the
token field on the page, or `.env`) — deliberately; see §6.

---

## 4. The run, made visible

### 4a. Nodes appearing, live

A real event stream from a real run (`GET /v1/runs/{run_id}/events`),
captured verbatim while verifying the `/runs/async` endpoint the agent had
just built:

```
id: 1
data: {"type": "RUN_STARTED", "seq": 1, "source_kind": "run_started"}

id: 2
data: {"type": "STATE_DELTA", "seq": 2, "source_kind": "graph_patched", "delta": {"op": "graph_patched", "reason": "model proposed the next useful frontier", "trigger": 1}}

id: 3
data: {"type": "STEP_STARTED", "seq": 3, "source_kind": "task_started", "stepName": "calculate_sum"}

id: 4
data: {"type": "STEP_FINISHED", "seq": 4, "source_kind": "task_succeeded", "stepName": "calculate_sum", "delta": {"op": "add", "path": "/results/calculate_sum", "value": {"expression": "2+2", "result": 4}}}

id: 6
data: {"type": "STEP_STARTED", "seq": 6, "source_kind": "task_started", "stepName": "answer_final"}

id: 7
data: {"type": "STEP_FINISHED", "seq": 7, "source_kind": "task_succeeded", "stepName": "answer_final", "delta": {"op": "add", "path": "/results/answer_final", "value": {"answer": "The result of 2+2 is 4 (source: graph://run-.../calculate_sum).", "provider": "gemini_2"}}}

data: {"type": "RUN_FINISHED", "seq": 9, "source_kind": "derived"}
```

This is exactly what the answers-page client parses to drive the colored
step dots in §2.

### 4b. Commands actually running

A real `edit_code` call, its exact result, cross-checked against the
resulting file — not the agent's self-reported summary:

```
edit_general_py_final  edit_code  succeeded  {'occurrences_found': 1, 'replaced': 1}
```

```diff
--- a/s17code/workers/general.py
+++ b/s17code/workers/general.py
@@ -198,14 +198,18 @@ async def load_skill(ctx: RunContext, task: TaskSpec) -> dict[str, Any]:
         raise RuntimeError("no skills are configured; set S17_SKILLS_DIR")
     name = str(task.input["name"]).strip()
     reference = (task.input.get("reference") or "").strip()
-    if reference:
-        return {"skill": name, "reference": reference,
-                "instructions": manager.reference(name, reference)}
     skill = manager.get(name)
     if skill is None or not skill.enabled:
         available = [row["name"] for row in manager.listing()]
         raise RuntimeError(f"no enabled skill called {name!r}; available: {available}")
     refs = manager.references(name)
+    if reference:
+        if reference not in refs:
+            return {"skill": name, "reference": reference,
+                    "error": f"{reference!r} is not one of this skill's reference files: {refs}",
+                    "description": skill.description, "instructions": skill.instructions}
+        return {"skill": name, "reference": reference,
+                "instructions": manager.reference(name, reference)}
     return {"skill": name, "description": skill.description,
```

### 4c. A test going red, then green

The exact fail-before/pass-after sequence used for the bug in §5 below —
`git stash`-ing the fix, running the new regression test against the
*original* code, watching it fail with the real reproduced symptom, then
restoring the fix:

```
$ git stash push -- s17code/coding/search.py
$ pytest tests/test_coding_surface.py::test_a_double_star_glob_still_matches_a_root_level_file -q

    def test_a_double_star_glob_still_matches_a_root_level_file(repo) -> None:
        result = glob_files(repo, "**/*.py")
>       assert "calc.py" in result["files"]
E       AssertionError: assert 'calc.py' in ['tests/test_calc.py']

tests/test_coding_surface.py:164: AssertionError
1 failed in 0.03s

$ git stash pop
$ pytest tests/test_coding_surface.py -q
..............................                                           [100%]
30 passed in 0.29s

$ pytest -q          # the whole suite
531 passed, 1 skipped, 1 warning in 57.59s
```

### 4d. Show it failing too

Not every run succeeds, and this page doesn't hide that. A real run's
journal, where the planner tried to answer before it had gathered any
evidence, and the graph's own validation correctly refused:

```json
{
  "nodes": {},
  "events": [
    {"sequence": 1, "kind": "run_started"},
    {"sequence": 2, "kind": "graph_patched",
     "payload": {"finish": true, "reason":
       "planner failed validation after repair: terminal evidence is not ready; missing=[\"The current response lacks actionable instructions...\"]"}}
  ]
}
```

Zero nodes ever ran. No search, no retrieval — the graph refused to
fabricate an answer, and the client renders exactly that: an amber
"No grounded answer found for this request." panel, not a made-up response.
(Full diagnosis: `docs/myowncodeworking.md` §22.)

---

## 5. A real bug, found and fixed

**`glob_files` could never find a file sitting at the workspace root.**

Discovered while actually trying to use this project's `bento-slides`
skill — its own documentation instructs the agent to check for its base
template with `glob_files("**/*.bento.html")`. That call kept returning
empty, even after the file was confirmed present on disk.

Root cause, once traced: `s17code/coding/search.py`'s `glob_files` matches
patterns with Python's `fnmatch`, which does **not** give `**` the "zero or
more directory levels" meaning a real glob gives it:

```
$ python3 -c "
import fnmatch
print(fnmatch.fnmatch('Bento_Slides.bento.html', '**/*.bento.html'))
print(fnmatch.translate('**/*.bento.html'))
"
False
(?s:(?>.*?/).*\.bento\.html)\z
```

The compiled regex structurally *requires* a literal `/` before
`.bento.html`. A file at the workspace root — exactly where the skill's own
instructions recommend checking, and exactly where the base template was
placed — could never match, no matter what. This wasn't specific to this
rebuild: `s17code/coding/search.py` was byte-identical to the real
`S17Code` repository before this fix, so it's a genuine, pre-existing
defect in the shared harness, not something introduced here.

**The fix** (`s17code/coding/search.py`):

```diff
-        if fnmatch.fnmatch(rel, pattern) or fnmatch.fnmatch(rel.rsplit("/", 1)[-1], pattern):
+        root_pattern = pattern[3:] if pattern.startswith("**/") else None
+        if (fnmatch.fnmatch(rel, pattern) or fnmatch.fnmatch(rel.rsplit("/", 1)[-1], pattern)
+                or (root_pattern and fnmatch.fnmatch(rel, root_pattern))):
```

A pattern starting with `**/` now also gets tried with that prefix
stripped, so `**/*.bento.html` also tries plain `*.bento.html` — matching a
root-level file via real glob-style semantics — while nested-file matching
is untouched.

Applied through S17Code's own `edit_code` capability, in both this
repository and the real `S17Code` repository (they had been byte-identical
before the fix, confirmed identical again after). Verified: fail-before/
pass-after (§4c above), the regression test added directly (`tests/**` is
itself protected from agent edits by the harness's own guard — an agent
that could edit its own tests would be grading itself), full suite green,
and finally confirmed *live*, through a real run, that `glob_files` now
finds the base file end-to-end.

### Other real bugs found and fixed along the way

- **`run_command_worker` returned a non-JSON-serializable dataclass**
  instead of calling its own `.as_dict()` — every `run_command` capability
  call in a live graph crashed with `TypeError: Object of type CommandResult
  is not JSON serializable`. One-line fix, in the real `S17Code` repository
  (this repo's own copy of that file still carries the original bug — it
  was never re-patched here, since every coding-capability build in this
  project ran through `S17Code`'s own process, which already had the fix).
- **`load_skill` raised instead of degrading gracefully** when a requested
  reference file didn't exist for a skill, turning a recoverable planner
  guess into a hard, wasted graph-node failure. Fixed to return a soft
  `{"error": ..., "instructions": <the skill's main guidance>}` fallback
  instead (diff in §4b above).
- **`compose_surface` had no evidence projection declared**, so if its
  result (a UI component tree) ever needed to become text evidence for a
  later answer, the generic fallback dumped the *entire raw JSON tree* as
  evidence — and the answering model just echoed it back verbatim as its
  "answer" instead of writing prose. Fixed by giving it a real projection:
  a new helper flattens every `Text`/`Card.title` string the composer
  actually authored into plain text, exposed as `text_summary` and wired
  into a declared `EvidenceProjection`.

Full diagnosis, fail-before/pass-after evidence, and verification detail
for every one of these is in `docs/myowncodeworking.md` (§5, §21/§23,
§26, §29).

---

## 6. What's real, and what's a known, honestly-reported limit

This project's discipline throughout was: verify the actual diff, the
actual file, the actual run journal — never trust a self-reported
"succeeded." That surfaced real limits worth stating plainly rather than
hiding:

- **The planner (a small, fast model) is not perfectly reliable.** It
  occasionally hallucinates a tool argument that doesn't exist, misroutes
  "slides"/"deck" phrasing into a UI-composition capability instead of a
  text answer, or simply fails to make progress on a turn. None of this is
  reproducible on demand — retrying usually succeeds. Every instance
  encountered is logged, not edited out, in `docs/myowncodeworking.md`.
- **`/answers` deliberately has zero file-write authority**, by design,
  shared with the real product — every side-effect capability
  (`edit_code`, `copy_code_file`, `create_file`, `run_command`) is gated
  behind an explicit `allowed_side_effects` grant this page never sends.
  This is not a bug: it is the same "fails closed without it" philosophy
  the control plane already enforces everywhere else, holding at the
  capability level too. Building or editing a real file (like the working
  `Bento_Slides` deck in this repo) requires a properly-authorized call —
  see §12/§27 of the full log for exactly how that was done.
- **Large generated content hits a real planner JSON-emission ceiling** —
  a single `create_file`/`edit_code` call carrying roughly 150–200+ lines
  of new content reliably fails with `invalid planner JSON: Unterminated
  string...`. The practical workaround, used throughout this build: split
  large generations into several smaller, independently-verified calls.

---

## 7. Full development log

`docs/myowncodeworking.md` is the complete, unedited record: every
diagnostic step, every failed attempt, every real bug, every fix, every
independent verification — over 30 sections, written as the work happened,
not reconstructed afterward. This README is a summary of it; that document
is the evidence.

The underlying S17Code engine's own architecture and design — the live
graph, the capability registry, memory, A2A, the autonomy layer — is
documented separately in [`docs/ENGINE.md`](docs/ENGINE.md), unchanged from
the platform this product was built on top of.

---

## 8. Submission links

- **Part 1 — the product**: this repository.
  `https://github.com/deephazar-eva-ai/s17-answers-engine-agent-built`
- **Part 2 — a real bug fix, as a pull request against `S17Code` or
  `glc_v5`**: two, both independent of this product build, both real,
  both tested:
  - [`theschoolofai/S17Code#22`](https://github.com/theschoolofai/S17Code/pull/22)
    — the shipped `.env.example`'s `S17_PROTECTED_PATHS` silently weakened
    the coding guard below its own hardcoded default, so the README's own
    `cp .env.example .env` setup step left a fresh checkout less protected
    than the code intended, with no warning.
  - [`theschoolofai/glc_v5#39`](https://github.com/theschoolofai/glc_v5/pull/39)
    — email-channel pairing was case-sensitive; a real paired owner could
    be silently reclassified as untrusted by a later message arriving in
    different letter case than the address they first paired with.

Full detail on both, including verification, is in `docs/myowncodeworking.md`
§34.

---

## 9. CI, green for the first time

This repository's `ci` workflow (`uv run ruff check .` → `pytest` → the three
offline economics/trace proofs) had never once passed — both pushes to
`master` failed at the very first step, `ruff check`, which meant `pytest`
and the proofs had never actually run in CI at all.

**The lint failure itself**: ~50 accumulated `ruff` violations (unsorted/
unused imports, one multi-import line, one ambiguous `l` loop variable).
Cleaning them up was mostly mechanical (`ruff check --fix .`), but two of the
"unused imports" ruff wanted removed were load-bearing:

- `s17code/workers/special.py` called `os.getenv`/`os.environ` without ever
  importing `os` — `F821 Undefined name 'os'`, a real `NameError` waiting to
  happen the first time `run_validate_work` executed.
- `s17code/workers/general.py`'s `run_retriever` called a bare, undefined
  `recall(...)` — `F821 Undefined name 'recall'` — instead of delegating to
  `s17code.workers.special.recall`, the function that actually does the
  memory lookup. The retriever capability could never have worked.

Removing the *actually*-unused import of those same parsing helpers from
`s17code/runtime.py` (a leftover from before the worker-extraction refactor)
broke three tests that had been silently monkeypatching or importing through
that dead re-export instead of the real module (`s17code.workers.general`,
`s17code.workers.parsing`) — fixed by pointing them at the real call sites.

**Clearing the lint step exposed two more failures CI had never reached:**

- `proofs/p4_trace_export.py --offline` failed because the offline dummy
  transport (`proofs/harness.py`'s `OfflineTransport`) returns the same
  task-blind digest text for every prompt, including the planner's own
  strict JSON protocol — so the real planner could never parse a valid
  patch, burned all its repair attempts, and the run finished with zero
  graph nodes and no `node`-kind trace span. Gave `OfflineTransport` the
  smallest valid reply for exactly two roles — the planner's decision call
  and the evidence-readiness critic — so an offline run now produces one
  real `answer_with_evidence` node. That in turn exposed a genuine bug in
  the proof's own hierarchy check: `EXPECTED_PARENT` required every
  `provider_call` span's parent to be `node`, but `s17code/telemetry/
  spans.py` intentionally parents the planner's *own* metered calls under
  `plan` (they are provider calls too, made before any node exists).
  Widened the check to accept either.
- `tests/test_capability_contracts.py::test_every_registered_capability_
  has_a_worker_and_every_worker_is_registered` built `AgentRuntime()`
  directly without swapping in `DeterministicEmbedder` the way every other
  test in the suite does, so its one `memory.write()` call reached for the
  real Ollama embedder. That happened to succeed on a machine with Ollama
  running locally — masking the bug for months — and failed on the CI
  runner with `urllib.error.URLError: <urlopen error [Errno 111] Connection
  refused>`. Same fix as everywhere else: pin the embedder before the run.

None of these five bugs were introduced by this rebuild — `git log` shows
none of the touched files (`s17code/workers/*.py`, `s17code/runtime.py`,
`proofs/harness.py`, `proofs/p4_trace_export.py`,
`tests/test_capability_contracts.py`) were edited by either commit on this
branch; they were latent in the base import, invisible only because CI had
never gotten far enough to trip over them. Verified locally end-to-end
(`ruff check .`, full `pytest -q`, all three proofs `--offline`) before
pushing, then confirmed on the actual runner:

```
$ gh run list --workflow=ci.yml --limit 3
success  push  Isolate the capability-contract probe test from the real embedder
failure  push  Fix ruff lint failures and two bugs they were masking
failure  push  Document submission links: product repo, and both Part 2 bug-fix PRs
```
