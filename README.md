# 🚗 Automotive Specification Extraction – RAG Pipeline

*A Retrieval-Augmented Generation system for extracting structured vehicle specifications from unstructured service manuals.*

---

## 📘 1. Project Overview

Automotive service manuals contain thousands of torque specifications, dimensions, fluid capacities, and mechanical parameters distributed across dense text and engineering diagrams. Extracting these values manually is slow and error-prone.

This project implements a **Retrieval-Augmented Generation (RAG)** pipeline that automatically extracts such specifications from PDF service manuals and returns structured JSON output such as:

```json
{
  "component": "Rear Brake Caliper Bolt",
  "value": "35",
  "unit": "Nm",
  "source_page": 54
}
```

The system uses a hybrid retrieval strategy (semantic + keyword search) combined with a structured LLM output prompt to achieve high accuracy and reliability.

---

## 🏗 2. System Architecture

This end-to-end architecture ensures clean data ingestion, accurate retrieval, and structured LLM reasoning.

### 2.1 Data Ingestion & Preprocessing

**Input**
A PDF service manual such as `sample-service-manual 1.pdf`.

**Processing Steps**

1. **Loading**
   Using `PyPDFLoader` to extract text from all pages.

2. **Cleaning**
   Custom Regex pipeline removes:

   * headers / footers
   * repeating watermarks
   * page numbers
   * excessive whitespace

3. **Chunking (Updated Configuration)**
   The cleaned text is segmented using a window-based splitter:

   * **Chunk size:** `1200` characters
   * **Overlap:** `200` characters

   **Reasoning**

   * Larger chunks preserve rich technical context.
   * Overlap ensures continuity of specification values that span multiple paragraphs.

**Output**
A list of well-structured text chunks optimized for downstream embeddings.

---

### 2.2 Knowledge Base Construction

The system builds two parallel indexes—semantic and sparse—for robust retrieval.

1. **Semantic Embeddings**

   * Model: `sentence-transformers/all-MiniLM-L6-v2`
   * 384-dimensional vector representation for each chunk
   * Captures meaning, relationships, and engineering semantics

2. **Vector Store**

   * Engine: **FAISS FlatL2**
   * Performs fast similarity search to find semantically relevant chunks

3. **Sparse Keyword Index**

   * Engine: **BM25**
   * Essential for exact string matching:

     * torque values
     * abbreviations
     * part numbers
     * pressure units

4. **Hybrid Ensemble Retrieval**
   Scores are combined:

```text
final_score = 0.5 * FAISS_score + 0.5 * BM25_score
```

This ensures semantic context is preserved and exact match queries are always captured.

---

### 2.3 Retrieval Layer

When a user asks a question, the system:

* Runs semantic search on FAISS
* Runs keyword search on BM25
* Merges and ranks both result sets
* Returns the top-K most relevant chunks

This hybrid retrieval solves common issues such as:

* torque numbers missed by semantic search
* semantically relevant paragraphs missed by keyword search

---

### 2.4 Generation Layer

**LLM**

* Google Gemini 2.5 Flash

**Prompt Engineering**
A strict system prompt enforces:

* `"Do NOT use outside knowledge"`
* `"Respond ONLY using provided chunks"`
* `"Output valid JSON format"`

**Example Output**

```json
{
  "component": "Front Strut Upper Bolt",
  "value": "60",
  "unit": "Nm",
  "source_page": 47
}
```

This avoids hallucination and guarantees usable structured output.

---

### 2.5 Postprocessing & User Interface

**JSON Extraction**
Validates:

* numerical correctness
* unit consistency
* presence of required fields

**Gradio Frontend**
Displays:

* extracted component specifications
* JSON output
* source evidence
* highlighted text from retrieved chunks

This makes the system practical for both technicians and automation pipelines.

---

### 2.6 Self-Evaluation Framework

The system automatically:

1. Scans the PDF for spec-like patterns
2. Generates synthetic Q/A test cases
3. Queries its own RAG pipeline
4. Compares predicted vs expected values
5. Produces accuracy metrics

This allows continuous improvement and scaling to new manuals.

---

