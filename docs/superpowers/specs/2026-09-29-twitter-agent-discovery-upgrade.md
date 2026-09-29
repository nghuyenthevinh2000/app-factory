# Twitter Agent — Opportunity Discovery Upgrade

## Context

This spec documents changes discovered during a live session operating
`tools/twitter_agent` to find reply candidates scoring above 80.
It extends the base design in `2026-09-29-twitter-agent-design.md`.

---

## Problem: Why the home timeline never breaks 80

The opportunity score formula is:

```
score = (age_factor × 40) + (competition_factor × 30) + (velocity_factor × 30)
```

When a post's `<time datetime="...">` attribute is absent or points to a
relative string the extractor cannot parse (e.g. `"2h"`), `age_factor`
defaults to `0.5`. This caps the maximum achievable score at:

```
0.5 × 40 + 1.0 × 30 + 1.0 × 30 = 80.0
```

The home timeline DOM does not reliably expose full ISO timestamps in timeline
cards, so scanning the timeline alone can never produce scores above 80.
The fix is to target fresh content via advanced search so that real ISO
timestamps are present and `age_factor` can reach 0.7 or 1.0.

---

## Fix 1 — DOM extraction: `isInsideQuote` bug

**File:** `tools/twitter_agent/posts.py` — `EXTRACT_ARTICLES_JS`

**Bug:** The original `isInsideQuote` guard treats any element with
`role="link"` as a quote boundary. The `<a>` anchor wrapping each tweet's
`<time>` element has `role="link"`, so the timestamp is incorrectly
suppressed and `post['timestamp']` becomes `null`.

**Fix:** Exclude anchor tags (`tagName !== 'A'`) from the role check.
Only non-anchor `role="link"` divs and `data-testid="quoteTweet"` elements
are true quote containers.

```js
// Before
if (parent.getAttribute('role') === 'link' || parent.getAttribute('data-testid') === 'quoteTweet') {

// After
if (parent.getAttribute('data-testid') === 'quoteTweet') {
    return true;
}
if (parent.getAttribute('role') === 'link' && parent.tagName !== 'A') {
    return true;
}
```

---

## Fix 2 — Scoring: ad detection and relative timestamp fallback

**File:** `tools/twitter_agent/posts.py` — `score_tweet` and `EXTRACT_ARTICLES_JS`

### Ad detection

Promoted posts score 0.0 regardless of engagement. Detect ads in the JS
extractor by checking whether any `<span>` inside the article has text
content exactly equal to `"Ad"` or `"Promoted"`. Expose the flag as
`is_ad: boolean` in the post object. In `score_tweet`, return `0.0`
immediately if `post.get('is_ad')` is truthy.

```js
const isAd = Array.from(article.querySelectorAll('span'))
  .some(el => el.textContent === 'Ad' || el.textContent === 'Promoted');
```

```python
if post.get('is_ad'):
    return 0.0
```

### Relative timestamp fallback

When the ISO `datetime` attribute is present but parsing fails, or when only
a relative string (e.g. `"3m"`, `"12s"`, `"2h"`, `"1d"`) is available,
derive an approximate post time before giving up:

```python
import re
m = re.fullmatch(r'([0-9]+)\s*([smhd])', ts_str)
if m:
    val, unit = int(m.group(1)), m.group(2)
    multipliers = {'s': 1, 'm': 60, 'h': 3600, 'd': 86400}
    post_time = now_time - (val * multipliers[unit])
```

---

## New CLI flags

### `search QUERY --window-minutes M --min-score S`

`--window-minutes M` automatically appends `since_time:<now - M*60>` to the
query before navigating. Agents do not need to compute Unix timestamps manually.

`--min-score S` filters the emitted posts list to only entries with
`score >= S`. Combined with `--window-minutes`, this lets any agent call:

```bash
uv run python -m tools.twitter_agent search \
  "(AI OR LLM OR claude) min_faves:5 lang:en -filter:links" \
  --window-minutes 60 --min-score 80 --limit 50
```

