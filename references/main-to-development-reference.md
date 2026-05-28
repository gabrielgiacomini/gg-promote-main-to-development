# Main to Development Promotion Reference

This reference contains project-agnostic command templates. Replace placeholders with project-supplied values before running anything.

## Shell Variables

```bash
# Defaults for this promotion lane. Override only when project docs explicitly use different branch names.
SOURCE_BRANCH="${SOURCE_BRANCH:-main}"
TARGET_BRANCH="${TARGET_BRANCH:-development}"
PR_TITLE="${PR_TITLE:-chore(release): sync ${SOURCE_BRANCH} into ${TARGET_BRANCH}}"
PR_BODY_FILE="${PR_BODY_FILE:-$(mktemp)}"
# Fill this with the project-defined full release set.
REPOS=("<owner>/<repo-one>" "<owner>/<repo-two>")
```


## Repository Topology Setup

Default to monorepo mode. For a monorepo, use one repository and track release surfaces separately:

```bash
REPOS=("<owner>/<monorepo>")
RELEASE_SURFACES=("<apps/web>" "<packages/shared>")
```

For multi-repo projects, list each independent repository:

```bash
REPOS=("<owner>/<repo-one>" "<owner>/<repo-two>")
```

For submodule-aware projects, project policy determines whether to include root repo, submodule repos, or both:

```bash
ROOT_REPO="<owner>/<root-repo>"
SUBMODULE_REPOS=("<owner>/<submodule-one>" "<owner>/<submodule-two>")
REPOS=("$ROOT_REPO" "${SUBMODULE_REPOS[@]}")
```

Do not assume a root repository PR updates submodule repository branches. Confirm whether the release requires submodule source-target PRs, root pointer updates, both, or neither.

## Branch Discovery

```bash
for REPO in "${REPOS[@]}"; do
  git ls-remote --heads "git@github.com:${REPO}.git" "$SOURCE_BRANCH" "$TARGET_BRANCH"
done
```

Use the default `main` and `development` branch names unless the project explicitly documents a different ladder. Stop before mutations if either required branch is missing, ambiguous, or unexpectedly protected.

## Branch Preservation

Treat both `SOURCE_BRANCH` and `TARGET_BRANCH` as permanent release branches. Do not delete either branch, use merge cleanup flags that delete either branch, or instruct the user to use GitHub's **Delete branch** button unless a documented project policy explicitly requires it and the user authorizes the exact repository and branch.

When reporting a PR or merge result, tell the user that any provider branch-deletion prompt should be ignored for these source and target branches because they are supposed to remain available for future promotions.

## Plain-Language Progress and Link Reporting Template

Use this reference with user-facing narration. Before each command or mutation, explain in simple language what is about to happen, why it matters, whether it changes anything, and what evidence or links it should produce. After each command, explain the result, what it means, the next action, and all useful links.

Minimum step-update format:

```text
Next step: <plain-English action>
Why: <risk or decision this protects>
Changes anything?: <read-only | creates PR | updates PR | merges | verification only>
Expected evidence: <PR URL | compare URL | checks | deployment URL | logs | health URL>
```

```text
Result: <success | no-op | blocker | pending | failure>
What it means: <beginner-friendly interpretation>
Links: <GitHub PR, Files changed, Checks, compare, Vercel deployment/dashboard, logs, runtime URL>
What you should do next: <review PR | wait for checks | approve | ask release owner | provide missing input | no action>
```

PR handoff explanation:

```text
Here is the pull request: <pr-url>
What this is: a GitHub review page for the `main → development` promotion.
What to do: open it, read the description, inspect Files changed, check the Checks tab, and read comments.
Boundary: May merge only after project-approved readiness gates are satisfied.
Branch precaution: do not delete the `main` or `development` branches if GitHub offers branch cleanup; these are permanent promotion branches.
If something is red or pending: do not approve/merge yet; use the blocker list below.
```

Useful links to collect when applicable:

```bash
COMPARE_URL="https://github.com/${REPO}/compare/${TARGET_BRANCH}...${SOURCE_BRANCH}"
gh pr view <number> -R "<owner>/<repo>" --json url,number,title,body,headRefOid,baseRefName,headRefName,statusCheckRollup
# Check objects often include direct CI/deployment links.
gh pr checks <number> -R "<owner>/<repo>" --watch=false --json name,state,bucket,link,workflow,startedAt,completedAt
```

For Vercel, report exact links exposed by GitHub deployment cards, Vercel comments, or Vercel CLI output. If scope/project/deployment are known and verified, include the deployment URL and dashboard/project URL; otherwise say which Vercel input is missing instead of guessing.

## PR Create or Reuse

List existing open PRs first:

```bash
for REPO in "${REPOS[@]}"; do
  gh pr list -B "$TARGET_BRANCH" -H "$SOURCE_BRANCH" -R "$REPO" \
    --state open --json number,url,title,body,headRefOid
done
```

