# Twitter Agent Discovery Upgrade Implementation Plan

## Context
Implement the improvements specified in `docs/superpowers/specs/2026-09-29-twitter-agent-discovery-upgrade.md`.
This upgrade resolves the score ceiling issue on X by fixing DOM timestamp extraction inside anchor tags, adding ad detection and relative timestamp parsing in `score_tweet`, and introducing advanced search `--window-minutes` and `--min-score` filters alongside a multi-topic `--topics` discovery mode in `watch`.

---

## Proposed Changes

### 1. `projects/social-content/tools/twitter_agent/posts.py`
- In `EXTRACT_ARTICLES_JS`:
  - Fix `isInsideQuote`: exclude anchor tags (`tagName !== 'A'`) from `role="link"` check so `<a role="link"><time>` does not suppress timestamps.
  - Detect ads: scan `span` text for `'Ad'` or `'Promoted'`, set `is_ad: boolean` on post objects.
- In `score_tweet`:
  - Return `0.0` immediately if `post.get('is_ad')` is truthy.
  - Relative timestamp fallback: if ISO parsing fails or relative string (e.g. `"3m"`, `"12s"`, `"2h"`, `"1d"`), parse via `re.fullmatch(r'([0-9]+)\s*([smhd])', ts_str)` and compute `post_time = now_time - (val * multipliers[unit])`.

### 2. `projects/social-content/tools/twitter_agent/cli.py`
- In `search` subcommand:
  - Add `--window-minutes` (`int`, optional): automatically appends `since_time:<now - M*60>` to query before calling `read_posts`.
  - Add `--min-score` (`float`, optional): filters result posts to `post['score'] >= min_score`.
- In `watch` subcommand:
  - Add `--topics` (`str`, optional, `nargs='?'`, `const=DEFAULT_TOPICS`): enables topic-based discovery mode.
  - Add `--window-minutes` (`int`, optional).
  - Add `--min-score` (`float`, optional).
  - In handler:
    - If `--topics` provided: query each topic via search mode with `--window-minutes` and fixed `min_faves:5 lang:en -filter:links` suffix. Merge results, deduplicate by ID, sort descending by score, filter by `--min-score`, truncate to `--limit`.
    - If `--topics` not provided: execute existing watchlist discovery.

### 3. Tests
- `tools/twitter_agent/tests/test_dom.py`:
  - `test_is_inside_quote_preserves_anchor_time`: verify `<a role="link"><time>` is extracted.
  - `test_score_tweet_ad_detection_zero`: verify `is_ad: True` returns 0.0.
  - `test_score_tweet_relative_timestamp`: verify `"3m"` derives appropriate age factor.
- `tools/twitter_agent/tests/test_cli.py`:
  - `test_search_window_minutes_and_min_score`: verify `since_time:` injected and posts filtered by score.
  - `test_watch_topics_discovery`: verify topic queries merged, deduplicated, and ranked.

---

## Tasks

### Task 1: DOM Extractor Fix & Ad Detection
**Files:**
- Modify: `projects/social-content/tools/twitter_agent/posts.py`
- Test: `projects/social-content/tools/twitter_agent/tests/test_dom.py`

- [ ] **Step 1: Write failing tests in `test_dom.py`**
  - Add test asserting that `<a role="link" href="/status/123"><time datetime="...">` extracts the timestamp (not null).
  - Add test asserting that an article containing a span with text `"Ad"` or `"Promoted"` has `is_ad: True`.
- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Update `EXTRACT_ARTICLES_JS` in `posts.py`**
  - Adjust `isInsideQuote` condition:
    ```js
    if (parent.getAttribute('data-testid') === 'quoteTweet') {
        return true;
    }
    if (parent.getAttribute('role') === 'link' && parent.tagName !== 'A') {
        return true;
    }
    ```
  - Add `isAd` boolean check:
    ```js
    const isAd = Array.from(article.querySelectorAll('span'))
        .some(el => el.textContent === 'Ad' || el.textContent === 'Promoted');
    ```
  - Include `is_ad: isAd` in the returned post object.
- [ ] **Step 4: Run tests to verify they pass**

---

### Task 2: Scoring Upgrade: Ad Penalization and Relative Timestamp Fallback
**Files:**
- Modify: `projects/social-content/tools/twitter_agent/posts.py`
- Test: `projects/social-content/tools/twitter_agent/tests/test_dom.py`

- [ ] **Step 1: Write failing tests in `test_dom.py`**
  - Test `score_tweet({'is_ad': True, ...}) == 0.0`.
  - Test `score_tweet({'timestamp': '3m', ...}, now=now_ts) > 75.0` (3 minutes old yields age_factor 1.0).
  - Test `score_tweet({'timestamp': '2h', ...}, now=now_ts) < 40.0` (2 hours old yields age_factor 0.1).
- [ ] **Step 2: Run tests to verify failure**
- [ ] **Step 3: Implement changes in `score_tweet`**
  - Check `if post.get('is_ad'): return 0.0`.
  - In timestamp parsing: if ISO parsing fails, check regex `re.fullmatch(r'([0-9]+)\s*([smhd])', str(ts_str).strip())`.
- [ ] **Step 4: Run tests to verify they pass**

---

### Task 3: Search CLI Upgrade: `--window-minutes` and `--min-score`
**Files:**
- Modify: `projects/social-content/tools/twitter_agent/cli.py`
- Test: `projects/social-content/tools/twitter_agent/tests/test_cli.py`

- [ ] **Step 1: Write failing tests in `test_cli.py`**
  - Test `twitter_agent search "AI" --window-minutes 60` appends `since_time:` to query.
  - Test `twitter_agent search "AI" --min-score 80` filters results where `score < 80`.
- [ ] **Step 2: Run tests to verify failure**
- [ ] **Step 3: Implement `--window-minutes` and `--min-score` in `cli.py`**
- [ ] **Step 4: Run tests to verify they pass**

---

### Task 4: Watch CLI Upgrade: Topic-Based Discovery
**Files:**
- Modify: `projects/social-content/tools/twitter_agent/cli.py`
- Test: `projects/social-content/tools/twitter_agent/tests/test_cli.py`

- [ ] **Step 1: Write failing tests in `test_cli.py`**
  - Test `watch --topics "AI,LLM" --min-score 70` performs topic search, merges, deduplicates, and filters by score.
- [ ] **Step 2: Run tests to verify failure**
- [ ] **Step 3: Implement topic discovery in `watch` handler**
  - Support `DEFAULT_TOPICS`:
    `'AI, LLM, claude, cursor, openai, gemini, engineering, "just shipped", "just launched", "I built", startup, founder'`
  - Issue queries per topic with `min_faves:5 lang:en -filter:links` and `since_time:`.
  - Merge, deduplicate by ID, sort descending by score, filter by `--min-score`, truncate to `--limit`.
- [ ] **Step 4: Run tests to verify they pass**

---

### Task 5: Documentation & System Verification
**Files:**
- Modify: `projects/social-content/tools/twitter_agent/README.md`
- Test: Full test suite & README frontmatter linter

- [ ] **Step 1: Update README.md with new CLI flags and examples**
- [ ] **Step 2: Run full test suite (`uv run python -m unittest discover -s tools/twitter_agent/tests -v`)**
- [ ] **Step 3: Run README linter (`uv run python .agents/hooks/file-selector/lint_readme_tree.py --changed-only`)**
