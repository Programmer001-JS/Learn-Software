# OCR

> **In one line —** turning pictures of text into text; nearly solved for clean printed documents, and still genuinely hard for everything else.

| | |
|---|---|
| **Full name** | Optical Character Recognition |
| **Category** | Application Area |
| **Architectural Layer** | Application |
| **Related notes** | [Computer Vision](Computer%20Vision.md) · [OpenCV](OpenCV.md) · [NLP](NLP.md) · [AI Pipelines](AI%20Pipelines.md) |

---

## 1. Short Definition

*What is it?*

OCR extracts machine-readable text from images — scanned documents, photographs, screenshots, PDFs that contain images rather than text.

---

## 2. Purpose

*What is its main purpose?*

To make the enormous quantity of information that exists only as pixels searchable, queryable and processable.

---

## 3. Problem

*What engineering problem does it solve?*

```text
An invoice PDF that is a scan
    ↓
To a computer: a picture. Not searchable, not extractable.
    ↓
A human reads it and types the total into a system
    ↓
OCR replaces that step — for thousands of documents an hour
```

---

## 4. The difficulty spectrum

> [!IMPORTANT]
> OCR's reputation depends entirely on which of these you are attempting. They are not the same problem.

```text
EASY        clean scan, printed, one column, 300 dpi     → ~99%+ accuracy
MODERATE    photo of a document, some skew and shadow    → ~95%
HARD        receipts, low light, crumpled, tables        → ~80–90%
VERY HARD   handwriting, historical documents, forms
            with dense layout                            → highly variable
```

> [!CAUTION]
> **Even 99% character accuracy is not good enough for numbers.** A 12-digit invoice total at 99% per-character accuracy is correct only about 89% of the time. For anything financial, OCR output requires validation — checksums, cross-field arithmetic, or human review.

---

## 5. Architecture Position

```text
Document / photo
    ↓
PREPROCESSING  ← where most of the accuracy is won or lost
    deskew · denoise · threshold · perspective correct
    ↓
Text detection    where is text?
    ↓
Text recognition  what does it say?
    ↓
POSTPROCESSING
    layout reconstruction · validation · field extraction
    ↓
Structured data
```

---

## 6. Preprocessing is most of the work

```text
✓ Deskew                     even 2° of rotation hurts significantly
✓ Perspective correction     photographed pages are trapezoids
✓ Convert to greyscale
✓ Adaptive thresholding      handles uneven lighting
✓ Denoise
✓ Upscale if below ~300 dpi equivalent
```

> [!TIP]
> **Preprocessing with [OpenCV](OpenCV.md) typically improves accuracy more than changing the OCR engine.** Teams frequently swap engines looking for better results when a deskew and an adaptive threshold would have solved it.

---

## 7. Text extraction is not data extraction

```text
OCR returns:      "Invoice  Total  1,234.56  Date  05/08/2026"
You need:         { total: 1234.56, date: "2026-08-05" }
```

```text
POSITIONAL    "the value to the right of the word 'Total'"
              → brittle; breaks with any layout change

TEMPLATE      per-supplier coordinate templates
              → accurate, but one template per document type

LAYOUT-AWARE  models trained on document structure (LayoutLM and similar)
              → generalises across layouts

LANGUAGE MODEL  pass the OCR text and ask for structured fields
              → flexible; verify the output, it can invent plausible values
```

> [!IMPORTANT]
> **The extraction step is usually harder than the recognition step**, and it is where most document-processing projects actually spend their time.

---

## 8. PDFs — check before you OCR

```text
A PDF may contain a real text layer already
    ↓
Extract it directly — perfect accuracy, milliseconds, free
    ↓
Only fall back to OCR when there is no text layer
```

> [!TIP]
> A large proportion of PDFs sent to OCR pipelines never needed it. Testing for an existing text layer first is a few lines of code and removes both the cost and the error rate for those documents.

---

## 9. Real World Example

- **Invoice and receipt processing** — the highest-volume commercial use.
- **Identity document verification** in onboarding flows.
- **Digitising archives** — libraries, historical records, legal discovery.
- **Licence plate recognition** — a constrained, well-solved subset.
- **Accessibility** — making scanned material readable by screen readers.

---

## 10. Engine options

| Engine | Notes |
|---|---|
| **Tesseract** | Open source, mature, free; needs good preprocessing |
| **PaddleOCR** | Open source, strong on many languages, good detection |
| **EasyOCR** | Simple API, deep-learning based |
| **Cloud OCR services** | Higher accuracy on difficult input; per-page cost; data leaves your infrastructure |
| **Vision-capable language models** | Handle layout and extraction together; verify outputs |

> [!TIP]
> Start with **Tesseract or PaddleOCR plus proper preprocessing**. Move to a paid service only when you have measured that preprocessing is not enough — and weigh the data protection implications of sending documents to a third party.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use OCR when the information exists only as an image and a human currently retypes it. The business case is usually strong and easy to measure in hours saved.

> [!CAUTION]
> - **If a structured source exists — an API, a database, an EDI feed — use it.** OCR of a report generated from a database is solving a self-inflicted problem
> - **Not for anything requiring perfect accuracy** without validation and a human review path
> - **Handwriting remains unreliable** for arbitrary input

