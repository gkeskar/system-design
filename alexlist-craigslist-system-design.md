# Alexlist System Design (Craigslist Clone)

**Prompt:** Design a Craigslist-like classifieds platform from scratch.

---

## Step 1: Gather Requirements

### Functional

Start by asking the interviewer clarifying questions:

- "Web app, mobile, or both?" — Both
- "How many users?" — ~100,000 users
- "What are the core features?" — From the interviewer's answers:

1. User auth (login / create account)
2. Ability to post listings
3. Dashboard for reading listings
4. Search
5. Tagging / categories for listings

### Non-Functional

- **Scale:** 100K users — not massive, but enough that naive queries won't fly forever
- **Traffic shape:** Read-heavy — far more people browsing and searching than posting
- **Availability:** Users expect search to always work. Slight staleness in results is fine
- **Latency:** Search results should feel fast

### Back-of-Envelope Estimates

Quick math to sanity-check our design decisions:

- 100K users, ~10K daily active
- ~50 searches/user/day = **~500K reads/day** (~6 QPS average)
- A few hundred new listings/day = **write-light**
- Clearly **read-heavy** — justifies caching and read replicas later
- Storage: 100K listings × ~5KB each ≈ 500MB — a single MySQL instance handles this easily

---

## Step 2: Start Simple — Client and Server

Don't overcomplicate it. Start with the basics:

```
Web Client → Server → Database
```

### Web Client

The web client supports four core actions:

- Login / create account
- Create listing
- Read listings (dashboard)
- Search

These are the user-facing features. Everything else — the server, database, infrastructure — exists to support them.

---

## Step 3: Choose a Data Store — MySQL

**What kind of DB? Relational or document?**

We're storing users and listings — structured data with clear relationships (a user *creates* a listing). The data has well-defined columns (title, price, category, location). We need to filter and query on multiple fields for search. A relational database fits naturally.

**Why MySQL?** Posting size isn't too big, the data is structured, and at 100K users we don't need anything exotic. MySQL handles this scale easily and gives us the ability to do JOINs, filtering, and indexing.

---

## Step 4: Design the Schema

### User Table

```
User {
    id: string           -- primary key
    email: string
    password_hash: string  -- bcrypt/argon2, NOT encryption
    username: string
    address: string      -- added later for location
    latitude: float
    longitude: float
    created_at: datetime
    updated_at: datetime
}
uniqueConstraint(email)
```

**Important distinction:** The field is `password_hash`, not "encrypted password." Encryption is reversible (decrypt to get the original); hashing is one-way (you can never recover the original password). When a user logs in, you hash their input and compare it to the stored hash. Interviewers care about this — use **bcrypt** or **argon2**, never plain text, never reversible encryption.

### Listing Table

```
Listing {
    id: string           -- primary key
    user_id: string      -- foreign key → User
    title: string
    description: TEXT
    category: string     -- for tagging/filtering
    price: decimal       -- needed for price_min/price_max search
    status: enum         -- 'open' or 'closed'
    address: string
    latitude: float
    longitude: float
    created_at: datetime
    updated_at: datetime
}
```

**Why `status` instead of a boolean?** Using an enum (`open`, `closed`, maybe `pending`, `flagged` later) is more extensible than a boolean `is_active`. Closed listings should not appear in search results.

**Why `price` is critical:** The search API supports `price_min` and `price_max` — if there's no `price` column on the listing, there's nothing to filter against. This is an easy inconsistency to miss.

**Why timestamps?** `created_at` lets you sort results by recency (newest listings first). `updated_at` lets you expire stale listings and helps with debugging.

### Image Table

A single `img_url` column on the Listing table limits you to one image per listing. Craigslist lets you upload multiple photos. Use a separate table:

```
Image {
    id: string           -- primary key
    listing_id: string   -- foreign key → Listing
    url: string          -- points to S3
    display_order: int   -- controls which image shows first
}
```

This is more normalized — you can add/remove images independently without touching the Listing row.

---

## Step 5: API Design

Every requirement maps to an endpoint. The **noun** from the requirement becomes the resource in the URL, the **action** maps to an HTTP verb.

### Users

- `POST /api/v1/users` — create account (registration)
- `POST /api/v1/auth/login` — sign in (validate credentials, return a token)

### Listings

- `POST /api/v1/listings` — create a listing
- `GET /api/v1/listings/:id` — get a specific listing
- `PUT /api/v1/listings/:id` — update a listing
- `DELETE /api/v1/listings/:id` — deactivate/delete a listing

