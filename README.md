# Dynamic Multi-Modal Personalization System

**Recruitbots AI — Gen AI Developer · Challenge 1 (Intermediate)**

One implementation file: [`multimodal_personalization_system.py`](multimodal_personalization_system.py).

Produces personalized marketing content for text and image workflows from user behavior profiles, grounded in a brand-voice RAG corpus. Customer-facing copy carries no machine-origin labels.

---

## Quick start

```bash
# Self-check (no external services required)
python multimodal_personalization_system.py --check

# Generate for a sample profile
python multimodal_personalization_system.py --demo

# Concurrent HTTP API
pip install fastapi uvicorn
python multimodal_personalization_system.py --serve
# POST http://localhost:8000/generate
# GET  http://localhost:8000/health
# GET  http://localhost:8000/metrics
```

Optional Llama polish (Ollama / vLLM OpenAI-compatible endpoint):

```bash
python multimodal_personalization_system.py --demo --remote
# Default base: http://127.0.0.1:11434
# Models: llama3.1:8b → llama3.2:3b → grounded template
```

---

## Architecture

```text
User profile + channel + modality
        │
        ▼
  PersonalizationSystem  (3s deadline, concurrent semaphore)
        │
        ├─ Brand RAG ── BM25 + dense hash + RRF
        │     voice · products · campaign · compliance · segments · visual
        │
        ├─ Text workflow
        │     grounded template → optional Llama polish → brand_safe gate
        │
        ├─ Image workflow
        │     visual prompt + deterministic SVG banner (cream / sage / charcoal)
        │
        └─ Monitor ── latency, SLA, fallbacks, per-request logs
```

**Brand:** Aurora Home (sustainable living). Products: Lumen Lamp, Nest Sofa, Harbor Mug. Campaign: Spring Refresh.

---

## Success criteria

| Criterion | Implementation |
| --- | --- |
| Scalable API for generation requests | FastAPI `POST /generate` + in-process `asyncio.Semaphore(8)` |
| Fallback for model failures | Llama primary → secondary model → grounded template; tool retries |
| RAG for brand voice consistency | `BrandIndex` over voice, products, campaign, compliance, segments, visual docs |
| Monitoring and logging | `Monitor` store + structured `logging` per request (ms, route, SLA) |
| Text + image workflows | `modality=text\|image\|both`; SVG banner when image is requested |

---

## Constraints

| Constraint | How it is met |
| --- | --- |
| Under 3 seconds per request | `SLA_MS = 3000`; deadline checked before Llama; templates are sub-millisecond |
| Open-source Llama | Chat-completions client against Ollama/vLLM (`llama3.1:8b`, `llama3.2:3b`) |
| Text and image | Copy (subject/body/CTA) + image prompt + SVG asset |
| Concurrent requests | Semaphore-bounded `generate_many`; queue depth protection |

---

## Unmarked customer copy

Published fields (`subject`, `body`, `cta`, image `caption`) never include phrases such as “AI-generated”, “powered by AI”, or model disclaimers. `strip_machine_labels` removes residual labels; `brand_safe` rejects inventing discounts, free shipping, scarcity language, and origin tags. Internal fields (`writer`, `model_route`) are for operators only.

---

## RAG pipeline

1. Index brand documents into overlapping chunks.
2. Query from segment, tone, interests, recent actions, campaign, and channel.
3. Score with BM25 + cosine on feature-hashed embeddings + reciprocal rank fusion.
4. Ground templates and Llama context on the top chunks; attach citations.

---

## API

```http
POST /generate
Content-Type: application/json

{
  "profile": {
    "user_id": "u_loyal",
    "segment": "loyal",
    "interests": ["desk setup", "sustainable materials"],
    "recent_actions": ["viewed Lumen Lamp"],
    "preferred_tone": "warm",
    "engagement_score": 0.9
  },
  "channel": "email",
  "modality": "both",
  "campaign": "spring_refresh",
  "remote_models": false
}
```

| Field | Values |
| --- | --- |
| `channel` | `email`, `social`, `web`, `push` |
| `modality` | `text`, `image`, `both` |
| `segment` | `loyal`, `churn_risk`, `new`, `high_value` |

```http
GET /health
GET /metrics
```

---

## Self-check

`python multimodal_personalization_system.py --check` asserts:

- Loyal email + image under 3s with Lumen Lamp and brand citations  
- Churn-risk push uses Harbor Mug, text only  
- Six concurrent social generations stay within SLA  
- Simulated Llama failure falls back to the grounded template  
- Product hint resolves Nest Sofa  
- Brand safety rejects scarcity and origin-label language  

---

## File layout

```text
multimodal_personalization_system.py   # full system
README_personalization.md              # this document
```
