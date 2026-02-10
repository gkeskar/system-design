# System Design: Artist Sentiment Dashboard

## The Prompt

> "I run an agency for talented people. I want a tool that helps me understand the public sentiment towards my artists."
> 
> Requirements:
> - Visual representation of sentiment
> - Data sources: Twitter, Facebook, TikTok
> - How sentiment has changed over time

---

## Design Approach

> "I start with the user: what do they see and need? Then I design APIs to serve that UI. Then I pick patterns (microservices, event-driven, CQRS) that fit the read/write characteristics. Then I choose technologies. Finally, I design infrastructure for scale and reliability."

### Interview Flow (45 minutes)

| Time | What you do |
|------|-------------|
| **0–5 min** | Clarify requirements, state assumptions |
| **5–10 min** | **UI first**: sketch dashboard, identify data needs |
| **10–18 min** | **API design**: REST endpoints from UI needs |
| **18–28 min** | **Services & patterns**: microservices, event-driven, CQRS |
| **28–35 min** | **Technology choices**: DBs, event log, cache |
| **35–42 min** | **Infrastructure**: scaling, network, performance |
| **42–45 min** | Recap, tradeoffs, future improvements |

---

## Clarifying Questions (Ask First!)

### User/Product Scope
- Who are the users? (agency admins, artists, clients?)
- What's the main workflow? (search artist → dashboard → alerts?)
- What decisions will you make with this data?

### Sentiment Definition
- What is "sentiment"? (pos/neu/neg? 1-5 score? emotions?)
- Do you need explainability? (why negative—top posts, key phrases)
- Language support? (English only vs multilingual)

### Time & Freshness
- Real-time vs batch? (seconds/minutes vs hourly/daily)
- **Answer: Hourly buckets**
- Historical backfill needed? (last 7 days vs years)

### Scale Estimates (for back-of-envelope)
- Number of artists tracked?
- Mentions per artist per day?
- Retention requirements?

---

# Step 1: User Perspective (UI First)

## What does the user need?

```
"I'm an agency manager. I want to:
 1. Pick an artist
 2. See how their sentiment changed over time
 3. Understand WHY it changed (what topics/events caused it)
 4. See actual posts as evidence"
```

---

## Dashboard Wireframe

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Artist Sentiment Dashboard                                       [User: Agency Admin]│
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  Artist: [ John Smith      v]    From: [01/01/2020]  To: [01/01/2025]  [Go] │    │
│  │                                                                              │    │
│  │  Platform: (*) All  ( ) Twitter  ( ) Facebook  ( ) TikTok                   │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  SENTIMENT OVER TIME                                          Bucket: [Week v]   │
│  │                                                                              │    │
│  │   1.0 |                                                                     │    │
│  │       |          /---\                                    /--\              │    │
│  │   0.5 |    /----/    \---\              /----\      /----/  \--\           │    │
│  │       | --/              \------\  /---/    \------/          \---        │    │
│  │   0.0 +-------------------------- \/-------------------------------------  │    │
│  │       |                          ^                                          │    │
│  │  -0.5 |                     [Click to drill down]                          │    │
│  │       |                                                                     │    │
│  │       +----+----+----+----+----+----+----+----+----+----+----+----+----   │    │
│  │          2020  '21  '22  '23  '24  '25                                      │    │
│  │                                                                              │    │
│  │  --- Avg Sentiment Score    ### Positive %    ### Negative %               │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌──────────────────────────────────┐  ┌──────────────────────────────────────┐    │
│  │  TOP NEGATIVE DRIVERS            │  │  TOP POSITIVE DRIVERS                │    │
│  │                                  │  │                                      │    │
│  │  1. "Summer Tour 2023"  -0.6    │  │  1. "New Album 2024"    +0.8        │    │
│  │  2. "Concert Delay"     -0.4    │  │  2. "Charity Event"     +0.7        │    │
│  │  3. "Interview Clip"    -0.3    │  │  3. "Collab Song"       +0.6        │    │
│  │                                  │  │                                      │    │
│  │  [View more...]                  │  │  [View more...]                      │    │
│  └──────────────────────────────────┘  └──────────────────────────────────────┘    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Drill-down Panel (when user clicks a chart point)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Week of Oct 15, 2023                                                  [X Close]    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Summary:  Total: 1,247 mentions | Pos: 312 (25%) | Neu: 435 (35%) | Neg: 500 (40%) │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  TOP NEGATIVE MENTIONS                                  Filter: [Negative v] │    │
│  ├─────────────────────────────────────────────────────────────────────────────┤    │
│  │                                                                              │    │
│  │  @user123 · Twitter · Oct 16                                 Score: -0.9    │    │
│  │  "Can't believe John cancelled the show last minute. Never buying           │    │
│  │   tickets again! #disappointed"                                              │    │
│  │   Comments: 234  Retweets: 89  Likes: 12                                     │    │
│  │  Topic: [Summer Tour 2023]                                                   │    │
│  │                                                                              │    │
│  │  ──────────────────────────────────────────────────────────────────────────  │    │
│  │                                                                              │    │
│  │  Jane Smith · Facebook · Oct 17                              Score: -0.7    │    │
│  │  "The concert delay was handled so poorly. No communication at all."        │    │
│  │   Likes: 56  Comments: 23                                                    │    │
│  │  Topic: [Concert Delay]                                                      │    │
│  │                                                                              │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  [Load more...]                                                    Page 1 of 12     │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## What data does the UI need?

