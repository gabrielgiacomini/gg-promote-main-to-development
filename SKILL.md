---
name: promote-main-to-development
description: Use when promoting a mainline branch into a development integration branch across one or more repositories, including inferred PR titles/descriptions, PR creation or reuse, clean merges, conflict-safe resolution, CI/CD verification, and runtime health reporting.
---

# GG → Promote Main To Development → Full Promotion

> **Snapshot age:** reusable operational guidance. Verify live `gh`, `git`, deployment-provider, and CI/CD CLI flags before running commands in a target repository.

## Overview

Use this skill for a full `main` → `development` promotion in any multi-repository or single-repository software project. The workflow is intentionally project-agnostic: the operator supplies the repository topology, repository list, release surfaces, merge policy, CI/CD providers, deployment endpoints, and runtime health checks for the target project. Branch names default to `main` → `development` and must be verified remotely before mutation. Monorepos are the default topology unless project docs say otherwise.

This skill covers branch discovery, PR creation or reuse, safe merge execution, isolated conflict resolution, deployment evidence, deep-lie verification, and final closeout reporting. When it creates PRs, it automatically infers the PR title and description from the promotion lane, repository topology, source/target evidence, changed files, impacted release surfaces, and verification plan; project docs may override the metadata pattern, but the operator does not need to supply routine PR text. It treats the source and target release branches as permanent branches and must not delete them unless a project policy explicitly requires it and the user authorizes the exact repository and branch. It does **not** define project-specific repositories, domains, services, credentials, or infrastructure commands.

## When to Use This Skill

**TRIGGER when:**
- The user asks to promote `main` or a mainline branch into a development/integration branch.
- The promotion includes merge execution, conflict handling, or post-merge verification.
- A repository set must be promoted together and verified as a release surface, not as isolated code pushes.

**SKIP when:**
- The user only wants PR creation and handoff without merging.
- The source or target branch is not a mainline-to-development promotion.
- The task requires project-specific deployment knowledge that has not been supplied or cannot be discovered from local docs.

## Required Project Inputs

Before mutating anything, identify or ask for these project-specific inputs:

| Input | Purpose | Example placeholder |
|-------|---------|---------------------|
| Repository topology | Monorepo by default, multi-repo, or submodule-aware | monorepo |
| Repository set | Repositories promoted together | `<owner>/<repo>` list; one repo for monorepos |
| Release surfaces | Apps/packages/workspaces/deployments affected inside each repo | `<apps/web>`, `<packages/api>` |
| Source branch default | Use `main` unless project docs explicitly override it; verify remotely | `main` |
| Target branch default | Use `development` unless project docs explicitly override it; verify remotely | `development` |
| Branch preservation policy | Source and target release branches are permanent; never delete them unless explicitly required and authorized | preserve source and target |
| PR metadata policy | Optional project-provided title/body convention; otherwise infer from lane and evidence | inferred title and description |
| Merge method | Branch-protection-compatible merge mode | merge, squash, rebase, admin merge |
| Root tracking policy | Whether a meta/root repo branch tracks the promotion | optional |
| Deployment providers | CI/CD surfaces to inspect | project-supplied deployment providers |
| Runtime checks | Health URLs, smoke tests, alias checks | project-supplied endpoints |
| Conflict policy | Whether agent may resolve conflicts | isolated clone only |

If these inputs are absent, perform read-only discovery first and report unknowns instead of guessing.

## Branch Defaults and Verification

Default branch names for this skill are:

| Role | Default branch |
|------|----------------|
| Source | `main` |
| Target | `development` |

Do not ask the user to supply these branch names for standard workflows. Instead, verify that both branches exist in every in-scope repository before creating PRs, merging, or inspecting deployments. If either branch is missing, ambiguous, protected in an unexpected way, or project docs explicitly use a different branch ladder, stop and report the mismatch before mutating anything.

Override `SOURCE_BRANCH` or `TARGET_BRANCH` only when project docs or the operator explicitly identify non-standard branch names.

### Branch Preservation Requirement

The source and target branches for this workflow are long-lived release branches, not disposable feature branches. Never delete either branch, pass merge flags that delete either branch, close a PR with branch cleanup, or tell the user to click GitHub's **Delete branch** button for `SOURCE_BRANCH` or `TARGET_BRANCH` unless a documented project policy explicitly requires that cleanup and the user authorizes the exact repository and branch.

