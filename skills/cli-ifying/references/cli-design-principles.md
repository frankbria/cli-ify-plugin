# CLI Design Principles for Dual-Use Tools

This document outlines the philosophy and design principles for creating CLI tools that work equally well for humans and AI agents.

## The Problem

When building agent skills (reusable capabilities), there's tension between:

- **API-first design**: Skills as functions/classes—great for programmatic use, but hard to debug and test manually
- **GUI-first design**: Skills as visual tools—easy for humans, but agents can't invoke them

Teams end up building two interfaces or choosing one audience over the other.

## The Solution: CLI-First Design

Design all skills as **CLI tools first**. A well-designed CLI is naturally dual-use: humans can invoke it from the terminal, and agents can invoke it via shell commands.

```
┌─────────────────┐
│   Skill Logic   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  CLI Interface  │
└────────┬────────┘
         │
    ┌────┼────┬────────────┐
    ▼    ▼    ▼            ▼
 Human Agent Scripts    Cron
```

## Core Principles

### 1. One Script, One Skill

Each capability is a standalone executable:

```
skills/
├── trello.sh          # Trello operations
├── github-issues.sh   # GitHub issue management
├── priority-report.sh # Composes other skills
└── deploy.sh          # Deployment operations
```

### 2. Subcommands for Operations

Organize related operations under a single script:

```bash
skill.sh list              # List resources
skill.sh get <id>          # Get specific resource
skill.sh create            # Create new resource
skill.sh update <id>       # Update existing
skill.sh delete <id>       # Delete resource
skill.sh run               # Execute main operation
```

### 3. Structured Output

JSON for programmatic use, readable for humans:

```bash
# Programmatic output (default for non-TTY)
$ trello.sh boards
{"id": "abc123", "name": "Personal", "url": "..."}
{"id": "def456", "name": "Work", "url": "..."}

# Human-readable output (when TTY detected)
$ trello.sh boards
ID       NAME       URL
abc123   Personal   https://trello.com/b/...
def456   Work       https://trello.com/b/...
```

### 4. Exit Codes for Chaining

Use meaningful exit codes to enable `&&` and `||` chaining:

```bash
# Standard exit codes
0   # Success
1   # General error
2   # Invalid arguments
3   # Missing dependencies
4   # Network/API error
5   # Permission denied
126 # Command not executable
127 # Command not found
```

Example chaining:
```bash
# Only deploy if tests pass
./test.sh && ./deploy.sh

# Fallback on failure
./primary-api.sh || ./backup-api.sh
```

### 5. Environment Configuration

Credentials and configuration via environment variables:

```bash
# In ~/.envrc or shell config
export TRELLO_API_KEY="..."
export TRELLO_TOKEN="..."
export GITHUB_TOKEN="..."

# Script reads from environment
: "${TRELLO_API_KEY:?TRELLO_API_KEY must be set}"
```

## CLI Structure Template

### Bash

```bash
#!/usr/bin/env bash
set -euo pipefail

# ============================================================
# Script: skill-name.sh
# Purpose: Brief description
# ============================================================

# Configuration
: "${API_KEY:?API_KEY environment variable required}"
: "${OUTPUT_FORMAT:=json}"

# Help text
show_help() {
    cat << 'EOF'
skill-name - Brief description

USAGE:
    skill-name <command> [options] [arguments]

COMMANDS:
    list                List all resources
    get <id>            Get resource by ID
    create [options]    Create new resource
    help                Show this help

OPTIONS:
    -o, --output FORMAT   Output format: json (default) | text | csv
    -q, --quiet           Suppress non-essential output
    -v, --verbose         Enable verbose output
    -h, --help            Show this help

ENVIRONMENT:
    API_KEY               Required. API authentication key
    OUTPUT_FORMAT         Default output format (json)

EXAMPLES:
    skill-name list
    skill-name get abc123
    skill-name create --name "New Item"
    skill-name list | jq '.[0].id'

EXIT CODES:
    0   Success
    1   General error
    2   Invalid arguments
    3   Missing dependencies
    4   API error
EOF
}

# Output helpers
output_json() {
    echo "$1"
}

output_text() {
    echo "$1" | jq -r '.[] | "\(.id)\t\(.name)"'
}

error() {
    echo "ERROR: $*" >&2
}

# Command implementations
cmd_list() {
    local result
    result=$(curl -s "https://api.example.com/items" \
        -H "Authorization: Bearer $API_KEY")

    case "$OUTPUT_FORMAT" in
        json) output_json "$result" ;;
        text) output_text "$result" ;;
        *)    output_json "$result" ;;
    esac
}

cmd_get() {
    local id="${1:?ID required}"
    curl -s "https://api.example.com/items/$id" \
        -H "Authorization: Bearer $API_KEY"
}

# Main
main() {
    local cmd="${1:-help}"
    shift || true

    case "$cmd" in
        list)   cmd_list "$@" ;;
        get)    cmd_get "$@" ;;
        help)   show_help ;;
        -h|--help) show_help ;;
        *)
            error "Unknown command: $cmd"
            show_help >&2
            exit 2
            ;;
    esac
}

main "$@"
```

