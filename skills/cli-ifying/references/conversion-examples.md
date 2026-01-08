# CLI Conversion Examples

This document provides complete worked examples of converting Claude Code capabilities to CLIs.

## Example 1: validating-pre-commit → pre-commit-validate

### Source Analysis

**Source**: `skill:validating-pre-commit`

**What it does**:
1. Detects staged git files
2. Categorizes by language (Python/TypeScript)
3. Runs language-specific validators (ruff, mypy, lint, type-check)
4. Reports pass/fail with details
5. Blocks commit if validation fails

**Assessment**:
- ✅ All inputs come from git and file system
- ✅ Processing is deterministic (same files → same results)
- ✅ Output is structured (pass/fail with details)
- ✅ No Claude reasoning required
- ✅ Stateless

**Verdict**: ✅ Excellent CLI candidate

### Generated Bash CLI

```bash
#!/usr/bin/env bash
# pre-commit-validate.sh - Validate code before git commits
set -euo pipefail

readonly VERSION="1.0.0"
readonly SCRIPT_NAME="pre-commit-validate"

# Colors
if [[ -t 1 ]]; then
    readonly RED='\033[0;31m'
    readonly GREEN='\033[0;32m'
    readonly YELLOW='\033[0;33m'
    readonly NC='\033[0m'
else
    readonly RED='' GREEN='' YELLOW='' NC=''
fi

: "${OUTPUT_FORMAT:=text}"
: "${MAJOR_THRESHOLD:=10}"

show_help() {
    cat << 'EOF'
pre-commit-validate - Validate code quality before git commits

USAGE:
    pre-commit-validate [command] [options]

COMMANDS:
    run             Run all applicable validations (default)
    python          Run Python validation only (ruff + mypy)
    typescript      Run TypeScript validation only (lint + type-check)
    status          Show what would be validated
    help            Show this help

OPTIONS:
    -m, --major     Force comprehensive validation (include implementation check)
    -o, --output    Output format: text (default) | json
    -q, --quiet     Suppress progress messages
    -h, --help      Show help

EXIT CODES:
    0   All validations passed
    1   Validation failed
    2   Invalid arguments
    3   Missing dependencies
    4   No files staged
EOF
}

# Check for staged files
get_staged_files() {
    git diff --cached --name-only 2>/dev/null || echo ""
}

get_python_files() {
    get_staged_files | grep '\.py$' || true
}

get_typescript_files() {
    get_staged_files | grep -E '\.(ts|tsx|js|jsx)$' || true
}

# Validation functions
validate_python() {
    local files
    files=$(get_python_files)

    if [[ -z "$files" ]]; then
        echo '{"python": {"status": "skipped", "reason": "no files"}}'
        return 0
    fi

    local ruff_result=0
    local mypy_result=0
    local ruff_output=""
    local mypy_output=""

    # Run ruff
    if command -v ruff &>/dev/null; then
        ruff_output=$(ruff check . 2>&1) || ruff_result=$?
    else
        echo -e "${YELLOW}WARN: ruff not found, skipping${NC}" >&2
    fi

    # Run mypy
    if command -v mypy &>/dev/null; then
        mypy_output=$(mypy . 2>&1) || mypy_result=$?
    else
        echo -e "${YELLOW}WARN: mypy not found, skipping${NC}" >&2
    fi

    if [[ "$OUTPUT_FORMAT" == "json" ]]; then
        jq -n \
            --argjson ruff_code "$ruff_result" \
            --arg ruff_output "$ruff_output" \
            --argjson mypy_code "$mypy_result" \
            --arg mypy_output "$mypy_output" \
            '{
                python: {
                    ruff: {passed: ($ruff_code == 0), exit_code: $ruff_code, output: $ruff_output},
                    mypy: {passed: ($mypy_code == 0), exit_code: $mypy_code, output: $mypy_output}
                }
            }'
    else
        if [[ $ruff_result -eq 0 ]]; then
            echo -e "${GREEN}✓ Ruff: PASSED${NC}"
        else
            echo -e "${RED}✗ Ruff: FAILED${NC}"
            echo "$ruff_output"
        fi

        if [[ $mypy_result -eq 0 ]]; then
            echo -e "${GREEN}✓ Mypy: PASSED${NC}"
        else
            echo -e "${RED}✗ Mypy: FAILED${NC}"
            echo "$mypy_output"
        fi
    fi

    [[ $ruff_result -eq 0 && $mypy_result -eq 0 ]]
}

validate_typescript() {
    local files
    files=$(get_typescript_files)

    if [[ -z "$files" ]]; then
        echo '{"typescript": {"status": "skipped", "reason": "no files"}}'
        return 0
    fi

    local lint_result=0
    local tsc_result=0
    local lint_output=""
    local tsc_output=""

    # Run lint
    if [[ -f package.json ]] && grep -q '"lint"' package.json; then
        lint_output=$(npm run lint 2>&1) || lint_result=$?
    else
        echo -e "${YELLOW}WARN: No lint script found${NC}" >&2
    fi

    # Run type-check
    if [[ -f package.json ]] && grep -q '"type-check"' package.json; then
        tsc_output=$(npm run type-check 2>&1) || tsc_result=$?
    elif command -v tsc &>/dev/null || npx tsc --version &>/dev/null; then
        tsc_output=$(npx tsc --noEmit 2>&1) || tsc_result=$?
    else
        echo -e "${YELLOW}WARN: TypeScript not found${NC}" >&2
    fi

    if [[ "$OUTPUT_FORMAT" == "json" ]]; then
        jq -n \
            --argjson lint_code "$lint_result" \
            --arg lint_output "$lint_output" \
            --argjson tsc_code "$tsc_result" \
            --arg tsc_output "$tsc_output" \
            '{
                typescript: {
                    lint: {passed: ($lint_code == 0), exit_code: $lint_code, output: $lint_output},
                    typecheck: {passed: ($tsc_code == 0), exit_code: $tsc_code, output: $tsc_output}
                }
            }'
    else
        if [[ $lint_result -eq 0 ]]; then
            echo -e "${GREEN}✓ Lint: PASSED${NC}"
        else
            echo -e "${RED}✗ Lint: FAILED${NC}"
            echo "$lint_output"
        fi

        if [[ $tsc_result -eq 0 ]]; then
            echo -e "${GREEN}✓ Type-check: PASSED${NC}"
        else
            echo -e "${RED}✗ Type-check: FAILED${NC}"
            echo "$tsc_output"
        fi
    fi

    [[ $lint_result -eq 0 && $tsc_result -eq 0 ]]
}

cmd_status() {
    local staged_count py_count ts_count
    staged_count=$(get_staged_files | wc -l)
    py_count=$(get_python_files | wc -l)
    ts_count=$(get_typescript_files | wc -l)

    local is_major="false"
    [[ $staged_count -ge $MAJOR_THRESHOLD ]] && is_major="true"

    if [[ "$OUTPUT_FORMAT" == "json" ]]; then
        jq -n \
            --argjson total "$staged_count" \
            --argjson python "$py_count" \
            --argjson typescript "$ts_count" \
            --argjson major "$is_major" \
            '{staged_files: $total, python_files: $python, typescript_files: $typescript, is_major_commit: $major}'
    else
        echo "Staged files: $staged_count"
        echo "  Python: $py_count"
        echo "  TypeScript: $ts_count"
        echo "  Major commit: $is_major"
    fi
}

cmd_run() {
    local force_major=false
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -m|--major) force_major=true; shift ;;
            *) shift ;;
        esac
    done

    local staged_count
    staged_count=$(get_staged_files | wc -l)

    if [[ $staged_count -eq 0 ]]; then
        echo -e "${YELLOW}No files staged for commit${NC}" >&2
        exit 4
    fi

    local py_result=0
    local ts_result=0

    # Run Python validation if we have Python files
    if [[ -n "$(get_python_files)" ]]; then
        echo -e "${YELLOW}=== Python Validation ===${NC}"
        validate_python || py_result=$?
    fi

    # Run TypeScript validation if we have TypeScript files
    if [[ -n "$(get_typescript_files)" ]]; then
        echo -e "${YELLOW}=== TypeScript Validation ===${NC}"
        validate_typescript || ts_result=$?
    fi

    # Check if major commit
    if [[ $staged_count -ge $MAJOR_THRESHOLD ]] || [[ "$force_major" == "true" ]]; then
        echo -e "${YELLOW}=== Major Commit: Implementation Check ===${NC}"
        echo "Checking for incomplete implementations..."
        # This would integrate with verifying-implementation
        # For CLI, we can grep for TODO/FIXME
        local todos
        todos=$(get_staged_files | xargs grep -l -E '(TODO|FIXME|NotImplementedError)' 2>/dev/null | wc -l)
        if [[ $todos -gt 0 ]]; then
            echo -e "${YELLOW}⚠ Found $todos files with incomplete markers${NC}"
        else
            echo -e "${GREEN}✓ No incomplete markers found${NC}"
        fi
    fi

    # Final result
    if [[ $py_result -eq 0 && $ts_result -eq 0 ]]; then
        echo -e "\n${GREEN}✓ All validations passed - ready to commit${NC}"
        exit 0
    else
        echo -e "\n${RED}✗ Validation failed - fix errors before committing${NC}"
        exit 1
    fi
}

main() {
    local cmd="${1:-run}"
    shift || true

    while [[ $# -gt 0 ]]; do
        case "$1" in
            -o|--output) OUTPUT_FORMAT="${2:-text}"; shift 2 ;;
            -m|--major) FORCE_MAJOR=true; shift ;;
            -h|--help) show_help; exit 0 ;;
            *) shift ;;
        esac
    done

    case "$cmd" in
        run)        cmd_run "$@" ;;
        python)     validate_python ;;
        typescript) validate_typescript ;;
        status)     cmd_status ;;
        help)       show_help ;;
        *)          echo "Unknown command: $cmd" >&2; exit 2 ;;
    esac
}

main "$@"
```

