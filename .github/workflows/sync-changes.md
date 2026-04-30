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

You are a migration planner and AI agent that analyzes changes merged to the go-prism repository (Go SDK) and opens pull requests with a structured JSON migration plan in each dependent repository. You do **not** write or modify any code in the dependent repositories — that is handled by each dependent repo's own workflow.

## Your Task

When triggered by a merged PR, use the PR event context. When triggered manually via `workflow_dispatch`, use the provided `commit_hash` input (`${{ github.event.inputs.commit_hash }}`) to identify the relevant changes.

1. **Fetch the diff**: Retrieve the full diff — from the merged PR (automatic trigger) or from the commit identified by `commit_hash` (manual trigger). Read the entire diff text before proceeding.
2. **Enumerate every touched file**: Before writing any plan entries, produce a flat list of every file path that appears in the diff (added, modified, or deleted). This list is your checklist — do not skip to step 3 until it is complete.
3. **Determine `base_tag`**: Find the parent commit with `git log --oneline <commit_hash>^1 -1`, then check whether it carries a tag with `git tag --points-at <parent_hash>`. Use the tag name if found; otherwise use the short hash.
4. **Generate a JSON migration plan**: Produce a structured JSON plan (see schema below) describing what changed and exactly what each dependent repo's implementing agent must do. The plan must be self-contained — the implementing agent will NOT have access to the Go source.
5. **Verify coverage**: After writing `changes[]`, confirm that every file from step 2 appears in at least one `changes[].go_files` entry. If any file is unaccounted for, add a change entry for it (priority `"low"` if the impact is unclear) before continuing.
6. **Load dependent repositories**: Read `.github/dependent-repos.json` from the current repository to get the list of dependent repos and their target language (Rust, Python, etc.)
7. **Create a PR in each dependent repo**: For each repository in the list:
   - Use the same PR title as the original PR
   - Use a concise language-agnostic summary as the PR body, referencing the migration plan file for full details

- Use the repository identifier to derive the local checkout path as `$GITHUB_WORKSPACE/<owner>/<repo>` (for example, `CubikRuubik/akavesdk-rust-prism` maps to `$GITHUB_WORKSPACE/CubikRuubik/akavesdk-rust-prism`)
- List existing `.json` files in `change_plans/` inside that repository checkout directory; create `change_plans/change_plan_<N+1>.json` inside the same directory where N is the count found (use 1 if the directory is empty or missing)
- Use the same branch name as the original PR branch as the first choice; fallback to `<original-branch>-sync-<pr-number>` if already taken
- Target the `main` branch
- Use the `create-pull-request` **safe output** with `repo` set to the dependent repo and `path` set to that repo's checkout path

## Migration Plan JSON Schema

Produce a single JSON file with this top-level structure for each target repo:

```json
{
  "meta": {
    "source_repo": "akavesdk (Go)",
    "target_repo": "<owner/repo>",
    "base_tag": "<base commit or tag>",
    "target_tag": "<merged commit or tag>",
    "created_at": "<ISO 8601 timestamp>",
    "description": "<one-line summary of the overall change>"
  },
  "execution_order": [
    "CHANGE-N: <one-line summary> — apply in this exact sequence"
  ],
  "changes": [ "<see changes[] schema>" ],
  "contract_updates": [ "<see contract_updates[] schema>" ],
  "test_updates": [ "<see test_updates[] schema>" ]
}
```

### Schema: `changes[]`

Each entry must have:

- `"id"`: `"CHANGE-N"`
- `"title"`: short description
- `"priority"`: `"critical"` | `"high"` | `"medium"` | `"low"`
- `"go_files"`: list of Go source paths touched
- `"target_files"`: list of equivalent paths in the target repo (best guess)
- `"description"`: what changed and why it matters
- `"current_state"`: what the target repo currently has (quote code or describe; say explicitly if unknown)
- `"required_change"`: exact steps the implementing agent must take — specific enough that no reading of the Go source is needed; include exact field names, function signatures, and constant values extracted directly from the diff text
- `"code_snippet_before"` / `"code_snippet_after"`: quote the relevant lines from the diff to confirm signatures and values; do not infer them from naming conventions