Whenever reporting a PR or merge result, remind the user that the source and target branches should remain in place after the promotion. If GitHub or another provider offers branch deletion after merge, tell the user not to delete these branches because they are supposed to be permanent promotion lanes.

## PR Metadata Inference

When this skill creates a PR, infer the title and description automatically. Do not ask the user to write routine PR text.

Default inferred title pattern:

```text
chore(release): sync <source-branch> into <target-branch>
```

Use a project-provided exact title only when project docs, branch protection, or release automation explicitly require one. Otherwise derive the title from the active source and target branch names.

Infer the PR description from verified evidence before calling `gh pr create`:

1. **Promotion lane:** source branch, target branch, repository, topology mode, and whether this is monorepo, multi-repo, or submodule-aware.
2. **Diff evidence:** source SHA, target SHA, commit count or compare URL, changed-file summary, and risky paths such as migrations, lockfiles, generated files, infrastructure, CI/CD, secrets, or environment files.
3. **Release surfaces:** impacted apps, packages, workspaces, deployments, or submodules discovered from changed paths and project docs.
4. **Verification plan:** PR review/check gates before merge plus deployment, runtime, routing, and provider checks after merge when relevant.
5. **Known gaps:** missing project inputs, blocked checks, unresolved reviewers, unknown release surfaces, or provider details still needed.

Never create promotion PRs with an empty body unless the project explicitly requires blank descriptions. If any metadata input is unknown, include an `Unknowns / follow-up` section rather than inventing project facts.

## Plain-Language Progress Updates and Handoffs

This skill must be useful while it runs, not just after it finishes. Assume the user may have little coding experience and may not know what a branch, pull request, check, deployment, or Vercel preview means.

Before every meaningful step, explain:

- **What I am about to do:** one plain-English sentence.
- **Why it matters:** the release risk or decision the step protects.
- **Whether it changes anything:** say `read-only`, `creates a PR`, `updates a PR`, `merges`, or `deployment verification only`.
- **What evidence or links I expect to return:** PR URL, compare URL, check links, deployment links, logs, or runtime URL.

After every meaningful step, explain:

- **What happened:** success, no-op, blocker, waiting state, or failure.
- **What it means:** ready for the next step, no PR needed, needs review, checks failed, conflict, missing input, or safe to stop.
- **Useful links:** include every applicable URL discovered during the step.
- **What the user should do next:** review a PR, wait for checks, approve, ask a release owner, provide missing project input, or take no action.

Use short, consistent blocks such as:

```text
Next step: I am going to check whether GitHub has both release branches.
Why: This prevents creating a PR in the wrong direction.
Changes anything?: No, this is read-only.
Expected evidence: branch names and GitHub compare/PR links if available.
```

```text
Result: The promotion PR is ready for review.
What it means: GitHub now has a review page where people can inspect the change before it is merged.
Links: PR, Files changed, Checks, compare, deployment preview if available.
What you should do next: Open the PR link, read the Summary, review Files changed, and check whether required checks are green before approving or asking the release owner to continue.
```

This workflow may merge after readiness gates pass, so explain any merge request or merge result especially clearly.

### Pull Request Handoff Instructions

Whenever you provide a PR URL, also explain what the user is supposed to do with it:

1. Open the PR URL.
2. Read the PR description sections: Summary, Scope, Diff evidence, Verification/Handoff status, and Unknowns.
3. Open **Files changed** to see what code or configuration changed.
4. Open **Checks** to see whether CI/build/deployment checks passed, failed, or are still running.
5. Read comments and reviewer threads for unresolved questions.
6. Do not delete the source or target branch after merge; ignore any GitHub branch-deletion prompt for these permanent promotion branches.
7. Follow the project policy for approval, merge, deployment, or escalation.

If you are waiting for merge approval, ask the user to review the PR URL, confirm that checks and reviews are acceptable, and explicitly approve the merge. If the PR is already merged, explain that the PR link is evidence/history rather than an action request.

### Useful Link Inventory

Collect and report links whenever they are applicable and discoverable:

| Link Type | What to Provide | Why It Helps |
|-----------|-----------------|--------------|
| GitHub PR | PR URL plus PR number and title | Main review and handoff page |
| GitHub Files changed | PR files URL when known | Shows exactly what changed |
| GitHub Checks | Required/failing/pending check URLs | Shows build/test/review status |
| GitHub compare | Source-target compare URL | Shows branch delta before PR creation |
| GitHub Actions/CI | Workflow run URLs from check output | Lets users inspect failing jobs |
| Vercel deployment | Deployment URL and deployment detail/dashboard URL when available | Shows preview/build status and runtime entrypoint |
| Vercel project dashboard | Project dashboard URL when scope/project are known and verified | Lets users inspect deployments, logs, and settings |
| Vercel logs | Build/runtime log URL or exact command used when URL is unavailable | Explains failed or suspicious deployments |
| Runtime/health URL | Project-supplied smoke or health URL | Shows user-facing status |
| Provider dashboard | Render, Netlify, cloud, or internal dashboard links when relevant | Lets release owners continue investigation |

Do not invent links. If a useful link cannot be discovered, say exactly which link is missing and what input is needed to get it, such as Vercel scope, project name, deployment URL, or provider dashboard access.


## Repository Topology Modes

Default to **monorepo mode** unless project docs or the operator identify multiple repos or submodules.

| Mode | How to Model Scope | Promotion Unit | Reporting Unit |
|------|--------------------|----------------|----------------|
| **Monorepo default** | `REPOS=("<owner>/<monorepo>")` plus a release-surface list | One PR in one repository | Per app/package/workspace/deployment surface |
| **Multi-repo** | Multiple independent repositories in `REPOS` | One PR per repository | Per repository plus shared release surface |
| **Submodule repo** | Root repo plus submodule repos when project policy requires both | Root PR and/or submodule PRs, depending on policy | Per root/submodule repo and per deployed surface |

For monorepos, identify the release surfaces before PR review and deployment verification:

- app/workspace/package paths changed by the PR
- shared libraries that can affect multiple apps
- lockfiles and dependency manifests
- database migrations or generated clients
- infrastructure, CI/CD, routing, and environment files
- deployment-provider projects mapped to each app path

For repos with submodules, do not assume the root branch promotion updates submodule branches. Confirm whether the project requires:

1. submodule repo source-target PRs,
2. a root repo PR updating submodule pointers,
3. both, or
4. root-only tracking with submodule pointers unchanged.

Report the topology choice explicitly so reviewers know whether the promotion was monorepo-scoped, multi-repo-scoped, or submodule-aware.


## Quick Commands

```bash
# Branch defaults are built in; override only for non-standard projects after verification.
SOURCE_BRANCH="${SOURCE_BRANCH:-main}"
TARGET_BRANCH="${TARGET_BRANCH:-development}"
PR_TITLE="${PR_TITLE:-chore(release): sync ${SOURCE_BRANCH} into ${TARGET_BRANCH}}"
PR_BODY_FILE="${PR_BODY_FILE:-$(mktemp)}"
# Monorepo default: use a one-entry REPOS array. Add more repos only when the project is multi-repo or submodule-aware.
REPOS=("<owner>/<monorepo-or-repo>")

# Branch discovery.
for REPO in "${REPOS[@]}"; do
  git ls-remote --heads "git@github.com:${REPO}.git" "$SOURCE_BRANCH" "$TARGET_BRANCH"
done

# Existing PR inspection before creation.
gh pr list -B "$TARGET_BRANCH" -H "$SOURCE_BRANCH" -R "<owner>/<repo>" \
  --state open --json number,url,title,body,headRefOid

# Infer the PR body from verified promotion evidence before creation.
cat > "$PR_BODY_FILE" <<'EOF'
## Summary
Promotes `<source-branch>` into `<target-branch>` for `<owner>/<repo>`.

## Scope
- Repository topology: `<monorepo|multi-repo|submodule-aware>`
- Release surfaces: `<apps/packages/deployments inferred from changed paths>`

## Diff evidence
- Source SHA: `<source-sha>`
- Target SHA: `<target-sha>`
- Changed files / risk notes: `<summary>`

## Verification plan
- Review GitHub conversation, files, reviews, and required checks before merge.
- After merge, correlate CI/CD, deployment provider evidence, and runtime checks when relevant.

## Unknowns / follow-up
- `<none or explicit missing project inputs>`
EOF

# Create only when no matching PR exists and the branches differ.
gh pr create -B "$TARGET_BRANCH" -H "$SOURCE_BRANCH" -R "<owner>/<repo>" \
  --title "$PR_TITLE" --body-file "$PR_BODY_FILE"

# Inspect readiness before merge.
gh pr view <number> -R "<owner>/<repo>" \
  --json state,mergeable,mergeStateStatus,comments,reviews,url,headRefOid,reviewDecision,statusCheckRollup
gh pr checks <number> -R "<owner>/<repo>" --watch=false

# Merge using the project-approved method; this example preserves the source branch.
gh pr merge <number> -R "<owner>/<repo>" --merge --delete-branch=false
```