| UI Element | Data needed |
|------------|-------------|
| Artist dropdown | List of artists |
| Sentiment chart | Sentiment scores per time bucket |
| Top drivers | Topics with sentiment deltas |
| Drill-down | Actual posts with sentiment |

**This tells us what APIs we need → Step 2**

---

# Step 2: API Design

## REST Endpoints (derived from UI needs)

| Endpoint | Purpose | UI Element |
|----------|---------|------------|
| `GET /artists?query=john` | Search artists | Artist dropdown |
| `GET /artists/{id}` | Artist details | Artist info |
| `GET /artists/{id}/sentiment/timeseries?from=&to=&bucket=week&platform=all` | Chart data | Sentiment chart |
| `GET /artists/{id}/sentiment/drivers?from=&to=&limit=10` | Top topics | Drivers panel |
| `GET /artists/{id}/mentions?from=&to=&sentiment=neg&topicId=...&limit=50&cursor=...` | Sample posts | Drill-down |

---

## Response Shapes

### Timeseries Response
```json
{
  "artistId": "john_123",
  "bucket": "week",
  "from": "2020-01-01",
  "to": "2025-01-01",
  "series": [
    {
      "bucket_start": "2020-01-06T00:00:00Z",
      "pos_count": 70,
      "neu_count": 30,
      "neg_count": 20,
      "total_count": 120,
      "avg_score": 0.32
    }
  ]
}
```

### Drivers Response
```json
{
  "artistId": "john_123",
  "drivers": [
    {
      "topic_id": "summer_tour_2023",
      "topic_name": "Summer Tour 2023",
      "avg_score": -0.6,
      "mention_count": 450,
      "delta_from_baseline": -0.4
    }
  ]
}
```

### Mentions Response
```json
{
  "mentions": [
    {
      "mention_id": "m_001",
      "platform": "twitter",
      "created_at": "2023-10-16T14:30:00Z",
      "text": "Can't believe John cancelled...",
      "sentiment_label": "negative",
      "sentiment_score": -0.9,
      "topic_ids": ["summer_tour_2023"],
      "metrics": { "likes": 12, "retweets": 89 }
    }
  ],
  "cursor": "abc123",
  "has_more": true
}
```

**This tells us what services and storage we need → Step 3**

---

# Step 3: Services & Patterns

