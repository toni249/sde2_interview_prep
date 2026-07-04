# Pattern 6 — Handling Large Blobs

> **Problem:** Files bigger than a few MB (videos, images, PDFs, ML models, backups) don't belong in your API request pipeline. Streaming a 5 GB video through your app server ties up memory, blocks a worker thread, and racks up egress bills.

## The Golden Rule

> **Never route bytes through your app server.** Push the client to talk to blob storage directly. The app server only issues *permission* (a signed URL) and *records metadata* (in the DB).

## Presigned URLs — The Core Pattern

```
1. Client → API: "I want to upload photo.jpg (2 MB, image/jpeg)"
2. API validates, then generates a presigned PUT URL (signed by S3 with 15-min expiry)
   API → DB: INSERT upload {id, userId, key, status='pending'}
   API → Client: {uploadUrl, uploadId}

3. Client → S3: PUT uploadUrl <bytes>          (bypasses app server)

4. Client → API: "done, uploadId=X"           (or S3 → EventBridge notifies)
   API → S3 HEAD to verify presence and size
   API → DB: UPDATE upload SET status='ready'
```

**Why this works:**
- Bandwidth stays between client ↔ S3 (or Cloud Storage / Azure Blob).
- App server stays lightweight; scaling is decoupled from upload volume.
- Signed URL is time-boxed and scoped (bucket, key, method, content-type, max size).

### Presigned URL security constraints
- Include `Content-Length` and `Content-MD5` in the sign to prevent tampering.
- Restrict `Content-Type` so users can't upload arbitrary executables to a "photos" bucket.
- Short expiry (5-15 min).
- **Never** sign a URL for the whole bucket — one key only.
- Enable S3 Object Lock or bucket policies to disallow overwrites of finalized objects.

## Multipart Uploads

For files > ~100 MB, single PUT fails on flaky networks. Multipart splits into parts:

```
1. Client → API: initiate → API returns S3 uploadId + presigned URLs per part
2. Client → S3: PUT part 1, PUT part 2, ...  (parallel, resumable)
3. Client → API: complete {uploadId, partList[eTag, partNum]}
4. API → S3: CompleteMultipartUpload → single object
```

- Failed part? Retry that part only.
- Pause/resume support in the client SDK.
- S3 aborts incomplete uploads via lifecycle rule (avoids "orphan parts" bill).

## Resumable Uploads (tus / Google Resumable)

Alternative to S3 multipart when your storage doesn't support it natively:
- Client sends `Content-Range: bytes 0-999999/5000000`.
- Server appends to the object. On failure, client queries current offset and resumes.

## Downloading

Same shape in reverse:
```
Client → API: "give me video 42"
API validates entitlement → API returns presigned GET URL (or CDN URL)
Client → CDN → S3: GET
```
- **CDN in front is essential**: origin egress is expensive; CDN cache hits are near free.
- Signed CDN URLs (CloudFront, Fastly) let you enforce auth without hitting origin.
- **Range requests** — clients (video players) request byte ranges. Enable on CDN + S3.

## Storage Tier Choices

| Tier | Latency | Cost | Use |
|------|---------|------|-----|
| **S3 Standard** | ms | $$$ | Hot access, e.g. active user avatars |
| **S3 Standard-IA** | ms | $$ | Infrequently accessed, 30-day access charge |
| **S3 Intelligent-Tiering** | ms | $$ | Unknown pattern, auto-moves |
| **S3 Glacier Instant** | ms | $ | Rare access, low retrieval fee |
| **S3 Glacier Flexible / Deep Archive** | minutes-hours | $ / ¢ | Long-term retention, backups |

Lifecycle rule: transition after 30d → IA, 90d → Glacier.

## Media-Specific Handling

**Videos**:
- Store the master (H.264 4K).
- Transcode to multiple bitrates (pattern 2 — long-running task).
- Serve via **HLS or DASH** (`.m3u8` manifest + `.ts` segments) so player adapts to bandwidth.
- CDN caches segments — huge cache hit rate since segments are immutable.