---

## Example 2: fhb:update-readme → Hybrid Approach

### Source Analysis

**Source**: `command:fhb:update-readme`

**What it does**:
1. Reads current README.md
2. Analyzes git history and recent changes
3. **Generates** updated README content (requires Claude)
4. Writes updated README.md

**Assessment**:
- ✅ Git/file analysis is deterministic
- ❌ Content generation requires natural language ability
- ⚠️ Partial candidate

**Verdict**: ⚠️ Hybrid - CLI for data gathering, Claude for content

### Generated Hybrid CLI

```bash
#!/usr/bin/env bash
# update-readme-data.sh - Gather data for README updates
# Use with Claude for actual content generation
set -euo pipefail

readonly VERSION="1.0.0"

show_help() {
    cat << 'EOF'
update-readme-data - Gather git/project data for README updates

USAGE:
    update-readme-data [command] [options]

COMMANDS:
    gather          Gather all data as JSON (default)
    git-history     Get recent git history
    changes         Get file changes summary
    structure       Get project structure
    help            Show help

This is a DATA GATHERING tool. Use the output with Claude to generate
actual README content, or pipe directly to Claude CLI:

    update-readme-data gather | claude "Update README based on this data"

OUTPUT:
    JSON object with all gathered data suitable for Claude processing
EOF
}

gather_git_history() {
    local history commits_json

    # Recent commits
    commits_json=$(git log --oneline -10 --format='{"hash":"%h","message":"%s","date":"%ci"}' 2>/dev/null | jq -s '.')

    # Changed files in last week
    local changed_files
    changed_files=$(git log --since="1 week ago" --name-only --pretty=format: | sort -u | grep -v '^$' | jq -R -s 'split("\n") | map(select(. != ""))')

    jq -n \
        --argjson commits "$commits_json" \
        --argjson changed "$changed_files" \
        '{recent_commits: $commits, changed_files_week: $changed}'
}

gather_changes() {
    local added deleted modified

    added=$(git diff --name-status HEAD~10 2>/dev/null | grep "^A" | cut -f2 | jq -R -s 'split("\n") | map(select(. != ""))')
    deleted=$(git diff --name-status HEAD~10 2>/dev/null | grep "^D" | cut -f2 | jq -R -s 'split("\n") | map(select(. != ""))')
    modified=$(git diff --name-status HEAD~10 2>/dev/null | grep "^M" | cut -f2 | jq -R -s 'split("\n") | map(select(. != ""))')

    jq -n \
        --argjson added "$added" \
        --argjson deleted "$deleted" \
        --argjson modified "$modified" \
        '{added: $added, deleted: $deleted, modified: $modified}'
}

gather_structure() {
    local docs_files src_structure

    docs_files=$(find . -name "*.md" -not -path "./node_modules/*" -not -path "./.git/*" 2>/dev/null | jq -R -s 'split("\n") | map(select(. != ""))')

    # Get top-level structure
    src_structure=$(ls -1 2>/dev/null | jq -R -s 'split("\n") | map(select(. != ""))')

    jq -n \
        --argjson docs "$docs_files" \
        --argjson structure "$src_structure" \
        '{documentation_files: $docs, top_level_structure: $structure}'
}

cmd_gather() {
    local readme_content=""
    if [[ -f README.md ]]; then
        readme_content=$(cat README.md)
    fi

    local git_data changes structure
    git_data=$(gather_git_history)
    changes=$(gather_changes)
    structure=$(gather_structure)

    jq -n \
        --arg readme "$readme_content" \
        --argjson git "$git_data" \
        --argjson changes "$changes" \
        --argjson structure "$structure" \
        '{
            current_readme: $readme,
            git: $git,
            file_changes: $changes,
            project: $structure,
            instructions: "Use this data to update the README.md. Preserve important existing content while incorporating recent changes."
        }'
}

main() {
    local cmd="${1:-gather}"
    shift || true

    case "$cmd" in
        gather)       cmd_gather ;;
        git-history)  gather_git_history ;;
        changes)      gather_changes ;;
        structure)    gather_structure ;;
        help|-h)      show_help ;;
        *)            echo "Unknown: $cmd" >&2; exit 2 ;;
    esac
}

main "$@"
```

