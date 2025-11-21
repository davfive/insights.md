# GitHub Branch Protections Guide

A practical, opinionated guide to configuring branch protections on GitHub for a safe, efficient workflow. This is written for someone familiar with GitLab concepts (merge request approvals, pipelines) but new to GitHub branch protections.

> Goal: protect `main` (and other release branches) so only safe, reviewed changes are merged — while allowing an admin (you) to push directly if needed. This guide explains recommended checks, tradeoffs, UI steps, and exact API/CLI commands to apply settings.

---

## Quick summary / recommendations

- Essentials:
  - Require Pull Request (PR) reviews before merging.
  - Require passing CI status checks (choose a small set of canonical checks).
  - Dismiss stale approvals on new commits.
  - Require conversation resolution before merge.
  - Restrict direct pushes so only intended actors (for you: your user) can push.
  - Prevent branch deletion.
- Helpful:
  - Require CODEOWNERS reviews for important paths.
  - Require branches to be up-to-date before merging (strict status checks).
  - Enforce protection for administrators (optional — admins can still change protections).
  - Optionally require signed commits for stronger provenance.

---

## GitLab → GitHub equivalences

- GitLab "Merge request approvals" → GitHub "Require pull request reviews before merging"
- GitLab "Pipelines must succeed" → GitHub "Require status checks to pass before merging"
- GitLab "Protected branches (allowed to push/merge)" → GitHub "Restrict who can push" + branch protection rules
- GitLab "Resolve discussions before merge" → GitHub "Require conversation resolution before merging"

---

## Recommended protection presets

Pick a preset that matches your risk tolerance.

1. Minimal (fast, small teams)
   - Require PR reviews: 1 reviewer
   - Require status checks: basic unit test CI
   - Restrict pushes: your user only
   - Prevent deletion
   - Do NOT enforce for administrators

2. Recommended (balanced)
   - Require PR reviews: 1 (or 2) reviewers
   - Dismiss stale approvals on new commits
   - Require CODEOWNERS reviews for owned paths
   - Require status checks: `python-tests`, `lint`, `security-scan` (pick exact names)
   - Require branches to be up-to-date before merging (strict)
   - Require conversation resolution
   - Restrict pushes: your user (and CI apps if needed)
   - Prevent deletion
   - Enforce for administrators = true

3. Strict / High-security (for critical repos)
   - Require PR reviews: 2+ reviewers
   - Require CODEOWNERS reviews
   - Require status checks: all critical workflows (no exceptions)
   - Require branches to be up-to-date
   - Require signed commits
   - Require conversation resolution
   - Restrict pushes to a tiny admin set
   - Prevent deletion
   - Enforce for administrators = true

---

## Which status checks should I require?

- Choose checks that are reliable and meaningful:
  - CI unit tests (e.g. `python-tests` or the umbrella workflow name).
  - Linting / formatting checks (flake8 / black).
  - Security scans (CodeQL / bandit / safety).
- For matrix jobs, consider whether you need to require every matrix run (can be many checks) or just representative runs (faster merges).
- To find the exact check names:
  - Run a workflow on the branch and observe the "Checks" list on a PR or Actions run.
  - Or use the API / UI to inspect recent workflow run job names (GitHub uses the job name strings as contexts).

---

## UI: How to enable protections (recommended for one-off changes)

1. Go to the repository → Settings → Branches.
2. Under Branch protection rules, add a new rule (or edit existing) for the branch name pattern (e.g. `main`).
3. Enable options you need:
   - Require pull request reviews before merging → set required reviewers count.
     - Check "Dismiss stale pull request approvals when new commits are pushed".
     - Optionally check "Require review from Code Owners".
   - Require status checks to pass before merging → pick the checks you want and enable "Require branches to be up to date before merging" if desired.
   - Require conversation resolution before merging.
   - Restrict who can push to matching branches → add your username (and apps/teams if needed).
   - Do not enable "Allow deletions". If an "allow deletions" option exists, make sure it's not granted to undesired actors.
   - Optionally check "Include administrators" if you want rules to apply to admins.
4. Save.

---

## API / CLI: Exact commands (curl + gh)

Notes:
- You must be an admin on the repository.
- Use a token with `repo` (admin) scope in `$GITHUB_TOKEN` (or set `Authorization: Bearer <PAT>`).
- Replace `OWNER/REPO` and `branch` with your repo and branch names.

1) Require specific status checks (strict = require branch up-to-date)
- Endpoint: PUT /repos/{owner}/{repo}/branches/{branch}/protection/required_status_checks

curl:
```bash
export GITHUB_TOKEN="..."; OWNER=davfive; REPO=gitspaces; BRANCH=main
curl -sS -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/branches/$BRANCH/protection/required_status_checks" \
  -d '{
    "strict": true,
    "contexts": ["python-tests", "security-scan", "lint"]
  }'
```

