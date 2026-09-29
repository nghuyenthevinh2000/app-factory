# Supervised X DOM CLI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans for inline execution or superpowers:subagent-driven-development if the user selects delegation. Steps use checkbox syntax for tracking.

**Goal:** Build an agent-facing X DOM CLI with visible browser operations and mandatory, per-reply human approval in a separate terminal.

**Architecture:** Python modules attach to an existing Chrome session through Playwright CDP. SQLite persists drafts, limits, and audit events; one locked supervisor process owns the reply tab and the only submission path. Read commands use temporary tabs.

**Tech Stack:** Python 3.12+, uv, Playwright, SQLite, argparse, unittest, POSIX flock on macOS.

**Spec:** `docs/superpowers/specs/2026-09-29-twitter-agent-design.md` (app-factory root).

## Global constraints

- Python 3.12+; implementation lives under `projects/social-content/tools/twitter_agent/`.
- Every reply requires explicit human approval before submission.
- Agent commands emit structured JSON to stdout and readable progress to stderr.
- There is no agent-facing approval command, automatic approval flag, or approval accepted through piped stdin.
- Default operational limits are 5 submission attempts per rolling hour, 30 per rolling 24 hours, and at least 60 seconds between attempts.
- Click at most once per approved attempt. Do not retry submissions automatically.
- Read and write through DOM interactions, not private APIs.
- Keep the Chrome profile outside the repository.
- Automated tests do not post to a real X account.
- Update directory README frontmatter and the root routing manifest as needed.
- Do not commit or update the parent submodule pointer unless the user requests it.

## Repository context and file map

All paths below are relative to `projects/social-content`, except the spec and
this plan. Run commands with that directory as the working directory. It is a
separate Git repository with a clean working tree at planning time. The parent
repository already has staged submodule changes that must be preserved.

The project uses `uv_build` and has a placeholder application entry point in
`src/social_content/__init__.py`. Leave that entry point alone; the new CLI runs
from the repository with `uv run python -m tools.twitter_agent`.

Create:

| File | Responsibility |
| --- | --- |
| `tools/README.md` | Tool directory routing and usage index |
| `tools/twitter_agent/README.md` | Setup, agent interface, supervisor workflow, recovery |
| `tools/twitter_agent/__init__.py` | Package marker |
| `tools/twitter_agent/__main__.py` | Call CLI main and set process exit code |
| `tools/twitter_agent/models.py` | Target normalization, draft digest, typed errors/config |
| `tools/twitter_agent/store.py` | SQLite queue, transitions, limits, audit, process lock |
| `tools/twitter_agent/selectors.py` | All X DOM selectors |
| `tools/twitter_agent/browser.py` | CDP lifecycle, temporary tabs, account/block detection |
| `tools/twitter_agent/posts.py` | Post extraction and bounded read operations |
| `tools/twitter_agent/replies.py` | Prepare, inspect, and submit browser interactions |
| `tools/twitter_agent/supervisor.py` | Interactive review and submission orchestration |
| `tools/twitter_agent/cli.py` | Parser, dispatch, JSON responses, progress |
| `tools/twitter_agent/tests/README.md` | Test directory frontmatter and commands |
| `tools/twitter_agent/tests/test_models.py` | Input normalization and exact-text hashing |
| `tools/twitter_agent/tests/test_store.py` | State, locking, recovery, quotas |
| `tools/twitter_agent/tests/test_dom.py` | Local HTML fixture browser tests |
| `tools/twitter_agent/tests/test_supervisor.py` | Human decision and failure boundary tests |
| `tools/twitter_agent/tests/test_cli.py` | Subprocess interface tests |

Modify `pyproject.toml`, regenerate `uv.lock`, add `.twitter-agent/` to
`.gitignore`, and add tools routing to `FRONTMATTER.md` and root `README.md`.
Use inline HTML fixtures in tests to avoid an extra fixture directory.

## Shared interfaces

Use synchronous Playwright; this is intentionally a sequential tool. Store
timestamps as UTC Unix seconds; permit an injected clock in store tests.