**Usage with Claude**:
```bash
# Gather data and pass to Claude
./update-readme-data.sh gather > readme-context.json
claude --print "Based on this context, update README.md: $(cat readme-context.json)"

# Or pipe directly (if claude CLI supports stdin)
./update-readme-data.sh gather | claude "Update README based on this JSON data"
```

---

## Example 3: code-reviewer → Push Back

### Source Analysis

**Source**: `agent:code-reviewer`

**What it does**:
1. Reads codebase files
2. **Understands** code semantics and intent
3. **Judges** quality, security, architecture
4. **Generates** review comments with suggestions
5. Adapts feedback to context

**Assessment**:
- ✅ File reading is deterministic
- ❌ Semantic understanding requires LLM
- ❌ Quality judgment is subjective
- ❌ Comment generation needs language ability
- ❌ Context-aware suggestions require reasoning

**Verdict**: ❌ Not suitable for CLI conversion

### Push Back Response

```markdown
## CLI Conversion Not Recommended

**Source**: `agent:code-reviewer`

### Why This Should Remain a Claude Agent

The code-reviewer agent fundamentally requires Claude's ability to:

1. **Understand code semantics**: Not just syntax, but what the code is trying to do
2. **Apply judgment**: Determine what's "good" vs "problematic" based on context
3. **Generate explanations**: Produce human-readable review comments
4. **Consider trade-offs**: Balance between different valid approaches
5. **Adapt to codebase**: Recognize project-specific patterns and conventions

A CLI version could only:
- Run static analysis tools (already exists: eslint, ruff, mypy)
- Check for hardcoded patterns (grep)
- Generate reports from tool output

This would be a significantly inferior experience compared to the agent.

### Alternative Approaches

If you need code review automation:

1. **Use existing linters as CLI** (already CLI-native):
   ```bash
   ruff check . && mypy . && npm run lint
   ```

2. **Wrapper script for multiple tools**:
   ```bash
   #!/bin/bash
   echo "=== Running static analysis ==="
   ruff check .
   npm run lint
   echo "=== Security scan ==="
   npm audit
   ```

3. **Keep using the Claude agent** for semantic review:
   ```bash
   # In automation (if really needed):
   claude --dangerously-skip-permissions "Review the code in src/"
   ```

4. **Hybrid approach**: CLI for data gathering, agent for review:
   ```bash
   # Gather code metrics
   ./code-metrics.sh > metrics.json

   # Agent reviews with context
   claude "Review this code given these metrics: $(cat metrics.json)"
   ```

### Recommendation

Keep `code-reviewer` as a Claude Code agent. The value is in Claude's understanding, not in automation. For automated checks, use the existing excellent CLI tools (eslint, prettier, ruff, mypy, etc.) which are already designed as CLIs.
```

