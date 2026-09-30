# Topic-Based Twitter Report Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Use superpowers:subagent-driven-development only if the user explicitly requests delegation. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a repeatable CLI report covering every configured topic in one fixed 24-hour interval, backed by inspectable evidence and observed engagement ranking.

**Architecture:** Reuse the browser and bounded post reader. Add topic loading/query construction in `topic_config.py`; keep collection, selection, artifact writing, and Markdown rendering together in `report.py`. The CLI connects once and dispatches the report workflow; the calling agent can synthesize narrative from the saved evidence.

**Tech Stack:** Python >=3.12, existing Playwright, argparse, pathlib, json, datetime, tempfile, unittest, unittest.mock; no new dependencies.

**Spec:** `docs/superpowers/specs/2026-09-30-twitter-topic-report-design.md` (workspace-root relative).

## Global Constraints

- Python >=3.12; use the existing Playwright dependency and standard library only.
- Keep rendering in `report.py`; do not create `report_render.py`.
- Reuse `Browser`, `read_posts`, DOM extraction, `AgentError`, and CLI JSON conventions.
- Preserve existing `search`, `watch`, timeline, thread, and reply behavior.
- Include every configured topic, including empty, failed, and skipped topics.
- Rank by observed likes + retweets + replies; do not filter by reply-opportunity score.
- All topics use one UTC half-open interval: start <= post timestamp < end.
- Label results as highest-engagement posts found, not exhaustive coverage of X.
- Do not commit, overwrite existing reports, or modify unrelated working-tree changes without user instruction.

## Working context and file map

Application paths below are relative to `projects/social-content`; run application commands there. Documentation paths beginning `docs/superpowers` are relative to the workspace root. The application is a Git submodule. Inspect both root and submodule status before implementation. Existing uncommitted `cli.py`, `README.md`, `topics.json`, and report artifacts are user work. The current search min-score default is already None; do not redo that change.

Create only these production modules:

| File | Responsibility |
| --- | --- |
| `tools/twitter_agent/topic_config.py` | Topic JSON validation and explicit query construction |
| `tools/twitter_agent/report.py` | Candidate selection, report orchestration, evidence and Markdown artifacts |

Modify `tools/twitter_agent/posts.py`, `cli.py`, and `README.md`. Add focused `tests/test_topic_config.py` and `tests/test_report.py`; extend `tests/test_dom.py` and `tests/test_cli.py`. Update `tests/README.md` frontmatter. Do not extract watch helpers or add a persistence service for this feature.

## Task 1: Topic configuration and explicit queries

**Files:** Create `tools/twitter_agent/topic_config.py`, `tools/twitter_agent/tests/test_topic_config.py`.

**Interfaces:**

```python
def load_topics(path: Path) -> list[dict]: ...
def build_topic_query(topic: dict, *, start: int, end: int) -> str: ...
```

Both functions raise `AgentError('invalid_topics_file', message)` for invalid topic data. Query time bounds are validated by the caller. Preserve metadata and input order; do not mutate input dictionaries.

- [ ] **Step 1: Add failing configuration/query tests.** Use `TemporaryDirectory` for file-backed tests and these exact behavioral assertions:

```python
import json
import tempfile
import unittest
from pathlib import Path
from tools.twitter_agent.models import AgentError
from tools.twitter_agent.topic_config import load_topics, build_topic_query

class TopicConfigTests(unittest.TestCase):
    def test_query_preserves_phrases_and_scopes_time(self):
        topic = {'id': 'ai', 'name': 'AI',
                 'search_keywords': ['"agent orchestration"', 'AI security', 'CrewAI']}
        self.assertEqual(build_topic_query(topic, start=100, end=200),
            '("agent orchestration" OR "AI security" OR CrewAI) since_time:100 until_time:200')

    def test_duplicate_ids_rejected(self):
        topic = {'id': 'ai', 'name': 'AI', 'search_keywords': ['AI']}
        with tempfile.TemporaryDirectory() as directory:
            path = Path(directory) / 'topics.json'
            path.write_text(json.dumps({'topics': [topic, topic]}), encoding='utf-8')
            with self.assertRaises(AgentError) as raised:
                load_topics(path)
        self.assertEqual(raised.exception.code, 'invalid_topics_file')
```

