# Marginalia

*Private PDF search that runs entirely in your browser.*

Add one or more PDFs, ask a question in plain English, and get the passages that answer it, with the page number. Everything runs inside your browser tab. There is no server, no account and no API key, and your files are never uploaded.

> **Status:** search works. Chat-style written answers are next on the roadmap (see below).

<!-- Add a demo GIF here once you have one:
![Demo](docs/demo.gif)
-->

## Why I built it

Most "chat with your PDF" tools upload your files to someone else's server and charge for the language model behind them. I wanted to see how far a fully local, fully free version could go. That means the privacy story is simple (nothing leaves the device) and the hosting cost is zero (it is one static page).

## What it does

- **Search by meaning, not just keywords.** Ask "what did the author conclude?" and it finds the right passage even if those exact words are not on the page.
- **Works across several PDFs at once.** Add as many as you like, then leave any of them out of the search with one click.
- **Reads scanned and scrambled pages.** Pages with no text layer, or a broken one (common in designed magazines), are read with OCR instead.
- **Shows its work.** Each result shows the document, the page number, the matched words highlighted, and a Strong, Good or Weak match label.
- **Says so when nothing matches.** Results under a minimum score are hidden rather than showing the "closest" irrelevant text.
- **Private by design.** Files are read in memory in your tab and are gone when you close it.

## How it works

```
PDF file
  -> PDF.js reads the text of each page
  -> page text looks wrong or is missing?  -> Tesseract.js OCR reads the page image instead
  -> text is cut into overlapping chunks (180 words, step 150), each remembering its page
  -> Transformers.js turns every chunk into a vector (a list of numbers that captures meaning)

Your question
  -> turned into a vector the same way
  -> every chunk scored: cosine similarity + 0.35 x keyword score
  -> top 8 passages above the minimum score are shown
```

A few ideas worth knowing:

- **Embeddings.** A small AI model (`all-MiniLM-L6-v2`, about 25 MB) turns text into vectors so that passages with similar meaning end up close together. Closeness is measured with cosine similarity.
- **Hybrid scoring.** Meaning-based search is weak on names and rare words, so a keyword score is added. Rare words count for more than common ones (an idf weighting), so searching for a person's name works.
- **Overlapping chunks.** Passages overlap so an answer sitting on a chunk boundary is still found.
- **Web Worker.** The search model runs in a background thread, so the page stays responsive while it loads and indexes.
- **Garbage-text detection.** A heuristic (`looksGood`) flags pages that are empty, mostly symbols, or mostly one- and two-letter "words", and sends them to OCR.

More detail, design decisions and the data model are in [`ARCHITECTURE.md`](ARCHITECTURE.md).

## Tech stack

| Job | Library |
|---|---|
| Read PDF text and render pages | [PDF.js](https://mozilla.github.io/pdf.js/) 4.10.38 |
| OCR for scanned pages | [Tesseract.js](https://tesseract.projectnaptha.com/) 5.1.1 |
| Embeddings in the browser | [Transformers.js](https://huggingface.co/docs/transformers.js) 3.3.3 |
| Embedding model | `Xenova/all-MiniLM-L6-v2` (q8) |

All open source, loaded from the jsDelivr CDN. No build step.

## Run it locally

The page must be served over http. Opening the file directly will not work.

```bash
python3 -m http.server 8000
```

Then open <http://localhost:8000> in a modern browser. The first load needs internet to fetch the libraries and the search model (about 25 MB, with a progress percentage shown). After that the browser caches them. If you open `index.html` directly from your files instead, it still runs, but the model is downloaded again on every visit.

## Deploy for free

1. Push this folder to a GitHub repository.
2. In the repository, go to **Settings > Pages**.
3. Under **Build and deployment**, choose **Deploy from a branch**, pick `main` and `/ (root)`, and save.
4. Your site will be live at `https://<your-username>.github.io/<repository-name>/` after a minute or two.

## Known limitations

- Needs internet on the first visit (library and model downloads).
- Embeddings are recomputed on every reload. There is no cache yet.
- Tables come out as messy text.
- Handwriting, stylised lettering and blurry scans OCR poorly, and those pages are skipped.
- OCR is English only. A misspelled or mis-read name will not keyword-match.
- The search model is English-focused. PDFs in other languages (Hindi, Telugu and so on) are read correctly when they have a text layer, but the meaning-based search will be weaker on them. OCR is English only.
- It finds passages but does not yet write an answer.
- The garbage-text and scoring thresholds (40 characters, 55% letters, 50% tiny words, OCR confidence 45, minimum score 0.22, keyword weight 0.35) are hand-tuned guesses and need testing on more PDFs.

## Roadmap

- [x] Multi-PDF search with a sidebar, queue and progress
- [x] OCR fallback for scanned and scrambled pages
- [x] Hybrid meaning and keyword scoring with a minimum score
- [x] Redesigned "reading desk" interface with light and dark themes
- [ ] **Chat answers** with a small in-browser language model (WebLLM, needs WebGPU), citing `[document, page]` and saying "I don't know" when the passages do not answer. Search stays as the fallback when WebGPU is missing.
- [ ] **IndexedDB cache** keyed by file hash, so reloads skip re-reading and re-embedding
- [ ] **Better retrieval:** BM25 plus vector rank fusion, optional reranking, and chunking along headings and paragraphs
- [ ] **Show sources:** the cited page image beside each answer
- [ ] **Tables and figures:** table-aware extraction, OCR for text inside images
- [ ] More OCR languages
- [ ] Demo GIF and architecture diagram

## Project files

| File | Purpose |
|---|---|
| `index.html` | The whole app: markup, styles and script |
| `ARCHITECTURE.md` | Pipeline, data model, design decisions and roadmap notes |
| `README.md` | This file |

## Contributing

Issues and suggestions are welcome, especially PDFs where search or OCR gives poor results. Please describe the kind of document (scanned, designed magazine, table-heavy) but do not attach private files.

## License

Released under the [MIT License](LICENSE).

## Acknowledgements

Built on the work of the PDF.js team at Mozilla, the Tesseract OCR project, Hugging Face (Transformers.js) and the authors of the MiniLM model.
