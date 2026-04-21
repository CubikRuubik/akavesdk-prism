---
description: Analyze merged PR changes and create a description-only PR in each dependent repository
on:
  pull_request:
    types: [closed]
  workflow_dispatch:
    inputs:
      commit_hash:
        description: 'Commit hash to analyze'
        required: true
        type: string

if: github.event.pull_request.merged == true || github.event_name == 'workflow_dispatch'

permissions:
  contents: read
  pull-requests: read
  issues: read

steps:
  - name: Checkout main repository
    uses: actions/checkout@v6
    with:
      persist-credentials: false
  - name: Checkout rust dependent repository workspace
    uses: actions/checkout@v6
    with:
      repository: CubikRuubik/akavesdk-rust-prism
      persist-credentials: false
      path: CubikRuubik/akavesdk-rust-prism
  - name: Checkout python dependent repository workspace
    uses: actions/checkout@v6
    with:
      repository: CubikRuubik/akavesdk-py-prism
      persist-credentials: false
      path: CubikRuubik/akavesdk-py-prism

tools:
  github:
    toolsets: [default, repos, pull_requests]

safe-outputs:
  github-token: ${{ secrets.GH_AW_CROSS_REPO_PAT }}
  create-pull-request:
    max: 10
    allowed-repos:
      - CubikRuubik/akavesdk-rust-prism
      - CubikRuubik/akavesdk-py-prism
    excluded-files:
      - ".github/**"
network:
  allowed:
    - defaults
---

# Sync Changes to Dependent Repositories

You are an AI agent that analyzes changes merged to the go-prism repository and opens pull requests with description of required changes in each dependent repository. You do **not** write or modify any code in the dependent repositories — that is handled by each dependent repo's own workflow.

## Your Task

When triggered by a merged PR, use the PR event context. When triggered manually via `workflow_dispatch`, use the provided `commit_hash` input (`${{ github.event.inputs.commit_hash }}`) to identify the relevant changes.

1. **Analyze the changes**: Extract the title and diff — from the merged PR (automatic trigger) or from the commit identified by `commit_hash` (manual trigger)
2. **Generate a change description**: Produce a detailed, language-agnostic bullet-point description of what changed (semantics and intent, not Go-specific syntax)
3. **Load dependent repositories**: Read `.github/dependent-repos.json` from the current repository to get the list of dependent repos
4. **Create a PR in each dependent repo**: For each repository in the list:
   - Use the same PR title as the original PR
   - Use the detailed change description as the PR body

- Use the repository identifier to derive the local checkout path as `$GITHUB_WORKSPACE/<owner>/<repo>` (for example, `CubikRuubik/akavesdk-rust-prism` maps to `$GITHUB_WORKSPACE/CubikRuubik/akavesdk-rust-prism`)
- List existing `.md` files in `change_plans/` inside that repository checkout directory; create `change_plans/change_plan_<N+1>.md` inside the same directory where N is the count found (use 1 if the directory is empty or missing)
- Use the same branch name as the original PR branch as the first choice; fallback to `<original-branch>-sync-<pr-number>` if already taken
- Target the `main` branch
- Use the `create-pull-request` **safe output** with `repo` set to the dependent repo and `path` set to that repo's checkout path

## Guidelines

- **Change description format**: Structured bullet-point list with clear categories (e.g., "Features", "Fixes", "Breaking Changes", "Dependencies", "Documentation"). Describe _what_ changed and _why_, in language-agnostic terms — no Go syntax, just semantics and intent.
- **Change plan file**: Each dependent repo is checked out at `$GITHUB_WORKSPACE/<owner>/<repo>`. For each target repo, list existing `.md` files in `$GITHUB_WORKSPACE/<owner>/<repo>/change_plans/`, then write the full change description to `$GITHUB_WORKSPACE/<owner>/<repo>/change_plans/change_plan_<N+1>.md` where N is the count of files found (starting at 1 if the directory is empty or missing). **Do not write this file anywhere in the main `go-prism` workspace.** This file is the only change committed to that repo branch.
- **PR titles**: Use the exact title from the merged PR
- **Branch naming**: Use the original PR branch name first. When a conflict exists, use `<original-branch>-sync-<pr-number>`
- **Branch creation**: Always pass the chosen branch name in `create-pull-request`; if it does not exist yet in the target repository, the PR flow should create it from local changes.
- **Target repos**: Read the full `owner/repo` list from `.github/dependent-repos.json` — never hardcode repository names
- **Cross-repo settings**: Always include `repo: "owner/repo"` in the `create-pull-request` safe output to specify the destination repository

## Safe Outputs

When creating each pull request, use the `create-pull-request` **safe output**.

- Call `create-pull-request` with:
  - `title`: The exact title from the merged PR
  - `body`: The detailed language-agnostic change description
  - `branch`: The selected branch name (original first, fallback when needed)
  - `repo`: Each dependent repository in turn (from `dependent-repos.json`)
  - `path`: The path to the checked-out dependent repo workspace derived from the repo name (for example, `CubikRuubik/akavesdk-rust-prism` or `CubikRuubik/akavesdk-py-prism`)

If no dependent repos are configured or errors occur, use `noop` to signal completion.
