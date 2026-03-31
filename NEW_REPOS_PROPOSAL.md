# New Repository Proposals — HackerRank Coding Assessments

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Pattern Coverage Strategy](#2-pattern-coverage-strategy)
3. [Repo 1 — LinkUp (LinkedIn Clone / GraphQL)](#3-repo-1--linkup-linkedin-clone--graphql)
4. [Repo 2 — MatchDay (ESPN Clone / WebSocket)](#4-repo-2--matchday-espn-clone--websocket)
5. [Repo 3 — ShopCart (Amazon Clone / Microservices + Event-Driven)](#5-repo-3--shopcart-amazon-clone--microservices--event-driven)
6. [Repo 4 — BookIt (Calendly Clone / REST + Webhooks + CRON)](#6-repo-4--bookit-calendly-clone--rest--webhooks--cron)
---

## 1. Executive Summary

### What's Already Covered (Existing Repos)

| Repo | Domain | Pattern |
|---|---|---|
| ShowPass | Event ticketing (Ticketmaster) | REST + Monolith |
| DoorDash/Zomato | Food delivery | REST + Monolith |
| IMDB | Movie ratings & watchlist | REST + Monolith |
| GitHub/Linear | Dev tools / project management | REST + Monolith |
| Spotify | Music streaming | REST + Monolith |

**Gap:** All existing repos use REST + Monolith. No coverage of GraphQL, WebSockets, Microservices, Event-Driven, CQRS, Event Sourcing, Webhooks, CRON, or Saga patterns.

### What's New

| Repo | Clone Of | Primary Pattern | Secondary Patterns |
|---|---|---|---|
| **LinkUp** | LinkedIn | GraphQL | Subscriptions, DataLoader, Caching |
| **MatchDay** | ESPN | WebSocket | Pub/Sub, Rooms/Channels |
| **ShopCart** | Amazon | Microservices + Event-Driven | Webhooks, API Gateway |
| **BookIt** | Calendly | REST + Webhooks | CRON/Scheduled Jobs |

---

## 2. Pattern Coverage Strategy

### 2.1 Primary Communication Patterns

These define **how clients and servers talk to each other**. These are the core patterns we will use in the repos.

| Pattern | What It Does | Used In (Real World) | Our Coverage |
|---|---|---|---|
| **REST** | Request-response over HTTP | Stripe, Twitter, 90% of public APIs | Already covered (ShowPass) + BookIt, ShopCart |
| **GraphQL** | Client queries exactly what it needs; flexible, nested | GitHub API, Shopify, Facebook | **LinkUp** |
| **WebSocket** | Persistent 2-way connection; server pushes in real-time | Slack, Discord, trading platforms | **MatchDay** |
| **Webhooks** | Server calls another server's URL when an event happens | Stripe payment events, GitHub events | **ShopCart** (payment events) + **BookIt** (booking events) |

### 2.2 Primary Architectural Patterns

These define **how the system is structured internally**.

| Pattern | What It Does | Used In (Real World) | Our Coverage |
|---|---|---|---|
| **Monolith** | Everything in one app | Most startups, ShowPass | Already covered |
| **Microservices** | App split into independent services | Netflix, Uber, Amazon | **ShopCart** |
| **Event-Driven (Pub/Sub)** | Services publish events; others subscribe and react | Order systems, Kafka pipelines | **ShopCart** |
| **API Gateway** | Single entry point routing to multiple backend services | Kong, AWS API Gateway | **ShopCart** |

### 2.3 Operational Patterns

| Pattern | What It Does | Used In (Real World) | Our Coverage |
|---|---|---|---|
| **CRON / Scheduled Jobs** | Run tasks on a recurring schedule | Reminders, cleanup, reports, retries | **All 4 repos** (see Section 8) |
| **Rate Limiting** | Throttle requests per user/IP | All public APIs | **LinkUp**, **ShopCart** |
| **Idempotency** | Same request multiple times = same result | Payment processing, webhooks | **ShopCart**, **BookIt** |
| **Caching (Redis)** | Store frequently accessed data for fast retrieval | All production APIs | **All 4 repos** |

### 2.4 Other Possible Patterns (Can Be Accommodated)

These are advanced patterns that can be introduced as additional tasks or future enhancements within the repos. They are widely used in production systems but are not part of the initial core scope.

| Pattern | Type | What It Does | Could Fit In | How It Would Be Used |
|---|---|---|---|---|
| **SSE (Server-Sent Events)** | Communication | One-way server-to-client push over HTTP | **MatchDay** | Fallback for WebSocket — simpler one-way score streaming |
| **gRPC** | Communication | Fast binary protocol with strict contracts (protobuf) | **ShopCart** | Inter-service communication between microservices for speed |
| **CQRS** | Architecture | Separate models/paths for reading vs writing | **BookIt**, **ShopCart** | Separate availability read model from booking write model |
| **Event Sourcing** | Architecture | Store every state change as an event; rebuild state by replaying | **BookIt** | Booking history as immutable event log — audit trail, undo |
| **Saga Pattern** | Architecture | Coordinate multi-step transactions across services with rollback | **ShopCart** | Checkout flow — on payment failure, release inventory automatically |
| **Message Queue** | Architecture | Async job processing; tasks queued and processed later | **ShopCart**, **BookIt** | Email sending, inventory sync, report generation via BullMQ/RabbitMQ |
| **Circuit Breaker** | Architecture | Stop calling a failing service to prevent cascade failure | **ShopCart** | If Payment Service is down, stop calling it after N failures |
| **Database per Service** | Architecture | Each microservice owns its own data store | **ShopCart** | Each service (Catalog, Order, Payment) has independent MongoDB |
| **Retry with Backoff** | Operational | Retry failed operations with increasing delay | **ShopCart** | Failed event processing retried with exponential backoff |

---

## 3. Repo 1 — LinkUp (LinkedIn Clone / GraphQL)

### 3.1 The Real App: LinkedIn

LinkedIn is the world's largest professional networking platform with 1B+ members. Every professional has a LinkedIn profile. Candidates are deeply familiar with its core features: profiles, connections, posts/feed, job listings, and messaging.

**Why LinkedIn?**
- Universally familiar to every candidate (they literally use it to find jobs)
- Rich relational data model — profiles, connections, companies, jobs, posts, comments, skills, endorsements
- GraphQL is a **natural fit** — the LinkedIn-style feed requires nested data (post → author → company → comments → replies) with different clients needing different fields
- No location dependency — everything is content/data based

### 3.2 Clone Name: **LinkUp**

"LinkUp" — Link up with professionals. A professional networking platform where users build profiles, connect with peers, share posts, and browse job listings.

### 3.3 Features We're Cloning

| Feature Area | What It Does | GraphQL Relevance |
|---|---|---|
| **User Profiles** | Create/edit professional profile with headline, summary, experience, education, skills | Nested object queries — profile → experiences → company, profile → skills → endorsements |
| **Connection System** | Send/accept/reject connection requests, mutual connections | Relationship queries — connections → mutual → suggestions |
| **Feed / Posts** | Create posts, like, comment, share; personalized feed | Deeply nested: feed → posts → author → comments → replies → likes. Different clients need different fields |
| **Job Listings** | Companies post jobs; users browse, search, and apply | Multi-entity joins: job → company → applicants → application status |
| **Job Applications** | Apply to jobs, track application status, withdraw | State machine: applied → reviewed → shortlisted → interview → offered → accepted/rejected |
| **Company Pages** | Company profiles with details, employees, posted jobs | Nested: company → employees → jobs → applicants |
| **Skills & Endorsements** | Add skills to profile, endorse connections' skills | Many-to-many: user ↔ skill ↔ endorser |
| **Search** | Search users, jobs, companies with filters | Complex filtered queries — GraphQL's strength over REST |
| **Notifications** | Real-time notifications for connection requests, likes, comments, application updates | GraphQL Subscriptions — push notifications over WebSocket transport |

### 3.4 Communication & Architectural Patterns

| Pattern | How It's Used in LinkUp |
|---|---|
| **GraphQL Queries** | All read operations — profile, feed, jobs, search. Client asks for exactly the fields it needs. |
| **GraphQL Mutations** | All write operations — create post, send connection request, apply to job, endorse skill. |
| **GraphQL Subscriptions** | Real-time notifications — new connection request, someone liked your post, application status changed. Uses WebSocket transport under the hood. |
| **DataLoader / N+1 Prevention** | Feed query fetches 20 posts → each needs author + comments. Without batching = 60+ DB calls. DataLoader batches into 3 calls. |
| **Schema Design** | Type definitions, interfaces, unions, input types, enums for job status, connection status, etc. |
| **Resolver Architecture** | Field-level resolvers, resolver chains, context-based auth, error handling in resolvers. |
| **Caching (Redis)** | Cache frequently accessed data — profile views, feed, search results. Cache invalidation on mutations. |
| **Authentication in GraphQL** | JWT via context, field-level authorization (e.g., salary visible only to job poster), directive-based auth. |

### 3.5 Task Potential

This domain enables tasks across these areas:

| Task Area | Bug Examples | Feature Examples |
|---|---|---|
| **Resolver Logic** | N+1 query problem in feed resolver, incorrect pagination cursor logic | Implement feed algorithm with connection-based ranking |
| **Auth & Permissions** | Field-level authorization leak (salary visible to wrong role), connection-only content bypass | Implement directive-based field authorization |
| **Schema Design** | Wrong union/interface usage causing type resolution failures | Implement polymorphic search returning users + jobs + companies |
| **Data Integrity** | Connection request allowing duplicate requests, endorsement count not updating atomically | Implement mutual connection suggestions algorithm |
| **Subscriptions** | Subscription not filtering events for the correct user, memory leak on unsubscribe | Implement real-time notification system |
| **Caching** | Stale cache after profile update, cache key collision across users | Implement cache layer with proper invalidation |
| **Input Validation** | GraphQL input type accepting invalid data (negative experience years, future dates) | Implement custom scalar validators |
| **Aggregation** | Job application analytics returning wrong counts per status | Implement company analytics dashboard resolver |

---

## 4. Repo 2 — MatchDay (ESPN Clone / WebSocket)

### 4.1 The Real App: ESPN

ESPN is the world's leading sports media platform. Live scores, match events, standings, and statistics are consumed by hundreds of millions of fans globally. Every candidate has checked a live score on their phone during a match.

**Why ESPN?**
- Universally familiar — sports are global (football/soccer, basketball, cricket, tennis)
- WebSocket is the **only way** to deliver live scores — polling is wasteful, SSE is one-way
- Location-agnostic — scores are just numbers, team names are universal
- Solo-testable — we build a **Match Simulator** service that auto-generates events (candidate doesn't touch it)
- Rich domain — matches, scores, events (goals, cards, substitutions), standings, player stats, seasons

### 4.2 Clone Name: **MatchDay**

"MatchDay" — Every fan lives for match day. A live sports platform where users subscribe to matches and receive real-time score updates, match events, and league standings.

### 4.3 The Match Simulator (Pre-built, Not Candidate's Task)

A background service that generates realistic match events on a timer:

```
Every few seconds the simulator emits events like:
→ { match: "Arsenal vs Chelsea", event: "GOAL", team: "Arsenal", player: "Saka", minute: 34, score: { home: 1, away: 0 } }
→ { match: "Arsenal vs Chelsea", event: "YELLOW_CARD", player: "Silva", minute: 41 }
→ { match: "Lakers vs Celtics", event: "SCORE", team: "Lakers", player: "James", points: 3, score: { home: 78, away: 72 } }
→ { match: "Arsenal vs Chelsea", event: "HALF_TIME", score: { home: 1, away: 0 } }
→ { match: "Arsenal vs Chelsea", event: "FULL_TIME", score: { home: 2, away: 1 } }
```

- Simulator is **pre-built and running** — candidate does NOT build it
- Candidate's job = build the **WebSocket server** that receives simulator events and broadcasts to connected clients correctly
- Test cases verify: connection handling, event broadcasting, room/channel subscription, filtering, reconnection, etc.

### 4.4 Features We're Cloning

| Feature Area | What It Does | WebSocket Relevance |
|---|---|---|
| **Live Score Feed** | Real-time score updates for all ongoing matches | Core WebSocket broadcast — server pushes every score change to all connected clients |
| **Match Subscription** | Subscribe to specific matches to get detailed events | WebSocket rooms/channels — client joins "match:arsenal-vs-chelsea" room |
| **Match Events Stream** | Goals, cards, substitutions, penalties streamed live | Event filtering — client subscribes to event types (goals only, all events) |
| **League Standings** | Live standings table updated after each match result | Computed updates — recalculate standings on FULL_TIME event, push to subscribers |
| **Player Statistics** | Player stats updated in real-time during matches | Aggregation — accumulate goals, assists, cards per player across matches |
| **Match Schedule** | Upcoming and completed matches with details | REST endpoints for static data (non-WebSocket) |
| **Score Alerts** | Configurable alerts — notify when your team scores | Conditional push — check user preferences before sending |
| **Match Summary** | Post-match summary with all events, timeline | Event storage and retrieval — rebuild match from stored events |
| **Sport/League Filtering** | Filter by sport, league, team | Multi-room subscription management |
| **Connection Management** | Heartbeat, reconnection, graceful disconnect | WebSocket lifecycle — ping/pong, connection state, cleanup |

### 4.5 Communication & Architectural Patterns

| Pattern | How It's Used in MatchDay |
|---|---|
| **WebSocket (Core)** | Persistent connection for live score streaming. Server pushes events as they happen. |
| **Pub/Sub (Internal)** | Match simulator publishes events → WebSocket server subscribes → broadcasts to clients. Redis Pub/Sub as the message bus. |
| **Rooms / Channels** | Clients subscribe to specific match rooms. Events are broadcast only to relevant rooms. |
| **Event Filtering** | Clients specify what events they care about (goals only, all events). Server filters before sending. |
| **Connection Lifecycle** | Heartbeat (ping/pong), connection timeout, graceful disconnect, reconnection with replay. |
| **REST (Supporting)** | Static data — match schedule, league info, historical stats. Not everything needs to be real-time. |
| **Caching** | Cache standings, match data. Invalidate on match events. |
| **Rate Limiting** | Limit connections per user, limit subscription count. |

### 4.6 Task Potential

| Task Area | Bug Examples | Feature Examples |
|---|---|---|
| **Connection Handling** | Connection not cleaned up on disconnect (memory leak), heartbeat not timing out dead connections | Implement heartbeat ping/pong with configurable timeout |
| **Room Management** | Client receiving events from unsubscribed matches, room not cleaned up when match ends | Implement multi-room subscription with join/leave |
| **Event Broadcasting** | Events sent to all clients instead of room subscribers, duplicate events on reconnect | Implement targeted broadcast with event deduplication |
| **Event Filtering** | Filter ignoring user preferences, wrong event type matching | Implement configurable event filter per subscription |
| **Reconnection** | Client missing events during disconnect, replay sending stale events | Implement reconnection with event replay from last received |
| **Standings Calculation** | Standings not updated atomically, wrong points/goal difference after concurrent match ends | Implement live standings recalculation engine |
| **Score Alerts** | Alert sent to wrong user, alert not triggered for correct conditions | Implement user-configurable score alert system |
| **Data Consistency** | Player stats not matching match events, aggregate stats drifting | Implement player stats aggregation from event stream |

---

## 5. Repo 3 — ShopCart (Amazon Clone / Microservices + Event-Driven)

### 5.1 The Real App: Amazon

Amazon is the world's largest e-commerce platform. Everyone has browsed products, added items to cart, placed an order, and tracked a shipment. The order lifecycle (browse → cart → checkout → payment → fulfillment → shipping → delivery) is a textbook example of event-driven microservices.

**Why Amazon?**
- Universally familiar — everyone shops online
- The order lifecycle is a **perfect event-driven pipeline** — each step is a distinct service emitting events
- Full scope offers massive task variety — products, cart, orders, payments, inventory, shipping, notifications
- Location-agnostic — product catalog is just data, no GPS needed
- Microservices are natural — each domain (catalog, cart, order, payment, shipping) is an independent service

### 5.2 Clone Name: **ShopCart**

"ShopCart" — Shop it, cart it, done. A full-scope e-commerce platform where users browse products, manage carts, place orders, process payments, and track shipments — all powered by independent microservices communicating via events.

### 5.3 Features We're Cloning (Full Scope)

| Feature Area | What It Does | Microservice / Event Relevance |
|---|---|---|
| **Product Catalog** | Browse products, categories, search, filter, reviews, ratings | **Catalog Service** — independent service with its own database |
| **Shopping Cart** | Add/remove items, update quantities, cart total calculation | **Cart Service** — manages cart state, emits `cart.updated` events |
| **Inventory Management** | Track stock levels, reserve on checkout, release on cancellation | **Inventory Service** — listens to order events, manages stock atomically |
| **Order Management** | Place order, track status, order history, cancel | **Order Service** — orchestrates the checkout saga, emits lifecycle events |
| **Payment Processing** | Process payment, refund, webhook verification | **Payment Service** — handles payment gateway integration, emits `payment.completed` / `payment.failed` |
| **Shipping & Tracking** | Ship order, track shipment status, delivery confirmation | **Shipping Service** — listens to `payment.completed`, manages shipment lifecycle |
| **Notifications** | Email/push notifications for order updates, shipping, delivery | **Notification Service** — listens to ALL domain events, sends appropriate notifications |
| **User Management** | Register, login, profile, addresses, order history | **User Service** — authentication, user data |
| **Coupons & Discounts** | Apply promo codes, validate eligibility, track usage | Part of Order Service — coupon validation during checkout |
| **Reviews & Ratings** | Post-delivery reviews, star ratings, helpful votes | Part of Catalog Service — only allowed after delivery confirmed |
| **Wishlist** | Save products for later, move to cart | Part of Cart Service or standalone |

### 5.4 The Order Lifecycle (Event Flow)

```
Customer places order
    │
    ▼
[Order Service] ──── emits: order.created ────┐
    │                                          │
    ▼                                          ▼
[Inventory Service]                    [Payment Service]
  Reserves stock                        Initiates charge
  emits: inventory.reserved             emits: payment.completed
    │                                          │
    │         ┌────────────────────────────────┘
    ▼         ▼
[Order Service] ──── emits: order.confirmed ──┐
    │                                          │
    ▼                                          ▼
[Shipping Service]                    [Notification Service]
  Creates shipment                     Sends confirmation email
  emits: shipment.created
    │
    ▼
  emits: shipment.in_transit
    │
    ▼
  emits: shipment.delivered ──────────┐
                                       │
                                       ▼
                              [Order Service]
                                Updates: order.delivered
                              [Notification Service]
                                Sends delivery email
                              [Review Service]
                                Enables review for this order
```

### 5.5 The Checkout Saga (Distributed Transaction)

When a checkout fails at any step, the saga rolls back all previous steps:

```
Step 1: Reserve Inventory     ──── FAIL? → (nothing to rollback)
Step 2: Process Payment        ──── FAIL? → Release Inventory
Step 3: Create Shipment        ──── FAIL? → Refund Payment + Release Inventory
Step 4: Confirm Order          ──── FAIL? → Cancel Shipment + Refund Payment + Release Inventory
```

Each step emits compensating events on failure — this is the **Saga Pattern** in action.

### 5.6 Communication & Architectural Patterns

| Pattern | How It's Used in ShopCart |
|---|---|
| **Microservices** | 5-7 independent services, each with its own API. Catalog, Cart, Order, Payment, Inventory, Shipping, Notification. |
| **Event-Driven (Pub/Sub)** | Services communicate via events through a message broker (Redis Streams / RabbitMQ). Order Service emits `order.created`, Payment Service listens and processes. |
| **Webhooks** | Payment gateway (simulated) sends webhook on payment success/failure. Webhook handler verifies signature, ensures idempotency. |
| **API Gateway** | Single entry point for all client requests. Routes to appropriate microservice. Handles auth, rate limiting. |
| **REST (External API)** | Client-facing API is REST. Internal communication uses events. |
| **Idempotency** | Payment processing and webhook handling must be idempotent — same request twice = same result. |
| **Caching** | Product catalog, cart data cached in Redis. Invalidated on mutations. |

### 5.7 Task Potential

| Task Area | Bug Examples | Feature Examples |
|---|---|---|
| **Event Routing** | Event published to wrong topic, consumer not acknowledging correctly | Implement event publisher with topic routing |
| **Webhook Security** | Webhook handler not verifying signature, not checking idempotency key | Implement secure webhook handler with HMAC verification |
| **Inventory Atomicity** | Race condition — two orders reserve last item simultaneously | Implement atomic inventory reservation |
| **Data Consistency** | Order status out of sync with payment status across services | Implement eventual consistency reconciliation |
| **API Gateway** | Auth not propagated to downstream services, rate limit bypass | Implement gateway middleware with auth forwarding |
| **Coupon Validation** | Coupon used beyond limit due to non-atomic check-and-decrement | Implement atomic coupon redemption |
| **Shipping Lifecycle** | Status transition skipping states (shipped → delivered without in_transit) | Implement shipment state machine |
| **Order Cancellation** | Cancellation not releasing inventory, not triggering refund | Implement order cancellation with rollback across services |
| **Payment Processing** | Payment status not synced back to order, duplicate charges | Implement payment flow with idempotency |

---

## 6. Repo 4 — BookIt (Calendly Clone / REST + Webhooks + CRON)

### 6.1 The Real App: Calendly

Calendly is the leading scheduling automation platform. Hosts expose their availability, guests book time slots — eliminating back-and-forth emails. Used by sales teams, customer success managers, recruiters, and anyone who schedules meetings.

**Why Calendly?**
- Universally familiar — candidates have booked meetings via scheduling links
- Manager already identified this as a strong assessment domain
- The scheduling engine is logic-heavy — timezone math, conflict detection, buffer windows, availability rules
- **REST + Webhooks + CRON** is a natural fit — booking endpoints, webhook notifications to external calendars, and CRON for reminders/cleanup
- The scheduling engine is the real challenge — availability computation, conflict detection, timezone math, buffer rules
- Solo-testable — one host, simulated guests (or candidate acts as both)
- Location-agnostic — timezones are handled in code, no physical location needed

### 6.2 Clone Name: **BookIt**

"BookIt" — Just book it. A scheduling platform where hosts define availability rules, create event types, and share booking links. Guests view available slots in their timezone and book instantly.

### 6.3 Features We're Cloning

| Feature Area | What It Does | Technical Relevance |
|---|---|---|
| **Host Availability** | Set working hours, blocked dates, timezone, minimum notice, max booking window | Core scheduling logic — availability computation from rules |
| **Event Types** | Create meeting types: name, duration, buffer before/after, max per day, color, description | CRUD + validation — business rules per event type |
| **Booking Page** | Guest opens link, sees available slots in their timezone, picks one | Availability query — compute open slots from rules + existing bookings |
| **Slot Booking** | Guest selects slot, enters details, confirms | Booking endpoint + conflict check + webhook notification |
| **Conflict Detection** | Prevent double-booking, respect buffers, check max-per-day limit | Core validation logic — atomic check before booking |
| **Timezone Handling** | All times stored in UTC; displayed in user's local timezone | Conversion logic when computing and displaying available slots |
| **Rescheduling** | Change booking time, notify both parties | Update booking + revalidate conflicts + fire webhook |
| **Cancellation** | Cancel booking, free up the slot, notify | Cancel flow + slot release + webhook notification |
| **Reminders** | Email reminders N hours before meeting | **CRON job** — scans upcoming bookings, sends reminders (see Section 8) |
| **Calendar Integration** | Connect Google/Outlook calendar, sync busy times | **Webhook** — receive external calendar events, block those slots |
| **Booking Analytics** | Booking conversion rate, popular times, no-show rate, busiest days | Aggregation queries over bookings data |
| **Team Scheduling** | Round-robin across team members, pool availability | Advanced feature — distributes bookings across team based on load/rules |
| **Booking Limits** | Max bookings per day, per week, per event type | Part of availability rules — enforced during booking validation |
| **Buffer Time** | Enforce X minutes before/after each meeting (prep time) | Buffer windows subtracted from available slots during computation |

### 6.4 The Booking Flow

```
Guest opens booking page
    │
    ▼
[Availability Engine]
  1. Load host's working hours + rules
  2. Subtract existing bookings
  3. Subtract buffer windows (before/after each meeting)
  4. Subtract external calendar busy times (synced via webhook)
  5. Apply booking limits (max per day/week)
  6. Apply minimum notice + max booking window
  7. Convert to guest's timezone
  8. Return available slots
    │
    ▼
Guest selects a slot and confirms
    │
    ▼
[Booking Handler]
  1. Re-validate slot is still available (prevent race condition)
  2. Check conflict with buffer windows
  3. Check booking limits not exceeded
  4. Create booking record
  5. Fire outgoing webhook → external calendar, CRM, notification
  6. Schedule reminder via CRON
    │
    ▼
[CRON Jobs]
  - Send reminders N hours before meeting
  - Clean up expired/unconfirmed bookings
  - Re-sync external calendar data
```

### 6.5 Communication & Architectural Patterns

| Pattern | How It's Used in BookIt |
|---|---|
| **REST (Core API)** | Client-facing API for all operations — create event type, book slot, get availability, manage bookings. |
| **Webhooks (Outgoing)** | When a booking is created/changed/cancelled, fire webhook to external systems (calendar, CRM, notification). |
| **Webhooks (Incoming)** | Receive webhooks from Google/Outlook Calendar when external events change (new meeting added → blocks that slot). |
| **CRON / Scheduled Jobs** | Reminders, cleanup expired pending bookings, calendar re-sync, daily analytics (see Section 8). |
| **Caching** | Cache computed availability with TTL. Invalidate when bookings change. |
| **Idempotency** | Booking requests must be idempotent — double-click protection, webhook replay protection. |

### 6.6 Task Potential

| Task Area | Bug Examples | Feature Examples |
|---|---|---|
| **Availability Calculation** | Available slots not accounting for buffers, timezone offset wrong by 1 hour | Implement availability computation engine |
| **Conflict Detection** | Double-booking when two guests book simultaneously | Implement atomic slot reservation with conflict check |
| **Timezone Handling** | DST transition causing 1-hour shift in displayed slots | Implement timezone-aware slot computation |
| **Booking Limits** | Max-per-day check not counting rescheduled bookings | Implement booking limit enforcement across event types |
| **Webhook Delivery** | Outgoing webhook not retrying on failure, no signature | Implement webhook dispatch with HMAC signing and retry |
| **Reminder System** | Reminder sent for cancelled bookings, reminder timing wrong across timezones | Implement CRON-based reminder with timezone awareness |
| **Calendar Sync** | External busy times not blocking availability, sync race condition | Implement incoming webhook handler for calendar sync |
| **Rescheduling** | Rescheduled booking not freeing original slot, buffer not recalculated | Implement reschedule flow with full revalidation |
| **Cancellation** | Cancelled booking's slot not freed, webhook not fired | Implement cancellation with slot release + webhook |
| **Analytics** | Conversion rate calculation dividing by wrong denominator, no-show tracking broken | Implement booking analytics aggregation endpoint |