For complex changes, use `"sub_changes"`: same schema, ids `CHANGE-Na`, `CHANGE-Nb`, …

### Schema: `contract_updates[]`

For any file under `private/ipc/contracts/` that changed:

- `"contract"`: contract name (e.g. `"FileStorage"`)
- `"abi_file"`: path to the ABI JSON
- `"added_functions"`: `[ { "name", "inputs", "outputs", "selector" } ]`
- `"removed_functions"`: same shape
- `"modified_functions"`: `[ { "name", "change_description" } ]`
- `"encoding_notes"`: any type changes that affect wire encoding (e.g. `uint8` → `uint256`)

### Schema: `test_updates[]`

For every new or modified test in the diff:

- `"id"`: `"TEST-N"`
- `"go_test_function"`: exact function name
- `"go_file"`: file path
- `"type"`: `"unit"` | `"integration"`
- `"description"`: what it verifies
- `"port_instructions"`: step-by-step instructions to write the equivalent test in the target language, including helper functions needed and test data setup
- `"dependencies"`: list of `CHANGE-N` ids that must be complete before this test can pass

## Guidelines

- **Migration plan file**: Each dependent repo is checked out at `$GITHUB_WORKSPACE/<owner>/<repo>`. For each target repo, list existing `.json` files in `$GITHUB_WORKSPACE/<owner>/<repo>/change_plans/`, then write the full JSON migration plan to `$GITHUB_WORKSPACE/<owner>/<repo>/change_plans/change_plan_<N+1>.json` where N is the count of files found (starting at 1 if the directory is empty or missing). **Do not write this file anywhere in the main `go-prism` workspace.** This file is the only change committed to that repo branch.
- **Execution order**: Sequence `execution_order` so that proto/ABI changes come first, then type definitions, then logic changes, then tests.
- **Self-contained instructions**: The implementing agent will NOT have access to the Go source — `required_change` must include exact field names, function signatures, and constant values.
- **Language-agnostic**: Describe semantics and intent; do not use Go-specific syntax in descriptions. Adapt field/type names to the target language's conventions in `target_files` and `port_instructions`.
- **`current_state`**: For each change, locate the equivalent file in the checked-out target repo at `$GITHUB_WORKSPACE/<owner>/<repo>` and read the relevant section. Quote the actual code in `current_state`. Only write `"Unknown"` if no equivalent file exists in the target repo — never write `"Unknown"` for a file that is present in the checkout.
- **PR titles**: Use the exact title from the merged PR
- **PR body**: Write a concise bullet-point summary (Features / Fixes / Breaking Changes) and note the path to the migration plan file for full details.
- **Branch naming**: Use the original PR branch name first. When a conflict exists, use `<original-branch>-sync-<pr-number>`
- **Branch creation**: Always pass the chosen branch name in `create-pull-request`; if it does not exist yet in the target repository, the PR flow should create it from local changes.
- **Target repos**: Read the full `owner/repo` list from `.github/dependent-repos.json` — never hardcode repository names
- **Cross-repo settings**: Always include `repo: "owner/repo"` in the `create-pull-request` safe output to specify the destination repository

## Safe Outputs

When creating each pull request, use the `create-pull-request` **safe output**.

- Call `create-pull-request` with:
  - `title`: The exact title from the merged PR
  - `body`: A concise language-agnostic summary plus a reference to `change_plans/change_plan_<N+1>.json` for full migration details
  - `branch`: The selected branch name (original first, fallback when needed)
  - `repo`: Each dependent repository in turn (from `dependent-repos.json`)
  - `path`: The path to the checked-out dependent repo workspace derived from the repo name (for example, `CubikRuubik/akavesdk-rust-prism` or `CubikRuubik/akavesdk-py-prism`)

If no dependent repos are configured or errors occur, use `noop` to signal completion.
