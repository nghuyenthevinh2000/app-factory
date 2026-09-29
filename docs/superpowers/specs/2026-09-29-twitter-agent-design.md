# Supervised X DOM CLI design

## Goal and scope

Build a Python 3.12+ CLI under `projects/social-content/tools/twitter_agent/`
that an agent can use to read X and prepare replies on the user's behalf.
The user supervises through a visible Chrome tab and a separate interactive
terminal. Every reply requires explicit human approval before submission.

Use Playwright attached over CDP to a user-started Chrome instance with a
dedicated, manually authenticated profile. Use the social-content project's
existing `uv` environment. No LLM dependency is needed: the calling agent
supplies reply text. Reading and writing use DOM interactions, not private APIs.

## Architecture and alternatives

Playwright + CDP is selected for deterministic CLI operations, persistent
login, and visible browser interactions. A private-API client would depend on
changing endpoints and would not expose DOM interactions for supervision.
A browser extension relay would require an additional installed component.

Separate modules own the command interface, browser connection and selectors,
post extraction, persistent draft state, supervisor loop, and audit events.
Use SQLite transactions for state changes and a single supervisor process
for browser writes. Keep selectors centralized and scope them to the target
article or active composer dialog.

## Command interface

Run commands using `uv run python -m tools.twitter_agent` from social-content.

- `doctor`: check CDP connectivity, browser context, and X authentication.
- `timeline --limit N`: read loaded home timeline posts with bounded scrolling.
- `search QUERY --limit N`: read search results with bounded scrolling.
- `thread URL_OR_ID`: read the target post and loaded thread context.
- `reply prepare URL_OR_ID --text TEXT`: persist a pending draft and request
  browser preparation; never submit it.
- `queue FILE.jsonl`: import ordered drafts, one JSON object per line containing
  `target` and `text`. Each draft requires its own approval.
- `status`: report supervisor availability, active draft, pause state, and queue.
- `pause` / `resume`: control further submissions without approving drafts.
- `cancel DRAFT_ID`: cancel a pending draft.
- `supervise`: run the interactive human review loop and sole submission path.

The supervisor owns a dedicated visible X tab. Pending drafts are composed
one at a time when the supervisor is connected; a preparation request made
without it remains queued and clearly reports that browser preparation has
not occurred. Read commands use their own temporary tabs and do not disturb
the active review composer. Disconnecting the CLI does not close Chrome.

Agent commands emit structured JSON to stdout and readable progress to stderr.
Errors include a stable code, message, and whether human action is required.
Read results include post IDs, canonical URLs, author information, text,
timestamps where available, and explicit partial-result information. Thread
reads do not claim to return every reply or reconstruct invisible ancestors.

## Human approval and submission

The supervisor terminal shows the draft ID, target URL, fetched target text,
exact proposed reply, and screenshot path. The browser displays the target
and filled composer before the approval prompt. The user approves or rejects
the specific draft interactively. There is no agent-facing approval command,
automatic approval flag, or approval accepted through piped stdin.

Approval binds the canonical target ID and exact reply text, recorded with
a content digest. Immediately before clicking Submit, check cancellation,
pause state, limits, target identity, and the actual composer text. Any changed
target or text invalidates the review and requires a fresh approval. A paused
or deferred draft is presented again before submission.

This separates normal agent commands from human decisions; it is not an OS
security boundary against a process with the same user's terminal, browser,
and filesystem access. The agent's operating instructions must reserve the
supervisor terminal for the human.

Only the supervisor submits. Persist a `submitting` state before the click.
Click at most once per approved attempt. Confirm the result using concrete
browser evidence tied to that reply, capturing its published URL when available.
A closed composer alone is insufficient proof. An ambiguous outcome becomes
`uncertain` and requires manual inspection, without automatic retry.

## Persistence, queue, and controls

Draft states are `pending`, `reviewing`, `submitting`, `submitted`, `rejected`,
`cancelled`, `failed`, and `uncertain`. Store timestamps, target ID, exact text,
digest, outcome details, and artifact paths. Record approval and state changes
as append-only audit events. Duplicate target/text drafts are returned as
existing drafts rather than silently enqueued again.

Process drafts sequentially in FIFO order. Use an exclusive supervisor lock
and transactional state transitions to prevent competing submission workers.
On restart, return interrupted `reviewing` drafts to `pending` for fresh review;
convert interrupted `submitting` drafts to `uncertain`.

Default operational limits are 5 submission attempts per rolling hour, 30 per
rolling 24 hours, and at least 60 seconds between attempts. These are configurable
and persisted across CLI invocations. Count uncertain attempts toward limits.
Pacing is an operational throttle, not a guarantee against platform restrictions.
Pause and cancel are rechecked immediately before the submit click; neither
can undo a click already sent to the browser.

## Visibility and local artifacts

Emit timestamped events for navigation, extraction, preparation, human decision,
submission, confirmation, and errors. Include draft IDs and target URLs to
correlate live progress with audit history. Capture a screenshot before approval
and after submission, plus diagnostic screenshots for browser failures when
possible. A missing review screenshot is reported before approval.

Keep the SQLite database, audit data, and screenshots in an ignored local state
directory with user-only permissions where supported. Document that these
artifacts contain reply text and account-visible page content. Do not export
browser credentials or cookies into logs. Keep the Chrome profile outside the
repository.

## Browser and failure behavior

Use `data-testid` selectors and timestamp links to identify top-level posts;
avoid confusing quoted posts with their containing article. Normalize supported
`x.com` and `twitter.com` status URLs and numeric IDs before navigation. Preserve
textless/media-only posts. Bound navigation, selector waits, and scrolling.

Use real contenteditable input events through Playwright, then verify the
resulting text and submit button state. Login expiry, challenges, account locks,
rate limits, disabled composers, missing targets, or changed DOM produce clear
errors and halt the affected operation. Challenges require manual resolution
in Chrome. Do not retry submissions automatically.

## Verification

- Unit tests: URL normalization, exact-text approval binding, queue ordering,
  duplicate handling, pause/cancel transitions, persisted quotas, concurrent
  supervisor exclusion, and crash recovery.
- Local browser fixtures: DOM extraction including quoted and textless posts,
  contenteditable composition, target/text mismatch rejection, disabled submit,
  successful confirmation, and uncertain outcomes without a second click.
- CLI tests: JSON output, error exit codes, missing supervisor reporting, and
  refusal of non-interactive approval.
- Manual acceptance: start Chrome, log in, run doctor/read commands, prepare a
  draft, review it in the visible tab, reject it, then optionally approve a
  real reply under the user's direct supervision. Automated tests do not post
  to a real X account.
- Update directory README frontmatter and the root routing manifest as needed;
  run social-content's required changed-only frontmatter lint.

## Deliverables

CLI modules and tests under `projects/social-content/tools/twitter_agent/`,
Playwright dependency in the project's Python configuration and lockfile,
ignored runtime artifacts, and README documentation covering Chrome startup,
agent usage, the supervisor workflow, state recovery, and test commands.
