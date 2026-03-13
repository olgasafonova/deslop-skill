---
name: deslop
version: 2.0.0
description: Scan codebases for mechanical code health issues using only built-in tools. Checks file sizes, test coverage gaps, secret leaks, structural problems, and TODO debt. No external dependencies. Use when user says "deslop", "code quality scan", "code health", "portfolio health", or "scan this repo". Do NOT use for style linting, formatting, or subjective code review.
allowed-tools:
  - Read
  - Write
  - Glob
  - Grep
  - Bash
license: MIT
compatibility: Any OS. No external tools required. Works on Go, Python, TypeScript, JavaScript, Rust, Java, and C# projects.
metadata:
  author: olgasafonova
  version: 2.0.0
  category: code-quality
---

# Deslop - Zero-Dependency Code Health Scanner

Scan a codebase for mechanical quality issues using only Claude's built-in tools. No pip install, no external CLI, no language runtime required.

## What It Checks

Six detectors, each producing a 0-100% health score:

### 1. File Health (large files, flat directories)

Use Bash to count lines per file:

```bash
find . -name '*.go' -o -name '*.py' -o -name '*.ts' -o -name '*.js' -o -name '*.rs' -o -name '*.java' -o -name '*.cs' | grep -v vendor | grep -v node_modules | xargs wc -l | sort -rn | head -30
```

| Threshold | Severity |
|-----------|----------|
| >500 LOC | Warning: consider splitting |
| >800 LOC | High: likely has multiple responsibilities |
| >1200 LOC | Critical: split this file |

For flat directories, count files per directory. Flag any directory with >15 source files.

**Score:** 100% minus 5% per file over 500 LOC, minus 10% per file over 800 LOC.

### 2. Test Coverage Gaps

Use Glob to find source files, then check for matching test files.

**Go:** For each `foo.go`, check if `foo_test.go` exists. Exclude `main.go`, generated files, and test helpers.

**Python:** For each `foo.py`, check if `test_foo.py` or `foo_test.py` exists in the same directory or a `tests/` directory.

**TypeScript/JavaScript:** For each `foo.ts`, check if `foo.test.ts`, `foo.spec.ts`, or `__tests__/foo.ts` exists.

**Rust:** Check for `#[cfg(test)]` blocks in source files using Grep.

**Score:** (files with tests / total testable files) * 100.

### 3. Secret Detection

Use Grep to scan for common secret patterns. Skip vendor, node_modules, .git, and test fixtures.

```
# Patterns to search (case-insensitive where noted)
password\s*=\s*["'][^"']{8,}     # hardcoded passwords
(?i)(api[_-]?key|secret[_-]?key)\s*=\s*["'][^"']+   # API keys
sk-[a-zA-Z0-9]{20,}              # OpenAI keys
AKIA[0-9A-Z]{16}                 # AWS access keys
ghp_[a-zA-Z0-9]{36}              # GitHub tokens
-----BEGIN (RSA |EC )?PRIVATE KEY # Private keys
```

Exclude:
- Environment variable NAME declarations (e.g., `envSecretKey = "MY_SECRET"`)
- URL strings containing "token" or "secret" in path segments
- Comments with `#nosec`, `nolint`, or `NOSONAR`
- Test files with obvious fixture data

**Score:** 100% if clean, minus 25% per confirmed secret.

### 4. TODO/FIXME Debt

Use Grep to count TODO, FIXME, HACK, XXX, and TEMP comments:

```bash
grep -rn 'TODO\|FIXME\|HACK\|XXX\|TEMP:' --include='*.go' --include='*.py' --include='*.ts' --include='*.js' . | grep -v vendor | grep -v node_modules
```

| Count | Severity |
|-------|----------|
| 0-5 | Healthy |
| 6-15 | Warning |
| 16-30 | High |
| >30 | Critical |

**Score:** 100% minus 2% per TODO item, floor at 0%.

### 5. Dependency Health

Check for outdated or risky dependency signals:

**Go:** Run `go mod tidy -diff` (reports needed changes). Count indirect dependencies in go.sum.

**Node:** Check if `package-lock.json` exists. Look for `"deprecated"` warnings. Count dependencies in package.json.

**Python:** Check if `requirements.txt` or `pyproject.toml` pins versions.

**Score:** 100% if clean, minus points for unpinned deps, missing lockfile, or excessive dependency count.

### 6. Structural Quality

Check for code organization signals:

- **God packages:** Any package/directory with >20 source files
- **Orphan files:** Source files not imported by anything (Go: use Grep for import paths)
- **Deep nesting:** Files more than 5 directories deep from project root
- **Naming consistency:** Mixed naming conventions in same directory (camelCase vs snake_case filenames)

**Score:** 100% minus 10% per structural issue.

## Modes

### Single Repo

1. Detect the primary language from project files (`go.mod`, `package.json`, `pyproject.toml`, `Cargo.toml`)
2. Run all six detectors
3. Calculate per-detector scores and overall score (average of all detectors)
4. Present results as a table
5. List top 5 actionable findings

### Portfolio Scan

When user says "deslop all" or "portfolio health":

1. Discover repos: scan `~/Projects/` (or user-specified directory) for directories containing project files
2. Run single-repo scan on each
3. Generate HTML dashboard at `/tmp/deslop-portfolio-dashboard.html`
4. Present summary table sorted by overall score

## Output

```
## Code Health: my-project

| Detector          | Score  | Issues | Top Finding                          |
|-------------------|--------|--------|--------------------------------------|
| File health       |  80.0% | 3      | client.go (1,083 LOC)                |
| Test coverage     |  75.0% | 5      | cache.go has no test file            |
| Secret detection  | 100.0% | 0      | Clean                                |
| TODO debt         |  90.0% | 5      | 5 TODOs across 3 files               |
| Dependency health |  95.0% | 1      | 1 unpinned dependency                |
| Structural quality|  90.0% | 1      | handlers/ has 18 files               |

Overall: 88.3/100
```

## Score Ranges

| Range | Meaning |
|-------|---------|
| 90-100 | Clean codebase |
| 75-89 | Good shape; minor issues |
| 60-74 | Needs attention |
| <60 | Significant technical debt |

## False Positive Guidance

Common false positives to suppress:

- **Secret detection** flags on OAuth endpoint URLs, env var name constants, and test fixtures. Always verify by reading the flagged line before reporting.
- **Test coverage** flags on `main.go`, CLI entry points, and generated code. These rarely need unit tests.
- **File health** flags on definition files (tool specs, route tables, type declarations). Data-heavy files are fine to be large.
- **TODO debt** in active feature branches is expected. Only flag on main/production branches.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Language not detected | Provide the language explicitly: "deslop this Go repo" |
| Too many false positives in secrets | Check if flagged lines are env var names or URLs, not actual secrets |
| `wc -l` errors on binary files | Add `--include` patterns to limit to source files |
| Portfolio scan too slow | Limit to specific repos: "deslop these 3 repos" |

## Credits

Inspired by [desloppify](https://github.com/peteromallet/desloppify) by Pete O'Mallet, which combines mechanical linting with LLM-driven subjective review. This skill reimplements the mechanical detection layer using only Claude's built-in tools, requiring zero external dependencies.
