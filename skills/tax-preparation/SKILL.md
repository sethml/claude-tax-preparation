---
name: tax-preparation
description: Prepare and fill federal and state tax return PDF forms. Use this skill whenever the user mentions taxes, tax returns, filing taxes, 1040, W-2, refund, deductions, or wants help with any aspect of preparing or completing their tax return — even if they just say "help me do my taxes." Also trigger for questions about tax brackets, deductions, credits, or anything tax-related.
user-invocable: true
---

# Tax Preparation Skill

Prepare federal and state income tax returns: read source documents, compute taxes, fill official PDF forms.

**Year-agnostic** — always look up current-year brackets, deductions, and credits. Never reuse prior-year values.

## Folder Structure

Organize all work into subfolders of the working directory:

```
working_dir/
  source/              ← user's source documents (W-2, 1099s, prior return, CSVs)
  work/                ← ALL intermediate files (extracted data, field maps, computations)
    pdf_pdfplumber/    ← pdfplumber text extraction (one .txt per document)
    pdf_ocrmac/        ← ocrmac extraction, if available (one .txt per document)
    pdf_tesseract/     ← Tesseract tessdata_best extraction (one .txt per document)
    pdf_img2table/     ← img2table bordered-table extraction (one .txt per document)
    pdf_marker/        ← marker --strip_existing_ocr extraction, if used (one .md per document)
    images/            ← 300 DPI JPEG page images for ambiguous documents
    summary/           ← reconciled per-document tax value summaries (one .txt per document)
    computations.txt   ← all tax math (federal, state, capital gains, rental)
    f1040_fields.json  ← field discovery dumps (one per form)
    f8949_fields.json
    f1040sd_fields.json
    ca540_fields.json  ← (or equivalent state form)
    ...                ← add more as needed (Schedule E, Form 4562, Form 1116, etc.)
    expected_*.json    ← verification expected values
  forms/               ← blank downloaded PDF forms
    f1040_blank.pdf
    f8949_blank.pdf
    f1040sd_blank.pdf
    ca540_blank.pdf    ← (or equivalent state form)
    ...                ← add all applicable forms (Schedule E, A, B, 1-3, 8959, 8960, 4562, etc.)
  output/              ← final filled PDFs + fill script
    fill_YEAR.py       ← the fill script
    f1040_filled.pdf
    f8949_filled.pdf
    f1040sd_filled.pdf
    ca540_filled.pdf   ← (or equivalent state form)
    ...                ← all filled forms
```

Create these folders at the start. Keep the working directory clean — no loose files.

## Computation Rules

**ALL math must be done in code (Python), NEVER by reasoning/inference.** LLMs are
unreliable calculators. Even simple arithmetic (multiplying percentages, summing
columns) must be executed in a Python script or one-liner, not reasoned about in text.

Specific rules:
- **Never round intermediate results.** Use exact values (fractions, full decimal
  precision) throughout. Only round to whole dollars at the final step when writing
  to the Form Values sheet. Rounding a percentage from 50.9227% to 50.90% at an
  intermediate step can introduce $100+ errors that compound across years.
- **Use exact fractions** (e.g., `1200/4500`) instead of decimal approximations
  in all allocation computations.
- **Cross-check computed values** against prior year returns where available
  (e.g., `prior_depreciation / prior_MACRS_rate` should equal the depreciable basis).

### The Computation Workbook

All tax computations are captured in `work/tax_computations.xlsx` — a spreadsheet
built by a Python script (`work/build_computations.py`) using the `build_workbook.py`
framework. This workbook is the single source of truth for all numbers on the return.

**Required sheets:**

| Sheet | Contents | Rule |
|-------|----------|------|
| Source Data | Raw numbers from tax documents and user input. **Include document-level totals** (e.g., "Schwab 1099 — Total Proceeds") so validation can reconcile individual items against the document total. | No computation — extraction only |
| Tax Tables | Brackets, rates, thresholds, MACRS rates | Values from government sources |
| *(computation sheets)* | One per area: Capital Gains, Rental Property, Depreciation, FTC, QDCG Worksheet, etc. | ALL values are computed in Python with exact precision; each row records its formula as a string for auditing |
| Federal Return | One row per 1040 line | Formula references to computation sheets |
| State Return | One row per state form line | Same |
| Form Values | Rounded output for the fill script | `round_dollar()` applied here and ONLY here |
| Validation | Cross-form consistency checks | Must all PASS before filling forms |
| Carryforwards | Items carrying to next year | For next year's preparer |

