# Topic-Based Twitter Report Design

## Goal and approval

Provide a supported CLI workflow for collecting the highest-engagement posts found in a fixed 24-hour window across every topic in `topics.json`. The user approved proceeding to planning and requested that rendering live inside `report.py`, not a separate renderer module.

The initial output is a deterministic evidence report. A calling agent may use it to compose a narrative briefing with citations; narrative generation is not embedded in the CLI.

The bundled topic list is general guidance for technical builders and founders, not a taxonomy of personal projects. It currently contains ten broad themes with neutral summaries and search terms. Personal biography, fixed editorial opinions, and reply strategy are outside the topic configuration. The report must load every configured topic dynamically; neither implementation nor tests should hardcode a topic count or specific topic IDs. Optional metadata must remain optional.

## Constraints

- Python >=3.12; use the existing Playwright dependency and standard library only.
- Keep rendering in `report.py`; do not create `report_render.py`.
- Reuse `Browser`, `read_posts`, DOM extraction, `AgentError`, and CLI JSON conventions.
- Preserve existing `search`, `watch`, timeline, thread, and reply behavior.
- Include every configured topic, including empty, failed, and skipped topics.
- Rank by observed likes + retweets + replies; do not filter by reply-opportunity score.
- All topics use one UTC half-open interval: start <= post timestamp < end.
- Label results as highest-engagement posts found, not exhaustive coverage of X.
- Do not commit, overwrite existing reports, or modify unrelated working-tree changes without user instruction.

## Current implementation

The code is in the `projects/social-content` Git submodule. Its working tree already contains changes to `cli.py`, `README.md`, and untracked topic/report artifacts. `search --min-score` currently defaults to `None`. `read_posts` hardcodes Latest search, caps reads at 100 candidates, deduplicates per read, and returns `reason`, `scrolls`, and `partial`. Its `partial` field describes fulfillment of the read limit, not daily coverage.

## CLI contract

Run from `projects/social-content`:

```bash
uv run python -m tools.twitter_agent report \
  --topics-file tools/twitter_agent/topics.json \
  --window-hours 24 \
  --per-topic 5 \
  --candidate-limit 100 \
  --output-dir .twitter-agent/reports
```

`--topics-file` defaults to `topics.json` next to the module, independent of working directory. `--window-hours` is a positive integer (default 24). `--candidate-limit` is an integer from 2 through 100 (default 100). `--per-topic` is an integer from 1 through candidate-limit (default 5). Ranking is fixed to engagement in this first version; no single-choice `--rank-by` flag is needed.

Global `--cdp`, `--timeout-ms`, and `--state-dir` retain existing meanings. When omitted, output-dir resolves to `<state-dir>/reports`. Each invocation creates a unique run directory containing `evidence.json` and `report.md`.

## Components and data flow

### `topic_config.py`

Load UTF-8 JSON and validate a nonempty topics array. Every topic requires a nonempty string `id`, a nonempty string `name`, and a nonempty list of nonempty string `search_keywords`. IDs must be unique. Optional category, summary, priority, and other metadata are preserved. Errors use `AgentError('invalid_topics_file', ...)` with a useful field path.

Keywords are literal discovery terms, not arbitrary search expressions. Preserve a balanced surrounding pair of double quotes; automatically quote unquoted multiword terms; reject embedded/unbalanced quotes, control characters, and operator-shaped unquoted terms containing `:` or parentheses or standalone AND/OR/NOT. Join with ` OR ` inside parentheses. Append `since_time:<start>` and `until_time:<end>` using integer epoch seconds. Use `search_keywords`, not CSV-style `watch_query`. Do not inherit link exclusions, language restrictions, or minimum likes from `watch`.

### `posts.py`

Extend `read_posts(..., *, search_mode='latest')` to support `latest` and `top`. Latest keeps `&f=live`; Top uses `&f=top`. Validate the option before navigation. Retain existing positional parameters, extraction, bounds, and result shape. Confirm Top behavior with a live smoke check when an authenticated browser is available; local tests verify URL construction and extraction independently.

### `report.py`

Own orchestration, report selection, JSON artifacts, and Markdown rendering. Receive an existing page; the CLI owns a single Browser context for the run.