For the full reusable command template, see `references/main-to-development-reference.md`.

## Common Misconceptions

| # | Misconception | Correction | Key concept |
|---|---------------|------------|-------------|
| 1 | Only the recently changed repository needs promotion. | Promote the full project-defined release set or explicitly report excluded repos. | Full-surface promotion |
| 2 | A successful HTTP health response proves the deploy is current. | Correlate the deployed commit, deploy record, logs, and health payload before declaring success. | Deep-lie verification |
| 3 | Branch conflicts can be resolved in any local checkout. | Resolve conflicts only in an isolated clean clone or explicitly approved safe branch workflow. | Workspace safety |
| 4 | Domain aliases and environment routes always follow new deploys. | Verify routing, aliases, protection, and auth/CORS behavior when the project uses them. | Route drift |
| 5 | Project-specific service IDs can be embedded in a reusable skill. | Keep service IDs, domains, repo names, and credentials in project docs or operator input. | Agnostic guidance |
| 6 | Deployment verification is the same as merge verification. | A merge is only complete after provider state and runtime evidence match the intended SHA. | Release evidence |

## Non-Negotiable Policy

1. Treat repository names, domains, service IDs, environment names, credentials, and health endpoints as project inputs, never as built-in knowledge.
2. Promote the full repository set and release-surface set requested by the project; if either set is unknown, pause mutations and report that gap.
3. Check existing PRs before creating new ones.
4. Infer PR titles and descriptions from verified promotion evidence before creation; do not create empty-body PRs by default.
5. Explain each step before and after it runs, and include useful links plus user next actions in every handoff.
6. Inspect PR mergeability, status checks, reviews, comments, and head SHA before merging.
7. Do not resolve conflicts in a dirty workspace. Use an isolated temporary clone or a project-approved safe branch workflow.
8. Do not force-push or directly update target branches unless the user explicitly confirms the exact repository, branch, and expected discarded commits.
9. After merge, verify CI/CD and runtime state by correlating commit SHA, deployment record, logs, timestamps, and health responses.
10. Report every repository's mutation, verification evidence, and unresolved anomaly.

## Workflow

1. **Classify the promotion.** Confirm this is `main` or equivalent mainline source into `development` or equivalent integration target.
2. **Load project release inputs.** Use supplied instructions, local release docs, or read-only discovery. Default to monorepo topology when only one repo is identified. Do not invent repo lists, release surfaces, or endpoints.
3. **Verify branch existence.** For every repo, confirm source and target branches exist remotely. For monorepos, this is usually one repo; for submodules, verify the root and each required submodule repo according to project policy.
4. **Check existing PRs.** Reuse matching open PRs; create missing PRs only when the source has changes not already represented. For monorepos, one PR can cover multiple release surfaces.
5. **Infer PR metadata before creation.** Generate the active title and description from the lane, repo, source/target SHAs, changed paths, impacted release surfaces, and verification plan; use project-provided metadata rules only when documented.
6. **Inspect PR readiness.** Capture URL, number, title/body, head SHA, mergeability, review state, comments, checks, branch-protection notes, changed paths, and impacted release surfaces.
7. **Merge clean PRs.** Use the project-approved merge method and preserve source branches unless the project says otherwise.
8. **Resolve conflicts only if authorized.** Use an isolated clone, push the corrected target branch or PR branch only after explicit conflict policy confirmation, then re-check.
9. **Verify deployments.** Inspect provider state for the merged SHA, including success/failure, timestamps, environment, routing, logs, and per-surface deployment coverage. In monorepos, map each impacted app/package to its deployment project.
10. **Run runtime checks.** Use project-supplied smoke tests, health endpoints, browser checks, auth/CORS checks, or alias checks.
11. **Close out.** Report merged PRs, skipped/no-diff repos, conflicts, provider evidence, runtime evidence, and any follow-up.

## Reference Loading by Task Type

