# System Design Interview Prep

A collection of system design solutions for interview preparation.

## Designs

| Design | Description |
|--------|-------------|
| [Artist Sentiment Dashboard](artist-sentiment-dashboard-design.md) | Real-time sentiment analytics tool for a talent agency — ingests social media data (Twitter, Facebook, TikTok), computes sentiment, and visualizes trends over time with drill-down explanations. |
| [Twitter System Design](twitter-system-design.md) | Step-by-step design of a Twitter-like platform — covers requirements gathering, API design, PostgreSQL schema, MongoDB timeline cache, fan-out on write with job queues, S3 for media, likes caching, and WebSockets for real-time updates. |
| [Facebook News Feed System Design](facebook-news-feed-system-design.md) | Step-by-step design of a Facebook-like news feed — covers MySQL with read replicas, load balancing, MongoDB for pre-built feed documents, fan-out on write, Redis for like count caching, and WebSockets for real-time updates. |
| [Alexlist / Craigslist System Design](alexlist-craigslist-system-design.md) | Step-by-step design of a Craigslist-like classifieds platform — covers MySQL schema, REST API design with URL params, full-text and geo search, S3 for images with CDN, pagination, auth (build vs buy), and back-of-envelope estimates. |

## Approach

Each design follows this structure:

1. **User Perspective (UI First)** — what does the user need?
2. **API Design** — REST endpoints derived from UI
3. **Services & Patterns** — microservices, event-driven, CQRS, event sourcing
4. **Technology Choices** — databases, event log, cache
5. **Infrastructure** — network, scaling, performance
6. **Back-of-Envelope Estimates** — quick math to justify decisions
