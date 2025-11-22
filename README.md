# PSR Deploy Keys for Protected Branches

This repository serves as a proof-of-concept and reference implementation for configuring **Python Semantic Release (PSR)** to work with **GitHub Protected Branches** using **Deploy Keys**.

It uses **Poetry** for dependency management and project configuration.

## The Problem

When using Python Semantic Release in a GitHub Actions workflow on a protected branch (e.g., `main`), the default `GITHUB_TOKEN` often lacks the permissions to push commits (version bumps) and tags directly to the branch if "Restrict pushes to matching branches" is enabled.

While using a **Personal Access Token (PAT)** is a common workaround, it is often considered **not developer friendly** and problematic for automation due to:

* **User Dependency:** PATs are tied to individual user accounts. If the user leaves or their account is disabled, automation breaks.
* **Security Exposure:** PATs often grant access to *all* repositories the user has access to, creating a larger blast radius if compromised.
* **Rotation Complexity:** Managing rotation for PATs across multiple repositories is tedious and error-prone.
* **Accountability:** It can be difficult to distinguish between actions taken by the user and the automation.

## The Solution

The solution involves a hybrid approach:

1. **Git Operations (Pushing Commits/Tags):** Use an **SSH Deploy Key** with write access. This bypasses the standard token restrictions for git pushes.
    * **Note:** We use `webfactory/ssh-agent` to **forward the SSH key** to the Docker container/runner environment, ensuring the `git push` commands executed by PSR can authenticate successfully.
2. **API Operations (Creating Releases):** Use the standard **GITHUB_TOKEN** to authenticate with the GitHub API for creating the Release entry and uploading artifacts.

## Configuration

### 1. GitHub Repository Settings

1. **Generate an SSH Key Pair**: `ssh-keygen -t ed25519 -C "git@github.com:owner/repo.git" -N ""`
    * **Important:** A subtle difference from the original `sbellone` example is that for `webfactory/ssh-agent` to work properly here, you should use your **SSH repository URL** as the comment (`-C`).
2. **Deploy Keys**: Add the **Public Key** to your repository's **Settings > Deploy keys**. Enable **"Allow write access"**.
3. **Secrets**: Add the **Private Key** to your repository's **Settings > Secrets and variables > Actions** as `DEPLOY_KEY`.

### 2. `pyproject.toml`

Configure PSR to ignore the token for push operations (forcing it to use the system's SSH config) and explicitly map the API token.

```toml
[tool.semantic_release.remote]
type = "github"
token = { env = "GITHUB_TOKEN" }
ignore_token_for_push = true
```

### 3. GitHub Actions Workflow

Your workflow must:

1. Checkout the repository using the `DEPLOY_KEY` (sets up the git remote with SSH).
2. Start the SSH agent with the `DEPLOY_KEY`.
3. Pass the `GITHUB_TOKEN` to the semantic-release command.

```yaml
steps:
  - name: Checkout Repository
    uses: actions/checkout@v4
    with:
      ssh-key: ${{ secrets.DEPLOY_KEY }}
      fetch-depth: 0

  - name: Setup SSH Agent
    uses: webfactory/ssh-agent@v0.9.0
    with:
      ssh-private-key: ${{ secrets.DEPLOY_KEY }}

  # ... setup python, poetry ...

  - name: Python Semantic Release
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: |
      poetry run semantic-release version
```

## Reference

This configuration addresses the scenario discussed in [Python Semantic Release Issue #1343](https://github.com/python-semantic-release/python-semantic-release/issues/1343#issuecomment-3473285753).

The deploy key workaround to bypass branch protection was originally discovered and documented by [sbellone/release-workflow-example](https://github.com/sbellone/release-workflow-example).
