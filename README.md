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
| `build.yml` | Docker image build, manifest generation, or container build — `build-type` input (`docker`, `manifest`, `container`) |
| `security.yml` | Keyring checksum sync, secret scanning — `scan-type` input (`checksum`, `secrets`, `all`) |
| `notify-telegram.yml` | Telegram bot notifications — needs `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` secrets |

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
```

## Notifications

Chat notifications go to **Telegram** (not Discord). Set these repo secrets:

- `TELEGRAM_BOT_TOKEN` — bot token from @BotFather
- `TELEGRAM_CHAT_ID` — chat ID the bot posts into

## License

MIT