Add subtests for missing/unreadable file, invalid UTF-8, malformed JSON, non-object root, empty/missing/non-list topics, non-object entries, invalid IDs/names, empty/non-string keywords, embedded or unbalanced quotes, control characters, and operator-shaped terms. Assert errors identify the offending topic/field. Load the real bundled configuration and confirm every configured topic validates without editing it. Assert that the loaded list equals the configured list instead of hardcoding its count or IDs. The current version-2 list targets technical builders and founders; personal relevance, priority, reply angles, and archetypes are not required fields.

- [ ] **Step 2: Run red.** `uv run python -m unittest discover -s tools/twitter_agent/tests -p test_topic_config.py -v` should fail because the module does not exist.
- [ ] **Step 3: Implement the two functions.** Catch `OSError`, `UnicodeError`, and `json.JSONDecodeError` around loading and wrap with the file path. Validate using explicit types (do not coerce values). A private keyword-normalization function shared by loading and query building should follow:

```python
term = keyword.strip()
if term.startswith('"') and term.endswith('"') and len(term) > 2:
    inner = term[1:-1]
    # Reject empty inner text, embedded quotes, or control characters.
    normalized = '"' + inner + '"'
else:
    # Reject quotes, control characters, ':', '(', ')', standalone AND/OR/NOT.
    normalized = '"' + term + '"' if any(c.isspace() for c in term) else term
query = '(' + ' OR '.join(normalized_terms) + ')'
query += f' since_time:{start} until_time:{end}'
```

Raise the specified typed error for each rejected condition; record field paths such as `topics[2].search_keywords[1]`. Reject whitespace-only quoted terms too.
- [ ] **Step 4: Run the same tests green.** Confirm keyword input, metadata, and topic ordering remain unchanged.

## Task 2: Extend the reader with Top/Latest selection

**Files:** Modify `tools/twitter_agent/posts.py` and `tools/twitter_agent/tests/test_dom.py`.

**Consumes:** Existing extraction/scrolling implementation.

**Produces:** Backward-compatible signature:

```python
def read_posts(page, mode: str, value: Optional[str] = None,
               limit: int = 10, *, search_mode: str = 'latest') -> dict: ...
```

- [ ] **Step 1: Add mocked navigation tests in a separate `SearchModeTests` unittest class so URL tests do not launch Chromium.** Assert Latest remains the default, Top navigates with `f=top`, and invalid modes raise `invalid_search_mode` before navigation. Use the existing real DOM tests to retain extraction coverage.

```python
from urllib.parse import parse_qs, urlsplit

class SearchModeTests(unittest.TestCase):
    @patch('tools.twitter_agent.posts.selectors.detect_block', return_value=None)
    def test_top_navigation(self, _detect):
        page = MagicMock()
        page.url = 'https://x.com/home'
        page.evaluate.return_value = [{'id': '1', 'metrics': {'total': 0}}]
        read_posts(page, 'search', 'AI agents', limit=1, search_mode='top')
        params = parse_qs(urlsplit(page.goto.call_args.args[0]).query)
        self.assertEqual(params['q'], ['AI agents'])
        self.assertEqual(params['f'], ['top'])
```

- [ ] **Step 2: Run red.** `uv run python -m unittest discover -s tools/twitter_agent/tests -p test_dom.py -k SearchModeTests -v` should reject the new keyword before implementation.
- [ ] **Step 3: Add keyword-only search_mode validation and URL selection.** Keep all other reader behavior intact.

```python
if search_mode not in ('latest', 'top'):
    raise AgentError('invalid_search_mode', 'search-mode must be latest or top.')
# In the search-navigation branch only:
feed = 'live' if search_mode == 'latest' else 'top'
page.goto(f'https://x.com/search?q={encoded}&f={feed}', wait_until='domcontentloaded')
```

