# Shared CI workflows

Reusable GitHub Actions workflows for initial validation deferral.

## No-build gate

Call `.github/workflows/no-build-gate.yml` using a full commit SHA from protected `main`. The gate has no inputs, workflow outputs or custom secrets. It uses the caller's GitHub context and token with only `contents: read`.

On the first `pull_request` workflow attempt, the gate reads the captured PR head commit message. Case-insensitive `#no-build` fails the step named exactly `Broccoli validation deferred v1`. Unmarked initial attempts, other events and every native rerun pass the gate without consulting review eligibility. API errors or malformed commit data fail the lookup step instead. The original candidate SHA is logged on every attempt.

The gate does not check out or execute repository code. Its internal step output is not exposed to callers. Callers use `needs.validation-gate.result`.

## Consumer wiring

Replace `<GATE_SHA>` with a full commit SHA from this repository's `main` history:

```yaml
jobs:
  validation-gate:
    uses: besimple-interns/ci-workflows/.github/workflows/no-build-gate.yml@<GATE_SHA>
    permissions:
      contents: read

  build:
    needs: validation-gate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.sha }}
      - run: ./scripts/validate.sh

  required-aggregate:
    if: always()
    needs: [validation-gate, build]
    runs-on: ubuntu-latest
    steps:
      - name: Broccoli validation deferred v1
        if: needs.validation-gate.result != 'success'
        run: exit 1
      - name: Require build success
        env:
          BUILD_RESULT: ${{ needs.build.result }}
        run: test "$BUILD_RESULT" = success
```

The validation command above is illustrative. Keep your existing checks, required-check names, event guards and change classifiers. Do not pass `secrets: inherit`. Apply restricted permissions to the gate job without reducing permissions required by sibling jobs.

Ordinary dependent jobs rely on GitHub's implicit success condition. Always-running required jobs need the explicit failure step before their existing result checks; listing a dependency alone does not validate its result. Keep classification-permitted skips in those existing checks, and keep independent review and close-cleanup jobs outside the gate.

## Deferral and native reruns

Review eligibility and scheduling belong to the caller's automation. To release deferred validation, rerun every job of the original workflow. Do not use failed-jobs-only reruns, substitute dispatch runs, or a moving checkout ref. A full rerun executes ordinary validation against the original immutable candidate; it does not grant passing checks or merge authority.

Classify a run as deliberate deferral only when at least one job failed, every failed job failed solely at the exact deferral step, and all other jobs completed successfully, skipped or neutral. Inspect the entire run attempt. A lookup error remains a real failure because the gate fails at a different step, even though downstream aggregates fail at the deferral step. Missing evidence is not proof of deliberate deferral.

## Updates and recovery

Pin callers to a reviewed commit retained on protected `main`. Keep force pushes and branch deletion disabled. Update pins through normal repository review; no version tags or copied gate scripts are required.

Stop creating marked iterations before removing gate support. Recover through a reviewed caller update or a prior compatible retained gate commit. Already deferred runs remain nonpassing until a compatible full native rerun or an ordinary-validation source update. Preserve the caller's merge protection and existing deployment and cleanup procedures.
