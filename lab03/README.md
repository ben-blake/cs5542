# Lab 3: Multimodal RAG Systems & Retrieval Evaluation

Ben Blake <bebpph@umsystem.edu>

## 1. Dataset Description

**Project Domain:** Cybersecurity & Risk Management

**Data Sources:**
- **PDFs:** 5 documents (NIST Framework, Data Breach Laws, Zero Trust Architecture, etc.)
- **Images:** 10 figures (Risk matrices, Architecture diagrams, Charts)
- **Total Volume:** ~220 pages of text, 10 key figures.

**Relevance:**
The dataset represents a real-world compliance knowledge base where answers require synthesizing text from policy documents and interpreting visual diagrams like risk heatmaps and architecture flows.

## 2. Methodology

We implemented a Full-Stack Multimodal RAG pipeline with the following components:

### Ingestion & Chunking
- **PDF Extraction:** PyMuPDF (`fitz`) used to extract text.
- **Chunking Strategies:**
    - **Page-based:** Each page is a chunk. Preserves document structure.
    - **Fixed-size:** 900 characters with 150 overlap. Ensures consistent context size.
- **Image Processing:** Images are indexed via captions/filenames.

### Indexing & Retrieval
- **Sparse:** TF-IDF and BM25 (Keyword matching).
- **Dense:** `sentence-transformers/all-MiniLM-L6-v2` with FAISS (Semantic search).
- **Hybrid:** Weighted fusion of Sparse + Dense scores.
- **Reranking:** `cross-encoder/ms-marco-MiniLM-L-6-v2` re-scores top candidates.

## 3. Results Table

| Query | Method | Chunking | P@5 | P@10 | Faithfulness |
|-------|--------|----------|-----|------|--------------|
| Q1 | sparse | page | 0.4 | 0.2 | 0.67 |
| Q1 | bm25 | page | 1.0 | 0.5 | 0.67 |
| Q1 | dense | page | 0.4 | 0.2 | 0.67 |
| Q1 | hybrid | page | 0.2 | 0.1 | 0.67 |
| Q1 | hybrid_rerank | page | 0.2 | 0.1 | 0.67 |
| Q1 | sparse | fixed | 0.4 | 0.2 | 0.67 |
| Q1 | bm25 | fixed | 0.8 | 0.4 | 0.67 |
| Q1 | dense | fixed | 0.2 | 0.1 | 0.67 |
| Q1 | hybrid | fixed | 0.2 | 0.1 | 0.67 |
| Q1 | hybrid_rerank | fixed | 0.2 | 0.1 | 0.67 |
| Q2 | sparse | page | 0.6 | 0.3 | 0.33 |
| Q2 | bm25 | page | 0.6 | 0.3 | 0.33 |
| Q2 | dense | page | 0.2 | 0.1 | 0.33 |
| Q2 | hybrid | page | 0.2 | 0.1 | 0.33 |
| Q2 | hybrid_rerank | page | 0.2 | 0.1 | 0.33 |
| Q2 | sparse | fixed | 0.8 | 0.4 | 0.33 |
| Q2 | bm25 | fixed | 0.6 | 0.3 | 0.33 |
| Q2 | dense | fixed | 0.4 | 0.2 | 0.33 |
| Q2 | hybrid | fixed | 0.6 | 0.3 | 0.33 |
| Q2 | hybrid_rerank | fixed | 0.6 | 0.3 | 0.33 |
| Q3 | sparse | page | 0.2 | 0.1 | 1.00 |
| Q3 | bm25 | page | 0.2 | 0.1 | 1.00 |
| Q3 | dense | page | 0.2 | 0.1 | 1.00 |
| Q3 | hybrid | page | 0.2 | 0.1 | 1.00 |
| Q3 | hybrid_rerank | page | 0.2 | 0.1 | 1.00 |
| Q3 | sparse | fixed | 0.2 | 0.1 | 1.00 |
| Q3 | bm25 | fixed | 0.0 | 0.0 | 1.00 |
| Q3 | dense | fixed | 0.2 | 0.1 | 1.00 |
| Q3 | hybrid | fixed | 0.2 | 0.1 | 1.00 |
| Q3 | hybrid_rerank | fixed | 0.2 | 0.1 | 1.00 |

## 4. Evidence & Answers

![Retrieval Screenshot](screenshots/screenshot1.png)
*(Screenshot 1: Retrieval output showing text and image evidence)*

### Query 1 Example
**Question:** "Based on the risk matrix shown in the figures and the accompanying text, which combination of likelihood and impact corresponds to the highest risk level?"

**Retrieved Evidence (Hybrid + Rerank):**
- `[IMAGE | img1.png | score=2.103] caption=img1`
- `[TEXT | doc3.pdf::p16 | score=1.854] Preparing to create and use Organizational Profiles...`

**Grounded Answer (Extractive Baseline):**
> [TEXT | doc3.pdf::p16 | score=1.854] Preparing to create and use Organizational Profiles involves gathering information about organizational priorities, resources, and risk direction from executives.
> [IMAGE | img1.png | score=2.103] caption=img1

![Answer Screenshot](screenshots/screenshot2.png)
*(Screenshot 2: Generated Answer)*

## 5. Reflection & Failure Analysis

### Failure Case
**Scenario:** Query Q3 ("Encryption algorithm mandated") using Sparse Retrieval.
**Failure:** The system returned high-ranking chunks about "security policies" generally but missed the specific table row defining "AES-256".
**Cause:** The term "mandated" did not exactly match the table header "Required Standards", leading to a lexical mismatch.

### Proposed Improvement
**Solution:** Implement **Multimodal Table Extraction**. Instead of treating PDF pages as flat text, use a table parsing tool (like `pdfplumber` or `unstructured`) to structure table rows as distinct, high-quality chunks. This would allow the dense retriever to map "mandated" to "Required Standards" more effectively.
