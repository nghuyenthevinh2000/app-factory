# Facebook personal-profile agent design

## Approved design

Publish already-approved text and an optional local image directly through the
visible Facebook DOM. No supervisor, approval prompt, draft queue, or API tokens.
Use the existing Chrome launcher and `$HOME/chrome-twitter-profile`, connecting
over CDP at `http://127.0.0.1:9222`. Login remains manual.

Limit the first implementation to English-language personal profiles. Verify an
own-profile edit control, profile name, composer, audience, exact text, and image
upload before dispatching one Post click. Do not change the current audience.
Stop on challenges, ambiguous controls, Pages, or Groups. A dispatched click
without positive confirmation is uncertain, never retried. Preserve the tab on
uncertainty for manual inspection. Never publish live content in automated tests.
Reject restored media drafts and recheck final attachments against the request.
Keep visibility selection compatible with the declared Playwright 1.50 minimum.
Exclude contenteditable post text from the posting-identity lookup.

## Implementation plan

See [`2026-10-02-facebook-agent.md`](../plans/2026-10-02-facebook-agent.md).
