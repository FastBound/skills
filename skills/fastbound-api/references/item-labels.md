# Item label formats

FastBound prints **item labels** for the firearms in your inventory — typically the small barcode/info sticker that goes on the box, the tag, or the firearm's case. Customers can use the default Dymo-sized label or design their own with a constrained subset of HTML, CSS, and merge fields.

This is a UI-side feature, not a REST endpoint — but it's a real customer-facing capability and one of the most common "can you help me with my label?" questions support gets. The skill should be able to help users design and debug a label.

Reference:
- **Article**: https://fastbound.help/en/articles/1332831-item-label-formats
- **Sample HTML in the FastBound Support repo**: https://github.com/FastBound/Support/tree/main/labels — multiple working samples (file names like `Label Design 01-07.html`, `label01-01.html` through `label01-08.html`, `largeLabel02-02.html`, `flex-columns.html`, `flex-rows.html`). When helping a user, **start by reading one of these samples** so you see what FastBound's renderer accepts and structures look like in practice. They're the canonical reference for "what works."

## Supported label sizes

| Size | Note |
|---|---|
| **2-1/4" × 1-1/4" (≈ 2.25 × 1.25)** | The default. Dymo part 30334. No settings change needed to use it. |
| **2-15/16" × 4" (≈ 2.31 × 4)** | Larger; common alternate. |
| Other sizes | The samples directory includes 1" × 2-1/8", 3.5" × 1.4", and more. Pick the closest sample as your starting point. |

> **Sheet labels (Avery, etc.) are NOT supported.** Roll-fed labels only. If the user is asking about Avery sheets, redirect them — that's a feature gap, not a configuration question.

## Browser and printing requirements

These are real, documented requirements — if a user reports that "my label looks wrong," check these first:

- **Use Google Chrome or Microsoft Edge.** Safari is **not supported** for printing labels. (Firefox isn't called out either way; Chrome/Edge are the safe answer.)
- **Disable Print Headers and Footers** in the print dialog. Otherwise the page header and URL get printed across the label.
- **Print at the highest available DPI.** Low-quality printing degrades barcode readability — barcodes are the whole point of the label, so don't compromise.

## What HTML you can use

Allowed tags:

- **Headings**: `<h1>` through `<h6>`
- **Text**: `<p>`, `<strong>`, `<em>`, `<i>`, `<code>`
- **Structure**: `<div>`, `<br>`, `<hr>`, `<pre>`
- **Media**: `<img>` (use this for static logos; for the dynamic firearm data use merge fields, not images)
- **Tables**: `<table>`, `<thead>`, `<tbody>`, `<tr>`, `<th>`, `<td>`
- **Styling**: `<style>` (inline CSS in a style block, scoped to the label)

What that means in practice: standard semantic HTML works. Don't expect `<script>`, custom elements, web components, iframes, forms, or anything that would let arbitrary code run — the renderer is locked down on purpose.

CSS-wise: standard properties for layout and typography are fair game. Flexbox works (the samples include `flex-columns.html` and `flex-rows.html` — those are good starting points if you want columnar layouts). Stick to print-friendly properties; don't depend on browser-specific behavior beyond Chromium's print pipeline.

## Merge fields (the dynamic data)

These are the placeholders FastBound substitutes with the item's data when rendering the label. The article lists them; substitute the value from the current item:

- `Item Number`
- `Manufacturer`
- `Importer`
- `Model`
- `Serial`
- `Type`
- `Caliber`
- `Price`
- `MPN`
- `SKU`
- `UPC`
- `Location`
- `Barrel Length`
- `Overall Length`
- `Condition`
- `Acquire Purchase Order Number`

Look at the GitHub samples for the exact merge-field syntax FastBound expects (the article references "click a label image to view and copy the code" — the actual templates carry the binding syntax). When a user shares a draft label, **read the sample that's closest to their layout first** so you know what merge-field syntax their template should use.

## Workflow: design, preview, save

1. The user opens label settings in the FastBound UI.
2. They paste/edit HTML + CSS in the editor.
3. **Preview** with the Preview button before saving — that's the article's explicit recommendation. Catches layout breakage before it hits a real label roll.
4. Save. Some content is required to save — an empty template won't go through.

## Helping a user with their label

When a user shares a label HTML they're struggling with, a useful response sequence:

1. **Confirm size.** Which physical label are they printing on? Their HTML must be sized to fit. Mismatched label-stock and design dimensions is the most common "why is half my text cut off" issue.
2. **Confirm browser.** Chrome or Edge. Not Safari.
3. **Confirm print dialog settings.** Headers/footers off; high DPI.
4. **Read a sample close to their layout** (start with the GitHub samples at https://github.com/FastBound/Support/tree/main/labels) so you know the working merge-field syntax and structural conventions.
5. **Diagnose the HTML** — common issues:
   - Using disallowed tags (`<script>`, `<iframe>`, custom elements, etc.). Strip them.
   - Wrong merge-field syntax (typo, wrong case, used a field name not in the supported list).
   - Layout overflow because the design assumes a different label size.
   - Print CSS gaps — if it looks fine on screen but wrong on the label, look at `@media print`, page size, and margin handling. Use a sample as the reference for known-good print CSS.
   - Barcode rendering: if a barcode font is being used, make sure the font is embedded or referenced from a reliable source the print pipeline can resolve.
6. **Suggest concrete fixes** with the corrected HTML inline. The FastBound samples are not necessarily the most idiomatic HTML — they're functional and proven, which matters more for label rendering than purity. Don't refactor a working sample to be "nicer" without confirming it still renders correctly.

## What this skill cannot do

- **The skill cannot upload, save, or test a label design** — those are UI actions in the FastBound web app. The skill can produce HTML for the user to paste in.
- **The skill cannot guarantee a specific label-stock part number prints correctly** — different printers, drivers, and roll-stock vendors have small differences. Always preview, and if possible test-print on the actual hardware before committing a design across the whole inventory.

## Common pitfalls

- **Using Safari** to design or print — not supported.
- **Print headers and footers enabled** — leaves the page URL stamped across the label.
- **Designing for the wrong label size** — design dimensions must match the loaded stock.
- **Treating merge fields as case-insensitive without verifying** — always test the exact spelling and casing against a sample.
- **Trying to use disallowed HTML** — `<script>`, `<iframe>`, network calls, etc. The renderer ignores or strips them; don't waste time debugging a feature that won't render.
- **Skipping the Preview step** before saving — preview is one click and catches layout regressions.
- **Refactoring a working sample for "code quality"** — these samples render correctly; ergonomic improvements that break print rendering are net negative.