## What services do we need?

| Service | Responsibility |
|---------|----------------|
| **Query API** | Serve the UI (read path) |
| **Collectors** | Fetch data from Twitter/FB/TikTok |
| **Enrichment** | Process mentions (sentiment, topics) |
| **Aggregation** | Compute time buckets |
| **Artist Config** | Manage artist data |

---

## Pattern 1: Microservices

> **Each service does ONE thing well and can be deployed/scaled independently.**

| Service | Why separate? |
|---------|---------------|
| **Twitter Collector** | Different API, auth, rate limits |
| **Facebook Collector** | Different API, auth, rate limits |
| **TikTok Collector** | Different API, auth, rate limits |
| **Enrichment Service** | CPU-intensive; scales with volume |
| **Sentiment Service** | ML model; may need GPU; swappable |
| **Topic Extraction** | Different logic; can evolve independently |
| **Aggregation Service** | Write-heavy; scales with throughput |
| **Query API** | Read-heavy; user-facing; needs low latency |

---

## Pattern 2: Event-Driven Architecture

> **Decouple producers and consumers via an event log/queue.**

```
Collectors → Event Log → Enrichment → Aggregation
```

**Benefits:**
- Services scale independently
- Queue absorbs traffic spikes
- Failures don't cascade
- Can replay events

---

## Streaming Windowing (how we create hourly buckets)

We have a continuous stream of mentions. How do we group them into "hourly" data points for the chart?

### 1) Tumbling Window (Recommended)

```
Time:   12:00         13:00         14:00         15:00
        │─────────────│─────────────│─────────────│
        │  Window 1   │  Window 2   │  Window 3   │
        │  12:00-12:59│  13:00-13:59│  14:00-14:59│
        │─────────────│─────────────│─────────────│
```

- Fixed, **non-overlapping** blocks
- Every mention belongs to **exactly one** window
- At 13:00, Window 1 closes → you have final counts for 12:00 hour
- **Clean, simple, no double-counting**
- **Best for:** "Show me hourly sentiment" → one chart point per hour

### 2) Hopping Window (Optional enhancement)

```
Time:   12:00    12:15    12:30    12:45    13:00    13:15
        │────────────────────────────────────│  Window A (12:00-12:59)
                 │────────────────────────────────────│  Window B (12:15-13:14)
                          │────────────────────────────────────│  Window C (12:30-13:29)
```

- Windows **overlap**
- A mention at 12:30 is in **Window A, B, and C**
- You get a **new result every 15 minutes** (more frequent updates)
- But each result still covers a 1-hour range
- **Best for:** "Update the chart every 15 min but show a 1-hour moving average"
- **Downside:** More complex, mentions counted in multiple windows

### 3) Sliding Window (Not ideal for charts)

```
At 12:37:  window = 11:37 – 12:37
At 12:38:  window = 11:38 – 12:38
At 12:39:  window = 11:39 – 12:39
```

- Window moves **continuously** with every event or clock tick
- Always shows "the last 60 minutes" relative to right now
- No fixed boundaries
- **Best for:** Real-time dashboards ("what's sentiment right now?")
- **Downside:** No clean hourly buckets; can't easily roll up to daily/weekly; expensive to compute

### 4) Session Window (Not a fit)

- Groups events by **activity gaps** (e.g., no mentions for 30 min = new session)
- Used for user sessions, clickstreams
- Our data is **continuous and time-based**, not session-based
- **Skip this one**

### Comparison Table

| Requirement | Tumbling | Hopping | Sliding |
|-------------|----------|---------|---------|
| Clean hourly chart points | Yes | Sort of | No |
| Easy rollups (day/week/month) | Yes | Complex | No |
| Simple to implement | Yes | Medium | Complex |
| Frequent updates within the hour | No | Yes | Yes |
| No double-counting | Yes | No | No |
| Easy to explain in interview | Yes | Medium | Medium |