- [ ] **Step 4: Run navigation tests green, then the full `test_dom.py` module.** Command: `uv run python -m unittest discover -s tools/twitter_agent/tests -p test_dom.py -v`. If local Chromium is missing, use `uv run python -m playwright install chromium`, then rerun. These tests need no X credentials.

## Task 3: Deterministic per-topic selection and evidence

**Files:** Create `tools/twitter_agent/report.py` and `tools/twitter_agent/tests/test_report.py`.

**Produces:**

```python
def select_candidates(observations: list[dict], *, start: int, end: int,
                      per_topic: int) -> dict: ...
```

An observation is `{'post': <reader post>, 'mode': 'top'|'latest', 'observed_at': <UTC ISO string>}`. Return `{'candidates': [...], 'selected_ids': [...], 'counts': {...}}` with the fields in the spec. Each candidate flattens the retained post plus observation metadata and exclusion_reason. Use internal parsed timestamps only for filtering/sorting; serialize the original timestamp.

- [ ] **Step 1: Write failing selection tests with fixed epoch boundaries.** Cover start inclusion/end exclusion, malformed/naive/missing timestamps, ads, low opportunity/high engagement, ties, fewer than N, and duplicate observation replacement without summing metrics.

```python
from datetime import datetime, timezone
from tools.twitter_agent.report import select_candidates

def post(pid, timestamp, total, **extra):
    return {'id': pid, 'url': f'https://x.com/i/status/{pid}',
            'author': {'name': 'Test', 'handle': '@test'}, 'text': 'Evidence',
            'timestamp': timestamp, 'is_ad': False, 'score': 0,
            'metrics': {'likes': total, 'retweets': 0, 'replies': 0,
                        'views': 100, 'total': total}, **extra}

def observation(item, mode='top'):
    return {'post': item, 'mode': mode, 'observed_at': '2026-09-30T12:01:00Z'}

class SelectionTests(unittest.TestCase):
    def test_rank_after_time_filter_and_not_opportunity_score(self):
        end = int(datetime(2026, 9, 30, 12, tzinfo=timezone.utc).timestamp())
        result = select_candidates([
            observation(post('1', '2026-09-30T11:00:00Z', 2, score=99)),
            observation(post('2', '2026-09-30T10:00:00Z', 200)),
            observation(post('3', '2026-09-30T12:00:00Z', 999)),
            observation(post('4', '2026-09-29T12:00:00Z', 100)),
        ], start=end-86400, end=end, per_topic=2)
        self.assertEqual(result['selected_ids'], ['2', '4'])
        self.assertEqual(result['counts']['excluded_outside_window'], 1)
```

- [ ] **Step 2: Run red.** `uv run python -m unittest discover -s tools/twitter_agent/tests -p test_report.py -v`.
- [ ] **Step 3: Implement selection.** Copy input posts. Merge IDs in insertion order, keeping the latest observation and ordered unique source_modes. Parse ISO timestamps with `datetime.fromisoformat(value.replace('Z', '+00:00'))`; reject nonstrings and timezone-naive values. Apply one exclusion per candidate in the specified order. Build counts from the final candidate set, not from duplicates. Select using:

```python
eligible.sort(key=lambda item: (
    -item['metrics']['total'], -parsed_timestamps[item['id']], item['id']))
selected_ids = [item['id'] for item in eligible[:per_topic]]
```

The reader owns numeric metric extraction; retain its observed values, including rounded displayed metrics. Do not call `score_tweet` or introduce another engagement formula.
- [ ] **Step 4: Run selection tests green.** Add a non-mutation assertion and verify `collected - unique == duplicates` and `unique == eligible + sum(exclusion counts)`.

## Task 4: All-topic collection with bounded budgets and partial failures

**Files:** Modify `tools/twitter_agent/report.py`, `tools/twitter_agent/tests/test_report.py`.

**Consumes:** `build_topic_query`, `read_posts`, `select_candidates`.

**Produces:**

```python
def collect_report(page, topics: list[dict], *, topics_file: str,
                   start: int, end: int, per_topic: int,
                   candidate_limit: int) -> dict: ...
```

