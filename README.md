# BOM Excel to PDF

A browser-only tool that converts Excel BOM / cost-breakdown files to PDF, one PDF per file, with every visible sheet merged into that PDF and the file name built from the team's naming rule. Single file (`index.html`), no server, nothing is uploaded.

Part 1 is the user guide. Part 2 ("Project notes") is a hand-off for whoever continues the work (Claude or a developer): read it first when resuming.

---

# Part 1. User guide

## Publish with GitHub Pages

1. Create a **Public** repository (for example `bom-pdf-tool`).
2. Put `index.html` at the **root** of the repo, commit and push.
3. **Settings > Pages > Build and deployment**: Deploy from a branch, `main`, `/(root)`, Save.
4. Link after about a minute: `https://<github-username>.github.io/<repo-name>/`

To update: replace `index.html`, then `git add . && git commit -m "..." && git push`, wait about a minute, hard refresh (Ctrl+F5).

## Use

1. Choose **Season** (auto-filled from the first file if empty) and **Stage** (LR2, FLC, SMS, CFM). Leave **Date** empty if the file has no date.
2. Drag and drop or choose several `.xlsx` / `.xlsm` files. **Gender** and **Model name** are read from each file and can be edited per row.
3. **Convert to PDF**, then download each file or **Download all (.zip)**.
4. **Output**: *Sharp vector* (default, recommended): text, borders and fills are real vector graphics, crisp at any zoom, text is selectable, about 0.5 MB per file. *Exact fonts (image)*: each page is a picture (about 324 dpi, about 10 MB per file), keeps the original fonts but gets soft when zoomed.

## File name rule

```
SEASON STAGE BOM – GENDER MODEL NAME - MM.DD.YYYY.pdf      (ALL CAPITAL LETTERS)
FW27 FLC BOM – W BRIDGELINE SHORTY WP.pdf                  (no date)
SS28 FLC BOM – M SNAPLINE 6INCH WP CF - 10.06.2026.pdf     (with date)
```