### Search

- `GET /api/v1/search?keyword=puppies&price_min=50&price_max=200&max_distance=10&page=1&limit=20`

**Why URL params and not a JSON body?** GET requests should not have a body — it's technically allowed by HTTP but most clients and servers don't support it well. Use **query parameters** instead:

- `?` starts the params
- `&` chains multiple params
- `key=value` format

### Search Response

Don't return just a list of IDs — that forces the client to make N additional requests to fetch each listing's details (the N+1 problem). Return **listing summaries** directly:

```json
{
    "listings": [
        {
            "id": "abc123",
            "title": "Golden Retriever Puppy",
            "price": 150,
            "thumbnail_url": "https://s3.../images/puppies/1",
            "city": "San Francisco",
            "created_at": "2026-01-15T10:30:00Z"
        }
    ],
    "page": 1,
    "limit": 20,
    "total": 87
}
```

**Pagination** is essential — you can't return every matching listing in one response. Include `page`, `limit`, and `total` so the client can render page controls.

---

## Step 6: Search — The Hard Part

The search API looks simple, but the implementation has real challenges.

### Keyword Search

The naive approach is a SQL `LIKE` query:

```sql
SELECT * FROM listing
WHERE title LIKE '%puppies%' OR description LIKE '%puppies%'
```

**Problem:** `LIKE '%puppies%'` (leading wildcard) does a **full table scan** — no index can help. At 100K listings this is tolerable, but it's slow and doesn't scale.

**Better approach:** Use **MySQL FULLTEXT indexes** on `title` and `description`. This lets you do:

```sql
SELECT * FROM listing
WHERE MATCH(title, description) AGAINST('puppies')
```

FULLTEXT indexes are purpose-built for text search — they tokenize the text, build an inverted index, and can match quickly. At larger scale, you'd move to **Elasticsearch** for more advanced features (fuzzy matching, relevance scoring, synonyms), but FULLTEXT is a solid starting point.

### Geo Search (Distance Filtering)

The search supports `max_distance` in miles. Given the user's lat/long and a listing's lat/long, we need to compute distance.

**Naive approach:** Calculate the Haversine distance for every listing in the result set. This is O(n) per query — expensive.

**Better approach:** Use a **bounding box pre-filter** first:

1. Given the user's location and max distance, compute a lat/long bounding box
2. Filter listings with `WHERE latitude BETWEEN ? AND ? AND longitude BETWEEN ? AND ?` — this uses an index
3. Then compute exact distance only for listings inside the box

At even larger scale, use **MySQL spatial indexes** (or PostGIS if on PostgreSQL) for native geospatial queries.

### Putting It Together

A search query combines keyword, price, distance, and status filters:

```sql
SELECT l.*, i.url AS thumbnail
FROM listing l
LEFT JOIN image i ON l.id = i.listing_id AND i.display_order = 1
WHERE MATCH(l.title, l.description) AGAINST('puppies')
  AND l.price BETWEEN 50 AND 200
  AND l.latitude BETWEEN ? AND ?
  AND l.longitude BETWEEN ? AND ?
  AND l.status = 'open'
ORDER BY l.created_at DESC
LIMIT 20 OFFSET 0
```

---

## Step 7: How to Handle Images?

We don't store images in the database — that would be massive and slow. Instead, upload them to **Amazon S3** (Binary Large Object / BLOB storage) and store only the URL in the database.

**S3** is basically a giant file storage service in the cloud. Example paths:

```
s3://alexlist-images/puppies/1.jpg
s3://alexlist-images/puppies/2.jpg
s3://alexlist-images/kitties/1.jpg
```

The Image table stores the URL, and the client fetches the image directly from S3 (or a CDN in front of it) when rendering a listing.

**Why a CDN?** If every image request hits S3 directly, that's a lot of bandwidth and latency (especially for users far from the S3 region). A **CDN like CloudFront** caches images at edge locations worldwide — users get images from the nearest edge, not from S3 every time.

---

## Step 8: Infrastructure

### Server Fundamentals

A server is a computer:
- **Hard drive** → persistent storage
- **RAM** → in-memory storage (fast, volatile)
- **CPU** → processing

### Scaling with a Load Balancer

One server can handle our initial traffic, but for reliability and growth, we run multiple servers behind a **load balancer**:

```
                    ┌──────────┐
                    │  Client  │
                    └────┬─────┘
                         ↓
                 ┌───────────────┐
                 │ Load Balancer │
                 └───┬───┬───┬──┘
                     ↓   ↓   ↓
               ┌────┐ ┌────┐ ┌────┐
               │ S1 │ │ S2 │ │ S3 │
               └──┬─┘ └──┬─┘ └──┬─┘
                  │      │      │
          ┌───────┴──────┴──────┘
          ↓                ↓
    ┌──────────┐    ┌──────────┐
    │  MySQL   │    │    S3    │
    │  SQL DB  │    │  (BLOB)  │
    └──────────┘    └──────────┘
```

### VPC (Virtual Private Cloud)

The servers, database, and load balancer all live inside a **VPC** — a private network in the cloud. The database is not exposed to the internet; only the servers inside the VPC can talk to it. The load balancer is the only public-facing component.

---

## Step 9: Authentication

Users need to log in. The question is: **build or buy?**

### Build vs Buy

- **Build:** Implement your own auth (password hashing, session management, token generation). Full control, but more work and more security risk if you get it wrong.
- **Buy:** Use an identity provider like **Auth0**, **Firebase Auth**, or **AWS Cognito**. Handles login, signup, password reset, OAuth (Google/Facebook login), and token management out of the box.

**For an MVP:** Buy. Use an identity provider. It's faster to ship, and auth is a solved problem — you don't want to be the team that accidentally stores passwords in plain text.

**For the interview:** Show you know the concepts. After login, the server issues a **JWT (JSON Web Token)** that the client includes in every subsequent request (`Authorization: Bearer <token>`). The server validates the token on each request — this is how it knows who's making the request without requiring a username/password every time.

---

## Step 10: What's Missing — Scaling Considerations

At 100K users with read-heavy traffic, the design above works. But here's what you'd add as the system grows:

### Caching Layer (Redis)

Most users are browsing, not posting. Add **Redis** to cache:
- **Hot listings** — frequently viewed listings don't need to hit MySQL every time
- **Popular search results** — cache the top search queries and their results with a short TTL

### Read Replicas

MySQL supports **read replicas** — copies of the database that handle read queries. The primary handles writes, replicas handle reads. Since we're read-heavy, this scales the read side without changing the application much.

### CDN for Images

Put **CloudFront** (or any CDN) in front of S3 so images are served from edge locations near the user, not from S3 every time.

---

## Final Architecture

```
                         ┌──────────┐
                         │  Client  │
                         │ (Web +   │
                         │  Mobile) │
                         └────┬─────┘
                              ↓
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └──┬─────┬─────┬──┘
                       ↓     ↓     ↓
                   ┌────┐ ┌────┐ ┌────┐
           ┌───────│ S1 │ │ S2 │ │ S3 │───────┐
           │  VPC  └─┬──┘ └─┬──┘ └─┬──┘       │
           │         └──────┼──────┘           │
           │       ┌────────┼────────┐         │
           │       ↓        ↓        ↓         │
           │  ┌────────┐ ┌───────┐ ┌─────┐    │
           │  │ MySQL  │ │ Redis │ │ S3  │    │
           │  │Primary │ │(Cache)│ │(IMG)│    │
           │  │+ Replica│ └───────┘ └──┬──┘    │
           │  └────────┘               ↓       │
           │                      ┌────────┐   │
           │                      │  CDN   │   │
           │                      └────────┘   │
           └───────────────────────────────────┘
```

**How it all flows:**

1. **Client** sends requests through the **Load Balancer**, which distributes traffic to servers inside the **VPC**
2. **MySQL** stores core data — users, listings, images metadata
3. **Redis** caches hot listings and popular search results for fast reads
4. **S3** stores listing images; **CDN** serves them from edge locations
5. Servers are stateless — any server can handle any request (JWT carries auth state)
6. MySQL **read replicas** handle search queries; primary handles writes

---

## Notes for Self

- `password_hash` not "encrypted password" — hashing is one-way, encryption is reversible
- Don't forget `price` on the Listing table if search filters by price
- Multiple images per listing → separate Image table, not a single column
- `LIKE '%keyword%'` = full table scan. Use FULLTEXT indexes or Elasticsearch
- Geo search: bounding box pre-filter → spatial index → exact distance calc
- Return listing summaries in search response, not just IDs (avoid N+1)
- Pagination: always include `page`, `limit`, `total`
- Read-heavy workload → caching (Redis) + read replicas
- CDN in front of S3 for images
- Auth: build vs buy → buy for MVP (Auth0, Cognito), know JWT flow for the interview
- Back-of-envelope math: 100K users, ~6 QPS reads, write-light, ~500MB storage
