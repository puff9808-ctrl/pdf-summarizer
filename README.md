# PDF Summarizer

![CI](https://github.com/puff9808-ctrl/pdf-summarizer/actions/workflows/ci.yml/badge.svg)

Upload a text-based PDF and get back its most important sentences, plus keywords.
Built with **Python, FastAPI, pypdf and scikit-learn**. It needs no API keys and
runs fully offline.

## How it works

This is an **extractive** summarizer: the summary is made of sentences copied from
your document, ranked by importance. It does not generate new text.

1. **Extract** - `pypdf` pulls the text out of every page.
2. **Clean and split** - hyphenated line breaks are joined, the text is split into
   sentences, and fragments and repeated headers/footers are dropped.
3. **Vectorize** - each sentence becomes a TF-IDF vector: words that are frequent in
   a sentence but rare across the document get higher weight, and stop words are ignored.
4. **Rank** - two methods are available:
   - **TF-IDF**: score a sentence by the sum of its term weights.
   - **TextRank**: build a graph where sentences are nodes and edges are cosine
     similarities, then run PageRank (power iteration, damping 0.85). Sentences
     similar to many other sentences score higher.
5. **Select** - the top *n* sentences are returned in their original document order.

## Run it

```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

Open http://localhost:8000. Interactive API docs are at http://localhost:8000/docs.

With Docker:

```bash
docker build -t pdf-summarizer .
docker run -p 8000:8000 pdf-summarizer
```

## API

`POST /api/summarize` (multipart form)

| Field | Type | Default | Notes |
|---|---|---|---|
| `file` | PDF | required | up to 10 MB and 300 pages |
| `sentences` | int | 5 | 1 to 20 |
| `method` | string | `textrank` | `textrank` or `tfidf` |

```bash
curl -F "file=@paper.pdf" -F sentences=5 http://localhost:8000/api/summarize
```

Response:

```json
{
  "filename": "paper.pdf", "pages": 12, "words": 5400,
  "method": "textrank", "total_sentences": 210,
  "summary": ["...", "..."], "keywords": ["..."]
}
```

Errors return a JSON `detail` message: `413` file too large, `422` invalid PDF,
password-protected, no extractable text, or invalid parameters.

## Tests

```bash
pip install -r requirements-dev.txt
pytest
```

## Limitations

- Scanned PDFs (images of text) have no text layer and are rejected; they would need OCR first.
- Extractive summaries can read choppily and can miss ideas spread across many sentences.
- Sentence splitting is rule-based, so unusual abbreviations can split a sentence early.
- Documents over 2,500 sentences are pre-filtered by TF-IDF score before TextRank runs, to bound memory.

## Project structure

```
app/
  main.py        FastAPI app and routes
  pdf_utils.py   PDF text extraction and validation
  summarizer.py  sentence splitting, TF-IDF, TextRank
static/          single-page web UI (HTML, CSS, JS)
tests/           pytest suite (unit and API tests)
```

## Ideas to extend it

- OCR fallback for scanned PDFs (Tesseract)
- Abstractive summaries using a language model API
- Per-page or per-chapter summaries
