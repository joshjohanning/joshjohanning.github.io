---
title: 'HSA Expense Analyzer CLI Tool'
author: Josh Johanning
date: 2025-09-04 10:00:00 -0500
description: A Node.js CLI tool that analyzes and visualizes HSA expense and reimbursement totals by year from a folder of receipts.
categories: [Hobbies, Tools]
tags: [Node.js, Personal Finance]
media_subpath: /assets/screenshots/2025-09-04-hsa-expense-analyzer
image:
  path: hsa-expense-analyzer.png
  width: 100%
  height: 100%
  alt: HSA Expense Analyzer sample output showing yearly expense breakdown and charts
---

## Overview

After a few years of saving HSA receipts, I had hundreds of PDFs and images with completely inconsistent naming. Then I saw [this post on X](https://x.com/VicVijayakumar/status/1733869605645385959) where someone shared a utility they built that standardizes receipt naming and graphs yearly totals. That's exactly what I needed! Therefore, I built my own Node.js CLI tool (with the help of GitHub Copilot 🤖) that parses receipt filenames and generates visual reports showing healthcare expenses and reimbursements year-over-year.

The tool works by reading filenames like `2023-05-15 - dentist - $150.00.pdf` and automatically categorizing them by year, tracking reimbursements, and creating ASCII charts. It's saved me hours of manual spreadsheet work which was my previous method of tracking this data.

You can find the complete source code on GitHub: [joshjohanning/hsa-expense-analyzer](https://github.com/joshjohanning/hsa-expense-analyzer-cli) and the npm package is published at [@joshjohanning/hsa-expense-analyzer-cli](https://www.npmjs.com/package/@joshjohanning/hsa-expense-analyzer-cli).

## The Problem

My HSA receipt folder looked like this mess:

- `receipt_scan_jan_2021.pdf`{: .filepath}
- `Doctor visit Jan 15 2022 $45.pdf`{: .filepath}
- `IMG_2847.jpg`{: .filepath} (a pharmacy receipt from who knows when and how much)
- `2023-dental-cleaning-receipt.pdf`{: .filepath}

Sound familiar? After years of saving receipts, I had three main problems:

- **Tracking reimbursements**: Which expenses have been reimbursed and which haven't?
- **Yearly analysis**: An orderly way to track receipts (expenses) and reimbursements by years without manually entering data into a spreadsheet
- **Reimbursable amount**: Since I don't submit my HSA claims right away, I wanted to know how much I could withdraw at any given time

## The Solution

Instead of tracking 400+ files by hand, I decided to establish a naming convention going forward and build a tool to parse it. The format is simple but consistent:

```text
<yyyy-mm-dd> - <description> - $<total>.pdf|png|jpg|whatever
```
{: .nolineno}

When I get reimbursed, I just add `.reimbursed.` before the file extension. The tool automatically detects this:

```text
<yyyy-mm-dd> - <description> - $<total>.reimbursed.pdf|png|jpg|whatever
```
{: .nolineno}

It did take me a little bit to go through each receipt and add a date and total, but it was worth it to have a consistent format going forward.

### Example File Structure

Here's what my receipt folder looks like now:

```text
receipts/
├── 2021-01-01 - doctor - $45.00.pdf               # Expense
├── 2021-02-15 - pharmacy - $30.00.reimbursed.pdf  # Reimbursed expense
├── 2022-02-01 - doctor - $50.00.reimbursed.pdf    # Reimbursed expense
├── 2022-03-15 - dentist - $150.00.png             # Expense
├── 2023-05-01 - doctor - $45.00.pdf               # Expense
├── 2024-07-15 - doctor - $50.00.pdf               # Expense
└── 2025-01-15 - doctor - $125.00.pdf              # Expense
```
{: .nolineno}

## Technical Implementation

I built this with Node.js because I wanted something I could run locally without dependencies on external services. Three main packages do the heavy lifting:

- **[`chartscii`](https://github.com/tool3/chartscii)** - Creates those ASCII bar charts you see in the output
- **`yargs`** - Handles command-line arguments (`--dirPath`, `--help`, etc.)
- **`prettyjson`** - Formats the summary tables nicely

The core logic is surprisingly simple - it's mostly string manipulation and basic math.

### What It Actually Does

1. **Reads filenames** - Splits on " - " and validates the date/amount format
2. **Groups by year** - Extracts the year from each date
3. **Tracks reimbursements** - Looks for `.reimbursed.` in the filename
4. **Warns about mismatched files** - Lists files that don't match the expected pattern
5. **Generates reports** - Summary tables plus three different ASCII charts
6. **Does the math** - Calculates the yearly totals for expenses and reimbursements

## Usage

The easiest way is to install as a global package from [npm](https://www.npmjs.com/package/@joshjohanning/hsa-expense-analyzer-cli) and run it:

```bash
npm install -g @joshjohanning/hsa-expense-analyzer-cli
hsa-expense-analyzer-cli --dirPath="/path/to/your/receipts"
```
{: .nolineno}


Or if you want to clone locally and hack on the code:

```bash
git clone https://github.com/joshjohanning/hsa-expense-analyzer-cli
cd hsa-expense-analyzer-cli
npm install
npm run test # runs with test data
npm run start -- --dirPath="/your/receipts/folder" # runs with your data
```
{: .nolineno}

## What The Output Looks Like

Here's what you get when you run it on a folder with a few years of receipts:

```text
2021:
  expenses:       $75.00
  reimbursements: $30.00
  receipts:       2
2022:
  expenses:       $250.00
  reimbursements: $100.00
  receipts:       3
2023:
  expenses:       $100.00
  reimbursements: $55.00
  receipts:       2
2024:
  expenses:       $50.00
  reimbursements: $0.00
  receipts:       1
2025:
  expenses:       $125.00
  reimbursements: $0.00
  receipts:       1
Total:
  expenses:       $600.00
  reimbursements: $185.00
  receipts:       9

Expenses by year
2021 ╢██████░░░░░░░░░░░░░░ $75.00
2022 ╢████████████████████ $250.00
2023 ╢████████░░░░░░░░░░░░ $100.00
2024 ╢████░░░░░░░░░░░░░░░░ $50.00
2025 ╢██████████░░░░░░░░░░ $125.00
     ╚════════════════════

Reimbursements by year
2021 ╢██████░░░░░░░░░░░░░░ $30.00
2022 ╢████████████████████ $100.00
2023 ╢███████████░░░░░░░░░ $55.00
2024 ╢░░░░░░░░░░░░░░░░░░░░ $0.00
2025 ╢░░░░░░░░░░░░░░░░░░░░ $0.00
     ╚════════════════════

Expenses vs Reimbursements by year
2021 Expenses       ╢██████░░░░░░░░░░░░░░ $75.00
2021 Reimbursements ╢██░░░░░░░░░░░░░░░░░░ $30.00
2022 Expenses       ╢████████████████████ $250.00
2022 Reimbursements ╢████████░░░░░░░░░░░░ $100.00
2023 Expenses       ╢████████░░░░░░░░░░░░ $100.00
2023 Reimbursements ╢████░░░░░░░░░░░░░░░░ $55.00
2024 Expenses       ╢████░░░░░░░░░░░░░░░░ $50.00
2024 Reimbursements ╢░░░░░░░░░░░░░░░░░░░░ $0.00
2025 Expenses       ╢██████████░░░░░░░░░░ $125.00
2025 Reimbursements ╢░░░░░░░░░░░░░░░░░░░░ $0.00
                    ╚════════════════════
```
{: .nolineno}

If you have files that don't match the expected naming pattern, you'll see a warning at the top of the output:

```text
⚠️  WARNING: The following files do not match the expected pattern:
Expected pattern: <yyyy-mm-dd> - <description> - $<amount>.<ext>
Files with issues:
  - 2021-01-15- doctor - 50.00.pdf
  - 2021-01-15-wrong-format-missing-dashes.pdf
  - wrong-format.pdf
```
{: .nolineno}

## What's Next

A few things I'm thinking about adding:

- **CSV export** - For people who want to analyze the data in Excel
- **Category parsing** - Extract "doctor" vs "pharmacy" vs "dental" from descriptions
- **Family member tracking** - Parse names to see spending per person
- **Year reimbursed vs. year incurred** - Track the year expenses were reimbursed vs. the year they were incurred (but does this matter??)

## Summary

This started as a weekend project to solve my own problem, but I realized other people probably have the same messy folder of HSA receipts. I love the [`chartscii`](https://github.com/tool3/chartscii) package that creates the ASCII charts which provide immediate visual feedback.

Plus, it was built entirely with GitHub Copilot 🤖, which was quite fun. I also learned a lot about creating and publishing `npm` packages, which has been useful for my day job.

The complete source code and documentation are available on GitHub at [joshjohanning/hsa-expense-analyzer-cli](https://github.com/joshjohanning/hsa-expense-analyzer-cli) and the npm package is published at [@joshjohanning/hsa-expense-analyzer-cli](https://www.npmjs.com/package/@joshjohanning/hsa-expense-analyzer-cli). Feel free to fork, contribute, or adapt it for your own needs!
