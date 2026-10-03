# LinkedIn image-post agent design

## Approved scope

Create `tools/linkedin_agent` as an independent Python/Playwright CLI for publishing
provided text and local images to the user's personal LinkedIn profile through
visible DOM interactions. The input is already approved: no supervisor, queue,
approval prompt, or draft-generation feature. Do not modify the Twitter agent.

## Browser and authentication

Reuse `$HOME/chrome-twitter-profile` and CDP at `http://127.0.0.1:9222` by default.
The launcher delegates to the existing Twitter launcher, preserving its profile
and port options. The user logs into LinkedIn manually in that Chrome profile.
Connect to the existing browser, create only a tool-owned tab, and never close
the user's browser or existing tabs. Keep an uncertain submission tab open for
manual inspection. Detect login screens and security checkpoints and stop rather
than trying to solve or bypass them.

## Interface

Run from `projects/social-content`:

```bash
./tools/linkedin_agent/launch_browser.sh
uv run python -m tools.linkedin_agent doctor
uv run python -m tools.linkedin_agent post --text "Post text" --image /absolute/image.png
```

`--image` is repeatable for ordered multiple-image posts. Require nonblank text
of at most 3,000 characters and 1–20 existing, readable PNG/JPEG images, checking
file signatures rather than trusting extensions. Reject duplicate paths and
files over 10 MiB. Read validated image bytes before browser navigation so later
file changes cannot change the submitted content. Global `--cdp` and
`--timeout-ms` options allow configuration; operations have bounded waits.

JSON stdout contains `ok` and `data` or a stable `error` object. Progress goes to
stderr. Exit codes: 0 confirmed success, 2 invalid input, 3 browser/authentication
or pre-submit DOM failure, 4 uncertain submission, 1 unexpected failure.

## Posting flow

1. Validate text and images locally before opening a tab.
2. Navigate to LinkedIn's feed and verify authentication.
3. Open the personal-profile composer through scoped, centralized selectors.
4. Verify the publishing identity is a personal profile; reject company-page
   composers or ambiguous identity instead of silently posting elsewhere.
5. Fill text and upload validated bytes in their supplied order, completing any
   media dialog using its own scoped controls.
6. Wait for all expected image previews and no visible upload progress/errors.
   Re-read text, verify exact normalized line breaks, and ensure the final Post
   button is uniquely identified and enabled.
7. Invoke the final Post button exactly once. Revalidate draft invariants at
   actual trusted click dispatch and cancel changed drafts before click-driven
   application handlers run. A positively acknowledged cancellation with clean
   completion is a pre-submit failure; otherwise, once invocation starts, any
   exception or lack of explicit LinkedIn success evidence becomes `uncertain`.
   A closed composer alone is not proof of success. Return a post URL only if
   LinkedIn provides one with the confirmation.
8. Never retry submission automatically. Preserve the tab and tell the user to
   inspect LinkedIn manually for uncertain outcomes.

English-language LinkedIn UI is the first supported locale. Unknown markup,
ambiguous selectors, authentication failures, upload errors, or unsupported
composers fail closed before submission.

## Files and responsibilities

- `__init__.py`, `__main__.py`: package and module entry point.
- `models.py`: structured errors and immutable validated image inputs.
- `browser.py`: CDP lifecycle and authentication doctor.
- `selectors.py`: LinkedIn composer/media selectors and checkpoint detection.
- `posts.py`: input validation, composer preparation, one-click submission,
  confirmation classification.
- `cli.py`: arguments, JSON output, and exit-code mapping.
- `launch_browser.sh`: thin wrapper around the existing shared-profile launcher.
- `tests/`: standard-library unit tests, mocked browser tests, and Playwright DOM
  fixtures using synthetic LinkedIn-like HTML.
- `README.md`: setup, shared-profile usage, posting examples, uncertainty warning,
  limitations, and standardized frontmatter.

Use existing Python >=3.12 and Playwright >=1.50,<2 dependencies. Do not add
credentials, cookies, screenshots, image bytes, or post text to repository logs.
No real post is published as part of automated verification.

## Verification

Test validation before browser connection, repeatable images, shared-profile
launcher defaults, exact composer text, completed image previews, ambiguous DOM,
upload failures, company identity rejection, login/checkpoints, disabled Post,
single-click success, and uncertainty after a click or click exception. Test
CLI JSON and exit codes through subprocesses. Run the existing Twitter suite
for regressions and the README frontmatter linter before completion.

## Exclusions

Comments, reactions, company pages, documents, scheduling, discovery, stealth,
challenge bypass, and automatic retries are outside this version. Synthetic DOM
tests do not guarantee compatibility with every live LinkedIn UI variant; report
live verification separately from fixture verification.

The dispatch guard supports click-driven submission, not an application that
submits earlier on pointerdown or an earlier window capture handler. Upload
association is checked at the browser's selected-file boundary with an empty
media baseline and fresh previews, not through independent server verification.
