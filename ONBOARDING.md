# Onboard a repository to pending review-first CI

Use this procedure when a human requests review-first onboarding or migration of
an existing repository. The supported default is the current v2 pending-status
contract, not the historical v1 intentional-failure gate. Requesting one repository
does not authorize migrating other repositories, changing rulesets, merging,
deploying, granting App permissions or synchronizing runtimes.

Start with [the workflow template](examples/review-first-ci.yml), which pins both
shared components to the same reviewed commit retained on `main`. Merge its job
pattern into the repository's existing workflow; do not overwrite working triggers,
validation commands, deployment prerequisites or security restrictions. The
[shared contract](README.md#no-build-gate-v2) owns lifecycle semantics and permissions.

## 1. Inventory the repository before editing

Resolve the current source revision, branch-protection/ruleset required context
names and expected GitHub integration. Use read-only inspection. Enumerate every
independent validation, preview and admission workflow, including classifiers,
mobile/worker checks and secondary drift checks. For each, record in the PR:

| Workflow/job | Required context (or nonrequired) | Existing events and candidate SHA | Gate, validation and result owner |
| --- | --- | --- | --- |
| Existing validation | Exact required name, unchanged | PR merge SHA; preserve other events | Existing commands plus matched gate/reporter |

Follow downstream `workflow_run` and release triggers. Identify provider review
setup, trusted-base opt-in and deployed review/autofix compatibility; a source
change alone does not establish those prerequisites. Broccoli's current
[provider contract](https://github.com/besimple-graveyard/broccoli/blob/main/docs/review-first-validation.md)
owns strict-review configuration. Do not recreate the retired approval-eligibility
check or make it an onboarding dependency.

Search local workflow guards, lint configuration and maintained docs for the old
failure marker, pinned gate, required names and concurrency restrictions. Update
those existing contracts with the workflow. A YAML-valid change alone will not
catch a repository guard that still rejects the new status queue.

Do not add `#no-build` blindly to an unonboarded repository: ordinary CI remains
available until compatible wiring and review prerequisites are established.

## 2. Wire every required context

The template has one required context, `CI`. Replace it consistently in the gate
input, aggregate name, concurrency key and reporter input with the exact existing
required name. Replace its sample validation command with the original commands.
Preserve immutable `github.sha` checkout and existing required result assertions.
Keep candidate-code execution separate from short status-writing jobs.

The gate accepts one required context per call. For multiple required contexts in
one workflow, use one admission call and one matching final reporter per context;
connect the appropriate validation jobs. Do not protect only the first context,
combine distinct required names, or replace a required context with an admission
job name. A shared validation job may depend on multiple admissions only when all
must explicitly admit it; preserve existing matrix/classifier behavior.

Nonrequired validation roots also require explicit `run_validation == 'true'`, but
their gate calls omit `required-context` and keep read-only permissions. Review and
PR-close cleanup stay outside deferral. Preserve fork/Dependabot validation and
existing credential restrictions; never switch to `pull_request_target` or add
privileged checkout to enable deferral.

For classified suites, preserve the exact allowed success/skipped result tuple.
Set the reporter's `validation-required: 'false'` only for the classifier's valid
no-work case. Review deferral is not that case. Missing output, admission failure,
validation failure and unexpected skipping must not become green.

Use the template's base-retarget handling if the repository supports PR edits:
base changes validate; title/body-only edits neither replace required results nor
cancel real validation. Preserve merge-group, main-push, manual and scheduled
behavior. The optional manual-label input is separate from standard onboarding;
do not enable it or change label permissions without task-specific scope.

## 3. Verify the scoped change and publish

Run existing source/workflow guards covering the changed wiring and validate YAML,
expressions, shell syntax, pins and permissions. Compare the validation commands
and required names before/after. Check that every required gate and reporter share
one candidate/context concurrency group with `queue: max` and cancellation disabled.
Use the repository's existing verification and test policy; onboarding does not
require new tests, application builds or a new permanent CI gate.

Some actionlint versions do not recognize GitHub's supported `concurrency.queue`.
If necessary, exclude only that known syntax diagnostic for the affected files;
validate the value independently against the
[GitHub concurrency contract](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency).
Do not suppress unrelated diagnostics or bypass repository guards.

Publish a ready PR with the inventory, source revision, focused evidence, shared
pin, provider/runtime prerequisites and recovery path. Unavailable live validation
is an explicit remaining step, not a reason to withhold reviewable repository work.
Keep each consumer's completion separate: a shared PR or another consumer's success
does not establish that this repository is onboarded.

## 4. Distinguish source delivery from activation

When live observation and rerunning are authorized, retain the candidate SHA,
original run/attempt and required-context evidence for this sequence:

| Stage | Expected evidence |
| --- | --- |
| Trusted marked PR before review | Admission succeeds; validation skips; every required status is Pending; GitHub blocks merging |
| Accepted review and full native rerun | Same immutable candidate; queued/running native attempt; required status links that attempt |
| Real validation completes | Required statuses reflect actual success or failure; no skipped attempt is treated as passing validation |

Do not manufacture application failures or production deployments for this proof.
Review release uses Colony's existing [candidate-bound rerun guidance](https://github.com/besimple-graveyard/rsi-broccoli/blob/main/src/colony/runtime_profile/guidance/repository.md#review-first-validation),
not an arbitrary dispatch, marker removal or `--failed` rerun. Without monitoring
authority, take one snapshot and report the exact remaining observation.

Historical runs retain old pins and display names even after migration. Roll back
the gate pin, reporter pin and caller routing/queues together. Existing statuses
survive source rollback; finish compatible real validation or publish a reviewed
ordinary-validation change. Never mark unvalidated statuses successful manually.
