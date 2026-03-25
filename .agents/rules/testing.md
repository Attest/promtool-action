# Testing Guide

## Testing Strategy

The `promtool-action` repository primarily consists of shell scripts executed in a Docker container. Testing should focus on script logic, error handling, and integration with GitHub Actions.

## Local Testing

### Prerequisites
- Docker installed and running
- Sample Prometheus config/rule files for testing

### Building the Action
```bash
cd /path/to/promtool-action
docker build -t promtool-action:test .
```

### Testing with Different Scenarios

#### Test 1: Valid Rules File
```bash
# Create test file
cat > test-rules.yml << 'EOF'
groups:
  - name: example
    rules:
      - alert: HighErrorRate
        expr: job:request_errors:rate5m > 0.5
        annotations:
          summary: High error rate
EOF

# Run action
docker run --rm \
  -v $(pwd):/github/workspace \
  -w /github/workspace \
  -e INPUT_PROM_VERSION="2.16.0" \
  -e INPUT_PROM_CHECK_SUBCOMMAND="rules" \
  -e INPUT_PROM_CHECK_FILES="test-rules.yml" \
  -e INPUT_PROM_COMMENT="0" \
  promtool-action:test

# Expected: Exit code 0, success message
```

#### Test 2: Invalid Rules File
```bash
# Create invalid test file
cat > test-rules-bad.yml << 'EOF'
groups:
  - name: example
    rules:
      - alert: InvalidRule
        expr: "this is not valid PromQL"
EOF

# Run action
docker run --rm \
  -v $(pwd):/github/workspace \
  -w /github/workspace \
  -e INPUT_PROM_VERSION="2.16.0" \
  -e INPUT_PROM_CHECK_SUBCOMMAND="rules" \
  -e INPUT_PROM_CHECK_FILES="test-rules-bad.yml" \
  -e INPUT_PROM_COMMENT="0" \
  promtool-action:test

# Expected: Exit code 1, error message
```

#### Test 3: Valid Config File
```bash
# Create test config
cat > test-config.yml << 'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
EOF

# Run action
docker run --rm \
  -v $(pwd):/github/workspace \
  -w /github/workspace \
  -e INPUT_PROM_VERSION="2.16.0" \
  -e INPUT_PROM_CHECK_SUBCOMMAND="config" \
  -e INPUT_PROM_CHECK_FILES="test-config.yml" \
  -e INPUT_PROM_COMMENT="0" \
  promtool-action:test

# Expected: Exit code 0, success message
```

#### Test 4: Different Prometheus Versions
```bash
# Test with older version
docker run --rm \
  -v $(pwd):/github/workspace \
  -w /github/workspace \
  -e INPUT_PROM_VERSION="2.9.2" \
  -e INPUT_PROM_CHECK_SUBCOMMAND="rules" \
  -e INPUT_PROM_CHECK_FILES="test-rules.yml" \
  -e INPUT_PROM_COMMENT="0" \
  promtool-action:test

# Expected: Downloads and uses v2.9.2
```

#### Test 5: Multiple Files (Glob Pattern)
```bash
# Create multiple files
mkdir -p prometheus/rules
cat > prometheus/rules/alerts.yml << 'EOF'
groups:
  - name: alerts
    rules:
      - alert: ExampleAlert
        expr: up == 0
EOF

cat > prometheus/rules/recording.yml << 'EOF'
groups:
  - name: recording
    rules:
      - record: job:up:sum
        expr: sum(up) by (job)
EOF

# Run action with glob pattern
docker run --rm \
  -v $(pwd):/github/workspace \
  -w /github/workspace \
  -e INPUT_PROM_VERSION="2.16.0" \
  -e INPUT_PROM_CHECK_SUBCOMMAND="rules" \
  -e INPUT_PROM_CHECK_FILES="prometheus/rules/*.yml" \
  -e INPUT_PROM_COMMENT="0" \
  promtool-action:test

# Expected: Checks all matching files
```

## GitHub Actions Integration Testing

### Test Workflow
Create `.github/workflows/test-promtool.yml` in a test repository:

```yaml
name: Test Promtool Action
on:
  pull_request:
    branches: [main]
    paths:
      - 'prometheus/**/*.yml'
  push:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Validate Rules
        uses: ./  # Or your-org/promtool-action@version
        with:
          prom_version: '2.16.0'
          prom_check_subcommand: 'rules'
          prom_check_files: 'prometheus/rules/*.yml'
          prom_comment: true
        env:
          GITHUB_ACCESS_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Validation Checklist
- [ ] Action runs successfully on valid files
- [ ] Action fails (exit code 1) on invalid files
- [ ] PR comments are posted when `prom_comment: true`
- [ ] No comments posted when `prom_comment: false`
- [ ] Output is captured in `promtool_output`
- [ ] Different Prometheus versions work correctly
- [ ] Glob patterns match multiple files
- [ ] Error messages are clear and actionable

## Manual Testing Checklist

### Script Logic
- [ ] `parse_inputs()` correctly reads all INPUT_* env vars
- [ ] Default values applied when inputs not provided
- [ ] `install_promtool()` downloads correct version
- [ ] Tarball extraction succeeds
- [ ] `promtool` binary moved to `/usr/bin/`
- [ ] `promtool_check()` captures output and exit code
- [ ] Success/failure detected correctly
- [ ] Output formatted for GitHub Actions correctly

### Error Scenarios
- [ ] Invalid Prometheus version (404) handled gracefully
- [ ] Missing input files produce clear error
- [ ] Network failures during download handled
- [ ] Invalid subcommand produces error
- [ ] Permission issues reported clearly

### PR Comment Integration
- [ ] Comment posted only on pull_request events
- [ ] Comment includes formatted output (HTML details tag)
- [ ] Comment includes workflow and action context
- [ ] Comment shows Success/Failed status
- [ ] `GITHUB_ACCESS_TOKEN` required when commenting enabled

## Test Data

### Minimal Valid Rules File
```yaml
groups:
  - name: test
    rules:
      - alert: Test
        expr: up == 0
```

### Minimal Valid Config File
```yaml
global:
  scrape_interval: 15s
```

### Invalid Rules (Common Errors)
```yaml
# Missing expr
groups:
  - name: test
    rules:
      - alert: NoExpr

# Invalid PromQL
groups:
  - name: test
    rules:
      - alert: BadExpr
        expr: "not valid promql!!!"

# Syntax error
groups:
  - name: test
    rules:
    - alert: Malformed
      expr: up
      labels
        severity: critical
```

## Debugging

### Enable Verbose Output
Add to scripts for debugging:
```bash
set -x  # Print commands as executed
```

### Inspect Docker Container
```bash
# Run interactively
docker run -it --entrypoint /bin/bash promtool-action:test

# Inside container
promtool --version
which promtool
ls -la /src/
```

### Check GitHub Actions Logs
- View full output in Actions tab
- Look for "check: info:" and "check: error:" messages
- Verify environment variables set correctly
- Check GITHUB_OUTPUT format

## Continuous Improvement

### When Adding Features
1. Add corresponding test scenario
2. Verify both success and failure paths
3. Test with multiple Prometheus versions
4. Validate PR comment formatting
5. Document new test cases in this file

### When Fixing Bugs
1. Create test case that reproduces bug
2. Verify fix resolves issue
3. Add test to prevent regression
4. Document edge case if non-obvious
