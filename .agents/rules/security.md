# Security Guidelines

## Overview

`promtool-action` is a GitHub Action that runs in CI/CD environments and interacts with the GitHub API. Security considerations are critical to prevent supply chain attacks, data leaks, and unauthorized access.

## Input Validation

### Environment Variables
```bash
# Always validate inputs before use
if [ "${INPUT_PROM_VERSION}" != "" ]; then
    # Validate version format (e.g., semver)
    if [[ "${INPUT_PROM_VERSION}" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
        prom_version=${INPUT_PROM_VERSION}
    else
        echo "error: Invalid version format"
        exit 1
    fi
fi
```

**Rules:**
- Validate all user-provided inputs
- Use regex patterns to ensure expected format
- Reject unexpected or malformed values
- Never execute user input directly as code

### File Paths
```bash
# Sanitize file paths to prevent path traversal
prom_check_files=${INPUT_PROM_CHECK_FILES}

# Avoid: eval, arbitrary command execution
# Never: eval "promtool check ${prom_check_files}"
```

**Rules:**
- Use glob patterns safely (let shell expand)
- Don't allow `../` path traversal in inputs
- Validate file paths exist before processing
- Never use `eval` with user input

## Secret Management

### GitHub Tokens
```bash
# GITHUB_ACCESS_TOKEN should only be used for API calls
if [ "${GITHUB_EVENT_NAME}" == "pull_request" ] && [ "${prom_comment}" == "1" ]; then
    # Token only used with curl to GitHub API
    curl -s -S -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
         --header "Content-Type: application/json" \
         --data @- "${check_comment_url}" > /dev/null
fi
```

**Rules:**
- Only access `GITHUB_ACCESS_TOKEN` when needed
- Never log tokens (check `set -x` is off)
- Never expose tokens in error messages
- Use tokens only for intended GitHub API calls
- Tokens should have minimal required permissions

### Token Scope
```yaml
# In workflows using this action
env:
  GITHUB_ACCESS_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**The `GITHUB_TOKEN` should have:**
- `pull_requests: write` (for commenting)
- No additional permissions required

## Download Security

### Binary Downloads
```bash
# Download from official Prometheus GitHub releases only
url="https://github.com/prometheus/prometheus/releases/download/v${prom_version}/prometheus-${prom_version}.linux-amd64.tar.gz"

wget ${url}
if [ "${?}" -ne 0 ]; then
    echo "Failed to download promtool v${prom_version}."
    exit 1
fi
```

**Current Issues:**
- ⚠️ No checksum verification
- ⚠️ No signature verification
- ⚠️ Uses HTTP/HTTPS without cert pinning

**Recommended Improvements:**
```bash
# Download with checksum verification
url="https://github.com/prometheus/prometheus/releases/download/v${prom_version}/prometheus-${prom_version}.linux-amd64.tar.gz"
checksum_url="https://github.com/prometheus/prometheus/releases/download/v${prom_version}/sha256sums.txt"

wget ${url}
wget ${checksum_url}

# Verify checksum
expected_checksum=$(grep "prometheus-${prom_version}.linux-amd64.tar.gz" sha256sums.txt | cut -d' ' -f1)
actual_checksum=$(sha256sum prometheus-${prom_version}.linux-amd64.tar.gz | cut -d' ' -f1)

if [ "${expected_checksum}" != "${actual_checksum}" ]; then
    echo "error: Checksum verification failed"
    exit 1
fi
```

### Supply Chain Security
**Rules:**
- Only download from `github.com/prometheus/prometheus` (official)
- Verify checksums when available
- Consider signature verification (GPG)
- Pin specific versions rather than "latest"
- Document version update process

## Container Security

### Base Image
```dockerfile
FROM alpine:3
```

**Current State:**
- Uses `alpine:3` (tracks latest Alpine 3.x)
- No specific version pinning

**Recommendations:**
```dockerfile
# Pin to specific version for reproducibility
FROM alpine:3.19

# Or use digest for immutability
FROM alpine:3@sha256:abc123...
```

### Package Installation
```dockerfile
RUN apk add --update --no-cache bash ca-certificates curl wget git jq openssh
```

**Security Considerations:**
- `--no-cache` prevents package cache in image (good)
- Consider pinning package versions
- Audit dependencies for vulnerabilities
- Minimize installed packages (remove unused tools)

### Privilege Escalation
**Rules:**
- Don't run as root if possible (consider `USER` directive)
- Minimize capabilities required
- Avoid `--privileged` flag
- Use read-only filesystems where possible

## Code Injection Prevention

### Command Execution
```bash
# SAFE: Variables properly quoted
check_output=$(promtool check ${prom_check_subcommand} ${prom_check_files} 2>&1)

