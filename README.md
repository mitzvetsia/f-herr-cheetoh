# f herr cheetoh 🧀

**Tariff COGS Sheet Builder** — drop a supplier commercial invoice (CSV/XLSX) and the
customs broker bill PDF (containing the CBP Form 7501 Entry Summary), get back the
invoice with per-item tariff costing:

| added column | meaning |
|---|---|
| `TARIFF %` | the matched 7501 entry line's ad-valorem duty rate (sum of Chapter-99 + base HTS rates; MPF/HMF excluded) |
| `ENTRY LINE` | which 7501 line (001, 002, …) the item was entered under |
| `DISCOUNTED COST EACH / AMOUNT` | the customs entered value pro-rated back to the item (invoice-level discounts are allocated pro-rata by the broker) |
| `ENTRY FEES EACH / AMOUNT` | MPF + HMF + other entry fees allocated in dollars, pro-rata by entered value (MPF is capped per entry, so it can't be a flat %) |

Everything runs **in the browser** — the invoice and customs bill never leave the
machine. Scanned PDFs are OCR'd locally (pdf.js render → tesseract.js sparse-text
OCR → positional 7501 parser). The only network activity is fetching three
libraries from jsDelivr on load, plus the optional webhook push.

## Using it

- **Hosted**: enable GitHub Pages once (Settings → Pages → deploy from `main`, `/ (root)`),
  then the tool lives at `https://mitzvetsia.github.io/f-herr-cheetoh/`
- **Local**: download `index.html`, double-click it.
- **In a Knack page**: add a Rich Text view and paste
  ```html
  <iframe src="https://mitzvetsia.github.io/f-herr-cheetoh/"
          style="width:100%;height:1500px;border:0"></iframe>
  ```

Workflow: drop the two files → *Scan PDF & build sheet* (tip: type the 7501's page
range, e.g. `5-10`, to skip cover pages) → review the entry-lines table (every cell
is editable and accepts arithmetic like `2368.50+947.40`; green ✓ = rate × entered
value matches the printed duty) → download.

## Outputs

- **CSV** — flat, Knack-import-ready by default (header as row 1, data rows only;
  untick "Knack-ready" for the verbatim supplier layout with preamble/footer).
- **Workbook (XLSX)** — the audit trail: same rows with *live formulas* (click any
  computed cell to see the math), a `7501 LINES` sheet with the entry lines as
  reviewed plus reconciliation checks, and a README sheet.
- **Send to webhook** — POSTs the full dataset as JSON to a Make webhook (URL is
  remembered per browser) for downstream automation (e.g. creating replenishment
  records in Knack).

### Webhook payload shape

```
{
  source: "scw-tariff-cogs-tool",
  generatedAt: ISO-8601,
  invoice:     { file, invoiceNo, gross, discounts, net },
  customsBill: { file, sumEnteredValues, entryFees, billDutyToCustoms },
  entryLines:  [ { line, coo, tariffPct, enteredValue, duty, mpf, hmf,
                   baseHts, groupInvoiceGross, matchedGroups[] } ],
  items:       [ { item, poNo, cPoNo, piNo, partNo, productCode, description,
                   qty, coo, unitPrice, amount,
                   tariffPct, entryLine, discountedCostEach, discountedAmount,
                   entryFeesAmount } ]
}
```

All numbers are plain numerics (not display strings). If the webhook doesn't send
CORS headers the browser can't read the response; the tool falls back to an opaque
send and says so — add a Webhook Response module with
`Access-Control-Allow-Origin: *` for a green "delivered" confirmation.

## How it reads the 7501

The parser is built around the form's redundancy, so OCR mistakes get caught
instead of trusted:

1. Only pages containing *Entry Summary* content are parsed.
2. Line blocks are segmented by **content**, not the (often OCR-dropped)
   line-number column: a line accumulates Chapter-99 rate rows until the base-HTS
   row that carries the entered value; read line-number tokens only corroborate
   the strictly-sequential numbering.
3. Every line must satisfy `rate × entered value ≈ printed duty`, and
   `MPF ≈ 0.3464% × value`. Misread rates/values are auto-corrected only when the
   redundancy proves the fix (and never to a rate above 200%); everything else is
   flagged amber for manual review.
4. Invoice items are grouped by (DESCRIPTION, origin) and matched to entry lines
   of the same origin by comparing group totals against entered values; totals are
   reconciled against the invoice net, the 7501's Total Entered Value, and the
   bill's "Duty to Customs" charge.

## Dev notes

- Single self-contained HTML file, no build step. Pinned CDN deps: pdfjs-dist
  3.11.174, tesseract.js 5.1.1, SheetJS 0.18.5.
- OCR uses Tesseract page-segmentation mode 11 (sparse text) — the 7501 is a
  sparse form and default segmentation drops isolated cells. Override with
  `?psm=N` in the URL for experiments.
- Development history (and the Playwright end-to-end / replay test harnesses) live
  in `mitzvetsia/scw-knack-custom-js`, branch `claude/invoice-pdf-tariff-tool-csu77v`.
  The test fixtures are real supplier/customs documents and are deliberately **not**
  committed to this public repo.