Infer the PR title and body before creating anything. Use the default lane-derived title unless project docs define an exact metadata contract. Build the body from verified branch, diff, topology, release-surface, and verification evidence:

```bash
cat > "$PR_BODY_FILE" <<'EOF'
## Summary
Promotes `<source-branch>` into `<target-branch>` for `<owner>/<repo>`.

## Scope
- Repository topology: `<monorepo|multi-repo|submodule-aware>`
- Release surfaces: `<apps/packages/deployments inferred from changed paths>`

## Diff evidence
- Source SHA: `<source-sha>`
- Target SHA: `<target-sha>`
- Compare or commit summary: `<compare-url-or-commit-count>`
- Risk notes: `<migrations/lockfiles/infra/generated/env/secrets/none>`

## Verification plan
- Review GitHub conversation, files, reviews, and required checks before merge.
- After merge, correlate CI/CD, deployment provider evidence, and runtime checks when relevant.

## Unknowns / follow-up
- `<none or explicit missing project inputs>`
EOF
```

Create only when no suitable open PR exists and there is a diff:

```bash
gh pr create -B "$TARGET_BRANCH" -H "$SOURCE_BRANCH" -R "<owner>/<repo>" \
  --title "$PR_TITLE" --body-file "$PR_BODY_FILE"
```

Do not use an empty body unless project policy explicitly requires it.

Inspect directly before merging:

```bash
gh pr view <number> -R "<owner>/<repo>" \
  --json state,mergeable,mergeStateStatus,comments,reviews,url,headRefOid,reviewDecision,statusCheckRollup
gh pr checks <number> -R "<owner>/<repo>" --watch=false
```

Use `headRefOid` for the PR head commit SHA.


## GitHub PR Review Commands

Use these commands to build the PR review packet. Verify command flags with `gh --help` when local versions differ.

```bash
gh pr view <number> -R "<owner>/<repo>" --comments \
  --json state,isDraft,author,title,body,baseRefName,headRefName,headRefOid,mergeable,mergeStateStatus,reviewDecision,reviewRequests,latestReviews,comments,reviews,statusCheckRollup,files,commits,url
gh pr diff <number> -R "<owner>/<repo>" --name-only
gh pr diff <number> -R "<owner>/<repo>" --patch
gh pr checks <number> -R "<owner>/<repo>" --watch=false --json name,state,bucket,link,workflow,startedAt,completedAt
gh pr checks <number> -R "<owner>/<repo>" --required --watch=false
gh pr view <number> -R "<owner>/<repo>" --web
```

Review packet fields to report:

1. PR URL, number, title, draft state, source branch, target branch, and `headRefOid`.
2. Author, latest reviews, requested reviewers, review decision, and unresolved discussion risk.
3. Changed file count and risky paths: infrastructure, dependencies, lockfiles, generated files, migrations, secrets, environment files, release scripts, CI/CD config, and monorepo app/package ownership.
4. Required and optional check state, including failing or pending check links.
5. Deployment cards, preview links, or provider comments shown on GitHub that are not captured by CLI JSON, mapped to monorepo release surfaces when applicable.
6. Human-readable recommendation: ready, blocked, no diff, waiting for checks, waiting for review, title mismatch, or conflict.

## Merge Clean PRs

Use the project-approved merge method. A common default is:

```bash
gh pr merge <number> -R "<owner>/<repo>" --merge --delete-branch=false
```

Do not use admin overrides unless the project explicitly requires them. Do not change `--delete-branch=false` for source-target promotion branches unless the project explicitly requires deletion and the user authorizes the exact repository and branch.

## Isolated Conflict Resolution

Use only after the user authorizes conflict resolution for the affected repo:

```bash
WORKDIR=$(mktemp -d)
git clone "git@github.com:<owner>/<repo>.git" "$WORKDIR/repo"
cd "$WORKDIR/repo"
git fetch origin "$SOURCE_BRANCH" "$TARGET_BRANCH"
git checkout "$TARGET_BRANCH"
git merge "origin/$SOURCE_BRANCH"
# Resolve conflicts, run project-required tests, then push only if approved.
git push origin "$TARGET_BRANCH"
```

Never run this in an unrelated dirty checkout. After a direct target-branch repair, re-check the original PR. If the PR became redundant because the branches are now equal, close it only when the user authorizes that cleanup.

## Optional Root Tracking Branch

When a root/meta repository tracks deploy state by branch, run a divergence check before any target-branch overwrite:

