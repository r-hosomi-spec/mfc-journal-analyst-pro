# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **Claude Skill** for analyzing MoneyForward Cloud Accounting (MFC) journal data. It is not a traditional application — the primary artifact is `SKILL.md`, which Claude reads at runtime to execute analysis tasks. The `scripts/` directory contains Python utilities that the skill invokes.

## Running the scripts

There is no build step or test suite. Install the only dependency and run scripts directly:

```bash
# Check/install dependency
python3 -c "import pandas; print(pandas.__version__)"
python3 -m pip install --user pandas

# Test the data loader (prints row count, transaction count, partner identification rate)
python3 scripts/load_journals.py <path.json|path.csv>

# Merge multiple paged API result files into one canonical JSON
python3 scripts/merge_journals.py <tool_result_p1.json> [p2.json ...] output.json

# Parse balance reconciliation screen text (残高照合), optionally cross-referencing journal JSON
python3 scripts/parse_balance_match.py [--journals journals.json] file1.txt [file2.txt ...]

# Parse account detail list screen text (明細一覧), optionally cross-referencing journal JSON
python3 scripts/parse_meisai_list.py [--journals journals.json] file1.txt [file2.txt ...]
```

All scripts write results to stdout as JSON (except `load_journals.py` which prints a summary). Diagnostic messages go to stderr.

## Architecture

### SKILL.md

The skill specification. When a user asks Claude to analyze MFC journals or create a handover sheet, Claude reads this file and follows the execution phases described in it. It defines:
- Skill trigger phrases
- Execution phases (Phase 1 auth → Phase 2 data fetch → Phase 3 analysis → Phase 4A/4B output)
- Interaction patterns (1-turn-per-account-type dialogue design)
- Output templates that must be reproduced **verbatim** — no summarization

### scripts/load_journals.py

The core library. Exposes three public functions used in Phase 3:

```python
from load_journals import load_df, enrich_partners, build_transaction_view, apply_tax_inclusive

df = load_df("journals_FY2024.json")   # auto-detects JSON (MFC API) or CSV (export)
df = apply_tax_inclusive(df)           # only for 税込経理 companies; adds tax_value to value
df = enrich_partners(df)               # adds 借方_実質取引先 / 貸方_実質取引先 columns
txn_df = build_transaction_view(df)    # aggregates to one row per 取引No
```

**Two views are critical** — using the wrong one produces incorrect counts:
- **行ビュー (`df`)**: One row per journal branch line. Use for BS balance calculation, account-level aggregation, department×account mapping, input-source analysis.
- **取引ビュー (`txn_df`)**: One row per transaction number. Use for partner aggregation, transaction counts, recurring pattern detection, compound journal templates. Compound journals (payroll, sales+tax, etc.) span multiple lines in df but are one row in txn_df.

Partner identification priority in both views: ① receivable/payable sub-account → ② trade partner field → ③ expense/revenue sub-account → ④ remark text extraction.

### scripts/parse_balance_match.py

Parses the 残高照合 (balance reconciliation) screen copy-paste. Input is tab-separated text: each processed transaction spans 2 lines (date/katakana/amounts/status/取引No/相手科目 on line 1, 補助科目/MF摘要 on line 2); unprocessed transactions are 1 line. When `--journals` is provided, cross-references by 取引No to enrich with full journal detail (tax classification, department, compound structure).

### scripts/parse_meisai_list.py

Parses the データ連携 > 登録済み一覧 > 明細一覧 screen copy-paste. Auto-detects format: **3-line format** (auto-fetch services: line 1 = date/katakana/amount/balance/service, line 2 = account name, line 3 = status/取引No) vs **1-line format** (manual-managed services: all fields on one line). Aggregates by (normalized katakana, account, sub-account, department, tax) and flags recurring/constant-amount patterns.

## Key domain conventions

### journal_type vs entered_by

`JOURNAL_TYPE_NORMAL` in the MFC public API covers **both** manual entry and data-linked transactions — the API does not distinguish them. `JOURNAL_TYPE_DATA_LINKAGE` exists in the OpenAPI spec but does not appear in real data as of 2026-04. Data-linked vs manual classification must be inferred by matching sub-account names against connected service names.

The 16 registration route categories and their `entered_by` values are defined in `SKILL.md` and in the `ENTERED_BY_LABEL` dict in `load_journals.py`.

### Tax accounting method (税込/税抜)

MFC API always returns `value` (pre-tax) and `tax_value` separately. For 税込経理 companies, call `apply_tax_inclusive(df)` before any analysis; otherwise all amounts will be understated by the tax amount.

### Terminology distinction

Three terms used in the skill must not be conflated:
- **入力ソース**: *Which route* journals entered MFC (16 categories, machine-detectable)
- **入力方法**: *Which screen/button* — not determinable from API data, not asked of users
- **証憑**: *Source document* (bank statement, invoice, etc.) — confirmed by dialogue with user

### Phase vs フェーズ naming

"Phase 1/2/3/4A/4B" refers to **internal skill execution phases** (used within SKILL.md and in conversation with the developer). "フェーズ1〜6" refers to the **monthly bookkeeping workflow phases** that appear in the generated handover sheet Step 5. Never write "フェーズA〜F" or "月次作業①〜⑥" — always use "フェーズ1〜6".

## Files that must not be committed

The `.gitignore` excludes runtime-generated files that contain client data or credentials:

```
.mfc_token.json          # OAuth access token
.tax_method.json         # tax accounting method for a company
.connected_accounts.json # connected service list
journals_*.json          # downloaded journal data
handover_*.md            # generated handover sheets
step9_reverse_map.json   # katakana → account reverse lookup
```
