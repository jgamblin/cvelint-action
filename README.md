# CVELint GitHub Action

This repository provides a GitHub Action for running [CVELint](https://github.com/mprpic/cvelint), a tool designed to validate CVE JSON 5.0 records. The action automates the process of identifying errors in CVE data and generates detailed reports in CSV and JSON formats.

## Outputs
The action generates the following files:

- `CVEV5Errors.csv`: A CSV file containing detailed error reports.
- `CVEV5Errors.json`: A JSON file containing detailed error reports.
- `CNAErrors.csv`: A summary CSV file of errors grouped by CNA.
- `CNAErrors.json`: A summary JSON file of errors grouped by CNA.
- `CNAReports/<CNA>.csv`: Individual CSV reports for each CNA.

## Workflow Overview

The action is defined in the [`cvelint.yml`](.github/workflows/cvelint.yml) file. It performs the following steps:

1. **Install Dependencies**: Installs required tools like `wget` and `jq`.
2. **Install CVELint**: Downloads and sets up the CVELint binary.
3. **Clone CVE Data**: Clones the CVE JSON 5.0 data from the [CVEProject/cvelistV5](https://github.com/CVEProject/cvelistV5) repository.
4. **Run Validation**: Executes CVELint to validate CVE data and generate reports:
   - `CVEV5Errors.csv` and `CVEV5Errors.json` for overall errors.
   - `CNAErrors.csv` and `CNAErrors.json` for CNA-specific summaries.
   - Individual CSV reports for each CNA.
5. **Commit Reports**: Commits the generated reports back to the repository.

## Features

- **Automated Validation**: Validates CVE JSON 5.0 records using CVELint.
- **Error Reporting**: Generates error reports in multiple formats (`CVEV5Errors.csv`, `CVEV5Errors.json`, etc.).
- **CNA-Specific Reports**: Creates individual reports for each CNA (CVE Numbering Authority).
- **Scheduled Runs**: Supports scheduled validation runs and triggers on repository events.
- **Customizable**: Easily extendable for additional workflows or custom validation rules.


## Contributing
Contributions are welcome! Feel free to open issues or submit pull requests to improve this action.

## License
This repository is licensed under the MIT License. 