### Our Choice

- **Primary: Tumbling window (1 hour)**
  - Matches the "hourly" requirement exactly
  - Clean buckets → easy rollups → simple queries
  - No double-counting
- **Optional enhancement: Hopping window**
  - If interviewer asks "can users see updates more frequently?"
  - Use 1-hour window hopping every 15 min

### Interview one-liners

- **Tumbling:** "Fixed 1-hour blocks, no overlap. Each mention counted once. Clean chart points."
- **Hopping:** "Same 1-hour window but slides every 15 min, so we get more frequent updates at the cost of overlap."
- **Sliding:** "Continuous window relative to now. Great for real-time but doesn't give clean hourly buckets."
- **Session:** "Not applicable—our data is time-based, not session-based."

---

## Pattern 3: CQRS (Command Query Responsibility Segregation)

> **Write path and read path are optimized differently.**

```
WRITE SIDE (Command)              READ SIDE (Query)
─────────────────────             ─────────────────
Collectors                        Query API
    ↓                                 ↑
Event Log                         Aggregates Store
    ↓                             (precomputed for fast reads)
Enrichment
    ↓
Aggregation → writes to Aggregates
```

**Why CQRS fits here:**
- **Write path**: High volume, append-only, async processing
- **Read path**: Low latency, precomputed aggregates, cacheable

---

## Pattern 4: Event Sourcing

> **Events are the source of truth. Aggregates are derived views.**

