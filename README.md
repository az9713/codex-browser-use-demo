# Codex Browser Use Demo

This repository documents a small Codex browser automation workflow: identify a product from a local photo, find the matching Amazon item, add quantity 1 to the shopping cart, and stop before checkout.

The demo is intentionally written as an audit trail rather than an app. The main artifact is the process report:

- [Amazon cart automation report](./amazon_cart_process_report_2026-05-02.md)

## What This Demonstrates

- Using local image tooling to inspect an uploaded product photo.
- Recovering when a `.JPG` file is actually HEIC data.
- Running `browser-use[cli]` through `uvx` without a permanent install.
- Driving Amazon through visible browser automation with `open`, `state`, and `click`.
- Verifying that the cart contains exactly one item.
- Avoiding checkout and payment actions.

## Tech Stack

- PowerShell for command orchestration.
- ImageMagick for image identification, conversion, and cropping.
- `uvx` with Python 3.13 for temporary CLI execution.
- Browser Use CLI (`browser-use`) for browser automation.
- Amazon.com web UI as the target browser surface.
- GitHub CLI (`gh`) for repository creation and push.

## Browser Use CLI

`browser-use[cli]` exposes the `browser-use` command. Its core workflow is:

```powershell
browser-use open https://example.com
browser-use state
browser-use click 5
```

The useful part for agent workflows is `state`: it returns visible/clickable page elements with numeric indices. The agent can inspect the page, choose the right index, click it, and re-run `state` after the page changes.

Browser Use is open source and licensed under MIT:

- <https://github.com/browser-use/browser-use>
- <https://docs.browser-use.com/open-source/browser-use-cli>

## Safety Notes

The local source photo and generated image crops are not committed. The original photo contained camera metadata, so the repo keeps only the Markdown documentation.

The automation stopped at the shopping cart. It did not open or submit checkout.
