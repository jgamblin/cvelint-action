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
