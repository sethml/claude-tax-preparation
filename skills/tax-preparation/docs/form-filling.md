# PDF Form Filling Reference

Read this document when you reach Phase 3 (Fill and Verify Forms).

## Field Discovery

- Field names change between years — always discover fresh
- XFA template is in `/AcroForm` → `/XFA` array, NOT from brute-force xref scanning
- Do NOT use `xml.etree` for XFA — use regex (IRS XML has broken namespaces)
- Save discovery output with an unambiguous filename per form: use `--compact` and redirect to `work/<form_name>_fields.json`. When two forms have similar field prefixes (e.g., IRS 1040-X and CA Schedule X both use `f1_`/`f2_`), name the output files differently (e.g., `f1040x_fields.json` vs `ca_schedule_x_fields.json`) to prevent overwrites

## PDF Filling Mechanics

- Remove XFA from AcroForm, set NeedAppearances=True, use auto_regenerate=False
- Checkboxes: set both `/V` and `/AS` to `/1` or `/Off`
- IRS fields need `[0]` suffix — use `add_suffix()`
- IRS checkboxes match by `/T` directly; radio groups match by `/AP/N` key via `radio_values`

## Critical: Field Name → Form Line Mapping

**Never guess field-to-line mappings.** IRS field names are opaque sequential identifiers — they do NOT correspond to form line numbers. Every mapping must be verified against the discovery output.

### Mapping procedure — MANDATORY for every form

1. Run `discover_fields.py` with `--compact` on the blank form
2. Review the XFA `<speak>` descriptions for each field — these describe the line/purpose
3. For fields lacking a clear `<speak>` description, use the field's `Rect` coordinates to determine its physical position on the form page, then match against the printed line labels using pdfplumber text extraction
4. **Multi-page forms**: field names restart with a new prefix on each page. IRS Form 1040 page 1 and page 2 use different prefixes — do NOT assume numbering continues across pages. Discover fields on each page separately and map them independently
5. **Write the mapping in a comment block** in the fill script before using it. Each entry must show: field name → form line number → description. Example:
   ```python
   # Schedule 1 field mapping (YEAR — re-verify each year):
   #   fX_XX[0] → Line N:  Business income or (loss) (Schedule C)
   #   fX_XX[0] → Line N:  Rental real estate, royalties... (Schedule E)
   #   fX_XX[0] → Line N:  Combine lines ... (Part I total)
   #   fX_XX[0] → Line N:  Total adjustments (Part II total)
   ```
6. **Fill ALL total/subtotal lines**, not just the individual detail lines. Schedules typically have Part I and Part II totals, column totals, and summary lines. Missing totals are a common error

### Common mapping traps

- **Schedule 1**: Has two parts (income additions and adjustments). Business income (Schedule C) and rental income (Schedule E) go on their own designated lines — verify which lines from the form. The part totals are at field numbers far from the individual income lines; don't assume they follow sequentially
- **Form 1040 page 2**: Tax computation, withholding, and payments sections are on page 2 and use a different field prefix than page 1. The fill script must map these with the page-2 prefix discovered from the blank form, not by continuing the page-1 numbering
- **Schedule E**: Has a Yes/No question ("Did you make any payments that would require you to file Form(s) 1099?") above the property table — don't skip it. Also, the property address may be a single combined field rather than separate street/city/state/ZIP fields

## Radio Buttons: Determining Values — MANDATORY

**Never assume radio button `/AP/N` values correspond to any logical numbering scheme.** The values are arbitrary identifiers assigned by the form designer based on physical widget position, not semantic meaning. For example, IRS filing status codes are 1=Single, 2=MFJ, 3=MFS, 4=HOH, 5=QSS — but the actual AP/N values on the 2024 1040 form were `/1`=Single, `/2`=HOH, `/3`=MFJ, `/4`=MFS, `/5`=QSS. Using `/2` for MFJ will silently check the wrong box.

### Procedure

