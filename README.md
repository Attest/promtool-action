# promtool-action

![GitHub Actions Logo](./img/github_actions_logo.png) ![Prometheus Logo](./img/prometheus_logo.png)

A GitHub Action that validates Prometheus configuration and rule files using `promtool` without starting a Prometheus server.

## Features

- Validates Prometheus config files (`promtool check config`)
- Validates Prometheus rule files (`promtool check rules`)
- Supports any Prometheus version (default: v2.16.0)
- Optional PR comment integration with validation results
- Glob pattern support for checking multiple files
- Detailed output capture for downstream workflow steps

## Success Criteria

An exit code of `0` indicates successful validation. Exit code `1` indicates validation failure or errors.

## Usage

### Basic Example

Validate Prometheus rule files on pull requests:

```yaml
name: Validate Prometheus Rules
on:
  pull_request:
    branches:
      - main
    paths:
      - 'prometheus/rules/*.yml'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Validate Rules
        uses: karancode/promtool-action@v0.0.1
        with:
          prom_version: '2.16.0'
          prom_check_subcommand: 'rules'
          prom_check_files: './prometheus/rules/*.yml'
          prom_comment: true
        env:
          GITHUB_ACCESS_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Validate Config Files

```yaml
- name: Validate Prometheus Config
  uses: karancode/promtool-action@v0.0.1
  with:
    prom_check_subcommand: 'config'
    prom_check_files: './prometheus.yml'
```

### Check Multiple Files

Use glob patterns or space-separated paths:

```yaml
- name: Validate Multiple Rule Files
  uses: karancode/promtool-action@v0.0.1
  with:
    prom_check_subcommand: 'rules'
    prom_check_files: './alerts/*.yml ./recording/*.yml'
```

### Without PR Comments

```yaml
- name: Validate Rules (No Comments)
  uses: karancode/promtool-action@v0.0.1
  with:
    prom_check_subcommand: 'rules'
    prom_check_files: './rules/*.yml'
    prom_comment: false
```

### Using Different Prometheus Versions

```yaml
- name: Validate with Prometheus 2.9.2
  uses: karancode/promtool-action@v0.0.1
  with:
    prom_version: '2.9.2'
    prom_check_subcommand: 'rules'
    prom_check_files: './rules/*.yml'
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `prom_version` | No | `2.16.0` | Prometheus version to use for validation (e.g., `2.9.2`, `2.16.0`) |
| `prom_check_subcommand` | Yes | - | Validation type: `config` or `rules` |
| `prom_check_files` | Yes | - | File path(s) or glob pattern to validate |
| `prom_comment` | No | `false` | Post validation results as PR comment (`true` or `false`) |

### Input Details

