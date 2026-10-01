# cvelint-action Improvements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Modernize and optimize the cvelint-action workflow — reducing redundant runs, adding trend tracking, cleaning up the README, and managing repo growth.

**Architecture:** The main workflow (`.github/workflows/cvelint.yml`) gets rewritten to run cvelint twice (full + summary) as JSON, then derives all CSV outputs and per-CNA reports via `jq`. A new `trends.csv` file tracks historical data. A separate maintenance workflow handles run pruning and commit squashing. The README is rewritten for the actual audience.

**Tech Stack:** GitHub Actions, cvelint CLI, jq, gh CLI, bash, git

---

### Task 1: Rewrite the Main Workflow

**Files:**
- Modify: `.github/workflows/cvelint.yml`

This is the core change. Replace the current 4+360 cvelint invocations with 2 cvelint runs + jq derivations.

- [ ] **Step 1: Replace the full workflow file**

Replace the entire contents of `.github/workflows/cvelint.yml` with:

```yaml
name: CVELint
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  schedule:
    - cron: "0 */12 * * *"

jobs:
  CVE-Lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Install Packages
        run: |
          sudo apt-get update
          sudo apt-get install -y wget jq

      - name: Install CVELint
        run: |
          wget -q https://github.com/mprpic/cvelint/releases/latest/download/cvelint_Linux_x86_64.tar.gz
          tar -xzf cvelint_Linux_x86_64.tar.gz
          chmod +x cvelint

      - name: Clone CVE Dataset
        run: |
          git clone https://github.com/CVEProject/cvelistV5 --depth=1

      - name: Run CVELint Full Scan (JSON)
        run: |
          ./cvelint -ignore="E007,E008,E009" -format json ./cvelistV5/cves > CVEV5Errors.json || {
            exit_code=$?
            if [ ! -s CVEV5Errors.json ]; then
              echo "::error::CVELint crashed with exit code $exit_code and produced no output"
              exit $exit_code
            fi
          }

      - name: Derive Full Scan CSV
        run: |
          jq -r '
            ["CVE","CNA","File","RuleName","ErrorCode","ErrorPath","ErrorText"],
            (.results[] | [.cve, .cna, .file, .ruleName, .errorCode, .errorPath, .errorText])
            | @csv
          ' CVEV5Errors.json > CVEV5Errors.csv

      - name: Run CVELint CNA Summary (JSON)
        run: |
          ./cvelint -summary -format json ./cvelistV5/cves > CNAErrors.json || {
            exit_code=$?
            if [ ! -s CNAErrors.json ]; then
              echo "::error::CVELint crashed with exit code $exit_code and produced no output"
              exit $exit_code
            fi
          }

      - name: Derive CNA Summary CSV
        run: |
          jq -r '
            ["CNA","ErrorCode","ErrorName","ErrorCount"],
            (.results[] | .cna as $cna | .errors[] | [$cna, .errorCode, .errorName, .errorCount])
            | @csv
          ' CNAErrors.json > CNAErrors.csv

      - name: Generate Per-CNA Reports
        run: |
          mkdir -p CNAReports
          jq -r '.results[].cna' CVEV5Errors.json | sort -u | while read -r CNA; do
            jq -r --arg cna "$CNA" '
              ["CVE","CNA","File","RuleName","ErrorCode","ErrorPath","ErrorText"],
              (.results[] | select(.cna == $cna) | [.cve, .cna, .file, .ruleName, .errorCode, .errorPath, .errorText])
              | @csv
            ' CVEV5Errors.json > "CNAReports/${CNA}.csv"
          done

      - name: Update Trends
        run: |
          TIMESTAMP=$(jq -r '.generatedAt' CVEV5Errors.json)
          TOTAL_ERRORS=$(jq '.results | length' CVEV5Errors.json)
          TOTAL_CNAS=$(jq -r '[.results[].cna] | unique | length' CVEV5Errors.json)

          # Count errors per error code
          ERROR_COUNTS=$(jq -r '
            [.results[].errorCode] | group_by(.) | map({(.[0]): length}) | add
            | to_entries | sort_by(.key) | map(.value) | join(",")
          ' CVEV5Errors.json)
          ERROR_HEADERS=$(jq -r '
            [.results[].errorCode] | group_by(.) | map(.[0]) | sort | join(",")
          ' CVEV5Errors.json)

          # Create header if file doesn't exist
          if [ ! -f trends.csv ]; then
            echo "timestamp,total_errors,total_cnas,${ERROR_HEADERS}" > trends.csv
          fi

          echo "${TIMESTAMP},${TOTAL_ERRORS},${TOTAL_CNAS},${ERROR_COUNTS}" >> trends.csv

      - name: Update README Stats
        run: |
          TIMESTAMP=$(jq -r '.generatedAt' CVEV5Errors.json)
          TOTAL_ERRORS=$(jq '.results | length' CVEV5Errors.json)
          TOTAL_CNAS=$(jq -r '[.results[].cna] | unique | length' CVEV5Errors.json)

          # Build the stats block
          STATS_BLOCK="| Metric | Value |
          |--------|-------|
          | Total Errors | ${TOTAL_ERRORS} |
          | CNAs with Errors | ${TOTAL_CNAS} |
          | Last Run | ${TIMESTAMP} |"

          # Replace content between markers
          sed -i '/<!-- STATS:START -->/,/<!-- STATS:END -->/{
            /<!-- STATS:START -->/{
              p
              r /dev/stdin
            }
            /<!-- STATS:END -->/!d
          }' README.md <<< "$STATS_BLOCK"

      - name: Commit Changes
        if: github.event_name != 'pull_request'
        uses: EndBug/add-and-commit@v10
        with:
          default_author: github_actions
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 2: Verify the YAML is valid**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/cvelint.yml'))"`
Expected: No output (valid YAML)

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/cvelint.yml
git commit -m "refactor: optimize workflow to 2 cvelint runs + jq derivations

