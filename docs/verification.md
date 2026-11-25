# Verification

Project-specific evidence only. The repository began from the public
`mstrhakr/go-template` scaffold; its prior releases, GitHub settings, and test
runs are not verification for Homestead-NG.

## Executed Locally

- 2026-11-23: `make check-contracts` passed structure and stable-ID checks for
  the three contract documents after the latest scope decisions. These checks
  do not approve behavior.
- 2026-11-20: `make check` passed: Go tests/build, 14 unit tests, 6 JS tests,
  24 browser tests, asset verification, and contract checks.
- 2026-11-20: `make check-workflows` passed workflow policy and actionlint checks.
- 2026-11-23: `make docs` built seven pages without warnings.

## GitHub Readback

Verified 2026-11-23 for `mstrhakr/go-household-erp`: public repository, `main`
default branch, Issues enabled, Wiki/Discussions disabled, squash-only merges,
secret scanning/push protection/private vulnerability reporting/Dependabot
security features enabled, read-only allowlisted Actions, active protected-main
and immutable-tag rulesets. Protected `main` requires PRs and the `CI gate`; the
latest `main` CI run succeeded. Release/container/docs publishing remains off.

The repository was transferred to `homestead-ng/core` on 2026-11-25. Readback
confirmed it remains public, with the protected-main CI gate, immutable version
tags, release approval, and selected pinned Actions. Secret scanning and push
protection were re-enabled and verified after transfer. Release and container
publication remain disabled. Organization-level Actions policy could not be
read with the current token.

No Homestead-NG application behavior has been implemented or verified. Native
platform CI, releases, images, and documentation publication have not been
executed for this repository.

Update this record only with commands actually run, environment details, and
remaining verification gaps.
