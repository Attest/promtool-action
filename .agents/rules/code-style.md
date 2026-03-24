# Code Style Guide

## Bash Scripting Conventions

### General Principles
- Use `#!/bin/bash` shebang for all bash scripts
- Set executable permissions on all shell scripts
- Use descriptive function and variable names
- Prefer lowercase with underscores for variables: `prom_version`, `check_exit_code`

### Functions
```bash
# Function definition style - no space before ()
function function_name {
    # implementation
}
```

- Encapsulate logical units in functions
- Functions should be declared before they are called
- Main logic should be in a `main` function

### Variables
```bash
# Input parsing from GitHub Actions
if [ "${INPUT_VAR_NAME}" != "" ]; then
    local_var=${INPUT_VAR_NAME}
fi

# Default values
variable='default_value'
```

- Use `${VAR}` braces for variable expansion
- Quote variables to prevent word splitting: `"${var}"`
- Use double quotes for strings with variables, single quotes for literals
- Prefix GitHub Action inputs with `INPUT_` (automatically provided)

### Conditionals
```bash
# Standard if syntax
if [ ${exit_code} -eq 0 ]; then
    # success case
fi

if [ ${exit_code} -ne 0 ]; then
    # failure case
fi

# String comparison
if [ "${var}" == "value" ]; then
    # implementation
fi
```

- Use `[` (test) for conditionals
- Use `-eq`, `-ne` for numeric comparison
- Use `==`, `!=` for string comparison
- Always quote strings in comparisons

### Error Handling
```bash
command
if [ "${?}" -ne 0 ]; then
    echo "Error message"
    exit 1
fi
```

- Check `${?}` immediately after commands that may fail
- Exit with code 1 on errors
- Provide descriptive error messages with context
- Use `echo` for logging to stdout/stderr

### Command Execution
```bash
# Capture output and exit code
output=$(command 2>&1)
exit_code=${?}
```

- Use `$(...)` for command substitution (not backticks)
- Redirect stderr to stdout with `2>&1` when capturing full output
- Store exit codes in variables for later logic

### Commenting
```bash
# Single line comments above code blocks
# Explain why, not what (when the what is obvious)

# gather check promtool output
# exit code 0 - success
```

- Use comments to explain non-obvious logic
- Comment sections of scripts that handle different scenarios

### Script Structure
```bash
#!/bin/bash

# Function definitions first
function helper_function {
    # implementation
}

function main_function {
    # implementation
}

# Main execution last
main_function "${*}"
```

- Helper functions defined first
- Main function defined last
- Script execution at bottom

## Docker

### Dockerfile Style
```dockerfile
FROM alpine:3
RUN apk add --update --no-cache package1 package2
RUN ["bin/sh", "-c", "mkdir -p /path"]
COPY ["src", "/dest/"]
ENTRYPOINT ["/script.sh"]
```

- Use specific Alpine version (e.g., `alpine:3`)
- Combine package installations in single `RUN` command
- Use `--no-cache` to reduce image size
- Use JSON array syntax for ENTRYPOINT when possible

## YAML (GitHub Actions)

### action.yml Structure
```yaml
name: 'Action Name'
author: 'author <email>'
description: 'Brief description'
branding:
  icon: 'icon-name'
  color: 'color'
inputs:
  input_name:
    description: 'Description'
    required: true|false
    default: 'default value'
outputs:
  output_name:
    description: 'Description'
runs:
  using: 'docker'
  image: 'Dockerfile'
```

- Use single quotes for strings
- Include branding for marketplace visibility
- Clearly document required vs optional inputs
- Provide sensible defaults

## Naming Conventions

### Files
- `lowercase_with_underscores.sh` for shell scripts
- `action.yml` for GitHub Action definition (not `action.yaml`)
- `Dockerfile` (capitalized, no extension)

### Environment Variables
- `UPPERCASE_WITH_UNDERSCORES` for environment variables
- `INPUT_*` prefix for GitHub Action inputs (automatic)
- `GITHUB_*` prefix for GitHub-provided variables

### Functions
- `lowercase_with_underscores` for function names
- Descriptive names: `parse_inputs`, `install_promtool`, `promtool_check`

## Output Formatting

### GitHub Actions Output
```bash
# Multi-line output format (new GitHub Actions standard)
echo "output_name<<EOF" >> "$GITHUB_OUTPUT"
echo "${content}" >> "$GITHUB_OUTPUT"
echo "EOF" >> "$GITHUB_OUTPUT"
```

### User Messages
```bash
echo "check: info: informational message"
echo "check: error: error message"
```

- Prefix messages with context (`check:`)
- Use severity levels (`info:`, `error:`)
- Provide actionable information

## Best Practices

1. **Fail fast**: Exit early on errors with descriptive messages
2. **Quote variables**: Always quote to prevent word splitting
3. **Check exit codes**: Verify command success before proceeding
4. **Use functions**: Encapsulate logic for clarity and reusability
5. **Source other scripts**: Use `source` to include utility scripts
6. **Clean up**: Remove temporary files after use
7. **Document inputs**: Comment expected environment variables at function start
