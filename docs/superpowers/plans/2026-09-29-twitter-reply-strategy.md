# Twitter Reply Strategy Enhancement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enhance the supervised X DOM CLI with reply-guy growth strategy limits, watchlist management, opportunity scoring, a watch discovery command, supervisor HUD metrics, and author an agent skill for crafting replies.

**Architecture:** Extend `Limits` and SQLite with burst tracking and a `watchlist` table. Add a heuristic tweet opportunity scorer in `posts.py` and discovery `watch` command in `cli.py`. Display strategy context in `supervisor.py`. Author `.agents/skills/twitter-reply-strategy/SKILL.md` to guide LLM agents without hardcoding rigid enums into the CLI.

**Tech Stack:** Python 3.12+, Playwright, SQLite, unittest, uv.

**Spec:** `docs/superpowers/specs/2026-09-29-twitter-reply-strategy-design.md`

## Global Constraints

- Python 3.12+; work lives under `projects/social-content/tools/twitter_agent/` and `.agents/skills/twitter-reply-strategy/`.
- All automated tests continue running offline without posting to real X.
- Maintain existing 100% backward compatibility for existing subcommands.
- Keep the tool unconstrained by enums: reply text is recorded as-is, leaving creative angles to the agent skill.
- Leave all git changes uncommitted unless explicitly instructed.

---

### Task 1: Strategy-Aligned Rate Limits and 30-Minute Burst Guard

**Files:**
- Modify: `projects/social-content/tools/twitter_agent/models.py`
- Modify: `projects/social-content/tools/twitter_agent/store.py`
- Test: `projects/social-content/tools/twitter_agent/tests/test_models.py`
- Test: `projects/social-content/tools/twitter_agent/tests/test_store.py`

**Interfaces:**
- Consumes: `Limits(hourly=6, daily=20, spacing=120.0, burst_max=3)`.
- Produces: Burst limit enforcement preventing more than 3 attempts in any rolling 30-minute window.

- [ ] **Step 1: Write failing test in `test_models.py` and `test_store.py`**

In `test_models.py`:
```python
    def test_limits_defaults_and_burst_validation(self):
        limits = Limits()
        self.assertEqual(limits.hourly, 6)
        self.assertEqual(limits.daily, 20)
        self.assertEqual(limits.spacing, 120.0)
        self.assertEqual(limits.burst_max, 3)

        with self.assertRaises(AgentError) as ctx:
            Limits(burst_max=0)
        self.assertEqual(ctx.exception.code, 'invalid_limits')
```

In `test_store.py`:
```python
    def test_burst_limit_blocks_fourth_attempt_in_thirty_minutes(self):
        self.now = 100_000.0
        # Configure spacing to 0 to test burst window independently
        self.store.configure_limits(Limits(hourly=10, daily=50, spacing=0, burst_max=3))
        for i in range(3):
            draft = self.store.enqueue(normalize_target(f'{100 + i}'), f'Reply {i}')
            self.store.claim_next()
            self.store.begin_submission(draft['id'], draft['digest'])
            self.now += 300  # 5 minutes apart

        # 4th attempt at 15 minutes (within 30m window) must be blocked
        draft4 = self.store.enqueue(normalize_target('104'), 'Reply 4')
        self.store.claim_next()
        with self.assertRaises(AgentError) as ctx:
            self.store.begin_submission(draft4['id'], draft4['digest'])
        self.assertEqual(ctx.exception.code, 'rate_limited')
        self.assertIn('Burst', ctx.exception.message)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_models.py' -v`
Expected: FAIL (missing `burst_max` attribute).

- [ ] **Step 3: Update `Limits` in `models.py` and `store.py`**

In `models.py`:
```python
@dataclass(frozen=True)
class Limits:
    hourly: int = 6
    daily: int = 20
    spacing: float = 120.0
    burst_max: int = 3

    def __post_init__(self):
        if (type(self.hourly) is not int or self.hourly <= 0
                or type(self.daily) is not int or self.daily <= 0
                or type(self.spacing) not in (int, float)
                or not math.isfinite(self.spacing) or self.spacing < 0
                or type(self.burst_max) is not int or self.burst_max <= 0):
            raise AgentError('invalid_limits', 'Limits need positive integer quotas, finite nonnegative spacing, and positive burst_max.')
```

