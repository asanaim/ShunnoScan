# Shunno Scan (শূন্য স্ক্যান)

**Shunno Scan** is a fast, lightweight, privacy-first offline document scanner, multi-script Optical Character Recognition (OCR) suite, and searchable PDF generator for Android. Designed with an Apple Cupertino aesthetic as a modern, standalone successor to Microsoft Lens, the name **Shunno** (*শূন্য* — meaning **Zero** / `0`) reflects its core philosophy: **Zero Cloud Uploads**, **Zero Ads or Tracking**, **Zero Subscriptions**, and **Zero Bloat**.

---

## 📲 1-Click Direct Download (For Non-IT Users)

Anyone on an Android phone (Android 7.0+ / 64-bit ARM) can download and install **Shunno Scan** with a single tap—no GitHub account or technical setup required:

- **[👉 Download ShunnoScan.apk (Direct 1-Click Install)](https://github.com/asanaim/LumaScan/releases/latest/download/ShunnoScan.apk)**
- **[🌐 Open Apple-Style Web Download Page](https://asanaim.github.io/LumaScan/)**

---

## 🎯 What It Does

1. **Camera & Gallery Document Scanning:**
   - Captures single or multi-page documents, whiteboards, receipts, books, and ID cards with automatic edge detection, perspective correction, manual corner adjustment, and glare/shadow removal filters.
   - Supports batch importing up to 50 existing photos directly from the Android Gallery.

2. **True Searchable "Sandwich" PDF Generation:**
   - Produces standard PDF files where every scanned page image is paired with an aligned, invisible selectable text layer.
   - Allows you to press `Ctrl+F` (or long-press on mobile) in any PDF viewer (Adobe Acrobat, Google Drive PDF Viewer, Chrome, macOS Preview) to search, highlight, and copy individual words directly off the scanned page.

3. **Hybrid Cupertino Multi-Script Offline OCR:**
   - **Built-In Default Languages:** **English** and **Bangla + English (`বাংলা + English`)** are bundled inside the app and work 100% offline immediately after installation.
   - **On-Demand Downloadable & Removable Scripts:** Tapping **`+ Languages`** lets users download **Devanagari (Hindi) + English**, **Chinese + English**, **Japanese + English**, or **Korean + English** directly inside the app when needed.
   - **Instant Space Recovery:** Users can **long-press** any downloaded language chip on the main bar (or tap **Remove** in the `+ Languages` sheet) to delete the model file from local storage.

4. **Unicode-Aware Smart File Naming:**
   - Automatically inspects the top heading of Page 1 and names the exported `.pdf`, `.txt`, and `.csv` files after the document's actual title (e.g., `বাংলাদেশ_কৃষি_ব্যাংক_9445.pdf` or `Electricity_Bill_1042.pdf`) while preserving Bangla combining vowel signs (`া`, `ি`, `ী`, `ু`, `ূ`, `ৃ`, `ে`, `ৈ`, `ো`, `ৌ`, `ং`, `্`).

5. **🪪 ID Card 2-in-1 Page Stitcher:**
   - When toggled on, scanning the Front and Back of a national ID card, passport, or driver's license automatically centers and composites both sides onto a single high-resolution A4 page before running OCR and PDF export.

6. **🔒 Auto-Redact PII (Privacy Sanitizer):**
   - When toggled on, automatically detects sensitive Personal Identifiable Information (phone numbers, email addresses, and 12-to-15-digit card/ID numbers), paints permanent solid black redaction bars directly onto the page image, and strips those tokens from both the searchable PDF layer and `.txt` output.

7. **📊 Spatial Table-to-CSV Exporter:**
   - Reconstructs printed tables, invoices, and financial statements into structured `.csv` files that open directly in Microsoft Excel or Google Sheets.

8. **📇 Business Card Contact Extractor:**
   - Detects phone numbers and email addresses inside scanned cards and provides a one-tap **Save Contact** button that pre-fills the native Android Contacts screen.

9. **Persistent Local Library & Instant Keyword Search:**
   - Saves all scan metadata and full OCR transcriptions in a local on-device SQLite database (`shunnoscan.db`) so you can search across thousands of historical pages in milliseconds.

---

## ⚙️ How It Does It (Technical Architecture)

### 1. The "Shunno / `0`" Visual & UI Architecture
- **Jetpack Compose Cupertino UI:** Built entirely with declarative Kotlin and Material 3 styled with iOS system colors (`#0A84FF` System Blue, `#30D158` Mint Green, `#FF9F0A` Amber, `#F2F2F7` Grouped Background).
- **Vector Adaptive Icon (`ic_launcher_foreground.xml`):** Features a sculpted geometric **`0` (Shunno)** capsule-ring acting as an optical scanner lens inside Continuity-style viewfinder brackets, bisected by a `#30D158` laser scan horizon beam and a subtle optical slash glint.

### 2. Dual-Engine Parallel Multi-Core OCR Pipeline (`MainActivity.kt`)
- **Latin Fast Path:** When `English` is selected, Shunno Scan dispatches page bitmaps concurrently across all CPU cores using Kotlin Coroutines (`coroutineScope { bitmaps.map { async(Dispatchers.Default) { ... } }.awaitAll() }`) powered by Google ML Kit v2 Text Recognition (`com.google.mlkit:text-recognition:16.0.1`).
- **Multi-Script LSTM Path:** When `Bangla + English` or any downloaded script is selected, Shunno Scan uses a mutex-guarded, memory-cached **Tesseract 5 OpenMP (`TessBaseAPI`)** instance (`cz.adaptech.tesseract4android:tesseract4android-openmp:4.9.0`). By keeping `TessBaseAPI` warm in memory and calling `tess.clear()` between pages instead of re-initializing from flash storage, multi-page Bangla OCR runs up to 3× faster.
- **Word-Level Bounding Box Extraction:** Both engines extract exact pixel `Rect` coordinates for every recognized word (`Element.boundingBox` in ML Kit and `PageIteratorLevel.RIL_WORD` in Tesseract).

### 3. Searchable "Sandwich" PDF Synthesis & 90% Size Compression (`SearchablePdfGenerator.kt`)
- **1:1 Coordinate Mapping:** Each page is decoded into a software-allocated `ARGB_8888` `Bitmap` and compressed through an in-memory **85% JPEG byte stream** (`compressBitmapEfficient`). This shrinks 40 MB multi-page raw bitmap PDFs down to **~2 MB** while keeping 300 DPI text edges razor-sharp.
- **Skia PDF Alpha-1 Text Overlay Trick:** Android's native `android.graphics.pdf.PdfDocument` relies on Google's Skia PDF backend, which automatically strips out text draw calls if `alpha == 0` (`Color.TRANSPARENT`). Shunno Scan sets `invisibleTextPaint.color = Color.argb(1, 0, 0, 0)` (1/255 opacity) and dynamically calculates `invisibleTextPaint.textScaleX = (box.width() / measuredWidth)` so that Skia writes real PDF text operators (`BT ... Tj ET`) aligned to the exact pixel width and baseline of every word on the image—completely invisible to the eye, yet 100% selectable and searchable.
- **Invisible Embedded App Link:** The direct app download link is written into the invisible Skia PDF text stream at the bottom margin (`Color.argb(1, 0, 0, 0)`), keeping the visual document 100% clean and unbranded while retaining the link inside the PDF data layer.

### 4. Spatial Y/X Clustering for Table-to-CSV (`convertWordsToCsv`)
- Words on each page are sorted by their vertical centroid (`boundingBox.centerY()`) and grouped into horizontal rows using an adaptive vertical tolerance of `0.6 × wordHeight`.
- Within each row, words are sorted left-to-right (`boundingBox.left`). Horizontal gaps exceeding `1.3 × wordHeight` trigger a column boundary (`,`), while smaller gaps join words into the same cell, outputting RFC-4180 compliant `.csv` spreadsheets.

---

## 🛠️ Project Structure

```text
ShunnoScan/
├── .github/workflows/main.yml       # Automated JDK 17 + Gradle 8.7 CI & 1-Click Release Publisher
├── app/
│   ├── build.gradle.kts             # arm64-v8a ABI filter, R8 shrinking, ML Kit + Tesseract OpenMP
│   └── src/main/
│       ├── AndroidManifest.xml      # Shunno Scan label & FileProvider configuration
│       ├── java/com/lumascan/app/
│       │   ├── MainActivity.kt              # Cupertino UI, Parallel OCR, On-Demand Language Manager
│       │   ├── SearchablePdfGenerator.kt    # Sandwich PDF engine, 2-in-1 A4 Stitcher, CSV & Redactor
│       │   └── LumaScanDb.kt                # Persistent SQLite history database (shunnoscan.db)
│       └── res/
│           ├── drawable/
│           │   ├── ic_launcher_background.xml  # Apple System Blue gradient + ambient lens ring
│           │   └── ic_launcher_foreground.xml  # Geometric "0" (Shunno) optical scanner emblem
│           └── xml/file_paths.xml
├── docs/index.html                  # Apple-style Web Download Landing Page for Non-IT Users
├── build.gradle.kts
├── settings.gradle.kts
└── README.md
```

## 📄 License
MIT License
