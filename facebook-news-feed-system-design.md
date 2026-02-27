# Facebook News Feed System Design

**Prompt:** Design how Facebook works under the hood. Imagine we are building it from scratch.

---

## Step 1: Gather Requirements

### Functional

- Create user account
- Create posts
- Follow other users
- See the feed (posts from people you follow)

---

## Step 2: Initial Schema Thinking

Before choosing a database, sketch out what data we need to store:

```
User: id, name, email
Followers: follower_id, following_id
Post: id, user_id, created_at
```

The basic flow is: `DB ← Server ← Client`. Creating a user is a POST request. Now we need to talk about what technologies we're building with.

---

## Step 3: Choose a Data Store — MySQL

**What kind of DB? Relational or document?**

We go with a **relational database (MySQL)** because:
- We need to store user and post data in a **structured way**
- We need to be able to **query the data easily** (joins between users, posts, followers)

### Schema

```
User {
    id
    name
    email
}

Post {
    id
    user_id
    content
}
```

---

## Step 4: Scaling — Load Balancer

As users grow, we need two things:
- **Cache** the posts (so we're not hitting the DB every time)
- **Load balancer** to distribute the traffic across multiple servers

```
Client → Load Balancer → Server 1
                       → Server 2
                            ↓
                          MySQL
```

---

## Step 5: The Feed Problem

**How are we retrieving the feed?**

First, try to get it from the **cache**. If not cached, we query the DB. That query is a **JOIN between the post and followers table**:

```sql
SELECT * FROM followers f
INNER JOIN post p ON f.following_id = p.user_id
WHERE f.follower_id = 1
ORDER BY p.created_at DESC
LIMIT 10
```

This gives us the latest 10 posts from the users we are following. We can add **pagination** and **filters** to this query. It returns a `user_id, post_id` list.

But running this join on every page load is expensive. We need to cache it.

---

## Step 6: MongoDB for Feed Cache

We use **MongoDB** as a document DB for caching because it's fast and scalable. We can store the feed as a structured blob:

```json
{
    "post_id": 1,
    "post_content": "Hello, how are you?",
    "post_created_at": "2026-02-25 10:00:00"
}
```

**Two benefits of caching:**
1. **Close to client** — faster reads, less network hops
2. **Computation/transformation done ahead of time** — the heavy JOIN is already done, the feed is pre-built and ready to serve

MongoDB stores the **Feeds** — pre-built feed documents per user.

---

## Step 7: How Does Data Get Into MongoDB? — Fan-Out on Write

When I make a post:

1. Request goes to the **server**
2. Server writes to **MySQL** (the source of truth)
3. Server writes **one message to the queue** (avoids duplicate writes)
4. The **job queue fans out** — writes the post into each follower's feed document in MongoDB

**Why the queue does the fan out, not the server?** If a lot of people are following you, the server can't write to all their feeds directly — that would block the response. Instead, the server writes once to the queue, and the queue handles distributing it to all followers asynchronously.

**For reading:** Read from MongoDB cache first. If not found in cache, fall back to querying MySQL.

```
Dave opens app
       ↓
  Check MongoDB → found? → return feed (fast)
       ↓ (not found)
  Query MySQL (expensive JOIN)
       ↓
  Return feed to Dave + save to MongoDB for next time
```

This is **eventual consistency** — feeds aren't updated instantly, but fast enough that users don't notice.

**What does that mean exactly?** When Alice posts at 10:00:00, her post is saved to MySQL immediately. But Bob's feed in MongoDB isn't updated until the queue worker gets to it — maybe 300 milliseconds later. In that tiny window, Bob's feed is **stale**. If he refreshes at exactly that moment, he won't see Alice's post yet. But 300ms later, he will. It *will* be consistent — just not right this second.

**The opposite — strong consistency** — would mean Alice's post doesn't succeed until ALL followers' feeds are updated. If she has a million followers, she could wait minutes before seeing "Posted!" Terrible experience.

**Why eventual consistency is fine here:** Nobody refreshes their feed and thinks "this post should have appeared 200 milliseconds ago." A half-second delay is invisible to humans. Social media feeds don't need to be real-time to the millisecond.

**Where you'd NOT want eventual consistency:** A bank transfer. If you send $500, the recipient needs to see it immediately — not "eventually." That needs strong consistency.

**Rule of thumb:** If a slight delay doesn't hurt anyone → eventual consistency is fine. If a delay could cause real problems (money, safety) → use strong consistency.

### Concrete Example: Fan-Out in Action

Say we have 5 users: Alice, Bob, Carol, Dave, and Eve.
- Bob follows Alice
- Carol follows Alice
- Dave follows Alice and Bob
- Eve follows Bob

MongoDB has **one document per user** — their personal feed. Think of it as a pre-built inbox. Before anyone posts, it's empty:

```json
{ "userId": "Bob",   "posts": [] }
{ "userId": "Carol", "posts": [] }
{ "userId": "Dave",  "posts": [] }
{ "userId": "Eve",   "posts": [] }
```

**Alice posts: "Hey everyone!"**

1. Server saves to MySQL
2. Queue gets: `"Alice posted, post_id: 101"`
3. Who follows Alice? → Bob, Carol, Dave
4. Fan out — write Alice's post into each of their feeds:

```json
{ "userId": "Bob",   "posts": [
    { "post_id": 101, "author": "Alice", "content": "Hey everyone!" }
]}
{ "userId": "Carol", "posts": [
    { "post_id": 101, "author": "Alice", "content": "Hey everyone!" }
]}
{ "userId": "Dave",  "posts": [
    { "post_id": 101, "author": "Alice", "content": "Hey everyone!" }
]}
{ "userId": "Eve",   "posts": [] }   ← still empty, doesn't follow Alice
```

**Now Bob posts: "Good morning!"**

Queue fans out to Dave and Eve (Bob's followers):

```json
{ "userId": "Dave",  "posts": [
    { "post_id": 102, "author": "Bob",   "content": "Good morning!" },
    { "post_id": 101, "author": "Alice", "content": "Hey everyone!" }
]}
{ "userId": "Eve",   "posts": [
    { "post_id": 102, "author": "Bob",   "content": "Good morning!" }
]}
```

Dave follows both Alice and Bob, so his feed has both posts. Eve only follows Bob, so she only has Bob's post.

**When Dave opens the app:** Server just does `db.feeds.findOne({ userId: "Dave" })` — grabs that one document, returns it. No JOINs, no aggregation. The feed is already built and waiting.

**At 1,000 users:** MongoDB has 1,000 documents — one per user. Each document is that user's personalized feed (posts from everyone they follow, capped at the latest 100-200 posts). When any user opens the app, grab their one document. The "fan out" is the write side — every time someone posts, the queue writes that post into every follower's document. The more followers, the more documents get updated. That's why the queue handles it asynchronously.

---

## Step 8: How to Handle Likes?

### Where do we record likes?

**First thought:** Just use Redis as a persistent datastore for likes.

**Problem with just Redis:** If you're querying Redis for analytics, it will slow down. Redis can be persistent, but it's not efficient for analytics queries.

**Second thought:** Store `number_of_likes` on the posts table in MySQL.

**Problem with that:** If you just store a count, you don't know *who* liked it. And you can't prevent a user from liking the same post multiple times — you have no record of individual likes.

### Solution: A Likes Table in MySQL

```
Like {
    id
    post_id
    user_id
    created_at
}
```

Now we can query anything we need:

```sql
-- Get the number of likes for a post
SELECT COUNT(*) FROM like WHERE post_id = 1

-- Get the users who liked a post
SELECT user_id FROM like WHERE post_id = 1

-- Get the posts liked by a user
SELECT post_id FROM like WHERE user_id = 1
```

### Caching Like Counts in Redis

Querying `COUNT(*)` on every post render is expensive. So we store the **like count in Redis** for fast reads: `post_id → like_count`.

The server fetches like counts from Redis, not MySQL.

**One subtlety:** If you build the feed (in MongoDB) before likes are counted, the likes won't be included in the feed document. That's why like counts live in Redis separately — the feed and the like counts are served independently.

### Real-Time Updates

If you want real-time like counts updating live on the page, use **WebSockets**.

**Without real-time (polling):** Your app has no idea someone just liked the post. The counter still says 42. The only way to see 43 is if you refresh — or the app keeps asking the server every few seconds:

```
App: "Any updates?" → Server: "Nope"
App: "Any updates?" → Server: "Nope"
App: "Any updates?" → Server: "Yes, post 101 now has 43 likes"
```

This is **polling**. It works, but it's wasteful — 90% of those requests return nothing. Multiply by a million users and your server is drowning in useless requests.

**With real-time (WebSockets):** When you open the app, your phone opens a regular HTTP connection and asks to "upgrade" it to a WebSocket. The server agrees. Now there's a **persistent, open connection** — like a phone call. Both sides can talk whenever they want:

```
App: "I'm looking at post 101"
       ... connection stays open ...
Server: "post 101 now has 43 likes"        ← server pushes this on its own
Server: "post 101 now has 44 likes"        ← and this
Server: "new post in your feed from Alice" ← and this
```

The app never has to ask. The server **pushes** updates the moment they happen.

**The handshake — how the connection upgrades:**

1. App sends a regular HTTP request with header: `Upgrade: websocket`
2. Server responds: `101 Switching Protocols`
3. The connection is now "upgraded" from HTTP (one request, one response, done) to WebSocket (persistent, bidirectional)
4. Server keeps this connection alive and pushes any relevant updates through it

**What triggers the server to push?** When someone likes a post, the write goes to Redis (like count) and MySQL (like record). The server also tells the **socket component**: "post 101 got a new like." The socket component checks which users currently have post 101 on their screen (they're subscribed via their open WebSocket) and pushes the update to them.

**Why it's a separate component:** The socket component is dedicated to managing thousands of open connections. Regular API servers handle request-response (stateless). WebSocket connections are **stateful** — they stay open. Mixing the two on the same server would be messy, so it's a separate component.

**When to use which:**

| | WebSockets | Polling |
|---|---|---|
| Use when | Updates need to feel instant (live likes, chat, notifications) | Updates can be slightly delayed (email inbox, dashboard) |
| Connection | Persistent, always open | New request every few seconds |
| Server load | Lower (only sends when there's something) | Higher (constant "any updates?" requests) |
| Complexity | More complex to implement | Simple to implement |

---

## Final Architecture

```
                         ┌──────────┐
                         │  Client  │
                         └────┬─────┘
                              ↓
                      ┌───────────────┐
                      │ Load Balancer │
                      └───┬───────┬───┘
                          ↓       ↓
                    ┌────────┐ ┌────────┐
                    │Server 1│ │Server 2│
                    └──┬──┬──┘ └──┬──┬──┘
                       │  │       │  │
          ┌────────────┘  │       │  └────────────┐
          ↓               ↓       ↓               ↓
    ┌──────────┐    ┌──────────┐    ┌─────────┐
    │  MySQL   │    │  Queue   │    │  Redis  │
    │(Primary +│    │ (Fan out)│    │ (Likes) │
    │ Replica) │    └────┬─────┘    └─────────┘
    └──────────┘         ↓
                   ┌──────────┐
                   │ MongoDB  │
                   │ (Feeds)  │
                   └──────────┘
```

**How it all flows:**

1. **Client** sends requests through the **Load Balancer**, which distributes them to available **Servers**
2. **MySQL** stores core data — users, posts, follows, likes
3. When a post is created, server writes to MySQL and puts **one message in the Queue**
4. The Queue **fans out** — workers write the post into each follower's feed in **MongoDB**
5. For reading, server grabs the pre-built feed from **MongoDB** (falls back to MySQL if not cached)
6. **Redis** caches like counts — the feed and likes are served independently
7. **WebSockets** for real-time updates if needed

---

## Notes for Self

- Read from cache first, fall back to DB
- Server writes once to queue — queue does the fan out (avoids duplicate writes)
- Likes are tricky — need both a Like table (to know *who* liked) and Redis (for fast count reads)
- Redis is not good for analytics — keep that in MySQL
- Feed and like counts are served independently (feed from MongoDB, likes from Redis)
- WebSockets for real-time
- Read developer blogs for each technology (MongoDB, Redis, etc.) to understand best practices

### Why MongoDB for feeds and Redis for likes — not one for both?

| | Feed Cache | Like Counts |
|---|---|---|
| Data shape | Large JSON document | Tiny key-value pair |
| Size per entry | Big (hundreds of posts) | Small (one number) |
| Best fit | MongoDB (document store) | Redis (in-memory key-value) |
| Persistence | Native in MongoDB | Not Redis's strength |
| Access pattern | Read/write whole blob | Read/increment a counter |

- **MongoDB for feeds:** The feed is a large, nested document per user — that's what document DBs are built for. We need persistence (if cache goes down, we don't want to rebuild every feed via expensive JOINs). Storing big blobs in Redis would eat up RAM fast, and memory is expensive.
- **Redis for likes:** A like count is tiny (`post_id → 42`), read on every single post render, needs sub-millisecond speed. Just a simple key-value lookup — perfect for Redis.
- They solve **different problems**. Using one for both would mean either paying too much for memory (Redis for feeds) or being slower than needed (MongoDB for counters).
