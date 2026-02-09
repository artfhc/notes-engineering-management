# Social Media Manager API Design

## Problem Statement

Design the public HTTP API for a web service that helps corporate social media managers post to multiple social media networks at once (such as Facebook, Twitter, or LinkedIn).

**Use Cases:**
1. Create a new post on one social network
2. Create a new post across multiple social networks
3. Provide statistics to show how users interacted with a post
4. Delete an existing post

---

## Core Entities

### 1. Post

Represents a social media message authored by a user.

| Field | Type | Description |
|-------|------|-------------|
| `postId` | string | Internal ID for this post |
| `content` | string | Text content of the post |
| `mediaUrls` | list of strings | Optional image or video URLs |
| `createdAt` | datetime | Time the post was created |
| `scheduledAt` | datetime (optional) | Time the post should be published |
| `networks` | list of strings | Platforms targeted (e.g., "facebook") |
| `status` | string | "posted", "scheduled", "deleted" |
| `networkPostIds` | map<string, string> | ID of post per platform |

### 2. Statistic

Describes engagement metrics for a post on a platform.

| Field | Type | Description |
|-------|------|-------------|
| `network` | string | e.g., "facebook" |
| `likes` | int | Number of likes |
| `shares` | int | Number of shares |
| `comments` | int | Number of comments |
| `impressions` | int | Number of views/impressions |

### 3. Media

Media assets attached to posts.

| Field | Type | Description |
|-------|------|-------------|
| `mediaId` | string | Internal ID |
| `mediaUrl` | string | CDN URL |
| `type` | string | "image" or "video" |
| `status` | string | "uploaded", "processing", etc. |
| `networkMediaIds` | map<string, string> | Per-network ID mappings |

---

## RESTful API Design

### POST /v1/posts

**Description:** Create a new post on one or more social networks.

**Request Body:**

```json
{
  "content": "Check out our new product!",
  "mediaUrls": ["https://cdn.example.com/image.jpg"],
  "networks": ["facebook", "linkedin", "twitter"],
  "scheduledAt": "2025-06-30T10:00:00Z"
}
```

**Response:**

```json
{
  "postId": "abc123",
  "networkPostIds": {
    "facebook": "fb-001",
    "linkedin": "ln-321",
    "twitter": "tw-999"
  },
  "createdAt": "2025-06-25T14:00:00Z"
}
```

---

### GET /v1/posts/{postId}

**Description:** Retrieve details of a post.

**Response:**

```json
{
  "postId": "abc123",
  "content": "Check out our new product!",
  "mediaUrls": ["https://cdn.example.com/image.jpg"],
  "networks": ["facebook", "linkedin", "twitter"],
  "status": "posted",
  "scheduledAt": "2025-06-30T10:00:00Z",
  "networkPostIds": {
    "facebook": "fb-001",
    "linkedin": "ln-321",
    "twitter": "tw-999"
  },
  "createdAt": "2025-06-25T14:00:00Z"
}
```

---

### GET /v1/posts/{postId}/statistics

**Description:** Get engagement metrics for a post.

**Response:**

```json
{
  "postId": "abc123",
  "statistics": [
    {
      "network": "facebook",
      "likes": 134,
      "shares": 22,
      "comments": 10,
      "impressions": 1500
    },
    {
      "network": "twitter",
      "likes": 59,
      "shares": 12,
      "comments": 4,
      "impressions": 900
    }
  ]
}
```

---

### DELETE /v1/posts/{postId}

**Description:** Delete a post from all networks.

**Response:**

```json
{
  "postId": "abc123",
  "deletedFrom": ["facebook", "linkedin", "twitter"],
  "postStatus": "deleted"
}
```

---

### DELETE /v1/posts/{postId}?networks=facebook,twitter

**Description:** Delete the post only from selected networks.

**Response:**

```json
{
  "postId": "abc123",
  "deletedFrom": ["facebook", "twitter"],
  "postStatus": "partial"
}
```

---

## Media Upload Flow

### POST /v1/media

To manage consistency and compatibility across networks, upload media to your server first.

