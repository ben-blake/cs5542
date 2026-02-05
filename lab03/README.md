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

**Grounded Answer (Extractive Baseline):**
> [TEXT | doc1.pdf::p11::c4500 | score=-6.947]  for which a health care facility or business associate, as applicable, determines there is a low probability of compromise in accordance with HIPAA’s 4-factor risk assessment (see “Analysis of Risk of Harm” section below for a complete listing of these facto
> Images: ['img1.png', 'img10.png', 'img2.png']

### Query 2 Example
**Question:** "Using both the Zero Trust architecture diagram and the document text, what core principle is emphasized for access decisions?"

**Grounded Answer (Extractive Baseline):**
> [TEXT | doc4.pdf::p45::c0 | score=0.097] 45 where they can be securely communicated to a platform. The foundational building block that zero trust architecture usually consists of several aspects: • Each entity can create proof of whom the identity is • Entities can independently authenticate other i
> [TEXT | doc4.pdf::p44::c1500 | 
> Images: ['img1.png', 'img2.png', 'img10.png']

### Query 3 Example
**Question:** "What specific encryption algorithm (for example, AES-256 or RSA-2048) is mandated by the organization’s policy?"

**Grounded Answer (Extractive Baseline):**
> [TEXT | doc5.pdf::p33::c3000 | score=-3.432] Inspected the company's Cryptography Policy to determine that the company restricts privileged access to encryption keys to authorized users with a business need. No exceptions noted.
> [TEXT | doc4.pdf::p28::c0 | score=-3.856] 28 path, size, and frequency of access as well regulations, compliance or ad
> Images: ['img1.png', 'img2.png', 'img10.png']

![Answer Screenshot](screenshots/screenshot2.png)
*(Screenshot 2: Generated Answer)*

## 5. Reflection & Failure Analysis

### Failure Case
**Scenario:** Query Q3 ("Encryption algorithm mandated") using Sparse Retrieval.
**Failure:** The system returned high-ranking chunks about "security policies" generally but missed the specific table row defining "AES-256".
**Cause:** The term "mandated" did not exactly match the table header "Required Standards", leading to a lexical mismatch.

### Proposed Improvement
**Solution:** Implement **Multimodal Table Extraction**. Instead of treating PDF pages as flat text, use a table parsing tool (like `pdfplumber` or `unstructured`) to structure table rows as distinct, high-quality chunks. This would allow the dense retriever to map "mandated" to "Required Standards" more effectively.