Replaces 4 full dataset scans + 360 per-CNA invocations with
2 cvelint runs (full JSON + summary JSON) and jq-based CSV
derivation and per-CNA report extraction. Adds smarter error
handling that distinguishes lint errors from crashes."
```

---

### Task 2: Rewrite the README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace the README contents**

Replace the entire contents of `README.md` with:

```markdown
# cvelint-action

Automated CVE data quality monitoring. Runs [CVELint](https://github.com/mprpic/cvelint) every 12 hours against the full [CVE v5 dataset](https://github.com/CVEProject/cvelistV5), producing error reports that help security researchers assess CVE data quality and help CNAs fix their records.

[![CVELint](https://github.com/jgamblin/cvelint-action/actions/workflows/cvelint.yml/badge.svg)](https://github.com/jgamblin/cvelint-action/actions/workflows/cvelint.yml)

## Latest Stats

<!-- STATS:START -->
| Metric | Value |
|--------|-------|
| Total Errors | — |
| CNAs with Errors | — |
| Last Run | — |
<!-- STATS:END -->

## Reports

### All Errors

- [`CVEV5Errors.json`](CVEV5Errors.json) — every lint error in the CVE v5 dataset (JSON)
- [`CVEV5Errors.csv`](CVEV5Errors.csv) — same data in CSV format

Each record contains:

| Field | Description |
|-------|-------------|
| `cve` | CVE ID (e.g., CVE-2026-33994) |
| `cna` | CVE Numbering Authority short name |
| `file` | Path to the CVE record file |
| `ruleName` | Lint rule that triggered |
| `errorCode` | Error code (e.g., E002) |
| `errorPath` | JSON path to the error location |
| `errorText` | Human-readable error description |

### CNA Summary

- [`CNAErrors.json`](CNAErrors.json) — error counts grouped by CNA and error code (JSON)
- [`CNAErrors.csv`](CNAErrors.csv) — same data in CSV format

### Per-CNA Reports

The [`CNAReports/`](CNAReports/) directory contains individual CSV reports for each CNA, making it easy to find errors for a specific organization.

### Trends

[`trends.csv`](trends.csv) tracks error counts over time — one row per run, with total errors, CNA count, and per-error-code breakdowns.

## Error Codes

| Code | Rule | Description |
|------|------|-------------|
| E001 | check-public-date | Invalid or missing public date |
| E002 | check-duplicate-reference-url | Duplicate reference URL in a CVE record |
| E003 | check-missing-reference-tags | Reference URL missing required tags |
| E004 | check-leading-trailing-space | Leading or trailing whitespace in fields |
| E005 | check-incorrect-cvss-severity | CVSS severity doesn't match the score |
| E006 | check-bogus-url | Reference URL is not reachable |
| E010 | check-invalid-self-references | Unnecessary self-reference URL |

> **Note:** E007 (invalid version string), E008 (invalid vendor string), and E009 are excluded from reports due to high volume. See [cvelint rules](https://github.com/mprpic/cvelint) for full details.

## About CVELint

This action wraps [CVELint](https://github.com/mprpic/cvelint) by [@mprpic](https://github.com/mprpic) — a CLI tool that validates CVE records for errors that aren't enforceable by the [CVE v5 JSON schema](https://github.com/CVEProject/cve-schema/tree/master/schema/v5.0) or validated by CVE Services.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: rewrite README for security researchers and CNAs

Replace upstream cvelint README copy with project-specific content:
badges, auto-updated stats section, report format docs, error code
table, and upstream attribution."
```

---

### Task 3: Initialize the Trends File

**Files:**
- Create: `trends.csv`

- [ ] **Step 1: Seed trends.csv from current report data**

Generate the initial trends file from the existing `CVEV5Errors.json` so the README stats work immediately (the workflow will append to this file on subsequent runs):

```bash
TIMESTAMP=$(jq -r '.generatedAt' CVEV5Errors.json)
TOTAL_ERRORS=$(jq '.results | length' CVEV5Errors.json)
TOTAL_CNAS=$(jq -r '[.results[].cna] | unique | length' CVEV5Errors.json)
ERROR_HEADERS=$(jq -r '[.results[].errorCode] | group_by(.) | map(.[0]) | sort | join(",")' CVEV5Errors.json)
ERROR_COUNTS=$(jq -r '[.results[].errorCode] | group_by(.) | map({(.[0]): length}) | add | to_entries | sort_by(.key) | map(.value) | join(",")' CVEV5Errors.json)

echo "timestamp,total_errors,total_cnas,${ERROR_HEADERS}" > trends.csv
echo "${TIMESTAMP},${TOTAL_ERRORS},${TOTAL_CNAS},${ERROR_COUNTS}" >> trends.csv
```

- [ ] **Step 2: Seed the README stats from current data**

Run the same `sed` replacement on the new README to populate the stats table with current values:

```bash
TIMESTAMP=$(jq -r '.generatedAt' CVEV5Errors.json)
TOTAL_ERRORS=$(jq '.results | length' CVEV5Errors.json)
TOTAL_CNAS=$(jq -r '[.results[].cna] | unique | length' CVEV5Errors.json)

STATS_BLOCK="| Metric | Value |
|--------|-------|
| Total Errors | ${TOTAL_ERRORS} |
| CNAs with Errors | ${TOTAL_CNAS} |
| Last Run | ${TIMESTAMP} |"

sed -i '' '/<!-- STATS:START -->/,/<!-- STATS:END -->/{
  /<!-- STATS:START -->/{
    p
    r /dev/stdin
  }
  /<!-- STATS:END -->/!d
}' README.md <<< "$STATS_BLOCK"
```

- [ ] **Step 3: Commit**

```bash
git add trends.csv README.md
git commit -m "feat: add trends tracking and seed initial stats

Create trends.csv with first data point from current reports.
Populate README stats section with current error counts."
```

---

### Task 4: Add Maintenance Workflow (Run Pruning + Commit Squashing)

**Files:**
- Create: `.github/workflows/maintenance.yml`

- [ ] **Step 1: Create the maintenance workflow**

Create `.github/workflows/maintenance.yml`:

```yaml
name: Maintenance
on:
  schedule:
    # Run weekly on Sundays at 3am UTC for run pruning
    - cron: "0 3 * * 0"
    # Run monthly on the 1st at 3am UTC for commit squashing
    - cron: "0 3 1 * *"
  workflow_dispatch:
    inputs:
      squash_commits:
        description: "Also squash old automated commits"
        type: boolean
        default: false

jobs:
  prune-runs:
    runs-on: ubuntu-latest
    steps:
      - name: Delete Old Workflow Runs
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          REPO="${{ github.repository }}"
          CUTOFF_DATE=$(date -d '30 days ago' -u +%Y-%m-%dT%H:%M:%SZ)

          echo "Deleting workflow runs older than ${CUTOFF_DATE}"

          # Get run IDs older than 30 days
          gh api repos/${REPO}/actions/runs \
            --paginate \
            --jq ".workflow_runs[] | select(.created_at < \"${CUTOFF_DATE}\") | .id" \
          | while read -r RUN_ID; do
            echo "Deleting run ${RUN_ID}"
            gh api repos/${REPO}/actions/runs/${RUN_ID} -X DELETE || true
          done

  squash-commits:
    if: github.event.inputs.squash_commits == 'true' || (github.event_name == 'schedule' && github.event.schedule == '0 3 1 * *')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Squash Old Automated Commits
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          # Find the oldest commit we want to KEEP (30 days ago)
          CUTOFF_DATE=$(date -d '30 days ago' -u +%Y-%m-%d)
          KEEP_FROM=$(git log --oneline --after="${CUTOFF_DATE}" --format="%H" | tail -1)

          if [ -z "$KEEP_FROM" ]; then
            echo "No commits to squash"
            exit 0
          fi

          # Find the parent of the oldest kept commit
          SQUASH_UNTIL=$(git rev-parse "${KEEP_FROM}^" 2>/dev/null)

          if [ -z "$SQUASH_UNTIL" ]; then
            echo "Nothing old enough to squash"
            exit 0
          fi

          # Count how many automated commits we're squashing
          SQUASH_COUNT=$(git log --oneline "${SQUASH_UNTIL}" --grep="Commit from GitHub Actions (CVELint)" | wc -l | tr -d ' ')

          if [ "$SQUASH_COUNT" -lt 10 ]; then
            echo "Only ${SQUASH_COUNT} automated commits to squash, skipping"
            exit 0
          fi

          echo "Squashing ${SQUASH_COUNT} automated commits up to ${SQUASH_UNTIL}"

          # Create a new root with the tree at SQUASH_UNTIL
          NEW_ROOT=$(git commit-tree "${SQUASH_UNTIL}^{tree}" -m "Squashed ${SQUASH_COUNT} automated CVELint report commits

          Historical reports squashed to reduce clone size.
          See trends.csv for historical error counts.")

          # Rebase the remaining commits on top
          git rebase --onto "${NEW_ROOT}" "${SQUASH_UNTIL}" main

          # Force push
          git push --force origin main
```

- [ ] **Step 2: Verify the YAML is valid**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/maintenance.yml'))"`
Expected: No output (valid YAML)

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/maintenance.yml
git commit -m "feat: add maintenance workflow for run pruning and commit squashing

Weekly job deletes workflow runs older than 30 days.
Manual dispatch option to squash old automated commits."
```

---

### Task 5: Re-test Ignore Flags

**Files:**
- Possibly modify: `.github/workflows/cvelint.yml`

This task requires running cvelint locally or in CI to check file sizes. It can only be done if the cvelint binary is available.

- [ ] **Step 1: Download cvelint and test locally (if on Linux/macOS)**

```bash
# Download cvelint for your platform
# On macOS:
wget -q https://github.com/mprpic/cvelint/releases/latest/download/cvelint_Darwin_x86_64.tar.gz
tar -xzf cvelint_Darwin_x86_64.tar.gz
chmod +x cvelint
```

If cvelint is not available locally (e.g., wrong platform, no CVE dataset clone), skip this task and document it as a follow-up. The ignore flags can be tested by temporarily removing them in the workflow and checking the output file sizes after the next scheduled run.

- [ ] **Step 2: Run with all rules enabled and measure output**

```bash
# Requires a local clone of cvelistV5
./cvelint -format json ./cvelistV5/cves > /tmp/full-output.json
ls -lh /tmp/full-output.json
```

If the output is under 50MB: remove the `-ignore="E007,E008,E009"` flags from the workflow and update the README error codes table to include E007/E008/E009.

If over 50MB: keep the flags. Add a comment in the workflow explaining why:

```yaml
      # E007, E008, E009 excluded: output exceeds 50MB with these rules enabled
```

- [ ] **Step 3: Commit (if changes made)**

```bash
git add .github/workflows/cvelint.yml README.md
git commit -m "chore: update ignore flags based on file size testing"
```

---

### Task 6: Final Verification

- [ ] **Step 1: Validate all YAML files**

```bash
python3 -c "
import yaml, glob
for f in glob.glob('.github/workflows/*.yml'):
    print(f'Checking {f}...')
    yaml.safe_load(open(f))
    print('  OK')
"
```

Expected: All files pass.

- [ ] **Step 2: Verify README renders correctly**

Open `README.md` in a markdown preview and verify:
- Badge renders
- Stats table has markers
- Report links point to correct files
- Error codes table is complete
- No broken formatting

- [ ] **Step 3: Verify trends.csv has valid data**

```bash
head -5 trends.csv
# Should show header row + 1 data row
```

- [ ] **Step 4: Review git log for clean commit history**

```bash
git log --oneline -10
```

Verify all commits are present and messages are clear.

- [ ] **Step 5: Final commit (if any cleanup needed)**

```bash
git add -A
git commit -m "chore: final cleanup for cvelint-action improvements"
```
