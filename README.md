# DocuInsight

**Intelligent answers grounded in your documents.**

DocuInsight is an open-ended document QA application powered by **GOV-RAG**, a unified retrieval pipeline that augments semantic retrieval with authority, scope, status and conflict-aware evidence reasoning.

## Live demo

https://govrag-api-347724837198.us-central1.run.app/app

The demo supports PDF, DOCX, TXT and Markdown uploads.

## How to use the app

### Test with your own documents

1. Open the live demo.
2. Click **New Collection** if a previous collection is open.
3. Create a collection name.
4. Drag and drop documents or click **Browse files**.
5. Wait until the uploaded filenames appear.
6. Enter any factual question in **Ask a question**.
7. Click **Resolve**.
8. DocuInsight returns the answer, source, confidence and supporting evidence. If relevant sources disagree, GOV-RAG also shows conflict information and competing evidence.

For ordinary documents with no meaningful disagreement, DocuInsight behaves like normal grounded document QA.

Demo collections currently expire after 24 hours.

## Reproduce the ConflictBench Easy experiment

1. Open the live demo.
2. Create a fresh collection.
3. Click **Load Example Dataset**.
4. Wait until the UI reports that **180 example documents** were loaded.
5. Run the questions below.
6. Compare the returned answer/source with the expected answer/source.

The Easy pack contains 15 question groups with 12 documents per question:

- 1 authoritative source;
- 8 conflicting sources;
- 3 noise sources.

Gold answers are used only for evaluation and are not supplied to GOV-RAG during inference.

## Easy benchmark questions

1. **Q001** — What is the annual reimbursement ceiling for Tier-B employees at Velora Dynamics?
   - Expected answer: `7,430 Quens`
   - Gold source: `Q001_source_10.md`
2. **Q002** — What is the maximum monthly API quota for Enterprise-X tenants at Nexora Systems?
   - Expected answer: `184,000 calls`
   - Gold source: `Q002_source_01.md`
3. **Q003** — How long must Tier-3 diagnostic records be retained at Arclume Health?
   - Expected answer: `11 years`
   - Gold source: `Q003_source_07.md`
4. **Q004** — What is the response-time SLA for Platinum-2 factory incidents at Solvane Robotics?
   - Expected answer: `42 minutes`
   - Gold source: `Q004_source_08.md`
5. **Q005** — What is the collateral threshold for Class-M counterparties at Brinex Capital?
   - Expected answer: `3.8 million Luma`
   - Gold source: `Q005_source_08.md`
6. **Q006** — What is the maximum idle dwell time for Route-Z cargo at Caventra Logistics?
   - Expected answer: `26 hours`
   - Gold source: `Q006_source_11.md`
7. **Q007** — At what temperature must Kappa-7 samples be stored at Orlith Bio?
   - Expected answer: `-47°C`
   - Gold source: `Q007_source_05.md`
8. **Q008** — What reserve margin is required for Sector-R microgrids at Ternova Energy?
   - Expected answer: `13.6%`
   - Gold source: `Q008_source_07.md`
9. **Q009** — What is the maximum defect rate allowed for Q4-certified suppliers at Meridian Forge?
   - Expected answer: `0.72%`
   - Gold source: `Q009_source_12.md`
10. **Q010** — What minimum audit sample size is required for Vela-class models at Asteron Labs?
   - Expected answer: `1,480 cases`
   - Gold source: `Q010_source_08.md`
11. **Q011** — After how many minutes must a P1-Silver network fault be escalated at Quoralis Telecom?
   - Expected answer: `17 minutes`
   - Gold source: `Q011_source_12.md`
12. **Q012** — How many inspection points are required for Delta-9 batches at Heliox Materials?
   - Expected answer: `23 inspection points`
   - Gold source: `Q012_source_03.md`
13. **Q013** — What is the maximum unmanned operating radius for Echo-4 vehicles at Sereva Mobility?
   - Expected answer: `38 kilometers`
   - Gold source: `Q013_source_12.md`
14. **Q014** — What is the maximum transit temperature for Cobalt-line products at Noventra Foods?
   - Expected answer: `3.4°C`
   - Gold source: `Q014_source_12.md`
15. **Q015** — How often must Omega-tier credentials be rotated at Elystrum Security?
   - Expected answer: `every 19 days`
   - Gold source: `Q015_source_11.md`

## Experimental results

| Method | Easy Accuracy |
| --- | ---: |
| Corvic baseline | 6.7% (1/15) |
| + RAG/data pipeline | 33.3% (5/15) |
| + retrieval-focused prompting | 46.2% (6/13)* |
| **GOV-RAG** | **100% (15/15)** |

\* The retrieval-focused Corvic condition was a partial diagnostic run; usage ended before all 15 questions were completed.

### Controlled Corvic setup

- Model: Gemini 2.5 Pro
- Web Search: Off
- Fresh chat/thread per question
- Same Easy document pack
- Gold answers used only for evaluation

The two prompts used for the Corvic conditions are in:

```text
prompts/corvic_baseline_prompt.txt
prompts/retrieval_focused_prompt.txt
```

## GOV-RAG in one view

```text
Question
   |
   v
Chunk-level semantic retrieval
   |
   v
Relevant evidence
   |
   v
Scope + authority + status + claim analysis
   |
   v
Conflict detection / diversification
   |
   v
Unified GOV-RAG reranking
   |
   v
Gemini 2.5 Pro
   |
   v
Answer + source + evidence
```

GOV-RAG does not require conflicting documents. When documents agree or governance metadata is absent, governance features remain neutral and the system behaves like grounded semantic RAG. When documents conflict, authority, scope, status/currentness and conflict diversification help determine which evidence should govern.

See [`ARCHITECTURE.md`](ARCHITECTURE.md) for implementation details.

## Suggested repository layout

```text
DocuInsight/
├── README.md
├── ARCHITECTURE.md
├── prompts/
│   ├── corvic_baseline_prompt.txt
│   └── retrieval_focused_prompt.txt
├── benchmark/
│   └── easy_questions.csv
├── govrag/
│   ├── gov_rag.py
│   ├── gov_rag_gemini.py
│   └── chunker.py
└── api/
    └── main.py
```

## Notes

- ConflictBench is an evaluation/example corpus, not a production dependency.
- Normal user-uploaded documents follow the same GOV-RAG pipeline.
- No benchmark gold labels are consulted during retrieval or answer generation.
- The current implementation uses Vertex AI `text-embedding-004` for chunk embeddings and Gemini 2.5 Pro for final evidence reasoning.
