# DocuInsight / GOV-RAG Architecture

## Overview

**DocuInsight** is the user-facing document intelligence application.

**GOV-RAG** is the retrieval and reasoning pipeline behind it.

The system is designed to work for both ordinary document question answering and conflicting-document question answering. GOV-RAG is a single unified pipeline. Conflict resolution is an additional capability, not a separate RAG system.

## End-to-end flow

```text
User
  |
  v
DocuInsight Web App
  |
  v
FastAPI on Google Cloud Run
  |
  +--> Google Cloud Storage
  |
  v
Document parsing + normalization
  |
  v
Chunking
  |  ~900-word target
  |  ~150-word overlap
  v
Vertex AI text-embedding-004
  |
  v
Chunk-level semantic retrieval
  |
  v
GOV-RAG evidence analysis
  |
  +--> semantic relevance
  +--> scope match
  +--> document status
  +--> authority
  +--> document type
  +--> claim extraction when useful
  |
  v
Conflict grouping / diversification
  |
  v
Unified GOV-RAG reranking
  |
  v
Gemini 2.5 Pro
  |
  v
Answer + source + explanation + conflict metadata + evidence
```

## Corpus storage

Each document collection receives a unique `corpus_id`.

Uploaded documents are persisted in Google Cloud Storage so they survive Cloud Run cold starts and instance replacement. Cloud Run may materialize a corpus into temporary local storage while rebuilding a GOV-RAG instance, but GCS remains the persistent source of truth.

Supported formats:

- PDF
- DOCX
- TXT
- Markdown

The demo supports up to 500 files per collection, subject to configured corpus-size limits.

## Parsing and chunking

Uploaded documents are normalized into text.

Long documents are split into overlapping chunks before retrieval:

- approximately 900 words per target chunk;
- approximately 150 words of overlap;
- paragraph and sentence boundaries preferred where practical;
- short documents remain a single chunk.

Each chunk retains its original source filename and document metadata.

## Embeddings and retrieval

DocuInsight uses Vertex AI `text-embedding-004`.

Each document chunk and user question are embedded using the same model. Retrieval compares the question vector with chunk vectors and returns a broad set of relevant chunks.

No external vector database is required in the current implementation. Embeddings and retrieval are handled inside the corpus-specific GOV-RAG instance.

## Governance metadata

For each source, GOV-RAG estimates governance-related metadata when explicit evidence exists, such as:

- active / authoritative;
- active;
- draft;
- obsolete / superseded;
- document type;
- scope match;
- authority score.

For ordinary documents without governance signals, these values remain neutral rather than preventing normal QA.

## Claim and conflict analysis

Retrieved evidence is inspected for question-relevant claims.

Example:

```text
official_policy.md -> 0.72%
draft_policy.md    -> 0.90%
audit_note.md      -> 1.00%
```

Multiple sources do not automatically imply conflict. Conflict is present only when question-relevant evidence contains meaningfully incompatible answers.

When sources agree, GOV-RAG behaves like grounded semantic RAG. When sources disagree, governance signals become more important for deciding which evidence should govern.

## Unified reranking

GOV-RAG uses one ranking pipeline for all queries.

The implementation combines semantic relevance, authority and scope signals. When governance metadata is absent, authority remains neutral and semantic relevance/scope dominate.

When conflicting sources contain strong governance metadata, active authoritative evidence can outrank repeated but weaker sources.

A central principle is:

> Frequency is not authority.

Conflict grouping/diversification reduces candidate crowding from repeated versions of the same competing claim.

## Gemini 2.5 Pro reasoning

After retrieval and reranking, the strongest evidence passages are sent to Gemini 2.5 Pro.

For ordinary QA, Gemini answers from the strongest supporting evidence.

For conflicts, it reasons over the supplied source text using exact scope, authority, currentness, supersession, and applicability.

The response includes fields such as:

```json
{
  "answer": "0.72%",
  "selected_source": "Q009_source_12.md",
  "conflict_detected": true,
  "confidence": "high",
  "reason": "...",
  "num_conflicts": 3,
  "top_sources": []
}
```

## Normal-document behavior

GOV-RAG does not require a conflict.

```text
one research paper
    |
    v
retrieve relevant chunks
    |
    v
governance metadata is neutral
    |
    v
semantic relevance + scope dominate
    |
    v
answer from supporting passage
```

In this case `conflict_detected = false`.

## Conflict behavior

If relevant evidence disagrees, GOV-RAG:

1. retrieves relevant evidence;
2. detects incompatible claims;
3. evaluates authority/status/scope;
4. diversifies competing claims;
5. reranks the evidence;
6. asks Gemini to resolve the conflict from the supplied evidence.

## ConflictBench role

ConflictBench is an evaluation dataset, not a hard-coded part of GOV-RAG inference.

The Easy set contains:

- 15 questions;
- 12 documents per question;
- 180 documents total;
- 1 authoritative source per question;
- 8 conflicting sources;
- 3 noise sources.

The app includes a **Load Example Dataset** button that loads these 180 documents through the same document pipeline used for arbitrary uploads.

No gold answers or question IDs are used by production inference.

## Deployment architecture

```text
Browser
  |
  v
Cloud Run: DocuInsight + FastAPI
  |
  +--> /demo/corpora/*
  |
  +--> GCS persistent corpus storage
  |
  +--> Vertex AI text-embedding-004
  |
  +--> Gemini 2.5 Pro
```

The frontend and API are served from the same Cloud Run service.