| Task type | Load first | Skip |
|-----------|------------|------|
| Promotion execution | `references/main-to-development-reference.md` | Provider-specific docs until repos are known |
| Conflict resolution | `references/main-to-development-reference.md` conflict section | Runtime checks until merge is clean |
| Deployment verification | `references/main-to-development-reference.md` verification section | PR creation templates |
| Root tracking branch | `references/main-to-development-reference.md` tracking section | Provider sections |
| Runtime smoke testing | Project docs and supplied endpoint list | Guessing endpoints |
| GitHub PR review | PR page plus `references/main-to-development-reference.md` PR review section | Merge commands until review is clean |
| Vercel deployment analysis | `references/main-to-development-reference.md` Vercel section plus project Vercel inputs | Vercel mutation/alias commands |
| Infrastructure contract validation | `references/main-to-development-reference.md` contract section plus project IaC docs | Provider apply/sync commands |
| Routing or alias drift | Project route map plus `references/main-to-development-reference.md` routing section | Hardcoded domains |

## Verification Surface Inventory

Before post-merge verification, build a project-specific inventory with these columns:

| Surface | Required Evidence | Notes |
|---------|-------------------|-------|
| Source control | Merged PR number, URL, and final target SHA | Use `headRefOid` for PR head SHA capture. |
| CI/CD provider | Build/deploy record for the merged SHA | Verify status, timestamp, environment, and actor. |
| Runtime health | Health or smoke-test payload from the active environment | Correlate payload time/version/deploy ID when available. |
| Routing/alias layer | Domains, aliases, load balancers, or ingress routes point to the expected deploy | Check non-destructively before applying changes. |
| Auth/protection layer | Preview protection, auth callbacks, CORS, or credentials behavior match the environment policy | Only run when relevant to the project. |
| Infrastructure contract | IaC or provider blueprint validates cleanly | Contract validation is not an apply/sync operation. |


## GitHub PR Review Guidance

Before any merge, review the PR itself, not just the source and target branch names:

1. **Conversation:** read the description, comments, bot/deployment assistant notes, and unresolved reviewer threads.
2. **Commit identity:** capture `headRefOid`, base branch, target branch, author, and whether the PR is a draft or cross-repository PR.
3. **Files changed:** inspect file names and patch risk; look for generated files, lockfiles, migrations, infrastructure files, secrets, or environment changes.
4. **Checks and deployments:** inspect required checks, optional checks, and deployment statuses from the PR page or CLI.
5. **Reviews:** identify approvals, changes-requested reviews, stale approvals, and requested reviewers.
6. **Merge control:** do not press a web merge button or call `gh pr merge` until the project-approved merge policy is satisfied.

Useful read-only commands:

```bash
gh pr view <number> -R "<owner>/<repo>" --comments \
  --json state,isDraft,author,title,body,baseRefName,headRefName,headRefOid,mergeable,mergeStateStatus,reviewDecision,reviewRequests,latestReviews,comments,reviews,statusCheckRollup,files,commits,url
gh pr diff <number> -R "<owner>/<repo>" --name-only
gh pr diff <number> -R "<owner>/<repo>" --patch
gh pr checks <number> -R "<owner>/<repo>" --watch=false --json name,state,bucket,link,workflow,startedAt,completedAt
gh pr view <number> -R "<owner>/<repo>" --web
```

For large diffs, inspect file names first, then targeted patches for risky files. If the PR page shows deployments or provider comments that the CLI summary does not expose, use the browser view and include those observations in the closeout.

## Infrastructure and Routing Guardrails

- Treat infrastructure-as-code, deployment blueprints, route maps, and environment manifests as project inputs.
- Validate contracts locally or with a read-only provider command when the project requires it.
- Do **not** apply, sync, or mutate infrastructure as part of ordinary `main` → `development` promotion unless the user explicitly requests that infrastructure change.
- For routing or alias drift, use a non-destructive check first. Apply corrections only when the project has supplied the command and the operator authorizes mutation.
- Verify provider CLI output shape before parsing logs or JSON. Some providers emit nested objects, mixed stdout/stderr, pagination, or newline-delimited JSON.


## Vercel Deployment Analysis When Relevant

Use this only when the project uses Vercel for the promoted surface or when GitHub checks/deployments link to Vercel. Vercel analysis is evidence gathering; it does not replace source-control checks.