## 🧰 3. Tech Stack

* **Languages:** Python 3.10+
* **Embeddings / NLP:** SentenceTransformers (`all-MiniLM-L6-v2`), LangChain text splitters
* **LLM:** Google Gemini 2.5 Flash (via API)
* **Vector DB:** FAISS (FlatL2)
* **Keyword Search:** BM25
* **PDF Parsing:** PyPDFLoader + custom Regex cleaning
* **Table Parsing / Optional:** PDFPlumber, Camelot, Unstructured.io (future)
* **UI:** Gradio
* **Evaluation:** Python scripts (regex-based ground truth & metrics)

---

## 🚀 4. Project Capabilities

* Extract torque values and other numeric specifications
* Extract dimensions or mechanical parameters
* Retrieve relevant engineering context for extracted specs
* Output clean, validated JSON with `component`, `value`, `unit`, `source_page`
* Provide page-level traceability and highlighted evidence
* Auto-evaluate extraction accuracy using generated test cases

Works best on digital PDFs and semi-structured technical content.

---

## 🔧 5. Areas for Improvement (Future Roadmap)

Below is a structured roadmap of enhancements that can make this pipeline production-grade and significantly more powerful.

### 5.1 OCR Integration for Scanned Manuals

**Limitation**
`PyPDFLoader` cannot read text from scanned PDFs, diagrams containing embedded text, or images of tables.

**Improvement**
Integrate OCR: Tesseract, AWS Textract, or Google Document AI.

**Benefit**
Unlocks image-based specification tables, engineering diagrams, and older manuals.

---

### 5.2 Cross-Encoder Reranking

**Limitation**
Hybrid retrieval often finds relevant chunks, but not always at the top positions.

**Improvement**
Use a Cross-Encoder Reranker (e.g., `ms-marco-MiniLM-L-6-v2`) to re-score the top N retrieved chunks.

**Benefit**
Dramatically improves precision, reduces hallucinations, and provides cleaner context for the LLM.

---

### 5.3 Table-Aware Parsing

**Limitation**
Tables are flattened during text extraction → relationships between headers and cells are lost.

**Improvement**
Use table parsers (Unstructured.io, PDFPlumber, Camelot) to convert tables into JSON/Markdown/row-column structures before embedding.

**Benefit**
Perfect extraction of torque charts, fluid capacities, and dimensional tables.

---

### 5.4 Agentic RAG (LangGraph Integration)

**Limitation**
Current workflow is strictly linear: `Retrieve → Answer`. It cannot perform multi-step reasoning, conversions, or self-correction.

**Improvement**
Add LangGraph agents capable of:

* rewriting failed queries
* performing conversion calculations (e.g., Nm → ft-lb)
* comparing specifications across trims
* multi-hop retrieval

**Benefit**
Greatly enhances reasoning, autonomy, and robustness.

---

### 5.5 Metadata Indexing

**Improvement**
Auto-extract metadata such as:

* section titles
* subsystem tags (`Brakes`, `Suspension`, etc.)
* unit types
* page anchors

**Benefit**
Improves filtering, targeted retrieval, and structured querying.

---

### 5.6 Smarter Chunking Strategies

Although the current configuration (1200 chars + 200 overlap) works well, future improvements include:

* semantic chunking
* layout-aware chunking (respecting columns, captions, table boundaries)
* section-level chunking

**Benefit**
Higher coherence and less noise sent to the LLM.

---

## 🏁 6. Conclusion

This project establishes a strong and scalable RAG system for extracting vehicle specifications from complex automotive service manuals. It uses a hybrid retrieval strategy, structured LLM prompting, and automated evaluation to deliver accurate and explainable outputs.

The enhancement roadmap—including OCR, reranking, table parsing, and agentic workflows—opens the door for a production-grade automotive knowledge engine capable of operating across manufacturers, vehicle models, and documentation formats.

---

## 📎 Appendix

### Example JSON schema (recommended)

```json
{
  "component": "string",
  "value": "string | number",
  "unit": "string",
  "source_page": "integer",
  "confidence": "float (optional)",
  "source_text": "string (optional)"
}
```


