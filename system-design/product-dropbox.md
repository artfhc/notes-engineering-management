# Design Dropbox / Google Drive

## 1. Clarify the Scope

Start with questions like:

- Are we designing personal cloud storage or collaborative docs too?
- Do we need real-time collaboration like Google Docs?
- Is the focus on file storage + sync?
- Do we support folder hierarchy, sharing, version history?

**A strong assumption:** I'll focus on a Dropbox-style file storage and sync system: upload/download files, folder organization, multi-device sync, version history, and shared links. I won't include Google Docs-style live co-editing in the initial scope.

## 2. Functional Requirements

- Upload files
- Download files
- Sync file changes across devices
- Folder hierarchy
- Rename / move / delete files
- Basic sharing
- Version history

**Non-Functional Requirements:**

- High durability
- High availability
- Efficient large-file transfer
- Incremental sync
- Reasonable conflict handling
- Good mobile / desktop experience

## 3. Core Idea

Split the system into two major parts:

**A. Metadata Plane** — stores:
- File names
- Folder structure
- Ownership
- Permissions
- Versions
- Checksums
- Mapping from logical file to physical chunks

**B. Blob / Chunk Storage Plane** — stores:
- Actual file contents
- Chunks / objects
- Replicated durable blobs

> This separation is one of the most important things to say.

## 4. High-Level Architecture

```
                +----------------------+
                | Client App           |
                | Web / Mobile / Desk  |
                +----------------------+
                           |
                           v
                +----------------------+
                | API Gateway / Auth   |
                +----------------------+
                     /            \
                    v              v
      +----------------------+   +----------------------+
      | Metadata Service     |   | Upload Session Svc   |
      | files/folders/share  |   | presigned URLs/chunk |
      +----------------------+   +----------------------+
                |                              |
                v                              v
      +----------------------+      +----------------------+
      | Metadata DB          |      | Blob / Object Store  |
      | file tree, versions  |      | chunks, replicas     |
      +----------------------+      +----------------------+
                |
                v
      +----------------------+
      | Change Log / Notify  |
      | sync events          |
      +----------------------+
                |
                v
      +----------------------+
      | Push / Sync Service  |
      +----------------------+
```

## 5. Simple Interview Diagram

This is the version you can quickly draw on a whiteboard:

```
[Client]
   |
   v
[API Gateway]
   | \
   |  \
   v   v
[Metadata Service]    [Upload Service]
   |                      |
   v                      v
[Metadata DB]         [Blob Storage]
   |
   v
[Change Log / Notification Service]
   |
   v
[Other Devices Sync]
```

## 6. Core Entities

| Entity | Fields |
|--------|--------|
| User | id |
| File | id, owner_id, name, size, checksum |
| Folder | id, parent_id, owner_id |
| FileVersion | file_id, version, chunk_refs |
| Chunk | chunk_id, hash, blob_path |
| Share | file_id/folder_id, permissions |

## 7. Upload Flow

This is the most important flow to explain clearly.

**Upload Steps:**

1. Client asks metadata/upload service to start upload
2. Server creates an upload session
3. Large file is split into chunks
4. Client uploads chunks directly to blob storage
5. Server validates checksums
6. Metadata service commits a new file version
7. Change log emits sync event
8. Other devices learn about new version

**Diagram:**

```
Client
  |
  v
Start Upload
  |
  v
Upload Service ----> returns upload session / chunk instructions
  |
  v
Client uploads chunks ----> Blob Storage
  |
  v
Metadata Service commits file version
  |
  v
Change Log / Sync Event
```

## 8. Download Flow

1. Client requests file metadata
2. Metadata service returns latest version + chunk references
3. Client downloads chunks from blob/object storage
4. Client reassembles file locally
5. Local cache updated

**Diagram:**

```
Client -> Metadata Service -> get file version + chunk refs
Client -> Blob Storage -> download chunks
Client -> reconstruct file
```

## 9. Important Design Choice: Chunking

Large files should not be stored as one giant blob only.

Use chunking because it helps with:

- Resumable uploads
- Retries on partial failure
- Deduplication
- Incremental sync
- Efficient large-file handling

**Good thing to say:** I'd chunk large files and track chunk hashes in metadata, which improves resumability and enables deduplication.

## 10. Deduplication

A common interview deep dive.

If two files or versions share identical chunks:
- Store chunk once
- Reference it from many file versions
- Usually done using content hash

**Pros:**
- Saves storage
- Saves bandwidth

**Cons:**
- Added metadata complexity
- Privacy/security concerns if global dedupe is too aggressive

