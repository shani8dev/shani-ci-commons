# shani-ci-commons

Shared GitHub Actions reusable workflows for the Shanios ecosystem.

All 15 shani repos reference these templates via `uses:` instead of copy-pasting workflow files.

## Available Workflows

| Workflow | Description |
|----------|-------------|
| `lint.yml` | Shellcheck, py_compile, pre-commit hooks |
| `test.yml` | Test runner with Python setup |
| `build.yml` | Docker/Podman container build |
| `security.yml` | Secret scanning, Trivy, dependency audit |
| `notify-discord.yml` | Discord webhook notifications |

## Usage

```yaml
jobs:
  lint:
    uses: shani8dev/shani-ci-commons/.github/workflows/lint.yml@main
    with:
      shellcheck-version: v0.10.0.1

  test:
    uses: shani8dev/shani-ci-commons/.github/workflows/test.yml@main
    with:
      test-command: bash tests/run-all.sh

  security:
    uses: shani8dev/shani-ci-commons/.github/workflows/security.yml@main
```

## License

MIT
