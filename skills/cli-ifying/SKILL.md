---
name: cli-ifying
description: Convert Claude Code slash commands, skills, or agents into standalone CLI tools for Linux/Mac (Bash) and Windows (PowerShell). Use when user says "cli-ify", "make this a CLI", "convert to command line", or wants to reuse a skill outside of Claude Code. First assesses suitability - pushes back if the capability is better as a Claude skill.
---

# CLI-ifying Claude Code Capabilities

Converts Claude Code slash commands, skills, or agents into dual-use CLI tools that work for both humans and AI agents. Can also create CLIs from scratch using natural language prompts.

## Invocation Modes

### Mode 1: Convert Existing Capability

```bash
/cli-ify --<platforms> command:<path-to-command>
/cli-ify --<platforms> skill:<skill-name>
/cli-ify --<platforms> agent:<agent-name>
```

### Mode 2: Create from Prompt (NEW)

```bash
/cli-ify --prompt "Description of what the CLI should do"
```

**Platforms**:
- `--linux` or `--bash` → Bash script (works on Linux/Mac)
- `--windows` or `--powershell` → PowerShell script
- `--all` → Both platforms (default for prompt mode)

## Quick Reference

```bash
# Convert existing capabilities
/cli-ify --linux,windows command:fhb:update-readme
/cli-ify --linux skill:validating-pre-commit
/cli-ify --all skill:verifying-implementation

# Create from natural language
/cli-ify --prompt "Sync all git repos in a directory"
/cli-ify --prompt "Find and delete node_modules folders recursively"
/cli-ify --prompt "Convert images to WebP format in batch"
```

---

# Mode 2: Create CLI from Prompt

When user provides `--prompt`, enter **Socratic discovery mode** to ensure the CLI is well-scoped and follows Unix philosophy.

## Step P1: Understand the Intent

Read the prompt and identify:
- **Core action**: What is the user trying to accomplish?
- **Input sources**: Files, directories, APIs, stdin?
- **Output targets**: Stdout, files, exit codes?
- **Scope concerns**: Is this too broad for a single CLI?

## Step P2: Socratic Discovery Questions

Before creating the CLI, ask focused questions to ensure:
1. The scope is atomic and CLI-appropriate
2. The inputs and outputs are well-defined
3. The user understands the trade-offs

### Question Framework

Ask 2-4 questions from these categories:

**Scope Clarification** (always ask at least one):
```
"Should this CLI [do X] or [do Y], or both? For Unix-style tools, each CLI should do one thing well."

"You mentioned [broad task]. Should we focus on just [specific subtask], or do you need the full scope?"

"This could be broken into: (a) [subtask1], (b) [subtask2]. Which is the priority, or should I create both?"
```

**Input/Output Clarification**:
```
"What's the primary input: (a) directory path, (b) file list, (c) stdin, or (d) something else?"

"Should output be: (a) JSON for piping to other tools, (b) human-readable text, or (c) both (detect TTY)?"

"Should this modify files in place, or output to stdout/new files?"
```

**Behavior Clarification**:
```
"For errors, should the CLI: (a) stop immediately, (b) continue and report at end, or (c) user's choice via flag?"

"Should this require confirmation for destructive operations, or is --force / --dry-run enough?"

"What should happen if [edge case]?"
```

**Environment Clarification**:
```
"Does this need any credentials or API keys? If so, from environment variables?"

"Should this work on both Linux/Mac and Windows, or just one platform?"
```

### Atomic Task Validation

A good CLI task should pass the **"one sentence" test**:

> Can you describe what this CLI does in one clear sentence without using "and" or "then"?

If not, suggest breaking it down:

```markdown
Your description includes multiple operations. Let's break this into atomic CLIs:

1. **find-git-repos**: Recursively find all git repositories in a directory
2. **git-sync**: Pull latest changes for a given git repository
3. **batch-git-sync**: Compose the above to sync multiple repos

Which should we create first? Or would you prefer a single composed script?
```

## Step P3: Confirm Scope

After questions, summarize the CLI spec for confirmation:

