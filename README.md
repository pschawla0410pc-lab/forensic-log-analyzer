# Automated Data Pipeline & Forensic Log Analyzer

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/pschawla0410pc-lab/forensic-log-analyzer/blob/main/Automated_Data_Pipeline_Forensic_Log_Analyzer.ipynb)

A Python data pipeline that ingests a folder of mixed-format server logs (`.txt` and `.json`), separates valid files from corrupt ones, and reports which IP addresses generate the most **5xx server errors**.

## Overview

Real log folders are messy. This project handles that by making sure a single corrupt file never stops the run: every file is either parsed into a clean record or isolated in a quarantine folder.

## Pipeline steps

1. **Download:** fetch the raw log files from a GitHub repository using the GitHub REST API and `requests`.
2. **Filter:** keep only `.txt` and `.json` files, skipping hidden files (starting with `.`) and backup files (starting with `~`).
3. **Parse:** read each file with a format-specific parser (`parse_json_file`, `parse_text_file`). Each parser returns a `(record, error)` pair instead of raising exceptions.
4. **Wrap:** a `DataParser` class chooses the right parser from the file extension and tracks whether the file was valid.
5. **Analyze:** collect clean records, queue corrupt files for quarantine, and count 5xx responses per IP with `collections.Counter`.
6. **Export:** save the clean records to `analytics_ready.json` and copy corrupt files into `data/quarantine/`.

## Results

| Metric | Value |
|--------|-------|
| Files downloaded | 74 |
| Valid log files after filtering | 72 |
| Clean records | 65 |
| Corrupt files quarantined | 7 |

**Top 5 IPs by 5xx errors**

| IP address | 5xx errors |
|------------|-----------|
| 45.33.32.156 | 3 |
| 198.51.100.23 | 3 |
| 10.0.0.8 | 3 |
| 192.168.1.14 | 2 |
| 203.0.113.55 | 2 |

**The 7 corrupt files**

- `access_070.txt`, `access_071.txt`: text logs with only one field (the status code is missing)
- `event_065.json`, `event_066.json`: JSON with syntax errors
- `event_067.json`, `event_068.json`, `event_069.json`: valid JSON that is missing `ip_address` or `status_code`

## Repository contents

```
forensic-log-analyzer/
├── Automated_Data_Pipeline_Forensic_Log_Analyzer.ipynb   # the full pipeline
├── analytics_ready.json                                  # output: the 65 clean records
├── requirements.txt
└── README.md
```

The notebook creates two more items when it runs: `daily_logs/` (the downloaded raw logs) and `data/quarantine/` (copies of the corrupt files). Neither is committed.

## How to run

**Google Colab (easiest):** click the Colab badge above, then choose *Runtime > Run all*.

**Locally:**

```bash
git clone https://github.com/pschawla0410pc-lab/forensic-log-analyzer.git
cd forensic-log-analyzer
pip install -r requirements.txt jupyter
jupyter notebook
```

Open the notebook and run all cells. The raw logs are downloaded in the first cell.

> GitHub's unauthenticated API allows 60 requests per hour. If the first cell fails with an error, wait and try again.

## Output format

Each record in `analytics_ready.json` looks like this:

```json
{
  "source": "daily_logs/event_000.json",
  "ip": "203.0.113.55",
  "status": 200
}
```

## Known limitations and next steps

- Status codes keep the type they had in the source file (integer from JSON, string from text), so they are compared as strings when counting 5xx errors. Converting them all to integers would make the output more consistent.
- Parsers only catch `JSONDecodeError`/`KeyError` (JSON) and `ValueError` (text); values are not validated beyond the file being readable.
- The extra fields in the JSON logs (timestamp, endpoint, response time) are not used yet. They would allow time-series and per-endpoint analysis.
- Possible additions: a chart of status codes and top offending IPs, unit tests for the parsers, and a standalone `pipeline.py` script.

## Tech stack

Python 3, `requests`, `json`, `os`, `shutil`, `collections.Counter`
