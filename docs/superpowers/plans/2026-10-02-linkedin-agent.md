# LinkedIn Image-Post Agent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish pre-approved text and images to the user's personal LinkedIn account through a visible Chrome session, without a supervisor command.

**Architecture:** An independent Python CLI connects to Chrome over CDP. Input validation snapshots image bytes before navigation; scoped DOM operations prepare and verify the composer, then submit once and distinguish confirmed from uncertain outcomes.

**Tech Stack:** Python >=3.12, existing Playwright >=1.50,<2, argparse, unittest, Bash.

**Spec:** `docs/superpowers/specs/2026-10-02-linkedin-agent-design.md`

## Global Constraints

- Reuse `$HOME/chrome-twitter-profile` and CDP at `http://127.0.0.1:9222` by default.
- No supervisor, queue, approval prompt, or draft-generation feature.
- Do not modify the Twitter agent.
- Require nonblank text of at most 3,000 characters and 1–20 existing, readable PNG/JPEG images.
- Reject duplicate paths and files over 10 MiB; validate signatures and snapshot bytes.
- Never retry submission automatically; keep uncertain submission tabs open.
- English-language LinkedIn UI is the first supported locale.
- No real post is published as part of automated verification.
- Do not add credentials, cookies, screenshots, image bytes, or post text to repository logs.
- Preserve unrelated `tools/facebook_agent/` work.

## File Map

Paths below are relative to `projects/social-content`, unless stated otherwise.

- `tools/linkedin_agent/models.py`: `AgentError`, `ImageInput`, `validate_inputs`.
- `tools/linkedin_agent/selectors.py`: scoped DOM selectors and block detection.
- `tools/linkedin_agent/browser.py`: CDP connection, tool-tab ownership, doctor.
- `tools/linkedin_agent/posts.py`: prepare/verify/submit/confirm flow.
- `tools/linkedin_agent/cli.py`: argument parsing, JSON output, error exit codes.
- `tools/linkedin_agent/__init__.py`, `__main__.py`: package entry points.
- `tools/linkedin_agent/launch_browser.sh`: shared-profile launcher wrapper.
- `tools/linkedin_agent/tests/test_models.py`, `test_browser.py`, `test_dom.py`, `test_cli.py`: automated coverage.
- `tools/linkedin_agent/README.md`, `tests/README.md`, `tools/README.md`: documentation and frontmatter.

Run each command below from `projects/social-content`. Execute inline unless the user selects delegation. Do not automatically commit unrelated changes.

### Task 1: Immutable inputs and validation

**Interfaces:** `AgentError(code: str, message: str, human_action_required: bool = False)`; frozen `ImageInput(name: str, mime_type: str, buffer: bytes)`; `validate_inputs(text: str, paths: list[str]) -> tuple[str, tuple[ImageInput, ...]]`.

- [ ] Add failing unittest cases for blank/oversized text, zero/21 images, missing files, duplicate resolved paths, invalid signatures, oversized files, PNG/JPEG MIME classification, and preservation of text/image order. Use temporary directories, never real account data.

```python
def test_blank_text_is_rejected(self):
    with self.assertRaises(AgentError) as caught:
        validate_inputs(" \n", [])
    self.assertEqual(caught.exception.code, "invalid_text")
```

- [ ] Run `uv run python -m unittest discover -s tools/linkedin_agent/tests -p test_models.py -v`; confirm failure from missing implementation.
- [ ] Implement frozen image snapshots with bounded reads (`10 * 1024 * 1024 + 1` bytes), PNG magic `b'\x89PNG\r\n\x1a\n'`, JPEG magic `b'\xff\xd8\xff'`, and actionable errors without echoing text or bytes. Keep original text intact; normalize CRLF only for composer comparison.

```python
@dataclass(frozen=True)
class ImageInput:
    name: str
    mime_type: str
    buffer: bytes
```

- [ ] Rerun the validation tests; require all cases passing.

### Task 2: Browser lifecycle and shared-profile launcher

**Interfaces:** `Browser(endpoint: str = 'http://127.0.0.1:9222', timeout_ms: int = 15000)` supports context management, `new_page()`, `preserve_page(page)`, and `doctor() -> dict`. `detect_block(page)` raises structured authentication/checkpoint errors.

- [ ] Add mocked lifecycle tests asserting CDP arguments, context checks, only owned tabs closed, preserved tabs left open, and no call to `browser.close()`. Add synthetic DOM cases for authenticated feed, login and checkpoint pages.

```python
def test_user_browser_is_not_closed(self):
    with self.connected_browser() as connection:
        connection.new_page()
    self.chrome.close.assert_not_called()
```

The test helper patches `playwright.sync_api.sync_playwright` and supplies one mock browser context; no network connection is made.

- [ ] Run `uv run python -m unittest discover -s tools/linkedin_agent/tests -p test_browser.py -v`; confirm red.
- [ ] Implement CDP connection/disconnection following Twitter's lifecycle pattern without importing Twitter-specific models. Reject nonpositive timeouts. Navigate doctor to `https://www.linkedin.com/feed/`, wait for authenticated navigation/start-post controls, and detect login/checkpoints before reporting success. Centralize selectors and reject ambiguous matches.
- [ ] Create launcher delegating all arguments without changing Twitter's launcher:

```bash
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
exec bash "$SCRIPT_DIR/../twitter_agent/launch_browser.sh" "$@"
```

- [ ] Make wrapper executable; run `bash -n tools/linkedin_agent/launch_browser.sh` and wrapper `--help`. Verify help retains `chrome-twitter-profile` and port 9222; rerun browser tests.

### Task 3: Scoped composer, media upload, single submission

