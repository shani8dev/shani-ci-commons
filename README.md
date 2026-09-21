# shani-ci-commons

Shared GitHub Actions reusable workflows for the Shanios ecosystem.

These templates are referenced by 9 of the 19 shani repos via `uses:`
instead of copy-pasting workflow files. The other 10 repos roll their own
standalone workflows — see "Cross-repo impact" below for the full caller
list, including `shani-keyring`, which re-implements its own checksum-sync
check rather than calling `security.yml`.

## Available Workflows

| Workflow | Description |
|----------|-------------|
| `lint.yml` | Shellcheck (bash) / py_compile (python) — `language` input (`bash`, `python`, `all`) |
| `test.yml` | Test runner with Python setup — `language` + `test-command` inputs |
| `build.yml` | Docker image build, manifest generation (runs the repo's `manifest-script`, default `generate-manifest.js`), or container build — `build-type` input (`docker`, `manifest`, `container`) |
| `security.yml` | Keyring checksum sync, secret scanning — `scan-type` input (`checksum`, `secrets`, `all`) |
| `notify-telegram.yml` | Telegram bot notifications — needs `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` secrets |
| `validate-html.yml` | html5lib strict parse + SRI-hash verification + staleness warning for a single `index.html` — `strict-parse` / `require-sri` / `stale-days` inputs |

## Actions

| Action | Description |
|--------|-------------|
| `send-telegram` | Composite action for Telegram bot notifications — `status` (`success`/`failure`/`started`) selects the emoji; pass `bot-token: ${{ secrets.TELEGRAM_BOT_TOKEN }}` and `chat-id: ${{ secrets.TELEGRAM_CHAT_ID }}` at the call site (composite actions cannot read `secrets` themselves). No-op with a warning when either token is empty. Use this instead of pasting inline curl blocks. |

## Usage

```yaml
jobs:
  lint:
    uses: shani8dev/shani-ci-commons/.github/workflows/lint.yml@main
    with:
      language: bash

  test:
    uses: shani8dev/shani-ci-commons/.github/workflows/test.yml@main
    with:
      language: bash
      test-command: bash tests/run-all.sh

  security:
    uses: shani8dev/shani-ci-commons/.github/workflows/security.yml@main
    with:
      scan-type: checksum

  validate:
    uses: shani8dev/shani-ci-commons/.github/workflows/validate-html.yml@main
    with:
      strict-parse: true
      require-sri: false
```

## Notifications

Chat notifications go to **Telegram** (not Discord). Set these repo secrets:

- `TELEGRAM_BOT_TOKEN` — bot token from @BotFather
- `TELEGRAM_CHAT_ID` — chat ID the bot posts into

## License

MIT