#### `prom_version`
Specify any valid Prometheus version available on [Prometheus releases](https://github.com/prometheus/prometheus/releases). The action downloads the specified version dynamically.

#### `prom_check_subcommand`
- `config`: Validates Prometheus configuration files (typically `prometheus.yml`)
- `rules`: Validates Prometheus alerting and recording rule files

#### `prom_check_files`
Supports:
- Single file: `./prometheus.yml`
- Glob pattern: `./rules/*.yml`
- Multiple files: `./alerts.yml ./recording.yml`
- Nested paths: `./prometheus/rules/**/*.yml`

#### `prom_comment`
When enabled (`true` or `1`):
- Posts validation results as a collapsible comment on pull requests
- Requires `GITHUB_ACCESS_TOKEN` with `pull_requests: write` permission
- Only posts on `pull_request` events
- Comment includes workflow name, action name, and file list


## Outputs

| Output | Description |
|--------|-------------|
| `promtool_output` | Complete output from `promtool check` command |

### Using Outputs

```yaml
- name: Validate Rules
  id: validate
  uses: karancode/promtool-action@v0.0.1
  with:
    prom_check_subcommand: 'rules'
    prom_check_files: './rules/*.yml'

- name: Process Results
  run: |
    echo "Validation output:"
    echo "${{ steps.validate.outputs.promtool_output }}"
```

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `GITHUB_ACCESS_TOKEN` | Conditional | GitHub API token for posting PR comments. Required only when `prom_comment` is `true`. Use `${{ secrets.GITHUB_TOKEN }}` (automatically provided by GitHub Actions). |

### Token Permissions

The `GITHUB_TOKEN` requires:
- `pull-requests: write` (for posting comments)
- `contents: read` (for checking out code)

Configure in workflow:
```yaml
permissions:
  contents: read
  pull-requests: write
```

## Examples

### Complete Workflow

```yaml
name: Prometheus Validation
on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  validate-config:
    name: Validate Config
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Check Prometheus Config
        uses: karancode/promtool-action@v0.0.1
        with:
          prom_check_subcommand: 'config'
          prom_check_files: './prometheus.yml'
          prom_comment: ${{ github.event_name == 'pull_request' }}
        env:
          GITHUB_ACCESS_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  validate-rules:
    name: Validate Rules
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Check Alert Rules
        uses: karancode/promtool-action@v0.0.1
        with:
          prom_check_subcommand: 'rules'
          prom_check_files: './rules/alerts/*.yml'
          prom_comment: ${{ github.event_name == 'pull_request' }}
        env:
          GITHUB_ACCESS_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Check Recording Rules
        uses: karancode/promtool-action@v0.0.1
        with:
          prom_check_subcommand: 'rules'
          prom_check_files: './rules/recording/*.yml'
          prom_comment: ${{ github.event_name == 'pull_request' }}
        env:
          GITHUB_ACCESS_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Matrix Strategy

Test against multiple Prometheus versions:

```yaml
jobs:
  validate:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        prom_version: ['2.9.2', '2.16.0', '2.30.0']
    steps:
      - uses: actions/checkout@v3

      - name: Validate with Prometheus ${{ matrix.prom_version }}
        uses: karancode/promtool-action@v0.0.1
        with:
          prom_version: ${{ matrix.prom_version }}
          prom_check_subcommand: 'rules'
          prom_check_files: './rules/*.yml'
```

## How It Works

1. **Container Setup**: Action runs in Alpine Linux Docker container
2. **Download**: Downloads specified Prometheus version from GitHub releases
3. **Extract**: Extracts `promtool` binary from tarball
4. **Validate**: Runs `promtool check <subcommand> <files>`
5. **Report**: Captures output and optionally posts to PR
6. **Exit**: Returns promtool's exit code (0=success, 1=failure)

## Troubleshooting

### Validation Fails

**Check the logs** for promtool output:
- Invalid PromQL syntax
- Missing required fields
- YAML parsing errors

**Test locally:**
```bash
# Install promtool
wget https://github.com/prometheus/prometheus/releases/download/v2.16.0/prometheus-2.16.0.linux-amd64.tar.gz
tar -xzf prometheus-2.16.0.linux-amd64.tar.gz
cd prometheus-2.16.0.linux-amd64

# Validate
./promtool check rules your-rules.yml
```

### No PR Comment Posted

Ensure:
- ✅ `prom_comment: true` is set
- ✅ Event is `pull_request` (not `push`)
- ✅ `GITHUB_ACCESS_TOKEN` is provided
- ✅ Token has `pull-requests: write` permission

### Download Fails

Check:
- Prometheus version exists: https://github.com/prometheus/prometheus/releases
- Version format is correct (e.g., `2.16.0`, not `v2.16.0`)
- Network connectivity to GitHub

### Permission Denied

Add workflow permissions:
```yaml
permissions:
  contents: read
  pull-requests: write
```

## Local Development

### Build Docker Image
```bash
docker build -t promtool-action:test .
```

### Test Action
```bash
docker run --rm \
  -v $(pwd):/github/workspace \
  -w /github/workspace \
  -e INPUT_PROM_VERSION="2.16.0" \
  -e INPUT_PROM_CHECK_SUBCOMMAND="rules" \
  -e INPUT_PROM_CHECK_FILES="./test-rules.yml" \
  -e INPUT_PROM_COMMENT="0" \
  promtool-action:test
```

## Contributing

Contributions welcome! This repository is maintained by [@Attest/audience-accounts](https://github.com/orgs/Attest/teams/audience-accounts).

### Development Guidelines
- See `AGENTS.md` for project structure and conventions
- Review `.agents/rules/` for code style, testing, and security guidelines
- Check `docs/ARCHITECTURE.md` for technical details

## License

MIT License - see [LICENSE](LICENSE) file.

Copyright (c) 2020 Karan Thanvi

## Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Promtool Reference](https://prometheus.io/docs/prometheus/latest/command-line/promtool/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Creating Docker Container Actions](https://docs.github.com/en/actions/creating-actions/creating-a-docker-container-action)