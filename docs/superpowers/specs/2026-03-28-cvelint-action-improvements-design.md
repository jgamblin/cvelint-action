# cvelint-action Improvements Design

**Date:** 2026-03-28
**Status:** Approved

## Overview

Modernize and optimize cvelint-action — the automated GitHub Actions workflow that runs CVELint every 12 hours against the full CVE v5 dataset, producing error reports for security researchers and CVE Numbering Authorities (CNAs).

**Primary audience:** Security researchers and CVE data consumers.
**Secondary audience:** CVE Numbering Authorities wanting to fix their records.

## 1. Workflow Optimization

### Problem

The workflow currently runs cvelint 4 times against the full dataset (CSV full, JSON full, CSV CNA summary, JSON CNA summary) plus 360+ times for individual CNA reports. This is redundant and burns excessive Actions minutes.

### Design

- **Run cvelint twice instead of four times.** Run the full scan as JSON, then use `jq` to convert JSON to CSV. Same for the CNA summary. This cuts full-dataset scan time roughly in half.
- **Generate CNA reports from existing JSON.** Instead of invoking cvelint 360+ times, extract per-CNA data from `CVEV5Errors.json` using `jq` filters. Eliminates ~360 cvelint invocations.
- **Apply `-ignore` flags consistently.** CNA reports should use the same ignore rules as the main reports (currently they don't).
- **Smarter error handling.** cvelint returns exit code 1 when it finds lint errors (expected behavior) and other non-zero codes on actual failures. Replace blanket `continue-on-error: true` with a wrapper that allows exit code 1 (lint errors found) but fails the workflow on other exit codes (actual crashes). If cvelint doesn't distinguish exit codes, treat any non-zero exit with output as success and non-zero without output as a crash.

### Workflow Step Sequence (After Optimization)

1. Checkout repo
2. Install packages (wget, jq)
3. Download cvelint binary
4. Shallow clone CVE dataset
5. Run cvelint full scan → `CVEV5Errors.json`
6. Derive `CVEV5Errors.csv` from JSON via `jq`
7. Run cvelint CNA summary → `CNAErrors.json`
8. Derive `CNAErrors.csv` from JSON via `jq`
9. Generate per-CNA reports from `CVEV5Errors.json` via `jq`
10. Update trends file
11. Update README stats
12. Commit all changes

## 2. README Modernization

### Problem

The current README is a copy of the upstream cvelint tool's README. It describes CLI installation and usage rather than what this repo actually does.

### Design

New README structure:

1. **Title + one-line description** — "Automated CVE data quality monitoring"
2. **Badges** — workflow status, last run date, total error count
3. **What this does** — 2-3 sentences: runs cvelint every 12 hours against the full CVE v5 dataset, produces error reports, tracks per-CNA quality
4. **Latest stats** — auto-updated section with error count, CNA count, timestamp. Uses `<!-- STATS:START -->` / `<!-- STATS:END -->` comment markers for safe automated updates.
5. **Reports** — describes each output file:
   - `CVEV5Errors.csv` / `.json` — all errors in the CVE v5 dataset
   - `CNAErrors.csv` / `.json` — error summary by CNA
   - `CNAReports/` — individual CSV reports per CNA
   - `trends.csv` — historical error counts over time
   - Include example data formats
6. **Error codes** — brief table of what each error code means, linking to upstream cvelint for full details
7. **About cvelint** — short attribution section pointing to the upstream tool
8. **Trend data** — description of the trends file and what it tracks

Removes: installation instructions, build-from-source, CLI usage examples (belong on upstream repo).

## 3. Trend Tracking & Auto-Updated Stats

### Historical Trends File (`trends.csv`)

- Appended with one row per run
- Columns: `timestamp,total_errors,total_cnas,E001,E002,E003,E004,E005,E006,E010` (individual error code counts)
- Simple CSV format — machine-readable and browsable on GitHub
- Growth rate: ~2 lines/day (~730 lines/year)

### Auto-Updated README Stats

- Workflow step uses `sed` to update content between `<!-- STATS:START -->` and `<!-- STATS:END -->` markers
- Stats pulled from JSON output via `jq`
- Displays: total errors, number of CNAs with errors, timestamp of last run
- Committed alongside report files in the existing commit step

## 4. Workflow Run Pruning

### Problem

1,900+ automated workflow runs clutter the Actions tab.

### Design

- Add a cleanup step at the end of the workflow (or a separate lightweight scheduled job)
- Uses GitHub CLI (`gh run list` / `gh run delete`) to delete runs older than 30 days
- Uses the already-available `GITHUB_TOKEN` for authentication
- Keeps ~60 most recent runs for debugging

## 5. Commit History Squashing

### Problem

1,900+ automated commits make cloning slow and the git history unwieldy.

### Design

- Separate monthly scheduled workflow
- Squashes all automated commits ("Commit from GitHub Actions (CVELint)") older than 30 days into a single commit
- Preserves the last 30 days of individual commits for recent history browsing
- Requires force push to main — acceptable risk since the only committers are GitHub Actions and Dependabot
- The squash commit message summarizes what was collapsed (e.g., "Squash automated CVELint reports from 2026-01 to 2026-02")

## 6. Re-test Ignore Flags

### Problem

E007, E008, E009 are excluded from main reports. This was originally due to file size exceeding 100MB. May no longer be necessary.

### Design

- During implementation, run a test with all rules enabled and measure output file sizes
- If JSON/CSV are under ~50MB with all rules: drop the ignore flags for more complete reporting
- If still too large: keep the flags, document the reason in a workflow comment and in the README error codes section
- This is a validation step, not a guaranteed change

## Out of Scope

- Reusable composite action for others to consume
- GitHub Pages site for report browsing
- Data visualization / charting
- Notifications or alerting on error spikes
- Converting to a different CI platform