**Workflow:**
1. Run `extract_tax_tables.py forms/` → produces `work/extracted_tables.json`
   with values parsed directly from the downloaded forms (standard deductions,
   thresholds, exemption credits). Import these into your build script — do NOT
   look up these values separately.
2. Write `work/build_computations.py` — a Python script that imports
   `build_workbook.TaxWorkbook` and computes everything. The script must call
   `preflight_check(work_dir, forms_dir)` before building — this verifies
   that instruction-notes files exist (proving you read the instructions).
3. Run it → produces `work/tax_computations.xlsx`
4. Run `validate_return.py work/tax_computations.xlsx --forms-dir forms/`
   → must pass ALL checks, including input-level checks that cross-verify
   Tax Tables values against the actual forms.
5. The fill script reads from the Form Values sheet — no hardcoded values

**The fill script must not contain any tax math.** It is a thin mapping from
the Form Values sheet to PDF field names. If a number needs to change, change
the build script and re-run — never edit the fill script's values directly.

```python
# Example: build_computations.py structure
import sys
sys.path.insert(0, "path/to/scripts")
from build_workbook import TaxWorkbook, Row, FormField, Check, round_dollar, macrs_rate

wb = TaxWorkbook(tax_year=YEAR, taxpayer="Jane Doe")

# 1. Source data (from extracted documents)
wb.source_data("W-2", [
    ("Box 1 - Wages", 92347.50, "w-2.pdf"),
    ("Box 2 - Federal withheld", 14283.00, "w-2.pdf"),
])

# 2. Computation (all math in Python, exact precision)
wages = 92347.50
std_ded = 15750
taxable = wages - std_ded
tax = compute_tax_simple(taxable, fed_brackets)  # looked up in 1b

wb.computation("Federal Tax", [
    Row("Wages", val=wages, formula="SourceData[W-2 Box 1]"),
    Row("Standard deduction", val=std_ded, formula="TaxTables[Single std ded]"),
    Row("Taxable income", val=taxable, formula="wages - std_ded", is_subtotal=True),
    Row("Tax", val=tax, formula="compute_tax_simple(taxable, brackets)", is_total=True),
])

# 3. Form values (rounding happens here)
wb.form_values("1040", [
    FormField("1a", "Wages", val=round_dollar(wages), pdf_field="f1_47"),
    FormField("15", "Taxable income", val=round_dollar(taxable), pdf_field="f2_06"),
])

# 4. Validation
wb.validate([Check("Tax > 0", expected=True, actual=tax > 0)])

wb.save("work/tax_computations.xlsx")
```

## Context Budget Rules

These rules prevent context blowouts that cause compaction:

1. **NEVER read PDFs with the Read tool.** Each page becomes ~250KB of base64 images (a 9-page return = 1.8 MB). Use the PDF extraction pipeline in Phase 1a instead.
2. **NEVER read the same document twice.** Each document's tax-relevant values are saved to `work/summary/<document>.txt` on first read. Use those summaries for all downstream work.
3. **Run field discovery ONCE per form** as a bulk JSON dump to `work/`. Do NOT use `--search` repeatedly.
4. **Save all computed values to `work/computations.txt`** so they survive compaction.
5. **Do NOT send raw reader output to subagents.** Subagents that need document data should read the reconciled `work/summary/` files, not the raw `work/pdf_*/` extractions. Raw reader output can be 10–50× larger than the summary.

## Workflow

### Phase 1: Gather Information

Do all automated/cheap work first, then ask the user.

#### 1a. Read and extract user-provided documents

Move source files to `source/` if needed. Use the Read tool for CSVs. For PDFs,
use the multi-reader extraction pipeline below. The pipeline runs multiple
readers, cross-checks their output, and produces a reconciled summary per document.

##### Step 1: Ask about marker-pdf

Before extracting, ask the user whether to also run marker-pdf:

> **Would you like to also run marker-pdf for PDF extraction?**
>
> Marker uses deep learning for OCR and table recognition — it produces the
> highest-accuracy structured output (6 GB install, around 30 min for a typical
> set of documents on Apple Silicon, 2–3× slower on CPU).
>
> Without marker, I'll run pdfplumber, ocrmac (if on macOS), Tesseract, and
> img2table (for bordered-form table detection), then cross-check them against
> each other. Where they disagree or the data looks suspicious, I'll view the
> page images directly to resolve it. This is fast (~1 min) but uses more of
> my context for the image review step.
>
> With marker, disagreements are rarer (marker is the most accurate local
> reader), so image review is only needed when marker itself looks off.

Record their choice in `work/pdf_extraction_method.txt`.

##### Step 2: Run readers

Run these readers on every PDF in `source/`. Save output to per-document
files — one per reader per document, using the document's filename (without
`.pdf`) as the base name:

| Reader | Output directory | File ext | When to run |
|--------|-----------------|----------|-------------|
| pdfplumber | `work/pdf_pdfplumber/` | `.txt` | Always |
| ocrmac | `work/pdf_ocrmac/` | `.txt` | Always on macOS (skip on Linux/Windows) |
| Tesseract (tessdata_best) | `work/pdf_tesseract/` | `.txt` | Always |
| img2table (Tesseract) | `work/pdf_img2table/` | `.txt` | Always |
| marker `--strip_existing_ocr` | `work/pdf_marker/` | `.md` | Only if user opted in |

**Reader setup:**

*pdfplumber:* `pip install pdfplumber` — extracts the embedded text layer.
Instant and perfect on machine-generated PDFs. Returns garbage on scanned docs.
```python
import pdfplumber
with pdfplumber.open(pdf_path) as pdf:
    text = "\n\n".join(p.extract_text() or "" for p in pdf.pages)
```

*ocrmac (macOS only):* `pip install ocrmac pdf2image` — uses Apple Vision.
Render each page to 300 DPI image, then:
```python
from ocrmac.ocrmac import OCR  # NOTE: must import from ocrmac.ocrmac, not ocrmac
text = "\n".join(annotation[0] for annotation in OCR(image_path).recognize())
```
~1 second per page. Near-perfect accuracy (~1 digit error per 500 values).

*Tesseract:* Install the system package (`brew install tesseract` or
`apt install tesseract-ocr`), then download `tessdata_best` models into
`work/` so they don't modify the system install:
```bash
mkdir -p work/tessdata_best
curl -L -o work/tessdata_best/eng.traineddata \
  https://github.com/tesseract-ocr/tessdata_best/raw/main/eng.traineddata
```
Render each page to 300 DPI image, then run:
```bash
tesseract page.jpg output -l eng --tessdata-dir work/tessdata_best
```
**Must use tessdata_best** — the default `tessdata_fast` has systematic 0↔8
digit confusion (e.g., `0.00` → `8.00`).

*img2table:* Install from GitHub main (PyPI version is broken with modern polars),
then ensure `opencv-contrib-python` is installed (img2table needs
`cv2.ximgproc.niBlackThreshold`, which is only in the contrib package — the base
`opencv-python` and `opencv-python-headless` packages do NOT include it):
```bash
pip install "git+https://github.com/xavctn/img2table.git@main"
# Remove any base opencv packages that would override contrib:
pip uninstall -y opencv-python opencv-python-headless 2>/dev/null || true
pip install opencv-contrib-python
```
Verify before running:
```bash
python3 -c "import cv2; assert hasattr(cv2.ximgproc, 'niBlackThreshold'), 'Need opencv-contrib-python'"
```
If you see `AttributeError: module 'cv2.ximgproc' has no attribute 'niBlackThreshold'`, run
`pip list | grep opencv` — if you see `opencv-python` or `opencv-python-headless` listed
alongside `opencv-contrib-python`, the base package is overriding the contrib build.
Fix: `pip uninstall -y opencv-python opencv-python-headless && pip install opencv-contrib-python`.