**Request:**
- `file`: binary (image or video)
- `type`: "image" or "video"

**Response:**

```json
{
  "mediaId": "media_12345",
  "mediaUrl": "https://cdn.example.com/media_12345.jpg"
}
```

**Note:** Alternatively, clients can provide publicly accessible media URLs, but you lose control over content format, lifespan, and transformations.

### Backend Media Processing Flow

Each social network has different requirements for media:

| Network | Requirements |
|---------|-------------|
| Facebook | Accepts image/video URLs or file uploads via API |
| Twitter | Requires separate media upload before tweeting |
| LinkedIn | Requires media upload first, then post referencing that media |

**Your Backend Flow:**

1. Fetch and validate media (if external URL provided)
2. Convert or compress media as needed
3. Upload to each network's media endpoint (e.g., Twitter's `media/upload.json`)
4. Use resulting media IDs in post creation API calls per network
5. Store network-specific media IDs for reuse/debugging

---

## API Versioning

Versioning protects your clients from breaking changes when you update your API.

### Recommended Approach: URL Versioning

Prefix your routes with a version number:

```
POST /v1/posts
GET /v1/posts/{postId}
GET /v1/posts/{postId}/statistics
DELETE /v1/posts/{postId}
```

### When to Version:

- Adding/removing fields
- Changing field types or semantics
- Changing response structure

### Alternatives (not recommended for public APIs):

- Header-based versioning: `Accept: application/vnd.myapi.v1+json`
- Query params: `/posts?version=1`

---

## Rate Limiting

Rate limiting protects your API from abuse and ensures fair usage.

### Best Practices:

- Set limits per API key or user (e.g., 1000 requests/day)
- Use HTTP headers to expose rate limit info:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 250
X-RateLimit-Reset: 1691424000
```

### Enforcement Strategies:

- Token Bucket or Leaky Bucket algorithms
- API Gateway (e.g., Amazon API Gateway, Kong, NGINX)

### Response Example on Limit Exceeded:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

---

## Error Handling

Robust APIs must prepare for real-world failures.

### Standard HTTP Status Codes

| Status | Use Case |
|--------|----------|
| 200 OK | Successful GET/POST |
| 201 Created | Resource was created |
| 400 Bad Request | Validation error |
| 401 Unauthorized | Invalid/missing token |
| 403 Forbidden | No access to the resource |
| 404 Not Found | Resource doesn't exist |
| 409 Conflict | Duplicate post attempt |
| 429 Too Many Requests | Rate limit exceeded |
| 500 Internal Server Error | Unexpected failure |
| 503 Service Unavailable | Temporary unavailability (use with Retry-After) |

### Standard Error Response Format

```json
{
  "error": {
    "code": 400,
    "type": "ValidationError",
    "message": "Missing 'content' field in request body"
  }
}
```

### For Scheduled Retries:

```
HTTP/1.1 503 Service Unavailable
Retry-After: 120
```

---

## Authentication

Use token-based authentication to secure your API.

### Implementation:

- Issue a JWT or OAuth token upon login
- Every request must send:

```
Authorization: Bearer eyJhbGciOi...
```

- Failing to send or using an expired token results in:

```
HTTP/1.1 401 Unauthorized
```

---

## Security Considerations

### Media Upload Security

If media is user-uploaded:

- Enforce file size/type limits
- Scan for malware
- Use expiring signed URLs for private access (e.g., AWS S3 pre-signed URLs)

### API Security

- Always use HTTPS for all endpoints
- Validate and sanitize all inputs
- Implement CORS policies appropriately
- Use API keys or OAuth for authentication

---

## Optional Enhancements

- Add a media type validation API
- Support thumbnails or captions for video
- Allow reposting existing media
- Implement webhook notifications for post status updates
- Add analytics dashboards
- Support scheduling bulk posts
- Implement post templates

---

## Summary Table

| Concern | Recommendation |
|---------|---------------|
| **Versioning** | Use URL-based `/v1/...` paths |
| **Rate Limiting** | Return headers like `X-RateLimit-Remaining` |
| **Error Handling** | Use standard HTTP codes and JSON error bodies |
| **Retries** | Use `Retry-After` header on 503/429 |
| **Auth** | Use `Authorization: Bearer <token>` header |
| **Media** | Upload to your backend first for consistency |
| **Validation** | Validate all inputs and provide clear error messages |

## Media Upload API (`POST /v1/media`)

### API Type

`POST /v1/media` is a **RESTful HTTP API**, but media binaries are **not sent as JSON**.

Two upload modes are supported:

| Mode                       | When to Use                 |
| -------------------------- | --------------------------- |
| `multipart/form-data`      | Small images / short videos |
| Chunked (resumable) upload | Large media, mobile clients |

---

## Option A: Simple Upload (Small Media)

**Use case**

* Images
* Short videos
* < ~10–20MB

```
POST /v1/media
Content-Type: multipart/form-data
```

**Request**

```
file: binary
type: "image" | "video"
```

**Pros**

* Simple
* One request

**Cons**

* Not resumable
* Poor for mobile/network failures

---

## Option B: Chunked / Resumable Upload (Large Media)

For large uploads, media is uploaded via a **stateful upload session**.
This improves reliability, supports retries, and enables resume.

---

## Chunked Upload — Backend Managed

### Flow Overview

```
Client        Backend
  |              |
  | POST /uploads|
  |------------->|
  | uploadId     |
  |<-------------|
  |              |
  | PUT chunk 0  |
  |------------->|
  | PUT chunk 1  |
  |------------->|
  |     ...      |
  |              |
  | POST /complete|
  |------------->|
  | mediaId/url  |
  |<-------------|
