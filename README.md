# Medical Guidelines RAG Chatbot

A Retrieval-Augmented Generation (RAG) chatbot built over three World Health Organization (WHO) clinical guideline PDFs. This project explores how document chunking, retrieval depth, clinical query expansion, and prompt design affect the quality and traceability of generated answers.

> **Disclaimer:** This is an educational project for learning and experimentation. It is not a clinically validated medical system and must not be used to diagnose, treat, or make patient-care decisions. Consult the original WHO publications and qualified healthcare professionals.

## Notebook

- **Notebook:** [WHO Guidelines RAG Chatbot — Groq + Hugging Face](./WHO_Guidelines_RAG_Chatbot_Groq_HF.ipynb)
- **Open in Google Colab:** [Launch notebook](https://colab.research.google.com/drive/1ddNzy2ZydDJKKHXpsoepCxX5Oeu1UicF?usp=sharing)

## Knowledge Sources

The notebook ingests three WHO guideline PDFs:

1. `9789241548397_eng.pdf` — *Prevention and Control of Noncommunicable Diseases: Guidelines for Primary Health Care in Low-Resource Settings* (2012), covering Type 2 Diabetes, Asthma, and COPD.
2. `WHO-UCN-NCD-20.1-eng.pdf` — *HEARTS-D: Diagnosis and Management of Type 2 Diabetes* (2020).
3. `ist pdf.pdf` — *Guideline for the Pharmacological Treatment of Hypertension in Adults* (2021).

The source PDFs are not committed to this repository. Upload the files to the Colab runtime before running the notebook.

## Workflow

```text
WHO guideline PDFs
       |
       v
Page-wise PDF text extraction (PyPDF)
       |
       v
Text + source filename + page metadata
       |
       v
Chunking strategies
       |
       v
PubMedBERT embeddings (Sentence Transformers)
       |
       v
ChromaDB vector store
       |
       v
Question / clinical query expansion
       |
       v
Top-k context retrieval
       |
       v
Groq LLM answer generation
       |
       v
Answer with source-document and page references
       |
       v
LLM-as-a-Judge evaluation
```

## Technology Stack

| Component | Tool | Purpose |
|---|---|---|
| Notebook environment | Google Colab | Development and execution |
| PDF extraction | PyPDF | Read guideline text page by page |
| Embeddings | `NeuML/pubmedbert-base-embeddings` | Generate domain-specific medical text embeddings locally |
| Vector store | ChromaDB | Store and retrieve embedded chunks |
| Text splitting | LangChain Text Splitters | Compare fixed and recursive chunking |
| Answer generation | Groq API | Generate responses using an OpenAI-compatible client |
| Evaluation | Groq + JSON outputs | Score generated answers against a small reference set |
| Analysis | Pandas, Matplotlib | Aggregate and visualize experiment scores |

## Implementation

### 1. Document ingestion

- Loads the three PDF files page by page.
- Extracts text using PyPDF.
- Preserves the source filename and 1-based page number as metadata.
- Skips pages that contain too little extractable text.

### 2. Baseline RAG pipeline

The baseline provides a reference configuration:

- Fixed-character chunking with `chunk_size=500`.
- `chunk_overlap=0`.
- Local Hugging Face PubMedBERT embeddings.
- ChromaDB in-memory collection.
- Similarity retrieval with `top_k=2`.
- Groq-generated answers based on retrieved context.

### 3. Evaluation setup

The notebook defines a six-question benchmark set with reference answers drawn from the guideline documents. An LLM-as-a-Judge evaluates each response on a 1–5 scale for:

- **Faithfulness:** whether the answer is supported by the retrieved context.
- **Relevance:** whether it addresses the question.
- **Completeness:** whether it includes the important information represented in the reference answer.

The notebook combines the three dimensions into an overall score and compares the configurations. LLM-judge scores are useful for exploratory comparison, but they are not independent clinical validation.

### 4. Retrieval improvement experiments

| Configuration | Main change | Settings |
|---|---|---|
| Baseline | Fixed character chunks | 500 characters, no overlap, `top_k=2` |
| Experiment 1 | Recursive chunking with overlap | 1,000-character chunks, 200 overlap, `top_k=4` |
| Experiment 2 | Clinical query expansion | Rewrite the question with relevant clinical terms before retrieval |
| Experiment 3 | Combine original and expanded query; strengthen clinical answer instructions | `top_k=5`, grounded answers, source/page citations |

**Experiment 1 — Recursive chunking:** uses natural text boundaries and overlap to reduce the chance that related sentences, lists, or table content are separated across chunks.

**Experiment 2 — Query expansion:** uses the LLM to rewrite a question into a concise, keyword-rich clinical search query. The prompt can include relevant abbreviations and synonyms such as FPG, HbA1c, T2DM, COPD, ACEi, ARB, and CCB.

**Experiment 3 — Combined retrieval and clinical prompt:** searches using both the original question and its expanded form. The system prompt instructs the model to use only retrieved context, preserve exact numerical values and protocol steps, prioritize newer guidance where both older and newer guidance are present, and cite the PDF filename and page number.

The notebook builds a summary table and grouped bar chart to compare the evaluation scores. The reported values should be generated by running the notebook; this README does not claim a particular experiment achieved a specific score.

## Run the Notebook

The notebook is designed for Google Colab.

1. Open the notebook from this repository or use the Colab link above.
2. Upload the three WHO PDF files listed under **Knowledge Sources** to the Colab session.
3. Add a Groq API key to Colab Secrets using the key name `Groq_API_Key`. The notebook also supports entering a key when prompted.
4. Run the cells from top to bottom.

The notebook installs these dependencies:

```bash
pip install -q openai chromadb pypdf langchain-text-splitters pandas matplotlib sentence-transformers
```

The embedding model is downloaded from Hugging Face and runs locally in the notebook runtime. Groq is used for answer generation and LLM-based evaluation.

**API key safety:** Do not hard-code API keys or commit them to the repository. Use Colab Secrets or environment variables.

## Current Scope and Limitations

- The benchmark contains six questions, so the evaluation is small and exploratory.
- LLM-as-a-Judge results can vary between runs and should not be interpreted as objective ground truth.
- PDF text extraction may not fully preserve complex tables, diagrams, or scanned pages.
- The multi-turn chat-memory section at the end of the notebook is commented example code; it is not an active, completed feature.
- A heading/subheading-based retrieval approach was considered as a possible extension, but it is not implemented in this notebook.
- The pipeline depends on external Groq model availability, API limits, and the Colab runtime environment.
- This project does not constitute clinical validation, safety testing, or a substitute for professional medical judgment.

## Potential Improvements

- Increase the size and diversity of the benchmark and have reference answers independently reviewed.
- Evaluate retrieval separately using Recall@k, Mean Reciprocal Rank (MRR), or nDCG.
- Compare chunk-based retrieval with heading/subheading-based retrieval.
- Validate that generated source citations point to the passages that support each claim.
- Add automated regression tests for retrieval and answer generation.
- Implement and test multi-turn conversation memory.
- Evaluate tables, scanned PDFs, and varied question formulations.

## Learning Outcomes

- Building a PDF-based RAG pipeline with page-level metadata.
- Generating domain-specific embeddings with a locally run Hugging Face model.
- Storing and retrieving text embeddings with ChromaDB.
- Comparing chunk size, chunk overlap, and retrieval depth.
- Applying clinical query expansion to improve semantic retrieval.
- Designing prompts that encourage grounded answers and source citations.
- Using LLM-as-a-Judge for exploratory evaluation of faithfulness, relevance, and completeness.

## Repository Contents

```text
Medical-Guidelines-RAG-Chatbot/
├── README.md
└── WHO_Guidelines_RAG_Chatbot_Groq_HF.ipynb
```