```python
@dataclass(frozen=True)
class Target:
    id: str
    url: str

@dataclass(frozen=True)
class Limits:
    hourly: int = 5
    daily: int = 30
    spacing: float = 60.0

class AgentError(Exception):
    # code: str; message: str; human_action_required: bool
    pass

def normalize_target(value: str) -> Target: ...
def draft_digest(target_id: str, text: str) -> str: ...

class Store:
    def __init__(self, root: Path, clock: Callable[[], float] = time.time): ...
    def enqueue(self, target: Target, text: str) -> dict: ...
    def enqueue_many(self, items: list[tuple[Target, str]]) -> list[dict]: ...
    def get(self, draft_id: str) -> dict: ...
    def status(self) -> dict: ...
    def set_paused(self, paused: bool) -> None: ...
    def cancel(self, draft_id: str) -> dict: ...
    def configure_limits(self, limits: Limits) -> None: ...
    def supervisor_lock(self) -> ContextManager[None]: ...
    def recover(self) -> None: ...
    def claim_next(self) -> dict | None: ...
    def defer(self, draft_id: str, reason: str) -> None: ...
    def reject(self, draft_id: str) -> None: ...
    def begin_submission(self, draft_id: str, digest: str) -> None: ...
    def finish(self, draft_id: str, state: str, detail: dict) -> None: ...
    def event(self, kind: str, draft_id: str | None, detail: dict) -> None: ...
    def heartbeat(self) -> None: ...

class Browser:
    # Context manager: start Playwright, attach, preserve existing tabs/browser.
    def __init__(self, endpoint: str, timeout_ms: int = 15000): ...
    def new_page(self): ...
    def doctor(self) -> dict: ...

def read_posts(page, mode: str, value: str | None, limit: int) -> dict: ...
def prepare_reply(page, draft: dict, artifact_dir: Path) -> dict: ...
def inspect_reply(page, draft: dict) -> str: ...
def submit_reply(page, draft: dict, artifact_dir: Path) -> dict: ...
def review_once(store: Store, page, decide: Callable[[dict], str]) -> bool: ...
def supervise(store: Store, browser: Browser) -> None: ...
def main(argv: list[str] | None = None) -> int: ...
```

These declarations specify interfaces, not implementation stubs. Implement
each in its owning task. Draft dictionaries contain `id`, `target_id`,
`target_url`, `text`, `digest`, `state`, `created_at`, `updated_at`, and `detail`.

### Task 1: Persistent drafts and guarded submission transitions

**Files:** `models.py`, `store.py`, `__init__.py`, `test_models.py`,
`test_store.py`, directory READMEs, `.gitignore`.

**Consumes:** Python standard library only.
**Produces:** `Target`, `Limits`, `AgentError`, normalization/digest functions,
and all `Store` methods defined above.

- [ ] Write unittest cases with temporary state directories and injected time:

```python
def test_paused_draft_cannot_begin_submission(self):
    draft = self.store.enqueue(normalize_target('123'), 'Exact reply')
    self.store.claim_next()
    self.store.set_paused(True)
    with self.assertRaises(AgentError) as caught:
        self.store.begin_submission(draft['id'], draft['digest'])
    self.assertEqual(caught.exception.code, 'paused')
    self.assertNotEqual(self.store.get(draft['id'])['state'], 'submitting')

def test_recovery_never_requeues_a_possible_submission(self):
    draft = self.store.enqueue(normalize_target('123'), 'Exact reply')
    self.store.claim_next()
    self.store.begin_submission(draft['id'], draft['digest'])
    self.store.recover()
    self.assertEqual(self.store.get(draft['id'])['state'], 'uncertain')
    self.assertIsNone(self.store.claim_next())
```

- [ ] Run `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_store.py' -v` and confirm missing implementation failure.
- [ ] Implement SQLite schema with `drafts`, `events`, and `settings`. Use
  unique digest for duplicate suppression; JSON detail fields; UUID draft IDs;
  transactions with `BEGIN IMMEDIATE` for queue mutations and quota checks.
  `enqueue_many` validates everything before one transaction, preventing partial
  queue imports. Enforce nonempty text without stripping stored text, and reject
  NUL/control characters other than tabs/newlines.

```python
payload = json.dumps([target_id, text], ensure_ascii=False, separators=(',', ':'))
digest = hashlib.sha256(payload.encode('utf-8')).hexdigest()
# begin_submission runs inside one IMMEDIATE transaction:
# check reviewing + digest + not paused + persisted attempt windows;
# update state and append approval/submission_started events before COMMIT.
```

