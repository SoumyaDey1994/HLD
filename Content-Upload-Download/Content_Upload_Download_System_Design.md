# Content Upload & Download System Design

## 1. Content Upload Flow

```text
Client
  ↓
Backend / Media Service
  ↓
Create Upload Session
  ↓
Generate Signed / Pre-Signed Upload URL
  ↓
Client splits file into chunks
  ↓
Direct Multipart Upload → Object Storage (S3 / Azure Blob)
  ↓
Each successful chunk → Client maintains checkpoint
  ↓
All chunks uploaded
  ↓
Backend finalizes multipart upload
  ↓
Object Storage confirms object creation
  ↓
Backend stores object key + metadata in Metadata DB
  ↓
Virus Scanner
  ↓
CLEAN → Content becomes AVAILABLE
INFECTED → Quarantine/Delete + mark REJECTED
```

### Step-by-Step

1. **Client → Backend:** Request upload with filename, size, content type, etc.
2. **Backend:** Authenticate/authorize and validate file constraints.
3. **Backend:** Create an `upload_id` and upload session.
4. **Backend:** Generate **Signed / Pre-Signed Upload URL(s)** for Object Storage.
5. **Client:** Split large file into chunks and upload directly to Object Storage using multipart upload.
6. **Client:** After each successful chunk, store its checkpoint/part number + ETag.
7. **Completion:** After all chunks succeed, Client asks Backend to finalize the upload.
8. **Backend → Object Storage:** Complete multipart upload.
9. **Backend → Metadata DB:** Store metadata and stable `object_key`; **do not store the temporary signed URL**.
10. **Virus Scanner:** Scan the uploaded object.
11. **Clean:** Mark content `AVAILABLE`.
12. **Malicious:** Quarantine/delete object and mark content `REJECTED`.

---

# 2. Content Download Flow

## A. Download Through Object Storage (If no CDN involved)

```text
Client
  ↓
Backend / Media Service
  ↓
Metadata DB
  ↓
Get object_key
  ↓
Generate Signed / Pre-Signed GET URL
  ↓
Client
  ↓
Object Storage
  ↓
Verify Signature + Expiry + Permissions
  ↓
Content
```

### Steps

1. **Client → Backend:** Request content using `media_id`.
2. **Backend → Metadata DB:** Retrieve the `object_key`.
3. **Backend:** Generate a short-lived **Signed / Pre-Signed GET URL**.
4. **Client:** Sends GET request directly to Object Storage.
5. **Object Storage:** Validates:
   - Signature
   - Expiration
   - HTTP method / object key
   - Access permissions
6. If valid → **Object Storage returns content**.
7. If invalid/expired → `403 Access Denied`.

---

## B. Download Through CDN

```text
Client
  ↓
Backend / Media Service
  ↓
Metadata DB
  ↓
Get object_key
  ↓
Generate CDN Signed URL / Token
  ↓
Client
  ↓
CDN
  │
  ├── Cache Hit ──────► Return Content
  │
  └── Cache Miss
          ↓
     Object Storage
          ↓
     Fetch Content
          ↓
     CDN caches Content
          ↓
     Return Content
```

### Steps

1. **Client → Backend:** Request content.
2. **Backend → Metadata DB:** Retrieve the `object_key`.
3. **Backend:** Generate an authorized **CDN Signed URL / Token**.
4. **Client → CDN:** Request the content.
5. **CDN:** Validates:
   - Signature/token
   - Expiration
   - Access restrictions
6. **Cache Hit:** CDN directly returns the cached content to Client.
7. **Cache Miss:** CDN fetches the content from **Object Storage**.
8. CDN caches the content and returns it to Client.
9. Object Storage can additionally enforce origin-access controls so that only the CDN can access the underlying objects.

---

# 3. Retry / Resume — Failure & Recovery

### Upload Failure

```text
Chunk 1 ✓
Chunk 2 ✓
Chunk 3 ✗
Chunk 4 ✓
```

- Client retries **only failed Chunk 3**.
- Use exponential backoff + limited retries.
- Successfully uploaded chunks are not re-uploaded.

### Resume After Client / Network Failure

```text
Chunk 1 ✓
Chunk 2 ✓
Chunk 3 ✓
Chunk 4 ✓
     ↓
Client / Network Failure
     ↓
Resume using upload_id + checkpoints
     ↓
Upload only missing chunks
```

- If the signed upload URL expires, Backend generates a **new URL** for the remaining chunks.

