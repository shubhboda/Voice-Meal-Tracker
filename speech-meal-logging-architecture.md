# Speech Meal Logging — Architecture Design Document

**Version:** 1.0  
**Author:** Senior Software Architect  
**Date:** March 2025

---

## Executive Summary

This document presents a **cost-optimized, scalable architecture** for a Speech Meal Logging feature. The design prioritizes local processing, rule-based parsing, and intelligent caching to achieve **70–90% reduction in AI API costs** while maintaining high accuracy and user experience.

---

## 1. User Journey

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  USER JOURNEY: "I ate 2 eggs and 1 banana for breakfast"                         │
└─────────────────────────────────────────────────────────────────────────────────┘

  [1] Tap Mic          [2] Speak              [3] STT                [4] Parse
  ┌─────────┐          ┌─────────┐           ┌─────────┐            ┌─────────┐
  │  🎤     │  ─────►  │  User   │  ─────►   │ Speech  │  ─────►    │ Text    │
  │  Button │          │  Speaks │           │ to Text │            │ Parser  │
  └─────────┘          └─────────┘           └─────────┘            └─────────┘
                                                                         │
  [7] UI Update        [6] Log Meal          [5] Match Foods              │
  ┌─────────┐          ┌─────────┐           ┌─────────┐                 │
  │ Show    │  ◄─────  │ Meal    │  ◄─────   │ Food    │  ◄──────────────┘
  │ Result  │          │ Logger  │           │ Matcher │
  └─────────┘          └─────────┘           └─────────┘
```

**Steps:**
1. User taps microphone button.
2. User speaks: *"I ate 2 eggs and 1 banana for breakfast"*.
3. **Speech-to-Text** converts audio to text (local or cloud STT).
4. **Text Parser** extracts meal type, quantities, and food phrases.
5. **Food Matcher** looks up foods in local DB → cache → AI fallback.
6. **Meal Logger** persists meal with nutrition values.
7. **UI** displays confirmation and nutrition summary.

---

## 2. Cost Optimization Strategy

### Tiered Processing Model

| Tier | Handler | Cost | % of Requests | Use Case |
|------|---------|------|---------------|----------|
| **Tier 1** | Rule-based parser + local DB | ~$0 | 60–70% | "2 eggs", "1 banana", "grilled chicken" |
| **Tier 2** | Fuzzy match + synonym lookup | ~$0 | 15–20% | "scrambled eggs", "ripe banana" |
| **Tier 3** | Local cache (previously resolved) | ~$0 | 5–10% | User said "avocado toast" before |
| **Tier 4** | AI API fallback | $$$ | 5–15% | "Grandma's lasagna", unknown dishes |

**Cost Reduction Math:**
- **Without optimization:** 100% of requests → AI → ~$0.002/request → $200/100K requests
- **With optimization:** 10% AI fallback → $20/100K requests → **90% cost reduction**

---

## 3. System Components

### 3.1 Component Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           SPEECH MEAL LOGGING SYSTEM                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐   │
│  │   Client     │    │   API        │    │  Text        │    │  Food Matching       │   │
│  │   (Web/Mobile)│───►│   Gateway   │───►│  Parser      │───►│  Engine              │   │
│  └──────┬───────┘    └──────────────┘    └──────────────┘    └──────────┬───────────┘   │
│         │                      │                  │                      │               │
│         │                      │                  │                      │               │
│  ┌──────▼───────┐              │                  │             ┌────────▼───────────┐   │
│  │  Speech-to-  │              │                  │             │  Local Food DB     │   │
│  │  Text Layer  │              │                  │             │  + Synonyms        │   │
│  │  (Local/Cloud)│             │                  │             └────────┬───────────┘   │
│  └──────────────┘              │                  │                      │               │
│                                │                  │             ┌────────▼───────────┐   │
│                                │                  │             │  Cache Layer       │   │
│                                │                  │             │  (Redis/Memcached) │   │
│                                │                  │             └────────┬───────────┘   │
│                                │                  │                      │               │
│                                │                  │             ┌────────▼───────────┐   │
│                                │                  │             │  AI Fallback       │   │
│                                │                  │             │  Service           │   │
│                                │                  │             └───────────────────┘   │
│                                │                  │                                      │
│                                │                  │             ┌──────────────────────┐ │
│                                │                  └────────────►│  Meal Logging        │ │
│                                │                               │  Service             │ │
│                                └──────────────────────────────►└──────────────────────┘ │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Component Descriptions

| Component | Responsibility | AI Usage |
|-----------|----------------|----------|
| **Speech-to-Text** | Convert audio → text. Prefer local (Whisper.cpp, Vosk) or low-cost cloud (AssemblyAI). | Optional |
| **Text Parser** | Extract meal_type, quantities, food_phrases using regex + rule-based NLP. | None |
| **Food Matching Engine** | Match food phrases to DB. Uses exact match → fuzzy → synonym → cache → AI. | Fallback only |
| **Nutrition DB** | Store foods, synonyms, portions, nutrients. | None |
| **Cache Layer** | Store resolved phrase → food_id mappings. | None |
| **AI Fallback Service** | Resolve unknown foods, estimate nutrition, return structured data. | Yes |
| **Meal Logging Service** | Persist meals, aggregate nutrition, serve history. | None |

---

## 4. Local Database Strategy

### 4.1 Schema Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        FOOD DATABASE STRUCTURE                                    │
└─────────────────────────────────────────────────────────────────────────────────┘

foods                          food_synonyms              food_portions
┌─────────────────────┐       ┌─────────────────────┐    ┌─────────────────────┐
│ id (PK)             │       │ id (PK)             │    │ id (PK)             │
│ canonical_name      │◄──────│ food_id (FK)        │    │ food_id (FK)        │
│ category            │       │ synonym             │    │ portion_name        │
│ base_unit           │       │ match_priority      │    │ grams_per_portion   │
│ source              │       └─────────────────────┘    │ is_default          │
│ ai_resolved         │                                  └─────────────────────┘
│ created_at          │
└──────────┬──────────┘
           │
           │ 1:N
           ▼
food_nutrients               ai_resolved_foods (cache for future)
┌─────────────────────┐      ┌─────────────────────┐
│ id (PK)             │      │ id (PK)             │
│ food_id (FK)        │      │ original_phrase     │  ◄── Hash of user input
│ nutrient_type       │      │ food_id (FK)        │
│ amount_per_100g     │      │ confidence          │
└─────────────────────┘      │ created_at          │
                             └─────────────────────┘
```