The CLI supplies already-validated parameters and frozen timestamps. `collect_report` does not create browser connections or write files. `window_hours` in config is `(end-start)//3600`.

- [ ] **Step 1: Add failing mocked collection tests.** Use two or more topics and patch `tools.twitter_agent.report.read_posts`. Assert exact Top/Latest calls, one interval, two-topic membership for the same ID, all-topic preservation, and ordered output. Use an odd budget to verify allocation and no extra collection calls:

```python
@patch('tools.twitter_agent.report.read_posts')
def test_budget_and_empty_topic_coverage(self, reader):
    reader.return_value = {'posts': [], 'reason': 'no_progress', 'scrolls': 3}
    topics = [{'id': 'ai', 'name': 'AI', 'search_keywords': ['AI']},
              {'id': 'zk', 'name': 'ZK', 'search_keywords': ['ZK']}]
    result = collect_report(object(), topics, topics_file='topics.json',
                            start=100, end=86500, per_topic=2, candidate_limit=5)
    self.assertEqual([t['id'] for t in result['topics']], ['ai', 'zk'])
    self.assertEqual([c.kwargs['limit'] for c in reader.call_args_list], [3, 2, 3, 2])
    self.assertEqual([c.kwargs['search_mode'] for c in reader.call_args_list],
                     ['top', 'latest', 'top', 'latest'])
    self.assertTrue(all(t['status'] == 'empty' for t in result['topics']))
    self.assertFalse(result['partial'])
```

Add cases for one recoverable failed search followed by success, both failed searches, Playwright `Error`, `reason='timeout'` with useful posts, fatal error after earlier success, fatal error on the last topic, and unexpected `RuntimeError` propagating. Verify summary selected entries versus unique selected posts.
- [ ] **Step 2: Run report tests red.** Use the Task 3 test command.
- [ ] **Step 3: Implement the collection loop and error records.** Prebuild one entry/query for every topic. Loop Top then Latest with budgets `(candidate_limit+1)//2` and `candidate_limit//2`. Record each read's observed time with timezone-aware UTC. On a caught error, serialize code/message/human_action_required. Fatal codes are:

```python
FATAL_REPORT_ERRORS = {
    'not_authenticated', 'browser_challenge', 'account_blocked',
    'browser_not_connected', 'browser_connection_failed', 'no_browser_context',
}
```

For fatal errors mark the current search as failed; mark its remaining search records skipped and all future topic records skipped. Complete candidate selection for the current topic if any evidence exists. Record skipped search records with `reason='skipped'`, collected=0, scrolls=0, observed_at=null and error=null. Distinguish them from completed no-progress searches. A topic with failed searches but no successful read is error; a topic with both success and failure is partial. Timeout stop reasons override successful topic status to partial. Any failed/skipped/timeout search makes top-level partial true.

Calculate summary after all entries are finalized. Do not derive report coverage from the reader's legacy partial field.
- [ ] **Step 4: Run report tests green.** Verify errors preserve collected evidence and never silently omit configured topics.

## Task 5: Render and save artifacts inside report.py

**Files:** Modify `tools/twitter_agent/report.py`, `tools/twitter_agent/tests/test_report.py`.

**Consumes:** The Task 4 report dictionary.

**Produces:**

```python
def render_markdown(report: dict) -> str: ...
def write_report(report: dict, output_dir: Path) -> dict[str, str]: ...
```

Return paths under keys `run_dir`, `evidence_json`, `report_markdown`.

- [ ] **Step 1: Add failing artifact tests.** Build a fixture via mocked `collect_report`, containing selected, empty, partial, and skipped topic entries. Assert the Markdown includes all names/statuses, exact window, metrics, observation time, and selected canonical URLs. Assert unselected candidate URLs are absent from Markdown but retained in JSON. Include source text containing `<script>`, headings, backticks, and Markdown links to verify it cannot alter the report structure.