**Images**:
- Store original once.
- Generate resized variants on-the-fly (imgproxy, Cloudinary) via CDN with `w=800&h=600`.
- Or pre-generate common sizes on upload.

## Reference Architecture — Photo Sharing App

```
Upload:
  Client → API: /photos, {mime, size}
  API → Client: presigned URL + photoId
  Client → S3: direct PUT
  S3 → SNS → Lambda thumbnailer → writes thumb.jpg alongside original
  Lambda → DynamoDB: UPDATE photo status='ready', variants=[thumb, medium, full]

View:
  Client → API: /photos/{id}
  API → DynamoDB → returns CDN URLs (signed if private)
  Client → CloudFront → S3 origin (cache miss) → S3
```

**Why this works at scale:**
- App servers never touch bytes; auto-scale API tier on RPS not GB.
- Thumbnailing is async so upload UX is instant.
- CDN absorbs 99% of read traffic.

## Interview Q&A

**Q: Why not just proxy uploads through the server for simplicity?**
- Doubles egress cost (client → server → S3).
- App server memory pressure — a 1 GB upload holds 1 GB heap or streams to disk (slow).
- Timeouts on load balancers (ALB 60s idle) kill big uploads.
- Blocks a thread/worker per upload — kills concurrency.

**Q: How do you validate the file after upload (virus scan, content check)?**
- S3 event → Lambda / worker scans → moves object from `staging/` prefix to `clean/` prefix (or deletes).
- App reads from `clean/` only. DB status: `SCANNING → READY / REJECTED`.

**Q: How do you prevent users from uploading arbitrarily large files?**
- Presigned URL includes `Content-Length` constraint or `content-length-range` in policy.
- Enforce max size at CDN / WAF layer.
- Post-upload check in the S3 event handler — delete oversize objects.

**Q: How do you resume a broken upload?**
- Multipart upload — client re-requests missing part URLs and PUTs those parts.
- tus protocol — client asks server for current offset, resumes from there.

**Q: How do you deduplicate identical files?**
- Compute content hash (SHA-256) client-side → API checks if `blob_hash` exists → if yes, don't re-upload, just link.
- Space savings for common files (memes, common attachments).

**Q: How do you handle private files (only owner can view)?**
- **Presigned GET URL** with short expiry — user's browser gets a fresh URL each session.
- Signed CDN URL (CloudFront Signed URLs / Cookies) so the CDN still caches by key but validates access via signature.
- Reverse proxy with auth check → 302 redirect to signed URL.

**Q: What about GDPR "right to delete"?**
- DB row deleted; object also deleted (S3 delete + lifecycle purge for versioned buckets).
- Ensure backups also expire; some regulators accept "encrypted with a key we destroy".

**Q: Streaming vs downloadable?**
- Streaming = HLS/DASH, chunked, adaptive bitrate, CDN caches segments.
- Downloadable = single object with `Content-Disposition: attachment` header.

## Real-World Examples
- **YouTube**: chunked upload via resumable protocol → GCS → multi-bitrate transcode → CDN.
- **Google Drive**: resumable upload API; per-user shard for metadata.
- **Instagram**: direct S3 upload from mobile; async CV pipeline for tags.
- **Zoom recordings**: uploaded post-call to S3; on-demand transcode when user first views.
- **Dropbox**: block-level dedup — file split into 4 MB blocks, each hashed and stored once.

## Gotchas
- **CORS on S3 bucket**: browser uploads fail without a permissive CORS policy. `AllowedMethods: [PUT], AllowedOrigins: [https://app.com]`.
- **Clock skew**: presigned URL expiry uses UTC; client with skewed clock can't upload. Use HTTPS + NTP.
- **Egress bills**: unexpected cross-region reads. Prefer same-region VPC endpoints.
- **Orphaned multipart uploads**: silently bill until aborted. Lifecycle rule: `AbortIncompleteMultipartUpload` after 7 days.
- **Immutable content addressability**: if you name objects `photo42.jpg` and re-upload a new version, CDN serves stale. Use versioned keys (`photo42-v3.jpg`) or immutable content hashes.
