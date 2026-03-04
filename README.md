# Semantic Release Workflow

A reusable GitHub Actions workflow that automatically handles semantic versioning and releases for any repository using emoji-based commit messages.

## What is Semantic Release?

Semantic Release is an automated versioning and release management system that eliminates the need for manual version management. It automatically determines the next version number based on **Conventional Commits** and updates your changelog, git tags, and publishes packages accordingly.

### Semantic Versioning (SemVer)

Semantic versioning follows the format: **MAJOR.MINOR.PATCH**

- **MAJOR** version: Breaking changes (incompatible API changes)
- **MINOR** version: New features (backward-compatible functionality)
- **PATCH** version: Bug fixes and patches (backward-compatible fixes)

**Example:** `1.2.3` where 1 is MAJOR, 2 is MINOR, and 3 is PATCH

## Overview

This repository contains a **reusable GitHub Actions workflow** (`.github/workflows/semantic-release.yaml`) that any repository can call to automate their release process. Instead of managing releases manually, your repository simply needs to:

1. Use emoji-prefixed commit messages
2. Call the semantic release workflow
3. Let automation handle versioning, changelogs, and releases

## Commit Message Emoji Mapping

The release workflow uses emoji-prefixed commits to determine version increments:

| Emoji | Type | Version Impact | Examples |
|-------|------|----------------|----------|
| `:boom:` | Breaking Change | **MAJOR** | Removes features, changes API |
| `:sparkles:` | New Feature | **MINOR** | Adds functionality |
| `:bug:` | Bug Fix | **PATCH** | Fixes issues |
| `:ambulance:` | Critical Fix | **PATCH** | Critical hotfixes |
| `:lock:` | Security | **PATCH** | Security improvements |
| `:rocket:` | Deployment | **PATCH** | Deployment-related changes |
| `:arrow_up:` | Dependencies | **PATCH** | Dependency updates |

## How It Works

### The Release Pipeline

1. **Detect** - Commits are analyzed by semantic-release-gitmoji
2. **Calculate** - Next version is determined based on commit emojis
3. **Generate** - Changelog is automatically created from commit messages
4. **Update** - Package version is bumped in configuration files
5. **Commit & Tag** - Changes committed with git tags (e.g., `V1.2.3`)
6. **Publish** - Release published to GitHub and/or npm registry
7. **Merge** - (Optional) Changes merged back to develop branch

## Workflow Configurations

### Reusable Workflow: `.github/workflows/semantic-release.yaml`

This is a **workflow_call** that any repository can reference and use. It provides a standardized way to handle releases across multiple projects.

```yaml
name: SemanticRelease

on:
  workflow_call:
    inputs:
      default-branch:
        default: main
        type: string
      develop-branch:
        default: develop
        type: string
      merge-back-to-develop:
        type: boolean
        default: false

jobs:
  Release:
    name: Release
    if: "!contains(github.event.head_commit.message, ':bookmark:')"
    runs-on: ubuntu-latest
    environment: release
    # ... (permissions and steps)
```

**Key Features:**
- Reusable workflow that other repositories can call
- Configurable branch names (main, develop, etc.)
- Optional merge-back to develop branch
- Skips release if commit message contains `:bookmark:` (prevents re-releasing)

### Workflow Inputs

When calling the semantic release workflow, you can customize:

| Input | Default | Type | Description |
|-------|---------|------|-------------|
| `default-branch` | `main` | string | The branch where releases are published from |
| `develop-branch` | `develop` | string | The develop branch (for merge-back option) |
| `merge-back-to-develop` | `false` | boolean | Whether to merge release changes back to develop |

## How to Use This Workflow in Your Repository

### 1. Create a Calling Workflow

In your own repository, create `.github/workflows/release.yaml`:

```yaml
name: Release

on:
  push:
    branches:
      - main

jobs:
  release:
    uses: dgs-genzt/semantic-release/.github/workflows/semantic-release.yaml@main
    with:
      default-branch: main
      develop-branch: develop
      merge-back-to-develop: false
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 2. Create a `.releaserc.yaml` Configuration

In your repository root, create a `.releaserc.yaml` file to configure what gets released:

**For NPM/JavaScript Projects:**
```yaml
branches:
  - "main"

tagFormat: "V${version}"

plugins:
  - - "semantic-release-gitmoji"
    - releaseRules:
        major: [":boom:"]
        minor: [":sparkles:"]
        patch: [":bug:", ":ambulance:", ":lock:", ":rocket:", ":arrow_up:"]
  - "@semantic-release/changelog"
  - - "@semantic-release/npm"
    - { "npmPublish": false }
  - - "@semantic-release/git"
    - assets: ["CHANGELOG.md", "package.json", "package-lock.json"]
      message: ":bookmark: release ${nextRelease.version}\n\n${nextRelease.notes}"
  - "@semantic-release/github"
```

**For Maven/Java Projects:**
```yaml
branches:
  - "main"

