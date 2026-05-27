# 🏌️ Enterprise RAG Product Blueprint
### A PM-Led Deep Dive into Building Privacy-First AI at Scale

> **TL;DR** — An enterprise-grade RAG system where embeddings stay local (zero data leakage) and cloud LLMs handle only reasoning. Built to pressure-test the real trade-offs PMs face when shipping AI features without burning budgets or losing enterprise deals.

---

## 📌 Table of Contents

1. [Problem Statement](#problem-statement)
2. [User Persona](#user-persona)
3. [Pain Points](#pain-points)
4. [Product Hypothesis](#product-hypothesis)
5. [Solution Architecture](#solution-architecture)
6. [Prioritization (RICE Framework)](#prioritization)
7. [Technical Stack Decisions](#technical-stack-decisions)
8. [Success Metrics & KPIs](#success-metrics--kpis)
9. [Cost Model](#cost-model)
10. [Trade-offs & Risks](#trade-offs--risks)
11. [What I'd Do Differently](#what-id-do-differently)
12. [Roadmap](#roadmap)
13. [PM Takeaways](#pm-takeaways)

---

## 🔴 Problem Statement

Most AI product teams face a brutal binary choice when shipping RAG-based features:

| Path | The Problem |
|---|---|
| **Cloud-only LLMs** | Sensitive documents sent to third-party APIs → enterprise deals blocked by InfoSec/Legal |
| **Fully on-premise** | High infrastructure cost + maintenance + quality degradation from smaller local models |
| **No RAG at all** | Token costs spiral on long documents; hallucinations increase without grounding |

**The gap nobody was solving:** *Can you get cloud-quality reasoning without leaking the documents that matter most to enterprise buyers?*

---

## 👤 User Persona

### Primary: "The AI PM at a Mid-Market SaaS Company"

| Attribute | Detail |
|---|---|
| **Name** | Arjun / Sarah |
| **Role** | Product Manager, AI/ML Platform |
| **Company Size** | 200–2,000 employees |
| **Industry** | Legal Tech, FinTech, HealthTech, Insurance |
| **Technical Level** | Medium — can read architecture diagrams, writes basic SQL, understands APIs |
| **Budget Ownership** | $50K–$500K annual AI infra budget |
| **Key Stakeholder** | CTO, CISO, VP Engineering |

### Secondary: "The Enterprise Buyer's CISO"

> Blocks deals. Doesn't care about AI quality. Cares about *where the data goes*.

---

## 😤 Pain Points

### 🔴 Critical (Deal-Breakers)
1. **Data residency compliance** — GDPR, HIPAA, SOC2 require documents to stay within network boundaries. Sending raw PDFs to OpenAI/Gemini APIs violates this for 60%+ of enterprise buyers.
2. **Vendor lock-in risk** — Full dependence on OpenAI/Anthropic means pricing changes can destroy unit economics overnight.

### 🟠 High Impact
3. **Token cost explosion** — Naive RAG implementations re-embed entire document corpora on every query. At 1,000 queries/day × 60-page PDFs, costs compound fast.
4. **Hallucination on domain-specific content** — General LLMs without retrieval grounding confidently answer wrong on niche documents (e.g., golf rules edge cases, legal clauses, insurance policy terms).

### 🟡 Medium Impact
5. **Retrieval quality uncertainty** — PMs can't measure whether semantic search is actually finding the right chunks until production failures surface.
6. **DevOps friction slows AI shipping** — Most AI PMs can't unblock Docker networking issues, delaying builds by days waiting for platform teams.

---

## 💡 Product Hypothesis

> **"If embeddings never leave the network, enterprise InfoSec will unblock deals — and if we only send retrieved context (not full documents) to the cloud LLM, we can reduce per-query token cost by 10x."**

**Testable assumptions:**
- [ ] Local embeddings (nomic-embed-text via Ollama) produce retrieval quality ≥ 85% of OpenAI `text-embedding-3-small`
- [ ] Sending only retrieved chunks (vs. full PDF) to Gemini reduces input token cost by ≥ 80%
- [ ] Hybrid architecture passes a basic CISO security review without a full on-premise deployment

---

## 🏗️ Solution Architecture

```
INGESTION PIPELINE (runs once, fully local)
┌─────────────────────────────────────────────────────┐
│  Google Drive PDF  →  n8n  →  Ollama Embed  →  Qdrant │
│  (Rules of Golf)       (local) (nomic-embed-text)  (local vector store) │
└─────────────────────────────────────────────────────┘
                              ↓ Embeddings NEVER leave network

QUERY PIPELINE (per user question)
┌──────────────────────────────────────────────────────────┐
│  User Question  →  Qdrant Semantic Search  →  Top K Chunks │
│                    (local)                                  │
│                              ↓ Only retrieved text (not full PDF) │
│  Top K Chunks  →  Google Gemini  →  Final Answer            │
│                   (cloud, minimal tokens)                   │
└──────────────────────────────────────────────────────────┘

MEMORY LAYER
└── Postgres Chat Memory → persists conversation context across sessions
```
<img width="1431" height="577" alt="image" src="https://github.com/user-attachments/assets/3df9e398-131e-4956-a25a-f94c1c821608" />

### Components

| Component | Tool | Where It Runs | Why |
|---|---|---|---|
| Orchestration | n8n (self-hosted) | Local | Visual workflow, no code needed for ingestion |
| Embeddings | Ollama + nomic-embed-text | Local | Zero data egress; nomic is MTEB-competitive |
| Vector Store | Qdrant | Local | Fast ANN search; Docker-native |
| LLM Reasoning | Google Gemini 2.5 Flash Lite | Cloud | High quality; cost-efficient for short context |
| Memory | PostgreSQL | Local | Persistent, structured, self-hosted |
| Trigger | n8n Chat Trigger | Cloud Edge | Webhook-based; handles session routing |

---

## 📊 Prioritization

### RICE Scoring for Feature Decisions

| Feature | Reach | Impact | Confidence | Effort | RICE Score | Decision |
|---|---|---|---|---|---|---|
| Local embeddings (privacy) | 8/10 | 10/10 | 9/10 | 3/10 | **240** | ✅ Build first |
| Chunk-only context to LLM | 8/10 | 8/10 | 8/10 | 2/10 | **256** | ✅ Core architecture |
| Postgres chat memory | 6/10 | 7/10 | 9/10 | 2/10 | **189** | ✅ Build |
| Retrieval quality benchmarking | 5/10 | 9/10 | 7/10 | 6/10 | **52** | 🔜 Next sprint |
| 100% offline (local LLM) | 4/10 | 7/10 | 6/10 | 8/10 | **21** | 📋 Backlog |
| Multi-document corpus (100+ PDFs) | 6/10 | 8/10 | 5/10 | 7/10 | **34** | 📋 Backlog |

### MoSCoW for MVP

- **Must Have:** Local embeddings, Qdrant retrieval, cloud LLM reasoning, working chat interface
- **Should Have:** Persistent memory, metadata filtering in Qdrant
- **Could Have:** Retrieval latency benchmarking, confidence scoring per answer
- **Won't Have (now):** Multi-tenant isolation, fine-tuned retrieval reranker, UI for non-technical users

---

## 🛠️ Technical Stack Decisions

### Decision Log

**Why nomic-embed-text over OpenAI embeddings?**
- Cost: $0 per embedding (local) vs $0.00002/1K tokens (OpenAI)
- Privacy: Vectors never leave the network
- Quality: nomic-embed-text scores 62.4 on MTEB vs 62.3 for text-embedding-3-small (comparable)
- Risk: No external API dependency for the ingestion pipeline

**Why Qdrant over Pinecone/Weaviate?**
- Self-hosted: no SaaS pricing, full control over data
- Docker-native: single `docker run` to start
- Filtering: payload-based metadata filtering without extra cost
- Risk: Ops burden is on us; no managed backups out of the box

**Why Gemini 2.5 Flash Lite over GPT-4o?**
- Cost: Gemini Flash Lite is among the cheapest frontier-class models
- Context: We're sending short chunks, not full documents — any capable model works
- Risk: Google's API terms may change; architecture makes swapping trivial

**Why n8n over custom Python?**
- Speed: Visual workflows built in hours, not days
- Accessibility: Non-engineers can inspect and modify ingestion logic
- Risk: Less flexible for complex preprocessing; mitigated by keeping n8n for orchestration only

---

## 📈 Success Metrics & KPIs

### North Star Metric
> **Retrieval Precision@5** — Of the 5 chunks returned for a query, what % are genuinely relevant?

### Leading Indicators (Health Metrics)

| Metric | Target | Measurement Method |
|---|---|---|
| Retrieval Precision@5 | ≥ 85% | Manual eval on 50 golden Q&A pairs |
| Answer Accuracy (vs. ground truth) | ≥ 90% | LLM-as-judge (Gemini evaluates own outputs) |
| p95 Query Latency | < 3s end-to-end | n8n execution logs |
| Token cost per query | < $0.001 | Gemini API usage dashboard |
| Zero PII/document egress | 100% | Network egress monitoring |

### Lagging Indicators (Business Metrics)

| Metric | Target | Why It Matters |
|---|---|---|
| Enterprise deals unblocked by InfoSec | 2+ within 90 days | Core hypothesis validation |
| Monthly AI infra cost reduction | ≥ 60% vs cloud-only baseline | Cost model proof |
| Time-to-first-answer for new document | < 30 min (ingest + query) | Operational efficiency |

### Anti-Metrics (What We're NOT Optimizing For)
- Raw query volume (vanity metric without quality gates)
- LLM response length (longer ≠ better)
- Number of PDFs ingested (breadth without retrieval quality is noise)

---

## 💰 Cost Model

### Scenario: 1,000 queries/day on a 60-page PDF corpus

| Approach | Embedding Cost | Query Cost (Input Tokens) | Monthly Estimate |
|---|---|---|---|
| **Naive cloud RAG** (send full doc each query) | ~$0.50/day | ~$15/day (60 pages × 1K tokens × 1K queries) | **~$465/month** |
| **This hybrid approach** (retrieve 5 chunks only) | $0/day (local) | ~$0.15/day (5 chunks × ~300 tokens × 1K queries) | **~$4.50/month** |
| **Savings** | | | **~99% cost reduction** |

> Note: Estimates use Gemini Flash Lite pricing. Actual costs vary with chunk size, overlap, and model selection.

---

## ⚖️ Trade-offs & Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| nomic-embed-text quality drops on domain-specific jargon | Medium | High | Benchmark against OpenAI embeddings on golf/legal vocabulary; swap if delta > 5% |
| Qdrant performance degrades at 10K+ documents | Medium | Medium | Shard collection; add HNSW index tuning; benchmark at 100/1K/10K docs |
| host.docker.internal networking breaks on Linux | High | Low | Use `--network host` flag or explicit container IP |
| Google Gemini API terms change | Low | Medium | Architecture allows drop-in model swap in < 1 hour |
| Local Ollama inference becomes a bottleneck at high concurrency | Low | High | Move to Ollama API server; add horizontal scaling; or pre-compute embeddings |

---

## 🔁 What I'd Do Differently

1. **Benchmark retrieval latency at scale before shipping** — Test at 10, 100, 1,000 PDFs. Qdrant with HNSW handles millions of vectors, but understanding *your* latency curve early prevents architectural surprises.

2. **Build a golden eval set first** — 50 questions with known correct answers from the document. Without this, you can't tell if a model/embedding swap is an improvement or a regression.

3. **Add chunk metadata from the start** — Page number, section heading, document source. This enables metadata filtering in Qdrant and dramatically improves retrieval precision on multi-document corpora.

4. **Log everything to a structured store** — Query, retrieved chunks, final answer, latency. This is your feedback loop for improving retrieval quality over time.

5. **User-test the chat UX earlier** — The RAG pipeline works. But "what question do I even ask?" is a UX problem, not an AI problem. Earlier user testing would have surfaced this.

---

## 🗺️ Roadmap

### ✅ Phase 1 — MVP (Complete)
- [x] PDF ingestion pipeline (Google Drive → Qdrant)
- [x] Local embedding with nomic-embed-text
- [x] Semantic retrieval via Qdrant
- [x] Cloud reasoning via Gemini 2.5 Flash Lite
- [x] Persistent chat memory via Postgres
- [x] n8n-orchestrated end-to-end workflow

### 🔜 Phase 2 — Quality & Observability
- [ ] Golden eval set (50 Q&A pairs, manual labeling)
- [ ] Retrieval Precision@5 baseline measurement
- [ ] Latency benchmarking at 10/100/1,000 documents
- [ ] Chunk metadata enrichment (page number, section)
- [ ] Query logging to structured store

### 📋 Phase 3 — 100% Offline Mode
- [ ] Swap Gemini for quantized Llama 3 (via Ollama)
- [ ] Quality/speed trade-off benchmark (Gemini vs Llama 3 8B vs 70B)
- [ ] Measure answer accuracy delta on golf rules eval set
- [ ] Document minimum hardware requirements for full offline deployment

### 🔭 Phase 4 — Enterprise Hardening
- [ ] Multi-tenant collection isolation in Qdrant
- [ ] Document-level access control
- [ ] Ingestion pipeline for 10+ document types (DOCX, HTML, CSV)
- [ ] Retrieval reranker (cross-encoder) for top-K refinement
- [ ] One-click Docker Compose deployment for enterprise self-hosting

---

## 🎓 PM Takeaways

### 1. Data Privacy Is a Product Feature, Not an Engineering Concern
Enterprise InfoSec teams don't evaluate AI quality. They evaluate *data flow*. Embeddings staying local is a feature you can put on a sales slide. Build it intentionally.

### 2. Token Economics Are a PM Responsibility
The cost difference between "send the whole document" and "send retrieved chunks only" is 99%. This is not an engineering optimization — it's a pricing model decision. PMs who understand token economics ship sustainable products.

### 3. Technical Empathy Unlocks Speed
The hardest part of this build wasn't the AI logic. It was `host.docker.internal`. PMs who can diagnose basic DevOps issues don't wait 3 days for a platform ticket. They unblock themselves in 20 minutes.

### 4. Ship the Eval Framework Before the Product
You can't improve what you can't measure. A 50-question golden eval set is more valuable than 3 extra features in the backlog.

### 5. Architecture Decisions Are Reversible; Data Decisions Aren't
Swapping Gemini for Llama 3 takes an hour. Re-embedding 10,000 documents because you chose the wrong chunking strategy takes a week. Spend time on data modeling, not model selection.

---

## 🔗 Stack

| Tool | Version | Purpose |
|---|---|---|
| n8n | Self-hosted | Workflow orchestration |
| Ollama | Latest | Local model serving |
| nomic-embed-text | Latest | Local text embeddings |
| Qdrant | Latest | Vector store |
| Google Gemini | 2.5 Flash Lite | Cloud reasoning |
| PostgreSQL | 15+ | Chat memory persistence |
| Docker | Latest | Container runtime |

---

## 👤 About

Built by a PM pressure-testing real AI product trade-offs — not just vibing with ChatGPT. If you're working on AI product strategy, RAG systems, or cost models for LLM features, let's connect.

*"The best AI product decisions are 20% model quality and 80% data flow, cost structure, and trust architecture."*

---

*Last updated: May 2026 | Test corpus: Rules of Golf (60 pages) | Status: Phase 1 complete, Phase 2 in progress*
