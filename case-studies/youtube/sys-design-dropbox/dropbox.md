# System Design: Dropbox

## Interview Roadmap

For user-facing product design questions, follow this structure:

1. **Functional Requirements** — What does the system do?
2. **Non-Functional Requirements** — How well does it do it?
3. **Core Entities** — What data is persisted / exchanged?
4. **API Design** — How do clients interact with the system?
5. **High-Level Design** — Components, services, and data flows
6. **Deep Dives** — Scalability, fault tolerance, non-functional concerns

---

## 1. Functional Requirements

Focus on the top 3 core features:

| # | Feature | Description |
|---|---------|-------------|
| 1 | **Upload** | Upload files to remote storage |
| 2 | **Download** | Download files from remote storage |
| 3 | **Auto-Sync** | Automatically sync files across multiple devices |

**Sync behavior:** A file added or modified locally → uploaded to remote storage → propagated to all other connected devices.

**Out of scope:** Designing blob storage itself (e.g., building S3 from scratch).

---

## 2. Non-Functional Requirements

Critical for senior+ candidates. Each requirement shapes the deep dives.

| Requirement | Detail |
|-------------|--------|
| **Availability > Consistency** | Prioritize availability; accept eventual consistency. A user in the US seeing a slightly stale file updated in Germany is acceptable. |
| **Low Latency** | Uploads and downloads should be as fast as possible. |
| **Large File Support** | Up to **50 GB** per file (matches real Dropbox limits). |
| **Resumable Uploads** | Interrupted uploads must resume without restarting. |
| **High Data Integrity** | Sync must be accurate. Eventual consistency is fine; incorrect data is not. |
| **Scale** | Hundreds of millions of daily active users. |

> **On estimations:** Defer back-of-the-envelope math until it directly impacts a design decision. Communicate this approach to your interviewer upfront.

---

## 3. Core Entities

| Entity | Description |
|--------|-------------|
| **File** | Raw bytes stored in blob storage (e.g., S3) |
| **File Metadata** | `fileId`, `name`, `mimeType`, `size`, `ownerId`, `s3Link`, `createdAt`, `updatedAt` |
| **User** | Present but not a focus — identity inferred from auth token |

---

## 4. API Design

Initial simplified endpoints aligned to functional requirements:

| Endpoint | Method | Input | Output | Purpose |
|----------|--------|-------|--------|---------|
| `/files` | `POST` | File bytes + Metadata | `200 OK` | Upload a file |
| `/files/{fileId}` | `GET` | `fileId` | File + Metadata | Download a file |
| `/changes` | `GET` | `lastSyncTimestamp` | List of changed `fileId`s | Sync changed files |

**Notes:**
- User identity is derived from JWT/session auth tokens — not passed in the request body.
- These APIs are **placeholders** — they evolve significantly in the deep dives (chunked uploads, pre-signed URLs, delta sync).

---

## 5. High-Level Design

### Components

```
Client (Local Folder + Sync App)
    ↓
API Gateway / Load Balancer
(auth, rate limiting, SSL termination, routing)
    ↓
┌─────────────────┐     ┌──────────────────┐
│   File Service  │────▶│  Blob Storage    │
│  (metadata DB)  │     │     (S3)         │
└─────────────────┘     └──────────────────┘
         │
┌─────────────────┐
│  Sync Service   │
│ (getChanges API)│
└─────────────────┘
```

### Upload Flow

1. Client sends file bytes + metadata to **File Service** via `POST /files`.
2. File service stores raw bytes in **S3** and metadata in the database.
3. Returns success/failure to client.

### Download Flow

1. Client calls `GET /files/{fileId}` → File Service returns metadata including **S3 link**.
2. Client downloads file **directly from S3** — avoids proxying through the service.

### Sync Flow

**Remote → Local:**
- Client app polls **Sync Service** via `GET /changes?since=<timestamp>`.
- Sync service returns list of changed file IDs/metadata.
- Client compares against its local DB, downloads updated files from S3.

**Local → Remote:**
- Client app monitors local folder using OS file system APIs:
  - Windows: **File System Watcher**
  - macOS: **FS Events**
- Detected changes trigger the standard **upload flow**.

**Client-side components:**
- **Local folder:** Files on disk.
- **Local DB:** Tracks file metadata and sync state (prevents redundant uploads/downloads).

---

## 6. Deep Dives

### Deep Dive 1 — Large Files & Resumable Uploads (up to 50 GB)

**Problems with the naive design:**

