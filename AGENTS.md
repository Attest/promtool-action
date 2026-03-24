# AGENTS.md - AI Agent Instructions

**Single source of truth for AI agents working with the promtool-action repository.**

## Project Summary

`promtool-action` is a GitHub Action that validates Prometheus configuration and rule files using `promtool` without requiring a running Prometheus server. The action downloads a specified version of Prometheus, extracts the `promtool` binary, runs validation checks, and optionally posts results as comments on pull requests.

**Key Features:**
- Validates Prometheus config and rule files via `promtool check`
- Supports multiple Prometheus versions (default: 2.16.0)
- Optional PR comment integration with validation results
- Runs in Docker container (Alpine-based)
- Exit code 0 indicates success

**Primary Use Case:** CI/CD validation of Prometheus configurations in GitHub Actions workflows.

## Tech Stack

| Technology | Purpose | Version/Details |
|------------|---------|-----------------|
| **Bash** | Scripting | Shell scripts for orchestration and validation |
| **Docker** | Container runtime | Alpine 3 base image |
| **GitHub Actions** | CI/CD platform | Action defined in `action.yml` |
| **Prometheus** | Validation tool | `promtool` binary (default v2.16.0, configurable) |
| **Alpine Linux** | Base OS | Version 3 (latest) |
| **jq** | JSON processing | For formatting PR comments |
| **curl** | HTTP client | For posting PR comments via GitHub API |

## Repository Layout

```
promtool-action/
├── action.yml              # GitHub Action definition with inputs/outputs
├── Dockerfile              # Alpine-based container image
├── src/
│   ├── entrypoint.sh      # Main entry point: parses inputs, installs promtool
│   └── promtool_check.sh  # Executes promtool check and handles PR comments
├── img/                    # Logo images for README
├── .github/
│   └── CODEOWNERS         # Code ownership (@Attest/audience-accounts)
├── .gitignore             # Visual Studio-focused ignore patterns
├── LICENSE                # MIT License
└── README.md              # Usage documentation
```

## Key Entry Points

### 1. `action.yml`
Defines the GitHub Action interface:
- **Inputs**: `prom_version`, `prom_check_subcommand`, `prom_check_files`, `prom_comment`
- **Outputs**: `promtool_output`
- **Runtime**: Docker container using local Dockerfile

### 2. `Dockerfile`
- Base: Alpine 3
- Installs: bash, ca-certificates, curl, wget, git, jq, openssh
- Entry point: `/src/entrypoint.sh`

### 3. `src/entrypoint.sh`
Main orchestration script:
1. Parses GitHub Action inputs (environment variables prefixed with `INPUT_`)
2. Downloads and extracts Prometheus tarball from GitHub releases
3. Moves `promtool` binary to `/usr/bin/`
4. Sources and executes `promtool_check.sh`

### 4. `src/promtool_check.sh`
Validation and reporting script:
1. Executes `promtool check <config|rules> <files>`
2. Captures output and exit code
3. Optionally posts PR comment with results (if `prom_comment=1` and event is `pull_request`)
4. Writes output to `$GITHUB_OUTPUT` using multi-line format
5. Exits with promtool's exit code

## Development Workflow

### Prerequisites
- Docker (for local testing)
- GitHub repository with Prometheus config/rule files

### Testing Locally
```bash
# Build Docker image
docker build -t promtool-action .

# Run with environment variables
docker run --rm \
  -e INPUT_PROM_VERSION="2.16.0" \
  -e INPUT_PROM_CHECK_SUBCOMMAND="rules" \
  -e INPUT_PROM_CHECK_FILES="./test-rules.yml" \
  -e INPUT_PROM_COMMENT="0" \
  promtool-action
```

### Making Changes
1. Modify scripts in `src/` (ensure `+x` permissions)
2. Update `action.yml` if inputs/outputs change
3. Update Dockerfile if dependencies change
4. Test with Docker build
5. Create PR (owned by @Attest/audience-accounts)

### Release Process
- Tag versions following semver (e.g., `v0.0.1`)
- Users reference action by tag: `uses: karancode/promtool-action@v0.0.1`

## Agent Rules

When working with this codebase, AI agents should adhere to rules defined in:

- **Code Style**: `.agents/rules/code-style.md` - Bash scripting conventions
- **Testing**: `.agents/rules/testing.md` - Testing practices for shell scripts
- **Security**: `.agents/rules/security.md` - Security considerations for CI/CD actions

## Updating This File

**When to update:**
- Adding/removing major components
- Changing tech stack or dependencies
- Modifying development workflow
- Adding new entry points or key files

**How to update:**
1. Make changes to reflect current state of repository
2. Ensure consistency with `README.md` (user-facing) and `docs/ARCHITECTURE.md` (technical details)
3. Update "Last Updated" timestamp below
4. Commit with descriptive message

**Last Updated:** 2026-03-24