gh:
```bash
gh api --method PUT repos/$OWNER/$REPO/branches/$BRANCH/protection/required_status_checks \
  -f strict=true \
  -F contexts='["python-tests","security-scan","lint"]'
```

2) Restrict pushes to a single user (only you)
- Endpoint: PUT /repos/{owner}/{repo}/branches/{branch}/protection/restrictions

curl:
```bash
curl -sS -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/branches/$BRANCH/protection/restrictions" \
  -d '{"users":["davfive"],"teams":[],"apps":[]}'
```

gh:
```bash
gh api --method PUT repos/$OWNER/$REPO/branches/$BRANCH/protection/restrictions \
  -f users='["davfive"]' -f teams='[]' -f apps='[]'
```

3) Ensure deletions are not allowed (remove any allow-deletions allowance)
- Endpoint: DELETE /repos/{owner}/{repo}/branches/{branch}/protection/allow_deletions

curl:
```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/branches/$BRANCH/protection/allow_deletions"
```

gh:
```bash
gh api --method DELETE repos/$OWNER/$REPO/branches/$BRANCH/protection/allow_deletions
```

4) Apply enforce_admins (make rules apply to admins)
- Endpoint: PUT /repos/{owner}/{repo}/branches/{branch}/protection/enforce_admins

curl:
```bash
curl -sS -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/branches/$BRANCH/protection/enforce_admins" \
  -d '{"enabled": true}'
```

gh:
```bash
gh api --method PUT repos/$OWNER/$REPO/branches/$BRANCH/protection/enforce_admins -f enabled=true
```

5) Read the existing protection to verify
```bash
curl -sS -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/branches/$BRANCH/protection" | jq .
```

---

## CODEOWNERS and reviews

- Add a `CODEOWNERS` file to automatically request reviews from owners for particular paths.
- Typical pattern:
  - `docs/ @docs-team`
  - `infra/** @infra-team`
- GitHub will add required codeowner reviews when "Require review from Code Owners" is enabled in the branch protection rule.

---

## Practical tips & pitfalls

- Flaky tests are your biggest source of friction. Fix flakiness before making checks required.
- For matrix workflows: requiring every matrix job can multiply “required checks”. Consider requiring only an umbrella check or a subset for faster merges.
- Forked PRs: workflows triggered on PRs from forks might not have access to secrets. Be mindful when requiring checks that depend on secrets.
- Finding exact check names: open a workflow run / PR and inspect the Checks list — the context strings there are what you must use in the `contexts` array.
- If you need CI to push tags/releases or merge on your behalf, add the CI app or a specific GitHub App to the protections `apps` list.

---

## Example: apply a balanced configuration (one-shot)

This sequence:
- sets required status checks (`python-tests`, `security-scan`, `lint`) and strict = true
- restricts pushes to `davfive`
- removes any deletion allowance
- enforces admins

Run (careful — destructive to prior protection config):
```bash
export GITHUB_TOKEN="..."
OWNER=davfive; REPO=gitspaces; BRANCH=main

# required checks
curl -sS -X PUT -H "Authorization: Bearer $GITHUB_TOKEN" -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/branches/$BRANCH/protection/required_status_checks" \
  -d '{"strict": true, "contexts":["python-tests","security-scan","lint"]}'

# restrict pushes to you
curl -sS -X PUT -H "Authorization: Bearer $GITHUB_TOKEN" -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/branches/$BRANCH/protection/restrictions" \
  -d '{"users":["davfive"],"teams":[],"apps":[]}'

# remove deletion allowance (safe to attempt)
curl -sS -X DELETE -H "Authorization: Bearer $GITHUB_TOKEN" -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/branches/$BRANCH/protection/allow_deletions"

# enforce admins
curl -sS -X PUT -H "Authorization: Bearer $GITHUB_TOKEN" -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/branches/$BRANCH/protection/enforce_admins" \
  -d '{"enabled": true}'

# verify
curl -sS -H "Authorization: Bearer $GITHUB_TOKEN" -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/branches/$BRANCH/protection" | jq .
```

---

## Final notes

- Branch protection is about preventing accidental or unchecked changes. Admins can always change protections, so for irrevocable rules you must reduce admin privileges.
- Start with the Recommended preset, run it for a few weeks, watch merge friction (and flaky checks), and iterate.
- If you want, I can generate the exact `gh api` or `curl` commands pre-filled for your repo and branch to apply one of the presets automatically.

---

## Resources

- GitHub branch protection docs: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-branches-in-your-repository/about-protected-branches
- GitHub API: Branch protections endpoints: https://docs.github.com/en/rest/branches/branch-protection