Requires Tesseract system package (already installed for the Tesseract reader above).
Render each page to 300 DPI image, then extract bordered tables:
```python
import os
os.environ["TESSDATA_PREFIX"] = "work/tessdata_best"  # Use tessdata_best models
from img2table.document import PDF
from img2table.ocr import TesseractOCR

ocr = TesseractOCR(n_threads=4, lang="eng")
doc = PDF(src=pdf_path)
tables = doc.extract_tables(ocr=ocr, implicit_rows=True, implicit_columns=True, min_confidence=50)
for page_num, page_tables in tables.items():
    for table in page_tables:
        df = table.df  # pandas DataFrame with cell contents
```
img2table uses OpenCV to detect bordered table cells and runs Tesseract within
each cell. It only extracts content inside bordered tables — documents without
grid lines return zero tables. For many documents its output will be empty or
gibberish, but for forms that structure numbers into boxes (W-2, 1099-DIV,
1099-INT), it produces **cell-level label–value association** that no other
text-based reader achieves. This is critical for forms where other readers
misassociate values with the wrong box due to reading-order ambiguity.

*marker-pdf:* `pip install marker-pdf` (~6 GB total with models).
**Always use `--strip_existing_ocr`.** Default mode trusts embedded OCR text,
which is garbage on some scanned documents (e.g., W-2 values come back blank).
```bash
marker_single "source/document.pdf" --output_dir work/pdf_marker \
  --strip_existing_ocr --disable_image_extraction
```

**All readers marked "Always" are MANDATORY.** Do NOT skip or ignore a reader
because it failed to install or produced an error. If a reader fails, diagnose
and fix the issue before moving on. Declaring "we have enough readers" and
proceeding without a required reader is NOT acceptable.

**Use subagents to run readers in parallel when possible.** Each subagent should
run one reader across a batch of documents. Keep subagent context small — give
them only the list of file paths and the extraction command, not the full
tax_data or other document contents.

##### Step 3: Reconcile and summarize

**Wait until ALL selected readers have finished successfully before starting
reconciliation.** Verify by checking output file counts for each reader directory.
Do NOT read any raw reader output during this wait — doing so violates the context
budget (you will re-read the same data again during reconciliation). If a reader is
still running, wait for it to complete.

For each source document, read the output from all readers and produce a
reconciled summary in `work/summary/<document>.txt`. The summary format is:

```
## YYYY <Form Type>: <Issuer> — <Recipient>
- EIN: XX-XXXXXXX
- Box 1 (Description): $X,XXX.XX
- Box 2 (Description): $X,XXX.XX
...
```

(See `tax_data_organized.txt` for a complete example of the expected format —
one section per document, with every tax-relevant field labeled by box number
and description, dollar amounts with `$` prefix and commas.)

**Cross-check procedure:** For each value, compare what each reader reported.
If all readers that produced usable output agree, use that value. Flag a value
as ambiguous if:
- Readers disagree (e.g., ocrmac says `$10,782.66`, Tesseract says `$10,702.66`)
- Only one reader produced output for a scanned page (pdfplumber returned garbage)
  and marker was not used
