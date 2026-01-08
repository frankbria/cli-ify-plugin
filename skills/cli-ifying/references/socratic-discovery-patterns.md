# Socratic Discovery Patterns for CLI Design

This document provides question templates and patterns for the Socratic discovery process when creating CLIs from natural language prompts.

## The Core Principle

> Don't just build what the user asked for. Build what they actually need, scoped to what a CLI can do well.

Before writing any code, engage in focused discovery to ensure:
1. The scope is atomic and CLI-appropriate
2. Inputs and outputs are well-defined
3. Edge cases are considered
4. The user understands trade-offs

## The "One Sentence" Test

A well-scoped CLI should pass this test:

> Can you describe what this CLI does in one clear sentence without using "and" or "then"?

**Good examples**:
- "Pull latest changes for all git repositories in a directory"
- "Find all node_modules folders and report their total size"
- "Convert images to WebP format with specified quality"

**Bad examples** (too broad):
- "Sync git repos and clean up stale branches and update dependencies"
- "Process images: resize, convert, and upload to S3"

If the description requires "and" or "then", suggest breaking it into multiple CLIs.

## Question Categories

### 1. Scope Questions (Always Ask)

**Goal**: Ensure atomic scope, prevent feature creep

```
"Your description mentions [X and Y]. For Unix-style tools, each CLI should do one thing well. Should we:
(a) Focus on just [X], or
(b) Focus on just [Y], or
(c) Create separate CLIs for each?"
```

```
"This could be broken into smaller tools:
1. [tool-1]: [single responsibility]
2. [tool-2]: [single responsibility]
Which is highest priority, or would you prefer a composed script?"
```

```
"Should this CLI [do everything automatically] or [provide building blocks for scripting]? Unix philosophy favors building blocks."
```

### 2. Input Questions

**Goal**: Define exactly what the CLI accepts

```
"What's the primary input source:
(a) Command-line argument (e.g., `cli path/to/file`)
(b) Stdin for piping (e.g., `cat file | cli`)
(c) Both (detect automatically)?"
```

```
"Should the [path/file] argument be:
(a) Required
(b) Optional with default (e.g., current directory)
(c) Read from config file if not provided?"
```

```
"For multiple inputs, should the CLI:
(a) Accept multiple arguments (`cli file1 file2 file3`)
(b) Accept a glob pattern (`cli '*.txt'`)
(c) Read file list from stdin?"
```

### 3. Output Questions

**Goal**: Define what the CLI produces

```
"Should output be:
(a) Human-readable text (progress, status messages)
(b) JSON for piping to other tools
(c) Both (detect if output is TTY vs pipe)?"
```

```
"Should this CLI:
(a) Modify files in place
(b) Output to stdout (let user redirect)
(c) Create new files with different names/extensions?"
```

```
"For progress, should it:
(a) Show real-time progress per item
(b) Be quiet and only show summary at end
(c) User's choice via --quiet/--verbose flags?"
```

### 4. Behavior Questions

**Goal**: Handle edge cases and errors

```
"For [error condition], should the CLI:
(a) Stop immediately with error
(b) Skip the problematic item and continue
(c) User's choice via --strict/--lenient flag?"
```

```
"For destructive operations, should it:
(a) Always require confirmation
(b) Provide --dry-run and --force flags
(c) Just proceed (trust the user)?"
```

```
"If [resource] is not found, should the CLI:
(a) Exit with error code
(b) Create it automatically
(c) Ask the user?"
```

### 5. Environment Questions

**Goal**: Handle configuration and secrets

```
"Does this need authentication or API keys? If so, should they come from:
(a) Environment variables (recommended)
(b) Config file
(c) Command-line flags (less secure)?"
```

```
"Should this work on:
(a) Linux/Mac only (Bash)
(b) Windows only (PowerShell)
(c) Both platforms?"
```

## Breakdown Patterns