### Download Failure

- Client retries the failed request.
- For large media, use **HTTP Range Requests** to resume from the last downloaded byte.
- If the signed URL expires, Client requests a **new signed URL** from Backend and resumes the download.

---

# 4. Core Responsibilities

| Component | Responsibility |
|---|---|
| **Client** | Chunking, parallel upload, checkpointing, retry/resume, download |
| **Backend / Media Service** | Authentication, validation, upload session, signed URL generation, metadata management |
| **Metadata DB** | File metadata, `object_key`, upload/content status |
| **Object Storage** | Durable media storage, multipart upload, signature validation |
| **Virus Scanner** | Malware detection and quarantine/rejection |
| **CDN** | Content caching, delivery, and signed URL/token validation |

---

# 5. Metadata DB Table Structure
```
media_id
user_id
file_name
content_type
file_size_bytes
storage_provider
bucket_name
object_key
status
virus_scan_status
created_at
updated_at
```
Example:
```
| Column              | Example Value                               |
| ------------------- | ------------------------------------------- |
| `media_id`          | `550e8400-e29b-41d4-a716-446655440000`      |
| `user_id`           | `USR_10245`                                 |
| `file_name`         | `holiday_video.mp4`                         |
| `content_type`      | `video/mp4`                                 |
| `file_size_bytes`   | `104857600`                                 |
| `storage_provider`  | `S3`                                        |
| `bucket_name`       | `production-media`                          |
| `object_key`        | `raw-media/USR_10245/550e8400/original.mp4` |
| `status`            | `AVAILABLE`                                 |
| `virus_scan_status` | `CLEAN`                                     |
| `created_at`        | `2026-09-19 02:15:30`                       |
| `updated_at`        | `2026-09-19 02:18:45`                       |

```
Conceptually;
```
Media ID
   │
   ├── Owner → USR_10245
   ├── File  → holiday_video.mp4
   ├── Size  → 100 MB
   │
   └── Storage
        ├── Provider → S3
        ├── Bucket   → production-media
        └── Key      → raw-media/USR_10245/550e8400/original.mp4
```
So when the user later requests:
`GET /media/550e8400-e29b-41d4-a716-446655440000`

Backend looks up the object_key, generates a short-lived signed/pre-signed GET URL, and returns it to the client. The URL itself is not stored in this table.

The object is identified as:
`s3://production-media/raw-media/USR_10245/550e8400/original.mp4`

A pre-signed S3 URL would typically look conceptually like:
```
https://production-media.s3.amazonaws.com/
    raw-media/USR_10245/550e8400/original.mp4
    ?X-Amz-Algorithm=...
    &X-Amz-Credential=...
    &X-Amz-Expires=...
    &X-Amz-Signature=...
```

---

# 5. Overall Architecture

```text
                         UPLOAD
                           │
                           ▼
Client ──► Backend ──► Signed URL
                           │
                           ▼
                    Object Storage
                    (Multipart Upload)
                           │
                           ▼
                     Virus Scanner
                           │
                     ┌─────┴─────┐
                     │           │
                   CLEAN       INFECTED
                     │           │
                     ▼           ▼
                  AVAILABLE    REJECTED
                     │
                     ▼
                 Metadata DB
                 (object_key)


                         DOWNLOAD
                           │
                           ▼
Client ──► Backend ──► Metadata DB
                           │
                       object_key
                           │
                           ▼
                  Signed URL / Token
                           │
                           ▼
                         CDN
                      ┌────┴────┐
                      │         │
                  Cache Hit  Cache Miss
                      │         │
                      ▼         ▼
                   Client   Object Storage
                                │
                                ▼
                              CDN
                                │
                         Cache + Return
                                │
                                ▼
                              Client
```

## Key Design Principles

- **Media bytes → Object Storage**, not Metadata DB.
- **Metadata + object key → Metadata DB**.
- **Signed URLs are short-lived** and generated at runtime.
- **Client uploads directly to Object Storage**, avoiding Backend bandwidth bottlenecks.
- **Chunk-level checkpoints** enable efficient retry/resume.
- **Virus scanning happens before content becomes available.**
- **CDN handles high-volume downloads.**
- **Cache Hit → CDN serves content directly.**
- **Cache Miss → CDN fetches from Object Storage, caches it, then serves Client.**
- **Download/upload signed URL validation is performed by the service receiving the signed request** — Object Storage for storage URLs, CDN for CDN URLs.