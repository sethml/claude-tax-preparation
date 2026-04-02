# Visual Verification of Filled Forms

After filling any PDF form, verify both **correctness** and **completeness**
by visually inspecting the output. This applies during normal tax preparation,
not just skill testing.

## Correctness: every displayed value is right

Convert each page of the output PDF to an image. Go through every value
visible in the image and compare it to the fill script:

1. For each value on the image, find the corresponding line in the fill script
2. Confirm the value matches what was intended
3. If you find a discrepancy, examine the fill script, the field mapping
   (from `discover_fields.py` output), and the form instructions to decide
   which is correct — the displayed value, the fill script value, or neither
4. Fix whichever is wrong (mapping, computation, or both)

Common discrepancy sources:
- Field mapped to the wrong line (field names rarely match line numbers)
- Value placed on a credit line instead of a tax-subtotal line, or vice versa
- Radio button or checkbox state not matching the intended selection
- Stale values left over from a prior fill run

## Completeness: every expected value appears

Go through the tax computation (workbook or fill script) and confirm that
each output value appears on the correct form to be filed:

1. For each value in the Form Values sheet, find it on the rendered PDF image
2. Verify it appears on the correct line of the correct form
3. Missing values usually mean a field mapping is absent or the value was
   assigned to the wrong form
