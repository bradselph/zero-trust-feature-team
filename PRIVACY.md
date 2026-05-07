# Privacy Policy

**Zero-Trust Feature Team** is a local Claude Code plugin. It does not collect, transmit, or store any user data externally.

## What this plugin does

- Reads source files in your project directory to research integration points and conventions
- Writes feature-state files to `.claude/feature-state/` within your project directory
- Edits source files only when you have explicitly approved the implementation plan and only within the scope of an approved step
- All data stays on your local machine

## What this plugin does NOT do

- No external network requests
- No telemetry or analytics
- No data sent to third-party services
- No user accounts or authentication required beyond your existing Claude Code session

## Data stored

All state is written locally to `.claude/feature-state/` in your project:

- `task.json` -- captured feature requirements
- `research.json` -- codebase research index
- `plan.json` -- implementation plan and review state
- `steps/STEP-*.json` -- per-step specifications and status
- `log/` -- per-area research traces and per-step implementation/verification logs
- `FINAL_REPORT.md` -- the final summary, when produced

This data never leaves your machine unless you explicitly commit it to version control. The `.gitignore` included with this plugin excludes all runtime state from git by default.

## Contact

For questions or concerns: https://github.com/bradselph/zero-trust-feature-team/issues
