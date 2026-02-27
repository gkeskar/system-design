# Twitter System Design

## Step 1: Gather Requirements

We're building a mini Twitter. Users can sign up, write tweets, follow people, see a feed of tweets from people they follow, like tweets, and attach images/videos. It's for about a thousand active users. More people will be reading tweets than writing them.

### Functional

You don't need to know the product well. The requirements come from **asking the interviewer questions**, not from personal knowledge. The interviewer gives you a vague prompt like "Design Twitter" — your job is to ask clarifying questions to narrow the scope. Think of it like being a contractor — before you build a house, you ask the client what they want.

**Questions you'd ask:**

- "What are the core features we need to support?" — posting tweets, following users, a home timeline
- "Do we need to support media (images, videos)?"
- "Do we need real-time updates or is polling okay?"
- "What's the expected scale? Thousands of users? Millions?"
- "Is read traffic higher than write traffic?"
- "Do we need to support search?"

**The pattern is always the same regardless of the product:**

1. **Who** are the users? (mobile, web, both?)
2. **What** can they do? (the core actions — create, read, update, delete *what*?)
3. **How many** users? (scale)
4. **What matters most?** (speed? reliability? consistency?)

From asking these questions, we gathered:

- CRUD tweets
- Allow users to create profiles, edit, delete
- Follow each other
- Timeline for a user based on who they follow
- Timeline of a user

### Non-Functional / Performance

Same approach — ask the interviewer, but now the questions are about **how the system should behave** rather than what it does.

**Questions you'd ask:**

- "How many active users are we expecting?" — A couple thousand vs a hundred million are completely different designs.
- "Is read traffic heavier than write traffic?" — Most social media apps are read-heavy. This tells you where to optimize (caching, read replicas, etc.)
- "How important is availability vs consistency?" — Should the system always be up (even if data is slightly stale) or should every read return the latest data?
- "Are users distributed globally or concentrated in one region?" — This affects whether you need CDNs, multiple data centers, or edge servers.
- "What clients need to be supported?" — Mobile, browser, both? This affects API design and payload sizes.
- "Is there a latency requirement?" — Should timelines load in under 200ms? Under a second?

**The pattern for non-functional requirements always boils down to:**

1. **Scale** — how many users, how much data?
2. **Traffic shape** — read-heavy or write-heavy?
3. **Availability vs Consistency** — can we tolerate stale data?
4. **Latency** — how fast does it need to be?
5. **Reliability** — what happens if something fails?
6. **Geography** — where are the users?

These questions work for any system — Twitter, Uber, Netflix, a chat app, anything. The answers just change the design decisions you make.

The requirements aren't things you come up with on your own — they're the interviewer's answers rephrased as bullet points. You ask the question, they give you the constraint, you write it down:

| Question Asked | Interviewer's Answer | Requirement |
|---|---|---|
| "How many active users?" | "A couple thousand" | Active users: couple thousand |
| "Is read traffic heavier than write?" | "Yes, reads are much higher, but writes are significant too" | High read traffic (higher than write) + High write traffic |
| "How important is availability?" | "Very — users expect it to work when they open it" | Reliability |
| "Are users in one region or distributed?" | "Global" | Global distribution of user base |
| "What clients?" | "Both mobile and web" | Mobile and browser clients |

---

## Step 2: API Design

Before jumping into the data store, define the API endpoints first. This translates the requirements into concrete operations — it's the contract between client and server. Once you know the endpoints, you know what data they need, which tells you what to store and how to structure it.

### How to come up with endpoints

You don't need to memorize endpoints. Every requirement follows the pattern: *"A user can [action] a [thing]"*. The **thing** becomes the resource (the noun in the URL), and the **action** maps to an HTTP verb:

| Action | HTTP Verb | Example |
|---|---|---|
| Create something | POST | `POST /tweets` |
| Read/Get something | GET | `GET /tweets/:id` |
| Update something | PUT | `PUT /tweets/:id` |
| Delete something | DELETE | `DELETE /tweets/:id` |

The pattern is:

1. Pick the **noun** from the requirement — that's your resource (`/tweets`, `/users`)
2. Pick the **verb** — that maps to the HTTP method (create = POST, read = GET, update = PUT, remove = DELETE)
3. If one resource acts on another, **nest it** — "user follows user" → `/users/:id/follow`, "user likes tweet" → `/tweets/:id/like`

### Tying requirements to endpoints

**Tweets**

- `POST /tweets` — create a new tweet
- `GET /tweets/:id` — get a specific tweet
- `PUT /tweets/:id` — edit a tweet
- `DELETE /tweets/:id` — delete a tweet

**Users**

- `POST /users` — create a profile (sign up)
- `GET /users/:id` — get a user's profile
- `PUT /users/:id` — edit profile
- `DELETE /users/:id` — delete profile

**Follow**

- `POST /users/:id/follow` — follow a user
- `DELETE /users/:id/follow` — unfollow a user