**Simple analogy: Bank account**
- Without event sourcing: Store "Balance = $500" (can't explain how)
- With event sourcing: Store every transaction (can recalculate)

**Benefits for our system:**
- **Reprocessing**: If sentiment model improves, replay and recalculate
- **New views**: Add region breakdown without re-fetching from APIs
- **Audit trail**: Explain any chart point with original posts

---

## Services Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           EXTERNAL APIS                                      │
│         Twitter API        Facebook API         TikTok API                  │
└──────────┬─────────────────────┬─────────────────────┬──────────────────────┘
           │                     │                     │
           ▼                     ▼                     ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│     Twitter      │  │    Facebook      │  │     TikTok       │
│    Collector     │  │    Collector     │  │    Collector     │
└────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                               ▼
              ┌────────────────────────────────┐
              │          EVENT LOG             │
              │    (append-only, immutable)    │
              └───────────────┬────────────────┘
                              │
                              ▼
              ┌────────────────────────────────┐
              │      ENRICHMENT SERVICE        │
              │  • Dedupe                      │
              │  • Calls Sentiment Service     │
              │  • Calls Topic Service         │
              └───────┬───────────┬────────────┘
                      │           │
         ┌────────────┘           └────────────┐
         ▼                                     ▼
┌─────────────────┐                 ┌─────────────────┐
│   SENTIMENT     │                 │ TOPIC EXTRACTION│
│    SERVICE      │                 │    SERVICE      │
└─────────────────┘                 └─────────────────┘

              ┌────────────────────────────────┐
              │     AGGREGATION SERVICE        │
              │  (event consumer - subscribes) │
              │  • Hourly buckets              │
              │  • Daily/weekly rollups        │
              │  • Topic aggregates            │
              └───────────────┬────────────────┘
                              │
                              ▼
              ┌────────────────────────────────┐
              │          DATA STORES           │
              └───────────────┬────────────────┘
                              │
                              ▼
              ┌────────────────────────────────┐
              │       QUERY API SERVICE        │
              └───────────────┬────────────────┘
                              │
                              ▼
              ┌────────────────────────────────┐
              │        DASHBOARD UI            │
              └────────────────────────────────┘
```

---

## How Aggregation Service Works (Event Consumer)

**Nobody "calls" it — it subscribes to events and processes them.**

```python
# Pseudocode
while True:
    event = event_log.read_next()
    hour_bucket = floor_to_hour(event.created_at)
    
    # Update artist aggregate
    update_aggregate(
        key = (event.artist_id, hour_bucket),
        label = event.sentiment_label,
        score = event.sentiment_score
    )
    
    # Update topic aggregates
    for topic_id in event.topic_ids:
        update_aggregate(
            key = (event.artist_id, topic_id, hour_bucket),
            label = event.sentiment_label,
            score = event.sentiment_score
        )
    
    event_log.commit()
```

---

## Service Communication

| Type | Where | Why |
|------|-------|-----|
| **Async (Event Log)** | Collectors → Enrichment → Aggregation | Decoupled, reliable, replayable |
| **Sync (HTTP)** | Enrichment → Sentiment Service | Need immediate response |
| **Sync (HTTP)** | Enrichment → Topic Service | Need immediate response |
| **Sync (HTTP)** | Query API → DB | User-facing reads |

---

# Step 4: Technology Choices

## Database Choices

| Store | Requirement | Good fit | Why |
|-------|-------------|----------|-----|
| **Event Log** | Append-only, high throughput, replay | Kafka, Pulsar | Distributed, durable, replayable |
| **Mentions Store** | Flexible schema, text search | Document DB (MongoDB) | Posts have varying shapes per platform |
| **Aggregates Store** | Fast time-range queries | Document DB or Time-series DB | Precomputed buckets, indexed by time |
| **Artist Config** | Simple CRUD, low volume | Relational (Postgres) | Structured data, joins for aliases |
| **Cache** | Low-latency reads | Redis | Hot data, session state |

---

## Why Document DB fits well

- Social posts have **different shapes** per platform
- Schema can **evolve** (add fields later)
- Supports **flexible queries** (by time, sentiment, topic)
- Good for **nested data** (metrics, topics array)

---

## Data Schemas

### Mention Document
```json
{
  "mention_id": "m_001",
  "artist_id": "john_123",
  "platform": "twitter",
  "platform_post_id": "tw_123456",
  "created_at": "2024-10-15T12:37:10Z",
  "text": "Love the new album!",
  "sentiment_label": "positive",
  "sentiment_score": 0.85,
  "topic_ids": ["album_xyz"],
  "metrics": { "likes": 42, "retweets": 5 }
}
```

### Hourly Aggregate Document
```json
{
  "_id": "john_123#2024-10-15T12:00:00Z",
  "artist_id": "john_123",
  "bucket_start": "2024-10-15T12:00:00Z",
  "bucket_size": "hour",
  "pos_count": 70,
  "neu_count": 30,
  "neg_count": 20,
  "total_count": 120,
  "score_sum": 38.4
}
```

### Topic Aggregate Document
```json
{
  "_id": "john_123#album_xyz#2024-10-15T12:00:00Z",
  "artist_id": "john_123",
  "topic_id": "album_xyz",
  "bucket_start": "2024-10-15T12:00:00Z",
  "pos_count": 40,
  "neu_count": 10,
  "neg_count": 5,
  "total_count": 55,
  "score_sum": 22.5
}
```

---

## Key Concepts

### What is an "Hourly Bucket"?
Group mentions by the hour they occurred:
- Post at `12:37:10` → bucket `12:00:00`
- All posts from 12:00–12:59 go into the same bucket
- Each bucket = one point on the chart

### What does the Aggregator do?
Updates bucket counters as mentions arrive:
```
total_count += 1
pos/neu/neg_count += 1 (based on label)
score_sum += score
avg_score = score_sum / total_count
```

### What is Deduplication?
Social APIs can re-deliver the same post (retries, pagination, backfill). 
Dedupe ensures each post is counted only once using unique key: `(platform, platform_post_id)`.

### Bucketing for Long Time Ranges
For queries spanning years (2020–2025):
- Store **rollups** (daily/weekly/monthly)
- API picks bucket size automatically:
  - ≤ 2 days → hourly
  - ≤ 6 months → daily
  - ≤ 5 years → weekly

### Aggregates Store vs Mentions Store

| Store | What it holds | When used |
|-------|---------------|-----------|
| **Aggregates Store** | Precomputed hourly/daily summaries | Charts (fast reads) |
| **Mentions Store** | Individual posts with full details | Drill-down ("why?") |

**Analogy:**
- Aggregates = Monthly bank statement (totals)
- Mentions = Individual transactions (details)

---

# Step 5: Infrastructure (Scalability & Network)

## Network Zones

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET (Public)                                       │
│    [Users]              [Twitter API]      [Facebook API]      [TikTok API]         │
└───────┼──────────────────────┼──────────────────┼───────────────────┼───────────────┘
        │                      │                  │                   │
        ▼                      ▼                  ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              DMZ / Public Subnet                                     │
│    ┌─────────────┐      ┌─────────────────────────────────────────────┐             │
│    │   CDN       │      │            LOAD BALANCER                    │             │
│    │ (static UI) │      │  (routes /api/* to backend)                 │             │
│    └─────────────┘      └──────────────────┬──────────────────────────┘             │
└────────────────────────────────────────────┼────────────────────────────────────────┘
                                             │
┌────────────────────────────────────────────┼────────────────────────────────────────┐
│                              PRIVATE SUBNET (Application Tier)                       │
│                                            │                                         │
│    ┌───────────────────────────────────────┼───────────────────────────────────┐    │
│    │                         APPLICATION SERVICES                               │    │
│    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │    │
│    │  │ Query API   │  │ Enrichment  │  │ Sentiment   │  │   Topic     │       │    │
│    │  │ Service     │  │ Service     │  │  Service    │  │ Extraction  │       │    │
│    │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘       │    │
│    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                        │    │
│    │  │ Aggregation │  │  Artist     │  │  Collectors │──→ NAT Gateway         │    │
│    │  │ Service     │  │  Config     │  │ (outbound)  │    (to social APIs)    │    │
│    │  └─────────────┘  └─────────────┘  └─────────────┘                        │    │
│    └────────────────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────┬────────────────────────────────────────┘
                                             │
┌────────────────────────────────────────────┼────────────────────────────────────────┐
│                              PRIVATE SUBNET (Data Tier)                              │
│    ┌────────────────────────────────────────────────────────────────────────────┐    │
│    │  EVENT LOG (Kafka)    │  DATABASES (MongoDB)   │  CACHE (Redis)            │    │
│    │  [Broker 1,2,3]       │  [Primary + Replica]   │  [Cluster]                │    │
│    └────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Traffic Flows

### Flow 1: User → Dashboard (read path)
```
User → CDN (static UI) → Load Balancer → Query API → Cache/DB → Response
```

### Flow 2: Collectors → Event Log (ingestion)
```
Social APIs ← Collectors (via NAT) → Event Log
```

### Flow 3: Event Log → Processing (internal)
```
Event Log → Enrichment → Sentiment/Topic Services → Mentions Store
         → Aggregation → Aggregates Store
```

---

## Security Boundaries

| Zone | What lives here | Inbound access | Outbound access |
|------|-----------------|----------------|-----------------|
| **Internet** | Users, Social APIs | N/A | N/A |
| **DMZ / Public** | Load Balancer, CDN | From internet (HTTPS 443) | To private subnet |
| **Private (App)** | All services | From LB only | To data tier; collectors to internet via NAT |
| **Private (Data)** | DBs, Event Log, Cache | From app tier only | None |

---

## Performance Optimizations

| Technique | Where | Why |
|-----------|-------|-----|
| **Precomputed aggregates** | Aggregates Store | Charts load fast |
| **Rollups** (day/week/month) | Aggregates Store | Long ranges are fast |
| **Caching** | Query API → Redis | Reduce DB load |
| **Pagination** | Mentions API | Don't load 10K posts |
| **CDN** | Static UI | Fast asset delivery |

---

# Step 6: Back-of-Envelope Estimates (2-3 min)

## Step 1: State assumptions (30 sec)

| What | Value |
|------|-------|
| Artists | 1,000 |
| Mentions per artist per day | 1,000 |
| Retention | 2 years |

---

## Step 2: Quick math (1 min)

**Volume:**
```
1,000 artists × 1,000 mentions = 1M mentions/day
1M / 86,400 sec ≈ ~10-50 mentions/sec
```

**Storage:**
```
1 mention ≈ 1 KB
1M/day × 1 KB = 1 GB/day
2 years = ~750 GB ≈ 1 TB
```

---

## Step 3: One-liner conclusion (30 sec)

> "At 50 mentions/sec peak and ~1 TB storage, this is moderate scale. A single DB with replica handles it. We add cache for hot data and precompute aggregates for fast charts."

---

## Quick Reference Card

```
┌─────────────────────────────────────┐
│  QUICK ESTIMATES                    │
├─────────────────────────────────────┤
│  Mentions/day:   ~1M                │
│  Mentions/sec:   10-50              │
│  Storage:        ~1 TB (2 years)    │
│  Scale:          Moderate           │
│  DB:             1 primary + replica│
└─────────────────────────────────────┘
```

---

## How to adjust if interviewer gives different numbers

| If they say... | Adjust to... |
|----------------|--------------|
| 10K artists | 10M/day, 10 TB, need sharding |
| 10K mentions/artist/day | 10M/day, consider streaming |
| Real-time (seconds) | Add streaming layer |
| 10 years retention | Consider cold storage tier |

---

# Summary: Complete System Overview

## What We Built

| Layer | Components |
|-------|------------|
| **UI** | Dashboard with charts, filters, drill-down |
| **API** | REST endpoints for timeseries, drivers, mentions |
| **Services** | Collectors, Enrichment, Sentiment, Topic, Aggregation, Query API |
| **Patterns** | Microservices, Event-Driven, CQRS, Event Sourcing |
| **Event Log** | Append-only source of truth |
| **Storage** | Mentions Store (posts), Aggregates Store (charts), Topic Aggregates (drivers) |
| **Network** | Public LB → Private app tier → Isolated data tier |

---

## Design Principles Applied

1. **User-First** — designed from dashboard backward
2. **Single Responsibility** — each service does one thing
3. **Event Sourcing** — events are truth, aggregates are derived
4. **CQRS** — write path and read path optimized separately
5. **Scalability** — queue decoupling, horizontal scaling, precomputed aggregates
6. **Tool-Agnostic** — no vendor lock-in in the design

---

## UI → API → Storage Mapping

| UI Element | API Call | Storage |
|------------|----------|---------|
| Artist dropdown | `GET /artists?query=john` | Artist Config |
| Sentiment chart | `GET /artists/{id}/sentiment/timeseries` | **Aggregates Store** |
| Top drivers | `GET /artists/{id}/sentiment/drivers` | **Topic Aggregates** |
| Drill-down | `GET /artists/{id}/mentions` | **Mentions Store** |

---

## The One-Liner for the Whole System

> "Collectors ingest from social APIs into an event log. Processing enriches with sentiment and topics. Aggregation precomputes hourly buckets for fast charts. Query API serves the dashboard, which shows trends and lets users drill down to understand why sentiment changed."

---

## Interview Tips

### Do This
- Ask 2–3 clarifying questions first (freshness, scale, platforms)
- Sketch UI on whiteboard early — interviewers love visuals
- Justify technology choices briefly ("Document DB because schema varies by platform")
- Explain CQRS simply: "write path optimized for ingestion, read path optimized for queries"

### Avoid This
- Jumping straight to technologies without understanding requirements
- Over-engineering for scale you don't need
- Using specific vendor names without being able to explain alternatives
- Forgetting to tie everything back to user needs

---

*Generated from system design interview prep session*
