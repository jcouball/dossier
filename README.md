# dossier

A Ruby toolkit for processing a Mail export (`.mbox`) into clean, readable text compilations suitable for AI-assisted legal review. Uses Claude internally for OCR transcription and document summaries; the resulting context files can be used with any AI.

## What it does

1. **Extracts** emails and attachments from a raw MBOX export, normalizing dates to Pacific Time and stripping MIME encoding artifacts.
2. **Filters** known junk attachments by MD5 checksum, then deduplicates remaining attachments by content hash.
3. **Compiles emails** into a single chronological narrative (`emails.txt`), with HTML-only emails automatically converted to plain text.
4. **Transcribes attachments** by extracting text from PDFs, Word docs, RTF, and plain text files. Scanned PDFs are sent to Claude for transcription, with Tesseract OCR as a fallback. Results are cached under `cache/text/`.
5. **Syncs** all context files into a `context/` directory alongside relevant `.xlsx` spreadsheets, ready to upload to any AI tool.

## Requirements

- macOS (uses `textutil` for Word/RTF conversion)
- Ruby (standard macOS install is fine)
- [Homebrew](https://brew.sh)
- An [Anthropic API key](https://console.anthropic.com/) for Claude-powered transcription and document summaries (optional — see Configuration)

Run the setup script once to install all dependencies:

```bash
bin/setup
```

This installs `poppler`, `tesseract`, `ocrmypdf`, and `imagemagick` via Homebrew, then runs `bundle install`.

## Configuration

Create a `.env` file in the project root with your Anthropic API key:

```
ANTHROPIC_API_KEY=sk-ant-...
```

This file is listed in `.gitignore` and will not be committed. Without it, Claude transcription and document summaries are silently skipped and the pipeline falls back to local Tesseract OCR.

## Usage

Place your MBOX export at:

```
exported.mbox
```

Then run the full pipeline:

```bash
rake
```

This runs all the extraction in order. You can also run individual steps:

| Command | Description |
| --- | --- |
| `rake extract` | Step 1 — Parse MBOX, extract attachments to `work/attachments/` |
| `rake filter_md5` | Step 1.1 — Delete known junk files by MD5 |
| `rake dedupe` | Step 1.2 — Remove duplicate attachments |
| `rake compile_emails` | Step 2 — Build `work/emails.txt` |
| `rake compile_docs` | Step 3 — Transcribe attachments, build `work/attachments.txt` (uses Claude + OCR cache) |
| `rake sync` | Step 4 — Sync everything to `context/` |
| `rake clean` | Remove all generated output files (`work/`, `context/`) |
| `rake clobber` | Clean + remove the OCR/transcription cache (`cache/`) and `Gemfile.lock` |

### Claude transcription flags

`rake compile_docs` passes arguments directly to `scripts/transcribe_attachments`. You can disable AI features if needed:

| Flag | Effect |
| --- | --- |
| `--no-ai` | Skip Claude entirely; use local Tesseract OCR only |
| `--no-ai-all-pages` | Only send scanned (image-only) pages to Claude; use `pdftotext` for digital text |

## Output

After `rake sync`, the `context/` directory contains:

- `emails.txt` — All emails in chronological order, plain text
- `attachments.txt` — Full extracted/transcribed text from all attachments
- `index.txt` — AI-generated one-line summary per document, sorted by date
- `docs_<type>.txt` — Attachments grouped by document type (e.g. `docs_court_filing.txt`)
- `*.xlsx` — Spreadsheet attachments (copied directly)

Upload this directory to any AI tool for review.

## Project structure

```
exported.mbox              # Input: raw Mail export
sideload/                  # Extra documents to include alongside attachments
work/
  attachments/             # Extracted attachments (intermediate)
  emails.txt               # Compiled email narrative
  attachments.txt          # Compiled attachment text
  index.txt                # Document index (summaries)
  docs_*.txt               # Per-type document compilations
cache/
  text/                    # Cached OCR/transcription results (survives `rake clean`)
context/                   # Final output, ready for AI context
scripts/
  import_mbox              # Step 1: parse MBOX, extract attachments
  delete_by_md5            # Step 1.1: remove known junk by MD5
  deduplicate              # Step 1.2: remove duplicate attachments
  compile_emails           # Step 2: build email narrative
  transcribe_attachments   # Step 3: extract/OCR/transcribe attachment text
  create_context           # Step 4: sync to context/
bin/
  setup                    # Install dependencies
Rakefile                   # Orchestrates all steps
.env                       # API credentials (not committed — see Configuration)
```