- [ ] Normalize only positive numeric IDs or HTTPS/HTTP X/Twitter status URLs
  on exact supported hosts (`x.com`, `www.x.com`, `twitter.com`,
  `www.twitter.com`); reject credentials, unexpected ports, and foreign hosts.
  Canonical navigation URL is `https://x.com/i/status/ID`. Accept standard
  `/HANDLE/status/ID` and `/i/status/ID`, stripping query/fragment.
- [ ] Add tests for quoted URL tricks, empty text, Unicode exactness, duplicate
  imports, FIFO, cancellation during review, invalid digest, hourly/daily/spacing
  boundaries, persisted settings, and a second process failing `flock` acquisition.
  Count `submission_started` events including uncertain outcomes. Record
  supervisor PID/session/heartbeat; determine running state with lock availability
  rather than heartbeat alone, since a human may review for a long time.
- [ ] Create state directory with mode 0700 and files with 0600; persist limits
  on first initialization only. Recover reviewing drafts to pending and
  submitting drafts to uncertain while holding the supervisor lock.
- [ ] Run both model and store test files to green. Review diff; leave uncommitted.

### Task 2: CDP attachment and bounded read operations

**Files:** `browser.py`, `selectors.py`, `posts.py`, `test_dom.py`,
`pyproject.toml`, `uv.lock`.

**Consumes:** `AgentError`, `normalize_target`.
**Produces:** `Browser`, `read_posts`; post records with ID, URL, author, text,
timestamp, and result metadata `partial`, `reason`, `scrolls`.

- [ ] Add Playwright with `uv add 'playwright>=1.50,<2'`; install fixture browser
  with `uv run playwright install chromium` after verifying the cache parent
  directory. Tests use this isolated browser, never the logged-in Chrome.
- [ ] Write a local fixture test for a textless outer tweet containing a quoted
  tweet; assert the outer timestamp ID is used and quoted text is not promoted.
  Add empty timeline, duplicate virtualized posts, and scroll-budget cases.

```python
page.set_content('''<article data-testid="tweet">
  <div data-testid="User-Name">Alice @alice</div>
  <a href="/alice/status/123"><time datetime="2026-09-29T10:00:00Z">now</time></a>
  <div role="link"><div data-testid="tweetText">Quoted text</div>
    <a href="/bob/status/456"><time>earlier</time></a></div>
</article>''')
# Route navigation to the fixture in the test harness.
result = read_posts(page, 'timeline', None, 1)
self.assertEqual(result['posts'][0]['id'], '123')
self.assertEqual(result['posts'][0]['text'], '')
```

- [ ] Run `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_dom.py' -v` and confirm missing read implementation failure.
- [ ] Implement CDP context entry/exit; fail clearly if no context exists.
  Close only tool-created temporary tabs. Stopping Playwright disconnects the
  client without calling `browser.close()` on the user's browser.
- [ ] Implement centralized selectors for articles, author, body, timestamp,
  reply button, active dialog/composer, submit, login, and visible error/challenge
  indicators. Detect blocks before waits and on timeout; classify unknown timeout
  as `dom_timeout` rather than asserting a challenge without evidence.
- [ ] Extract only elements belonging to each top-level article, excluding
  nested quote cards. Deduplicate by numeric ID. Navigate to home, URL-encoded
  search with latest filter, or normalized target; thread mode includes a
  separately identified target. Stop at limit, 10 scrolls, 3 no-progress rounds,
  or 30 seconds. Report the actual stop reason and do not equate loaded thread
  context with a complete thread. Permit limits 1–100.
- [ ] Run local DOM tests plus model/store tests. Add a lifecycle mock test
  asserting existing tabs are never closed by read cleanup.

### Task 3: Browser preparation, exact review binding, and one-click submission

**Files:** `replies.py`, `supervisor.py`, `test_supervisor.py`, `test_dom.py`.

**Consumes:** store transitions, browser page, centralized selectors.
**Produces:** preparation/inspection/submission functions and `review_once`.

- [ ] Write tests proving rejection and text mismatch never call submit:

