# TawasolPay ThreatLens: AI-Powered Risk Intelligence

**TawasolPay ThreatLens** is a live, multi-dimensional cyber risk analysis tool built with Streamlit. It leverages a Retrieval-Augmented Generation (RAG) architecture to map scored vulnerabilities to specific NIST SP 800-53 Rev. 5 controls, providing explainable and actionable remediation guidance via Groq-powered LLMs (Llama 3.3 70B).

## Features
* **Multi-Dimensional Risk Scoring:** Automatically ingests and scores vulnerabilities across threat, exposure, business criticality, severity, and hygiene metrics.
* **NIST 800-53 RAG Integration:** Uses ChromaDB and BAAI/bge-small-en-v1.5 embeddings to semantically retrieve the most relevant security controls.
* **AI-Powered Briefings:** Generates executive-level explanations and recommended actions using Groq's high-speed inference.
* **Interactive Dashboard:** A highly stylized, high-contrast Streamlit UI for analyzing top organizational risks in real-time.

## Prerequisites

* **Python 3.10 or 3.11** (Recommended for compatibility with ChromaDB and PyTorch)
* **Git**
* A valid **Groq API Key**.

## Installation

**1. Clone the repository**
```bash
git clone https://github.com/jk-7-dev/tawasolpay-risk-assistant.git
cd tawasolpay-risk-assistant
```

**2. Create a Virtual Environment**
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

**3. Install Dependencies**
```bash
pip install -r requirements.txt
```

## Configuration

Create a `.env` file in the root directory and add your Groq API key:
```env
GROQ_API_KEY=your_actual_api_key_here
```

## Architecture & Design Decisions

TawasolPay ThreatLens is built on a deliberate, multi-stage architecture designed to prioritize deterministic risk ranking and grounded AI explanations.

### Stage 1: Data Ingestion & Joins (`data_pipeline.py`)
* **Vulnerabilities as the "Spine":** The join order starts with vulnerabilities because a risk is fundamentally (vulnerability × asset). Assets with no vulnerabilities are excluded early.
* **LEFT Joins for Noise Filtering:** We use LEFT joins sequentially (Assets → Services → Threats → KEV). This preserves vulnerabilities without matched threat campaigns while automatically dropping orphaned threat-intel noise.
* **Separating CISA KEV and Internal Intel:** Internal ransomware associations and CISA KEV flags are kept separate to prevent conflating authoritative government attestations with internal security team analysis.

### Stage 2: Multi-Dimensional Risk Scoring (`risk_engine.py`)
CVSS measures abstract technical severity, which systematically over-rates isolated dev infrastructure and under-rates exposed, targeted assets. ThreatLens uses a deterministic, five-dimension model:
* **Threat (30%):** Active ransomware, KEV status, named actors. Weighted highest because active exploitation is the strongest signal of near-term risk.
* **Exposure (25%):** Internet-facing status, environment (prod vs. dev). Exposure is the precondition for exploitation.
* **Business Criticality (25%):** Asset criticality, data classification, revenue impact. Consequence matters as much as likelihood.
* **Technical Severity (10%):** CVSS + severity rating. Deliberately de-weighted to serve as a baseline rather than the dominant factor.
* **Hygiene (10%):** EDR coverage, patch availability, days-open. Captures the presence or absence of compensating controls.
* **Log-Linear Scaling (1/4/7/10):** Categorical variables are mapped to a 1/4/7/10 scale to reflect that real-world risk is log-linear (Critical is exponentially worse than Low, not just linearly worse).
* **Deterministic Execution:** Scoring is purely mathematical. The LLM is excluded from this step to guarantee reproducible rankings across runs.

### Stage 3: RAG over NIST 800-53 (`rag_agent.py`)
* **Why RAG?** Hardcoding remediation guidance fails to handle unanticipated controls. Relying on LLM memory causes hallucinations. RAG ensures all guidance is backed by the authoritative NIST catalog.
* **Embedding Choice (BGE-small):** `BAAI/bge-small-en-v1.5` offers the best retrieval quality per CPU-second for technical policy text and supports asymmetric query/passage encoding.
* **Vector Store (ChromaDB):** An embedded, serverless persistent store utilizing cosine distance, perfect for sub-50ms queries over a ~1k document corpus.
* **Chunking Strategy:** Each chunk represents one full NIST control (Control + Discussion). Splitting them smaller breaks the semantic link between a policy mandate and its rationale.

### Stage 4: Grounded LLM Explanation (`explainer.py`)
* **The LLM as a Translator:** The LLM does not score, rank, or invent. It translates a fully-evidenced risk record and a set of candidate NIST controls into a structured English briefing.
* **Multi-Angle Query Expansion:** Instead of a single query, the system generates up to 3 queries conditioned on specific gaps (e.g., querying "endpoint monitoring" if EDR is missing, or "boundary protection" if internet-exposed).
* **Hallucination Guardrails:** The LLM selects the most actionable control from the retrieved candidates. A strict validation step ensures the LLM's chosen control ID exists in the candidate set; if it hallucinates, the system falls back to the top retrieval.
* **Structured Output (JSON Mode):** Using Groq (Llama 3.3 70B) at `temperature=0.2` with JSON mode guarantees structural reliability. The output enforces actionable fields, specifically including a `success_metric` to define when the risk is actually mitigated.

## Project Structure

```text
tawasolpay-risk-assistant/
├── src/
│   ├── app.py                            # Streamlit UI entry point
│   ├── data_pipeline.py                  # Data ingestion & left-joins
│   ├── risk_engine.py                    # Deterministic 5-dimension scoring
│   ├── rag_agent.py                      # BGE embeddings & ChromaDB index
│   ├── explainer.py                      # Multi-query RAG & LLM grounding
│   ├── test_rag_real.py                  # Retrieval smoke test
│   └── test_llm_real.py                  # LLM grounding & structure test
├── data/
│   ├── raw/                              # Internal CSVs + generated artifacts
│   ├── external/                         # External catalogs (CISA KEV, NIST)
│   └── chroma_nist/                      # Persistent ChromaDB vector index
├── requirements.txt                      # Python dependencies
├── .env                                  # Environment variables (gitignored)
├── .gitignore
└── README.md
```

## Testing

**Retrieval Test:**
Validate that the NIST retriever surfaces controls for realistic risk descriptions:
```bash
cd src
python test_rag_real.py
```

**LLM Grounding Test:**
Validate that the LLM explanation is properly grounded in retrieved controls and structures correctly:
```bash
cd src
python test_llm_real.py
```

## Troubleshooting

* **ChromaDB / SQLite Errors:** Use the "Clear caches & rebuild" button in the Streamlit sidebar to force a fresh index build.
* **Missing API Key:** Ensure your `.env` file is in the root directory and properly formatted.
* **Network Timeout on CISA KEV Fetch:** The system gracefully falls back to an empty DataFrame or a locally cached copy if the download fails.

## Customization

* **Risk Scoring Weights:** Edit the `DIMENSION_WEIGHTS` dictionary in `src/risk_engine.py` to adjust the relative importance of each dimension.
* **Data Classification Scores:** Modify the `DATA_CLASS_MAP` in `risk_engine.py` to reflect your organization's specific data hierarchy.
* **LLM Hyperparameters:** Tune the model ID, retrieval top-k, and temperature settings within `src/explainer.py`.