### 4.2 Table Definitions

**foods**
- `canonical_name`: Standard name (e.g., "chicken egg")
- `category`: breakfast, fruit, protein, etc.
- `base_unit`: "piece", "100g", "cup"
- `source`: "usda", "open_food_facts", "ai_resolved"
- `ai_resolved`: boolean — if true, came from AI fallback

**food_synonyms**
- Maps "scrambled eggs" → eggs, "ripe banana" → banana
- `match_priority`: Higher = prefer when ambiguous

**food_portions**
- "1 medium banana" = 118g, "1 large egg" = 50g

**food_nutrients**
- Per 100g: calories, protein, carbs, fat, fiber, etc.

**ai_resolved_foods**
- Stores `original_phrase → food_id` for instant lookup next time
- Key: hash(normalized_phrase) → value: food_id

### 4.3 Synonym Strategy

| Input | Canonical Food | Synonym Entry |
|-------|----------------|---------------|
| "2 eggs" | chicken egg | (egg, eggs, 2) |
| "scrambled eggs" | chicken egg | scrambled eggs |
| "banana" | banana | banana, bananas |

---

## 5. Caching Strategy

### 5.1 Multi-Layer Cache

```
Request: "avocado toast"
    │
    ├─► L1: In-memory (app server) — phrase_hash → food_ids
    │       TTL: 1 hour, LRU eviction
    │
    ├─► L2: Redis
    │       Key: meal:phrase:{hash}
    │       Value: { food_id, portion_id, nutrients }
    │       TTL: 7 days
    │
    └─► L3: ai_resolved_foods (PostgreSQL)
            Permanent storage for AI-resolved items
```

### 5.2 Cache Key Design

```
phrase_hash = SHA256(normalize("avocado toast"))
  - Lowercase
  - Trim whitespace
  - Remove punctuation
  - Optional: collapse numbers ("2 eggs" → "eggs" for quantity-agnostic match)
```

### 5.3 Cache Population

1. **On AI resolution:** Write to Redis + ai_resolved_foods
2. **On DB match:** Write to Redis (optional, for hot paths)
3. **On app startup:** Warm cache with top N user phrases (analytics-driven)

### 5.4 Cache Invalidation

- **Never** for ai_resolved_foods (permanent)
- **TTL** for Redis (7–30 days)
- **LRU** for in-memory

---

## 6. AI Fallback Strategy