```python
def test_reject_never_clicks(self):
    self.store.enqueue(normalize_target('123'), 'Reviewed text')
    with patch('tools.twitter_agent.supervisor.prepare_reply', return_value={}), \
         patch('tools.twitter_agent.supervisor.submit_reply') as submit:
        review_once(self.store, self.page, lambda review: 'reject')
    submit.assert_not_called()

def test_changed_text_requires_new_review(self):
    draft = self.store.enqueue(normalize_target('123'), 'Reviewed text')
    with patch('tools.twitter_agent.supervisor.prepare_reply', return_value={}), \
         patch('tools.twitter_agent.supervisor.inspect_reply', return_value='different'), \
         patch('tools.twitter_agent.supervisor.submit_reply') as submit:
        review_once(self.store, self.page, lambda review: 'approve')
    submit.assert_not_called()
    self.assertEqual(self.store.get(draft['id'])['state'], 'pending')
```

- [ ] Run supervisor tests and confirm missing implementation failure.
- [ ] Implement prepare: navigate to target; identify its exact top-level
  article; click its reply button; scope to the visible composer dialog; fill
  contenteditable through Playwright input events; read back exact text; capture
  target context and screenshot. Do not assume the first article is the target.
  A screenshot failure appears as a warning in the review metadata.
- [ ] Implement inspect: verify current target and dialog identity, read exact
  composer text, check enabled submit button, and return its digest. Ambiguity
  raises `review_changed`. The final click path performs another DOM check.
- [ ] Implement `review_once`: claim FIFO draft, prepare, call decision callback,
  reject/defer/approve; compare digest and call transactional `begin_submission`.
  On pause, cancellation, quota exhaustion, or changed review, do not submit.
  Errors before attempting submission become failed or pending as appropriate;
  errors after `begin_submission` become uncertain. Persist diagnostic artifacts.
- [ ] Implement one submit click and bounded DOM confirmation. Before click,
  record existing reply IDs and signed-in handle. Confirm only a newly observed
  status link for the signed-in author with exact reply text in the target
  context; otherwise return uncertain. A disappearing composer or generic toast
  alone is not confirmation. Capture after screenshot independently of outcome.

```python
# Once begin_submission has committed, never loop back into submit on failure.
try:
    outcome = submit_reply(page, draft, artifact_dir)
except Exception as exc:
    store.finish(draft['id'], 'uncertain', {'error': str(exc)})
else:
    store.finish(draft['id'], outcome['state'], outcome)
```

- [ ] Add fixture tests for exact text/newlines, quoted-target confusion,
  disabled submit, edited composer, wrong target, confirmed new reply, old
  same-text reply not confirming, and composer closure without confirmation.
  Assert uncertain cases have exactly one click. Add crash/restart tests and
  cancellation/pause during the decision callback.
- [ ] Run DOM and supervisor tests to green; review all click call sites.

### Task 4: Agent CLI and interactive supervisor terminal

**Files:** `cli.py`, `__main__.py`, `supervisor.py`, `test_cli.py`.

**Consumes:** all prior interfaces.
**Produces:** executable command surface and real human review loop.

- [ ] Write subprocess tests using temporary state and no browser connection:

```python
cmd = [sys.executable, '-m', 'tools.twitter_agent', '--state-dir', str(root)]
prepared = subprocess.run(cmd + ['reply', 'prepare', '123', '--text', 'Hello'],
                          capture_output=True, text=True)
self.assertEqual(prepared.returncode, 0)
body = json.loads(prepared.stdout)
self.assertEqual(body['data']['draft']['state'], 'pending')
self.assertFalse(body['data']['browser_prepared'])
supervisor = subprocess.run(cmd + ['supervise'], input='approve\n',
                            capture_output=True, text=True)
self.assertNotEqual(supervisor.returncode, 0)
self.assertEqual(json.loads(supervisor.stdout)['error']['code'], 'interactive_required')
```

- [ ] Run CLI tests and confirm missing entry-point failure.
- [ ] Implement argparse subcommands from the spec; global options
  `--state-dir` (default `.twitter-agent`), `--cdp` (default
  `http://127.0.0.1:9222`), and `--timeout-ms`. Limits are optional flags on
  `supervise`: `--hourly-limit`, `--daily-limit`, `--min-spacing`; persist explicit
  changes only, reject nonpositive values. No approve subcommand exists.
