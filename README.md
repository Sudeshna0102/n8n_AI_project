# Customer Data Cleaning Workflow (n8n + Google Sheets)

An n8n workflow that pulls a messy customer dataset from Google Sheets, cleans it step by step, and writes a formatted, table-style result to a separate sheet, leaving the raw data untouched.

> Built with **Claude (Anthropic)** as an AI pair-programmer. 

## Problem

Real-world customer data is rarely clean. This project uses a 30-row dummy dataset that deliberately contains the problems analysts meet every day:

| Problem | Example in the raw data |
|---|---|
| Exact duplicate rows | The same order appears twice |
| Fuzzy duplicates | `ROHIT   VERMA` / `+91 96325 87410` / `10-Mar-2026` vs `Rohit Verma` / `9632587410` / `10/03/2026` |
| Extra spaces | `"  rahul sharma "`, `"vikram  patel"` |
| Inconsistent case | `PRIYA DAS`, `kavita nair` |
| Messy phone numbers | `+91-91234-56780`, `(909) 090-9090`, `98765 43210` |
| Messy ages | `34 yrs`, `41 years` |
| Mixed date formats | `05/01/2026`, `2026-01-12`, `1-Mar-2026`, `23.06.2026` |
| Missing values | Missing contact numbers and purchase dates |
| Impossible dates | Delivery before purchase, shipping after delivery |

All dates fall between **01-01-2026 and 01-11-2026**.

## Workflow

```
Manual Trigger
  -> Get Raw Data
  -> Standardize Data
  -> Trim Spaces
  -> Remove Duplicates
  -> Flag Date Errors
  -> Replace Missing With NA
  -> Capitalize Column Names
  -> Write to Cleaned Data Sheet
  -> Bold Headers and Create Table
```

| # | Node | n8n node type | What it does |
|---|---|---|---|
| 1 | Manual Trigger | Manual Trigger | Starts the workflow on demand. |
| 2 | Get Raw Data | Google Sheets (Get Row(s)) | Reads every row from the raw sheet (`Sheet1`). |
| 3 | Standardize Data | Code (JavaScript) | Capitalises each word in the name, keeps only digits 0-9 in phone and age, drops a leading `+91`, converts all dates to `dd-mm-yyyy`. |
| 4 | Trim Spaces | Edit Fields (Set) | Applies `trim()` and collapses repeated spaces in all 6 columns. |
| 5 | Remove Duplicates | Remove Duplicates | Removes rows that are identical across all fields. |
| 6 | Flag Date Errors | Code (JavaScript) | Adds a `Date Flag` column: flags shipping date after delivery date, delivery before purchase, and missing dates; otherwise `OK`. |
| 7 | Replace Missing With NA | Code (JavaScript) | Replaces every empty cell with `NA` (after flagging, so missing dates are still detected). |
| 8 | Capitalize Column Names | Code (JavaScript) | Converts all headers to capitals, e.g. `CUSTOMER NAME`. |
| 9 | Write to Cleaned Data Sheet | Google Sheets (Append Row) | Writes the cleaned rows to the separate `Cleaned Data` tab. |
| 10 | Bold Headers and Create Table | HTTP Request (Sheets API `batchUpdate`) | Bolds the header row and converts the range into a Google Sheets table. The built-in Sheets node cannot do this, so the API is called directly. |

## Before and after

| Raw | Cleaned |
|---|---|
| `"  rahul sharma "` | `Rahul Sharma` |
| `98765 43210` | `9876543210` |
| `34 yrs` | `34` |
| `2026-01-12` | `12-01-2026` |
| `1-Mar-2026` | `01-03-2026` |
| *(empty contact number)* | `NA` |
| Shipping `24-03-2026`, Delivery `21-03-2026` | `FLAG: Shipping date is after delivery date; Delivery date is before purchase date` |

The 30 raw rows reduce to **20 unique rows** after de-duplication.

## Repository contents

