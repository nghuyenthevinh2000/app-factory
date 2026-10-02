# Facebook Personal-Profile Agent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `python -m tools.facebook_agent doctor` and `post --text TEXT [--image PATH]` provide structured JSON results.

**Architecture:** Independent validation/errors, browser lifecycle, scoped DOM
posting, and CLI modules. Reuse only the existing launcher, not X-specific code.

**Tech Stack:** Python 3.12+, existing Playwright >=1.50,<2 dependency, unittest.

**Spec:** [`2026-10-02-facebook-agent-design.md`](../specs/2026-10-02-facebook-agent-design.md)

## Global Constraints

- Use `$HOME/chrome-twitter-profile` and CDP at `http://127.0.0.1:9222`.
- Publish already-approved text and an optional image directly; no supervisor or approval prompt.
- Support English-language personal profiles; reject Pages, Groups, and challenges.
- Do not change the current audience or retry a dispatched submission click.
- Verify exact content and attachments; reject restored media drafts.
- Preserve uncertain tabs for manual inspection and never publish live content in automated tests.

## Implementation checklist

Paths and commands below are relative to `projects/social-content`.
This preserves the original implementation checklist; it is not a new execution request.

### Task 1: Validation and CLI
- [ ] Write failing tests for unchanged text, invalid/empty content, file signatures,
  JSON errors, no supervisor command, and validation before browser connection.
- [ ] Run `uv run python -m unittest discover -s tools/facebook_agent/tests -v`.
- [ ] Implement `models.py`, `cli.py`, `__init__.py`, and `__main__.py` in `tools/facebook_agent/`.
- [ ] Re-run the tests until green.

### Task 2: Browser and DOM publishing
- [ ] Write local Playwright fixtures for text/image posts, own-profile checks,
  ambiguous controls, altered text, disabled submission, challenges, and
  uncertain results with exactly one click.
- [ ] Run the tests to establish the missing implementation failure.
- [ ] Implement `browser.py`, `selectors.py`, and `posting.py` in `tools/facebook_agent/`.
  Browser navigation is a separate wrapper so fixtures exercise the real posting flow locally.
- [ ] Re-run all tests and CLI help. Do not run the live post command.

### Task 3: Documentation and verification
- [ ] Document shared Chrome setup, direct publishing, inherited audience, English
  DOM support, exit codes, and manual recovery for uncertain results.
- [ ] Update tools and project README routing frontmatter.
- [ ] Run Facebook and X regression suites and the changed-only frontmatter linter.
