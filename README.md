# Fileora: online PDF and image tools on WordPress

**Complete implementation guide.** This package contains a working website, a WordPress plugin and a processing server. This README explains how they fit together and how to put them live, step by step.

| | |
|---|---|
| Version | 1.0, October 2026 |
| Tested with | WordPress 7.1.3, PHP 8.3, MariaDB 10.11, Ubuntu 24.04 LTS, Python 3.12/3.13, LibreOffice 24.2, Ghostscript 10.02, qpdf 11.9, Tesseract 5.3, Chromium (Playwright 1.56) |
| Tools | 37 (31 PDF and document tools, 6 image tools) plus workflows |

> **Note on the brief.** The brief this guide answers stops halfway through section 5 ("Processing progress and…"). Sections 1 to 5 follow the brief. Sections 6 to 12 cover the topics its introduction lists: hosting limits, deployment, security, performance and scalability.

---

## Contents

0. [Start here](#0-start-here)
1. [Website architecture](#1-website-architecture)
2. [Tools and features](#2-tools-and-features)
3. [WordPress plugins and technologies](#3-wordpress-plugins-and-technologies)
4. [The custom WordPress plugin](#4-the-custom-wordpress-plugin)
5. [Front end and user interface](#5-front-end-and-user-interface)
6. [Hosting requirements and limitations](#6-hosting-requirements-and-limitations)
7. [Deployment, step by step](#7-deployment-step-by-step)
8. [Security](#8-security)
9. [Performance](#9-performance)
10. [Scalability](#10-scalability)
11. [Testing](#11-testing)
12. [Operations and maintenance](#12-operations-and-maintenance)
13. [Troubleshooting](#13-troubleshooting)
14. [Reference](#14-reference)

---

## 0. Start here

### What is in this package

```
fileora/
├── README.md                    this guide
├── fileora.html                 the complete website in one file (works on its own, browser tools only)
├── build.mjs                    rebuilds fileora.html and the plugin's copy from src/
├── src/                         website source: index.html, style.css, image-tools.js, app-core.js, app-tools.js
├── wordpress/
│   ├── fileora-plugin.zip       upload this in WordPress → Plugins → Add New → Upload Plugin
│   └── fileora/                 the same plugin as source code
└── server/
    ├── app.py                   processing service (Python, FastAPI)
    ├── requirements.txt         Python libraries
    ├── install.sh               one-command installer for Ubuntu 24.04
    └── deploy/
        ├── fileora.service      systemd unit
        ├── fileora.env.example  configuration
        ├── nginx-wordpress-site.conf
        └── smoke_test.sh        checks the service end to end
```

### What is done and what you still do

| Part | Status |
|---|---|
| Website: design, 37 tools, drag and drop, progress, results, dark mode, mobile | Done and tested |
| 19 tools that run fully in the browser (Merge, Split, Sign, image tools, …) | Done and tested |
| Processing service for 18 server tools | Done. 15 tested on real files. The 2 AI tools are built but need your Claude API key, so they weren't tested here. Edit PDF needs an editor screen (2.5) |
| WordPress plugin: serves the site, tool pages, REST API, jobs, limits, admin screens, sitemap | Done and tested |
| Rent a VPS, point your domain, install, connect | **You**, sections 6 and 7 |
| Real legal pages, prices, login/sign-up flow, payment | **You** (WordPress pages and plugins, section 3.2) |
| Licensing decision for PyMuPDF and Ghostscript (AGPL) | **You**, section 3.9 |
| Edit PDF screen | Optional build, section 2.5 |

### The short path to a live site

1. Rent an Ubuntu 24.04 VPS with at least 4 GB RAM and point your domain to it (7.1).
2. Install Nginx, PHP 8.3 and MariaDB (7.2), then WordPress (7.3) and HTTPS (7.4).
3. Upload `wordpress/fileora-plugin.zip` in WordPress and activate it (7.7).
4. Copy the **Shared key** from WordPress → Fileora → Settings.
5. Copy the `server/` folder to the VPS and run `sudo FILEORA_KEY=<that key> bash install.sh` (7.5).
6. Raise the PHP and Nginx upload limits (7.6), switch WP-Cron to a real cron (7.8).
7. Run `sudo bash deploy/smoke_test.sh`, then work through the go-live checklist (7.9).

---

## 1. Website architecture

### 1.1 Components and what each one does

| Component | Runs on | Responsible for |
|---|---|---|
| **Web app** (`fileora.html`) | Visitor's browser | The whole interface. Runs 19 tools locally with pdf-lib, pdf.js and the Canvas API. Uploads files only for server tools. |
| **WordPress** | VPS, PHP 8.3 | Pages (privacy, terms, blog), users, admin, SEO, email. |
| **Fileora plugin** | Inside WordPress | Serves the app on the homepage and tool pages, provides the REST API, validates uploads, enforces limits, records jobs, streams results, admin screens. |
| **Database** | MariaDB | WordPress data plus one table, `wp_fileora_jobs`. It holds job records only, never files. |
| **Processing service** (`server/app.py`) | Same VPS, `127.0.0.1:8000` | Runs server tools as background jobs with LibreOffice, Ghostscript, qpdf, OCRmyPDF/Tesseract, Chromium, pdf2docx, PyMuPDF and pdfplumber. Deletes files automatically. |
| **External APIs** (optional) | Internet | Claude API for AI Summarizer and Translate PDF. |

### 1.2 Architecture diagram

```
                         ┌───────────────────────────────────────────┐
  Visitor's browser      │  Fileora web app (fileora.html)            │
                         │  19 browser tools: pdf-lib, pdf.js, Canvas │
                         └───────────────┬───────────────────────────┘
                     HTTPS: page,        │ only for the 18 server tools:
                     /merge-pdf/ ...     │ POST /wp-json/fileora/v1/jobs/{tool}
                                         │ GET  /wp-json/fileora/v1/jobs/{id}        (status)
                                         │ GET  /wp-json/fileora/v1/jobs/{id}/file   (download)
┌────────────────────────────────────────▼───────────────────────────────────────────┐
│ VPS (Ubuntu 24.04)                                                                  │
│  Nginx ─► PHP-FPM ─► WordPress + Fileora plugin ──────► MariaDB (wp_fileora_jobs)   │
│                         │  validate, limit, job token                               │
│                         │  HTTP + X-Fileora-Key, localhost only                     │
│                         ▼                                                           │
│              Processing service (FastAPI, 127.0.0.1:8000, systemd)                  │
│              job queue (thread pool) ─► LibreOffice · Ghostscript · qpdf            │
│                                        OCRmyPDF/Tesseract · Chromium · pdf2docx     │
│              /var/lib/fileora/jobs/<id>/  (deleted after download or 60 min)        │
└──────────────────────────────────────────┬──────────────────────────────────────────┘
                                           │ HTTPS (AI tools only)
                                           ▼
                                     Claude API
```

### 1.3 Two kinds of tools

Every tool runs in exactly one place:

- **Browser tools** run in the visitor's browser. Files are never uploaded, the server does no work and there are no usage limits. Examples: Merge, Split, Rotate, Organize, Sign, Watermark, Page numbers, Crop, Compare, PDF to JPG, PDF to Markdown, JPG to PDF, Scan to PDF, and all image tools.
- **Server tools** need software that can't run in a browser, such as LibreOffice, Tesseract or Ghostscript. They use the job API: Word/Excel/PowerPoint conversion, OCR, PDF/A, HTML to PDF, PDF to Word/Excel/PowerPoint, Protect, Unlock, Redact, Forms, AI tools. When a server is connected, Compress and Repair also run there, because Ghostscript keeps text selectable and qpdf repairs more files.

The rule for new tools: if it can be done well in the browser, do it there. It is free, private and scales by itself.

### 1.4 How the browser, WordPress and the processing service talk

| Step | From → to | Endpoint | Authentication |
|---|---|---|---|
| Submit | Browser → WordPress | `POST /wp-json/fileora/v1/jobs/{tool}` (files, options) | Public, with limits; logged-in users add `X-WP-Nonce` |
| Submit | WordPress → service | `POST http://127.0.0.1:8000/jobs/{tool}` | `X-Fileora-Key` (shared secret) |
| Status | Browser → WordPress → service | `GET …/jobs/{id}` | Browser sends `X-Fileora-Token` (random, per job) |
| Download | Browser → WordPress → service | `GET …/jobs/{id}/file` (streamed) | `X-Fileora-Token` |
| Delete | WordPress → service | `DELETE /jobs/{id}` after download | `X-Fileora-Key` |
| Health | WordPress admin → service | `GET /health` | Admins only in WordPress |

The processing service listens on `127.0.0.1` only. It is not reachable from the internet, and only WordPress holds the shared key. The browser never talks to it directly.

### 1.5 Job workflow

```
Browser                      WordPress (plugin)                   Processing service
  │ POST files + options ───►  check tool on, limits, size,
  │   (upload progress bar)    real file type ─────────────────►  save to jobs/<id>/in, queue
  │ ◄── 202 {job, token} ─────  store job row (token hash)  ◄────  202 {job}
  │                                                                worker: running → done/failed
  │ GET status (every 1–3 s) ►  check token ───────────────────►  status.json
  │ ◄── queued, 2 ahead ──────  ◄────────────────────────────────
  │ ◄── done {name, size} ────  record result in job row
  │ GET file ────────────────►  check token, stream ───────────►  result file
  │ ◄── file (streamed) ──────  mark downloaded, DELETE ───────►  delete jobs/<id>
```

### 1.6 Lifecycle of a file

| Stage | Where the file is | How long | Removed by |
|---|---|---|---|
| Browser tool | Visitor's memory only | Until the tab closes | Browser |
| Upload | PHP temp folder | During the request | PHP, automatically |
| Waiting and processing | `/var/lib/fileora/jobs/<id>/in` | Seconds to minutes | Service, as soon as the job ends (success or failure) |
| Result | `/var/lib/fileora/jobs/<id>/result` | Until downloaded | Plugin calls `DELETE` right after download |
| Not downloaded | Same | At most `FILEORA_TTL_MIN` (60 min) | Service janitor (every 5 min) |
| Text results (AI) | Status response | Once | Deleted when the result is read |
| Job record (no file) | `wp_fileora_jobs` | `retention_days` (30) | Hourly WP-Cron cleanup |

### 1.7 Database

One table, created on activation with `dbDelta()`:

| Column | Holds |
|---|---|
| `id` | Job id from the service (32 hex characters) |
| `tool`, `status` | Tool id; `queued`, `done` or `failed` |
| `token_hash` | SHA-256 of the browser's job token (the token itself is not stored) |
| `user_id`, `ip_hash` | Account (0 for visitors); keyed hash of the IP, used for daily limits |
| `files`, `bytes_in`, `bytes_out`, `result_name` | Sizes and result file name, for statistics |
| `error` | The message shown to the visitor if the job failed |
| `created_at`, `finished_at`, `downloaded_at` | Timestamps (UTC) |

No file contents, passwords or raw IP addresses are stored.

### 1.8 Why a separate Python service, not PHP

- The best libraries for PDF to Word, redaction, table extraction and OCR are Python libraries.
- OCR and conversions can take minutes. In PHP they would hit `max_execution_time` and tie up PHP-FPM workers, which slows the whole site. The service runs them in the background and WordPress answers in milliseconds.
- Shared hosts disable `exec()`. A separate service needs no special PHP permissions.
- The heavy part can later move to its own server (section 10) by changing one setting.

---

## 2. Tools and features

### 2.1 Overview

The brief's tools map to Fileora tools like this:

| Brief | Fileora tool | Runs in |
|---|---|---|
| Compress PDF | Compress PDF | Server (Ghostscript), browser fallback |
| Repair PDF | Repair PDF | Server (qpdf, Ghostscript), browser fallback |
| Edit PDF | Edit PDF | Needs an editor screen (2.5) |
| Merge PDF / Split PDF / Rotate PDF | Merge / Split / Rotate PDF | Browser |
| Convert PDF to Word / Word to PDF | PDF to Word / Word to PDF | Server |
| Convert PDF to Images | PDF to JPG (JPG or PNG) | Browser |
| Protect PDF with a password | Protect PDF | Server (qpdf) |
| Unlock PDF (with authorization) | Unlock PDF | Server (qpdf), see 2.4 |
| Add page numbers | Page numbers | Browser |
| Extract pages from PDF | Split PDF → "Page ranges" (optionally into one file) | Browser |
| Remove PDF pages | Organize PDF (delete pages) | Browser |
| Compress / Resize / Crop Image | Compress / Resize / Crop image | Browser |
| Convert Image Formats, JPG ↔ PNG | Convert image, JPG to PNG, PNG to JPG | Browser |
| Convert Image to PDF | JPG to PDF (any image) | Browser |
| Word / Excel / PowerPoint to PDF | Word / Excel / PowerPoint to PDF | Server (LibreOffice) |
| Basic document conversion utilities | PDF to Excel, PDF to PowerPoint, PDF to Markdown, HTML to PDF, PDF to PDF/A, OCR PDF | Server / browser |

Fileora also includes Organize, Crop PDF, Watermark, Sign, PDF Forms, Redact, Compare, Scan to PDF, AI Summarizer and Translate PDF.

### 2.2 Every tool in detail

**PDF tools that run in the browser** (pdf-lib, pdf.js; no upload)

| Tool | What it does | Limits | Output |
|---|---|---|---|
| Merge PDF | Joins PDFs in the order shown; drag to reorder | Password-protected files must be unlocked first | One PDF |
| Split PDF | Page ranges (`1-3, 5, 8-`), every page, or fixed chunks; ranges can go into one file (extract) | Same | One PDF per range, ZIP if several |
| Organize PDF | Page thumbnails: reorder, rotate, delete pages | Very large files (hundreds of pages) take memory on phones | One PDF |
| Rotate PDF | 90°, 180°, 270° for all, odd or even pages | Same | One PDF per file |
| Crop PDF | Trims margins in mm with a live preview | Sets the visible area; hidden content stays in the file | One PDF per file |
| Page numbers | 6 positions, 4 formats, start number, skip cover | Latin digits and text (Helvetica) | One PDF per file |
| Watermark | Text, size, angle, opacity, colour, centred or tiled, live preview | Drawn as an image, so any language works; not searchable | One PDF per file |
| Sign PDF | Draw or type a signature, choose pages and position, optional date, live preview | A visual signature, not a certificate-based digital signature | One PDF |
| Compare PDF | Line-by-line text differences with page numbers | Compares text, not layout; scans need OCR first | On-screen diff plus a .txt report |
| PDF to JPG | Each page as JPG or PNG at 96, 150 or 300 dpi | Large pages at 300 dpi use a lot of memory | Images, ZIP if several |
| PDF to Markdown | Headings, paragraphs and lists from the text | Tables and multi-column layouts become plain text | .md file and preview |
| Compress PDF (no server) | Light: lossless re-save. Balanced/strong: pages become JPEG images | Balanced/strong remove selectable text | One PDF per file |
| Repair PDF (no server) | pdf-lib rebuild, then pdf.js page images as a fallback | Fallback output has no selectable text | One PDF |

**Conversion tools**

| Tool | Runs in | Engine | Limits | Output |
|---|---|---|---|---|
| Word to PDF | Server | LibreOffice headless | Fonts must be installed (section 7.5 installs common ones); complex Word features can shift | PDF per file |
| PowerPoint to PDF | Server | LibreOffice | Animations and video are dropped; option: speaker notes pages | PDF per file |
| Excel to PDF | Server | LibreOffice | "Fit columns" puts each sheet on one page; very wide sheets get small | PDF per file |
| HTML to PDF | Server | Chromium (Playwright) | Pages behind logins can't be captured; private addresses are blocked | PDF |
| JPG to PDF | Browser | pdf-lib | Any image the browser can open; EXIF rotation applied | One PDF |
| Scan to PDF | Browser | Camera + Canvas | Has a "document" filter; it does not straighten perspective | One PDF |
| PDF to Word | Server | pdf2docx (+ OCR for scans) | Good for text and simple tables; complex layouts need manual fixes | .docx |
| PDF to Excel | Server | pdfplumber + openpyxl | Finds ruled tables, then text-aligned tables; scans need OCR | .xlsx (sheet per page or one sheet) |
| PDF to PowerPoint | Server | PyMuPDF + python-pptx | Each page becomes a picture on a slide (not editable text) | .pptx |
| PDF to PDF/A | Server | OCRmyPDF + Ghostscript | PDF/A-1b, 2b, 3b | PDF |
| OCR PDF | Server | OCRmyPDF + Tesseract | One language per job; install language packs (7.5) | Searchable PDF or .txt |

**Edit, security and AI tools**

| Tool | Runs in | Engine | Limits | Output |
|---|---|---|---|---|
| Edit PDF | Not built yet | See 2.5 | Off by default | – |
| PDF Forms | Server | PyMuPDF | Basic detection: underscore lines, drawn lines, ☐ boxes | Fillable PDF |
| Protect PDF | Server | qpdf, AES-256 | Option to block printing and copying | PDF per file |
| Unlock PDF | Server | qpdf | Needs the password if the file needs one to open; see 2.4 | PDF per file |
| Redact PDF | Server | PyMuPDF | Removes the text under the box for good; finds words, emails, phone numbers | PDF |
| AI Summarizer | Server | Claude API | Needs an API key; text PDFs only (OCR scans first) | Text on screen + .txt |
| Translate PDF | Server | Claude API | Returns translated text, not a re-laid-out PDF (see 2.6) | Text on screen + .txt |

**Image tools** (all in the browser, `src/image-tools.js`)

| Tool | What it does | Limits | Output |
|---|---|---|---|
| Compress image | Quality slider, optional largest side, save as same type, JPG or WebP | PNG quality setting isn't supported by browsers, so PNG photos are saved as WebP | Image per file |
| Resize image | By pixels (keep ratio) or percentage | Upscaling doesn't add detail | Image per file |
| Convert image | JPG, PNG, WebP; reads anything the browser can open | HEIC opens in Safari only; add `heic2any` for other browsers | Image per file |
| JPG to PNG / PNG to JPG | Shortcuts for the two most searched conversions | PNG to JPG fills transparency with white | Image per file |
| Crop image | Shape (1:1, 4:5, 4:3, 3:2, 16:9, original), size and position with live preview | Preset positions, no free drag box (add Cropper.js for that) | Image |

### 2.3 PDF Repair: what can and can't be fixed

A PDF is a set of numbered objects (pages, fonts, images) plus a **cross-reference table** that says where each object starts. Most "damaged PDF" errors are about that table, not the content.

**Usually repairable**
- Broken or missing cross-reference table, for example after a download or email attachment was cut short at the very end, or an editor wrote wrong offsets. qpdf rebuilds the table by scanning the whole file for objects.
- Wrong object lengths, stray bytes before `%PDF`, duplicated or out-of-order objects, a missing `%%EOF`.
- Small problems in individual objects (broken references, odd syntax). Ghostscript reads what it can and writes a clean new file.

**Recovery methods, in the order Fileora tries them**

1. **qpdf** (`qpdf --warning-exit-0 in.pdf out.pdf`): reconstructs the cross-reference table and rewrites every object. Lossless. Tested here on a PDF with a corrupted `startxref`: all 3 pages were recovered.
2. **Ghostscript** (`gs -sDEVICE=pdfwrite`): re-interprets every page that still renders and writes a new PDF. Keeps text in most cases; drops what it can't read.
3. **Browser fallback** (no server): pdf-lib rebuild, then pdf.js renders each page to an image. Always readable, but text is no longer selectable.

**Why some files can't be recovered**
- **Missing data:** if the file was truncated early, the page contents simply aren't in the file. No tool can recreate them.
- **Damaged compressed streams:** page content is usually compressed. If bytes inside a stream are corrupted, the rest of that stream can't be decompressed.
- **Encryption:** a damaged encrypted file can't be decrypted, and a file with a forgotten password can't be opened.
- **Not a PDF:** a renamed Word or HTML file, a zero-filled file from a failed disk, or malware with a .pdf name. Fileora rejects these on upload by checking the first bytes (section 8).

### 2.4 Unlock PDF and authorization

PDFs have two kinds of protection, and qpdf reports which one a file has:

| Protection | What it means | What Unlock does |
|---|---|---|
| Open password ("user password") | The file can't be opened without it | Unlocks only if the visitor types the correct password |
| Restrictions only ("owner password") | Opens normally, but printing or copying is blocked | Removes the restrictions |
| None | – | Says the file isn't protected |

Fileora never tries to guess or crack passwords. Before running, the visitor must tick **"I own this file or have permission to unlock it"**. Put the matching rule in your terms of use: users may only unlock documents they own or are authorized to change. Keep the job records (section 1.7) so you can respond to abuse reports.

### 2.5 Edit PDF: the options

Editing the text that is already in a PDF is genuinely hard, because PDFs store positioned glyphs, not paragraphs. Pick one option:

| Option | What users can do | Cost | Effort |
|---|---|---|---|
| **pdf.js viewer with its built-in editor** (recommended first step) | Add text, freehand drawing, images (stamps) and highlights, then save the PDF. The viewer saves with `pdfDocument.saveDocument()`. Use a current pdf.js release; the app's 3.11 copy is for reading pages. | Free (Apache-2.0) | Medium: embed the pdf.js viewer as a tool screen |
| Commercial PDF SDK (for example Apryse, Nutrient) | Also edit existing text, forms, redaction UI | Paid licence | Medium |
| Server conversion round trip (PDF → Word → PDF) | Edit in Word | Free | Poor layout fidelity |

When the editor screen exists, turn on **Edit PDF** in Fileora → Settings and change its tool entry in `src/app-core.js` to open your editor.

### 2.6 Tools that need special software, APIs or infrastructure

| Needs | Tools |
|---|---|
| Nothing (browser) | 19 tools listed in 1.3 |
| LibreOffice on the server | Word, PowerPoint, Excel to PDF |
| Tesseract + OCRmyPDF + Ghostscript | OCR PDF, PDF to PDF/A, OCR step of PDF to Word |
| qpdf | Protect, Unlock, Repair |
| Ghostscript | Compress (server), Repair fallback, PDF/A |
| Chromium | HTML to PDF |
| Python libraries | PDF to Word, Excel, PowerPoint, Redact, Forms |
| Paid external API (Claude) | AI Summarizer, Translate PDF |
| Commercial SDK or own development | Edit PDF (existing-text editing), layout-preserving translation |

---

## 3. WordPress plugins and technologies

Rule used here: one tool per job, no overlaps. Licences were checked against the projects' own pages in October 2026; links are in each table.

### 3.1 Essential WordPress plugins

| Name | Purpose | Licence / cost | Key needed | Runs on |
|---|---|---|---|---|
| **Fileora** (custom, included) | Everything Fileora needs (section 4) | Yours (GPL-2.0-or-later header) | Generates its own shared key | WordPress |

That is the only essential plugin. Fileora does not need a page builder, a forms plugin or a file-upload plugin.

### 3.2 Optional WordPress plugins (pick at most one per row)

| Need | Choose one of | Licence / cost | Key needed | Notes |
|---|---|---|---|---|
| Backups | UpdraftPlus | Free, paid add-ons | Remote storage login | Back up the database and `wp-content`. Job files are never in backups. |
| Firewall / login protection | Wordfence, or Cloudflare in front of the site | Free, paid tiers | Cloudflare account | Don't run two firewalls. |
| Page cache | LiteSpeed Cache (LiteSpeed servers) or WP Super Cache (Nginx/Apache), or Nginx FastCGI cache | Free | No | Never cache `/wp-json/`. The app page is cacheable. |
| SEO | Rank Math or Yoast SEO | Free, paid tiers | No | For your blog and pages. Fileora's own pages set their own title, description and canonical; turn off the SEO plugin's sitemap or keep WordPress's (Fileora adds its tool pages to `wp-sitemap.xml`). |
| Email delivery | WP Mail SMTP | Free, paid tiers | SMTP or API key | Needed once you send account or contact emails. |
| Bot protection on forms | Simple Cloudflare Turnstile | Free | Turnstile site + secret key | Login, registration, comments. For Fileora's own upload endpoint, see 8.3. |
| Paid "Pro" plans | Paid Memberships Pro (free core + paid add-ons), or WooCommerce with a subscriptions extension (paid) | Free / paid | Payment gateway keys | Connect to Fileora with two filters (4.5). |

### 3.3 Custom WordPress development (the Fileora plugin)

| Module | File | Job |
|---|---|---|
| Bootstrap | `fileora.php`, `class-fileora-plugin.php` | Loads modules, activation (table, key, cron) |
| Tool registry | `class-fileora-tools.php` | Server tools, accepted file types, per-tool file limits |
| Settings | `class-fileora-settings.php` | Options, validation, defaults |
| Front end | `class-fileora-frontend.php` | Serves the app, tool pages, SEO tags, sitemap |
| REST API | `class-fileora-rest.php` | Submit, status, download, health |
| Processor client | `class-fileora-processor.php` | Talks to the processing service (cURL streaming) |
| Security | `class-fileora-security.php` | Upload checks, file-type sniffing, IP hashing, job tokens |
| Jobs | `class-fileora-jobs.php` | Database table, limits, statistics |
| Cleanup | `class-fileora-cleanup.php` | Hourly WP-Cron housekeeping |
| Admin | `class-fileora-admin.php`, `admin/views/*.php` | Dashboard and Settings screens |

### 3.4 Backend processing libraries (Python, on the processing server)

| Library | Purpose | Licence | Essential |
|---|---|---|---|
| [FastAPI](https://fastapi.tiangolo.com) + Uvicorn + python-multipart | HTTP service, uploads | MIT / BSD | Yes |
| [OCRmyPDF](https://pypi.org/project/ocrmypdf/) | OCR and PDF/A (drives Tesseract and Ghostscript) | MPL-2.0 | For OCR, PDF/A |
| pikepdf | Checks repaired PDFs (comes with OCRmyPDF) | MPL-2.0 | Yes |
| [PyMuPDF](https://pypi.org/project/PyMuPDF/) | Redact, Forms, PDF pages to images | **AGPL-3.0 or Artifex commercial** | For Redact, Forms, PDF to PowerPoint |
| [pdf2docx](https://pdf2docx.readthedocs.io/en/latest/license.html) | PDF to Word | MIT (but requires PyMuPDF) | For PDF to Word |
| pdfplumber, openpyxl | PDF to Excel | MIT | For PDF to Excel |
| python-pptx | Writes .pptx | MIT | For PDF to PowerPoint |
| Playwright (Python) | Drives headless Chromium | Apache-2.0 | For HTML to PDF |
| anthropic | Claude API client | MIT | For AI tools |

### 3.5 Server software dependencies

| Software | Purpose | Licence | Installed by |
|---|---|---|---|
| Ubuntu 24.04 LTS | Operating system | Free | Your VPS provider |
| Nginx | Web server | BSD-2-Clause | 7.2 |
| PHP 8.3 (FPM, curl, mysql, xml, mbstring, zip, gd, intl) | Runs WordPress | PHP License | 7.2 |
| MariaDB 10.11 | Database | GPL-2.0 | 7.2 |
| LibreOffice 24.2 (writer, calc, impress) | Office to PDF | MPL-2.0 | `install.sh` |
| [Ghostscript](https://ghostscript.com/licensing) 10.x | Compress, PDF/A, repair | **AGPL-3.0 or Artifex commercial** | `install.sh` |
| qpdf 11 | Protect, unlock, repair | Apache-2.0 | `install.sh` |
| Tesseract 5 + language packs | OCR engine | Apache-2.0 | `install.sh` |
| unpaper, pngquant | Cleaner scans, smaller OCR output | GPL-2.0 / GPL-3.0 (used as separate programs) | `install.sh` |
| Chromium (via Playwright) | HTML to PDF | BSD-3-Clause and others | `install.sh` |
| Noto, Liberation, Carlito, Caladea, DejaVu fonts | Correct fonts in conversions (Hindi, CJK, Calibri/Cambria metrics) | OFL / Apache-2.0 | `install.sh` |
| Certbot | Free HTTPS certificates | Apache-2.0 | 7.4 |

### 3.6 Browser libraries (front end)

| Library | Purpose | Licence | Loaded |
|---|---|---|---|
| [GSAP](https://gsap.com/community/standard-license/) 3.12 + Flip | Animations | GSAP Standard License: free, including commercial use and all plugins. Not allowed in visual no-code animation builders that compete with Webflow (Fileora isn't one). | cdnjs |
| pdf-lib 1.17 | Create and edit PDFs in the browser | MIT | On demand, cdnjs (unpkg fallback) |
| pdf.js 3.11 | Read and render PDFs | Apache-2.0 | On demand, cdnjs (jsDelivr fallback) |
| JSZip 3.10 | "Download all as ZIP" | MIT or GPL-3.0 (dual) | On demand |
| Bricolage Grotesque, Geist, Caveat | Fonts | SIL OFL | Google Fonts |

To avoid third-party CDNs (privacy, offline intranets), download these files into the plugin and change the URLs at the top of `src/app-tools.js` (`LIBS`) and in `src/index.html`.

### 3.7 External APIs and paid services

| Service | Used for | Cost | Key |
|---|---|---|---|
| Claude API (Anthropic) | AI Summarizer, Translate PDF | Pay per use. The default model is `claude-haiku-5-5`, the cheapest current model; set `FILEORA_AI_MODEL` to change it. See [models and prices](https://platform.claude.com/docs/en/models/overview). | `ANTHROPIC_API_KEY` in `/etc/fileora/fileora.env` |
| Cloudflare (optional) | DNS, CDN, firewall, Turnstile | Free plan available | Account |
| Commercial PDF SDK or conversion API (optional) | Existing-text editing, layout-preserving translation, highest-fidelity Office conversion | Paid | Vendor key |

### 3.8 Evaluated and not used

| Technology | Why not |
|---|---|
| **Imagick / ImageMagick** for PDFs | ImageMagick hands PDFs to Ghostscript and turns pages into pictures, so text is lost. Ubuntu's default ImageMagick security policy also blocks PDF. Fine for images; Fileora does image work in the browser instead. Imagick is still a good choice if you ever need server-side image processing in PHP (for example HEIC on every browser). |
| PHP `exec()` of LibreOffice/Ghostscript from WordPress | Blocks PHP workers for minutes, hits timeouts, is disabled on most hosts, and is risky with user input. |
| FPDI / TCPDF / mPDF (PHP) | They create PDFs well but can't merge every PDF version for free, compress, OCR or convert Office files. |
| A paid conversion API as the main engine | Works on shared hosting, but every file goes to a third party and costs money per conversion. Kept as an option (3.7). |
| Docker | Fine if you already use it; plain systemd keeps the setup simpler for one VPS. |

### 3.9 Licensing: AGPL components

**This is not legal advice.** Two components are dual-licensed by Artifex under the **AGPL-3.0** or a paid commercial licence:

- **PyMuPDF**, used directly by `server/app.py` for Redact, Forms and PDF to PowerPoint, and required by pdf2docx for PDF to Word.
- **Ghostscript**, run as a separate program for Compress, PDF/A and Repair.

The AGPL is a copyleft licence written for software offered over a network. Your choices:

1. **Publish the processing service's source code** (for example on GitHub, with a link in your footer) under the AGPL. This is the usual route for the program that imports PyMuPDF. Your WordPress theme, the web app and the plugin talk to the service over HTTP.
2. **Buy commercial licences** from Artifex if you want to keep the service closed.
3. **Replace the components:**
   - pypdfium2 (Apache/BSD) for page images.
   - Image-based redaction (render the page, black out, rebuild) instead of PyMuPDF's text removal.
   - LibreOffice or a paid API for PDF to Word.
   - Turn off Forms.

Ask a lawyer which applies to your business. Every other component in this guide uses a permissive licence (MIT, BSD, Apache-2.0, MPL-2.0, OFL) or is used as a separate program.

---

## 4. The custom WordPress plugin

The plugin is complete in `wordpress/fileora/`. Use it as is, or as the reference for your own.

### 4.1 Folder structure

```
fileora/
├── fileora.php                        main file: header, constants, includes, activation hooks
├── uninstall.php                      removes table, options and cron when the plugin is deleted
├── includes/
│   ├── class-fileora-plugin.php       boot, activate, deactivate
│   ├── class-fileora-tools.php        server tool registry (tool modules)
│   ├── class-fileora-settings.php     configuration
│   ├── class-fileora-frontend.php     serves the app and tool pages, SEO tags
│   ├── class-fileora-sitemap.php      adds tool pages to wp-sitemap.xml
│   ├── class-fileora-rest.php         REST API handlers
│   ├── class-fileora-processor.php    processing-service client (upload, status, streaming download)
│   ├── class-fileora-security.php     validation, file sniffing, tokens, IP hashing
│   ├── class-fileora-jobs.php         database table, limits, statistics
│   ├── class-fileora-cleanup.php      background cleanup (WP-Cron)
│   ├── class-fileora-log.php          logging
│   └── class-fileora-admin.php        admin menu and screens
├── admin/views/
│   ├── dashboard.php                  server status, usage, recent failures
│   └── settings.php                   all settings
└── public/
    ├── fileora.html                   the web app (CSS and JS inside), written by build.mjs
    └── tools.json                     tool names and descriptions for tool pages, written by build.mjs
```

### 4.2 How it works

1. **Activation** creates `wp_fileora_jobs`, saves default settings, generates a 48-character shared key and schedules `fileora_cleanup` hourly.
2. **Page requests:** on `template_redirect`, the plugin checks whether the address is the app (homepage or chosen path) or a tool page (`/merge-pdf/`). If so, it outputs `public/fileora.html` with:
   - the settings injected into `CONFIG`;
   - the page's own title, description and canonical link;
   - WordPress's login links.
   It then stops. Any other address goes to your theme as usual.
3. **REST requests** go to `Fileora_REST` (4.3).
4. **Hourly cleanup** closes jobs that never finished and deletes old records.
5. **Admin:** Fileora → Dashboard shows the service's health and 30-day usage. Fileora → Settings holds everything else.

### 4.3 Key code

These excerpts come from the included files.

**Registering REST endpoints** (`class-fileora-rest.php`)

```php
register_rest_route(
	self::NS,
	'/jobs/(?P<tool>[a-z][a-z0-9]{1,19})',
	array(
		'methods'             => WP_REST_Server::CREATABLE,
		'callback'            => array( __CLASS__, 'submit' ),
		'permission_callback' => '__return_true', // public tools; limits are enforced in submit()
	)
);
register_rest_route(
	self::NS,
	'/jobs/(?P<id>[a-f0-9]{32})',
	array(
		'methods'             => WP_REST_Server::READABLE,
		'callback'            => array( __CLASS__, 'status' ),
		'permission_callback' => '__return_true', // access is checked with the job token
	)
);
```

**Validating uploads** (`class-fileora-security.php`): upload error, size, extension, then the file's real first bytes, so a renamed `.exe` can't pass as a PDF.

```php
$zip = 0 === strpos( $head, "PK\x03\x04" );
$ole = 0 === strpos( $head, "\xD0\xCF\x11\xE0\xA1\xB1\x1A\xE1" );
switch ( $ext ) {
	case 'pdf':
		return false !== strpos( $head, '%PDF-' );
	case 'docx':
	case 'xlsx':
	case 'pptx':
```

**Submitting a processing job** (`class-fileora-processor.php`): the WordPress HTTP API can't send multipart file uploads, so this uses cURL with `CURLFile`. Files stream from PHP's temp folder without being read into memory.

```php
$fields = array( 'options' => $options );
foreach ( array_values( $files ) as $i => $file ) {
	$fields[ "files[{$i}]" ] = new CURLFile( $file['tmp_name'], 'application/octet-stream', sanitize_file_name( $file['name'] ) );
}
```

**Checking status** (`class-fileora-rest.php`): the browser must send the job token it received. Only the token's hash is stored.

```php
private static function authorized_job( WP_REST_Request $request ) {
	$row = Fileora_Jobs::find( (string) $request['id'] );
	if ( ! $row || ! Fileora_Security::token_matches( $request->get_header( 'x_fileora_token' ), $row->token_hash ) ) {
		return self::error( 'This task wasn’t found. Please run it again.', 404 );
	}
	return $row;
}
```

**Returning a downloadable file** (`class-fileora-processor.php`): `CURLOPT_WRITEFUNCTION` sends each chunk to the browser as it arrives, so a 500 MB result never sits in PHP memory. After the download, the REST handler marks the job downloaded and tells the service to delete it.

**Handling errors and timeouts**
- Every failure returns JSON `{"error": "…"}` with a fitting HTTP status (400, 403, 404, 410, 413, 415, 429, 502, 503). The app shows the message as is, so messages are written for visitors.
- The service turns tool failures into readable messages, for example "That password doesn't open file.pdf." It caps each tool's run time (4 to 15 minutes) and returns "This file took too long to process".
- The app polls status every 1 to 3 seconds and gives up after 30 minutes. A dropped connection during polling is retried.
- The plugin's cURL calls use a 5-second connect timeout. If the service is down, visitors see "The processing server isn't responding."

**Scheduling temporary file cleanup**
- Files are deleted by the service: inputs when a job ends, results after download (plugin `DELETE`), anything else after `FILEORA_TTL_MIN` by its janitor thread.
- Records are cleaned by WP-Cron (`class-fileora-cleanup.php`):

```php
public static function schedule() {
	if ( ! wp_next_scheduled( self::HOOK ) ) {
		wp_schedule_event( time() + 300, 'hourly', self::HOOK );
	}
}
```

### 4.4 Building your own plugin step by step

If you write your own instead of using the included one, build it in this order and test each step:

1. **Main file, activation and settings** with a generated shared key.
2. **Processor client:** `health()` first, and check it from WP-CLI with `wp eval 'print_r(Fileora_Processor::health());'`.
3. **Submit endpoint**, with validation (4.3) and the jobs table.
4. **Status endpoint** with token check, then the **download** endpoint with streaming and delete.
5. **Front end:** serve the app with injected `CONFIG`, then tool pages and the sitemap.
6. **Limits, cleanup cron, admin screens and uninstall.**
7. Run the tests in section 11.

### 4.5 Hooks for developers

| Hook | Type | Use |
|---|---|---|
| `fileora_daily_limit` | filter `($limit, $user_id)` | Different limits per user; return 0 for unlimited |
| `fileora_max_mb` | filter `($mb, $user_id)` | Bigger files for members (also updates the app's limit) |
| `fileora_pre_submit` | filter `($true, $request, $tool)` | Return a `WP_Error` to refuse a job (bots, members only) |
| `fileora_client_ip` | filter `($ip)` | Real visitor IP behind a proxy (prefer Nginx `real_ip`, see 8.3) |
| `fileora_server_tools` | filter `($tools)` | Add or change server tools |
| `fileora_config` | filter `($config)` | Change what the app receives in `CONFIG` |
| `fileora_html` | filter `($html, $tool)` | Change the page HTML, e.g. add analytics |
| `fileora_job_created` | action `($job_id, $tool, $user_id)` | Logging, analytics |
| `fileora_job_finished` | action `($job_id, $tool, $status)` | Notifications, statistics |

**Example: unlimited tasks and 1 GB files for Pro members.** Put this in a small custom plugin or your theme's `functions.php`.

```php
add_filter( 'fileora_daily_limit', function ( $limit, $user_id ) {
	return ( $user_id && user_can( $user_id, 'fileora_pro' ) ) ? 0 : $limit;
}, 10, 2 );

add_filter( 'fileora_max_mb', function ( $mb, $user_id ) {
	return ( $user_id && user_can( $user_id, 'fileora_pro' ) ) ? 1024 : $mb;
}, 10, 2 );
```

Give the `fileora_pro` capability to your paid members' role in your membership plugin. Raise `FILEORA_MAX_MB`, PHP and Nginx limits to match.

### 4.6 Adding a new tool

1. **Processing service** (`server/app.py`): write a function `def mytool(files, opts, job, url) -> list[Path]` and add it to `TOOLS` with its accepted input group.
2. **Plugin** (`class-fileora-tools.php`): add the id, label, input group, file limit and dependency to `all()`, or use the `fileora_server_tools` filter.
3. **Web app** (`src/app-core.js`): add an entry to `TOOLS` with its name, category, icon, description, options and button text. Leave out `client: true` so it uses the server. Run `node build.mjs`.
4. A browser-only tool needs only step 3, plus a function `P.mytool` in `src/app-tools.js`.

---

## 5. Front end and user interface

### 5.1 What is built

| Brief | Fileora |
|---|---|
| Professional homepage with tool categories | Hero with animated "light table" drop zone and quick picks. All tools in 9 categories with filter tabs, search filter, "In browser" filter, favourites, recently used. |
| Individual pages for each tool | Every tool has its own address (`/merge-pdf/`, `/compress-image/`) with its own title, description and canonical link, listed in `wp-sitemap.xml`. Opening a tool on the homepage changes the address without reloading; the back button closes it. |
| Drag-and-drop uploads | Drop on the hero, anywhere on the page (Fileora suggests tools for that file type), or inside a tool. |
| File selection and upload progress | File cards with thumbnails, page counts, reorder by drag or buttons, remove, add more. Upload progress in % for server tools. |
| Processing progress | Per-page progress for browser tools. Server tools show "Waiting in line, 2 ahead of you", then the tool's own wording. |
| Download | One-click download, preview, "Download all as ZIP", copy text (Markdown, AI). "Keep going with this file" passes the result to the next tool. |
| Errors | Plain messages that say what happened and what to do, with "Try again" and "Repair the PDF" where useful. |
| Live previews | Signature, watermark, page numbers, PDF crop and image crop are drawn on the page before running. |
| Responsive | Desktop, tablet and phone layouts; full-screen tool view with a sticky action button on phones. |
| Extras | Command palette (Ctrl K / ⌘K), workflows (chain tools and save), light and dark themes, keyboard navigation, focus management in dialogs, reduced-motion support. |

### 5.2 How the app is organized

| File | Contains |
|---|---|
| `src/index.html` | Page structure, icon sprite, overlays |
| `src/style.css` | Design tokens (colours, fonts) in `:root` and `[data-theme="dark"]`, all components, breakpoints at 1180, 1020, 760 and 560 px |
| `src/image-tools.js` | Image compress, resize, convert, crop (5.4) |
| `src/app-core.js` | `CONFIG`, categories (`CATS`), the tool list (`TOOLS`), catalog, search, menus, routing |
| `src/app-tools.js` | Tool screen, options, all browser processors (`P.merge`, `P.sign`, …), job API client, workflows, hero animation |
| `build.mjs` | Joins everything into `fileora.html`, copies it to the plugin and writes `tools.json` |

After any change: `node build.mjs` (Node.js 18 or newer, no npm packages needed), then upload the plugin's `public/` folder again.

### 5.3 Changing text and design

- **Tool names, descriptions, options and button text:** `TOOLS` in `src/app-core.js`. Tool page titles and the sitemap follow automatically.
- **Colours:** the tokens at the top of `src/style.css`. Brand colour `--hl` (yellow), ink `--ink`, one colour per category (`--c-organize`, …).
- **Homepage sections, FAQ, footer, Pro table:** `src/index.html`. The Pro table's limits and the "deleted within 2 hours" wording are examples. Change them to your real plans and policy.
- **Links** like `/pricing`, `/privacy`, `/terms`, `/about` point to WordPress pages you create with those slugs.

### 5.4 Image tools code

`src/image-tools.js` is a small module you can also use outside Fileora:

```js
const out = await FileoraImage.compress(file, { quality: 0.7, maxSide: 2400 }); // smaller file
const out = await FileoraImage.resize(file, { width: 1080 });                   // keeps the ratio
const out = await FileoraImage.convert(file, 'image/png');                      // JPG -> PNG
const out = await FileoraImage.crop(file, { x: 0, y: 0, width: 800, height: 600 });
// out = { blob, name, width, height }  ->  URL.createObjectURL(out.blob) for a download link
```

It decodes with `createImageBitmap` (applying phone EXIF rotation), draws on a canvas and encodes with `canvas.toBlob`. Tested results: a 62 KB photo compressed to 28 KB at 72%, converted to WebP at 7 KB, and cropped to 16:9 at exactly 1200 × 675 px.

Optional upgrades:
- **heic2any** (MIT) to open iPhone HEIC photos in every browser.
- **Cropper.js** (MIT) for a free-drag crop box.
- **libvips or pngquant on the server** for stronger PNG compression.

### 5.5 SEO

- One page per tool with a unique `<title>`, meta description, canonical URL, `og:title` and `og:description`.
- `wp-sitemap.xml` includes the app and all tool pages. Submit it in Google Search Console.
- Each tool page is a complete HTML document from the server with its own title and description. The tool list and tool screen are drawn by JavaScript, which Google renders. For stronger rankings, add a short paragraph of static text per tool with the `fileora_html` filter (4.5).
- Add a blog (WordPress posts) with "how to" articles that link to tool pages. This is how most tools sites get traffic.

### 5.6 Accessibility and front-end performance

- **Accessibility:** buttons and links are real elements, every icon button has a label, dialogs trap focus and return it on close, text colours meet WCAG AA contrast (4.5:1) in both themes, and motion respects `prefers-reduced-motion`.
- **Front-end performance:** heavy libraries (pdf-lib, pdf.js, JSZip) load only when a tool needs them. The page itself is one file of about 230 KB (about 65 KB gzipped).

---

## 6. Hosting requirements and limitations

### 6.1 What works where

| Hosting | Browser tools (19) | Server tools (18) | Verdict |
|---|---|---|---|
| Static hosting (just `fileora.html`) | Yes | No | Good for a demo |
| Shared WordPress hosting | Yes | No: no LibreOffice/Tesseract, `exec()` disabled, low time limits | Browser tools only. Turn off server tools in Settings. |
| Managed WordPress hosting | Yes | Only if the processing service runs on a separate VPS (section 10) | Possible, but more moving parts |
| **VPS, Ubuntu 24.04** (recommended) | Yes | Yes | Everything on one server |

### 6.2 Server size

| Traffic | VPS | `FILEORA_WORKERS` |
|---|---|---|
| Launch, up to about 500 server tasks a day | 2 vCPU, 4 GB RAM, 80 GB SSD, 2 GB swap | 2 |
| Growing, about 5,000 a day | 4 vCPU, 8 GB RAM | 3–4 |
| Larger | Move processing to its own server(s) (section 10) | – |

Memory is the limit that matters: LibreOffice, OCR and Chromium each use several hundred MB per running job. Keep about 2 GB of RAM per parallel job. Browser tools cost the server nothing, so these numbers only count server tools.

---

## 7. Deployment, step by step

All commands are for Ubuntu 24.04 LTS, run as a user with `sudo`. Replace `example.com` with your domain.

### 7.1 Prepare the server

1. Create the VPS (section 6.2). Point your domain's `A` record, and `www` if you use it, to the server's IP address.
2. Update the system, add a firewall and swap:

```bash
sudo apt update && sudo apt -y upgrade
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 7.2 Nginx, PHP and MariaDB

```bash
sudo apt install -y nginx mariadb-server unzip \
  php8.3-fpm php8.3-mysql php8.3-curl php8.3-xml php8.3-mbstring php8.3-zip php8.3-gd php8.3-intl php8.3-imagick
sudo mysql_secure_installation
sudo mysql -e "CREATE DATABASE fileora DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'fileora'@'localhost' IDENTIFIED BY 'CHANGE-THIS-PASSWORD';
GRANT ALL ON fileora.* TO 'fileora'@'localhost'; FLUSH PRIVILEGES;"
sudo ufw allow OpenSSH && sudo ufw allow 'Nginx Full' && sudo ufw --force enable
```

The firewall opens only SSH, HTTP and HTTPS. Port 8000 stays closed; the processing service also listens on 127.0.0.1 only.

### 7.3 WordPress

```bash
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x wp-cli.phar && sudo mv wp-cli.phar /usr/local/bin/wp
sudo mkdir -p /var/www/fileora && sudo chown www-data:www-data /var/www/fileora
cd /var/www/fileora
sudo -u www-data wp core download
sudo -u www-data wp config create --dbname=fileora --dbuser=fileora --dbpass='CHANGE-THIS-PASSWORD'
sudo -u www-data wp core install --url=https://example.com --title=Fileora \
  --admin_user=YOUR-ADMIN --admin_password='A-LONG-PASSWORD' --admin_email=you@example.com
sudo -u www-data wp rewrite structure '/%postname%/'
```

Copy `server/deploy/nginx-wordpress-site.conf` to `/etc/nginx/sites-available/fileora`, set `server_name` and `root`, then:

```bash
sudo ln -s /etc/nginx/sites-available/fileora /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

### 7.4 HTTPS

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot renews automatically. Check with `sudo certbot renew --dry-run`.

### 7.5 Processing service

1. Install the plugin first (7.7) and copy the **Shared key** from WordPress → Fileora → Settings.
2. Copy the `server/` folder to the VPS, for example with `scp -r server you@example.com:~/`.
3. Run the installer. It installs LibreOffice, Ghostscript, qpdf, Tesseract, fonts, Python libraries and Chromium. It also creates the `fileora` system user and folders, writes `/etc/fileora/fileora.env` and starts the systemd service. It takes 5 to 10 minutes.

```bash
cd ~/server
sudo FILEORA_KEY='paste-the-shared-key' OCR_LANGS="hin" bash install.sh
```

4. **Extra OCR languages:** list them in `OCR_LANGS`, using Tesseract's codes: `hin` Hindi, `ben` Bengali, `tam` Tamil, `tel` Telugu, `mar` Marathi, `urd` Urdu, `ara` Arabic, `chi-sim` Chinese, `jpn` Japanese, `fra` French, `deu` German, `spa` Spanish. English is always included.
5. **AI tools:** add `ANTHROPIC_API_KEY=` to `/etc/fileora/fileora.env`, run `sudo systemctl restart fileora`, then turn on AI Summarizer and Translate PDF in Fileora → Settings.
6. **Check:** `curl http://127.0.0.1:8000/health` lists every tool as `true` or `false`.

Re-running `install.sh` updates the service and keeps your configuration.

### 7.6 Upload limits and timeouts

PHP (create `/etc/php/8.3/fpm/conf.d/99-fileora.ini`):

```ini
upload_max_filesize = 100M     ; per file, match Fileora → Settings → Max file size
post_max_size = 512M           ; a whole task (several files)
max_file_uploads = 50
memory_limit = 256M
max_execution_time = 300
max_input_time = 300
```

```bash
sudo systemctl restart php8.3-fpm
```

Nginx: the included site config already sets `client_max_body_size 512m` and `fastcgi_read_timeout 600s`. Keep these three numbers in line: the plugin's max file size, `FILEORA_MAX_MB` in `/etc/fileora/fileora.env`, and the PHP/Nginx limits. The Settings screen warns you if PHP accepts less than the plugin allows.

### 7.7 Install and configure the plugin

1. WordPress → Plugins → Add New → Upload Plugin → `wordpress/fileora-plugin.zip` → Install → Activate.
2. Fileora → Settings:
   - **Show Fileora on:** the homepage, or one address (e.g. `/tools/`) if your homepage is a blog or landing page.
   - **Tool pages:** on.
   - **Server address:** `http://127.0.0.1:8000`. **Shared key:** the same as `FILEORA_KEY`.
   - **Limits:** start with 100 MB, 20 files, 30 server tasks per visitor per 24 hours.
   - **Server tools:** the status column shows what the server can run. Turn off anything that says "Not available".
3. Create WordPress pages for `privacy`, `terms`, `cookies`, `about`, `contact` and `pricing`.

### 7.8 Real cron

WP-Cron only runs when someone visits. Use the system's cron instead:

```bash
sudo -u www-data wp --path=/var/www/fileora config set DISABLE_WP_CRON true --raw
echo '*/5 * * * * www-data /usr/local/bin/wp --path=/var/www/fileora cron event run --due-now >/dev/null 2>&1' | sudo tee /etc/cron.d/wordpress
```

### 7.9 Smoke test and go-live checklist

```bash
cd ~/server && sudo bash deploy/smoke_test.sh
```

It checks health, runs a real Protect PDF job through submit → status → download → delete and prints "All good."

Go-live checklist:
- [ ] `https://example.com/` shows Fileora; `https://example.com/merge-pdf/` opens Merge PDF.
- [ ] One browser tool (Merge) and one server tool (Word to PDF) work from a phone and a computer.
- [ ] Fileora → Dashboard shows the server as Running and lists your test jobs.
- [ ] Settings shows no warnings; tools that can't run are switched off.
- [ ] `ls /var/lib/fileora/jobs` is empty a few minutes after your tests.
- [ ] Privacy policy explains browser processing, server processing, deletion times, job records and the AI provider.
- [ ] Pro table, prices and the deletion wording in `src/index.html` match your real offer.
- [ ] Backups configured (3.2), `wp-sitemap.xml` submitted in Google Search Console.
- [ ] AGPL decision made (3.9).

---

## 8. Security

### 8.1 Controls already in place

| Threat | Control |
|---|---|
| Malicious or fake files | Extension allow-list per tool **and** first-byte check (`%PDF-`, ZIP, OLE, RTF), in both the plugin and the service. Files are never executed or served from the web root. |
| Huge uploads | Per-file limit (plugin and service), files per task, PHP and Nginx request limits. |
| Abuse and cost | Daily server-task limit per visitor (IP hash) or account; admins exempt; browser tools need no limit. |
| Reading someone else's result | Random job id plus a per-job token; only the token's SHA-256 is stored. Wrong or missing token returns 404. |
| Direct access to the processing service | Listens on 127.0.0.1 only and needs the shared key (constant-time comparison). Port 8000 is not opened in the firewall. |
| Server-side request forgery (HTML to PDF) | Only `http`/`https`. Addresses that resolve to private, loopback, link-local or reserved IPs are refused, and every sub-request is checked the same way (blocks cloud metadata such as 169.254.169.254). |
| Passwords in Protect/Unlock | Passed to qpdf in a private argument file, never on the command line or in logs. |
| Leftover files | Inputs deleted when a job ends, results after download, everything after 60 minutes. A restarted service marks unfinished jobs as failed and removes their inputs. |
| Service compromise | Runs as the unprivileged `fileora` user with systemd hardening: `NoNewPrivileges`, `PrivateTmp`, `ProtectSystem=full`, `ProtectHome`; writes only to `/var/lib/fileora`. |
| WordPress requests | REST nonces for logged-in users; settings saved through the Settings API with capability checks and sanitization; all output escaped. |
| Privacy | No file contents or raw IPs stored; IPs are hashed with your site's secret salt. |

### 8.2 Recommended additions

- **Cloudflare** (free plan) in front of the site: DDoS protection, firewall rules and a CDN for the app page.
- **Virus scanning** for uploaded Office files: install ClamAV (`clamav-daemon`) and scan the `in/` folder at the start of each job in `app.py`, failing the job if anything is found.
- **Automatic security updates:** `sudo apt install unattended-upgrades`.
- **Keep everything updated:** WordPress core, plugins, `install.sh` (re-run to update Python libraries).
- **Rotate the shared key** if it leaks: tick "Generate a new key on save", then update `/etc/fileora/fileora.env` and restart the service.

### 8.3 Behind Cloudflare: real IPs and bot protection

The daily limits need the visitor's real IP. Let Nginx restore it rather than trusting a header in PHP. Add this to the `http` block, listing Cloudflare's current IP ranges from cloudflare.com/ips:

```nginx
real_ip_header CF-Connecting-IP;
set_real_ip_from 173.245.48.0/20;   # ...one line per Cloudflare range
```

To block bots on the upload endpoint with **Cloudflare Turnstile**, render an invisible Turnstile widget in the app, send its token as an `X-Turnstile` header with the upload, and check it in WordPress:

```php
add_filter( 'fileora_pre_submit', function ( $ok, WP_REST_Request $request ) {
	$response = wp_remote_post( 'https://challenges.cloudflare.com/turnstile/v0/siteverify', array(
		'body' => array(
			'secret'   => TURNSTILE_SECRET,                  // define in wp-config.php
			'response' => (string) $request->get_header( 'x_turnstile' ),
			'remoteip' => Fileora_Security::client_ip(),
		),
	) );
	$result = json_decode( wp_remote_retrieve_body( $response ), true );
	return ! empty( $result['success'] ) ? $ok : new WP_Error( 'bot', 'Please confirm you are human and try again.', array( 'status' => 403 ) );
}, 10, 2 );
```

---

## 9. Performance

- **Browser first.** About half the tools never reach the server, so they cost nothing and are as fast as the visitor's device.
- **Parallel jobs.** `FILEORA_WORKERS` limits heavy work running at once (about 1 per 2 GB RAM); extra jobs wait in line and visitors see their position.
- **One service process.** The service runs a single Uvicorn process on purpose: jobs live in its thread pool and status files. More capacity comes from more workers or more servers (section 10), not more Uvicorn processes.
- **PHP-FPM workers.** Downloads stream through PHP. Set `pm.max_children` in `/etc/php/8.3/fpm/pool.d/www.conf` to cover concurrent downloads; 20 is a good start on 4 GB.
- **Caching.** The app page and tool pages are fully cacheable for visitors (the nonce is only added for logged-in users). Never cache `/wp-json/`.
- **OCR speed:** `--skip-text` leaves pages that already have text alone; scans of 10–20 pages take tens of seconds on 2 vCPU.
- **LibreOffice:** each job uses its own profile folder, so conversions can run in parallel safely. The first conversion after a restart is slower while it starts up.
- **Monitoring:** Fileora → Dashboard shows average processing time and failures per tool. Watch `journalctl -u fileora` and memory (`free -m`) during the first weeks.

---

## 10. Scalability

Grow in steps. Each step keeps the same plugin, app and API.

| Stage | When | Change |
|---|---|---|
| 1. One VPS | Launch | As in this guide |
| 2. Bigger VPS | Jobs often wait in line | More RAM and CPU, raise `FILEORA_WORKERS` |
| 3. Separate processing server | Server tools slow down the website | Run `install.sh` on a second VPS. Bind the service to its private network address instead of 127.0.0.1 and allow only the WordPress server in its firewall. Set the plugin's **Server address** to that address. |
| 4. Several processing servers | One server isn't enough | Put a load balancer in front with "sticky" routing by job id, or move to a shared queue (Redis with RQ or Celery) and shared storage (S3-compatible, such as MinIO). Then replace the thread pool in `app.py` with the queue. The HTTP API stays the same. |
| 5. Global | Visitors far away | CDN for the app page and libraries (Cloudflare), a processing region near them |

Database load stays small: one row per server job. Add an index or archive old rows if you reach millions.

---

## 11. Testing

### 11.1 What was tested for this release

| Area | Result |
|---|---|
| Browser tools (headless Chromium) | Every browser tool produced correct output: page counts, rotation, crop boxes, ZIP, Markdown, text diff, image sizes and formats |
| Rotated pages | Page numbers, watermark and signature land upright on pages with a `/Rotate` flag |
| Processing service | All 15 server tools that need no API key, on real files, plus readable errors (wrong password, no tables, missing OCR language, unsupported file). AI tools: only the "no API key" message was tested |
| Job flow | Queue with "ahead" counts, 4 parallel jobs, uploads deleted at job end, results deleted after download, wrong key → 401, bad id → 404 |
| Security | Private and cloud-metadata addresses refused for HTML to PDF; fake PDF rejected; missing token → 404; second download → 410 |
| WordPress 7.1.3 plugin | Activation (table, key, cron), app and tool pages with SEO tags, sitemap, settings save, tools switched off disappear, daily limit returns 429, admin exempt, dashboard statistics |
| Responsive | 1440, 820 and 390 px wide, light and dark themes, no horizontal scrolling |

### 11.2 Checklist before each release

- [ ] `node build.mjs` runs without errors; open `fileora.html` and check the browser console is clean.
- [ ] Each category: run one tool with a small file and one with a large file (50+ pages).
- [ ] Phone (Android and iPhone): upload from files and from the camera (Scan to PDF).
- [ ] Password-protected, damaged and scanned PDFs in Merge, Compress, Repair and OCR.
- [ ] Limits: too-large file, too many files, daily limit reached.
- [ ] After testing, `/var/lib/fileora/jobs` is empty within 60 minutes.
- [ ] Load test the server tools before marketing pushes. For example, use k6 or Locust to submit 20 jobs at once, and check the queue times and memory.

---

## 12. Operations and maintenance

| Task | How |
|---|---|
| Service status and logs | `systemctl status fileora`, `journalctl -u fileora -f` |
| WordPress errors | Turn on `WP_DEBUG_LOG` briefly; read `wp-content/debug.log` (Fileora lines start with `[Fileora]`) |
| Restart after config change | `sudo systemctl restart fileora` |
| Update the service | Copy the new `server/` folder, run `sudo bash install.sh` again |
| Update the app | Change `src/`, run `node build.mjs`, upload `wordpress/fileora/public/` (or the new plugin zip) |
| Add OCR languages | `sudo apt install tesseract-ocr-<code>`, restart the service |
| Backups | Database and `wp-content` daily (UpdraftPlus or your host); `/etc/fileora/` once after changes. Job folders never need backups. |
| Monthly | Update Ubuntu, WordPress and plugins; check the Dashboard for failing tools |

---

## 13. Troubleshooting

| Problem | Cause and fix |
|---|---|
| Server tools say "needs the Fileora server" | `CONFIG.apiBase` is empty: you are opening `fileora.html` directly, or the plugin isn't serving the page. Open the site through WordPress. |
| "The processing server isn't responding" | Service down. Check `systemctl status fileora` and `journalctl -u fileora -n 50`. |
| Dashboard: "Running without a shared key", or jobs fail with "isn't set up correctly" | `FILEORA_KEY` in `/etc/fileora/fileora.env` doesn't match the plugin's Shared key. Fix it, then restart the service. |
| Uploads fail at a certain size | PHP `upload_max_filesize`/`post_max_size` or Nginx `client_max_body_size` too low (7.6). |
| Tool pages (`/merge-pdf/`) show 404 | Permalinks set to "Plain". Settings → Permalinks → Post name → Save. Or "Tool pages" is off, or that tool is switched off. |
| Word to PDF looks different from Word | Missing fonts. Install the fonts the documents use, such as `ttf-mscorefonts-installer` (accept the licence), then restart the service. |
| OCR says the language isn't installed | `sudo apt install tesseract-ocr-<code>`, then restart. |
| HTML to PDF: "That page didn't load" | The site blocks automated browsers, needs a login, or took over 45 seconds. |
| AI tools: "aren't switched on yet" | Add `ANTHROPIC_API_KEY`, restart, and turn the tools on in Settings. |
| Visitors hit the daily limit too soon | Behind a proxy or Cloudflare, every visitor looks like the same IP. Set up `real_ip` (8.3). |
| Admin wants to see the theme on the homepage | Add `?fileora-theme=1` to the address while logged in. |
| Design broken after an optimization plugin | Exclude the Fileora page from "combine/minify JS and CSS"; it is already a single optimized file. |

---

## 14. Reference

### 14.1 WordPress REST API (`/wp-json/fileora/v1`)

| Method and path | Request | Response |
|---|---|---|
| `POST /jobs/{tool}` | multipart: `files[]`, `options` (JSON), `url` (HTML to PDF) | `202 {job, token, status}`, or `{error}` with 400/403/404/413/415/429/502/503 |
| `GET /jobs/{id}` | header `X-Fileora-Token` | `{status, ahead?, kind?, name?, size?, text?, error?}` |
| `GET /jobs/{id}/file` | header `X-Fileora-Token` | The file (streamed). 409 if not finished, 410 if already downloaded |
| `GET /health` | admin cookie + nonce | Same as the service's `/health` |

### 14.2 Processing service API (`127.0.0.1:8000`, header `X-Fileora-Key`)

| Method and path | Response |
|---|---|
| `POST /jobs/{tool}` | `202 {job, status: "queued"}` |
| `GET /jobs/{id}` | Status as above |
| `GET /jobs/{id}/file` | Result file; several results come as `fileora-results.zip` |
| `DELETE /jobs/{id}` | `{deleted: true}` |
| `GET /health` (no key) | `{ok, key_set, ocr_languages, tools: {id: bool}, workers}` |

Server tool ids: `compress`, `repair`, `ocr`, `pdfa`, `word2pdf`, `ppt2pdf`, `excel2pdf`, `html2pdf`, `pdf2word`, `pdf2ppt`, `pdf2excel`, `edit`, `forms`, `protect`, `unlock`, `redact`, `summarize`, `translate`.

### 14.3 Service configuration (`/etc/fileora/fileora.env`)

| Variable | Default | Meaning |
|---|---|---|
| `FILEORA_KEY` | – | Shared secret, must equal the plugin's Shared key |
| `FILEORA_WORKDIR` | `/var/lib/fileora/jobs` | Job folders |
| `FILEORA_MAX_MB` | 200 | Max size per file |
| `FILEORA_WORKERS` | 2 | Jobs running at once |
| `FILEORA_TTL_MIN` | 60 | Minutes before unclaimed jobs are deleted |
| `ANTHROPIC_API_KEY` | – | Turns on AI tools |
| `FILEORA_AI_MODEL` | `claude-haiku-5-5` | Claude model for AI tools |
| `FILEORA_ALLOW_PRIVATE_URLS` | – | **Testing only.** Never set on a public server |

### 14.4 Plugin settings (`fileora_settings` option)

`service_url`, `service_key`, `serve` (`home`/`path`/`off`), `path`, `tool_pages`, `max_mb`, `max_files`, `daily_limit`, `retention_days`, `disabled` (array of server tool ids). Default disabled tools: `edit`, `summarize`, `translate`.

### 14.5 App configuration (`CONFIG`, injected by the plugin)

`apiBase` (REST base URL), `maxFileMB`, `nonce` (logged-in users), `disabled` (tool ids to hide), `toolPages` (bool), `toolBase` (e.g. `/` or `/tools/`). Without the plugin, all are empty and the app runs browser tools only.

### 14.6 Sources for licences and capabilities

- PyMuPDF licence (AGPL-3.0 or commercial): https://pypi.org/project/PyMuPDF/
- pdf2docx licence (MIT): https://pdf2docx.readthedocs.io/en/latest/license.html
- Ghostscript licensing (AGPL or commercial): https://ghostscript.com/licensing
- OCRmyPDF (MPL-2.0, needs Ghostscript and Tesseract): https://pypi.org/project/ocrmypdf/
- GSAP Standard License: https://gsap.com/community/standard-license/
- pdf.js annotation editor (free text, ink, stamp, highlight; `saveDocument()`): https://www.nutrient.io/blog/pdfjs-annotation-editor-layer/
- Claude models and prices: https://platform.claude.com/docs/en/models/overview
- Simple Cloudflare Turnstile (free): https://wpmayor.com/simple-cloudflare-turnstile/