```

---

### 1. Initiate Upload

```
POST /v1/media/uploads
```

```json
{
  "fileName": "video.mp4",
  "contentType": "video/mp4",
  "totalSize": 524288000
}
```

```json
{
  "uploadId": "upload_123",
  "chunkSize": 5242880
}
```

---

### 2. Upload Chunks

```
PUT /v1/media/uploads/{uploadId}/chunks/{index}
Content-Type: application/octet-stream
Content-Range: bytes start-end/total
```

**Client responsibilities**

* Retry failed chunks
* Persist `uploadId`
* Resume after app restart

---

### 3. Complete Upload

```
POST /v1/media/uploads/{uploadId}/complete
```

```json
{
  "mediaId": "media_123",
  "mediaUrl": "https://cdn.example.com/media_123.mp4"
}
```

---

## Chunked Upload — Pre-Signed URL (Recommended)

This offloads bandwidth to object storage (S3/GCS) and scales better.

---

### Flow Overview

```
Client        Backend        Storage
  |              |              |
  | POST /uploads|              |
  |------------->|              |
  | presigned URLs              |
  |<-------------|              |
  |              |              |
  | PUT part 1   |------------->|
  | PUT part 2   |------------->|
  |     ...      |              |
  |              |              |
  | POST /complete              |
  |------------->|              |
  | mediaId/url  |              |
  |<-------------|              |
```

---

### Key Properties

**Why this is preferred**

* Backend does not handle large payloads
* Parallel uploads
* Storage-level retries
* Industry standard (YouTube, Instagram, Twitter)

**Backend responsibilities**

* Validate uploaded parts
* Assemble final object
* Trigger media processing
* Persist `mediaId`

---

## Mobile Reliability Notes

Chunked uploads are critical due to:

* App backgrounding
* Network switches (Wi-Fi ↔ cellular)
* Intermittent connectivity

**Best practices**

* Persist `uploadId` locally
* Track completed chunk indexes
* Resume instead of restarting
* Exponential backoff per chunk

---

## Summary

| Scenario       | Approach              |
| -------------- | --------------------- |
| Small image    | `multipart/form-data` |
| Large media    | Chunked upload        |
| High traffic   | Pre-signed URLs       |
| Mobile clients | Resumable chunks      |

**Key takeaway:**
Media upload remains **REST**, but is implemented as a **protocol**, not a single request, to ensure scalability and reliability.