- A value looks suspicious (e.g., round numbers where decimals are expected,
  values that don't match document-level totals)
- Values don't seem internally consistent (e.g. withholding exceeds wages))

**When to trust img2table over other readers:** img2table's output is
authoritative for **which box a value belongs to** on bordered forms (W-2,
1099-INT, 1099-DIV). Other readers extract text in reading order and often
misassociate a value with the wrong box when multiple boxes share the same
y-coordinate (e.g., placing box 1's value into box 4). If img2table found a
bordered table and placed a value in a specific cell, trust its label–value
association over the other readers' positional guesses. However, img2table
returns nothing for documents without visible grid lines — don't treat empty
img2table output as a sign that data is missing.

**Resolve masked SSNs/TINs.** Employee copies of W-2s and recipient copies
of 1099s typically mask SSNs, showing only the last 4 digits (e.g.,
`XXX-XX-1803`). These masked values are **not valid for filing**. To get
full SSNs: (1) extract them from the prior year return if one was provided,
(2) otherwise ask the user during the Phase 1c questionnaire. Every summary
file must contain the **full 9-digit SSN** — never write a masked value like
`XXX-XX-1803` to a summary file when the full number is available from
another source.

##### Step 4: Resolve ambiguities with page images

For any document with ambiguous values, render the relevant pages to 300 DPI
JPEG images and read them directly:

```bash
mkdir -p work/images
pdftoppm -jpeg -r 300 -f <page> -l <page> source/document.pdf work/images/document-page
```

Then view the images (in Claude Code, read the image file directly; in
Copilot, use `view_image`) and determine the correct values. Update the
summary file with the resolved values.

**Do NOT view images for documents where all readers agree** — this wastes
context. Only use images as a tiebreaker.

##### Step 5: User verification — MANDATORY STOP

**Always stop here, regardless of how confident you are in the data.** Present
a compact summary of all extracted values (from the `work/summary/` files) —
one table per document showing key dollar amounts, SSNs/EINs, and form metadata.
Do not dump the raw summary files; show the key numbers in a readable format.

Before presenting, identify and prominently flag:
- Any value where readers disagreed and the ambiguity was resolved by image review
- Any document that was scanned (pdfplumber returned garbage), since OCR errors
  are more likely on scanned documents
- Any value that looks surprising: unusually round numbers, withholding that is
  very high or very low relative to income, values that changed dramatically from
  the prior year return, or amounts that seem inconsistent with other documents
- Any box where img2table's table-cell association disagrees with other readers'
  positional assignment (this often signals a misattributed box number)

Present the flags clearly before the table so they are not missed. Then ask the
user to confirm each flagged value and to do a sanity-check of the full table.

**Do NOT proceed to Phase 2 until the user has confirmed the data.**

**Prior year diff:** If the user provides a prior year return, don't just extract carryforwards — also compare what forms were filed last year vs. this year. Any form present last year but absent this year should be flagged later in 1c.

**Follow up on what documents imply.** Each document type may reveal additional forms or schedules. Don't just extract numbers — ask what's behind them:
- **1098 (Mortgage):** Property type? Co-borrowers? Ownership %? Rental use?
- **1099-R:** Rollover, early withdrawal, or conversion?
- **K-1:** What entity? Active or passive?
- **T4 / foreign tax docs:** Filed a return in that country? Actual tax owed vs. withheld? If not yet filed and expecting a refund, recommend filing the foreign return FIRST.
- **1099-B with basis not reported:** Employer stock plan? Supplemental basis statements?

#### 1b. Research tax policy, instructions, and year-specific values

**Start with government sources, then cross-check broadly.** The primary goal is **completeness** — you must not miss any applicable threshold, rate, or rule change.

**Layer 1 — Government sources (mandatory, comprehensive):**
1. **IRS Revenue Procedure** for the tax year's inflation adjustments (e.g., Rev. Proc. 2024-40 for 2025)
2. **Form instructions**: download 1040 instructions, extract Tax Computation Worksheet and QDCG Worksheet
3. **IRS Newsroom** (irs.gov/newsroom): mid-year legislative changes (e.g., OBBB Act)
4. **State tax authority**: form instructions, tax rate schedules, conformity announcements
5. Cross-check: standard deduction on the 1040 form itself

**Layer 2 — Broader sources (cross-checking):**
Search tax publications, CPA firm summaries, and news for the tax year. These surface recent legislation, known form issues, and state-specific gotchas. If a broader source contradicts a government source, investigate.

Do NOT hardcode thresholds — always look up fresh. These values go into the **Tax Tables** sheet:
- Federal brackets, standard deduction, QDCG thresholds
- Additional Medicare Tax / NIIT thresholds
- AMT exemption and phase-out thresholds
- Passive activity loss phase-out thresholds
- State brackets, standard deduction, exemption credits, phase-outs
- MACRS depreciation rates (use `macrs_rate()` from `build_workbook.py`)

**For state returns**, download the state form instructions and write line-by-line notes to `work/state_instructions_notes.txt`. For each line, note what it requires and flag any worksheets, limitations, phase-outs, or non-conformity with federal treatment.

#### 1c. Ask about life situations and reconcile documents — MANDATORY

**Generate the questionnaire dynamically from the current year's forms.**
Using the 1040, Schedules 1–3, and state form downloaded in 1b, scan each
line description and identify what information is needed that the documents
from 1a don't already provide. Convert each gap into a plain-language
question about the user's life situation — not tax jargon.

Always ask about these regardless (they affect filing but may not appear
on any document): filing status, dependents, state of residence, life
changes (marriage, divorce, new child, relocation), and tax payments made
(estimated, extension, prior-year overpayment applied).

Do NOT rely on a vague catch-all like "any other income or deductions?"
— people don't know what they don't know. Ask specifically about each
applicable area. When a topic comes up, ask the follow-up details that
forms alone won't tell you — for example:
- Rental property → ownership %, unit sizes (sq ft), rental income by unit, expense breakdown, existing depreciation schedule
- Home purchase/sale → date, price, primary residence or investment, co-owners
- Self-employment → type of business, estimated income, major expense categories
- Foreign income → which country, type of income, whether foreign return was filed

**Do NOT proceed until the user has answered.** "Same as last year" counts.

**After gathering answers, reconcile documents:**

1. **Generate an expected document checklist** from the user's answers.
   For each situation, identify the documents that should exist (e.g.,
   employed → W-2 + 1095-C; brokerage → 1099-B/DIV/INT; rental →
   depreciation schedule; foreign income → T4; mortgage → 1098).

2. **Compare against what was received in 1a.** Flag missing documents
   by name. Don't proceed until confirmed complete.

3. **Validate coverage against the 1040**: scan each line of the 1040
   and Schedules 1–3, confirm you have information for every applicable line.

4. **Check whether estimated taxes should have been paid.** Estimated
   payments are required when withholding doesn't cover the tax liability.
   This commonly applies when the taxpayer has significant income without
   withholding (self-employment, rental, capital gains, foreign income,
   investment income) or when the prior year return showed a large balance
   due. Check for these signals:
   - Prior year return shows amount owed > $1,000
   - Significant non-wage income in source documents (1099-NEC, 1099-MISC,
     K-1, rental income, large 1099-B gains, 1099-DIV/INT)
   - Prior year return includes Form 2210 or 1040-ES vouchers
   - IRS safe harbor: tax owed after withholding and credits ≥ $1,000
     AND withholding + credits < the lesser of 90% of current year tax
     or 100% of prior year tax (110% if prior year AGI > $150K)

   If any of these signals are present and NO estimated tax payment
   records (1040-ES receipts, bank records, or confirmation numbers) are
   found in `source/`, **explicitly ask the user:**

   > Your income this year includes [sources without withholding]. Were
   > any estimated tax payments (federal or state) made during the year?
   > If so, please provide the dates and amounts for each quarterly
   > payment (federal Form 1040-ES and/or state estimated payments).

   Do NOT silently assume no estimated payments were made — people
   commonly forget to include these records. If the user confirms no
   estimated payments were made, note this for the underpayment penalty
   computation in Phase 2.

### Phase 2: Build the Computation Workbook

Write `work/build_computations.py` — a Python script that creates the computation workbook using `build_workbook.TaxWorkbook`. This single script contains ALL tax math. Run it to produce `work/tax_computations.xlsx`.

**Before computing each form/schedule**, download its instructions and read the relevant sections. Do not compute any line from memory. This is especially important for:
- Worksheets embedded in form instructions (e.g., QDCG worksheet)
- Phase-outs and limitations that change with inflation adjustments
- New lines or changed line numbers between tax years

Compute supporting schedules BEFORE the main 1040, since their results flow into it.

**Capital Gains (if applicable):**
1. Extract **every** 1099-B box category (A, B, C, D, E, F). Include document-level total proceeds in Source Data so validation can reconcile. Missing a category (e.g., Box B short-term basis-not-reported) silently drops gains/losses.
2. Form 8949: individual transactions (Part I short-term, Part II long-term)
3. Schedule D: totals, loss limitation, carryover (check prior year for carryovers)
4. Net gain/loss → 1040 Line 7

**Rental Income (if applicable — Schedule E):**
1. **Allocate shared expenses** by square footage (not naive unit count). **Use exact fractions (e.g., 1200/4500) — never round percentages at intermediate steps.**
2. Rental income by unit
3. Rental expenses: mortgage interest, property tax, insurance, repairs, utilities — each allocated by sq ft × ownership %
4. **Depreciation:** carry forward from prior year or set up from scratch. **Cross-check:** `prior_year_dep ÷ prior_year_MACRS_rate` should equal the depreciable basis.
5. Net per property
6. **Passive activity loss rules:** if net loss, check Form 8582. Check prior year for suspended losses.
7. Net result → Schedule 1 → 1040 Line 8

**Self-Employment (if applicable):** Follow Schedule C instructions.

**Foreign Tax Credit (if applicable):** Use `compute_ftc.py` — do NOT implement
the Form 1116 computation manually. The script handles currency conversion and
the Line 18 QDCG adjustment (IRC 904(b)(2)(B)) which has a 100% manual error rate.
Pass the foreign amounts in their original currency along with the IRS yearly
average exchange rate.

**Federal Return:** Add a sheet with one row per 1040 line:
1. Gross Income (1a + 2b + 3b + 4b + 5b + 6b + 7a + 8)
2. Adjustments → AGI
3. Deductions → Taxable Income (compute both standard and itemized, use whichever is larger)
4. Tax (QDCG worksheet if applicable)
5. Credits, other taxes → Total Tax
6. Payments → Refund/Owed

**Underpayment Penalty (Form 2210, if applicable):**
After computing the Federal Return, check whether an underpayment penalty
applies.

If the penalty applies:
1. Download Form 2210 and its instructions
2. Compute the required annual payment
3. Compute the underpayment amount per quarter (required annual payment ÷ 4
   minus any estimated payments made for that quarter)
4. Compute the penalty using the IRS underpayment interest rate for each
   quarter (look up the rate — it changes quarterly)
5. Add a "Form 2210" computation sheet to the workbook
6. The penalty amount flows to 1040 Line 38 (estimated tax penalty)

If the taxpayer qualifies for an exception (e.g., annualized income
installment method — Schedule AI), compute that too. Do NOT skip the penalty
calculation — an unfiled Form 2210 means the IRS computes it for the taxpayer,
often at a higher amount.

**State Return:** Follow your line-by-line notes from 1b. Common non-conformity traps (not exhaustive — the instructions are authoritative):
- Itemized deduction limitations (e.g., CA Pease limitation — still applies post-TCJA)
- HSA non-conformity (CA, NJ don't recognize)
- Capital gains taxed as ordinary (CA)
- Exemption credit phase-outs, state-specific taxes (e.g., CA BH Tax > $1M)
- Different standard deduction amounts and itemization rules

**Run validation after building:**
```bash
python scripts/validate_return.py work/tax_computations.xlsx --forms-dir forms/
```
All checks must PASS before proceeding. This runs both consistency checks
(do the numbers add up?) and input checks (do the Tax Tables values match
what's printed on the actual forms?).

### Phase 3: Fill and Verify Forms

#### 3a. Download blank PDF forms

Download applicable forms to `forms/`. Use `/irs-prior/` for prior-year IRS forms (`/irs-pdf/` is always current year). Find state forms on the state tax authority's website. Verify each download has `%PDF-` header.

#### 3b. Discover fields, fill forms, and verify

**Discovery** — ONCE per form, use `--compact` (see `docs/form-filling.md` for details):
```bash
python scripts/discover_fields.py forms/f1040_blank.pdf --compact > work/f1040_fields.json
```

**Radio label discovery** — for any form with radio buttons (filing status, Yes/No, checkboxes grouped as radios), also run:
```bash
python scripts/discover_fields.py forms/f1040_blank.pdf --radio-labels
```
This maps each AP/N value to the text label next to its widget on the page. **Never assume AP/N values match any logical coding convention** — they follow physical widget position, not semantic meaning. Include the resulting mapping in the fill script comment block.

**Map fields to lines — MANDATORY before writing the fill script.** For every form, you MUST:
1. Review `discover_fields.py` output and map each field name to a form line number using the XFA `<speak>` descriptions and/or `Rect` positions
2. Write the mapping as a comment block at the top of the fill script section for that form
3. Watch for multi-page forms: IRS Form 1040 page 1 uses `f1_` fields, page 2 uses `f2_` fields. **Do NOT assume `f1_` numbering continues across pages**
4. For radio buttons (filing status, Yes/No questions), run `discover_fields.py --radio-labels` to determine the correct `/AP/N` value for each option — **never** assume the values follow any standard numbering convention (see `docs/form-filling.md` for details and examples)

**Fill script** — Write `output/fill_YEAR.py` using `scripts/fill_forms.py`. The fill script reads values from `work/tax_computations.xlsx` (Form Values sheet). It must not contain any tax math. See `docs/form-filling.md` for `fill_irs_pdf` vs `fill_pdf` usage.

**Common fill errors to check for:**
- Missing SSNs on any form (must be full 9-digit, never masked)
- Missing total/subtotal lines (Schedule 1 Part I/II totals, Schedule E column totals)
- Radio buttons and Yes/No questions left unfilled (especially Schedule E 1099 question)
- Addresses using incorrect data (verify against source documents)
- Page 2+ fields using wrong prefix (e.g., `f1_61` instead of `f2_01`)

**Verify** — Run `scripts/verify_filled.py` against expected values. Fix failures, re-run.

#### 3c. Verify against form instructions — MANDATORY

**For EVERY form you filled**, you MUST:
1. Fetch the form's instructions (IRS: `irs.gov/instructions/i{form}`; state: from state tax authority)
2. Read instruction text for every line you filled — especially **worksheets**, **special computations**, and **"see instructions"** references
3. Confirm each computation matches the instruction's method
4. Save verification notes to `work/verification.txt`

**Do NOT skip this.** Do NOT verify from memory — you must have the actual instruction text in context. With 1M token context, there is no cost reason to skip. The most common errors come from using a simplified formula when the instructions require a specific worksheet (e.g., Form 1116 Line 18 QDCG adjustment, Schedule CA Line 29 Pease limitation worksheet).

**Self-checks:**
1. Tax bracket year matches tax year (not filing year)
2. Arithmetic: totals add up, AGI = income − adjustments
3. Carryforwards picked up from prior year
4. Standard vs. itemized: computed both, used larger; state limitations applied
5. All payments included (withholding, estimated, extension, prior-year overpayment)
6. State return doesn't blindly follow federal treatment

#### 3d. Review other obligations

Systematically check whether the user's situation triggers obligations beyond the
returns. Look up current-year thresholds for each — do not assume prior-year values.

| Trigger | Obligation | Details |
|---------|-----------|---------|
| Foreign accounts > FBAR threshold | **FBAR (FinCEN 114)** | Separate filing at bsaefiling.fincen.treas.gov. Report account names, numbers, max balances. Look up current deadline. |
| Foreign assets > FATCA threshold | **Form 8938 (FATCA)** | Filed WITH the return. Look up current thresholds by filing status. |
| FTC based on withholding, foreign return not filed | **File foreign return first** | If expecting a refund, file foreign return before US return. Otherwise, warn that a refund will require amending. |
| Foreign trust distribution | **Form 3520** | Separate filing, due with return. Severe penalties for late filing. |
| Foreign mutual funds/ETFs (PFICs) | **Form 8621** | Filed with return. May need QEF or mark-to-market election. |
| Gifts > annual exclusion per recipient | **Form 709 (Gift Tax)** | Separate filing. No tax usually owed (uses lifetime exemption), but return still required. |
| Paid contractor > reporting threshold for rental | **File 1099-NEC/MISC** | Due Jan 31 of following year. If late, advise filing anyway (penalties increase over time). |
| Underpayment penalty | **Form 2210 + next-year safe harbor** | If estimated taxes were required but not paid (or underpaid), compute the penalty via Form 2210 (see Phase 2) and include it on 1040 Line 38. Also compute safe harbor for next year and recommend specific W-4 line 4(c) amount or quarterly 1040-ES amounts. |
| Can't file by deadline | **File for extension** | Extension = more time to file, NOT more time to pay. Estimated tax still due on original deadline. |
| ISO stock options exercised | **AMT exposure** | May need Form 6251. Must track AMT basis separately from regular basis for future sale. |
| HSA contributions | **Form 8889**; state conformity | Some states don't recognize HSA — add-back required on state return. |
| Home sold | **Exclusion rules** | Look up current exclusion. Must have owned and lived in home 2 of last 5 years. Partial exclusion may apply. |
| Rental property | **Local registration/license** | Varies by jurisdiction. Not a tax filing but a legal obligation. |
| Digital asset transactions | **Report per current-year requirements** | Cost basis tracking is complex. Brokers may not report basis. |
| Prior year overpayment applied | **Verify reflected in payments** | Easy to miss. Check prior year return and current year 1040-ES records. |

**Then pause and reason:** are there any obligations not covered by this table,
given everything you know about this taxpayer?

### Phase 4: Present Results

Show a summary table, then:
- **Sign your returns** — unsigned returns are rejected
- **Payment instructions** (if owed) — IRS Direct Pay, state web pay, deadline = filing deadline
- **Direct deposit** — recommend for refunds; ask for bank info
- **How to file** — e-file (IRS Free File, state e-file options) or mailing addresses
- **Action items** — every obligation from 3d, with deadlines, as a numbered checklist
