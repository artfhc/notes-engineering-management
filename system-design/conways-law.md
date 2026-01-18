# Conway's Law & Team Topologies

## Conway's Law

Organizations design systems that reflect their communication structures — meaning team boundaries drive service boundaries.

**Reference:** [Conway's Law - Martin Fowler](https://martinfowler.com/bliki/ConwaysLaw.html?utm_source=chatgpt.com)

## Inverse Conway Maneuver

The reverse of Conway's Law strategy: design team structure to produce the architecture you want.

## Domain-Driven Design

Focuses on defining bounded contexts and modeling domain logic — excellent for guiding how you break up teams around business capabilities, not just technical layers.

## Team Topologies (Conceptual Framework)

This is a modern approach that pairs organizational structure with software architecture explicitly (team types like stream-aligned, enabling, platform teams, etc.).

---

## Team Topologies 101 (Quick Primer)

Team Topologies defines **4 team types:**

| Team Type           | Purpose                                                      |
|---------------------|--------------------------------------------------------------|
| Stream-aligned      | Deliver value end-to-end for a specific product/domain      |
| Platform            | Provide reusable infrastructure, tooling, or services       |
| Enabling            | Help stream-aligned teams adopt new tech/processes          |
| Complicated Subsystem | Owns highly specialized areas needing deep expertise       |

### Three Interaction Modes

* **Collaboration** (close partnership to explore/innovate)
* **X-as-a-Service** (consume another team's output)
* **Facilitating** (short-term coaching)

---

## Simplified Team Breakdown for a Chat App

### 1. Chat Experience Team (Stream-Aligned)

**Scope:**

* Frontend (Web, iOS, Android)
* Message list UI, input, reactions, threads, attachments
* Works closely with design and product

💡 **Owns everything the user touches.**

### 2. Messaging Infra Team (Platform)

**Scope:**

* WebSocket/gRPC connections
* Real-time message delivery
* Retry logic, reconnections, ordering, deduplication

💡 **Provides messaging SDKs to frontend/mobile teams.**

### 3. Chat Storage & History Team (Platform)

**Scope:**

* Message persistence and retrieval
* Query APIs (for scrolling, search, etc.)
* Archiving, deletion (GDPR), backups

💡 **Ensures data durability, scalability.**

### 4. Presence & Notifications Team (Stream-Aligned)

**Scope:**

* Online/offline, typing indicators
* Push notifications (mobile + web)
* Read receipts, delivery status

💡 **Owns status signaling and engagement.**

### 5. Group & Permissions Team (Stream-Aligned)

**Scope:**

* Group creation, invites, roles
* Mute/block, access control
* Admin/mod tools

💡 **Manages who can see/talk to whom.**

### Optional Supporting Teams (As the Org Grows)

#### Platform UI/SDK Team (Platform)

* Shared React components or Compose/UIKit components
* Message rendering, emoji picker, media upload, etc.

#### Security & Abuse Team (Complicated Subsystem)

* Spam detection, blocklists, rate limiting
* (Optional: End-to-end encryption if needed)

### Visual Summary

```text
Stream-Aligned Teams:
├── Chat Experience (Web, iOS, Android)
├── Presence & Notifications
├── Group & Permissions

Platform Teams:
├── Messaging Infra (Real-time delivery, SDKs)
├── Storage & History (Persistence, search)
├── UI Platform SDK (Optional, shared components)

Complicated Subsystem (Optional):
└── Security & Abuse (Spam, privacy, encryption)
```

### Why This Works

* **Simple:** 5 core teams (3 product-facing, 2 infra-focused)
* **Scalable:** Can grow horizontally (add Chat Monetization, Voice/Video later)
* **Clear Boundaries:** Each team owns end-to-end delivery for their domain
* **Balanced:** Teams are not too big or too deeply specialized early on

---

## Simplified Team Breakdown for an Ecommerce Application

We'll divide into:

* **Customer-facing product teams** (Stream-aligned)
* **Infrastructure & platform support teams** (Platform)
* **Specialist teams** (Optional, only as you scale)

### 1. Search & Discovery Team (Stream-Aligned)

**Scope:**

* Search bar, filters, sort, results
* Ranking & ML-driven recommendations
* Autocomplete, spell correction, visual search

💡 **Goal: Help users find the right products fast.**

### 2. Product Detail Page (PDP) Team (Stream-Aligned)

**Scope:**

* Product images, title, price, reviews
* Variants (size, color), delivery estimates
* Buy Now/Add to Cart logic

💡 **Goal: Drive conversion with compelling and accurate product info.**

### 3. Cart & Checkout Team (Stream-Aligned)

**Scope:**

* Cart state, quantity management
* Promo codes, shipping/billing info
* Payment gateway integration

💡 **Goal: Ensure a smooth, trustworthy buying experience.**

### 4. Post-Purchase Team (Stream-Aligned)

**Scope:**

* Order history, tracking, cancellations
* Returns, refunds, customer communication
* Loyalty programs, repeat purchase flows

💡 **Goal: Encourage repeat usage and customer satisfaction.**

### 5. Mobile/Web UI Experience Team (Platform)

**Scope:**

* Shared UI component libraries
* Design system enforcement (Web, iOS, Android)
* Accessibility, localization, theming

💡 **Goal: Maintain UI consistency and dev velocity across clients.**

### 6. API Platform & Backend Infra Team (Platform)

**Scope:**

* GraphQL/REST API gateway
* API versioning, auth, rate limits
* Performance, caching, observability

💡 **Goal: Enable fast and reliable data access for client teams.**

### Optional Specialist Teams (As Org Grows)

#### Growth/Experimentation Team (Stream-Aligned or Enabling)

* A/B testing infra, onboarding flows, referrals, SEO optimization

#### Search Ranking & ML Infra Team (Complicated Subsystem)

* Builds and serves ranking/recommendation models

#### Customer Trust & Safety Team (Stream-Aligned + Complicated)

* Fraud detection, reviews moderation, abuse reports

### Visual Summary

```text
Stream-Aligned Teams:
├── Search & Discovery
├── Product Detail Page (PDP)
├── Cart & Checkout
├── Post-Purchase / Orders
├── Growth (optional)

Platform Teams:
├── Mobile & Web UI Infra
├── API Platform & Backend Infra

Complicated Subsystem:
├── Search Ranking / ML Models
├── Trust & Safety / Fraud Detection (optional)
```

### Why This Structure Works

* **Cross-platform aligned:** Web, iOS, Android can share logic via platform teams
* **Clear ownership:** Each team owns a distinct part of the customer journey
* **Scale-friendly:** As you grow, you can split teams by platform or vertical (e.g. split Cart into Cart UI and Payment Infra)
* **Customer-first:** Stream-aligned teams follow the user lifecycle (discover → decide → buy → track)

---

## Simplified Team Structure for a Social Media App

We'll assume a typical social app has the following key areas:

* Feed + Posts
* Messaging
* Notifications
* Profile
* Social Graph (follows, friends)
* Moderation / Trust & Safety
* Multi-platform (Web, iOS, Android)

### Stream-Aligned Teams (User-Facing Features, End-to-End Ownership)

| Team                        | Scope                                                                          |
|-----------------------------|--------------------------------------------------------------------------------|
| Feed & Posts Team           | Timeline/scrolling feed, post creation, comments, media upload, likes/reactions |
| Messaging Team              | Real-time DM/chat, typing indicators, read receipts, attachments              |
| Notification Team           | In-app + push notifications, alerts, follow events, tagging                   |
| Profile & Identity Team     | User profile page, username/avatar, bio, privacy settings                     |
| Social Graph Team           | Follow/unfollow, mutual friends, suggested users                              |
| Moderation & Safety Team    | Reporting, muting/blocking, comment filters, content takedown                 |
| Growth & Onboarding Team (optional) | Sign-up, login, onboarding flow, invites, referral tracking          |

💡 **Each team handles the backend, API, and client-side (Web + Mobile) responsibilities of its domain.**

### Platform Teams (Reusable Infra/Tools/Services)

| Team                           | Scope                                                                      |
|--------------------------------|----------------------------------------------------------------------------|
| Client Platform Team (Web + Mobile) | Shared UI components, image caching, input tools, platform SDKs      |
| Real-Time Platform Team        | WebSocket infra, delivery guarantees, connection monitoring                |
| API Platform Team              | GraphQL/REST APIs, versioning, authentication, rate limiting               |
| Observability & Reliability    | Logging, metrics, crash reporting, alerts                                  |
| Experimentation & A/B Infra    | Feature flags, cohort analysis, rollout tools                              |

💡 **Platform teams enable stream-aligned teams to move fast without reinventing infra.**

### Complicated Subsystem Teams (Expertise-Focused)

| Team                           | Scope                                                                      |
|--------------------------------|----------------------------------------------------------------------------|
| Recommendation & Ranking Team  | Timeline ranking, suggested users, trending content (ML-focused)           |
| Spam/Abuse Detection Team      | NLP filters, account scoring, coordinated behavior detection               |
| Media Processing Team          | Image/video compression, transcoding, moderation queues                    |

💡 **These teams maintain domain-specific expertise and expose APIs/libraries for other teams.**

### Client Ownership Model

Each stream-aligned team should have dedicated Web + iOS + Android engineers, so:

* **Feed & Posts Team** owns Feed across all platforms
* **Messaging Team** owns Chat UX and socket integration on all clients
* **Platform team** builds shared SDKs/components for speed + consistency

### Visual Summary

```text
Stream-Aligned Teams
├── Feed & Posts
├── Messaging
├── Notifications
├── Profile & Identity
├── Social Graph
├── Moderation & Safety
└── Growth & Onboarding (optional)

Platform Teams
├── Client UI Platform
├── Real-Time Infra
├── API Gateway
├── Observability
└── Experimentation Infra

Complicated Subsystems
├── ML-based Ranking / Recommender
├── Spam & Abuse Detection
└── Media Pipeline (Video/Image)
```

### Why This Structure Works

* **End-to-end ownership:** One team handles full-stack delivery per domain
* **Fast iteration:** Teams can launch features independently
* **Shared velocity tools:** Platform teams remove duplication
* **Focused expertise:** Specialized teams work in high-complexity areas like ML or abuse detection