**Strong answer:** I'd start with per-user or per-tenant dedupe to reduce complexity and privacy risk, then consider broader dedupe later.

## 11. Sync Across Devices

This is the second most important topic.

Each device needs to know:
- What changed
- What version it has
- How to reconcile

**Typical design:**
- Metadata changes appended to a change log
- Devices poll or keep long-lived sync channels
- Device sends last synced cursor
- Server returns deltas since that cursor

**Sync diagram:**

```
Device A changes file
   |
   v
Metadata Service
   |
   v
Change Log
   |
   v
Device B asks: "what changed since cursor X?"
   |
   v
Server returns delta updates
```

## 12. Conflict Resolution

Classic interview question — what if two devices edit the same file offline and both sync later?

For Dropbox-style file sync:
- Use version numbers / revision IDs
- If same base version diverges, create conflicted copies
- User resolves manually

**Good answer:** For binary files, I would not attempt smart merging initially. I'd detect version conflicts and create conflicted copies, since correctness matters more than automatic merging.

> That is a very strong and practical answer.

## 13. Sharing

Basic sharing can be modeled with:
- ACL / permission table
- Public signed links
- Folder/file level permissions

Mention: owner, viewer/editor roles, revoked links, tokenized share URLs.

## 14. Version History

Store:
- Logical file ID
- Multiple immutable versions
- Each version points to chunk list

**Benefits:** rollback, auditability, recovery from accidental overwrite.

## 15. Scaling Concerns

**Hotspots:**
- Huge file uploads
- Frequent sync checks
- Hot shared folders
- Popular shared links

**Solutions:**
- Direct-to-object-store uploads
- CDN for downloads
- Cache metadata reads
- Delta sync instead of full rescan
- Partition metadata by user / namespace

## 16. Reliability / Failure Modes

| Failure | Solution |
|---------|----------|
| Upload interrupted | Resumable chunk upload |
| Metadata commit succeeds but notification delayed | Devices catch up later through cursor-based sync |
| Duplicate uploads due to retry | Idempotent upload session / chunk hash checking |
| Device offline for a long time | Replay deltas from sync cursor |

## 17. Security

Mention briefly:
- Authenticated access
- Encrypted in transit
- Encrypted at rest
- Permission checks on metadata and download URLs
- Audit logging for shares

## 18. What to Draw in the Interview

This is the clean version to memorize:

```
                [Client]
                   |
                   v
             [API Gateway]
              /          \
             v            v
 [Metadata Service]   [Upload Service]
         |                  |
         v                  v
   [Metadata DB]      [Blob Storage]
         |
         v
 [Change Log / Sync Service]
         |
         v
   [Other Devices]
```

Then add one deep dive:

**Upload deep dive:**
```
Client -> Upload Service -> get upload session
Client -> Blob Storage -> upload chunks
Metadata Service -> commit new version
Change Log -> notify devices
```

**Sync deep dive:**
```
Device -> Sync Service with cursor
Sync Service -> return changed files/versions
Device -> fetch new chunks if needed
```

## 19. Short 30-Second Summary

I'd separate metadata from blob storage. Metadata service manages files, folders, versions, and permissions, while object storage holds chunked file contents. Clients upload and download chunks directly, and a change-log-based sync service propagates updates across devices. For conflicts, I'd use version IDs and create conflicted copies for binary files.

## 20. How This Maps to Common Patterns

Dropbox / Drive combines several reusable patterns:

| Pattern | Where it appears |
|---------|-----------------|
| Client → API → Service → DB | overall architecture |
| Read vs Write path | upload vs sync/download |
| Async event pipeline | change notifications |
| Realtime / sync propagation | multi-device updates |
| Metadata + blob separation | core storage design |

## 21. What Interviewers Often Ask Next

Be ready for these follow-ups:

- How do you support resumable uploads?
- How do you handle conflicts?
- How would you design version history?
- How do devices know what changed?
- How do you support very large files?
- How do you optimize mobile bandwidth?
- How do you design deduplication?

## 22. Quick Whiteboard Version

If you want the fastest possible sketch:

```
[Client]
   |
[API]
 /   \
v     v
[Metadata Svc]   [Upload Svc]
   |                 |
   v                 v
[Metadata DB]     [Blob Store]
   |
   v
[Change Log / Sync]
   |
   v
[Other Devices]
```

## 23. Mobile-Specific Angle You Can Add

This is where you can leverage your background:

> On mobile, I'd also think carefully about background sync policies, bandwidth constraints, battery usage, resumable uploads, and local cache consistency, because the user experience depends heavily on those client-side realities.

That makes your answer more differentiated.
