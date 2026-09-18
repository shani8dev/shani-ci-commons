# Agent instructions — shani-ci-commons

This file applies to any AI coding assistant working in this repository
(Claude Code, opencode, Kilo Code, Cursor, Aider, or similar). Read this
before editing, and follow the verification steps before calling any change
done.

## What this repo is

Shared GitHub Actions **reusable workflows** for the Shanios ecosystem —
`lint.yml` (shellcheck/py_compile), `test.yml` (pytest/bash test runners),
`build.yml` (docker/manifest/container builds), `security.yml` (shani-keyring
checksum sync + basic secret-pattern scanning), and `notify-telegram.yml`
(chat notifications). Other repos reference these via `uses:
shani8dev/shani-ci-commons/.github/workflows/<name>.yml@main` instead of
hand-rolling their own workflow logic. There is no `actions/` composite-action
directory — despite IMPLEMENTATION-ROADMAP.md's original proposal mentioning
one, only reusable `workflow_call` workflows were actually built.

## Empirical verification (mandatory)

**Reading code is analysis; running code is verification.** A change is not
verified by reading the YAML, running `python3 -c "import yaml; yaml.safe_load(...)"`,
or confirming it "looks correct." It is verified by observing the actual
behavior of the real thing in the real environment. If you haven't seen it
work (or fail) for real, it isn't verified.

## Verification for any change

There is no `act`/`actionlint` installed in the default environment (check
before assuming otherwise) — verification here means extracting the embedded
shell logic from a workflow step's `run:` block and executing it for real
against a constructed test case, not just reading it:

```bash
# 1. Every workflow file must at least parse as valid YAML
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/<file>.yml'))"

# 2. For embedded shell logic (the real risk surface — this class of bug
#    has already bitten this file once, see below): copy the exact shell
#    out of the `run:` block into a throwaway directory with both a
#    "should pass" and a "should fail" test case, and confirm the exit
#    code is actually right for both — not just that it runs without a
#    syntax error.
```

If you change what a workflow's `run:` step does, you MUST construct both a
positive case (nothing wrong — step should exit 0) and a negative case
(something wrong — step should exit non-zero) and run both for real. A step
that "looks like" a gate but always exits 0 is worse than no gate at all,
because it reports green.

## Audit-verified known issues

- **`security.yml`'s secret-scan step never failed the job, even on a real
  match — FIXED (2026-09-18).** `grep ... | head -20 || echo "..."` never
  fails without `pipefail`: the pipeline's exit status is `head`'s (always
  0 unless `head` itself errors), not `grep`'s, so a real secret match got
  printed but the step still reported success. Confirmed live with a
  planted, correctly-shaped fake token (`ghp_` + 36 chars): unpatched code
  printed the match and exited 0; patched code (`matches=$(grep ... ||
  true)`; explicit `exit 1` when non-empty) correctly exits 1, and a
  clean-tree negative control still exits 0. `shani-pkgbuilds` is currently
  the only real caller of `security.yml` and only passes `scan-type:
  checksum` (not `secrets`/`all`), so this specific bug was never live in
  production — but any future caller passing `secrets`/`all` would have
  gotten a no-op scan with a green checkmark.
- **`lint.yml`/`test.yml`/`build.yml`/`notify-telegram.yml`** — reviewed,
  all parse as valid YAML with correct `workflow_call` input/secret wiring;
  no equivalent always-succeeds bug found in their embedded shell logic
  (spot-checked this pass, not exhaustively execution-tested each branch).
- **`build.yml`'s `build-command` and `test.yml`'s `test-command` inputs are
  interpolated directly into a `run:`/`bash -lc "..."` step** — this is a
  script-injection surface if either input is ever sourced from something a
  non-trusted actor (e.g. a fork PR) can influence. Every current caller
  hardcodes these as literal strings in its own `.github/workflows/*.yml`
  (repo-owner-controlled), so this is safe in practice today — but never
  wire either input to PR title/body/label content or similar
  attacker-influenced text without re-reviewing this.
- **`notify-telegram.yml` puts the bot token in the request URL**
  (`https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/...`), not just a
  header — inherent to Telegram's Bot API design, not fixable here. GitHub
  Actions masks the literal secret value in job logs since it's sourced
  from `secrets.TELEGRAM_BOT_TOKEN`, but the token is still visible in that
  runner process's argv for the duration of the `curl` call. Accepted
  tradeoff, not a bug to "fix" — just don't assume this pattern is safe to
  copy for a service with a header-based auth option instead.

## Commit discipline

Before composing a commit message, run `git log --oneline -20` (and `git
log -5 -- <touched paths>` for the files you changed) and match the
existing style — subject shape, scope prefixes, body detail level —
rather than writing in a generic format.

## Boundaries

- ✅ **Always**: construct a real positive AND negative test case for any
  changed `run:` shell logic (see "Verification for any change" above) — a
  gate that always exits 0 is worse than no gate.
- ⚠️ **Ask first**: changing a workflow's `inputs:`/`secrets:` contract —
  every caller listed below needs its call site checked or updated in the
  same change, not after.
- 🚫 **Never**: wire `build.yml`'s `build-command` or `test.yml`'s
  `test-command` inputs to PR title/body/label content or other
  attacker-influenced text (script-injection surface, currently safe only
  because every caller hardcodes these as literal strings).

## Cross-repo impact — check before calling a fix complete

Real current callers (verified via `grep -rln "shani-ci-commons"
*/.github/workflows/*.yml` across the whole `/home/shrinivaskumbhar/Documents/shani`
workspace, 2026-09-18):

- `shani-pkgbuilds` — `security.yml` (`scan-type: checksum`), `lint.yml`
- `shani-blog` — `build-manifest.yml`, `notify-telegram.yml`
- `shani-builder` — `build.yml`, `notify-telegram.yml`
- `shani-docs` — `build-manifest.yml`
- `shani-install-media` — `build.yml`, `build-image.yml`, `notify-telegram.yml`
- `shani-platform` — `ci.yml`
- `shani-insights` — `ci.yml`
- `shani-fleet` — `ci.yml`

**Not** consumers (each rolled its own standalone workflow instead):
`shani-repo`, `shani-chronoa`, `shani-backup`, `shani-keyring`,
`shani-deploy`. `shani-keyring` in particular re-implements its own
checksum-sync CI check directly rather than calling this repo's
`security.yml` — if you fix a bug in the checksum-sync logic in one place,
check whether the other needs the same fix (they are not shared code, just
the same underlying idea implemented twice, same pattern as
`shani-fleet`/`shani-insights`'s independent agent scripts).

If you change a workflow's `inputs:`/`secrets:` contract (name, type,
required-ness, default), grep every caller listed above for that workflow
file and update the call site — a caller with a stale `with:`/`secrets:`
block will fail at call time, not at edit time here.

## Where things are documented

`README.md` has the usage table and example `uses:` blocks — keep it in
sync with the actual `inputs:` schema in each workflow file when either
changes.