### 6.1 When to Invoke AI

| Condition | Action |
|-----------|--------|
| No exact/fuzzy match in DB | → AI |
| Phrase not in cache | → AI |
| Confidence < 0.7 from fuzzy match | → AI |
| Composite dish ("Grandma's lasagna") | → AI |
| Ambiguous (e.g., "pie" — apple? pizza?) | → AI (or ask user) |

### 6.2 AI Request/Response Contract

**Request:**
```json
{
  "phrase": "avocado toast",
  "context": {
    "meal_type": "breakfast",
    "quantity_hint": 1,
    "user_id": "uuid" 
  }
}
```

**Response (structured):**
```json
{
  "foods": [
    {
      "canonical_name": "avocado toast",
      "estimated": true,
      "nutrients_per_100g": {
        "calories": 195,
        "protein": 4.2,
        "carbs": 18,
        "fat": 12
      },
      "default_portion": { "name": "slice", "grams": 85 }
    }
  ]
}
```

### 6.3 AI Result Persistence Flow

```
AI returns result
    │
    ├─► Create/update food in DB (source=ai_resolved)
    ├─► Add to food_synonyms (phrase → food_id)
    ├─► Write to ai_resolved_foods
    └─► Write to Redis cache
```

**Next time** user says "avocado toast" → cache/DB hit → **no AI call**.

---

## 7. Scalable Architecture

### 7.1 High-Level Architecture

```
                                    ┌─────────────────┐
                                    │   CDN / Edge    │
                                    └────────┬────────┘
                                             │
                                    ┌────────▼────────┐
                                    │  API Gateway    │
                                    │  (Kong/AWS API) │
                                    └────────┬────────┘
                                             │
              ┌──────────────────────────────┼──────────────────────────────┐
              │                              │                              │
    ┌─────────▼─────────┐          ┌────────▼────────┐          ┌──────────▼──────────┐
    │  STT Service      │          │  Meal Ingestion  │          │  Meal History       │
    │  (Async/Streaming)│          │  Service         │          │  Service            │
    └─────────┬─────────┘          └────────┬────────┘          └──────────┬──────────┘
              │                             │                              │
              │                    ┌────────▼────────┐                     │
              │                    │  Text Parser    │                     │
              │                    │  (Stateless)    │                     │
              │                    └────────┬────────┘                     │
              │                             │                              │
              │                    ┌────────▼────────┐                     │
              │                    │  Food Matching  │                     │
              │                    │  Service        │                     │
              │                    └────────┬────────┘                     │
              │                             │                              │
              │         ┌───────────────────┼───────────────────┐          │
              │         │                   │                   │          │
    ┌─────────▼─────────▼───┐   ┌──────────▼──────────┐   ┌────▼─────────────┐
    │  Redis Cluster        │   │  PostgreSQL         │   │  AI Service      │
    │  (Cache + Sessions)   │   │  (Foods + Meals)    │   │  (OpenAI/Claude) │
    └──────────────────────┘   └─────────────────────┘   └──────────────────┘
```

### 7.2 Service Breakdown

| Service | Scale Strategy | DB/Cache |
|---------|----------------|----------|
| **API Gateway** | Auto-scale, rate limiting | - |
| **STT Service** | Queue + workers, burst capacity | - |
| **Meal Ingestion** | Stateless, horizontal scale | Redis (cache), PG (meals) |
| **Food Matching** | Stateless, read-heavy | Redis, PG (foods) |
| **Meal History** | Read replicas | PG |
| **AI Service** | Queue, rate limit, circuit breaker | PG (ai_resolved) |

### 7.3 Data Flow (Write Path)

```
Client → API Gateway → Meal Ingestion Service
                            │
                            ├─► Parse text (in-process)
                            ├─► For each food phrase:
                            │       ├─► Check Redis cache
                            │       ├─► Check ai_resolved_foods
                            │       ├─► Query food DB (exact/fuzzy)
                            │       └─► If miss: enqueue AI job
                            │
                            └─► Persist meal → PostgreSQL
```

---

## 8. Technology Stack Suggestions

