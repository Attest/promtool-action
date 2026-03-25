# Architecture Documentation

## Overview

`promtool-action` is a Docker-based GitHub Action that validates Prometheus configuration and rule files. The action downloads the Prometheus binary, extracts the `promtool` utility, runs validation checks, and optionally reports results to pull requests.

## High-Level Architecture

```mermaid
flowchart TB
    subgraph "GitHub Actions Workflow"
        WF[Workflow YAML]
        WF --> |triggers| AC[promtool-action]
    end

    subgraph "Docker Container (Alpine)"
        AC --> EP[entrypoint.sh]
        EP --> PI[parse_inputs]
        PI --> IP[install_promtool]
        IP --> |downloads| GH[GitHub Releases]
        IP --> PC[promtool_check.sh]
        PC --> PT[promtool binary]
        PT --> |validates| CF[Config/Rules Files]
    end

    subgraph "Outputs"
        PC --> GO[GITHUB_OUTPUT]
        PC --> |optional| PR[PR Comment via API]
    end

    CF --> |exit code| EC[Exit Code 0/1]
    GO --> WF
    EC --> WF
```

## Component Breakdown

### 1. Action Definition (`action.yml`)

Defines the GitHub Action interface and metadata.

**Inputs:**
- `prom_version` (optional): Prometheus version to download (default: `2.16.0`)
- `prom_check_subcommand` (required): Either `config` or `rules`
- `prom_check_files` (required): File path(s) or glob pattern to validate
- `prom_comment` (optional): Whether to post PR comments (default: `0`/false)

**Outputs:**
- `promtool_output`: Full output from promtool validation

**Runtime:**
- Uses Docker container built from local `Dockerfile`
- Container runs in workflow execution environment

### 2. Docker Container (`Dockerfile`)

```mermaid
graph LR
    A[Alpine 3 Base] --> B[Install Packages]
    B --> C[Copy Scripts to /src/]
    C --> D[Set ENTRYPOINT]

    subgraph "Installed Packages"
        B1[bash]
        B2[ca-certificates]
        B3[curl]
        B4[wget]
        B5[git]
        B6[jq]
        B7[openssh]
    end

    B --> B1
    B --> B2
    B --> B3
    B --> B4
    B --> B5
    B --> B6
    B --> B7
```

**Base Image:** `alpine:3` (lightweight Linux distribution)

**Dependencies:**
- `bash`: Shell for script execution
- `ca-certificates`: SSL/TLS verification
- `curl`: GitHub API communication
- `wget`: Download Prometheus tarball
- `git`: Version control operations (if needed)
- `jq`: JSON processing for API payloads
- `openssh`: SSH operations (if needed)

**Entry Point:** `/src/entrypoint.sh`

### 3. Orchestration Script (`src/entrypoint.sh`)

Main control flow that coordinates the validation process.

```mermaid
sequenceDiagram
    participant GHA as GitHub Actions
    participant EP as entrypoint.sh
    participant PI as parse_inputs()
    participant IP as install_promtool()
    participant PC as promtool_check.sh

    GHA->>EP: Execute with INPUT_* env vars
    EP->>PI: Parse environment variables
    PI-->>EP: prom_version, prom_check_subcommand, etc.
    EP->>IP: Install promtool
    IP->>IP: Download from GitHub Releases
    IP->>IP: Extract tarball
    IP->>IP: Move binary to /usr/bin/
    IP-->>EP: Installation complete
    EP->>PC: Source and execute
    PC-->>GHA: Exit code & outputs
```

**Functions:**

1. **`parse_inputs()`**
   - Reads `INPUT_*` environment variables (set by GitHub Actions)
   - Applies default values if not provided
   - Converts boolean strings (`"true"`, `"1"`) to internal format

2. **`install_promtool()`**
   - Constructs download URL: `https://github.com/prometheus/prometheus/releases/download/v{version}/prometheus-{version}.linux-amd64.tar.gz`
   - Downloads tarball using `wget`
   - Extracts using `tar -zxf`
   - Moves `promtool` binary to `/usr/bin/promtool`
   - Cleans up temporary files

3. **`main()`**
   - Sources `promtool_check.sh`
   - Calls functions in sequence
   - Handles overall execution flow

### 4. Validation Script (`src/promtool_check.sh`)

Executes validation and handles result reporting.

```mermaid
flowchart TD
    START[promtool_check] --> EXEC[Execute: promtool check subcommand files]
    EXEC --> CAPTURE[Capture output & exit code]
    CAPTURE --> CHECK{Exit code == 0?}

    CHECK -->|Yes| SUCCESS[Set status: Success]
    CHECK -->|No| FAIL[Set status: Failed]

    SUCCESS --> LOG1[Log success message]
    FAIL --> LOG2[Log error message]

    LOG1 --> PR_CHECK{PR event & comment enabled?}
    LOG2 --> PR_CHECK

    PR_CHECK -->|Yes| COMMENT[Format & post PR comment]
    PR_CHECK -->|No| SKIP[Skip comment]

    COMMENT --> OUTPUT[Write to GITHUB_OUTPUT]
    SKIP --> OUTPUT

    OUTPUT --> EXIT[Exit with promtool exit code]
```

