# Claude Code Overnight Skill

Autonomous overnight improvement engine for [Claude Code](https://claude.ai/code). Run `/overnight` before bed, wake up to a comprehensive QA report with auto-fixes.

## What It Does

11-phase full-stack QA that runs for hours unattended:

| Phase | Name | What It Checks |
|-------|------|----------------|
| 0 | Setup | Worktree creation, project detection, state init |
| 1a | Infrastructure Health | Processes, ports, disk, Docker, log analysis |
| 1b | DB Operations | Connection, migrations, orphaned records, data quality, slow queries |
| 1c | API Operations | All endpoints hit-tested, auth bypass, rate limits, load test, security headers |
| 2 | Code QA | 5-pass deep scan (syntax, patterns, expert review, test coverage, auto-fix) |
| 3 | Interactive QA | Dead click detection, form testing, toggle persistence, CRUD flows, user journeys |
| 4 | Design QA | Token audit, responsive (3 viewports), contrast/a11y, Figma parity, expert review |
| 5 | Performance QA | N+1 queries, bundle size, memory patterns, light load test |
| 6 | Dependency QA | CVEs, outdated packages, license compliance |
| 7 | Documentation QA | Stale docs, undocumented APIs, broken links |
| 8 | Fix & Refactor | Priority-sorted auto-fix with test gate (commit or revert) |
| 9 | E2E Regression | Post-fix re-test, auto-revert if broken |
| 10 | Learning & Report | Pattern recording, trend analysis, morning report, macOS notification |

## Key Features

- **Interactive UI Testing** via Puppeteer MCP -- clicks buttons, fills forms, tests toggles, runs CRUD flows
- **Design Quality Scoring** -- contrast ratios, design token compliance, responsive checks, Figma parity
- **Operational Testing** -- real API calls, DB integrity checks, security header validation
- **Self-Learning** -- records bug patterns to memory, builds trend data across runs
- **7-Layer Safety** -- worktree isolation, atomic fix-or-revert, file protection, size limits, severity gates
- **Never pushes or creates PRs** -- you review the branch in the morning

## Installation

Copy the `overnight/` directory to your Claude Code skills folder:

```bash
cp -r overnight/ ~/.claude/skills/overnight/
```

## Usage

```bash
/overnight                      # 6 hours, all phases
/overnight 480                  # 8 hours
/overnight --dry-run            # Scan only, no fixes
/overnight --focus security     # Security-focused preset
/overnight --focus design       # Design-focused preset
/overnight --focus ops          # Operations-focused preset
/overnight --focus performance  # Performance-focused preset
/overnight --phases 1a,1b,1c    # Specific phases only
/overnight --skip 3,4           # Skip specific phases
/overnight --resume             # Resume interrupted run
```

## Configuration

On first run, `config-default.json` is copied to `.overnight-state/config.json` in your project. Customize it for:

- Login credentials for interactive testing
- Page lists and user journeys
- CRUD test configurations
- Figma file keys for design parity
- Severity thresholds and safety limits
- Phase enable/disable toggles

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI or IDE extension
- **Puppeteer MCP** (for interactive QA, design QA, E2E regression)
- **Figma MCP** (optional, for design parity checks)
- Git repository (for worktree isolation)

## File Structure

```
overnight/
  SKILL.md                          # Main skill definition (737 lines)
  reference/
    phase-details.md                # Detailed procedures for Phase 3, 4, 9 (667 lines)
  templates/
    config-default.json             # Default configuration (186 lines)
```

## Safety Mechanisms

1. **Worktree Isolation** -- All work in a dedicated `chore/overnight-YYYY-MM-DD` branch
2. **Atomic Fix-or-Revert** -- Tests must pass before commit; failures are instantly reverted
3. **File Protection** -- Lock files, env files, migrations, CI/CD configs are never modified
4. **Change Size Limits** -- Max 50 lines per fix, 20 fixes per run, 30 commits total
5. **Severity Gate** -- Only CRITICAL/HIGH auto-fixed by default; MEDIUM reported only
6. **Phase Fault Tolerance** -- One phase failure doesn't stop the entire run
7. **Time Guards** -- Always reserves 30 minutes for final report generation

## Morning Report

After completion, find your report at `.overnight-state/morning-report.md` with:

- Summary table (issues found/fixed/remaining)
- Critical items requiring human judgment
- Auto-fix commit log
- Phase-by-phase results
- Design quality score (A/B/C/D)
- Trend analysis (last 5 runs)
- Newly learned patterns

## License

MIT
