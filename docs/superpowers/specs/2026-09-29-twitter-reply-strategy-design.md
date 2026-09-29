# Supervised X Reply Strategy Enhancement Design

## Goal and Scope

Enhance `tools/twitter_agent` with reply-guy growth strategy capabilities from `reflections/twitter-reply-guy-strategy-2026.md`:

1. **Safety & Pacing:** Align operational limits with platform safety norms (`daily=20`, `hourly=6`, `spacing=120.0s`, and `burst_max=3` per rolling 30-min window).
2. **Target Watchlist:** Store monitored accounts in SQLite with CLI management commands (`watchlist add`, `watchlist remove`, `watchlist list`).
3. **Opportunity Scoring:** Score posts based on post age (< 5 min peak, > 30 min drop), reply competition (< 20 replies), and engagement velocity.
4. **Target Discovery:** Add a `watch` command that scans watched accounts for fresh, high-signal tweets.
5. **Supervisor Strategy HUD:** Display tweet age, reply count, watchlist status, and deboosting risk in the interactive review terminal.
6. **Agent Strategy Skill:** Author `.agents/skills/twitter-reply-strategy/SKILL.md` to guide LLM agents on target selection and crafting high-impact replies, keeping the tool unconstrained.

## Architecture and Components

### 1. Limits & Burst Protection (`models.py`, `store.py`)

- Updated `Limits` dataclass defaults:

  ```python
  @dataclass(frozen=True)
  class Limits:
      hourly: int = 6
      daily: int = 20
      spacing: float = 120.0
      burst_max: int = 3  # Max attempts within any rolling 30-minute window
  ```

- In `store.py` (`begin_submission`):
  Query `events` for `submission_started` within `now - 1800` (30 minutes). If count >= `limits.burst_max`, raise `AgentError('rate_limited', 'Burst limit reached (max 3 replies per 30 minutes).')`.

### 2. Watchlist Storage (`store.py`)

- SQLite table:

  ```sql
  CREATE TABLE IF NOT EXISTS watchlist (
      handle TEXT PRIMARY KEY,
      notes TEXT NOT NULL,
      created_at REAL NOT NULL
  );
  ```

- Store methods:
  - `add_watchlist(handle: str, notes: str = '') -> dict`: Strip leading `@`, validate non-empty ASCII handle, insert or replace into `watchlist`.
  - `remove_watchlist(handle: str) -> bool`: Strip leading `@`, delete from `watchlist`, return True if deleted.
  - `get_watchlist() -> list[dict]`: Return all rows ordered by `handle` ascending.

### 3. Tweet Scorer (`posts.py`)

- Function: `score_tweet(post: dict, now: Optional[float] = None) -> float`:
  - Input: extracted post dictionary with `metrics` and optional `timestamp`.
  - Factors:
    - **Age Factor (0.0 to 1.0, weight 40%):**
      If timestamp available, calculate `age_minutes = (now - post_timestamp) / 60`.
      If `age_minutes <= 5`: factor = 1.0.
      Else if `age_minutes <= 15`: factor = 0.7.
      Else if `age_minutes <= 30`: factor = 0.4.
      Else: factor = 0.1.
      If timestamp is None, default factor = 0.5.
    - **Competition Factor (0.0 to 1.0, weight 30%):**
      `replies = post.get('metrics', {}).get('replies', 0)`.
      If `replies < 10`: factor = 1.0.
      Else if `replies <= 20`: factor = 0.7.
      Else if `replies <= 50`: factor = 0.3.
      Else: factor = 0.05.
    - **Velocity Factor (0.0 to 1.0, weight 30%):**
      `total = post.get('metrics', {}).get('total', 0)`.
      Log-scaled normalized velocity score based on interactions.
  - Total score is normalized from 0 to 100.
  - Attach `score` to each post returned by `read_posts` and `watch`.

### 4. Discovery Command (`cli.py`, `posts.py`)

- `twitter_agent watch [--limit N] [--poll SECONDS]`:
  - Fetches watchlist from store. If empty, returns clean message/data indicating no watched accounts.
  - For each handle in watchlist, executes a search for `from:<handle>` (filtering for top-level tweets).
  - Collects posts, deduplicates by ID, scores each post via `score_tweet`.
  - Sorts descending by score.
  - Emits JSON `{ "ok": true, "data": { "opportunities": [...] } }`.
  - If `--poll <seconds>` is specified, loops with sleep interval, printing new opportunities as they appear.

- `twitter_agent watchlist`:
  - `watchlist add <handle> [--notes <notes>]`: Add account to watchlist.
  - `watchlist remove <handle>`: Remove account from watchlist.
  - `watchlist list`: List all watched accounts.

### 5. Supervisor Strategy HUD (`supervisor.py`)

- During `supervise`, when presenting draft for review:
  - Check if target author handle exists in `watchlist`: display `[WATCHLIST TARGET]` indicator.
  - Calculate post age if timestamp available and display: `Post Age: Xm (Window: <5m recommended)`.
  - Display `Current Replies: N (Saturation: <20 recommended)`.
  - Calculate rolling 30-min submission count: if count >= 2, display warning `⚠️ Deboost Risk: 2 replies in last 30m. Consider pacing.`

### 6. Agent Strategy Skill (`.agents/skills/twitter-reply-strategy/SKILL.md`)

- Create skill document for agents:
  - **Philosophy:** Replies as an audience-borrowing mechanism, not vanity commenting.
  - **Target Filtering Rules:**
    - Substantive topics (engineering, tech, product, economics, contrarian viewpoints).
    - Fresh posts (< 5 min optimal, never > 30 min).
    - Low competition (< 20 replies).
  - **Creative Angles Playbook:**
    - Contrarian: Respectful, reasoned dissent with logic.
    - Data: Specific metrics, benchmarks, or cited evidence.
    - First-Principles Question: Advancing the conversation.
    - Analogy: Cross-domain technical mapping.
    - Concrete Experience: Hard-won production / build realities.
  - **Hard Constraints:**
    - Never post generic agreement ("This!", "Great post").
    - No external links (X deboost trigger).
    - No sycophantic tone or generic AI phrasing.
  - **Workflow instructions:** Discover -> Select -> Angle -> Craft -> Enqueue via `tools.twitter_agent reply prepare`.

## Verification and Testing

1. **Unit Tests:**
   - `test_models.py`: New `burst_max` validation and default Limits values.
   - `test_store.py`: Watchlist CRUD tests, burst rate limit boundary tests.
   - `test_dom.py`: Scorer algorithm tests across various age and reply count fixtures.
   - `test_cli.py`: Subprocess tests for `watchlist add/remove/list` and `watch`.
2. **Regression:**
   - Full 45-test suite must continue passing with 100% success.
3. **Skill Lint:**
   - Frontmatter check with repository hooks.