### PowerShell

```powershell
#Requires -Version 5.1
<#
.SYNOPSIS
    Brief description of the skill
.DESCRIPTION
    Detailed description
.EXAMPLE
    .\skill-name.ps1 list
.EXAMPLE
    .\skill-name.ps1 get abc123
#>

[CmdletBinding()]
param(
    [Parameter(Position=0)]
    [ValidateSet('list', 'get', 'create', 'help')]
    [string]$Command = 'help',

    [Parameter(Position=1, ValueFromRemainingArguments)]
    [string[]]$Arguments,

    [Parameter()]
    [ValidateSet('json', 'text', 'csv')]
    [string]$OutputFormat = 'json',

    [Parameter()]
    [switch]$Quiet,

    [Parameter()]
    [switch]$Verbose
)

# Configuration
$ApiKey = $env:API_KEY
if (-not $ApiKey) {
    Write-Error "API_KEY environment variable required"
    exit 3
}

function Show-Help {
    @"
skill-name - Brief description

USAGE:
    .\skill-name.ps1 <command> [options] [arguments]

COMMANDS:
    list                List all resources
    get <id>            Get resource by ID
    create [options]    Create new resource
    help                Show this help

OPTIONS:
    -OutputFormat       Output format: json (default) | text | csv
    -Quiet              Suppress non-essential output
    -Verbose            Enable verbose output

ENVIRONMENT:
    API_KEY             Required. API authentication key

EXAMPLES:
    .\skill-name.ps1 list
    .\skill-name.ps1 get abc123
    .\skill-name.ps1 list | ConvertFrom-Json | Select-Object name
"@
}

function Invoke-List {
    $headers = @{ Authorization = "Bearer $ApiKey" }
    $result = Invoke-RestMethod -Uri "https://api.example.com/items" -Headers $headers

    switch ($OutputFormat) {
        'json' { $result | ConvertTo-Json -Depth 10 }
        'text' { $result | Format-Table -AutoSize }
        'csv'  { $result | ConvertTo-Csv -NoTypeInformation }
    }
}

function Invoke-Get {
    param([string]$Id)

    if (-not $Id) {
        Write-Error "ID required"
        exit 2
    }

    $headers = @{ Authorization = "Bearer $ApiKey" }
    Invoke-RestMethod -Uri "https://api.example.com/items/$Id" -Headers $headers |
        ConvertTo-Json -Depth 10
}

# Main
switch ($Command) {
    'list'   { Invoke-List }
    'get'    { Invoke-Get -Id $Arguments[0] }
    'create' { Invoke-Create @Arguments }
    'help'   { Show-Help }
    default  {
        Write-Error "Unknown command: $Command"
        Show-Help
        exit 2
    }
}
```

## Composition Pattern

CLIs should compose with each other:

```bash
#!/bin/bash
# priority-report.sh - Composes multiple skill CLIs

echo "## Priority Report $(date +%Y-%m-%d)"
echo

echo "### GitHub PRs Awaiting Review"
gh pr list --search "review-requested:@me" --json title,url | \
    jq -r '.[] | "- [\(.title)](\(.url))"'
echo

echo "### Trello Cards Due Today"
~/.claude/skills/trello/scripts/trello.sh cards today | \
    jq -r '.[] | "- \(.name) (\(.board))"'
echo

echo "### Asana Tasks"
~/.claude/skills/asana/scripts/asana.sh tasks --due today | \
    jq -r '.[] | "- \(.name)"'
```

## Best Practices

### Do

- Use `set -euo pipefail` in bash
- Provide `--help` for every command
- Output JSON by default for programmatic use
- Use stderr for errors and logs
- Include meaningful exit codes
- Read secrets from environment only
- Make scripts idempotent when possible
- Include usage examples in help text

### Don't

- Don't require interactive input
- Don't hard-code credentials
- Don't assume specific directory structure
- Don't output mixed format (JSON + text)
- Don't ignore exit codes from subprocesses
- Don't swallow errors silently

## Trade-offs

**Pros:**
- Dual-use by default
- Debuggable
- Composable
- Portable
- Transparent
- Testable

**Cons:**
- Shell limitations for complex data
- Less structured error handling
- Process spawn overhead
- No persistent state
- Windows requires WSL or Git Bash

**When NOT to use CLI:**
- High-frequency calls (>100/sec)
- Complex object graphs
- Real-time streaming
- Tasks requiring Claude's reasoning