- [ ] Emit success `{ "ok": true, "data": ... }` and failure
  `{ "ok": false, "error": { "code": ..., "message": ...,
  "human_action_required": ... } }`; exit 0 success, 2 bad input, 3 browser/auth
  error, 4 state/conflict, 1 unexpected failure. Override parser errors so even
  invalid arguments emit JSON. Keep help as human-readable text.
- [ ] Validate entire UTF-8 JSONL queue before committing it; reject malformed
  lines with a line number, missing or non-string values, and empty input.
  Status returns current draft, pending drafts with exact text, paused state,
  limits, supervisor availability, and recent audit events.
- [ ] Require `sys.stdin.isatty()` before connecting the supervisor browser.
  Hold supervisor lock, recover state, create visible reply tab, and loop with
  a one-second idle poll. Display target text, exact reply, screenshot status,
  and draft ID on stderr; require `approve DRAFT_ID` or `reject DRAFT_ID`.
  Read terminal input with a short polling timeout so external pause/cancel
  and heartbeat changes remain visible while waiting. On EOF/Ctrl-C, defer
  reviewing drafts and exit; already submitting drafts retain uncertainty.
- [ ] Add CLI tests for invalid arguments, absence of approve command, atomic
  malformed queue import, duplicate enqueue, status/cancel, pause/resume, and
  JSON stdout with progress on stderr. Use injected decision callbacks for unit
  tests and a pseudo-terminal test for the real interactive input path.
- [ ] Run all tests with `uv run python -m unittest discover -s tools/twitter_agent/tests -v`.

### Task 5: User guide, routing compliance, and final verification

**Files:** directory READMEs, root README, `FRONTMATTER.md`, plus any fixes
identified by required verification.

**Consumes:** implemented CLI and actual test results.
**Produces:** runnable instructions and a verified delivery summary.

- [ ] Document this two-terminal flow with exact commands:

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/chrome-twitter-profile"

# From projects/social-content; log into X manually in the Chrome profile.
uv sync
uv run python -m tools.twitter_agent doctor

# Human terminal:
uv run python -m tools.twitter_agent supervise

# Agent terminal:
uv run python -m tools.twitter_agent timeline --limit 10
uv run python -m tools.twitter_agent reply prepare 123 --text "Your reply"
uv run python -m tools.twitter_agent status
uv run python -m tools.twitter_agent pause
```

- [ ] Document JSONL format with `{"target":"123","text":"Your reply"}`;
  global options before subcommands; approval ownership; rejection/cancellation;
  persistent quotas; visible screenshots; state privacy; partial read results;
  and manual inspection of uncertain outcomes. Explain that same-user process
  access is not an OS isolation boundary and do not claim DOM automation avoids
  platform restrictions. Identify example ID `123` as a placeholder users replace.
- [ ] Add YAML frontmatter with `name`, `summary`, `tags`, and `submodules` to
  each created directory README. Register `tools/` in root routing and README.
  Check all module/test files are represented accurately in their parent map.
- [ ] Verify the target directory exists before dependency commands that create
  files. Run the complete test suite once after final code changes, CLI help,
  and the mandatory repository lint:

```bash
uv run python -m unittest discover -s tools/twitter_agent/tests -v
uv run python -m tools.twitter_agent --help
uv run --project . .agents/hooks/file-selector/lint_readme_tree.py --changed-only
git diff --check
git status --short
```

- [ ] Read created README files to verify nested directories even if Git reports
  the new `tools/` tree as a single untracked entry. Check `git diff` and untracked
  files for runtime artifacts and accidental profile data.
- [ ] Perform read-only live `doctor`/timeline checks only if the user's debug
  Chrome is available. Report unavailable live verification honestly. Real reply
  approval belongs to the user, not the implementing agent.
- [ ] Report implementation location, test results, startup commands, and any
  live-X validation still needed. Leave all work uncommitted.

## Plan self-review

- Scope coverage: tasks 1/3 cover approval binding, durable attempts, deduplication,
  FIFO, quotas, locking, and recovery; task 2 covers bounded DOM reads and browser
  lifecycle; task 4 covers command visibility and human interaction; task 5 covers
  setup, routing compliance, and final checks.
- Interface consistency: store calls, browser operations, draft fields, and CLI
  entry point use the shared declarations above throughout.
- Verification boundaries: local fixture tests prove controlled DOM behavior;
  live X selectors and successful real posting require supervised acceptance.