tagFormat: "v${version}"

plugins:
  - - "semantic-release-gitmoji"
    - releaseRules:
        major: [":boom:"]
        minor: [":sparkles:"]
        patch: [":bug:", ":ambulance:", ":lock:", ":rocket:", ":arrow_up:"]
  - "@semantic-release/changelog"
  - - "@semantic-release/npm"
    - { "npmPublish": false }
  - - "@semantic-release/exec"
    - verifyReleaseCmd: 'mvn versions:set -DnewVersion="${nextRelease.version}"'
  - - "@semantic-release/git"
    - assets: ["pom.xml", "CHANGELOG.md"]
      message: ":bookmark: release ${nextRelease.version}\n\n${nextRelease.notes}"
  - "@semantic-release/github"
```

### 3. Set Up GitHub Secrets

Configure the required secret in your repository:

- **`GH_TOKEN`** - Personal access token with permissions:
  - `contents: write` - Publish releases
  - `issues: write` - Comment on issues
  - `pull-requests: write` - Comment on PRs
  - `id-token: write` - OIDC for npm provenance

### 4. Make Commits with Emoji Prefixes

Use emoji in your commit messages:

```bash
git commit -m ":sparkles: add authentication system"
git commit -m ":bug: fix token validation"
git commit -m ":boom: change API endpoint structure"
```

## Workflow Example

**Initial State:** Repository version `1.0.0`

**Commits made:**
```bash
git commit -m ":sparkles: add authentication system"  # minor feature
git commit -m ":bug: fix token validation"            # bug fix
git commit -m ":boom: change API endpoint structure"  # breaking change
git push origin main
```

**Workflow Runs:**
- Semantic-release detects commits
- Breaking change detected (`:boom:`)
- Version calculated: `2.1.0` (breaking change takes priority)
- `CHANGELOG.md` generated with all changes
- Git tag created: `V2.1.0`
- GitHub release published automatically
- Package.json updated (if applicable)

**Result:**
- ✅ New version: `2.1.0`
- ✅ Changelog created
- ✅ GitHub release published
- ✅ Git tag created
- ✅ (Optional) Merge back to develop

## Customization

### Optional Steps in Workflow

The workflow includes commented-out steps you can uncomment:

**Setup Custom Node Version:**
```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version-file: ".node-version"
```

**Use Custom NPM Registry:**
```yaml
- name: Use .npmrc
  uses: bduff9/use-npmrc@v1.2
  with:
    dot-npmrc: ${{ secrets.GH_TOKEN }}
```

**Install Project Dependencies:**
```yaml
- name: Install dependencies
  run: npm clean-install
```

### Permissions Required

The workflow needs these GitHub permissions:

```yaml
permissions:
  contents: write        # Publish GitHub releases
  issues: write          # Comment on released issues
  pull-requests: write   # Comment on released pull requests
  id-token: write        # Enable OIDC for npm provenance
```

## Files in This Repository

- **`.github/workflows/semantic-release.yaml`** - The reusable workflow that handles releases
- **`.releaserc.yaml`** - Example configuration for NPM projects
- **`maven.releaserc.yaml`** - Example configuration for Maven projects
- **`package.json`** - Dependencies for semantic-release plugins

## What Gets Updated During Release

### NPM Projects
- `package.json` - Version number
- `package-lock.json` - Lock file
- `CHANGELOG.md` - Release notes
- Git tags - Release tags

### Maven Projects
- `pom.xml` - Version number
- `CHANGELOG.md` - Release notes
- Git tags - Release tags

## Benefits

✅ **Automated Versioning** - No manual version management  
✅ **Standardized Process** - Same workflow across all repos  
✅ **Reusable Workflow** - DRY principle for CI/CD  
✅ **Emoji-Based** - Simple, visual commit conventions  
✅ **Automatic Changelog** - Release notes generated automatically  
✅ **Git Integration** - Automatic tagging and branch management  
✅ **Multi-Platform** - Works with npm, Maven, GitHub, and more  
✅ **Skip Logic** - Prevents infinite release loops  
✅ **Flexible Configuration** - Customize per repository  

## Troubleshooting

### Workflow Not Triggering
- Ensure commits have emoji prefixes (`:emoji:`)
- Check that `GH_TOKEN` secret is configured
- Verify branch name matches `default-branch` input

### Release Skipped
- If commit message contains `:bookmark:`, release is automatically skipped
- This prevents infinite loops when semantic-release commits changes

### Version Not Updating
- Check `.releaserc.yaml` configuration is in repository root
- Verify emoji mappings match your commit messages
- Review workflow logs for errors

## Resources

- [Semantic Release Documentation](https://semantic-release.gitbook.io/)
- [semantic-release-gitmoji](https://github.com/momocow/semantic-release-gitmoji)
- [GitHub Actions Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [Semantic Versioning](https://semver.org/)

## License

See your repository's license file for details.
