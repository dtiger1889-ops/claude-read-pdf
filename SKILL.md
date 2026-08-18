---
name: read-pdf
description: Token-efficient reading of print-to-PDF files (mobile browser "Print to PDF" output from Firefox/Brave/Chrome, or any text-layer PDF where the CONTENT is the goal, not layout/forms). Use when the user asks to read a PDF, process a PDF drop folder, or any task requires reading a .pdf whose text is what matters. NOT for form-filling, signing, or visual layout work.
---

# read-pdf -- token-efficient PDF text extraction

Mobile print-to-PDFs are 1-3MB files whose useful content is ~8-20KB of text.
Extract the text locally; never push the PDF itself through the model.

## Never do these (measured 2026-08)

- **Never read a .pdf directly with the agent's file Read tool** -- it renders
  pages as images, thousands of tokens per page for content pdftotext gets in
  hundreds.
- **Never call PDF metadata/annotation-dump tools on a browser-printed PDF** --
  pages printed from link-dense sites (Reddit, news) carry hundreds of duplicate
  link annotations from navigation chrome; one metadata call dumped ~50KB of
  JSON for a 6-page file and still truncated (659 annotations encountered).
- **Never use markdown-conversion tools as the first rung** -- deterministic
  converters are fine as a fallback if pdftotext output looks scrambled, but
  they are slower and gain nothing on prose.

## Procedure

1. **Triage (optional, for big or multi-file batches)** -- page count + producer:
   ```
   pdfinfo "file.pdf" | grep -iE "pages|producer|creator"
   ```
2. **Extract** to a scratch file (never inline for unknown sizes):
   ```
   pdftotext -q "file.pdf" scratch.txt && wc -c scratch.txt
   ```
3. **Read by size:** under ~40KB, read the whole .txt. Bigger: grep for the
   relevant sections or read with offsets. Blank tail pages (common in mobile
   prints -- one sample had 16 blank pages after 7 content pages) cost nothing.
4. **Page ranges** when only part matters: `pdftotext -f 2 -l 5 ...`

Requires poppler (`pdftotext`, `pdfinfo`). Linux: `apt install poppler-utils`;
macOS: `brew install poppler`; Windows: ships with Git for Windows' MSYS
environment, or install a poppler release and use the full path to the exe.

## Expected artifacts (not corruption -- read through them)

| Producer (from pdfinfo) | Source | Artifacts |
|---|---|---|
| cairo (Mozilla Firefox) | Firefox / Firefox mobile | Repeated per-page header/footer: title, URL, `N of M`, timestamp. Bonus: the source URL is IN the header text. |
| Skia/PDF (Chrome UA) | Brave / Chrome mobile | Occasional interleaved overlay text -- e.g. "Skip to main content" woven letter-by-letter into a body line. Meaning stays recoverable; don't re-extract. |

## Reddit prints specifically

- `Join the conversation` in the extracted text = start of the comments section.
  Comments are usually part of what the user wants -- read them.
- **Collapsed comments are simply absent from the PDF.** No extraction fixes
  that; expansion has to happen in the browser before printing. If a big
  thread's text ends right after `Join the conversation`, say so -- don't guess
  at comments that aren't there.
- Suggested-posts / "Related Answers" junk survives extraction but costs only a
  few hundred tokens -- read past it rather than building precision trims.
- Duplicate prints happen (same article printed twice under different names).
  On multi-file batches, compare the first ~200 chars before reading both.

## Fallbacks (only if pdftotext output is near-empty for pages that visibly have content)

Near-empty = under ~200 chars/page on a text article, meaning the page is
raster (screenshot-style) or the text layer failed. Escalate in order:
1. A text-oriented PDF MCP tool with explicit extraction status, if available.
2. OCR (tesseract) or page rendering on the SPECIFIC pages only -- image
   tokens, last resort.

## Links (on request only -- pdftotext drops them)

Firefox prints: the source URL is in the extracted page header, no extra work.
For in-body links, use pypdf: collect unique `/A` -> `/URI` values across pages,
then filter obvious site chrome (nav, login, notifications, recommended-post
sidebars) before showing. Measured on one Reddit print: ~54 raw links -> ~30
useful after the chrome filter.

## After reading

If the PDF came from a drop/inbox folder, move it to its long-term home once
processed (project reference folder or a dated raw archive) so the inbox stays
empty. The scratch .txt is disposable; archive the original PDF, not the txt.
