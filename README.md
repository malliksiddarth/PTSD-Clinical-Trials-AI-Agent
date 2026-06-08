PTSD Clinical Trials Intelligence System

A natural language query system over real clinical trial data combining a SQL engine and a RAG pipeline under a single LLM orchestrator. 
Ask it a plain English question. It figures out whether to query a database, search documents, or both — then gives you one coherent answer.


What it does?
"How many PTSD trials are currently recruiting?"                            → SQL
"What are the eligibility criteria for veterans?"                           → RAG
"Which Phase 3 trials are recruiting and what do treatments do they use?"   → BOTH

The orchestrator routes each question, calls the right tool, and synthesizes the result. 
The SQL tool and RAG tool never talk to each other only the orchestrator sees both outputs

Architecture
text
┌─────────────────────┐
│ ClinicalTrials.gov  │
│        API          │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Bronze Layer        │
│ ptsd_trials_raw     │
│ Raw API Response    │
└──────────┬──────────┘
           │ Clean + Chunk
           ▼
┌─────────────────────┐
│ Silver Layer        │
│ ptsd_trials_silver  │
│ Cleaned Chunks      │
└──────────┬──────────┘
           │ MiniLM Embeddings
           ▼
┌─────────────────────┐
│ Gold Layer          │
│ ChromaDB            │
│ Vector Store        │
└──────────┬──────────┘
           │
           │
┌──────────▼──────────┐
│ Structured Data     │
│ Spark SQL Table     │
│ Trial Metadata      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────┐
│          Orchestrator           │
│                                 │
│ Route Question                  │
│ ├── SQL Tool                    │
│ └── RAG Tool                    │
│                                 │
│ Collect Results                 │
│        ↓                        │
│      Groq LLM                   │
│        ↓                        │
│    Final Answer                 │
└─────────────────────────────────┘

Dataset:
- Source: ClinicalTrials.gov public API
- Domain: PTSD (Post-Traumatic Stress Disorder)
- Size: 2,598 real clinical trials, 25,371 text vectors
- Date range: Trials from 2000 to present
- Structured fields: status, phase, sponsor, enrollment, dates, interventions
- Unstructured fields: brief summaries, detailed descriptions, eligibility criteria

This is real data, not synthetic. Every trial in the dataset is a live or historical study registered with the US National Library of Medicine.

Data pipeline:
Ingestion: 
Fetches all PTSD trials from the ClinicalTrials.gov v2 API with pagination. Each API response is a deeply nested JSON object with protocol, status, design, eligibility, 
and description modules.

Parsing:
Splits each study into two streams:
- Structured (flat fields) → goes into a Spark SQL table for numerical queries
- Unstructured (long text) → goes into the RAG pipeline for semantic search

ETL (Medallion architecture):
- Bronze: raw text stored in Delta
- Silver: cleaned text (HTML stripped, whitespace normalised) chunked into 500-character pieces with 50-character overlap
Gold: each chunk embedded into a 384-dimensional vector and stored in ChromaDB

Why this separation matters ?
SQL can answer "how many" and "which sponsor" in milliseconds. It cannot answer "what do the eligibility criteria say about veterans." 
ChromaDB can find the most semantically relevant chunks from 7,000 documents in milliseconds. It cannot count or aggregate. 
Keeping them separate means each tool does exactly what it is good at,and the orchestrator combines the results.

How the orchestrator works ?
1. Route: sends the question to Groq with a classification prompt,  returns SQL, RAG, or BOTH
2. Execute: calls the relevant tool:
   - SQL tool: asks Groq to write a Spark SQL query, runs it, retries up to 3 times if it fails
   - RAG tool: embeds the question, queries ChromaDB for top matching chunks, filters by similarity score
3. Synthesise: passes all results to Groq with the original question, gets a single coherent answer

The orchestrator never touches the database directly. The SQL tool never reads documents. The RAG tool never touches the database. Each component has exactly one job.

Setup:
Requirements:
- Databricks free edition account
- Groq API key (free at console.groq.com)

Run order:
Copy the notebook into Databricks and run all cells top to bottom. Cells 1–5 only need to run once (ingestion + embedding takes ~10 minutes on first run). 
After that, start from Cell 6 for daily use.

Cell 1 — install packages
Cell 2 — fetch PTSD trials from API
Cell 3 — parse structured + unstructured fields
Cell 4 — ETL pipeline: clean, chunk, save to Delta (Bronze → Silver)
Cell 5 — embed chunks, store in ChromaDB (Gold layer)
Cell 6 — load tools (SQL + RAG + router)
Cell 7 — orchestrator function
Cell 8 — interactive Q&A loop

Sample questions to try:
How many PTSD trials are currently recruiting?
What is the total number of patients enrolled across all trials?
Which sponsor has run the most PTSD trials?
What Phase 3 trials are still active?

What are the eligibility criteria for veterans in PTSD trials?
What MDMA-assisted therapy trials exist for PTSD?
Explain what cognitive processing therapy trials involve.

Which Phase 3 trials are recruiting and what treatments do they test?
List completed trials from the last 5 years and summarise their focus.

Why PTSD ?
PTSD affects roughly 12 million adults in the US annually. It is chronically underfunded relative to its prevalence and disproportionately affects veterans, first responders, 
and trauma survivors. ClinicalTrials.gov holds 2,500+ registered studies on the condition — rich, real data that has had almost no AI tooling built on top of it.
This system makes that data accessible to anyone who can ask a question in plain English.

Limitations:
- ChromaDB is stored in `/tmp/` on Databricks Serverless and is rebuilt each session from the Silver Delta table. On a persistent cluster this would not be necessary.
- The Groq free tier has rate limits. For high-frequency use, a paid tier or self-hosted model would be appropriate.
- The SQL tool relies on the LLM generating correct Spark SQL. The retry logic handles most failures but complex multi-join queries may require prompt tuning.
