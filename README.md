# 🚗 Automotive Specification Extraction RAG Pipeline

## 📌 Project Overview
This project implements a **Retrieval-Augmented Generation (RAG)** system designed to automatically extract technical vehicle specifications (e.g., torque values, dimensions, fluid capacities) from unstructured PDF service manuals.

Unlike standard search tools, this system understands technical context and returns structured **JSON data**, making it ready for database integration. It includes a robust **self-evaluation framework** that automatically generates test cases from the source text to benchmark performance.

## 🏗️ System Architecture
The pipeline is built using **LangChain** and follows a "Hybrid Search" architecture for maximum accuracy:

1.  **Ingestion & Cleaning**: 
    * Loads PDF using `PyPDFLoader`.
    * **Custom Preprocessing**: Regex cleaning removes page headers/footers to prevent context fragmentation.
2.  **Chunking**: 
    * Uses `RecursiveCharacterTextSplitter` (1200 chars, 200 overlap) to keep table rows and headers together.
3.  **Hybrid Retrieval (The "Secret Sauce")**:
    * **Semantic Search**: Uses **FAISS** vector store with `all-MiniLM-L6-v2` embeddings to understand intent.
    * **Keyword Search**: Uses **BM25** to catch exact part names (e.g., "W707628").
    * **Ensemble**: Combines results (50/50 weight) to feed the LLM the best context.
4.  **Extraction Intelligence**:
    * **Model**: Google **Gemini 1.5 Flash** (via API) for high-speed, cost-effective inference.
    * **Prompt Engineering**: Enforces strict JSON output format (`component`, `spec_type`, `value`, `unit`).
5.  **User Interface**:
    * **Gradio Web UI**: Provides a user-friendly way to query the manual and verify sources.

## 🛠️ Tech Stack
* **Framework**: LangChain, LangChain-Google-GenAI
* **LLM**: Google Gemini 1.5 Flash
* **Vector Store**: FAISS (Facebook AI Similarity Search)
* **Embeddings**: Hugging Face (`sentence-transformers`)
* **UI**: Gradio
* **Data Processing**: Pandas, PyPDF, Tqdm

## 📊 Automated Evaluation Pipeline
The system goes beyond simple queries by implementing an **Automated Ground Truth Generation** engine:

1.  **Auto-Discovery**: Scans the raw PDF text for regex patterns (e.g., `"Tighten to X Nm"`, `"Capacity X.X L"`) to identify ~100+ potential test specifications.
2.  **Test Set Creation**: Automatically compiles a `ground_truth.csv` file containing the Question, Expected Value, and Unit.
3.  **Batch Testing**: Runs the RAG model against a random sample (or full set) of these questions.
4.  **Scoring Logic**:
    * **Exact Match**: Predicted value equals expected value.
    * **Fuzzy/Contains Match**: Handles variations like "35" vs "35-40".
5.  **Reporting**: Generates a detailed CSV report (`final_evaluation_results.csv`) with Status (PASS/FAIL), Predicted vs Actual, and Source Page verification.

## 🚀 How to Run
1.  **Prerequisites**:
    ```bash
    pip install langchain langchain-google-genai faiss-cpu gradio sentence-transformers rank_bm25 tqdm
    ```
2.  **Environment Setup**:
    * Get a free API key from [Google AI Studio](https://aistudio.google.com/).
    * Set `GOOGLE_API_KEY` in the notebook when prompted.
3.  **Execution**:
    * Upload `sample-service-manual 1.pdf` to the root directory.
    * Run all cells in `assignment_predii.ipynb`.
    * The **Gradio UI** will launch at the bottom for interactive testing.
    * The **Evaluation Report** will be saved as `final_evaluation_results.csv`.

## 📂 File Structure
* `assignment_predii.ipynb`: Main notebook containing the pipeline, UI, and evaluation logic.
* `vehicle_specs_final.json`: Extracted output from manual queries.
* `ground_truth.csv`: Automatically generated test questions from the manual.
* `final_evaluation_results.csv`: Performance report of the model against the ground truth.
* `README.md`: This documentation file.
