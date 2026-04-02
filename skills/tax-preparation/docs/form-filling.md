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

## Radio Buttons: Determining Values

**Never assume radio button `/AP/N` values correspond to any logical numbering scheme.** The values are arbitrary identifiers assigned by the form designer based on physical widget position, not semantic meaning.

### How to determine the correct radio value

For each radio group, extract the widget positions and match against nearby text labels:

```python
import pdfplumber
from pypdf import PdfReader

def map_radio_labels(pdf_path, radio_field_prefix, page_num=0):
    """Map radio button AP/N values to their text labels.

    Args:
        pdf_path: Path to blank PDF form
        radio_field_prefix: e.g., "c1_3" for filing status
        page_num: 0-based page number
    """
    # 1. Get widget positions and AP/N values
    reader = PdfReader(pdf_path)
    page = reader.pages[page_num]
    widgets = []
    for annot in (page.get("/Annots") or []):
        obj = annot.get_object()
        t = str(obj.get("/T", ""))
        if not t.startswith(radio_field_prefix + "["):
            continue
        rect = obj.get("/Rect")
        ap_n = [k for k in (obj.get("/AP", {}).get("/N", {}).keys()) if k != "/Off"]
        if rect and ap_n:
            widgets.append({
                "ap_n": ap_n[0],
                "x": (float(rect[0]) + float(rect[2])) / 2,
                "y": (float(rect[1]) + float(rect[3])) / 2,
            })

    # 2. Get text labels from the page
    pdf = pdfplumber.open(pdf_path)
    words = pdf.pages[page_num].extract_words()

    # 3. Match each widget to nearest text to its right
    for w in widgets:
        best_label = "?"
        best_dist = float("inf")
        for word in words:
            # Text should be to the right and at roughly the same y
            dx = word["x0"] - w["x"]
            dy = abs(word["top"] - (pdf.pages[page_num].height - w["y"])) # flip y
            if dx > 0 and dx < 200 and dy < 15:
                dist = dx**2 + dy**2
                if dist < best_dist:
                    best_dist = dist
                    best_label = word["text"]
        print(f"  AP/N={w['ap_n']:4}  →  {best_label}")
```

**Example**: IRS 1040 filing status radio buttons — the AP/N values do **not** match the IRS filing status codes (1=Single, 2=MFJ, etc.). The values are assigned by physical widget position on the form and can differ from any logical ordering. Always run the position-matching script above to determine the correct value rather than guessing.

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