**Interfaces:** `publish_post(page, text: str, images: tuple[ImageInput, ...], timeout_ms: int = 15000) -> dict`. Returns `{'status': 'posted', 'url': str | None}` only after explicit success evidence; raises `AgentError('submission_uncertain', ..., True)` for every failure after beginning the final click.

- [ ] Add synthetic HTML fixtures inline in `test_dom.py`, using a real local Playwright Chromium page. Model start-post, personal identity, composer editor, media dialog, file input, image previews, upload progress/error and success notification. JavaScript counters distinguish media-next clicks from the final Post click.
- [ ] Add tests for single/multiple image order, exact multiline text, personal identity, company/ambiguous identity rejection, missing/duplicate editor and Post button, incomplete previews, progress, upload errors, disabled submission, stale success notification, success without URL, explicit success URL, closed composer without confirmation, and final-click exceptions.

```python
def test_uncertain_submission_is_not_retried(self):
    self.load_fixture(confirm=False)
    with self.assertRaises(AgentError) as caught:
        publish_post(self.page, "Approved text", self.images, timeout_ms=100)
    self.assertEqual(caught.exception.code, "submission_uncertain")
    self.assertEqual(self.page.evaluate("window.postClicks"), 1)
```

- [ ] Run `uv run python -m unittest discover -s tools/linkedin_agent/tests -p test_dom.py -v`; confirm red. If local Chromium is unavailable, install it with `uv run playwright install chromium` or report the blocker rather than silently skipping DOM coverage.
- [ ] Implement bounded feed navigation and visible start-post click; scope all editors and controls to the active composer/media dialog. Identify personal author through a profile link (`/in/`); reject organization (`/company/`) or missing identity evidence.
- [ ] Upload snapshots using Playwright file payloads. Follow scoped media Next/Done controls only when uniquely visible. Wait until preview count equals image count with no progress/error; verify text and enabled unique Post immediately before submission.

```python
payloads = [
    {"name": image.name, "mimeType": image.mime_type, "buffer": image.buffer}
    for image in images
]
file_input.set_input_files(payloads)
```

- [ ] Ignore pre-existing success notifications by recording their state before clicking. After invoking the final button, require a new explicit post-success notification within the configured deadline; never treat composer disappearance as sufficient. Accept only LinkedIn post links attached to that confirmation, not arbitrary page anchors.

```python
try:
    post_button.click(timeout=timeout_ms)
    return wait_for_new_confirmation(page, baseline, timeout_ms)
except Exception as exc:
    raise AgentError(
        "submission_uncertain",
        "Submission may have completed. Inspect LinkedIn manually before retrying.",
        True,
    ) from exc
```

Define `wait_for_new_confirmation(page, baseline, timeout_ms) -> dict` privately in `posts.py`; it polls new visible success evidence against the pre-click baseline using a monotonic deadline.

- [ ] Rerun DOM tests and add missing cases until all preparation failures produce zero final clicks and all attempted submissions produce exactly one invocation.

### Task 4: CLI and documentation

**Interfaces:** `main(argv: list[str] | None = None) -> int`; `build_parser() -> argparse.ArgumentParser`. Commands: `doctor`, `post --text TEXT --image PATH [--image PATH ...]`; global `--cdp`, `--timeout-ms`. No `supervise` command.

- [ ] Add subprocess cases for module help, absent image/text, invalid timeout, unreadable images, stable JSON errors and unknown commands. Add patched dispatch tests proving validation occurs before browser connection and uncertain errors call `preserve_page`.

```python
def test_invalid_input_never_connects(self):
    with patch("tools.linkedin_agent.cli.Browser") as browser:
        self.assertEqual(main(["post", "--text", " ", "--image", "missing.png"]), 2)
    browser.assert_not_called()
```

- [ ] Run `uv run python -m unittest discover -s tools/linkedin_agent/tests -p test_cli.py -v`; confirm red.
- [ ] Implement argument errors as JSON except normal help output. Map validation to 2, browser/auth/pre-submit errors to 3, uncertain to 4, unexpected to 1. On uncertainty preserve the tool tab before disconnecting; never include exception traces containing post text or image buffers in stdout.

```python
# __main__.py
from .cli import main
raise SystemExit(main())
```

- [ ] Document shared-profile manual login, direct-approved posting examples, multiple images, supported formats/limits, English locale, DOM fragility, explicit uncertainty handling, and absence of live verification. Add frontmatter to package/tests READMEs and list `linkedin_agent/` in `tools/README.md`.
- [ ] Rerun CLI tests and `uv run python -m tools.linkedin_agent --help`.

### Task 5: Final verification and review

- [ ] Run `uv run python -m unittest discover -s tools/linkedin_agent/tests -v` and inspect full results.
- [ ] Run `uv run python -m unittest discover -s tools/twitter_agent/tests -v`; distinguish existing failures from regressions without altering Twitter code.
- [ ] Run `uv run --project . .agents/hooks/file-selector/lint_readme_tree.py --changed-only`. Fix frontmatter for this agent; report unrelated Facebook-agent violations without editing that work.
- [ ] Inspect both outer and nested repository diffs, executable launcher permissions, submission exception boundaries, selector scope, tab preservation, and lack of retries. Review against every spec requirement.
- [ ] Report fixture versus live verification separately. Do not publish a test post or imply live compatibility unless explicitly tested with user-provided approved content.

## Review Checkpoints

After Tasks 1–2: validate input and shared-browser lifecycle before implementing writes.
After Tasks 3–4: verify all preparation failures have zero submission clicks and all attempted submissions have one click.
After Task 5: provide the command users can run, verification counts, and any blockers.