### 8.1 Recommended Stack

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Frontend** | React Native / Flutter (mobile), React/Next.js (web) | Cross-platform, good audio APIs |
| **Speech-to-Text** | **Local:** Whisper.cpp, Vosk / **Cloud:** AssemblyAI, Deepgram | Local = $0, Cloud = pay-per-use |
| **Backend** | Node.js (NestJS) or Go (Fiber/Gin) | Async, fast JSON handling |
| **API Gateway** | Kong, AWS API Gateway, or Traefik | Rate limiting, auth |
| **Food DB** | PostgreSQL + pg_trgm (fuzzy) | ACID, full-text, trigram search |
| **Cache** | Redis Cluster | Sub-ms reads, TTL support |
| **AI** | OpenAI GPT-4o-mini, Claude Haiku, or local Llama | Cost/quality balance |
| **Queue** | Redis (BullMQ) or RabbitMQ | Async AI processing |

### 8.2 Parsing & NLP

| Need | Tool |
|------|-----|
| Regex parsing | Built-in (quantity + food phrase extraction) |
| Fuzzy matching | pg_trgm, Levenshtein, or Fuse.js |
| Tokenization | Simple split + stop words (local) |
| Entity extraction | Rule-based patterns first; AI only on fallback |

---

## 9. Cost Reduction Strategy (70–90%)

### 9.1 Breakdown by Request Type

| Path | % Requests | Cost/Req | Total |
|------|------------|----------|-------|
| Local DB exact match | 50% | $0 | $0 |
| Synonym/fuzzy match | 20% | $0 | $0 |
| Cache hit (prior AI) | 15% | $0 | $0 |
| AI fallback | 15% | ~$0.002 | $0.0003 |
| **Blended** | 100% | **~$0.0003** | **~$30/100K** |

vs. **naive:** 100% AI → ~$200/100K

**Reduction: ~85%**

### 9.2 Additional Optimizations

1. **Batch AI requests:** If multiple unknowns in one utterance, single AI call.
2. **Smaller models:** Use GPT-4o-mini or Haiku, not GPT-4.
3. **Prompt caching:** For repeated prompts (e.g., system prompt), use API caching.
4. **Local STT:** Whisper.cpp on device = $0 for transcription.
5. **Aggressive TTL:** Cache AI results indefinitely in DB.

---

## 10. Deliverables

### 10.1 System Architecture Diagram (Text)

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                          SPEECH MEAL LOGGING — FULL ARCHITECTURE                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘

                                    ┌──────────────────┐
                                    │   Mobile / Web   │
                                    │   Client         │
                                    └────────┬─────────┘
                                             │ HTTPS / WebSocket
                                             ▼
                                    ┌──────────────────┐
                                    │   API Gateway    │
                                    │   Rate Limit     │
                                    └────────┬─────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
                    ▼                        ▼                        ▼
           ┌───────────────┐       ┌─────────────────┐       ┌─────────────────┐
           │  STT Service  │       │ Meal Ingestion  │       │ Meal History    │
           │  /speech/     │       │ /meals/log      │       │ /meals          │
           │  transcribe   │       └────────┬────────┘       └────────┬────────┘
           └───────┬───────┘                │                        │
                   │                        │                        │
                   │ (text)                 ▼                        │
                   │               ┌─────────────────┐               │
                   └──────────────►│ Text Parser     │               │
                                  │ (regex + rules) │               │
                                  └────────┬────────┘               │
                                           │                        │
                                           ▼                        │
                                  ┌─────────────────┐               │
                                  │ Food Matching   │               │
                                  │ Engine          │               │
                                  └────────┬────────┘               │
                                           │                        │
              ┌────────────────────────────┼────────────────────────┤
              │                            │                        │
              ▼                            ▼                        ▼
     ┌────────────────┐          ┌────────────────┐        ┌────────────────┐
     │ Redis Cache    │          │ PostgreSQL     │        │ AI Service     │
     │ phrase→food    │          │ foods, meals   │        │ (fallback)     │
     └────────────────┘          └────────────────┘        └────────────────┘
```

### 10.2 Example Database Schema (SQL)

```sql
-- Core food table
CREATE TABLE foods (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    canonical_name VARCHAR(255) NOT NULL,
    category VARCHAR(50),
    base_unit VARCHAR(20) DEFAULT '100g',
    source VARCHAR(50) DEFAULT 'usda',
    ai_resolved BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(canonical_name)
);

-- Synonyms for flexible matching
CREATE TABLE food_synonyms (
    id SERIAL PRIMARY KEY,
    food_id UUID REFERENCES foods(id),
    synonym VARCHAR(255) NOT NULL,
    match_priority INT DEFAULT 0,
    UNIQUE(synonym)
);

