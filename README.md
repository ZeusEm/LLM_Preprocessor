# LLM_Preprocessor 🚀

> Production-ready document preprocessor that prepares mixed-format study material for LLM ingestion — text extraction, high-quality OCR (Tesseract), format classification, memory-safe processing, and CPU-optimized parallelism.

---

## 🔥 Highlights

* ✅ Converts mixed-format document corpus into GPT4All/LocalDocs-ready text
* ✅ Detects scanned (image-only) files and performs OCR automatically
* ✅ Two output zones: `Processed_Compatible/` (case **a & b**) and `Incompatible/` (case **c**)
* ✅ Hash-based deduplication to skip previously processed files
* ✅ Memory-safe page-by-page OCR with optional multithreading (configurable)
* ✅ Minimal external dependencies — Tesseract required; no heavy ML libs by default

---

## 📖 Table of Contents

* [Overview 📚](#overview-📚)
* [How it works (algorithm) ⚙️](#how-it-works-algorithm-⚙️)
* [Quickstart 🚀](#quickstart-🚀)
* [Installation 🛠️](#installation-🛠️)
* [Usage — run the script ▶️](#usage---run-the-script-️)
* [Configuration 🔧](#configuration-🔧)
* [Output structure & samples 🗂️](#output-structure--samples-🗂️)
* [Architecture & processing flow 🏗️](#architecture--processing-flow-🏗️)
* [Performance & tuning ⚡](#performance--tuning-⚡)
* [Troubleshooting 🩺](#troubleshooting-🩺)
* [Contributing & License ❤️](#contributing--license-❤️)

---

## Overview 📚

LLM_Preprocessor is intended for people who have a large collection of learning material (PDFs, images, Word docs, Excel sheets, PPTs, EPUBs, etc.) and want to prepare a single, clean, text-first dataset to train or query local LLMs (e.g., GPT4All LocalDocs). It prioritizes quality of text extraction and OCR, robust memory control, and deterministic results across runs.

---

## How it works (algorithm) ⚙️

1. **Startup checks**: verify essential Python packages and static external binaries (Tesseract and Poppler if used).
2. **Discover files**: recursively scan `INPUT_DIR`.
3. For **each file**:
   * compute hash (skip if present in `filehash.json`)
   * attempt direct text extraction (PyMuPDF, python-docx, python-pptx, pandas, ebooklib, plain text)
   * if file is extension-compatible and text ≥ threshold → copy original to `Processed_Compatible` (case **a**)
   * if extension-compatible but text insufficient → OCR (page-by-page) and write `.txt` to `Processed_Compatible` (case **b**)
   * if extension not supported → copy to `Incompatible` for manual review (case **c**)
4. Record processed hashes to avoid re-processing; emit stats & logs.

---

## Quickstart 🚀

**1. Clone repo**

```bash
git clone https://github.com/yourusername/LLM_Preprocessor.git
cd LLM_Preprocessor
```

**2. Place files**

* Drop your corpus into the configured `INPUT_DIR` (by default the script uses the static path you set in script).

**3. Install system dependencies (one-time):**

* Install **Tesseract OCR** (Windows: UB Mannheim build recommended) and note its `tesseract.exe` path.
* (Optional) If using Poppler-based PDF→image conversion, install Poppler and note the `pdftoppm` path. The production script provided uses PyMuPDF for PDF rendering and does **not** require Poppler.

**4. Run the script (Spyder, VSCode, or terminal):**

```bash
python filePreparation_v2.py
```

---

## Installation 🛠️

### Python packages

The script auto-checks and attempts to `pip install` the *lightweight* packages it needs:

* `pytesseract`, `Pillow`, `PyMuPDF` (fitz), `python-docx`, `python-pptx`, `pandas`, `tqdm`, `ebooklib`, `openpyxl`.

> **Important:** Do **not** install heavy ML packages (torch, torchvision, easyocr) unless you explicitly need handwriting OCR and understand the environment requirements.

### System binaries

* **Tesseract** — required. Confirm via:

```bash
"C:\Program Files\Tesseract-OCR\tesseract.exe" --version
```

---

## Usage — run the script ▶️

* Edit the static paths at the top of `filePreparation_v2.py`:

  * `INPUT_DIR` — folder containing your raw files
  * `OUTPUT_DIR` — where processed results go
  * `TESSERACT_CMD` — path to `tesseract.exe`
  * `MAX_WORKERS` — concurrency (default conservative)

* Execute:

```bash
python filePreparation_v2.py
```

You’ll see live progress in the console. On completion the script prints a summary: total files, skipped (hash), case a (copied), case b (OCR -> txt), incompatible count, errors.

---

## Configuration 🔧

Top-of-file configurable variables (edit in script):

```py
INPUT_DIR = r"..."
OUTPUT_DIR = r"..."
TESSERACT_CMD = r"..."
MAX_WORKERS = 3          # threads: increases CPU usage but watch RAM
MIN_TEXT_CHARS = 80      # threshold to decide OCR vs copy
```

**Tip:** Start with `MAX_WORKERS = 2` on low-RAM machines. Increase gradually.

---

## Output structure & samples 🗂️

After a run, `OUTPUT_DIR` will contain:

```
knowledgeBase_prepared/
├─ Processed_Compatible/         # Case (a) originals or .txt from OCR (case b)
│  ├─ *.txt                      # OCR/extracted text files
│  └─ filehash.json              # hashes of processed files
├─ Incompatible/                 # Case (c) unsupported or failed
│  └─ <original files>
└─ logs/
   ├─ errors.log
   └─ processing_summary.json
```

**Sample outputs**

* `SomeDocument.pdf → SomeDocument.txt` (if OCR used or text extracted)
* `Report.docx → Report.docx` (copied as original if text present)
* `image_123.jpg → image_123.txt` (image OCR output)

---

## Architecture & processing flow 🏗️

```
+-----------------+
| INPUT_DIR       |
| (mixed files)   |
+--------+--------+
         |
         v
+------------------------------+
| Discover & hash (skip dupes) |
+------------------------------+
         |
         v
+----------------------+    +---------------------+
| Text extraction pass |--->| text >= MIN_TEXT ?  |--Yes--> copy original -> Processed_Compatible
+----------------------+    +---------------------+
         |
         No
         v
+------------------------+
| OCR fallback (page-by-page)
+------------------------+
         |
         v
save OCR output .txt -> Processed_Compatible
         |
If failed -> move original -> Incompatible
```

---

## Performance & tuning ⚡

* **CPU**: The script uses a thread pool by default (`MAX_WORKERS`). Threads are chosen because heavy C-level libs (PyMuPDF, Tesseract) release the GIL during processing.
* **RAM**: PDF pages are rendered and OCR’d one-by-one — images are deleted promptly and `gc.collect()` is used. If you still see MemoryErrors, reduce `MAX_WORKERS` to 1 or 2.
* **Throughput monitoring**: the script prints periodic throughput and ETA. Use these to decide whether to increase `MAX_WORKERS`.

---

## Troubleshooting 🩺

### Q: `ModuleNotFoundError: No module named 'pdf2image'` or similar

* Run the script again — it attempts to install missing lightweight packages.
* Or install manually with the interpreter Spyder uses:

```bash
python -m pip install pdf2image Pillow PyMuPDF pytesseract python-docx python-pptx pandas tqdm ebooklib openpyxl
```

### Q: Tesseract not found

* Ensure `TESSERACT_CMD` points to actual `tesseract.exe`
* Test in terminal:

```bash
"C:\Program Files\Tesseract-OCR\tesseract.exe" --version
```

### Q: MemoryError during OCR

* Reduce `MAX_WORKERS` in the script (start at 1–2)
* Close other heavy applications
* Consider splitting the job into smaller subfolders and run separately

### Q: Some `.doc` or `.ppt` files ended up in `Incompatible/`

* `.doc` (legacy Word) is unsupported by `python-docx` — convert `.doc` → `.docx` (Word or LibreOffice), or enable LibreOffice conversion logic (requires LibreOffice install).
* `.ppt` (older PPT) can be handled via conversion to `.pptx` (LibreOffice) or add an OLE parser.

### Optional: EasyOCR (handwriting)

* We **did not** enable EasyOCR by default because it requires `torch` and heavier dependencies which can cause environment issues.
* If you need handwriting support, install CPU PyTorch + easyocr in a clean environment:

```bash
pip install --upgrade pip
pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
pip install easyocr
```

---

## Contact Me ❤️

* Built for you. You can DM me either here or on LinkedIn (https://www.linkedin.com/in/shubham-mehta-5141172b3/)