---

## 12. Advantages and Disadvantages

**Advantages**
- Unlocks information otherwise trapped in images
- Very high accuracy on clean printed text
- Mature open-source options
- Immediate, measurable labour savings

**Disadvantages**
- Accuracy collapses on poor input quality
- Layout and table reconstruction is genuinely hard
- Extraction to structured fields needs separate work
- Language and script coverage varies considerably
- Errors are subtle — a `1` read as `7` is not flagged

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Speed** | ~0.5–5 seconds per page, depending on engine and size |
| **CPU/GPU** | Deep-learning engines benefit from a GPU |
| **Batch** | Process asynchronously — never in a request handler |
| **Preprocessing** | Cheap relative to recognition, and the highest-value step |

---

## 14. Security Considerations

> [!CAUTION]
> **Documents are among the most sensitive data an organisation holds** — invoices, contracts, identity documents, medical records. An OCR pipeline concentrates all of it in one place, and it is frequently built as an internal tool with correspondingly little security review.

- **Cloud OCR sends the document to a third party** — check what they retain, and whether that is lawful for the data in question
- **Identity documents are special-category data** in many jurisdictions, with specific handling requirements
- **Delete intermediate artifacts** — preprocessed images and raw OCR text often linger in temporary storage
- **Uploaded documents are untrusted input** — PDFs in particular have a long history of parser vulnerabilities; process them in an isolated worker
- **Validate extracted values before acting on them** — an OCR-derived bank account number used without verification is a fraud vector

---

## 15. Mental Model

> [!NOTE]
> **OCR is reading someone else's handwriting through a dirty window.**
>
> Clean the window first — that is preprocessing, and it helps more than squinting harder. Neat block capitals are easy; a hurried scrawl is guesswork. And you will read "1234" as "1284" occasionally without noticing, which is why anything that matters gets checked.

---

## 16. Mini Architecture Diagram

```text
Upload
    ↓
Is it a PDF with a text layer? → extract directly, done
    ↓ no
OpenCV: deskew · perspective · threshold · denoise
    ↓
Text detection → text recognition
    ↓
Field extraction (template or model)
    ↓
VALIDATION: checksums, totals, formats
    ↓
Confident → automatic     Uncertain → human review queue
```

---

## 17. Complete Request Flow

Invoice processing:

```text
Invoice uploaded
    ↓
File type and size validated
    ↓
Enqueued for background processing
    ↓
PDF text layer present? → extract, skip OCR entirely
    ↓ no
Rendered to an image at 300 dpi
    ↓
Deskewed, perspective-corrected, adaptively thresholded
    ↓
OCR → text with per-word confidence scores
    ↓
Field extraction: supplier, date, line items, total
    ↓
VALIDATION: do the line items sum to the total?
    ↓ YES → high confidence → posted automatically
    ↓ NO  → flagged for human review with the image alongside
    ↓
Human correction stored — and used to improve extraction
    ↓
Intermediate images deleted
```

> [!TIP]
> The arithmetic check in the middle is worth more than any accuracy improvement in the engine. **Validate against the document's own internal consistency** wherever it exists — invoices, forms and statements almost always contain a checkable relationship.

---

## 18. Key Takeaway

> [!IMPORTANT]
> Preprocessing determines OCR accuracy more than engine choice, extraction to structured fields is a separate and harder problem, and any number that matters must be validated.

---

## 19. Common Mistakes

- **OCR-ing PDFs that already contain text**
- **Skipping preprocessing** and blaming the engine
- **Trusting extracted numbers** without validation
- **Positional extraction** that breaks on any layout change
- **Running OCR in a request handler** rather than a background worker
- **Retaining intermediate images** containing sensitive documents
- **Sending confidential documents to a cloud service** without checking retention terms
- **No human review path** for low-confidence results

---

## 20. Open Source Technologies

- **Tesseract**, **PaddleOCR**, **EasyOCR**, **docTR** — engines
- **OpenCV** — preprocessing
- **pdfplumber**, **PyMuPDF** — extract existing PDF text layers
- **LayoutLM**, **Donut** — layout-aware document understanding
- **Label Studio** — annotating documents for extraction training

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Run OCR on a document before and after deskewing and thresholding, and compare the output.
- [ ] Check what proportion of your PDFs already contain a text layer.
- [ ] Find one internal consistency check in your documents that could validate the extraction.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Document → preprocessing → detection → recognition → extraction → validation
```

## 2. Request Flow

```text
Input       a scanned or photographed document
    ↓
Processing  cleaned, text located and recognised, fields extracted
    ↓
Output      structured data, validated, with uncertain cases escalated
```

## 3. Real-World Usage

**Invoice processing** is OCR's largest commercial application, and its architecture is instructive: the accuracy of the engine matters less than the preprocessing before it and the arithmetic validation after it.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Extracting machine-readable text from images |
| **Why does it exist?** | Because vast amounts of information exist only as pixels |
| **Where does it belong?** | In a background pipeline, between upload and structured data |
| **When should I use it?** | When no structured source exists and a human is currently retyping |