> The requirement says "follow each other" — it doesn't say "list all followers." Don't add endpoints the requirements don't ask for. If the interviewer wants it, they'll bring it up — then you'd add `GET /users/:id/followers?page=1&limit=20` with **pagination**, because returning a celebrity's 50 million followers in one response is a terrible idea.

**Timeline**

- `GET /timeline` — get the authenticated user's home timeline (tweets from people they follow)
- `GET /users/:id/tweets` — get a specific user's tweets

> Why just `GET /timeline` and not `GET /users/:id/timeline`? The user isn't creating, updating, or deleting anything — they're just **reading** a feed, so it's a GET. And the server already knows *who you are* from the authentication token in the request. Every user only ever sees their own home timeline, so you don't need the user ID in the URL. Think of it like opening your email inbox — you don't type your email address into the URL, you're already logged in and the app knows which inbox to show you.

**Likes**

- `POST /tweets/:id/like` — like a tweet
- `DELETE /tweets/:id/like` — unlike a tweet
- `GET /tweets/:id/likes` — get like count

Notice how every functional requirement maps to an endpoint. "CRUD tweets" gave us the four tweet endpoints. "Follow each other" gave us the follow/unfollow endpoints. "Timeline" gave us the timeline endpoint. This is the bridge between vague requirements and concrete system behavior.

Now that we know the APIs, we can see what data we need to support them — which leads us to the data store.

---

## Step 3: Choose a Data Store

We need a database. Two options — a document database (like MongoDB, stores data as big JSON blobs) or a relational database (like PostgreSQL, stores data in structured tables with rows and columns).

What are we actually storing? Tweets and users — structured data with clear relationships. A user *creates* a tweet, a user *follows* another user, a user *likes* a tweet. A relational database fits naturally.

Since this is scoped to ~1,000 users (not a massive scale), rather than having separate data stores for tweets and users, we can use **one store for both**. Keep it simple.

**Why relational (PostgreSQL)?** Data normalization is an advantage of relational databases. This is a trade-off of **time complexity vs space complexity** — normalized data avoids duplication (saves space) but may require joins (costs time). We accept that trade-off and go with **PostgreSQL**.

---

## Step 4: Design the Schema

Think of tables like spreadsheets. Start with the core tables:

**Tweet** — one row per tweet (who wrote it, the text, when it was posted)

```
Tweet {
    id: string
    createdBy: string
    content: TEXT
    createdAt: Date
}
```

**User** — one row per user (name, email, username)

```
User {
    id: string
    firstName: string
    lastName: string
    userName: string
    email: string
}
uniqueConstraint(email)
```

**Following — how do we represent users following each other?**

Use a join table — just two columns: followee and follower. The composite key here is really important — the combination of both is the unique key, so you can't follow the same person twice.

```
Follow {
    followeeId: string
    followerId: string
}
Primary Key: (followeeId, followerId)
```

---

## Step 5: Start with a Simple Architecture

Don't overcomplicate it. Start with a **single service** first. Only break into microservices when it makes sense.

```
Client (Browser + Mobile) → Server → PostgreSQL
```

One server, one database. This is enough to get CRUD tweets, user profiles, and follow relationships working. It works.

---

## Step 6: The Timeline Problem

Here's where it gets interesting. When you open Twitter, you see a feed of tweets from everyone you follow. How does that work?

The naive approach: every time you open the app, the server goes to PostgreSQL and says "find all the people this user follows, then find all their recent tweets, sort them by time, return the top 100." That's a lot of work — especially if you follow 500 people. Doing this on every single page load is slow.

**Solution: pre-build the timeline and cache it.**

Think of it like a restaurant. Instead of cooking every meal from scratch when a customer orders (reading from the database every time), you prep meals in advance and keep them warm (cache).

The timeline can initially be stored **in memory**. But as it grows, we need to persist it somewhere. What type of cache?

Options like Redis or Memcached come to mind, but the timeline is really a **massive JSON document** — the first 100 tweets from a GET request. This is a perfect fit for a **document-based caching layer** like **MongoDB** or **DynamoDB**.

### Why MongoDB over PostgreSQL for the cache?

The access pattern is simple — we're just writing the whole document (when a followed user tweets) and reading the whole document (when the user opens their feed). We're not querying individual fields, joining tables, or doing relational operations on it. That's exactly what a document database is built for — store a document, retrieve a document. It doesn't care about relationships between tables, it just gives you your blob back fast.

