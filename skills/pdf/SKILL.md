---
name: pdf
description: PDF操作スキル。読み取り、テキスト/テーブル抽出、結合、分割、新規作成、フォーム入力、暗号化、OCRなど。.pdfファイルに関する作業時に使用。
category: ユーティリティ
version: 1.0.0
tags:
  - pdf
  - document
  - reportlab
---

# PDF Processing Guide

Source: [anthropics/skills](https://github.com/anthropics/skills) (Official)

## Quick Start

```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("document.pdf")
print(f"Pages: {len(reader.pages)}")
text = "".join(page.extract_text() for page in reader.pages)
```

## Python Libraries

### pypdf - Basic Operations

#### Merge PDFs
```python
from pypdf import PdfWriter, PdfReader

writer = PdfWriter()
for pdf_file in ["doc1.pdf", "doc2.pdf", "doc3.pdf"]:
    reader = PdfReader(pdf_file)
    for page in reader.pages:
        writer.add_page(page)

with open("merged.pdf", "wb") as output:
    writer.write(output)
```

#### Split PDF
```python
reader = PdfReader("input.pdf")
for i, page in enumerate(reader.pages):
    writer = PdfWriter()
    writer.add_page(page)
    with open(f"page_{i+1}.pdf", "wb") as output:
        writer.write(output)
```

#### Rotate Pages
```python
reader = PdfReader("input.pdf")
writer = PdfWriter()
page = reader.pages[0]
page.rotate(90)
writer.add_page(page)
with open("rotated.pdf", "wb") as output:
    writer.write(output)
```

### pdfplumber - Text and Table Extraction

```python
import pdfplumber

with pdfplumber.open("document.pdf") as pdf:
    for page in pdf.pages:
        text = page.extract_text()
        tables = page.extract_tables()
```

#### Table to DataFrame
```python
import pandas as pd
import pdfplumber

with pdfplumber.open("document.pdf") as pdf:
    all_tables = []
    for page in pdf.pages:
        for table in page.extract_tables():
            if table:
                df = pd.DataFrame(table[1:], columns=table[0])
                all_tables.append(df)
    if all_tables:
        combined = pd.concat(all_tables, ignore_index=True)
```

### reportlab - Create PDFs

```python
from reportlab.lib.pagesizes import letter, A4
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, PageBreak
from reportlab.lib.styles import getSampleStyleSheet

doc = SimpleDocTemplate("report.pdf", pagesize=A4)
styles = getSampleStyleSheet()
story = [
    Paragraph("Report Title", styles['Title']),
    Spacer(1, 12),
    Paragraph("Body text here. " * 20, styles['Normal']),
    PageBreak(),
    Paragraph("Page 2", styles['Heading1']),
]
doc.build(story)
```

**IMPORTANT**: Never use Unicode subscript/superscript characters in ReportLab. Use `<sub>` and `<super>` tags instead:
```python
Paragraph("H<sub>2</sub>O and x<super>2</super>", styles['Normal'])
```

## Command-Line Tools

```bash
# pdftotext (poppler-utils)
pdftotext -layout input.pdf output.txt
pdftotext -f 1 -l 5 input.pdf output.txt

# qpdf
qpdf --empty --pages file1.pdf file2.pdf -- merged.pdf
qpdf input.pdf --pages . 1-5 -- pages1-5.pdf

# OCR (scanned PDFs)
pip install pytesseract pdf2image
```

### OCR Scanned PDFs
```python
import pytesseract
from pdf2image import convert_from_path

images = convert_from_path('scanned.pdf')
text = "\n\n".join(
    f"Page {i+1}:\n{pytesseract.image_to_string(img)}"
    for i, img in enumerate(images)
)
```

### Password Protection
```python
writer = PdfWriter()
for page in PdfReader("input.pdf").pages:
    writer.add_page(page)
writer.encrypt("userpassword", "ownerpassword")
with open("encrypted.pdf", "wb") as f:
    writer.write(f)
```

### Add Watermark
```python
watermark = PdfReader("watermark.pdf").pages[0]
reader = PdfReader("document.pdf")
writer = PdfWriter()
for page in reader.pages:
    page.merge_page(watermark)
    writer.add_page(page)
with open("watermarked.pdf", "wb") as f:
    writer.write(f)
```

## Quick Reference

| Task | Tool | Key API |
|------|------|---------|
| Merge | pypdf | `writer.add_page(page)` |
| Split | pypdf | One page per file |
| Extract text | pdfplumber | `page.extract_text()` |
| Extract tables | pdfplumber | `page.extract_tables()` |
| Create PDF | reportlab | SimpleDocTemplate + Platypus |
| CLI merge | qpdf | `qpdf --empty --pages ...` |
| OCR | pytesseract | Convert to image first |
| Encrypt | pypdf | `writer.encrypt(pw)` |
