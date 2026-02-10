# System Design Interview Prep

A collection of system design solutions for interview preparation.

## Designs

| Design | Description |
|--------|-------------|
| [Artist Sentiment Dashboard](artist-sentiment-dashboard-design.md) | Real-time sentiment analytics tool for a talent agency — ingests social media data (Twitter, Facebook, TikTok), computes sentiment, and visualizes trends over time with drill-down explanations. |

## Approach

Each design follows this structure:

1. **User Perspective (UI First)** — what does the user need?
2. **API Design** — REST endpoints derived from UI
3. **Services & Patterns** — microservices, event-driven, CQRS, event sourcing
4. **Technology Choices** — databases, event log, cache
5. **Infrastructure** — network, scaling, performance
6. **Back-of-Envelope Estimates** — quick math to justify decisions
