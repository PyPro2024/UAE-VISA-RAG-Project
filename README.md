# UAE Visa Q&A: a RAG Assistant (built step by step)

A Retrieval-Augmented Generation (RAG) assistant that answers questions about UAE residence and visa categories using **official ICP and GDRFA documents**, and shows the source file and page for every result.

I'm building it in six weekly stages and documenting what worked, what broke and what I measured.

**Status:** Week 2 of 6 complete (ingestion, cleaning, chunking, vector search with sources).

> **Disclaimer:** This is a learning project, not legal or immigration advice. Visa rules change often. Always confirm requirements on the official ICP, GDRFA and MOHRE websites.

## How it works

```
Official PDFs -> extract text (OCR when needed) -> clean -> split into chunks
   -> embed each chunk -> store in Chroma
Question -> embed -> find the most similar chunks -> show text + file + page
```

Week 3 adds a language model that writes the final answer from the retrieved chunks, with citations.

## Repository contents

| File | What it does |
|---|---|
| `01_embeddings_similarity_search.ipynb` | Week 1: embeddings and similarity search on practice sentences (sample text, not verified rules) |
| `02_ingest_chunk_chroma.ipynb` | Week 2: PDF ingestion, OCR, cleaning, chunking, Chroma index, search with sources, evaluation |
| `sources.csv` | The official pages used, with URLs and download dates |

The PDFs, `pages.json` and the `chroma_db/` folder are not committed. Download the original pages from the links in `sources.csv`.

## Data

Seven official pages saved as PDFs on 15 October 2026 (45 pages in total):

- ICP: Green Residence Guide, Golden Residency Guide, Blue Residence Guide
- ICP: Issuance of a Visa, Issuance of a Residence Visa for an Entrepreneur
- GDRFA Dubai: Green visa (self-employment), Green visa (high-level skilled worker)

A news article about the "AI specialist visa" was left out on purpose because it is a secondary source.

## Tech

Python, PyMuPDF, RapidOCR, sentence-transformers (`all-MiniLM-L6-v2`), ChromaDB (cosine similarity). Developed on Kaggle notebooks (CPU only).

## What I did and why

1. **OCR was needed.** The saved PDFs had no text layer, so normal text extraction returned nothing. I render each page and read it with RapidOCR.
2. **Cleaning.** I removed repeated headers, timestamps, footer URLs, page numbers and garbled OCR banner lines, per document. Without this, the same header text was embedded into many chunks and added noise to the search.
3. **Chunking with metadata.** Text is split into overlapping word chunks. Each chunk keeps its file name and page number, and the document title is added when embedding so categories are easier to tell apart.
4. **Vector store.** Chunks are embedded and stored in Chroma. A search returns the text, file, page and a similarity score.
5. **"Not in the documents" check.** If the best score is below 0.30, the search warns that the question is probably out of scope. The cutoff is provisional.

## Evaluation (12 in-scope + 2 out-of-scope questions)

The expected source pages were set by hand after reading the documents.

| Chunk size / overlap | Chunks | Top-1 correct | Top-5 contains the right page | Out-of-scope rejected |
|---|---|---|---|---|
| 60 / 15 | 107 | 6/12 | 11/12 | 2/2 |
| 120 / 30 | 60 | 8/12 | 11/12 | 2/2 |
| 200 / 40 | 48 | 7/12 | 12/12 | 2/2 |

I kept **200 words / 40 overlap** because the right page appeared in the top 5 for all 12 questions, and the language model in Week 3 will read the top 5 chunks. Most pages are shorter than 200 words, so this is close to one chunk per page. The trade-off is that 120/30 ranked the correct page first more often (8/12 vs 7/12).

Example search output:

```
Q: What are the requirements for a green visa for self-employment?
   0.71 | GDRFAD_GreenVisa_Selfemployment p.1
   0.60 | GDRFAD_Issuance_of_a_green_vis(Expert) p.1
   0.57 | ICP_Green Residence Guide p.4

Q: Who wins the football world cup?
   ⚠ Best score 0.12 is below 0.3: probably not in the documents.
```

## Fact check against the official text

I searched the extracted pages for salary figures. These lines appear in the documents:

- Green Residence Guide, p.4: a minimum monthly salary of AED 15,000 (the line is cut off at the end by OCR line breaks).
- Golden Residency Guide, p.11: a monthly salary of at least AED 30,000 (inside one category; I have not yet checked which one).
- Issuance of a Visa, p.4: AED 4,000, 8,000 and 15,000 for sponsoring visitors, depending on the relationship.

## Known limitations

- **Ranking is not perfect.** The correct page is first in only 6 to 8 of 12 questions. Example: "Can I sponsor my family on a green visa?" ranks the high-level skilled worker page above the Green guide page that lists sponsorship benefits.
- **Retrieval is not reasoning.** It finds the relevant text but cannot answer yes or no questions. That needs the language model in Week 3.
- **OCR can make mistakes**, especially with numbers and tables. Spot-check figures against the original PDFs.
- **Small test set.** Twelve questions are too few to tune the 0.30 cutoff, and the expected pages were chosen by hand. Week 4 builds a larger test set.
- **Documents are a snapshot** from 15 October 2026 and will go out of date.

## Roadmap

- [x] Week 1: embeddings and similarity search
- [x] Week 2: ingest official PDFs, OCR, cleaning, chunking, Chroma, evaluation
- [ ] Week 3: answer with a language model, with citations and "I don't know"
- [ ] Week 4: larger evaluation set and a tuned score cutoff
- [ ] Week 5: FastAPI backend, simple UI, Docker
- [ ] Week 6: deployment and demo

## Run it

1. Download the pages listed in `sources.csv` as PDFs and add them as an input dataset (Kaggle: Add Input -> Upload).
2. Open `02_ingest_chunk_chroma.ipynb` with Internet on and run the cell. The first run does OCR (a few minutes) and saves `pages.json`. Later runs load it.