```
data/customers_dummy_raw.csv                      Dummy dataset (30 rows, 6 columns)
workflow/n8n_customer_data_cleaning_v2.json       Importable n8n workflow
docs/prompt.md                                    The prompt used to build the project
docs/screenshots/                                 Workflow canvas, raw and cleaned sheets
```

## How to run it

**Requirements:** an n8n account (n8n Cloud works) and a Google account.

1. Create a Google Sheet. Import `data/customers_dummy_raw.csv` into the first tab and **untick** "Convert text to numbers, dates and formulas". Add an empty tab named `Cleaned Data`.
2. In n8n, choose **Create workflow -> Import from File** and select the JSON in `workflow/`.
3. In **Get Raw Data** and **Write to Cleaned Data Sheet**, create a *Google Sheets OAuth2* credential (sign in with Google) and select your spreadsheet and the correct tab.
4. In **Bold Headers and Create Table**:
   - Replace `YOUR_SPREADSHEET_ID` in the URL with the ID from your sheet link (the text between `/d/` and `/edit`).
   - Replace `123456789` (twice in the body) with the `gid=` number of the `Cleaned Data` tab.
   - Select the same Google credential and turn on **Execute Once**.
5. Click **Execute workflow**. The result appears in the `Cleaned Data` tab.

### Notes and limitations

- The table can only be created once. To run the workflow again, delete the table and rows in `Cleaned Data` first.
- Fuzzy duplicates are caught when they differ in case, spacing, phone format or date format, because standardisation makes them identical. Spelling typos (e.g. "Sharma" vs "Sharna") would need string-similarity matching and are not handled.
- Phone numbers keep their digits only; a leading `+91` country code is removed.
- Values are written as plain text (`RAW`) so Google Sheets does not reinterpret `dd-mm-yyyy` dates.

## How this was built

This project was built with **Claude (Anthropic)** as an AI pair-programmer.

- **Me:** defined the problem and the cleaning rules, imported and configured the workflow in n8n Cloud, connected the Google credentials, debugged the setup and checked the output.
- **Claude:** generated the dummy dataset, the n8n workflow JSON and the JavaScript for the Code nodes, and suggested troubleshooting steps.

> Edit the split above so it matches what you actually did.

## The prompt

The final version of the request that produced this workflow:

```text
Action: Make n8n workflow for cleaning a dummy google sheet dataset, Steps 1: Create a dummy google sheet dataset with 30 rows and 6 coloums - customer name, contact number, age, date of purchase, delivery date, shipping date, step 2: add some duplicate rows, some fuzzy duplicates, some extra space, some missing value in the contact number and date of purchase column, and some non standardized values, in a few rows the date of delivery should be before the date of purchase. The dates must varry between 1/1/2026 to 1/11/2026. Step 3: The workflow should be like this - first the data will be fetched from the sheet, then the standardization (start each word with capital letter, dates should be in "dd-mm-yyyy" format, the phone number should only have digits 0-9, The customer name should start with capital letter and the surname should also start with capital letter, age should only be in digits 0-9), in the next step trim the extra spaces with the trim function, in the next step remove the duplicate rows, in the next step check whether shipping date > deliver date, in cases of true flag the dates, replace the missing values with "NA". In the last step capitalize the column names, bold them and make the dataset into a table. Each step represent each node on n8n. Give me the name of each node that i should use and the inputs in them. I want the clean output in a separate sheet than the raw data sheet.
```

**How the prompt evolved:** the first version had no `NA` replacement and no requirement for a separate output sheet. After reviewing the first result I added both.

Link of the dataset: https://docs.google.com/spreadsheets/d/1lFozBvoZm8_sQfPvIrp4pzFNUtVt_De_uhGAQpU--HU/edit?usp=sharing
Link of the workflow: https://sud-01.app.n8n.cloud/workflow/kOkEz5LzTm2lVfT8?projectId=id6fBlbhXBjTHL03