```bash
REMOTE_TARGET_SHA=$(git ls-remote origin "refs/heads/$TARGET_BRANCH" | cut -f1)
SOURCE_SHA=$(git rev-parse "refs/heads/$SOURCE_BRANCH")
STALE=$(git rev-list --count "$SOURCE_SHA".."$REMOTE_TARGET_SHA" 2>/dev/null || echo "0")
AHEAD=$(git rev-list --count "$REMOTE_TARGET_SHA".."$SOURCE_SHA" 2>/dev/null || echo "0")
echo "Remote target: $REMOTE_TARGET_SHA"
echo "Source:        $SOURCE_SHA"
echo "Target-only commits that may be discarded: $STALE"
echo "Source commits not on target: $AHEAD"
```

If target-only commits exist, stop and get explicit confirmation before using `--force-with-lease`.

## Deployment Verification Template

For each repo/environment, gather evidence from the project's provider:

1. Expected commit SHA from merged PR.
2. Deployment record for the target environment.
3. Deployment status and timestamp.
4. Logs for startup/build/runtime errors.
5. Runtime health response or smoke-test result.
6. Route/alias/auth/protection checks when applicable.
7. User-facing links: PR URL, compare URL, CI/check URLs, deployment/dashboard URLs, log links, and runtime/health URLs when applicable.

Provider commands are project-specific; examples may include source-control CI commands, deployment-provider CLIs, cloud CLIs, or internal release tooling. Do not invent provider IDs or endpoints.


## Vercel Deployment Analysis Template

Use this only when the project identifies Vercel as a deployment provider for the promoted surface. All identifiers are project inputs.

```bash
vercel inspect <deployment-url-or-id> --format=json --scope <scope>
vercel inspect <deployment-url-or-id> --logs --scope <scope>
vercel list <project-name> --environment preview --status READY --format=json --scope <scope>
vercel logs --deployment <deployment-url-or-id> --json --since 30m --scope <scope>
vercel logs --project <project-name> --environment preview --branch <branch-name> --level error --since 1h --json --scope <scope>
```

Analysis checklist:

1. Capture the deployment URL/ID from GitHub checks, deployment cards, PR comments, project docs, or operator input.
2. Use `vercel inspect --format=json` and inspect the actual JSON keys before scripting assumptions.
3. Confirm environment, project, branch, status, age, creator, aliases, and Git metadata when exposed.
4. Compare the deployment commit/branch metadata to the expected PR head SHA or merged target SHA.
5. Use `vercel inspect --logs` for build logs and `vercel logs --json` for runtime logs.
6. Treat `vercel logs --json` output as JSON Lines suitable for line-by-line parsing.
7. Report build failures, runtime 4xx/5xx patterns, middleware/edge errors, serverless exceptions, environment-variable errors, skipped builds, and stale deployments.
8. Include Vercel dashboard/project links, deployment URLs, GitHub deployment/check links, and log links or exact log commands when available.
9. Do not mutate aliases, domains, environment variables, or protection settings unless the user explicitly asks for a separate provider mutation workflow.

## Routing and Alias Drift Template

When the project has public routes, aliases, ingress rules, load balancers, preview-protection settings, or auth callback hosts, verify them separately from deployment status:

1. Run the project's non-destructive route/alias check.
2. Confirm each route points to the intended environment and deployment SHA/version where that data is available.
3. Check auth, CORS, preview protection, or credentialed endpoints only when the project defines those contracts.
4. Apply route or alias corrections only with explicit authorization and a project-supplied command.
5. Re-run the non-destructive check after any correction.

## Infrastructure Contract Validation Template

When the project uses infrastructure-as-code or provider blueprints:

1. Identify the canonical contract file or provider resource from project docs.
2. Run a validation/diff command that does not apply changes.
3. Report validation errors as release blockers or follow-up items.
4. Do not run apply/sync/deploy infrastructure commands unless the user explicitly requested an infrastructure mutation.

## Provider Output Parsing Notes

Before scripting provider checks, verify the current CLI output shape. Providers may return nested JSON, paginated results, mixed text plus JSON, or newline-delimited JSON logs. Parse line-by-line when a log command emits one JSON object per line, and avoid assuming fields are top-level strings.

## Deep-Lie Verification

Do not accept a green probe alone. Closeout evidence should connect:

```text
merged PR SHA -> CI/build SHA -> deployment record SHA -> deploy logs -> runtime health payload
```

If any link is stale, missing, or contradictory, report the exact gap.

## Reporting Checklist

Start every report with:

- Plain-language result.
- What it means.
- What the user should do next.
- Useful links for the user.

Per repository, report:

1. Existing PR reused, new PR created, merged, skipped, or no diff.
2. PR number and URL.
3. Head SHA and resulting target SHA when available.
4. Review/check state before merge.
5. Conflict resolution path, if any.
6. Deployment record and provider status.
7. Runtime health/smoke evidence.
8. Monorepo release-surface mapping or submodule/root policy result.
9. Remaining blockers or anomalies.
10. Useful links for the user: PR URL, Files changed URL, Checks/CI links, compare URL, deployment/dashboard links, log links, and runtime/health URLs when applicable.
