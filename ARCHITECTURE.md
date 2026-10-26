# Marginalia: architecture and project notes

## What this is
Marginalia is a static web app (one `index.html`, no backend) where you add one or more PDFs and search them by meaning. It runs entirely in the browser, so files never leave the device. Hosting and running it are free (GitHub Pages works). Built as a portfolio and LinkedIn project.

## Stack (all open source, loaded from the jsDelivr CDN)
| Job | Library | Version |
|---|---|---|
| Read PDF text, render pages | PDF.js (`pdfjs-dist`) | 4.10.38 |
| OCR for scanned pages | Tesseract.js | 5.1.1 |
| Embeddings in the browser | Transformers.js (`@huggingface/transformers`) | 3.3.3 |
| Embedding model | `Xenova/all-MiniLM-L6-v2`, q8 (about 25 MB) | |

## Pipeline (function names in `index.html`)
1. `addFiles` creates a document record per PDF and queues it. PDFs are processed one at a time (`queue`).
2. `processDoc` reads each page with PDF.js `getTextContent()`.
3. `looksGood` checks the page text. If it is missing, too short (under 40 characters), mostly symbols, or mostly 1-2 letter "words" (a scrambled text layer, common in designed magazines), the page is OCR'd instead: `ocrPage` renders it to a canvas (scale 2) and runs Tesseract. OCR output is only used if its confidence is 45 or higher and it passes `looksGood`; otherwise the page is skipped and counted as unreadable. Each chunk is also checked before it is indexed.
4. `split` cuts page text into overlapping chunks (180 words, step 150), each remembering its page.
5. `embed` turns chunks (batches of 8) into normalized vectors with Transformers.js. The model runs inside a Web Worker (`WORKER_SRC`), so the page stays responsive while it downloads and runs. If the worker cannot start, `embed` falls back to the main thread. Download progress is shown in the status line and the sidebar.
6. `search` embeds the question and scores every chunk of every active document: cosine similarity (dot product of normalized vectors) plus 0.35 times a keyword score. The keyword score weights rare words (like a person's name) more than common ones, because pure meaning-based search is weak on names. Passages scoring under `MIN_SCORE` (0.22) are hidden, and if none pass the app says no strong match was found. Results show the top 8 with keywords highlighted and a Strong, Good or Weak match label.

## Data model
```js
doc = { id, name, color, pages, ocrPages, chunks: [{page, text}], vectors: [[...]],
        status: 'queued'|'working'|'ready'|'error', progress, note, active, removed }
```

## Decisions and why
- **OCR trigger (`looksGood`):** pages with no text layer are scans, and pages with a scrambled text layer are just as unusable, so both go to OCR. Thresholds (40 characters, 55% letters, 50% tiny words, OCR confidence 45) are heuristics, tuned by hand and worth revisiting with more test PDFs.
- **Hybrid scoring and a minimum score:** semantic search alone returns the "closest" passages even when nothing is relevant, so a floor (`MIN_SCORE`) and a keyword boost were added. Both numbers are guesses to tune.
- **PDF worker loaded via a blob:** loading the worker straight from the CDN was blocked as a cross-origin worker, which forced PDF.js onto the main thread. The blob avoids that.
- **Embedding model in a Web Worker:** a first test run froze the page (Chrome showed "Page Unresponsive") because loading the model and starting the WASM runtime blocked the main thread, with no visible progress. The worker removes the freeze, and `progress_callback` shows the download percentage. PDF.js is also loaded lazily (`getPdfjs`), so the page and its buttons work immediately even on a slow network.
- **Failures are contained:** an OCR failure on one page skips that page instead of failing the whole PDF, and a model-load failure shows its own message instead of "could not read this file".
- **Letters plus combining marks:** `looksGood` counts `\p{L}` and `\p{M}`. Scripts like Hindi and Telugu write vowel signs as combining marks, so counting letters alone wrongly flagged real text as garbage.
- **Stop words and near-duplicate results:** common words ("the", "and") are ignored for keyword scoring and highlighting, and neighbouring overlapping chunks are not both shown as separate results.
- **Sequential queue:** keeps the browser responsive and memory use predictable.
- **Overlapping chunks:** so an answer that sits on a chunk boundary is still found.
- **No backend:** zero cost and a strong privacy story.

## Run it
Serve over http (not by double-clicking): `python3 -m http.server 8000`, then open `http://localhost:8000`. Or push to GitHub and enable Pages. If you open the file directly (`file://`), the model may be re-downloaded on every visit because the browser cannot cache it for that origin.

## Known limitations
- Needs internet on first load (CDN libraries and model downloads), then the browser caches them.
- Embeddings are recomputed on every reload (no cache yet).
- Tables extract as messy text. Handwriting, stylized lettering and blurry scans OCR poorly (such pages are skipped). OCR is English only. A misspelled or mis-OCR'd name will not keyword-match.
- It finds passages but does not yet write answers.

## Roadmap with design notes
1. **Chat answers (WebLLM).** Pick a small model from WebLLM's prebuilt list (look up current model IDs). Needs WebGPU (Chrome or Edge on desktop). Prompt: answer only from the numbered passages, cite `[document, page]`, say "I don't know" if the passages do not answer. Stream tokens. Keep the current search view as the fallback when WebGPU is missing.
2. **IndexedDB cache.** Key by file hash; store vectors as `Float32Array`. Skip reading, OCR and embedding on repeat uploads.
3. **Better retrieval.** Chunk along headings and paragraphs; upgrade the simple hybrid score to proper BM25 plus vector rank fusion (reciprocal rank fusion); optional rerank of the top 20.
4. **Show sources.** Render the cited page image next to the answer; highlight the passage.
5. **Tables and figures.** Table-aware extraction; OCR for text inside images.
6. **More OCR languages.**
7. **Polish for the portfolio.** Demo GIF, architecture diagram, short write-up of the hard problems (chunking, scanned-page detection, worker issue), LinkedIn post.

## Interface design
- **Concept:** a reading desk. A fog-grey surface holds the document list; the white "sheet" on top holds the question and the results, laid out like a finding aid.
- **Type:** Archivo (expanded, for the name, the question and page numbers) with Literata (a book face, for the passages). Loaded from Google Fonts with Georgia and Arial as fallbacks.
- **Structure carries meaning:** the page number is the large element in each result because it is the citation; the document's colored tab in the sidebar matches the tab beside each result; result rank is shown by size (the best passage is set larger); left-out documents lose their color.
- **One motion:** matched words are highlighted with a quick highlighter sweep when results appear. It is disabled for people who prefer reduced motion.
- **Palette:** fog `#e8ebef`, sheet `#ffffff`, ink `#12151c`, cobalt `#2a3bd1`, highlighter `#ffe44d`, plus eight muted document colors. A dark theme is included.
- **Copy:** plain verbs ("Add PDFs", "Search", "Leave out of search"), and errors say what happened and what to do.

## Untested status
Versions 1 (single PDF), 2 (multi-PDF) and 4 (interface redesign) were tested earlier. Version 3 (garbage-text detection with OCR fallback, OCR confidence, hybrid keyword scoring, minimum score and match labels) was first run in a browser in a screen recording, where the page froze while loading the search model. Version 5 moves the model into a Web Worker, adds download progress and fixes several smaller bugs. It was syntax-checked, and the text-quality check and chunking were unit-tested on sample text, but it has not been run end to end in a browser yet. Next step: run it over http with a real PDF and the scrambled-text magazine PDF, and tune the thresholds from the results.
