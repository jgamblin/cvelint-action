# CVELint GitHub Action

This repository contains a GitHub Action for running [CVELint](https://github.com/mprpic/cvelint), a tool for validating CVE JSON 5.0 records. The action checks for errors in CVE data and outputs the results in both CSV and JSON formats.

## Features

- **Automated Validation**: Automatically validates CVE JSON 5.0 records using CVELint.
- **Error Reporting**: Outputs detailed error reports, `CVEV5Errors.csv`,`CVEV5Errors.json`, `CNAErrors.csv` and `CNAErrors.json`.
- **Scheduled Runs**: Runs on a schedule (every 12 hours) or on push/pull request events to the `main` branch.
- **Customizable**: Easily extendable for additional checks or workflows.

## Outputs

- `CVEV5Errors.csv`: A CSV file containing detailed error reports.
- `CVEV5Errors.json`: A JSON file containing detailed error reports.
- `CNAErrors.csv`: A summary CSV file of errors grouped by CNA.
- `CNAErrors.json`: A summary JSON file of errors grouped by CNA.

## Contributing
Contributions are welcome! Feel free to open issues or submit pull requests to improve this action.

## License
This repository is licensed under the MIT License. 