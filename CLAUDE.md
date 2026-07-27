# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, static HTML invoice generator for a massage therapist (Yusuke Komiya) invoicing Gwinganna Lifestyle Retreat. There is no build system, package manager, or test suite — `invoice_generator.html` is the entire application (HTML + CSS + vanilla JS in one file), served as a static file / installable PWA (`manifest.json`).

There is no local dev server config or CLI tooling in this repo. To work on it, edit `invoice_generator.html` directly and open it in a browser (or use a simple static file server) to test changes manually. There are no automated tests, linters, or build steps to run.

## Deployment

The repo is pushed directly (commit history shows repeated "Add files via upload" from the GitHub web UI). Changes to `invoice_generator.html` take effect wherever the file is hosted/served — there's no compilation or bundling step.

## Architecture

Everything lives in `invoice_generator.html`, structured as:

- **Two view states toggled via JS, not routing**: `#form-section` (data entry) and `#invoice-preview` (print-ready output). `generateInvoice()` / `editInvoice()` swap `display` between them. Printing uses `@media print` CSS with `.no-print` to hide chrome.
- **Photo intake → OCR → session extraction pipeline**: users add photos of a printed daily schedule sheet via Google Drive Picker (`openDrivePicker`/`pickerCallback`) or direct upload/drag-drop (`processFiles`). Images are resized/compressed client-side (`resizeImage`/`resizeImageBlob`, capped near 3MB base64) before being sent for OCR.
- **AI extraction via a Cloudflare Worker proxy**: `scanSingleImage()` posts each photo to a Cloudflare Worker (`WORKER_URL`, a top-level const alongside `CLIENT_ID`/`SCOPES`) which proxies to the Claude API (`claude-haiku-4-5-20251001`) with a prompt describing the schedule sheet format and the treatment rate table. The worker returns raw JSON session objects (date/treatment/client/amount/tip). `loadInvoiceNum()` and `openGmail()` also use `WORKER_URL` to sync the invoice number to Cloudflare KV.
- **Treatment/rate normalization**: `RATES` is the single source of truth mapping treatment name → price. `normTreatment()` fuzzy-matches OCR'd/free-text treatment strings (e.g. "intuitive", "80") onto canonical `RATES` keys; `lookupRate()` chains through it. When adding a new treatment/duration, update `RATES` (and `TREATMENTS`, which is derived from its keys) and add matching heuristics in `normTreatment()`.
- **Fallback text parser**: `parseScheduleText()` is a line-based regex parser for schedule text (date headers, time-prefixed appointment lines, trailing client name / tip lines) — a legacy/backup path alongside the AI OCR pipeline. Session objects from either path share the same shape: `{id, date, treatment, client, amount, tips, ocr}`.
- **Sessions state**: a single in-memory `sessions` array drives the editable table (`renderSessions`/`upd`/`addSession`/`removeSession`) and the printable invoice table (`generateInvoice`). Rows populated by OCR get the `ocr-filled` CSS class (blue highlight) until manually edited (`upd()` clears the `ocr` flag).
- **Invoice numbering**: stored in `localStorage` (`invoiceNum`) and mirrored to Cloudflare KV via the Worker so it can sync across devices; auto-increments each time a Gmail draft is opened (`openGmail`).
- **Output actions**: `printInvoice()` (browser print → PDF, with a computed filename via monthly invoice counters in `localStorage`), `exportCSV()`, and `openGmail()` (opens a prefilled Gmail compose window — does not send or attach anything automatically).
- **Google integration**: Google Identity Services (`accounts.google.com/gsi/client`) handles OAuth sign-in (`signIn`/`signOut`, `CLIENT_ID` hardcoded), and the Google Picker API (`apis.google.com/js/api.js`) provides the Drive file picker, scoped to `drive.readonly`.

## Known gotchas

- All business logic, styling, and markup are in one ~900-line HTML file — there's no module system, so search/edit by function name (they're all global) rather than expecting separate files.
- Secrets/config that would normally be environment variables (Google `CLIENT_ID`, Cloudflare `WORKER_URL`) are hardcoded directly in the JS at the top of the `<script>` block.
