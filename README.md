# TODO Tracking - Reusable GitHub Workflow

Automatically track TODOs in your code and create GitHub issues for them. When
you push commits with new TODOs, issues are created. When TODOs are removed,
issues are closed. Issue URLs are automatically inserted back into your code.

This is a reusable workflow that can be easily integrated into any repository.

## Features

- Automatically creates GitHub issues for TODOs in your code
- Closes issues when TODOs are removed
- Auto-assigns issues to the TODO author
- Inserts issue URLs back into the code comments
- Configurable per repository
- Supports multiple branches (main, dev, etc.)
- Manual initialization for existing TODOs

## Prerequisites

### 1. Create a GitHub App

You need a GitHub App to authenticate the workflow with elevated permissions.

1. Go to GitHub Settings → Developer settings → GitHub Apps → New GitHub App
2. Configure the app:
   - **Name**: `TODO Tracker` (or any name you prefer)
   - **Homepage URL**: Your organization or personal website
   - **Webhook**: Uncheck "Active"
   - **Permissions**:
     - Repository permissions:
       - Contents: Read and write
       - Issues: Read and write
       - Pull requests: Read and write
   - **Where can this GitHub App be installed?**: Only on this account
3. Create the app
4. Generate a private key (click "Generate a private key" at the bottom)
5. Note the **App ID** (shown at the top of the app page)
6. Install the app on your organization or repositories

### 2. Configure Repository Secrets

In each repository where you want to use TODO tracking:

1. Go to Settings → Secrets and variables → Actions
2. Add two secrets:
   - `TODO_APP_ID`: Your GitHub App ID
   - `TODO_APP_PRIV_KEY`: Your GitHub App private key (entire contents of the
     .pem file)

### 3. Add GitHub App to Branch Protection Bypass Rules

If your repository uses branch protection rules:

1. Go to Settings → Branches → Branch protection rules
2. Edit your branch protection rule (e.g., for `main` or `dev`)
3. Scroll to "Rules applied to everyone including administrators"
4. Under "Allow specified actors to bypass required pull requests", add your
   GitHub App
5. Check "Exempt" next to your TODO Tracker app

This allows the workflow to automatically merge the PR that adds issue links
back to your code.

## Setup

### Step 1: Add the Workflow to Your Repository

Copy `.github/workflows/todo-tracking-example.yml` to your repository as
`.github/workflows/todo-tracking.yml`:

```yaml
name: Track TODOs

on:
  push:
    branches:
      - main
      - dev
  workflow_dispatch:
    inputs:
      MANUAL_COMMIT_REF:
        description: "The SHA of the commit to get the diff for"
        required: true
      MANUAL_BASE_REF:
        description: "By default, the commit entered above is compared to the one directly before it; to go back further, enter an earlier SHA here"
        required: false

jobs:
  track:
    permissions:
      contents: write
      issues: write
      pull-requests: write
    uses: JHOFER-Cloud/todo-tracking/.github/workflows/todo-tracking-reusable.yml@main
    with:
      manual_commit_ref: ${{ inputs.MANUAL_COMMIT_REF }}
      manual_base_ref: ${{ inputs.MANUAL_BASE_REF }}
    secrets:
      TODO_APP_ID: ${{ secrets.TODO_APP_ID }}
      TODO_APP_PRIV_KEY: ${{ secrets.TODO_APP_PRIV_KEY }}
```

### Step 2: (Optional) Customize Language Support

If you need to track TODOs in file types not supported by default:

1. Copy `.github/workflows/syntax.json` from this repo to your repository
2. Customize it to add your file types
3. Update the workflow to use it:

```yaml
jobs:
  track:
    permissions:
      contents: write
      issues: write
      pull-requests: write
    uses: JHOFER-Cloud/todo-tracking/.github/workflows/todo-tracking-reusable.yml@main
    with:
      languages: ".github/workflows/syntax.json" # Add this line
      manual_commit_ref: ${{ inputs.MANUAL_COMMIT_REF }}
      manual_base_ref: ${{ inputs.MANUAL_BASE_REF }}
    secrets:
      TODO_APP_ID: ${{ secrets.TODO_APP_ID }}
      TODO_APP_PRIV_KEY: ${{ secrets.TODO_APP_PRIV_KEY }}
```