- Separator before gender is an en dash `–`, separator before the date is a hyphen `-` (copied from the team's example image).
- Date is `MM.DD.YYYY` with dots. The user asked for `mm/dd/yyyy`, but `/` is illegal in file names, so dots are used. Date is omitted entirely when empty.
- Duplicate names in one batch get ` (2)`, ` (3)` appended.

---

# Part 2. Project notes (hand-off)

## Who and why

- User: project manager in footwear (Keen-style cost breakdown sheets), Vietnamese speaker, prefers direct professional replies, wants to be asked when a request is unclear, deploys by cloning/pushing to GitHub himself. Claude cannot create the repo, so it delivers `bom-pdf-tool.zip` and push commands.
- Workflow before the tool: Excel to PDF manually, renaming by the team rule. The tool exists to batch this and remove manual renaming.
- Original requirements: merge all sheets into one PDF; name by the rule (image: red rule text, example `FW27 FLC BOM – M CLASSIC ALPINE 2 - 8.5.2026.xlsx`); UI to add Excel files and pick Season / Stage (LR2-FLC-SMS-CFM); import many files at once. Later asks: auto-detect gender and model for every file, English UI, MM.DD.YYYY date, sharper text.

## Architecture (all in `index.html`)

- Libraries from CDN with jsDelivr fallback: ExcelJS 4.4.0 (reads Excel) and JSZip 3.10.1 (zip download) only (`ensureLibs`). **jsPDF was removed**: the PDF is written by the tool's own small writer (`PdfDoc`, `makePainter`), which can be tested without any external library.
- Pipeline per file: ExcelJS reads the workbook, `planSheet` computes layout, then `workbookToPdf(wb, {mode})` renders every page in one of two modes.
  - `vector` (default): `paintPageVector` emits PDF drawing operators (fills, borders, clipped text, images). Text uses the PDF standard fonts (Helvetica/Times/Courier, WinAnsi) with width tables baked in as `WIDTHS` (taken from pdf-lib's AFM data). Each line gets a horizontal-scaling `Tz` so its width equals the width of the original font (Calibri etc.) measured on a canvas, which keeps the Excel line breaks and alignment. Any cell containing characters outside WinAnsi (for example Chinese) is rasterised at high resolution as a small image with alpha (`paintTextRaster`). Pictures are embedded losslessly (Flate + soft mask) via `prepareImages`.
  - `image`: `drawPage` paints the whole page on a canvas, exported as JPEG (R = 4.5, quality 0.98, 16 MP cap) and placed as one image per page. Exact fonts but not sharp when zoomed.
- Shared by both modes: `collectItems`, `layoutCellText`, `borderEdges`, `overflowExt`, `hfLines`. Compression uses the browser's `CompressionStream('deflate')` (Chrome/Edge 80+, Firefox 113+, Safari 16.4+).
- **Why the first versions were raster:** sheets contain CJK characters (for example 弹性基布) and Calibri, which PDF standard fonts cannot reproduce, and fonts cannot be read from the user's computer by a web page. Raster looked soft when zoomed even at 324 dpi, so vector became the default (the user's complaint: "quality of PDF still blur, not full HD"). Trade-off of vector mode: Latin text uses Arial-like glyphs squeezed to Calibri widths, so it looks slightly different from Excel. A server with LibreOffice would match Excel exactly, but GitHub Pages cannot host that. Embedding a real font (for example Carlito, a Calibri-compatible font) was considered but not done because the font file source could not be verified offline.
- Sections of the script: (1) pure helpers between `// <pure>` and `// </pure>` markers, with no DOM so they can be unit-tested in Node; (2) the drawing engine; (3) library loader; (4) UI.
- Layout honours: print area, paper size (map `PAPER`, A3 = 8), orientation, margins, fit-to-page (1x1), print-title rows, header/footer codes (`&P`, `&N`, `&A`, `&F`, `&D`), merged cells, column widths / row heights, hidden rows/columns, fills, borders, fonts, number and date formats, theme colours with tint, embedded pictures (anchored to cells).
- Hidden sheets are skipped.
- Image-mode constants: `IMG_R = 4.5`, `IMG_JPEG = 0.98`, `MAX_PIXELS = 16e6` (Safari/iOS canvas limit). Earlier raster versions used R=2 then R=3/4.5 with jsPDF and still looked soft.

## The source files (what they look like)

- Sheet title "Keen Cost Breakdown Sheet". Typical file: 2 to 4 visible sheets (one per colourway, for example `CSA WP W`, `Black Black`, `Dk Earth Black`, `Magnet Black`).
- Page setup: A3 portrait, fit to 1 page wide by 1 tall, print area like `A1:G80`, print titles `1:8`, footer right `&P/&N`, margins about 0.25 in left/right, 0.75 in top/bottom.
- Header block labels (column A or D, label cells are **merged**, value to the right): Season, Valid Period, Category, Sample Name, Colorway, SAP#, Costing Size, Factory, Tooling (Mold #), Gender, Size Range, Newness or Carryover, Material %, Submit Date (usually blank, so no date in the name).
- A product picture sits in the top-right (G2:G8).

## Metadata detection (`detectMeta`, `splitGender`, `metaFromFilename`, `pickModel`)

Gender priority: Gender cell, then M/W/K/Y prefix or suffix inside the sheet's Sample Name, then the file name, then Size Range (`W'S`, `M'S`), then Category (`Women's...`).

`splitGender` rules: underscores become spaces; parentheses removed; leading `[SEASON] [STAGE] BOM -` removed; trailing date removed; dev codes like `25KU-1120D` and `RK-W-2817` removed; leading single letter M/W/K/Y (followed by space, `_` or `-`) or trailing single letter becomes gender; `6"` and `6 INCH` become `6INCH`; result uppercased. A word that merely starts with M (for example `MAPLE TOE`) is not treated as gender.

`pickModel(sheetModel, fileModel)`: if one contains the other, take the longer (sheets sometimes omit the leading word, for example sheet `PLATE TOE` vs file `Evanston Plate Toe`); otherwise prefer the sheet.

Known filename styles and expected result:

| File name | Season | Gender | Model |
|---|---|---|---|
| `BRIDGELINE_SHORTY_WP_W_FW27__25KU-1025__RK-W-2817.xlsx` | FW27 | W | BRIDGELINE SHORTY WP |
| `Evanston Chelsea 25KU-1120D.xlsx` | (sheet) | M from sheet | EVANSTON CHELSEA |
| `Evanston Plate Toe 25KU-1119D.xlsx` | (sheet) | M from sheet | EVANSTON PLATE TOE |
| `SS28 FLC BOM-M_SNAPLINE 6INCH WP CF 10.5.2026.xlsx` | SS28 | M | SNAPLINE 6INCH WP CF |
| `FW27 FLC BOM – M CLASSIC ALPINE 2 - 8.5.2026.xlsx` | FW27 | M | CLASSIC ALPINE 2 |

The date inside a file name is **not** auto-filled into the Date field (ambiguous D.M vs M.D). Stage is not auto-detected from the file name; the user always picks it.

## Gotchas learned the hard way

- **ExcelJS returns the master cell's value for every merged slave cell.** Scanning right of a merged label like `Gender` returned `Gender` itself (the tool showed `GENDER` / `SAMPLE NAME` for every file). Fix: skip slave cells (`mergeSlaves`) and stop if the next non-empty cell is another known label (`META_LABELS`). Test mocks must reproduce this behaviour.
- Excel stores borders on every cell of a merged range, including interior edges. Excel does not draw interior borders, so the engine draws only the outer edges (`plan.mergeOf`).
- `"` is illegal in file names, so inch marks are converted to `INCH`, not deleted.
- Seasons in the dropdown are suggestions only (`SS26` to `FW29`); the field accepts any `XX00` value and remembers the last Season/Stage in `localStorage`.

## How it was tested (the sandbox resets, so rebuild if needed)

- Unit tests in Node: extract the text between `// <pure>` and `// </pure>`, append `module.exports = {...}`, and assert name building, number/date formats, colours, ranges, header/footer parsing, text wrapping, gender/model parsing (17 cases in the last run, all passing).
- Render test in headless Chromium (Playwright): convert the real workbook with openpyxl into a JSON mock of the ExcelJS worksheet API (cells, styles, merges, column widths, images), stub `window.ExcelJS` and `window.JSZip`, call `workbookToPdf` for both modes, write the returned PDF bytes to disk, then validate with `qpdf --check` / pikepdf, render with `pdftoppm` (compare 100 to 300 dpi crops), and extract text with pdfplumber. The mock must return the master value for merged slave cells. Last run: vector 4 pages 0.44 MB, image 4 pages 10.6 MB, both valid.
- **Not tested:** the real CDN libraries (the sandbox has no network), and real Calibri metrics (the sandbox falls back to another font when measuring widths). Assumptions about ExcelJS to re-verify if something breaks: `ws.getImages()` returns objects with `imageId` and `range.tl/br` using `nativeCol/nativeRow/nativeColOff/nativeRowOff` (EMU); `wb.getImage(id)` returns `{extension, buffer}`; `ws.model.merges` is an array of range strings; `ws.pageSetup.printArea` / `printTitlesRow` / `fitToPage` / `margins`; `ws.headerFooter.oddFooter`; `wb._themes` for theme colours; `ws.getRow(r).getCell(c)` / `cell.font|fill|border|alignment|numFmt`.
- The user has run the deployed version on real files (screenshots confirmed conversion, gender/model fields, naming and the quality complaint), which is the real validation.

## Changelog (in order)

1. First version: batch convert, Season/Stage/Date, per-file Gender/Model, ZIP download, Vietnamese UI.
2. Fixed interior borders of merged cells showing through text.
3. Model name reconciliation between sheet and file name (`PLATE TOE` to `EVANSTON PLATE TOE`).
4. Fixed gender/model detection returning the label text (merged-cell bug above).
5. UI switched to English.
6. Date format in the file name changed to `MM.DD.YYYY` (dots).
7. Sharper output: R=3 / 4.5, JPEG 0.95 / 0.98, 16 MP cap, default High.
8. Smarter gender/model parsing (`splitGender`, size range and category fallbacks, `6"` to `6INCH`).
9. Blur fix: removed jsPDF, wrote an own PDF writer, added **Sharp vector** mode (default) and kept the image mode as "Exact fonts (image)". Quality select replaced by an Output select.

## Known limits and ideas

- Vector mode: fonts are look-alikes (Arial/Times/Courier), not the original Calibri. Chinese or other non-Western text is an image. Image mode: text is a picture and files are about 10 MB. Wide sheets shrink under fit-to-page, so text can still look small. That needs a print-setup change in Excel, not a tool change.
- `.xls` is not supported (ExcelJS reads only `.xlsx` / `.xlsm`).
- Picture-in-cell and shape objects are not drawn; only floating pictures are.
- Not yet done, possible next asks: auto-fill Stage and Date from file names; a Vietnamese/English toggle; merging all converted PDFs into one; a per-row Stage override; embedding a Calibri-compatible font (Carlito) for an exact look in vector mode.

## Resume checklist

1. Read this file, then open `index.html` (the `// <pure>` block holds all naming logic).
2. Ask the user for a screenshot or the failing file name before changing detection rules; add the new name to the table above.
3. After any change: syntax-check the script, re-run the Node unit tests and the Chromium render test, rebuild `bom-pdf-tool.zip`, give the git commands, and remind the user to hard refresh and clear the file list before re-testing.
