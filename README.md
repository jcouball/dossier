# dossier

A Ruby toolkit for processing a Mail export (`.mbox`) into clean, readable text
compilations suitable for AI-assisted review. Uses Claude internally for OCR
transcription and document summaries. The resulting context files can be used with
any AI.

## What it does

1. **Extracts** emails and attachments from a raw MBOX export, normalizing dates to
   Pacific Time and stripping MIME encoding artifacts.
2. **Filters** known junk attachments by MD5 checksum, then deduplicates remaining
   attachments by content hash.
3. **Compiles emails** into a single chronological narrative (`emails.txt`), with
   HTML-only emails automatically converted to plain text.
4. **Transcribes attachments** by extracting text from PDFs, Word docs, RTF,
   plain text files, and images (JPEG, PNG, TIFF). By default all document pages
   are sent to Claude for transcription; use `--no-ai-all-pages` to limit AI to
   scanned (image-only) pages, or `--no-ai` to fall back to Tesseract entirely.
   Results are cached under `cache/text/`.
5. **Syncs** all context files into a `context/` directory alongside relevant `.xlsx`
   spreadsheets, ready to upload to any AI tool.

## Requirements

- macOS (uses `textutil` for Word/RTF conversion)
- Ruby (standard macOS install is fine)
- [Homebrew](https://brew.sh)
- An [Anthropic API key](https://console.anthropic.com/) for Claude-powered
  transcription and document summaries (optional — see Configuration)

Run the setup script once to install all dependencies:

```bash
bin/setup
```

This installs `poppler`, `tesseract`, `ocrmypdf`, and `imagemagick` via Homebrew,
then runs `bundle install`.

## Configuration

Create a `.env` file in the project root with your Anthropic API key:

```dotenv
ANTHROPIC_API_KEY=sk-ant-...
```

This file is listed in `.gitignore` and will not be committed. Without it, Claude
transcription and document summaries are silently skipped and the pipeline falls back
to local Tesseract OCR.

### Excluding attachments

Create an `excluded_attachments.txt` file in the project root to list attachments
that should be deleted before processing. This is useful for removing known-junk
files such as email signatures, logos, and blank pages that would otherwise clutter
the output.

Each line should contain the MD5 checksum of a file to exclude. Lines starting with
`#` and blank lines are ignored:

```text
# Outlook blank-page signature images
bfd87e02e5de8d1dc16d3fd2cd9d8507
dc445961e973e0beffb52836c5074a8a
```

To get the MD5 of a file:

```bash
md5 work/attachments/<filename>
```

This file is listed in `.gitignore` and will not be committed. If it is absent,
`rake delete_excluded_attachments` silently skips without deleting anything.

### Detecting junk attachments interactively

Run `rake detect_excluded_attachments` (or `scripts/detect_excluded_attachments`
directly) after `rake extract` to scan for likely-junk files and decide their fate
one by one:

- **y** — delete the file now and record its MD5 in `excluded_attachments.txt` so
  it is removed automatically on future runs.
- **n** — keep the file and record its MD5 in `included_attachments.txt` so it is
  never flagged again.
- **s** — skip for now; the file will be presented again on the next run.

Pass `--dry-run` (or `-n`) to list suspects without prompting or writing anything.

`included_attachments.txt` is listed in `.gitignore` and will not be committed.

## Usage

Place your MBOX export at:

```text
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
| `rake detect_excluded_attachments` | Optional — Interactively review suspect attachments; updates `excluded_attachments.txt` and `included_attachments.txt` (run after `rake extract`) |
| `rake delete_excluded_attachments` | Step 1.1 — Delete attachments listed in `excluded_attachments.txt` by MD5 |
| `rake deduplicate_attachments` | Step 1.2 — Remove duplicate attachments |
| `rake compile_emails` | Step 2 — Build `work/emails.txt` |
| `rake compile_docs` | Step 3 — Transcribe attachments, build `work/index.txt` and `work/docs_*.txt` (uses Claude + OCR cache) |
| `rake sync` | Step 4 — Sync everything to `context/` |
| `rake clean` | Remove all generated output files (`work/`, `context/`) |
| `rake clobber` | Clean + remove the OCR/transcription cache (`cache/`) and `Gemfile.lock` |

### Transcription flags

To use these flags, run `scripts/transcribe_attachments` directly instead of via
`rake compile_docs`:

| Flag | Effect |
| --- | --- |
| `--no-ai` | Skip Claude entirely; use local Tesseract OCR only |
| `--no-ai-all-pages` | Only send scanned (image-only) pages to Claude; use `pdftotext` for digital text |
| `--no-progress` | Disable the progress bar |
| `--verbose` | Print detailed processing output |

## Output

After `rake sync`, the `context/` directory contains:

- `emails.txt` — All emails in chronological order, plain text
- `index.txt` — AI-generated one-line summary per document, sorted by date
- `docs_<type>.txt` — Attachments grouped by document type (`docs_court.txt`,
  `docs_financial.txt`, `docs_medical.txt`, `docs_other.txt`)
- `*.xlsx` — Spreadsheet attachments (copied directly)

Upload this directory to any AI tool for review.

## Project structure

```text
bin/                       # Install dependencies
cache/                     # Cached OCR/transcription results (survives `rake clean`)
context/                   # Final output, ready for AI context
scripts/                   # Pipeline step scripts
sideload/                  # Extra documents to include alongside attachments
work/                      # Intermediate generated files
.env                       # API credentials (not committed — see Configuration)
exported.mbox              # Input: raw Mail export
excluded_attachments.txt   # MD5 checksums of attachments to exclude (Step 1.1, not committed)
included_attachments.txt   # MD5 checksums confirmed as real content (not committed)
Gemfile
Rakefile
```