In `store.py` (`begin_submission`):
```python
            attempts = db.execute('''SELECT
                COUNT(CASE WHEN created_at > ? THEN 1 END) AS burst,
                COUNT(CASE WHEN created_at > ? THEN 1 END) AS hourly,
                COUNT(CASE WHEN created_at > ? THEN 1 END) AS daily,
                MAX(created_at) AS latest
                FROM events WHERE kind = 'submission_started' ''',
                                   (now - 1800, now - 3600, now - 86400)).fetchone()
            if (attempts['hourly'] >= limits.hourly or attempts['daily'] >= limits.daily
                    or (attempts['latest'] is not None and now - attempts['latest'] < limits.spacing)):
                raise AgentError('rate_limited', 'Submission attempt quota or minimum spacing reached.')
            if attempts['burst'] >= limits.burst_max:
                raise AgentError('rate_limited', f'Burst limit reached (max {limits.burst_max} replies per 30 minutes).')
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_models.py' -v`
Run: `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_store.py' -v`
Expected: PASS.

---

### Task 2: Watchlist Storage and CLI Commands

**Files:**
- Modify: `projects/social-content/tools/twitter_agent/store.py`
- Modify: `projects/social-content/tools/twitter_agent/cli.py`
- Test: `projects/social-content/tools/twitter_agent/tests/test_store.py`
- Test: `projects/social-content/tools/twitter_agent/tests/test_cli.py`

**Interfaces:**
- Consumes: `store.add_watchlist(handle, notes)`, `store.remove_watchlist(handle)`, `store.get_watchlist()`.
- Produces: CLI subcommands `twitter_agent watchlist add|remove|list`.

- [ ] **Step 1: Write failing tests in `test_store.py` and `test_cli.py`**

In `test_store.py`:
```python
    def test_watchlist_crud(self):
        added = self.store.add_watchlist('@techopsasia', notes='Frontier Club founder')
        self.assertEqual(added['handle'], 'techopsasia')
        self.assertEqual(added['notes'], 'Frontier Club founder')

        items = self.store.get_watchlist()
        self.assertEqual(len(items), 1)
        self.assertEqual(items[0]['handle'], 'techopsasia')

        # remove
        self.assertTrue(self.store.remove_watchlist('techopsasia'))
        self.assertFalse(self.store.remove_watchlist('techopsasia'))
        self.assertEqual(len(self.store.get_watchlist()), 0)
```

In `test_cli.py`:
```python
    def test_watchlist_cli_commands(self):
        # Add
        add_res = subprocess.run(
            self.base_cmd + ['watchlist', 'add', '@sama', '--notes', 'OpenAI'],
            capture_output=True, text=True, check=True
        )
        self.assertTrue(json.loads(add_res.stdout)['ok'])

        # List
        list_res = subprocess.run(
            self.base_cmd + ['watchlist', 'list'],
            capture_output=True, text=True, check=True
        )
        data = json.loads(list_res.stdout)['data']
        self.assertEqual(len(data['watchlist']), 1)
        self.assertEqual(data['watchlist'][0]['handle'], 'sama')

        # Remove
        rem_res = subprocess.run(
            self.base_cmd + ['watchlist', 'remove', 'sama'],
            capture_output=True, text=True, check=True
        )
        self.assertTrue(json.loads(rem_res.stdout)['ok'])
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_store.py' -v`
Expected: FAIL (`AttributeError: 'Store' object has no attribute 'add_watchlist'`).

- [ ] **Step 3: Implement watchlist in `store.py` and `cli.py`**

In `store.py`:
- In `__init__`: `CREATE TABLE IF NOT EXISTS watchlist (handle TEXT PRIMARY KEY, notes TEXT NOT NULL, created_at REAL NOT NULL)`
- Implement `add_watchlist`, `remove_watchlist`, `get_watchlist`:
```python
    def add_watchlist(self, handle: str, notes: str = '') -> dict:
        clean = handle.strip().lstrip('@').lower()
        if not clean or not clean.isalnum() and '_' not in clean:
            raise AgentError('invalid_handle', f'Invalid X handle: {handle}')
        with self._transaction() as db:
            now = self.clock()
            db.execute('INSERT OR REPLACE INTO watchlist (handle, notes, created_at) VALUES (?, ?, ?)',
                       (clean, str(notes).strip(), now))
            row = db.execute('SELECT * FROM watchlist WHERE handle = ?', (clean,)).fetchone()
            return dict(row)

    def remove_watchlist(self, handle: str) -> bool:
        clean = handle.strip().lstrip('@').lower()
        with self._transaction() as db:
            cursor = db.execute('DELETE FROM watchlist WHERE handle = ?', (clean,))
            return cursor.rowcount > 0

    def get_watchlist(self) -> list[dict]:
        with self._connection() as db:
            rows = db.execute('SELECT * FROM watchlist ORDER BY handle ASC').fetchall()
            return [dict(r) for r in rows]
```