When a request is too broad, suggest these breakdown patterns:

### Data Flow Pattern
```
1. [producer]: Generates/finds the data
2. [processor]: Transforms the data
3. [consumer]: Outputs/saves the data

These compose: producer | processor | consumer
```

### CRUD Pattern
```
1. [list-thing]: Find all things
2. [get-thing]: Get details of one thing
3. [update-thing]: Modify one thing
4. [delete-thing]: Remove one thing

Main workflow composes these primitives.
```

### Pipeline Pattern
```
1. [discover]: Find targets
2. [analyze]: Examine targets
3. [act]: Perform operation
4. [report]: Summarize results

User can run individually or compose.
```

## Response Templates

### When scope is good:
```markdown
This looks well-scoped for a CLI. A few clarifying questions before we proceed:

1. [Question about input]
2. [Question about output]
3. [Question about behavior]
```

### When scope is too broad:
```markdown
This includes several distinct operations. For maintainable, composable tools, I suggest breaking this into:

1. **[tool-1]**: [one-sentence description]
2. **[tool-2]**: [one-sentence description]

These can be composed: `tool-1 | tool-2` or `tool-1 && tool-2`

Which should we build first, or would you prefer a single script that calls both?
```

### When scope requires Claude:
```markdown
Part of this requires [judgment/understanding/generation] that a standalone CLI can't provide:

**CLI-suitable parts**:
- [list of things a script can do]

**Requires Claude**:
- [list of things needing AI]

I recommend creating a CLI for the first part, and keeping the second part as a Claude skill/command that the CLI can optionally invoke.
```

## Example Discovery Sessions

### Example 1: File Conversion

**User**: "CLI to convert images to WebP"

**Questions**:
1. Should this handle all image formats (png, jpg, gif, etc.) or just specific ones?
2. Quality/compression: fixed value, or user-specified via flag?
3. Output: same directory with new extension, or specified output directory?
4. Existing WebP files: skip, overwrite, or error?

**Resulting spec**:
```
img2webp [files...] --quality 80 --output-dir ./webp --skip-existing
```

### Example 2: Git Operations

**User**: "Sync all my git repos"

**Questions**:
1. "Sync" = just `git pull`, or also fetch, push, branch cleanup?
2. Repos with uncommitted changes: skip, stash+pull+unstash, or fail?
3. Target: directory argument, or scan from current dir?
4. Output: progress per repo, or quiet until done?

**Resulting spec**:
```
git-sync-repos [dir] --include-dirty --json
```

### Example 3: Too Broad Request

**User**: "CLI to manage my project - create issues, track time, generate reports"

**Response**:
```markdown
This combines several distinct capabilities. For composable tools, I suggest:

1. **issue-create**: Create a new issue from template
2. **time-track**: Log time against an issue
3. **time-report**: Generate time reports

Each is useful standalone:
- `issue-create --template bug "Title here"`
- `time-track --issue 123 --hours 2 "Fixed the bug"`
- `time-report --week --json | jq '.total_hours'`

Which would be most valuable first?
```

## Anti-Patterns to Avoid

### Don't ask about:
- Implementation details the user shouldn't care about
- Edge cases that are unlikely to occur
- Options that have obvious sensible defaults

### Don't create CLIs that:
- Require interactive input during operation
- Have more than 5-6 flags (probably too complex)
- Try to be "smart" about user intent
- Hide errors or fail silently

### Don't:
- Ask too many questions (2-4 is ideal)
- Make the user repeat themselves
- Suggest overly complex solutions
- Ignore the Unix philosophy

## Summary Checklist

Before proceeding to code:

- [ ] Can the CLI be described in one sentence?
- [ ] Are all inputs defined (args, options, stdin)?
- [ ] Are all outputs defined (stdout, files, exit codes)?
- [ ] Are error cases handled?
- [ ] Is the scope appropriately atomic?
- [ ] Does the user understand what it will/won't do?