-- Portions (e.g., 1 medium banana = 118g)
CREATE TABLE food_portions (
    id SERIAL PRIMARY KEY,
    food_id UUID REFERENCES foods(id),
    portion_name VARCHAR(100),
    grams_per_portion DECIMAL(10,2),
    is_default BOOLEAN DEFAULT FALSE
);

-- Nutrients per 100g
CREATE TABLE food_nutrients (
    id SERIAL PRIMARY KEY,
    food_id UUID REFERENCES foods(id),
    nutrient_type VARCHAR(50),
    amount_per_100g DECIMAL(12,4),
    unit VARCHAR(20),
    UNIQUE(food_id, nutrient_type)
);

-- AI-resolved phrase cache (permanent)
CREATE TABLE ai_resolved_foods (
    id SERIAL PRIMARY KEY,
    phrase_hash CHAR(64) UNIQUE NOT NULL,
    original_phrase TEXT NOT NULL,
    food_id UUID REFERENCES foods(id),
    confidence DECIMAL(3,2),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ai_resolved_hash ON ai_resolved_foods(phrase_hash);
CREATE INDEX idx_food_synonyms_synonym ON food_synonyms(synonym);
CREATE INDEX idx_foods_canonical ON foods(canonical_name);

-- Meals (user meal logs)
CREATE TABLE meals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    meal_type VARCHAR(50),
    logged_at TIMESTAMPTZ DEFAULT NOW(),
    source VARCHAR(20) DEFAULT 'speech'
);

CREATE TABLE meal_items (
    id SERIAL PRIMARY KEY,
    meal_id UUID REFERENCES meals(id),
    food_id UUID REFERENCES foods(id),
    quantity DECIMAL(10,2),
    portion_id INT REFERENCES food_portions(id),
    calories DECIMAL(10,2),
    protein DECIMAL(10,2),
    carbs DECIMAL(10,2),
    fat DECIMAL(10,2)
);
```

### 10.3 Example API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/speech/transcribe` | Upload audio, return text |
| POST | `/api/v1/meals/log` | Log meal from text or structured data |
| GET | `/api/v1/meals` | List user meals (paginated) |
| GET | `/api/v1/meals/:id` | Get meal detail |
| POST | `/api/v1/meals/parse-preview` | Parse text only, return matched foods (no persist) |

**POST /api/v1/meals/log**

Request:
```json
{
  "text": "I ate 2 eggs and 1 banana for breakfast",
  "logged_at": "2025-03-07T08:30:00Z"
}
```

Response:
```json
{
  "meal_id": "uuid",
  "meal_type": "breakfast",
  "items": [
    { "food": "chicken egg", "quantity": 2, "calories": 140 },
    { "food": "banana", "quantity": 1, "calories": 105 }
  ],
  "total_calories": 245,
  "ai_calls_used": 0
}
```

### 10.4 Example Workflow (Pseudocode)

```
FUNCTION logMealFromSpeech(audioBlob):
    text = STT_SERVICE.transcribe(audioBlob)
    RETURN logMealFromText(text)

FUNCTION logMealFromText(text):
    parsed = PARSER.extract(text)
    // parsed = { meal_type: "breakfast", items: [{ phrase: "2 eggs", qty: 2 }, { phrase: "1 banana", qty: 1 }] }
    
    matched_items = []
    ai_calls = 0
    
    FOR EACH item IN parsed.items:
        result = FOOD_MATCHER.resolve(item.phrase)
        
        IF result.source == "cache" OR result.source == "db":
            matched_items.push(result)
        ELSE:
            ai_result = AI_SERVICE.resolve(item.phrase)
            ai_calls++
            SAVE_TO_DB(ai_result)
            SAVE_TO_CACHE(item.phrase, ai_result)
            matched_items.push(ai_result)
    
    meal = MEAL_SERVICE.create(parsed.meal_type, matched_items)
    RETURN { meal, ai_calls_used: ai_calls }
```

---

## Appendix A: Rule-Based Parser Patterns

Example regex/rule patterns for common phrases:

```
Pattern: (\\d+)\\s+(.+)
  "2 eggs" → qty=2, phrase="eggs"

Pattern: (one|two|a|an)\\s+(.+)
  "one banana" → qty=1, phrase="banana"

Meal type keywords: breakfast, lunch, dinner, snack
```

---

## Appendix B: Monitoring & Observability

- **Metrics:** AI call rate, cache hit rate, latency p95
- **Alerts:** AI call rate > 20%, cache hit rate < 60%
- **Logging:** Structured logs with trace_id for request flow

---

*Document end. Ready for implementation.*