In `cli.py`:
Add `watchlist` parser with `add`, `remove`, `list` subparsers, and handle dispatch.

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_store.py' -v`
Run: `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_cli.py' -v`
Expected: PASS.

---

### Task 3: Tweet Opportunity Scoring in `posts.py`

**Files:**
- Modify: `projects/social-content/tools/twitter_agent/posts.py`
- Test: `projects/social-content/tools/twitter_agent/tests/test_dom.py`

**Interfaces:**
- Consumes: post dictionary with metrics and timestamp.
- Produces: `score_tweet(post: dict, now: Optional[float] = None) -> float` attached to each post in `read_posts`.

- [ ] **Step 1: Write failing test in `test_dom.py`**

```python
    def test_tweet_scoring_algorithm(self):
        from tools.twitter_agent.posts import score_tweet
        # Fresh post (< 5 min), low replies (< 10), high velocity
        fresh_post = {
            'timestamp': '2026-09-29T20:56:00Z',
            'metrics': {'replies': 3, 'retweets': 20, 'likes': 100, 'total': 123}
        }
        # Reference now = 2026-09-29T21:00:00Z (4 min old)
        now_ts = 1790715600.0  # Unix timestamp corresponding to 21:00:00Z
        fresh_post_ts = 1790715360.0 # 20:56:00Z (4 min earlier)

        score_fresh = score_tweet(fresh_post, now=now_ts, post_timestamp_override=fresh_post_ts)
        self.assertGreater(score_fresh, 75.0)

        # Stale post (> 45 min), saturated replies (> 100)
        stale_post = {
            'timestamp': '2026-09-29T20:00:00Z',
            'metrics': {'replies': 150, 'retweets': 200, 'likes': 500, 'total': 850}
        }
        stale_post_ts = 1790712000.0 # 60 min earlier
        score_stale = score_tweet(stale_post, now=now_ts, post_timestamp_override=stale_post_ts)
        self.assertLess(score_stale, 40.0)
        self.assertGreater(score_fresh, score_stale)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_dom.py' -v`
Expected: FAIL (`ImportError: cannot import name 'score_tweet' from 'tools.twitter_agent.posts'`).

- [ ] **Step 3: Implement `score_tweet` in `posts.py` and attach to extracted posts**

```python
def score_tweet(post: dict, now: Optional[float] = None, post_timestamp_override: Optional[float] = None) -> float:
    now_time = now if now is not None else time.time()
    
    # 1. Age factor (weight 40)
    post_time = post_timestamp_override
    if post_time is None and post.get('timestamp'):
        try:
            # Parse ISO or datetime
            from datetime import datetime, timezone
            dt = datetime.fromisoformat(post['timestamp'].replace('Z', '+00:00'))
            post_time = dt.timestamp()
        except Exception:
            post_time = None

    if post_time is not None:
        age_minutes = max(0.0, (now_time - post_time) / 60.0)
        if age_minutes <= 5.0:
            age_factor = 1.0
        elif age_minutes <= 15.0:
            age_factor = 0.7
        elif age_minutes <= 30.0:
            age_factor = 0.4
        else:
            age_factor = 0.1
    else:
        age_factor = 0.5

    # 2. Competition factor (weight 30)
    metrics = post.get('metrics', {})
    replies = metrics.get('replies', 0)
    if replies < 10:
        comp_factor = 1.0
    elif replies <= 20:
        comp_factor = 0.7
    elif replies <= 50:
        comp_factor = 0.3
    else:
        comp_factor = 0.05

    # 3. Velocity / engagement factor (weight 30)
    total = metrics.get('total', 0)
    # Log-scaled engagement
    velocity_factor = min(1.0, math.log10(max(1, total) + 1) / 3.0)

    total_score = (age_factor * 40.0) + (comp_factor * 30.0) + (velocity_factor * 30.0)
    return round(total_score, 1)
```
In `read_posts`, compute and attach `'score'` to each post before returning.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_dom.py' -v`
Expected: PASS.

---

### Task 4: Watch Discovery Command and Supervisor Strategy HUD

