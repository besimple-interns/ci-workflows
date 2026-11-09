# Shared CI workflows

Reusable GitHub Actions workflows for review-first validation.

## No-build gate (v2)

Pin `.github/workflows/no-build-gate.yml` and `actions/validation-status` to the same
reviewed full commit SHA retained on protected `main`. There are no custom secrets
or inherited secrets. The gate never checks out or executes caller repository code.

The gate exposes `run_validation`, exactly `'true'` or `'false'`. On the initial
trusted `pull_request` attempt it reads the captured source head commit message.
Case-insensitive `#no-build` produces `'false'` and logs `Broccoli validation deferred v2`;
this is a successful admission decision, not a failed validation. Jobs that depend
on the gate must explicitly require `'true'`, so deferred validation shows **Skipped**.
Unmarked attempts, full native reruns and other events admit ordinary validation.
The original immutable candidate SHA is logged on every attempt.

Fork PRs and Dependabot PRs/actors run ordinary validation regardless of the marker:
their read-only tokens cannot establish the required pending statuses. Do not use
`pull_request_target`, additional credentials or privileged checkout to defer them.
API failures and malformed commit data are real failures; never treat a missing
output as deferral or admission.

### Optional manual label (Pila8 pilot only)

`manual-label` is an optional string, disabled by default (`''`). Pila8 alone opts
in with `manual-label: run-ci`. Apply that label to request real selected validation
on the new `pull_request` `labeled` event's immutable `github.sha`, even when the
commit has `#no-build` or an automation push produced no runnable attempt. Remove
and reapply an existing label to make a new request; persistent label membership
does not admit later pushes. Callers must subscribe to `labeled` and isolate
unrelated label events from validation and required-status publication.

The event label name must equal the configured nonempty name, ignoring case to
match GitHub expressions. Only trusted same-repository, non-Dependabot PRs qualify.
The original event sender must be a human with live write, maintain or admin
repository permission. Lookup errors or denied permission fail the gate, including
on reruns; the rerun triggering actor cannot replace that sender. Fork/Dependabot
events retain ordinary validation with their existing permissions. This input adds
no elevated checkout, generic bypass or successful validation result: required
contexts still become Pending and the reporter publishes actual success/failure.

## Required checks must remain outstanding

GitHub accepts skipped required jobs as successful. For each required validation
workflow, pass its exact existing required context as `required-context`. Before
emitting its output, the gate posts a **Pending** commit status on immutable
`github.sha`, the original test-merge candidate. A required context with both a
native check and a commit status needs both to pass. The pending status therefore
blocks merging even when native validation jobs show Skipped, or a previous run
on the same candidate passed. The test-merge status prevents fallback to an older
source-head result. See [GitHub's required-check rules](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks).

The gate also establishes Pending before admitted PR validation. After the existing
aggregate validates its exact result tuple, `actions/validation-status` publishes
the actual success/failure using the same context and original SHA. Cancellation
leaves validation outstanding. Failed status publication is a real reporting failure.
Metadata-only events must not create or complete these statuses.

Only the required gate calls and their aggregate jobs need `statuses: write` in
addition to `contents: read`. Other gate calls retain only `contents: read` and
omit `required-context`. Permissions are inherited from the explicit caller job;
no App grants, secrets, ruleset changes or runtime service are introduced. The
status writer uses the existing GitHub Actions identity. Preserve any ruleset's
expected integration and verify hosted enforcement before declaring activation.

## Consumer wiring

Replace `<GATE_SHA>` with the same reviewed full SHA in both references:

```yaml
jobs:
  validation-gate:
    uses: besimple-interns/ci-workflows/.github/workflows/no-build-gate.yml@<GATE_SHA>
    permissions:
      contents: read
      statuses: write
    with:
      required-context: Required validation

  build:
    needs: validation-gate
    if: needs.validation-gate.outputs.run_validation == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.sha }}
      - run: ./scripts/validate.sh

  required-aggregate:
    name: Required validation
    if: always() && !(needs.validation-gate.result == 'success' && needs.validation-gate.outputs.run_validation == 'false')
    needs: [validation-gate, build]
    permissions:
      contents: read
      statuses: write
    runs-on: ubuntu-latest
    steps:
      - name: Require a valid admission decision
        if: needs.validation-gate.result != 'success' || needs.validation-gate.outputs.run_validation != 'true'
        run: exit 1
      - name: Require build success
        env:
          BUILD_RESULT: ${{ needs.build.result }}
        run: test "$BUILD_RESULT" = success
      - name: Report actual validation result
        if: always() && !cancelled()
        uses: besimple-interns/ci-workflows/actions/validation-status@<GATE_SHA>
        with:
          context: Required validation
          result: ${{ job.status }}
```

The command and name above are illustrative. Preserve existing required contexts,
classification-permitted skips, result assertions, fork/event guards and immutable
checkouts. Guard **every** independent validation root, including otherwise
independent admission checks; keep review and PR-close cleanup outside this gate.
Metadata reporting can keep its distinct nonrequired name and execute without
validation or status publication. Merge-group, main, manual and scheduled validation
retain their normal native checks; pending commit statuses are PR-only.

## Review and native reruns

Broccoli's existing structured eligibility v1 is unchanged. Colony binds the newest
eligible review to the original run's immutable source/base/merge candidate before
rerunning **all jobs** of that run. Do not use `--failed`, substitute dispatches,
empty commits or moving refs. A successful workflow with the v2 deferral marker,
skipped validation and pending required statuses is unfinished work, not passing
validation. Read the pinned contract and the whole original attempt; do not reuse
its workflow success as proof of validation. Actual failures remain failures.

Consumers pinned to `fd4e80459a757affc4ec13eece9cf310b5262647` still use the historical
v1 intentional-failure protocol. That retained revision and Broccoli's existing v1
classifier remain available. Do not change other consumers merely to adopt v2;
rollout beyond the Pila8 pilot requires explicit human instructions.

## Release and recovery

Publish/review the shared change, land it on protected `main`, then pin the Pila8
consumer to the retained commit containing both gate and reporter. Release Colony's
matching scheduling guidance before using v2 marked iterations. Existing runtimes
need normal adoption or separately authorized steering. Merges, releases and runtime
synchronization remain operator-owned; source publication does not prove activation.

For the manual-label pilot, publish both PRs for review, then land this shared
prerequisite first and retain its reviewed SHA before activating Pila8's pinned
gate/reporter references and label-event wiring. No other consumer opts in or
changes pins. Roll back Pila8's manual input/event wiring and both pins together
to the prior v2 revision; retain the shared commit. Pending statuses still require
real validation as described below. No service or runtime release is needed for
this optional shared input.

After activation, verify one marked pilot: native validation Skipped, exact required
statuses Pending and merge blocked; eligible review then full rerun of the same
candidate; actual successful validation resolves those statuses. Local checks cannot
prove GitHub's live ruleset association or Colony's deployed behavior.

Recover by stopping marked iterations and reverting consumer wiring and both pins
together. Existing Pending statuses survive source rollback: finish a compatible
full rerun against that candidate, or publish a reviewed ordinary-validation source
update. Never manually mark unvalidated statuses successful. Preserve retained
shared history, native merge protection and existing preview/staging recovery.