**Function:** `promtool_check()`

**Steps:**

1. **Execute Promtool**
   ```bash
   check_output=$(promtool check ${prom_check_subcommand} ${prom_check_files} 2>&1)
   check_exit_code=${?}
   ```

2. **Determine Result**
   - Exit code `0` → Success
   - Exit code `!= 0` → Failure

3. **Conditional PR Comment**
   ```bash
   if [ "${GITHUB_EVENT_NAME}" == "pull_request" ] && [ "${prom_comment}" == "1" ]
   ```
   - Constructs markdown comment with collapsible details
   - Includes workflow metadata
   - Posts to PR via GitHub API using `GITHUB_ACCESS_TOKEN`

4. **Write Output**
   - Uses multi-line heredoc format for `GITHUB_OUTPUT`
   - Makes `promtool_output` available to subsequent workflow steps

5. **Exit**
   - Exits with promtool's exit code
   - Action succeeds if promtool validation passes
   - Action fails if promtool validation fails

## Data Flow

```mermaid
graph LR
    subgraph "Input"
        I1[Workflow YAML]
        I2[Prometheus Files]
        I3[GitHub Token]
    end

    subgraph "Processing"
        P1[Parse Inputs]
        P2[Download Promtool]
        P3[Run Validation]
        P4[Format Results]
    end

    subgraph "Output"
        O1[Exit Code]
        O2[GITHUB_OUTPUT]
        O3[PR Comment optional]
    end

    I1 --> P1
    I2 --> P3
    I3 --> O3
    P1 --> P2
    P2 --> P3
    P3 --> P4
    P4 --> O1
    P4 --> O2
    P4 --> O3
```

### Input Data
1. **GitHub Actions Inputs** → Environment variables (`INPUT_*`)
2. **Prometheus Files** → File paths in repository
3. **GitHub Context** → `GITHUB_EVENT_PATH`, `GITHUB_OUTPUT`, etc.
4. **GitHub Token** → `GITHUB_ACCESS_TOKEN` (optional)

### Processing Data
1. **Downloaded Binary** → `/tmp/prometheus-*.tar.gz` → `/usr/bin/promtool`
2. **Validation Output** → Captured stdout/stderr from promtool
3. **Exit Code** → Success/failure indicator

### Output Data
1. **Exit Code** → Determines workflow step success/failure
2. **GITHUB_OUTPUT** → `promtool_output` available to subsequent steps
3. **PR Comment** → Posted to GitHub via API (optional)

## State Management

The action is stateless between invocations. Each execution:

1. Starts fresh container
2. Downloads promtool binary
3. Runs validation
4. Exits

**No persistent state:**
- No caching of promtool binaries
- No state carried between workflow runs
- Each run is independent

## Error Handling

```mermaid
flowchart TD
    START[Start] --> DOWNLOAD{Download Success?}
    DOWNLOAD -->|No| ERR1[Exit 1: Download failed]
    DOWNLOAD -->|Yes| EXTRACT{Extract Success?}

    EXTRACT -->|No| ERR2[Exit 1: Extract failed]
    EXTRACT -->|Yes| VALIDATE[Run Validation]

    VALIDATE --> RESULT{Validation Success?}
    RESULT -->|Yes| EXIT0[Exit 0]
    RESULT -->|No| EXIT1[Exit 1]

    ERR1 --> END
    ERR2 --> END
    EXIT0 --> END
    EXIT1 --> END
```

**Error Conditions:**

1. **Download Failure**
   - Promtool version doesn't exist
   - Network issues
   - GitHub releases unavailable
   - **Effect:** Action exits with code 1

2. **Extraction Failure**
   - Corrupted tarball
   - Insufficient disk space
   - **Effect:** Action exits with code 1

3. **Validation Failure**
   - Invalid Prometheus syntax
   - Malformed YAML
   - Invalid PromQL expressions
   - **Effect:** Action exits with code 1, reports errors

4. **API Failure** (PR comments)
   - Invalid token
   - Insufficient permissions
   - Network issues
   - **Effect:** Logged but doesn't fail action

## Security Model

```mermaid
graph TB
    subgraph "Trust Boundaries"
        TB1[User Inputs]
        TB2[Downloaded Binary]
        TB3[GitHub API]
    end

    subgraph "Validation"
        V1[Input Parsing]
        V2[Download from Official Source]
        V3[Token Permissions]
    end

    subgraph "Execution"
        E1[Docker Container Isolation]
        E2[Limited Filesystem Access]
        E3[Network: GitHub only]
    end

    TB1 --> V1
    TB2 --> V2
    TB3 --> V3

    V1 --> E1
    V2 --> E1
    V3 --> E3
```