### `watch --topics "T1,T2,..." --window-minutes M --min-score S`

Adds a topic-based discovery mode alongside the existing watchlist mode.

When `--topics` is provided, each comma-separated topic string is issued as
an independent advanced search query (with `--window-minutes` and the fixed
`min_faves:5 lang:en -filter:links` suffix appended). Results are merged,
deduplicated by post ID, sorted by score descending, and truncated to
`--limit`.

When `--topics` is absent, the existing watchlist `from:HANDLE` behaviour
is unchanged.

Default topics (used when `--topics` is passed with no value):

```
AI, LLM, claude, cursor, openai, gemini, engineering,
"just shipped", "just launched", "I built", startup, founder
```

---

## X advanced search operators — reference

The `search` command passes its query string verbatim to
`x.com/search?q=<encoded>&f=live`. All standard X advanced search operators
work. Future agents must use them to control result quality and avoid the
score ceiling caused by missing timestamps.

| Operator            | Effect                                               | Example                      |
|---------------------|------------------------------------------------------|------------------------------|
| `since_time:<unix>` | Only posts after this Unix timestamp                 | `since_time:1790692388`      |
| `until_time:<unix>` | Only posts before this Unix timestamp                | `until_time:1790695988`      |
| `since:YYYY-MM-DD`  | Only posts on or after this date (UTC day boundary)  | `since:2026-09-29`           |
| `until:YYYY-MM-DD`  | Only posts before this date                          | `until:2026-09-30`           |
| `min_faves:N`       | Minimum like count                                   | `min_faves:50`               |
| `min_replies:N`     | Minimum reply count                                  | `min_replies:5`              |
| `min_retweets:N`    | Minimum retweet count                                | `min_retweets:10`            |
| `lang:en`           | English posts only                                   | `AI lang:en`                 |
| `-filter:links`     | Exclude posts containing external URLs               | `AI -filter:links`           |
| `filter:media`      | Only posts with images, video, or GIFs               | `startup filter:media`       |
| `-filter:replies`   | Exclude reply posts (top-level only)                 | `claude -filter:replies`     |
| `from:HANDLE`       | Posts from a specific account                        | `from:swyx`                  |
| `to:HANDLE`         | Posts replying to a specific account                 | `to:sama`                    |
| `OR`                | Match either term                                    | `claude OR openai`           |
| `"exact phrase"`    | Exact phrase match                                   | `"context window"`           |
| `-term`             | Exclude term                                         | `AI -crypto`                 |

### Why `since_time` beats `since:`

`since:YYYY-MM-DD` has day-level granularity. `since_time:<unix>` is
second-level and is the only way to reliably restrict results to posts
younger than 15 minutes — which is the threshold for `age_factor >= 0.7`
and scores above 80.

### Canonical pattern that consistently yields score > 80

```bash
# Compute since_time = now - 3600 (1-hour window) before calling:
uv run python -m tools.twitter_agent search \
  "AI min_faves:5 since_time:$(python3 -c 'import time; print(int(time.time()-3600))') lang:en -filter:links" \
  --limit 50 --min-score 80
```

Or using the new convenience flag once implemented:

```bash
uv run python -m tools.twitter_agent search \
  "AI min_faves:5 lang:en -filter:links" \
  --window-minutes 60 --min-score 80 --limit 50
```

---

## Tests to add

- `score_tweet` returns `0.0` for a post with `is_ad: True`.
- `score_tweet` correctly parses `"3m"` relative timestamp and assigns
  `age_factor = 1.0` when called within 3 minutes of the derived post time.
- `isInsideQuote` does not suppress the `<a role="link"><time>` pattern.
- `search --window-minutes 20` appends `since_time:` to the query string
  before navigation (assert the navigated URL contains `since_time:`).
- `search --min-score 80` emits only posts with `score >= 80`.
- `watch --topics "AI,LLM"` performs two search queries (one per topic),
  merges results, deduplicates, and returns at most `--limit` posts sorted
  by score descending.