### Step 3: Initialize Existing TODOs

For repositories with existing TODOs, you need to initialize them by running the
workflow manually:

1. Go to Actions → Track TODOs → Run workflow
2. Fill in:
   - **MANUAL_COMMIT_REF**: Current commit SHA (HEAD)
   - **MANUAL_BASE_REF**: First commit in the repository (or earliest commit you
     want to track from)
3. Click "Run workflow"

To find the first commit SHA:

```bash
git log --reverse --oneline | head -1
```

This will scan all TODOs between the first commit and HEAD, creating issues for
all of them.

## Usage

### Writing TODOs

Add TODOs in your code with standard comment syntax:

```go
// TODO: Implement user authentication
// TODO(username): Add error handling here
```

```python
# TODO: Optimize this query
# TODO(username): Add input validation
```

```javascript
// TODO: Refactor this function
/* TODO: Add comprehensive tests */
```

### What Happens

1. **On push to tracked branches** (main, dev):
   - Workflow scans the diff for new or removed TODOs
   - Creates issues for new TODOs
   - Closes issues for removed TODOs
   - Inserts issue URLs back into the code (e.g.,
     `// TODO: Fix bug https://github.com/org/repo/issues/123`)
   - Creates a PR with the issue links and auto-merges it back to the branch
     that triggered the workflow

2. **The TODO comment is updated**:
   ```go
   // TODO: Implement user authentication https://github.com/org/repo/issues/42
   ```

3. **Issues are automatically managed**:
   - Created when TODO is added
   - Closed when TODO is removed
   - Assigned to the commit author

## Configuration Options

You can customize the workflow behavior by adding `with` parameters:

```yaml
jobs:
  track:
    permissions:
      contents: write
      issues: write
      pull-requests: write
    uses: JHOFER-Cloud/todo-tracking/.github/workflows/todo-tracking-reusable.yml@main
    with:
      # Close issues when TODOs are removed (default: true)
      close_issues: true

      # Auto-assign issues to TODO author (default: true)
      auto_assign: true

      # Insert issue URLs back into code (default: true)
      insert_issue_urls: true

      # Path to custom syntax.json file (default: '.github/workflows/syntax.json')
      languages: ".github/workflows/custom-syntax.json"

      # For workflow_dispatch initialization
      manual_commit_ref: ${{ inputs.MANUAL_COMMIT_REF }}
      manual_base_ref: ${{ inputs.MANUAL_BASE_REF }}
    secrets:
      TODO_APP_ID: ${{ secrets.TODO_APP_ID }}
      TODO_APP_PRIV_KEY: ${{ secrets.TODO_APP_PRIV_KEY }}
```

## Supported Languages (Default)

The default configuration supports these file types:

- Shell scripts (`.sh`, `.fish`)
- Configuration files (`.conf`, `.env`, `.gitignore`)
- Build files (`justfile`)
- Nix (`.nix`)
- Docker (`Dockerfile`)
- YAML (`.yaml`, `.yml`)
- TOML (`.toml`)

Standard languages (JavaScript, Python, Go, etc.) are supported by default
through the
[TODO to Issue Action](https://github.com/alstr/todo-to-issue-action).

## Troubleshooting

### "Workflow is not accessible" error

Make sure:

- This repository (`JHOFER-Cloud/todo-tracking`) is public, OR
- Your calling repository is in the same organization

### Issues are not being created

Check:

1. GitHub App is installed on the repository
2. Secrets `TODO_APP_ID` and `TODO_APP_PRIV_KEY` are set correctly
3. GitHub App has correct permissions (Contents, Issues, Pull requests)

### PR fails to merge automatically

Check:

1. GitHub App is added to branch protection bypass rules
2. App is marked as "Exempt"
3. Branch protection settings allow force pushes from the app

### TODOs in certain file types are not detected

Add your file type to `.github/workflows/syntax.json` and configure the
`languages` input in your workflow.

## Example Repository Setup

See the `.github/workflows/todo-tracking-example.yml` file for a complete
example.

## Credits

This workflow uses:

- [TODO to Issue Action](https://github.com/alstr/todo-to-issue-action) by
  @alstr
- [create-github-app-token](https://github.com/actions/create-github-app-token)
  by GitHub

## License

MIT