```markdown
## Proposed CLI: `<name>`

**One-liner**: <clear description in one sentence>

**Inputs**:
- Arg 1: <description>
- Option --<flag>: <description>

**Processing**:
1. <step 1>
2. <step 2>

**Outputs**:
- stdout: <what it outputs>
- Exit 0: success
- Exit 1: <error condition>

**Does NOT do** (scope boundaries):
- <thing that's out of scope>
- <thing that would require separate CLI>

Shall I proceed with this specification?
```

## Step P4: Generate the CLI

Once confirmed, generate the CLI using the same process as Mode 1 (Steps 5-7 below).

## Prompt Mode Example

**User**: `/cli-ify --prompt "Create a CLI that syncs all the git repos in a directory"`

**Assistant Questions**:

1. **Scope**: "Should this CLI only do `git pull`, or also handle `git fetch`, stash/unstash, and branch switching?"

2. **Input**: "Should the directory argument be required, or default to current directory?"

3. **Behavior**: "For repos with uncommitted changes, should it: (a) skip with warning, (b) stash before pulling, or (c) fail?"

4. **Output**: "Should output be: (a) progress per repo (human-friendly), (b) JSON summary (script-friendly), or (c) both?"

**User Response**: "Just git pull. Current directory default. Skip repos with uncommitted changes. Human-friendly output with option for JSON."

**Confirmed Spec**:

```markdown
## Proposed CLI: `git-sync-repos`

**One-liner**: Pull latest changes for all git repositories in a directory tree

**Inputs**:
- Arg 1 (optional): Target directory (default: current directory)
- --json: Output JSON summary instead of progress text
- --include-dirty: Attempt to pull repos with uncommitted changes (default: skip)

**Processing**:
1. Find all directories containing .git folder
2. For each, check for uncommitted changes
3. If clean (or --include-dirty), run git pull
4. Collect results

**Outputs**:
- stdout: Progress messages or JSON summary
- Exit 0: All repos synced successfully
- Exit 1: One or more repos failed

**Does NOT do**:
- Branch switching
- Stashing uncommitted changes
- Handling merge conflicts

Shall I proceed with this specification?
```

---

# Mode 1: Convert Existing Capability

## Step 1: Parse the Request

Extract from user input:
1. **Platforms**: Which platform(s) to generate
2. **Source type**: `command:`, `skill:`, or `agent:`
3. **Source path**: The fully-qualified name

```bash
# Parse examples
/cli-ify --linux,windows command:fhb:update-readme
# → Platforms: [linux, windows]
# → Source type: command
# → Source path: fhb:update-readme → ~/.claude/commands/fhb/update-readme.md

/cli-ify --all skill:validating-pre-commit
# → Platforms: [linux, windows]
# → Source type: skill
# → Source path: validating-pre-commit → ~/.claude/skills/validating-pre-commit/SKILL.md
```

## Step 2: Load the Source

Read the source file based on type:

**For Commands** (`~/.claude/commands/<path>.md`):
- Parse YAML frontmatter for `allowed-tools` and `description`
- Extract the markdown instructions
- Note file references (`@file`) and shell commands (`` !`command` ``)

**For Skills** (`~/.claude/skills/<name>/SKILL.md`):
- Parse YAML frontmatter for `name` and `description`
- Extract methodology and step-by-step instructions
- Note referenced files (`./reference.md`)

**For Agents** (check Task tool subagent_type or CLAUDE.md):
- Extract the agent's purpose and tools
- Identify key workflows

## Step 3: Assess CLI Suitability

**CRITICAL**: Not everything should become a CLI. Assess suitability first.

See `./references/suitability-assessment.md` for full criteria.

### Quick Assessment Rules

**Good CLI Candidates** (proceed with conversion):
- Read files → transform → write output
- Call APIs → process response → output result
- Git operations → status checks → formatted reports
- File scanning → pattern matching → JSON output
- Configuration validation → success/error reporting

**Poor CLI Candidates** (push back):
- Requires multi-turn conversation with user
- Needs deep codebase reasoning/understanding
- Makes architectural decisions requiring context
- Involves subjective judgment (code review, refactoring)
- Requires Claude's knowledge (explanation, mentoring)

### Assessment Template

When assessing, answer these questions:

```markdown
## CLI Suitability Assessment

**Source**: [command/skill/agent name]

**Core Operations**:
1. [What does it read/input?]
2. [What processing does it do?]
3. [What does it output?]

**Deterministic?**: Yes/No - Can output be predicted from inputs?
**Stateless?**: Yes/No - Does each run start fresh?
**CLI-able I/O?**: Yes/No - Can inputs come from args/stdin, outputs to stdout/files?
**Human-debuggable?**: Yes/No - Can a human understand and verify output?

**Verdict**: ✅ Good CLI candidate / ⚠️ Partial candidate / ❌ Not suitable

**If Not Suitable, Explain Why**:
[Clear explanation of why this should remain a Claude skill]
```

### When to Push Back

If assessment shows poor fit, respond:

```markdown
## Not Recommended for CLI Conversion

**Source**: [name]

**Reason**: This capability relies on [specific Claude feature] which cannot be replicated in a standalone CLI:
- [Reason 1]
- [Reason 2]

**Recommendation**: Keep this as a [command/skill/agent] and invoke via Claude Code.

**Alternative**: If you need automation, consider:
- Using `claude --dangerously-skip-permissions` in scripts
- Creating a wrapper script that calls Claude with this skill
- Extracting only the [specific subset] that IS CLI-suitable
```

## Step 4: Extract CLI Operations

For suitable candidates, extract discrete operations:

```markdown
## Extracted Operations

1. **Operation**: [name]
   - Input: [what it reads]
   - Process: [what it does]
   - Output: [what it produces]
   - Exit codes: 0=success, 1=failure, 2=...

2. **Operation**: [name]
   ...
```

Map to CLI structure:

```
script-name <subcommand> [options] [arguments]

Subcommands:
  list        - List resources
  get <id>    - Get specific resource
  create      - Create new resource
  update <id> - Update existing resource
  delete <id> - Delete resource
  run         - Execute the main operation
```

---

# Common Steps (Both Modes)

## Step 5: Generate CLI Scripts

### For Bash (Linux/Mac)

See `./templates/bash-template.sh` for full template.

**Core structure**:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Configuration from environment
: "${CONFIG_VAR:=default_value}"

# Help text
show_help() {
    cat << 'EOF'
Usage: script-name <command> [options]

Commands:
    run         Execute main operation
    list        List resources
    help        Show this help

Options:
    -o, --output FORMAT   Output format (json|text)
    -v, --verbose         Verbose output
    -h, --help            Show help

Environment:
    CONFIG_VAR            Description of variable

Examples:
    script-name run --output json
    script-name list | jq '.[] | .name'
EOF
}

# Main logic
main() {
    local cmd="${1:-help}"
    shift || true

    case "$cmd" in
        run)    do_run "$@" ;;
        list)   do_list "$@" ;;
        help)   show_help ;;
        *)      echo "Unknown command: $cmd" >&2; exit 1 ;;
    esac
}

main "$@"
```

### For PowerShell (Windows)

See `./templates/powershell-template.ps1` for full template.

**Core structure**:
```powershell
#Requires -Version 5.1
[CmdletBinding()]
param(
    [Parameter(Position=0)]
    [ValidateSet('run', 'list', 'help')]
    [string]$Command = 'help',

    [Parameter()]
    [ValidateSet('json', 'text')]
    [string]$OutputFormat = 'text',

    [Parameter()]
    [switch]$Verbose
)

# Configuration from environment
$ConfigVar = $env:CONFIG_VAR ?? 'default_value'

function Show-Help {
    @"
Usage: script-name.ps1 <command> [options]

Commands:
    run         Execute main operation
    list        List resources
    help        Show this help

Options:
    -OutputFormat   Output format (json|text)
    -Verbose        Verbose output

Environment:
    CONFIG_VAR      Description of variable

Examples:
    .\script-name.ps1 run -OutputFormat json
    .\script-name.ps1 list | ConvertFrom-Json | Select-Object name
"@
}

# Main logic
switch ($Command) {
    'run'  { Invoke-Run }
    'list' { Invoke-List }
    'help' { Show-Help }
    default { Write-Error "Unknown command: $Command"; exit 1 }
}
```

## Step 6: Output Location

**For converted capabilities**:
```
~/.claude/skills/<source-skill>/scripts/
    ├── <name>.sh          # Bash version
    └── <name>.ps1         # PowerShell version