**Files:**
- Modify: `projects/social-content/tools/twitter_agent/cli.py`
- Modify: `projects/social-content/tools/twitter_agent/supervisor.py`
- Test: `projects/social-content/tools/twitter_agent/tests/test_cli.py`
- Test: `projects/social-content/tools/twitter_agent/tests/test_supervisor.py`

**Interfaces:**
- Consumes: `store.get_watchlist()`, `read_posts(page, 'search', query, limit)`.
- Produces: `twitter_agent watch [--limit N] [--poll SECONDS]`, supervisor HUD with watchlist and deboost warning.

- [ ] **Step 1: Write failing test in `test_cli.py` and `test_supervisor.py`**

In `test_cli.py`:
```python
    def test_watch_command_empty_watchlist(self):
        res = subprocess.run(
            self.base_cmd + ['watch'],
            capture_output=True, text=True, check=True
        )
        data = json.loads(res.stdout)['data']
        self.assertEqual(data['opportunities'], [])
        self.assertIn('Watchlist is empty', data['message'])
```

In `test_supervisor.py`:
```python
    def test_supervisor_hud_displays_strategy_metrics(self):
        self.store.add_watchlist('alice', 'Core target')
        draft = self.store.enqueue(normalize_target('123'), 'Test reply')
        captured_stderr = []
        with patch('tools.twitter_agent.supervisor.prepare_reply', return_value={'target_id': '123', 'target_author': '@alice', 'target_url': 'https://x.com/i/status/123'}), \
             patch('sys.stderr.write', side_effect=captured_stderr.append):
            # Test default prompt rendering
            from tools.twitter_agent.supervisor import _render_strategy_hud
            hud_text = _render_strategy_hud(self.store, {'target_author': '@alice', 'target_id': '123'})
            self.assertIn('WATCHLIST TARGET', hud_text)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_cli.py' -v`
Expected: FAIL (`unrecognized arguments: watch`).

- [ ] **Step 3: Implement `watch` in `cli.py` and strategy HUD in `supervisor.py`**

In `cli.py`:
- Add `watch` parser with `--limit` (default 10) and `--poll` (int, default None).
- Handler:
  Reads watchlist handles. If empty, emit `{ "ok": true, "data": { "opportunities": [], "message": "Watchlist is empty. Add accounts with watchlist add <handle>." } }`.
  Connects browser, searches `from:<handle>`, extracts posts, ranks by `score`, returns top opportunities.
  If `--poll` specified, loops every `poll` seconds.

In `supervisor.py`:
- In `default_prompt(info)`:
  Check if author is in watchlist -> display `[★ WATCHLIST TARGET]`.
  Calculate rolling 30-min submission count -> if >= 2, display `[⚠️ DEBOOST WARNING: High burst frequency]`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_cli.py' -v`
Run: `uv run python -m unittest discover -s tools/twitter_agent/tests -p 'test_supervisor.py' -v`
Expected: PASS.

---

### Task 5: Agent Strategy Skill and Final System Verification

**Files:**
- Create: `.agents/skills/twitter-reply-strategy/SKILL.md`
- Modify: `projects/social-content/tools/twitter_agent/README.md`
- Test: Full test suite and lint hooks

**Interfaces:**
- Produces: Agent skill `.agents/skills/twitter-reply-strategy/SKILL.md` with complete playbook for crafting replies.

- [ ] **Step 1: Write `.agents/skills/twitter-reply-strategy/SKILL.md`**

Include:
- Frontmatter with `name: twitter-reply-strategy` and description.
- Core philosophy: 70/30 reply rule for organic reach without ad spend.
- Sweet-spot filtering: account size 5-20x, post age < 5 min, replies < 20.
- Creative angle playbook: Data & Metrics, Respectful Contrarian, First-Principles Question, Analogy, Concrete Experience.
- Strict anti-patterns: no sycophantic agreement, no links, no generic AI phrasing.
- Workflow instructions using `twitter_agent watch` -> select -> draft -> `twitter_agent reply prepare`.

- [ ] **Step 2: Update `tools/twitter_agent/README.md`**

Document new `watchlist` and `watch` commands and burst limit defaults.

- [ ] **Step 3: Run full verification suite and linter**

```bash
uv run python -m unittest discover -s tools/twitter_agent/tests -v
uv run python -m tools.twitter_agent --help
uv run --project . .agents/hooks/file-selector/lint_readme_tree.py --changed-only
```
Ensure all tests pass and linter reports compliance.