```python
def test_two_writes_preserve_both_runs(self):
    with tempfile.TemporaryDirectory() as directory:
        first = write_report(self.report, Path(directory))
        second = write_report(self.report, Path(directory))
        self.assertNotEqual(first['run_dir'], second['run_dir'])
        for paths in (first, second):
            saved = json.loads(Path(paths['evidence_json']).read_text(encoding='utf-8'))
            self.assertEqual(saved, self.report)
            self.assertEqual(Path(paths['report_markdown']).read_text(encoding='utf-8'),
                             render_markdown(self.report))
```

Initialize `self.report` using the collection fixture in `setUp`. Test an output-dir path that is a file and a mocked failure on the second artifact write: both must raise `report_write_failed`, never return success paths.
- [ ] **Step 2: Run report tests red.** Use the Task 3 test command.
- [ ] **Step 3: Implement rendering and writing.** Use small private functions in `report.py`, not another module. Escape HTML via `html.escape` and backslash-escape Markdown punctuation in text/labels; prefix every extracted-text line with `> `. Render metrics from the retained observation. Build links from `https://x.com/i/status/{id}`. Include bounded-sample wording, coverage table, errors, search stop reasons and exclusion counts. No fabricated synopsis or automatic copying of reply angles as personalized recommendations.

For writing, create output-dir parents, then use `tempfile.mkdtemp(prefix='report-', dir=output_dir)`. Serialize evidence with `json.dumps(report, ensure_ascii=False, indent=2) + '\n'`. Write each artifact via a temporary file in the run directory and `Path.replace` to its final name. On failure remove leftover temporary files, retain already finalized artifacts for diagnosis, and raise `AgentError('report_write_failed', ...)` including the directory path. Return paths only after both writes succeed.
- [ ] **Step 4: Run report tests green.** Read both files back; verify rendering is deterministic and no existing run was modified.

## Task 6: CLI integration, documentation, and end-to-end verification

**Files:** Modify `tools/twitter_agent/cli.py`, `tools/twitter_agent/tests/test_cli.py`, `tools/twitter_agent/README.md`, `tools/twitter_agent/tests/README.md`.

**Consumes:** All interfaces from Tasks 1–5 plus existing `Browser`, `AgentError`, and emit helpers.

**Produces:** Supported `report` subcommand with spec-defined exit and output behavior.

- [ ] **Step 1: Add failing CLI tests.** Extend existing `CliTests` using its temp directory and stdout-capture conventions. Test defaults, actual JSON loading, mocked browser collection, real artifact writing, invalid flags before browser access, malformed topic data, browser connection failure, degraded run exit 4 with artifacts, and write failure exit 1. Assert no progress text contaminates stdout.

```python
def test_report_invalid_budget_does_not_connect(self):
    with patch('tools.twitter_agent.cli.Browser') as browser, \
         patch('sys.stdout', new_callable=io.StringIO) as out:
        code = main(['--state-dir', str(self.state_dir), 'report',
                     '--candidate-limit', '1'])
    self.assertEqual(code, 2)
    self.assertEqual(json.loads(out.getvalue())['error']['code'], 'invalid_arguments')
    browser.assert_not_called()
```

Add equivalent cases for zero/negative hours, candidate-limit=101, per-topic=0, and per-topic greater than candidate-limit. Patch `time.time` to verify the same frozen start/end is passed through collection. Verify omitted topics-file resolves beside the module and output-dir defaults under the supplied state-dir. Exercise subprocess `report --help` without opening a browser.
- [ ] **Step 2: Run CLI tests red.** `uv run python -m unittest discover -s tools/twitter_agent/tests -p test_cli.py -v`.
- [ ] **Step 3: Implement argparse and dispatch.** Add these arguments:

```python
rp = subparsers.add_parser('report', help='Report high-engagement posts across configured topics')
rp.add_argument('--topics-file', type=Path, default=Path(__file__).with_name('topics.json'))
rp.add_argument('--window-hours', type=int, default=24)
rp.add_argument('--per-topic', type=int, default=5)
rp.add_argument('--candidate-limit', type=int, default=100)
rp.add_argument('--output-dir', type=Path, default=None)
```