If we used PostgreSQL instead, we'd either store the timeline as a JSON column (PostgreSQL supports JSON but isn't optimized for it) or reconstruct it from relational tables every time (which is the expensive query we're trying to avoid).

### Why MongoDB over Redis/Memcached?

Redis and Memcached are pure in-memory caches — faster, but if they restart, every user's pre-built timeline is gone and has to be rebuilt from scratch. MongoDB writes to disk, so we get **persistence** — the cache survives a restart.

### Timeline cache structure

For each user, we store a ready-to-go JSON document with their latest tweets already assembled. When they open the app, we just grab that document — super fast, no heavy database queries.

```json
{
    "id": "userId (e.g., Robby)",
    "tweets": [
        {
            "id": "tweet-id",
            "createdBy": "Cy",
            "content": "Cy's tweet",
            "createdAt": "2026-01-01T00:00:00Z"
        }
    ]
}
```

### But how does the cache get updated?

Say someone I follow tweets — how does that get into my timeline? The tweet is stored in PostgreSQL, but the timeline cache also needs to be updated.

Doing this synchronously won't work — if a popular user tweets and has millions of followers, updating each follower's timeline one-by-one before the tweet "goes live" would take forever.

**Solution: Fan-out on write.** When a user tweets:

1. The tweet is stored in PostgreSQL
2. A message is dropped in a **job queue, prioritized based on popularity**
3. Background workers pick up that message and update each follower's cached timeline in MongoDB, one by one, asynchronously

Is every follower's timeline updated instantly? No. But it's fast enough — a few seconds delay is totally fine for a timeline. This is called **eventual consistency**.

The job queue can scale by adjusting the **number of workers**. If the queue gets backed up, add more workers.

### Cache strategy

We're using **cache on write** — update the cache when a tweet is created, not when a user reads their timeline. LRU eviction doesn't make sense here since entries are proactively written, not driven by read frequency.

---

## Step 7: How to Handle Media?

Tweets can have images and videos. We don't store them in the database — that would be massive and slow. Instead, we upload them to **Amazon S3** (basically a giant file storage service in the cloud) and just save the URL in the tweet record. When the app needs to show the image, it fetches it from S3 using that URL.

This means we need to add `mediaUrl` to the Tweet schema:

```
Tweet {
    id: string
    createdBy: string
    content: TEXT
    mediaUrl: string    ← added
    createdAt: Date
}
```

---

## Step 8: How to Represent Likes?

Add a Likes table in PostgreSQL — one row per like (which tweet, which user). The combination is unique so you can't like the same tweet twice.

```
Like {
    tweetId: string
    userId: string
}
uniqueConstraint(tweetId, userId)
```

**But this alone won't work at scale** — every time someone views a tweet, we'd have to count all the rows in the Likes table for that tweet. That's expensive on every load.

We can cache like counts. Since we're already using MongoDB as the timeline cache, we reuse it for likes too — no need for a separate cache at this scale (~1,000 users).

- Store a simple document per tweet: `{ tweetId: "abc", likeCount: 42 }` — fast to read
- MongoDB provides fast enough reads at this scale
- If this becomes a bottleneck later, Redis can be introduced (sub-millisecond, purpose-built for counters)

**Write path:** We don't update the count on every single like — that would be a lot of tiny writes. Instead, we place a queue in front of writes to both MongoDB and PostgreSQL and **batch updates** (e.g., collect 5 or 10 likes, then flush them in one go). This reduces the write pressure significantly.

---

## Step 9: Real-Time Updates

Normally, the web works like this: the client asks the server for data, the server responds. The client has to keep asking ("any new likes? any new likes?"). This is called polling and it's wasteful.

**WebSockets** flip this around. The client opens a connection and keeps it open. Now the server can push updates to the client whenever something happens — a new like, a new tweet in your timeline — without the client having to ask. Think of it like a phone call vs sending letters back and forth.

The connection starts as a regular HTTP request, then "upgrades" to a WebSocket — that's the handshake. This becomes a dedicated **socket component** connected to the server.

---

## Step 10: Add an API Gateway

As things grow, you don't want clients talking directly to your server. You put an **API gateway** in front — think of it like a receptionist at a building. It handles:

- **Routing** requests to the right place
- **Rate limiting** (stopping someone from spamming your server)
- **Authentication** (making sure the person is who they say they are)

---

## Final Architecture

Putting it all together: the phone talks to the API Gateway, which talks to the server. The server reads/writes to PostgreSQL for core data, uses MongoDB to serve fast timelines and like counts, uploads media to S3, pushes real-time updates through WebSockets, and uses a job queue with workers to keep timelines updated in the background.

We started with just a box and a database, and built our way here one problem at a time.

```
Client (Browser + Mobile)
        ↓
   API Gateway
        ↓
      Server ──→ Object Store (S3)
     ↓    ↓   ↘
  Postgres  Queue   Socket (WebSocket)
   SQL       ↓
         Job Queue (prioritized based on popularity)
           ↓            ↓
     MongoDB         Can set number
   (Timeline +        of workers
    Like Cache)
```

---

## Notes for Self

- Timeline can get slow as things grow
- Cache on write vs read — LRU doesn't make sense for cache on write
- Handshake: HTTP connection upgraded to a socket connection
- Start simple, split later