# UNSAFE: Never use eval with user input
# eval "promtool check ${user_input}"  # NEVER DO THIS
```

**Rules:**
- Quote all variable expansions
- Never use `eval` with user input
- Avoid dynamic command construction
- Use arrays for complex argument lists

### JSON Construction
```bash
# Current approach - potential injection risk
check_payload=$(echo "${check_comment_wrapper}" | jq -R --slurp '{body: .}')
```

**Safer Alternative:**
```bash
# Use jq to properly escape JSON
check_payload=$(jq -n --arg body "${check_comment_wrapper}" '{body: $body}')
```

## GitHub API Security

### API Calls
```bash
# Current implementation
curl -s -S -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
     --header "Content-Type: application/json" \
     --data @- "${check_comment_url}" > /dev/null
```

**Security Checklist:**
- ✅ Uses HTTPS (GitHub API)
- ✅ Token in Authorization header (not URL)
- ✅ Only posts to PR comment URLs from GitHub event
- ⚠️ Consider validating `check_comment_url` format

**Recommendations:**
```bash
# Validate URL is a GitHub API endpoint
if [[ ! "${check_comment_url}" =~ ^https://api\.github\.com/ ]]; then
    echo "error: Invalid GitHub API URL"
    exit 1
fi
```

## Data Exposure

### Logging
```bash
# Good: Log messages without sensitive data
echo "check: info: promtool check for ${prom_check_files}."

# Bad: Don't log tokens or secrets
# echo "Using token: ${GITHUB_ACCESS_TOKEN}"  # NEVER
```

**Rules:**
- Never log secrets, tokens, or credentials
- Be cautious with `set -x` (prints all commands)
- Sanitize output before logging
- Avoid including sensitive data in error messages

### Output Capture
```bash
# Current: Output includes full promtool results
echo "promtool_output<<EOF" >> "$GITHUB_OUTPUT"
echo "${check_output}" >> "$GITHUB_OUTPUT"
echo "EOF" >> "$GITHUB_OUTPUT"
```

**Considerations:**
- Promtool output should not contain secrets
- If checking configs with sensitive URLs/credentials, they may appear in output
- Users should use Prometheus secret management features

## Permissions

### CODEOWNERS
```
* @Attest/audience-accounts
```

**Security Benefits:**
- Requires review from authorized team
- Prevents unauthorized changes
- Maintains accountability

### GitHub Actions Permissions
```yaml
# In workflows using this action, use minimal permissions
permissions:
  pull-requests: write  # Only if commenting
  contents: read        # For checkout
```

## Vulnerability Management

### Dependency Updates
**Current Dependencies:**
- Alpine base image
- bash, curl, wget, jq, git, openssh
- Prometheus (promtool) - user-specified version

**Process:**
1. Monitor Alpine security advisories
2. Update base image regularly
3. Pin package versions for reproducibility
4. Document supported Prometheus versions
5. Test with multiple versions

### Scanning
**Recommended Tools:**
- Docker image scanning (Trivy, Grype)
- Shell script linting (ShellCheck)
- GitHub Dependabot (if dependencies added)

## Incident Response

### If Security Issue Found
1. **Do not** disclose publicly immediately
2. Report to repository maintainers (@Attest/audience-accounts)
3. Coordinate disclosure timeline
4. Prepare patch and security advisory
5. Notify users through GitHub Security Advisory

### Security Contacts
- CODEOWNERS: @Attest/audience-accounts
- Follow responsible disclosure practices

## Best Practices Summary

1. **Validate all inputs** - Never trust user-provided data
2. **Verify downloads** - Use checksums and signatures
3. **Minimize permissions** - Least privilege principle
4. **Never log secrets** - Tokens, credentials, sensitive data
5. **Use official sources** - Only download from trusted origins
6. **Quote variables** - Prevent injection and word splitting
7. **Pin versions** - For reproducibility and security
8. **Regular updates** - Keep dependencies current
9. **Review changes** - Use CODEOWNERS and peer review
10. **Document security** - Keep this file updated

## Future Improvements

- [ ] Add checksum verification for promtool downloads
- [ ] Pin Alpine base image to specific version
- [ ] Add input validation for version format
- [ ] Validate GitHub API URLs before use
- [ ] Consider running as non-root user
- [ ] Add automated security scanning to CI
- [ ] Document security update process
- [ ] Add security policy (SECURITY.md)