---

## Example 4: verifying-implementation → CLI

### Source Analysis

**Source**: `skill:verifying-implementation`

**What it does**:
1. Scans codebase for TODO/FIXME markers
2. Finds NotImplementedError exceptions
3. Detects empty function bodies
4. Finds placeholder implementations
5. Reports findings as structured list

**Assessment**:
- ✅ All pattern matching (grep/ast parsing)
- ✅ Deterministic output
- ✅ No judgment required
- ✅ Structured output possible

**Verdict**: ✅ Excellent CLI candidate

### Generated CLI

```bash
#!/usr/bin/env bash
# verify-implementation.sh - Find incomplete implementations
set -euo pipefail

show_help() {
    cat << 'EOF'
verify-implementation - Scan for incomplete implementations

USAGE:
    verify-implementation [scope] [options]

ARGUMENTS:
    scope           Directory to scan (default: .)

OPTIONS:
    -o, --output    Output: json (default) | text | count
    -t, --type      Filter: all | todo | error | empty | placeholder
    -h, --help      Show help

EXIT CODES:
    0   No incomplete implementations found
    1   Incomplete implementations found
    2   Invalid arguments
EOF
}

: "${OUTPUT_FORMAT:=json}"
: "${FILTER_TYPE:=all}"

find_todos() {
    local scope="${1:-.}"
    grep -rn --include="*.py" --include="*.ts" --include="*.js" --include="*.tsx" --include="*.jsx" \
        -E '(TODO|FIXME|XXX|HACK):?' "$scope" 2>/dev/null | \
        jq -R 'split(":") | {file: .[0], line: .[1], content: (.[2:] | join(":"))}' | jq -s '.'
}

find_not_implemented() {
    local scope="${1:-.}"
    grep -rn --include="*.py" 'NotImplementedError' "$scope" 2>/dev/null | \
        jq -R 'split(":") | {file: .[0], line: .[1], content: (.[2:] | join(":"))}' | jq -s '.'
}

find_empty_functions() {
    local scope="${1:-.}"
    # Python: def func(): pass
    grep -rn --include="*.py" -E 'def .+\(.*\):\s*$' "$scope" 2>/dev/null | \
        while read -r line; do
            file=$(echo "$line" | cut -d: -f1)
            lineno=$(echo "$line" | cut -d: -f2)
            next_line=$(sed -n "$((lineno+1))p" "$file" 2>/dev/null | tr -d '[:space:]')
            if [[ "$next_line" == "pass" ]] || [[ "$next_line" == '"""' ]] || [[ -z "$next_line" ]]; then
                echo "$line"
            fi
        done | jq -R 'split(":") | {file: .[0], line: .[1], content: "empty function body"}' | jq -s '.'
}

find_placeholders() {
    local scope="${1:-.}"
    grep -rn --include="*.py" --include="*.ts" --include="*.js" \
        -E '(raise NotImplementedError|throw new Error.*not implemented|return null.*todo|# placeholder|// placeholder)' \
        "$scope" 2>/dev/null | \
        jq -R 'split(":") | {file: .[0], line: .[1], content: (.[2:] | join(":"))}' | jq -s '.'
}

cmd_scan() {
    local scope="${1:-.}"
    local todos empty errors placeholders

    todos=$(find_todos "$scope")
    errors=$(find_not_implemented "$scope")
    empty=$(find_empty_functions "$scope")
    placeholders=$(find_placeholders "$scope")

    local total
    total=$(echo "$todos $errors $empty $placeholders" | jq -s 'add | length')

    if [[ "$OUTPUT_FORMAT" == "json" ]]; then
        jq -n \
            --argjson todos "$todos" \
            --argjson errors "$errors" \
            --argjson empty "$empty" \
            --argjson placeholders "$placeholders" \
            --argjson total "$total" \
            '{
                summary: {total: $total, todos: ($todos | length), errors: ($errors | length), empty: ($empty | length), placeholders: ($placeholders | length)},
                details: {todos: $todos, not_implemented: $errors, empty_functions: $empty, placeholders: $placeholders}
            }'
    elif [[ "$OUTPUT_FORMAT" == "count" ]]; then
        echo "$total"
    else
        echo "=== Incomplete Implementation Report ==="
        echo "TODOs: $(echo "$todos" | jq 'length')"
        echo "NotImplementedError: $(echo "$errors" | jq 'length')"
        echo "Empty functions: $(echo "$empty" | jq 'length')"
        echo "Placeholders: $(echo "$placeholders" | jq 'length')"
        echo "---"
        echo "Total issues: $total"
    fi

    [[ $total -eq 0 ]]
}

main() {
    local scope="."

    while [[ $# -gt 0 ]]; do
        case "$1" in
            -o|--output) OUTPUT_FORMAT="${2:-json}"; shift 2 ;;
            -t|--type) FILTER_TYPE="${2:-all}"; shift 2 ;;
            -h|--help) show_help; exit 0 ;;
            -*) echo "Unknown option: $1" >&2; exit 2 ;;
            *) scope="$1"; shift ;;
        esac
    done

    cmd_scan "$scope"
}

main "$@"
```

---

## Summary: Conversion Decision Tree

```
Is the capability...

1. Pure file/git operations?
   → ✅ CLI candidate (Example: verify-implementation)

2. Calling APIs with structured responses?
   → ✅ CLI candidate

3. Running existing CLI tools?
   → ✅ CLI candidate (Example: pre-commit-validate)

4. Pattern matching with well-defined rules?
   → ✅ CLI candidate

5. Data gathering for later Claude processing?
   → ⚠️ Hybrid: CLI gathers, Claude processes (Example: update-readme)

6. Requires understanding code semantics?
   → ❌ Keep as Claude agent

7. Makes judgment calls?
   → ❌ Keep as Claude agent (Example: code-reviewer)

8. Generates natural language?
   → ❌ Keep as Claude agent (unless using Claude API)

9. Adapts to context?
   → ❌ Keep as Claude agent
```
