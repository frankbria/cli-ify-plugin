# cli-ify

Convert Claude Code skills, commands, and agents into standalone CLI tools for Linux/Mac (Bash) and Windows (PowerShell).

## Why CLI-ify?

When building agent skills, there's tension between:
- **API-first design**: Great for programmatic use, hard to debug manually
- **GUI-first design**: Easy for humans, agents can't invoke them

**CLI-first design** solves this: a well-designed CLI is naturally dual-use—humans can invoke it from the terminal, and agents can invoke it via shell commands.

## Installation

### As a Claude Code Plugin

```bash
# Install the plugin
claude plugin add frankbria/cli-ify-plugin

# Or from local path
claude plugin add ~/projects/cli-ify-plugin
```

### Manual Installation

```bash
# Clone the repo
git clone https://github.com/frankbria/cli-ify-plugin.git

# Copy skill to your skills directory
cp -r cli-ify-plugin/skills/cli-ifying ~/.claude/skills/

# Optionally, add CLIs to your PATH
ln -s ~/cli-ify-plugin/bin/verify-impl ~/.local/bin/
```

## Usage

### Mode 1: Convert Existing Capabilities

Convert an existing skill, command, or agent to a CLI:

```bash
# In Claude Code, ask:
"CLI-ify the validating-pre-commit skill for Linux and Windows"

# Or use natural invocation:
"Convert the fhb:update-readme command to a standalone CLI"
```

### Mode 2: Create from Prompt

Create a new CLI from a natural language description:

```bash
# In Claude Code:
"CLI-ify: Create a CLI that syncs all git repos in a directory"

# Claude will ask Socratic questions to scope the CLI properly
```

## Philosophy

The skill follows Unix philosophy for CLI design:

- **Do one thing well**: Each CLI has a single, focused purpose
- **Composability**: JSON output, meaningful exit codes, works with pipes
- **Dual-use**: Same interface works for humans and AI agents
- **Debuggable**: Run manually to test, inspect output

### CLI Design Principles

Generated CLIs include:
- `--help` documentation
- Subcommands for operations
- JSON output option (for scripting)
- Meaningful exit codes (for `&&` chaining)
- Environment variable configuration
- TTY detection (pretty output for humans, JSON for pipes)

## Included CLIs

### verify-impl

Scan codebase for incomplete implementations before commits or PRs.

```bash
# Scan current directory
verify-impl

# Scan specific path
verify-impl src/

# Only critical issues (NotImplementedError, empty functions)
verify-impl --critical-only

# JSON output for CI
verify-impl --output json

# Gate commits
verify-impl && git commit -m "Feature complete"
```

**Exit codes:**
- `0`: Clean - ready to commit
- `1`: Warnings (TODOs, FIXMEs, skipped tests)
- `2`: Critical (NotImplementedError, fake implementations)
- `3`: Invalid arguments

## When NOT to CLI-ify

Some capabilities should remain as Claude skills:

| Keep as Skill | Why |
|---------------|-----|
| Code review | Requires semantic understanding |
| Explanation/mentoring | Requires language generation |
| Refactoring decisions | Requires judgment |
| Multi-turn workflows | Requires conversation |

The skill will push back and explain why when you try to CLI-ify something unsuitable.

## Contributing

1. Fork the repository
2. Create a feature branch
3. Add your CLI or improve existing ones
4. Submit a pull request

### Adding New CLIs

1. Create the script in `bin/`
2. Follow the design principles (help, exit codes, JSON output)
3. Update `plugin.json` with the new binary
4. Add documentation to this README

## License

MIT
