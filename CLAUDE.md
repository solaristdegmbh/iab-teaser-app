# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

A zero-dependency, single-file static web app that generates IAB bank teaser PDFs directly in the browser. No build step, no server, no npm. Designed for GitHub Pages deployment.

## How to run

Open `index.html` directly in a browser — or serve it with any static file server:

```
# Python (for local image loading — fetch() to pdf_images/ requires a server)
python -m http.server 8080

# Node
npx serve .
```

The `pdf_images/` folder contains the two demo images loaded when "Beispiel laden" is clicked (`page1_img1.png` = logo, `page1_img2.png` = photo). Without a local server, the sample images silently fail to load but the rest of the app works.

## Architecture

Everything lives in `index.html`. There is no bundler, no framework, no external JS files.

**Data flow:**
1. User fills the `<form id="teaser-form">` on the left
2. Every `input` event fires `updatePreview()`, which:
   - Reads all form fields and writes them into the `#pdf-root` preview via `data-bind` / `data-bind-html` attributes
   - Serialises images as base64 data URLs stored in `photoDataUrl` / `logoDataUrl` module-level variables
   - Saves the full state to `localStorage` under key `iab-teaser-static-draft-v1`
3. "PDF exportieren" calls `exportPdf()`, which runs `html2canvas` on each `.page` element inside `#pdf-root` and stitches the canvases into a jsPDF document

**Binding system** (no framework, hand-rolled):
- `data-bind="fieldName"` → sets `element.textContent` from `form.elements.namedItem(fieldName).value`
- `data-bind-html="fieldName"` → sets `element.innerHTML` (used only for `title` to allow `<br>` line breaks)
- Bullet lists (left/right box, yield, conclusion) are rendered by `fillList(targetId, lines[])` from newline-separated textarea values
- Cashflow table rows are parsed from `cashflow_rows` textarea with `|`-delimited columns: `Jahr | Kapitaldienst | Einnahmen | Ueberschuss`

**PDF generation:**
- `html2canvas` renders each `.page` div at 2× scale to `#ffffff` background
- The canvas is JPEG-encoded (quality 0.96) and added to jsPDF at A4 dimensions
- The output filename comes from the `pdf_title` field, sanitised with `/[^\w\-]+/g → "_"`

**Draft persistence:**
- On every `updatePreview()` call the form values + image data URLs are written to `localStorage`
- On load, `restoreDraft()` is tried first; if nothing is saved, `applySample()` runs instead
- `sampleConfig` (hardcoded JS object) holds the full Wilmersdorfer-Strasse demo data

## Key conventions

- The preview DOM (`#pdf-root`) is the single source of truth for the PDF — what you see is what gets exported
- Images never touch a server; they are loaded via `FileReader.readAsDataURL` and stored as data URLs
- The `<title>` field uses `data-bind-html` (not `data-bind`) — it is the only field that supports inline HTML
- All German text in the UI is intentionally unescaped (e.g. "Ablauf", "Fazit") — keep it that way