Freeze `end = int(time.time())` once and derive start. For each topic, allocate `ceil(candidate_limit / 2)` candidates to Top and `floor(candidate_limit / 2)` to Latest. Do not refill budgets or add adaptive queries in version one. Each underlying read retains its existing scrolling/time bounds. The budget caps returned observations, not all DOM articles inspected.

Merge results by ID within each topic. For duplicates, retain the last observation in collection order and all contributing modes. Record an observation timestamp after each read. Preserve the same post under different matching topics. Do not sum metrics across duplicate observations or topics.

For each unique candidate, apply exclusions in order: ad, invalid/missing/non-timezone-aware ISO timestamp, outside interval. Store the exclusion reason alongside the evidence. Sort eligible candidates by descending metrics.total, descending parsed timestamp, then ascending ID for deterministic ties. Select up to per-topic posts. Keep all deduplicated candidate evidence, including exclusions, so the selection can be inspected offline.

### Report schema

Top-level fields: `schema_version` (1), `generated_at` (UTC completion time), `window` (UTC start/end), `config` (topic file path, window_hours, per_topic, candidate_limit, ranking='engagement'), `coverage` ('bounded_sample'), `partial` (execution degradation), `topics`, and `summary`.

Each topic contains `id`, `name`, optional descriptive metadata, `query`, `status`, `shortfall`, `searches`, `candidates`, `selected_ids`, and `counts`. A candidate retains the extracted post fields plus `observed_at`, `source_modes`, and `exclusion_reason` (null if eligible). Each search records mode, allocated limit, collected count, observation time, stop reason, scrolls, and a structured error or null. Counts include collected observations, unique candidates, duplicates, excluded_ads, excluded_invalid_timestamp, excluded_outside_window, eligible, and selected.

Statuses: `ok` (both searches succeeded, at least one eligible post), `empty` (both succeeded, no eligible posts), `partial` (one search failed but another succeeded), `error` (neither search succeeded), `skipped` (not attempted after a fatal browser/session error). A timeout stop reason also makes that topic and the run partial. A successful sparse result has shortfall=true but need not indicate execution failure. Candidate caps and no-progress stops are recorded without claiming exhaustive search.

Summary includes configured topic count, counts by status, selected topic entries, and unique selected post count across topics.

## Failure semantics

Validate flags and topic JSON before browser connection. Configuration errors exit 2 through the existing error envelope. Browser connection failures retain exit 3.

Within collection, catch `AgentError` and Playwright `Error` at each search boundary. Normalize Playwright errors to `browser_navigation_failed` with the original message. Continue after a recoverable search failure, recording it. Authentication/challenge/account-block/no-context/not-connected failures stop collection, retain completed evidence, and mark unattempted searches/topics skipped. A fatal failure on the final topic still sets partial=true. Do not suppress programming errors with a broad per-topic `except Exception`.

Successful artifact generation emits the normal `ok: true` envelope with paths and summary. Complete execution exits 0; a degraded run emits `partial: true` and exits 4, even when artifacts were saved. Explain that ok means artifacts produced, not all searches succeeded. Artifact I/O failure emits `report_write_failed`, exits 1, and identifies the run directory if created. Use a temporary directory under output-dir to guarantee unique run names; remove temporary files on failed atomic per-file writes. A partially written run directory is not claimed as complete.

## Markdown output

Include window, generation time, bounded-sample wording, execution status, coverage table, and every topic in configuration order. For selected posts include canonical URL, author, timestamp, observed metrics and observation time, and extracted text. Escape source text and topic labels so embedded HTML/Markdown cannot alter the report structure. Generate links from canonical post IDs rather than arbitrary text. Show search errors, shortfalls, and exclusions clearly. Empty and skipped sections must remain visible. Topic metadata may provide context but must not be presented as individualized analysis or generated reply recommendations.

## Verification and documentation

Use existing unittest and mock conventions. Test topic validation/query semantics, search navigation mode, time boundaries, deduplication and ranking, all-topic coverage, recoverable/fatal failures, saved evidence/Markdown consistency, unique outputs, and CLI exit codes. Keep live browser checks separate from automated tests. Update tool README usage and frontmatter plus test-directory README frontmatter. Run the project suite and repository changed-only frontmatter lint.