1. **Identify the deployment:** get the deployment URL or deployment ID from GitHub checks, PR deployment cards, Vercel comments, project docs, or operator input.
2. **Inspect deployment metadata:** confirm project, environment, branch, status, age, creator, aliases, and any Git metadata exposed by the current CLI output.
3. **Compare to GitHub:** verify the Vercel deployment corresponds to the PR head SHA or merged target SHA expected for this promotion.
4. **Inspect build logs:** use deployment logs for build failures, warnings, skipped builds, environment-variable issues, or framework errors.
5. **Inspect runtime logs:** use scoped recent logs for runtime errors, middleware/edge failures, serverless exceptions, 4xx/5xx spikes, or request-specific failures.
6. **Check routing:** if aliases, custom domains, or preview-protection rules are relevant, verify the project-supplied route map non-destructively before any correction.
7. **Report Vercel links:** include the GitHub deployment/check link, Vercel deployment URL, Vercel deployment detail/dashboard URL when available, Vercel project dashboard URL when verified, and any log URL or exact log command used.
8. **Report uncertainty:** Vercel output fields can vary by CLI version, project settings, and deployment type. Record the exact command and evidence rather than assuming field names.

Reusable command templates, verified against local Vercel CLI help during authoring:

```bash
vercel inspect <deployment-url-or-id> --format=json --scope <scope>
vercel inspect <deployment-url-or-id> --logs --scope <scope>
vercel list <project-name> --environment preview --status READY --format=json --scope <scope>
vercel logs --deployment <deployment-url-or-id> --json --since 30m --scope <scope>
vercel logs --project <project-name> --environment preview --branch <branch-name> --level error --since 1h --json --scope <scope>
```

Replace `<scope>`, `<project-name>`, deployment IDs, branch names, and environments with project inputs. If no Vercel scope or project is known, stop at GitHub deployment evidence and ask for the missing project input.

### Monorepo Vercel Mapping

In monorepos, one GitHub PR can produce multiple Vercel deployments from the same commit. Build a mapping before declaring Vercel evidence complete:

| App/Package Path | Vercel Project | Deployment URL/ID | Environment | Branch | Commit/SHA Evidence | Status |
|------------------|----------------|-------------------|-------------|--------|---------------------|--------|
| `<apps/web>` | `<vercel-project>` | `<deployment>` | preview/production | `<branch>` | `<sha/source>` | ready/error/pending |

Use the PR changed-file list to decide which Vercel projects matter. If path filters skip a relevant project, report that as a verification gap rather than assuming it is safe.


## Quality Checklist

| # | Checklist Item | Why It Matters | Gate |
|---|---------------|---------------|------|
| 1 | Repository set identified | Prevents partial promotion | Pre-op |
| 2 | Default source/target branches verified remotely | Prevents wrong-direction PRs | Pre-op |
| 3 | Existing PRs inspected | Avoids duplicates | Pre-op |
| 4 | Plain-language step explanations and useful links provided | Keeps non-technical users oriented | Every step |
| 5 | PR title/body inferred from evidence before creation | Avoids empty or misleading release PRs | Draft |
| 6 | Mergeability, diff risk, and reviews checked | Respects branch protection and review quality | Draft |
| 7 | Conflicts isolated | Protects local worktree | Draft |
| 8 | Deployment SHA matched, including Vercel when relevant | Prevents stale deploy claims | Closeout |
| 9 | Runtime checks correlated | Verifies user-facing state | Closeout |
| 10 | Anomalies reported | Enables follow-up | Closeout |

### Quality Tiers

| Tier | Criteria | Use When |
|------|----------|----------|
| **Minimal** | Items 1-4 and 10 | Readiness check or dry-run planning |
| **Standard** | Items 1-7 and 10 | Merge-focused promotion with provider checks pending |
| **Full** | All 10 items plus verification surface inventory | Complete promotion with runtime evidence |

### Pre-Op Verification

```text
□ Repository set is known and explicitly in scope.
□ Default source and target branch names are accepted or a documented override exists.
□ User-facing progress updates will be given before and after each meaningful step.
□ Existing PRs will be reused before new PRs are created.
□ PR title and body will be inferred from lane, diff, release-surface, and verification evidence.
□ Merge method and conflict policy are known.
□ Root/meta repository tracking policy is known.
□ Deployment, runtime, route, and infra-contract checks are supplied or explicitly out of scope.
```

## Consistency Validator

Before finalizing, verify:

| Check | What to Verify | How to Fix |
|-------|----------------|------------|
| Scope vs repository set | Every requested repo was handled | Add missing repos or report exclusion |
| Monorepo release surfaces | Every impacted app/package/deployment was reviewed | Add path/deployment mapping or report unknowns |
| Submodule policy | Required submodule PRs or pointer updates were handled | Add missing submodule/root PR or report exclusion |
| Direction vs branch names | Source is mainline, target is development | Stop and recreate correct PRs |
| PR metadata | Created PRs have inferred title/body with lane, diff, scope, verification plan, and unknowns | Edit PR body/title or report blocker |
| Merge vs readiness | Only ready PRs were merged | Revert/repair per project policy |
| Deploy vs SHA | Active deploy matches merged SHA | Wait, redeploy, or report stale state |
| Health vs logs | Runtime response matches verified deploy | Investigate stale instance or routing drift |

### Red Flags

- [ ] Repository list inferred from memory.
- [ ] PR created with an empty or generic body instead of inferred release evidence.
- [ ] Monorepo treated as “done” without app/package/deployment surface mapping.
- [ ] Root repo promotion assumed to update submodule repos without policy confirmation.
- [ ] Conflict resolved in a dirty working tree.
- [ ] Direct target-branch push without explicit authorization.
- [ ] Health `200` accepted without SHA/deploy correlation.
- [ ] Project-specific IDs embedded into this reusable skill.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| PR creation reports no diff | Branches are already equal | Report no PR needed for that repo |
| PR is conflicting | Source and target diverged | Use isolated clone only after authorization |
| Checks are pending indefinitely | CI queue or required external approval | Report pending checks and URLs |
| Deploy provider shows old SHA | Stale deploy, failed build, or alias drift | Correlate provider deploy list, logs, and routing |
| Health endpoint returns success but old payload | Stale instance or cached route | Compare deploy ID/time/SHA and rerun checks |
| CLI flags fail | Local tool version differs | Run current help or official docs before retrying |
| PR becomes redundant after conflict repair | Target branch was updated directly through an authorized repair | Re-check source-target diff and close the PR only if authorized. |
| Infra validation command wants to apply changes | Contract validation and infra mutation are mixed by the provider | Stop before apply/sync and request explicit authorization. |

## Cross-Skill Coordination

- Use a GitHub PR-review or browser skill when authenticated web UI review is required for conversation threads, deployment cards, or files-changed navigation.
- Use a browser-validation or smoke-test skill after merge when the project needs user-facing evidence.
- Use a deployment-provider-specific skill when provider CLI setup, logs, aliases, or infrastructure validation require deep platform knowledge.
- Use a study or decision skill when branch policy, release scope, or conflict-resolution authority is unclear.
- Use a monorepo/workspace-aware build or package-manager skill when path ownership, workspace graphs, or package-level tests determine release scope.
- Use a git/submodule skill when the repository set includes nested repositories, submodules, or meta/root tracking branches.

## Guidance Alignment

- Keep this skill project-agnostic; project-specific release surfaces belong in project docs or operator-provided inputs.
- After editing this skill in a skills host repo, run that repo's skill sync and skill validation commands.
- Do not add real repository names, service IDs, domains, credentials, or provider resource IDs to this reusable skill.

## Common Pitfalls

1. Promoting only one repository from a multi-repo release set.
2. Treating a monorepo as a single undifferentiated deploy surface instead of mapping apps/packages/deployments.
3. Creating duplicate PRs instead of reusing existing source-target PRs.
4. Creating PRs with empty descriptions or titles not inferred from the active lane and diff evidence.
5. Merging before inspecting reviews and required checks.
5. Solving conflicts in the operator's dirty checkout.
6. Treating deployment success as runtime success without smoke checks.
7. Encoding a specific project's domains, service IDs, or credentials in a generic skill.
8. Omitting a per-repo and per-release-surface closeout table.

## Temporary Files

If this skill needs temporary files, place them under `.tmp/promote-main-to-development/YYYY-MM-DD-{subject}` unless the host project gives a stricter temp-file convention. Do not create top-level dotfile temp directories.

## Local Corpus Layout

The `references/` directory contains one hand-authored operational file:

| File | Purpose |
|------|---------|
| `main-to-development-reference.md` | Generic command templates for branch discovery, PR create/reuse, merge, isolated conflict resolution, root tracking, deployment verification, and reporting. |
