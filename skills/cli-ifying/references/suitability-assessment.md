# CLI Suitability Assessment Framework

This document provides detailed criteria for determining whether a Claude Code capability should be converted to a standalone CLI.

## The Core Question

> Can this capability be implemented as a deterministic script that produces predictable output from given inputs, without requiring Claude's reasoning, knowledge, or conversational ability?

## Detailed Assessment Criteria

### Dimension 1: Input/Output Model

**CLI-Suitable** ✅
- Inputs can be provided via command-line arguments
- Inputs can be read from files or stdin
- Outputs can be written to stdout or files
- Output format is structured (JSON, YAML, plain text)

**Not CLI-Suitable** ❌
- Requires multi-turn conversation to gather inputs
- Needs clarification questions based on partial input
- Output is natural language that requires generation
- Context from prior interactions affects output

### Dimension 2: Processing Logic

**CLI-Suitable** ✅
- Deterministic transformations (same input → same output)
- Pattern matching with well-defined rules
- Calling external tools/APIs with structured responses
- File manipulation with clear rules
- Status checks and validation

**Not CLI-Suitable** ❌
- Requires semantic understanding of code
- Makes judgment calls based on context
- Synthesizes information from multiple sources
- Generates creative or explanatory content
- Adapts approach based on discovered information

### Dimension 3: Tool Dependencies

**CLI-Suitable** ✅
- Uses standard Unix tools (grep, sed, awk, jq)
- Calls well-documented APIs with structured responses
- Runs other CLI tools and parses output
- Performs file system operations

**Not CLI-Suitable** ❌
- Requires LLM inference
- Needs natural language understanding
- Depends on Claude's knowledge base
- Uses Claude's ability to understand code semantics

### Dimension 4: State Requirements

**CLI-Suitable** ✅
- Stateless - each run is independent
- State stored in files between runs (if needed)
- Configuration via environment variables
- No memory of previous invocations

**Not CLI-Suitable** ❌
- Requires conversation context
- Builds understanding over time
- References prior interactions
- Maintains session state

## Quick Assessment Checklist

Rate each criterion. If 3+ are "No", the capability is likely not CLI-suitable.

```
□ Can all inputs be provided as arguments or files?
□ Is the processing logic deterministic?
□ Can outputs be structured (JSON/text)?
□ Does it run without needing Claude's reasoning?
□ Is each invocation independent (stateless)?
□ Could a bash script theoretically do this?
□ Would the output be identical if run twice with same inputs?
```

## Capability Categories

### Definitely CLI-Suitable

| Category | Examples | Why |
|----------|----------|-----|
| Validation | Lint checks, type checking, format verification | Rules-based, deterministic |
| Git Operations | Status, diff, log analysis, commit | Wrapping existing CLIs |
| File Scanning | Find TODOs, count lines, search patterns | Pattern matching |
| API Calls | Fetch data, post updates, check status | Structured I/O |
| Build/Test | Run tests, build projects, check coverage | Wrapping existing tools |
| Configuration | Generate configs, validate settings | Template-based |

### Partially CLI-Suitable

| Category | CLI Part | Claude Part | Hybrid Approach |
|----------|----------|-------------|-----------------|
| Documentation Updates | Gather git/file data | Generate content | CLI gathers, Claude writes |
| Code Review (automated) | Run static analysis | Explain findings | CLI runs tools, Claude explains |
| Refactoring | Apply known transforms | Choose what to refactor | CLI applies, Claude decides |

### Definitely Not CLI-Suitable

| Category | Why |
|----------|-----|
| Code Review (semantic) | Requires understanding code intent and quality |
| Mentoring/Explanation | Requires knowledge and teaching ability |
| Architecture Design | Requires judgment and trade-off analysis |
| Bug Investigation | Requires reasoning about code behavior |
| Creative Tasks | Requires generation ability |
| Conversation-based | Requires multi-turn interaction |

## Hybrid Approach

For partially suitable capabilities, consider:

