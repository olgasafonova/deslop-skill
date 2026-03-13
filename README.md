# deslop

Zero-dependency code health scanner for AI coding agents. Checks file sizes, test coverage gaps, secret leaks, TODO debt, dependency health, and structural issues using only the agent's built-in tools.

No pip install. No npm. No external CLI. Just copy one file.

## What It Does

Six mechanical detectors scan your codebase and produce a 0-100 health score:

| Detector | What it catches |
|----------|----------------|
| **File health** | Files over 500 LOC, directories with too many files |
| **Test coverage** | Source files without matching test files |
| **Secret detection** | Hardcoded API keys, passwords, private keys |
| **TODO debt** | Accumulated TODO, FIXME, HACK, XXX comments |
| **Dependency health** | Missing lockfiles, unpinned versions, excessive deps |
| **Structural quality** | God packages, orphan files, deep nesting, naming inconsistency |

Works on **Go, Python, TypeScript, JavaScript, Rust, Java, and C#** projects.

## Install

### Claude Code

```bash
claude skill add --url https://github.com/olgasafonova/deslop-skill
```

### Manual (any agent)

Copy `SKILL.md` to your skills directory:

```bash
# Claude Code
cp SKILL.md ~/.claude/skills/deslop/SKILL.md

# Cursor
cp SKILL.md .cursor/skills/deslop/SKILL.md

# Or just drop SKILL.md in whatever directory your agent reads skills from
```

## Usage

Talk to your AI agent:

```
"deslop this repo"
"run a code quality scan"
"check code health"
"scan all my projects"        # portfolio mode
```

### Example Output

```
## Code Health: my-project

| Detector           | Score  | Issues | Top Finding                |
|--------------------|--------|--------|----------------------------|
| File health        |  80.0% | 3      | client.go (1,083 LOC)      |
| Test coverage      |  75.0% | 5      | cache.go has no test file   |
| Secret detection   | 100.0% | 0      | Clean                      |
| TODO debt          |  90.0% | 5      | 5 TODOs across 3 files     |
| Dependency health  |  95.0% | 1      | 1 unpinned dependency      |
| Structural quality |  90.0% | 1      | handlers/ has 18 files     |

Overall: 88.3/100
```

### Portfolio Mode

Point it at a directory of repos and get a comparative dashboard:

```
"deslop all repos in ~/Projects"
```

Generates an HTML dashboard at `/tmp/deslop-portfolio-dashboard.html` with per-repo scores, dimension breakdowns, and a summary table sorted worst-first.

## Score Ranges

| Range | Meaning |
|-------|---------|
| 90-100 | Clean codebase |
| 75-89 | Good shape; minor issues |
| 60-74 | Needs attention |
| <60 | Significant technical debt |

## How It Works

The skill instructs Claude (or any compatible agent) to use its built-in file and search tools:

- **Glob** to discover source and test files
- **Grep** to search for secrets, TODOs, and import patterns
- **Bash** to count lines and check dependency files
- **Read** to inspect flagged files

No code runs outside the agent's standard toolset. The skill is a SKILL.md file: a structured prompt that tells the agent what to check and how to score it.

## False Positive Handling

The skill includes guidance for common false positives:

- OAuth endpoint URLs flagged as "secrets"
- Environment variable name constants (e.g., `envSecretKey = "MY_SECRET_KEY"`)
- `main.go` or CLI entry points flagged as "untested"
- Large definition files (tool specs, route tables) flagged for file size

The agent reads flagged lines before reporting them and applies exclusion rules.

## Requirements

- An AI coding agent with file read, search, and shell access (Claude Code, Cursor, Windsurf, Codex CLI)
- That's it

## Credits

Inspired by [desloppify](https://github.com/peteromallet/desloppify) by [Pete O'Mallet](https://github.com/peteromallet), which combines mechanical linting with LLM-driven subjective review. This skill reimplements the mechanical detection layer as a zero-dependency agent skill.

## License

MIT