- **Bandwidth waste:** Data proxies through the file service before reaching S3.
- **Request size limits:** API gateways typically limit request bodies to ~10 MB. A 50 GB file upload fails entirely.
- **No resumability:** A failed upload means starting from scratch.

**Solution: Pre-signed URLs + Chunking**

#### Step-by-Step Upload Flow

1. Client sends **metadata only** to File Service.
2. File Service generates a **time-limited pre-signed URL** per chunk and returns them to the client.
3. Client splits file into chunks (e.g., **5 MB each**).
4. Client uploads each chunk **directly to S3** via pre-signed URLs — serially or **in parallel**.
5. Client reports upload status per chunk to File Service.
6. File Service **verifies** with S3 (or uses S3 event notifications) before marking chunks as complete.

#### Chunk Tracking in Metadata

```
File Metadata:
  fileId, name, size, mimeType, ownerId, s3Link
  chunks: [
    { chunkId, fingerprint, s3Link, status: "uploaded" | "pending" },
    ...
  ]
```

#### Fingerprinting

- Each chunk is **hashed** (e.g., MD5, SHA-256) to produce a unique fingerprint.
- On resume: client compares local chunk fingerprints against uploaded chunk records and uploads **only missing chunks**.

#### Verifying Upload Status

| Approach | Description | Risk |
|----------|-------------|------|
| **Client reports** (naive) | Client tells service a chunk is done | Client can lie or fail mid-report |
| **Trust but verify** ✅ | Client reports; service cross-checks with S3 | Balanced — recommended |
| **S3 Notifications** | S3 asynchronously notifies service on chunk upload | Most reliable, eventual consistency |

> Note: S3's native multipart upload API supports chunking but may not support per-chunk event notifications.

---

### Deep Dive 2 — Low Latency Uploads & Downloads

| Technique | Benefit | Trade-off |
|-----------|---------|-----------|
| **Parallel chunk uploads** | Maximizes available bandwidth | More complex client logic |
| **Adaptive chunk sizing** | Smaller chunks on poor connections | More overhead on fast connections |
| **CDN** | Faster downloads for globally distributed access | Dropbox files are mostly user-private; CDN benefit is limited. Adds cost and complexity. |
| **Compression** | Reduces data transfer size | CPU overhead; only effective for compressible types (text, not JPEG/MP4). Algorithm stored in metadata. |

**CDN verdict for Dropbox:** Users primarily access their own files from nearby data centers. CDN ROI is lower than for public content delivery. Include only if usage patterns justify cost.

---

### Deep Dive 3 — High Data Integrity & Sync Accuracy

Two goals:
- **Fast sync** — Changes propagate quickly.
- **Consistent sync** — Local and remote state remain accurate.

#### Making Sync Fast

- **Adaptive polling:** Poll more frequently when the client is actively using the app; back off during idle periods.
- Avoid over-engineering: WebSockets and long polling add complexity with minimal benefit for sync use cases.
- Provide **manual refresh** as an escape hatch for immediate sync.

#### Reducing Data Transferred — Delta Sync

- Instead of re-downloading entire files, only download **changed chunks**.
- Client uses chunk fingerprints to identify what changed and reconstructs the file locally.

#### Consistency Strategies

| Approach | Description | Best For |
|----------|-------------|----------|
| **Poll metadata DB** | Query for files/chunks updated since last sync timestamp | Simple; sufficient for most use cases |
| **Event bus with cursor** (Dropbox's approach) | Kafka stream of change events with a per-folder cursor position | Audit trails, versioning, replay, rollback — adds significant complexity |

> **Recommendation:** Start with polling unless explicit requirements for versioning/audit trail are given. An event bus is powerful but overkill without those requirements.

#### Reconciliation

- Periodically (daily/weekly), client performs a **full reconciliation**:
  - Fetch all remote metadata.
  - Compare against local files and fingerprints.
  - Fix any inconsistencies from network failures, bugs, or data corruption.

---

## Updated API (Post Deep Dives)

### Upload Flow (Chunked)

```
1. POST /files/metadata       → File Service returns pre-signed URLs per chunk
2. PUT  <pre-signed-url>      → Client uploads each chunk directly to S3
3. POST /files/{fileId}/chunks/{chunkId}/confirm  → Client notifies File Service of upload
4. File Service verifies with S3 → marks chunk as uploaded
```

### Sync Flow

```
GET /changes?since=<timestamp>
→ Returns list of changed file IDs + chunk-level metadata
→ Client downloads only changed chunks (delta sync)
```

---