### Option 1: CLI + Claude Wrapper

```bash
#!/bin/bash
# CLI gathers data
git_data=$(git log --oneline -10)
file_list=$(find . -name "*.ts" -type f)

# Pass to Claude for processing
claude --print "Analyze this git history and suggest commit message: $git_data"
```

### Option 2: Data Gathering CLI + Manual Claude Step

```bash
# generate-context.sh
#!/bin/bash
# Gather all data needed for review
echo "=== Git Status ===" > context.txt
git status >> context.txt
echo "=== Recent Changes ===" >> context.txt
git diff HEAD~5 >> context.txt

# User then pastes context.txt into Claude
```

### Option 3: Structured Output for Claude Consumption

```bash
# analyze-codebase.sh
#!/bin/bash
# Output structured JSON that Claude can consume
jq -n \
    --arg files "$(find . -name '*.py' | wc -l)" \
    --arg todos "$(grep -r 'TODO' . | wc -l)" \
    --arg tests "$(find . -name '*_test.py' | wc -l)" \
    '{python_files: $files, todo_count: $todos, test_files: $tests}'
```

## Assessment Examples

### Example 1: fhb:update-readme

**What it does**: Analyzes git changes and updates README.md content

**Assessment**:
| Criterion | Rating | Notes |
|-----------|--------|-------|
| Inputs as args/files | ✅ | Git history, current README |
| Deterministic processing | ❌ | Content generation varies |
| Structured output | ⚠️ | Markdown, but creative |
| No Claude reasoning | ❌ | Needs language generation |
| Stateless | ✅ | Each run independent |
| Bash-scriptable | ⚠️ | Data gathering yes, generation no |

**Verdict**: Hybrid - CLI for data gathering, Claude for content

### Example 2: validating-pre-commit

**What it does**: Runs linters and type checkers on staged files

**Assessment**:
| Criterion | Rating | Notes |
|-----------|--------|-------|
| Inputs as args/files | ✅ | Staged files from git |
| Deterministic processing | ✅ | Same files → same results |
| Structured output | ✅ | Pass/fail with details |
| No Claude reasoning | ✅ | Just running tools |
| Stateless | ✅ | Each run independent |
| Bash-scriptable | ✅ | Absolutely |

**Verdict**: ✅ Excellent CLI candidate

### Example 3: code-reviewer agent

**What it does**: Reviews code for quality, security, architecture

**Assessment**:
| Criterion | Rating | Notes |
|-----------|--------|-------|
| Inputs as args/files | ✅ | File paths |
| Deterministic processing | ❌ | Judgment-based |
| Structured output | ⚠️ | Comments, but subjective |
| No Claude reasoning | ❌ | Core requirement |
| Stateless | ✅ | Each review independent |
| Bash-scriptable | ❌ | Requires understanding |

**Verdict**: ❌ Keep as Claude agent

## When to Push Back

If assessment shows the capability is not CLI-suitable, provide clear explanation:

```markdown
## CLI Conversion Not Recommended

**Source**: [name]

### Why This Should Remain a Claude Skill

This capability fundamentally requires Claude's ability to:
- [Specific capability 1]
- [Specific capability 2]

A CLI version would lack:
- [Missing capability 1]
- [Missing capability 2]

### Alternative Approaches

If you need automation, consider:

1. **Wrapper Script**: Create a script that invokes Claude with this skill
   ```bash
   claude --dangerously-skip-permissions "/skill-name"
   ```

2. **Partial CLI**: Extract just the [deterministic portion]:
   ```bash
   # This part can be a CLI
   ./gather-data.sh > data.json

   # This part needs Claude
   # Manually invoke skill with data.json
   ```

3. **Keep as Skill**: Continue using via Claude Code - it's designed for this
```

## Summary

The key insight is that CLI tools are for **automation of deterministic processes**, while Claude skills are for **intelligent assistance requiring reasoning**. Forcing a reasoning-required task into a CLI produces a inferior version that misses the point of both approaches.