For every form with radio buttons, run `discover_fields.py --radio-labels` to automatically map AP/N values to text labels by matching widget positions against nearby page text:

```bash
python scripts/discover_fields.py forms/f1040_blank.pdf --radio-labels
```

Output:
```
Radio group: FilingStatus_ReadOrder[0]  (page 0)
  /1      at (106.8, 588.0)  →  Single
  /3      at (106.8, 576.0)  →  Married filing jointly (even if only one had income)
  /4      at (106.8, 564.0)  →  Married filing separately (MFS)

Radio group: Page1[0]  (page 0)
  /2      at (373.2, 588.0)  →  Head of household (HOH)
  /5      at (373.2, 564.0)  →  Qualifying surviving spouse (QSS)
```

**Include the radio mapping in the fill script's comment block**, alongside the text field mapping:
```python
# 1040 Filing Status radio mapping (re-verify each year):
#   /1 → Single
#   /2 → Head of household (HOH)
#   /3 → Married filing jointly
#   /4 → Married filing separately
#   /5 → Qualifying surviving spouse
```

**Why this matters:** Radio groups that span multiple parent widgets (like Filing Status, which is split across `FilingStatus_ReadOrder` and `Page1`) have their AP/N values assigned by physical widget creation order, not by any standard coding convention. The mapping changes if the form designer rearranges the layout. Always re-discover for each tax year.

## SSN Handling

- SSNs on the return must be the **full 9-digit number**, never masked (e.g., `XXXXX1803` is NOT valid)
- Employee/recipient copies of W-2s and 1099s typically mask SSNs. Resolve full SSNs from the prior year return during Phase 1a extraction
- IRS form SSN fields accept digits only (no dashes, no spaces)
- Fill SSN fields on EVERY form that has them: main return, amended return, and all schedules

## Form-Specific Notes (IRS)

- **1040**: First few fields are fiscal year headers, not name fields. SSN = 9 digits, no dashes — must be the full number, never masked. Digital assets = crypto only, not stocks. Tax computation, withholding, and payments are on page 2 with a different field prefix than page 1
- **1040-X (Amended)**: Filing instructions require attaching a corrected Form 1040 (marked "AMENDED") behind the 1040-X, plus any changed schedules. Read the 1040-X instructions to determine all required attachments. State amended returns may require a separate schedule (e.g., CA Schedule X)
- **Schedule 1**: Has two distinct parts (income additions and adjustments to income). Business income (Schedule C) and rental income (Schedule E) each go on their own dedicated lines — confirm which lines by reading the form, not from memory. Part totals must also be filled; their field numbers are not adjacent to the detail lines
- **Schedule E**: Fill the Yes/No 1099 question. Property address may be a single combined field rather than separate components. Verify field layout from discovery
- **8949**: Box A/B/C checkboxes are 3-way radio buttons. Page totals appear at the end of the field list, not immediately after the last data row. Schedule D takes totals from 8949
- **Schedule D**: Some fields have `_RO` suffix (read-only) — skip those
- **Downloads**: Prior-year IRS = `irs.gov/pub/irs-prior/`, current = `irs.gov/pub/irs-pdf/`

## Form-Specific Notes (State)

- State forms vary widely in structure. Always run field discovery fresh.
- Some states use IRS-style XFA forms, others use simple AcroForm. Use `fill_irs_pdf` or `fill_pdf` accordingly based on what discovery reveals.
- Read the state form instructions to understand field naming conventions before mapping.
- **State amended returns** may require additional forms (e.g., CA Schedule X for amended 540/540NR). Read the state's amended return instructions to determine all required forms

### CA 540 Field Conventions

- Field names are `540-PPNN` (page+sequence, NOT line numbers). Checkboxes end with `" CB"`, radio buttons use named AP keys.
- CA forms available at `ftb.ca.gov/forms/YEAR/`.
- CA Schedule X (amended return explanation) uses IRS-style `f1_`/`f2_` field naming despite being a state form — use `fill_irs_pdf`, not `fill_pdf`