# For commands, create a scripts directory:
~/.claude/commands/<path>/scripts/
    ├── <name>.sh
    └── <name>.ps1
```

**For prompt-created CLIs**:
```
~/.local/bin/<name>              # Symlink or direct
# or
~/bin/<name>                     # User bin directory
# or create a dedicated location:
~/.claude/cli-tools/
    ├── <name>.sh
    └── <name>.ps1
```

Make bash scripts executable:
```bash
chmod +x <script-path>/*.sh
```

## Step 7: Generate Documentation

Create a README alongside the scripts:

```markdown
# CLI: <name>

<One-liner description>

## Installation

### Linux/Mac
\`\`\`bash
# Add to PATH or symlink
ln -s <script-path>/<name>.sh ~/.local/bin/<name>
\`\`\`

### Windows (PowerShell)
\`\`\`powershell
# Add to PATH or create alias
Set-Alias -Name <name> -Value "<script-path>\<name>.ps1"
\`\`\`

## Usage

\`\`\`bash
# Show help
<name> help

# Run main operation
<name> run [options]
\`\`\`

## Exit Codes

- 0: Success
- 1: General error
- 2: Invalid arguments
- 3: Missing dependencies

## Examples

### Human usage
\`\`\`bash
<name> run --output json | jq '.items | length'
\`\`\`

### Scripting
\`\`\`bash
for item in $(<name> list --format ids); do
    <name> process "$item"
done
\`\`\`
```

## Validation

After generating CLIs, verify:

```bash
# Check syntax
bash -n script.sh
pwsh -NoExecute -File script.ps1

# Check help works
./script.sh help
.\script.ps1 help

# Test basic operation
./script.sh run --help
```

---

# Core Philosophy

See `./references/cli-design-principles.md` for full rationale.

**Why CLI-first?**
- **Dual-use by default**: Same interface for humans and agents
- **Debuggable**: Run manually to test, inspect output
- **Composable**: Pipe, chain, and combine with Unix tools
- **Portable**: Works in any shell, no runtime dependencies

**CLI Design Checklist**:
- Standalone executable with shebang
- Help text via `--help` or no-args
- Subcommands for CRUD operations
- JSON output (pipe to `jq` for formatting)
- Credentials from environment
- Meaningful exit codes (0 = success)
- Stderr for errors, stdout for data

**Unix Philosophy**:
- Do one thing well
- Work with text streams
- Compose with other tools
- Fail fast and loud

---

# Examples

## Example 1: Prompt → git-sync-repos

**Prompt**: "Sync all git repos in a directory"

**Generated CLI** (after Socratic discovery):
```bash
git-sync-repos [directory] [--json] [--include-dirty]
git-sync-repos ~/projects           # Sync all repos in ~/projects
git-sync-repos --json               # Output JSON summary
```

## Example 2: Convert validating-pre-commit

**Source**: `skill:validating-pre-commit`

**Generated CLI**:
```bash
pre-commit-validate run       # Run all validations
pre-commit-validate python    # Python only
pre-commit-validate typescript # TypeScript only
pre-commit-validate --major   # Include implementation check
```

## Example 3: Push Back on code-reviewer

**Source**: `agent:code-reviewer`

**Response**: Not suitable - requires deep semantic understanding. Keep as agent.

---

# Best Practices

1. **Always assess/question first** - Don't blindly create
2. **Scope to atomic tasks** - One thing, done well
3. **Prefer composition** - Use existing CLI tools (gh, jq, git)
4. **JSON by default** - Structured output for chaining
5. **Exit codes matter** - Enable `&&` chaining
6. **Stderr for errors** - Keep stdout clean for data
7. **Environment for secrets** - Never hardcode credentials
8. **Idempotent operations** - Safe to re-run

---

# Related Skills

- `building-skills` - Create new Claude Code skills
- `managing-gitops-ci` - Git and CI/CD operations
- `validating-pre-commit` - Example of CLI-suitable skill

# Related References

- `./references/cli-design-principles.md` - Full Unix philosophy rationale
- `./references/suitability-assessment.md` - Detailed assessment criteria
- `./references/conversion-examples.md` - Complete worked examples
- `./templates/bash-template.sh` - Bash script template
- `./templates/powershell-template.ps1` - PowerShell script template