Use help text explaining candidate-limit is total per topic split Top/Latest, and per-topic is the final selection count. Add `invalid_topics_file` and `invalid_search_mode` to exit-code-2 mappings. `report_write_failed` uses the default exit 1. Dispatch follows:

```python
if args.window_hours <= 0:
    raise AgentError('invalid_arguments', 'window-hours must be positive.')
if not 2 <= args.candidate_limit <= 100:
    raise AgentError('invalid_arguments', 'candidate-limit must be between 2 and 100.')
if not 1 <= args.per_topic <= args.candidate_limit:
    raise AgentError('invalid_arguments', 'per-topic must be between 1 and candidate-limit.')
topics = load_topics(args.topics_file)
end = int(time.time())
start = end - args.window_hours * 3600
with Browser(endpoint=args.cdp, timeout_ms=args.timeout_ms) as browser:
    report = collect_report(browser.new_page(), topics,
        topics_file=str(args.topics_file.resolve()), start=start, end=end,
        per_topic=args.per_topic, candidate_limit=args.candidate_limit)
paths = write_report(report, args.output_dir or Path(args.state_dir) / 'reports')
emit_success({'artifacts': paths, 'summary': report['summary'],
              'partial': report['partial'], 'coverage': report['coverage'],
              'window': report['window']})
return 4 if report['partial'] else 0
```

Keep the existing outer error handling for global failures. Avoid refactoring Store construction or other subcommands as part of this work.
- [ ] **Step 4: Run CLI tests green.** Use the Task 6 command and check actual saved JSON and Markdown contents in the happy-path test, not only mocked method calls.
- [ ] **Step 5: Document the supported agent route in the tool README.** Add the spec's command example, prerequisites (`doctor`), exact time/metric semantics, two artifact roles, budget allocation, no-exhaustive-coverage statement, and exit-4/ok=true meaning. Tell agents to inspect coverage/errors and use evidence for narrative reports. Document that fewer than N matches is a legitimate shortfall. Update tool frontmatter for the two modules and test frontmatter for the two new test files. Preserve existing user-authored README changes.
- [ ] **Step 6: Run final checks from `projects/social-content`.**

```bash
uv run python -m unittest discover -s tools/twitter_agent/tests -v
uv run python -m tools.twitter_agent --help
uv run python -m tools.twitter_agent report --help
git diff --check
```

- [ ] **Step 7: Run repository documentation checks from the workspace root.**

```bash
uv run --project . .agents/hooks/file-selector/lint_readme_tree.py --changed-only
git diff --check
```

The AGENTS.md-mandated lint script was not found in this checkout during planning. Check whether it has been restored before this step; if still absent, report the documentation-lint gate as blocked and manually inspect the changed README frontmatter. Do not fabricate a passing result or add a replacement hook as part of this feature. Inspect any lint failures before editing; report pre-existing issues separately. Review submodule diff/status as well as root diff/status. No commits unless explicitly requested.
- [ ] **Step 8: Optional live smoke check when an authenticated CDP browser is available.** Run `doctor`, then the documented report command with `--candidate-limit 10 --per-topic 2` and a temporary output directory. Confirm X actually displays Top and Latest as requested, collected timestamps fit the saved interval, and report topic IDs match the configured topic IDs in order. Do not infer extraction correctness solely from mocked navigation tests. If no session is available, state that live verification was not run; automated checks remain the required gate.

## Acceptance checklist

- [ ] One command reads the real topic configuration and saves both artifacts.
- [ ] Every configured topic appears, including errors and empty results.
- [ ] Selection happens after deduplication and strict window filtering.
- [ ] High-engagement posts survive even with low opportunity scores.
- [ ] Shared posts preserve membership in each matching topic.
- [ ] Candidate budgets, exclusions, observation times, and stop reasons are inspectable.
- [ ] Errors preserve collected evidence and produce documented exit codes.
- [ ] Existing CLI behavior and DOM tests remain passing.
- [ ] Rendering remains in `report.py`; no extra renderer, dependency, database, or agent-generated scratch script is introduced.