**Security Considerations:**

1. **Container Isolation**: Runs in isolated Docker container
2. **Input Validation**: Minimal (should be improved)
3. **Download Trust**: Downloads from `github.com/prometheus/prometheus`
4. **Token Scope**: Uses `GITHUB_TOKEN` with minimal permissions
5. **Network Access**: Only to GitHub (releases + API)

**See:** `.agents/rules/security.md` for detailed security guidelines.

## Performance Characteristics

**Typical Execution Time:**
1. Container startup: ~1-2 seconds
2. Promtool download: ~3-10 seconds (depends on version/network)
3. Extraction: ~1 second
4. Validation: <1 second for most files
5. PR comment: ~1 second (if enabled)

**Total:** ~5-15 seconds per run

**Optimization Opportunities:**
- Cache promtool binary across runs (requires workflow-level caching)
- Pre-build Docker image with common versions
- Use multi-stage Docker build

## Extensibility

### Adding New Subcommands

Promtool supports additional commands (e.g., `test`, `tsdb`):

```yaml
# In action.yml - add to accepted values (documentation)
prom_check_subcommand:
  description: 'Promtool check command - config|rules|test'
```

```bash
# In scripts - no code changes needed
# Already passes through: promtool check ${prom_check_subcommand} ${prom_check_files}
```

### Adding New Outputs

To expose additional information:

1. Capture in `promtool_check.sh`
2. Write to `$GITHUB_OUTPUT` using heredoc format
3. Document in `action.yml` outputs section

### Supporting Other Architectures

Currently hardcoded to `linux-amd64`:

```bash
url="https://github.com/prometheus/prometheus/releases/download/v${prom_version}/prometheus-${prom_version}.linux-amd64.tar.gz"
```

**To support ARM:**
1. Detect architecture (e.g., via environment variable)
2. Adjust download URL accordingly
3. Test on ARM runners

## Dependencies

```mermaid
graph TD
    A[promtool-action] --> B[Docker]
    A --> C[GitHub Actions Runtime]
    A --> D[GitHub Releases API]
    A --> E[GitHub REST API]

    B --> F[Alpine Linux]
    F --> G[bash]
    F --> H[wget/curl]
    F --> I[jq]

    D --> J[Prometheus Releases]
    E --> K[PR Comments]
```

**External Dependencies:**
- GitHub Actions infrastructure
- GitHub Releases (for promtool download)
- GitHub REST API (for PR comments)
- Docker runtime
- Alpine package repositories

**Internal Dependencies:**
- `entrypoint.sh` depends on `promtool_check.sh`
- Both scripts depend on environment variables set by GitHub Actions

## Deployment

**Action Published As:**
- GitHub repository: `karancode/promtool-action`
- Tagged releases: `v0.0.1`, etc.

**Usage Pattern:**
```yaml
uses: karancode/promtool-action@v0.0.1
```

**Release Process:**
1. Make changes to scripts/Dockerfile
2. Test locally with Docker
3. Commit and push
4. Create Git tag (e.g., `v0.0.2`)
5. Users reference new tag in workflows

**No Build Process Required:**
- Docker image built on-the-fly by GitHub Actions
- No pre-built images to publish
- Dockerfile in repository is the source of truth

## Future Enhancements

1. **Caching**: Cache promtool binaries between runs
2. **Checksum Verification**: Verify downloaded tarballs
3. **Multi-Architecture**: Support ARM runners
4. **Additional Promtool Commands**: Support `promtool test`, `promtool tsdb`, etc.
5. **Custom Download URLs**: Support enterprise/mirror scenarios
6. **Improved Error Messages**: More actionable failure descriptions
7. **Metrics**: Expose validation statistics (rules checked, errors found)
8. **Parallel Validation**: Check multiple files concurrently
9. **Pre-built Images**: Publish Docker images with common versions

## Troubleshooting

**Common Issues:**

1. **Download Fails**
   - Check Prometheus version exists
   - Verify network connectivity
   - Check GitHub rate limits

2. **Permission Denied**
   - Ensure `GITHUB_ACCESS_TOKEN` provided (if commenting)
   - Check token permissions include `pull_requests: write`

3. **Validation Errors**
   - Review promtool output in logs
   - Test files locally with same promtool version
   - Check Prometheus documentation for syntax

4. **No PR Comment**
   - Verify `prom_comment: true` set
   - Ensure event is `pull_request`
   - Check token permissions

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Promtool Command Reference](https://prometheus.io/docs/prometheus/latest/command-line/promtool/)
- [Docker Actions](https://docs.github.com/en/actions/creating-actions/creating-a-docker-container-action)
