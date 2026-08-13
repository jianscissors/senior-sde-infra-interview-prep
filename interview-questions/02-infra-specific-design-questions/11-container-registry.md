# Design a Container Registry (like a simplified Docker Hub/ECR)

## Clarifying questions to ask first
- Single-region or must pulls be fast globally (matters a lot for large fleets doing rolling deploys)?
- Public registry (anyone can pull) or private/multi-tenant with access control?
- Do we need vulnerability scanning or is that explicitly out of scope?

## Requirements
### Functional
- Push an image (a manifest referencing a set of layers) and pull it back by name+tag or digest.
- Layers are content-addressed and deduplicated — pushing an image that shares layers with an existing one shouldn't re-upload those layers.
- Support tagging (mutable pointer, e.g., `myapp:latest`) and immutable digest references.

### Non-functional
- Pull latency matters enormously at scale — a fleet-wide deploy can mean thousands of nodes pulling the same image near-simultaneously.
- Storage efficiency via deduplication, since layers are frequently shared across many images (base OS layers, common dependency layers).
- Strong durability for pushed images — losing a production image blocks every future deploy/rollback that depends on it.

## High-level architecture
1. **Manifest store**: metadata describing an image — list of layer digests, config blob, architecture/OS — keyed by name:tag or by content digest.
2. **Blob/layer store**: the actual layer tarballs, stored in object storage, addressed purely by content hash (digest).
3. **Push/pull API**: implements the registry HTTP API (the same protocol Docker/OCI clients speak) — `HEAD` to check if a blob already exists before uploading, chunked upload for large layers, manifest upload as the final step that ties layers together into a pullable image.
4. **Garbage collector**: async job that finds and reclaims blobs no longer referenced by any manifest.
5. **Replication layer**: pushes/mirrors popular images to regional caches for low-latency pulls.

## Deep dives

### Content-addressable storage (dedup by digest)
- Every layer is stored keyed by the SHA256 digest of its contents. Before uploading a layer, the client (or registry) checks whether a blob with that digest already exists — if so, the upload is skipped entirely. This means pushing a new image that only changes the top application layer (common case: base OS + deps layers unchanged, just the app code layer is new) only actually uploads that one new layer, not the whole image. This single property is what makes registries storage-efficient at scale, since most organizations' images share the vast majority of their layers.

### Manifest/layer separation
- A manifest is a small JSON document listing layer digests in order plus a config blob (entrypoint, env vars, etc.) — it's the "recipe," not the data itself. This separation means a tag (`myapp:v2`) can be repointed to a different manifest cheaply (a small metadata write), without touching any of the actual layer bytes, and multiple tags/images can reference the exact same underlying layers with zero duplication.

### Push/pull protocol
- Push: client checks blob existence (`HEAD /blobs/<digest>`) for each layer, uploads only missing ones (often via chunked/resumable upload for large layers), then uploads the manifest last — the manifest upload is the atomic "publish" moment, since a manifest referencing layers that all now exist is what makes an image pullable.
- Pull: client fetches the manifest by tag or digest, then fetches each referenced layer blob (in parallel) if not already cached locally, and assembles the image filesystem via the local container runtime's layer overlay.

### Garbage collection of untagged/orphaned layers
- Deleting a tag doesn't delete the underlying layers immediately, since other manifests may still reference them. GC runs as a two-phase mark-and-sweep: mark every blob digest reachable from any current manifest, then delete any unmarked blob. Must handle the race of a push happening concurrently with a GC sweep — typically solved with a short grace period (don't delete anything uploaded very recently) or by pausing new manifest writes during the mark phase.

### Replication across regions for pull latency
- Pushing directly writes to a home region's storage; a background replication job (or an on-demand pull-through cache) mirrors images to regional caches close to where fleets actually pull from, so a rolling deploy across thousands of nodes in a region doesn't all hit a single distant origin. Pull-through caching (fetch-and-cache on first miss, serve from regional cache after) is often preferred over eager full replication since most images are only ever pulled in a subset of regions.

### Vulnerability scanning pipeline
- Implemented as an async hook triggered on push — scans layers against a CVE database and attaches results as metadata associated with the manifest (not blocking the push itself, since scanning can take time). Policy enforcement (e.g., "block deploy of images with critical CVEs") is a separate consumer of that scan result, applied at pull/deploy time rather than push time.

## Key tradeoffs
- Eager replication (copy to every region immediately) minimizes pull latency everywhere but wastes storage/bandwidth on images that are never pulled in most regions; pull-through caching is more storage-efficient but has a cold-start latency penalty on first pull in a new region.
- Deleting a tag doesn't reclaim space immediately (GC lag) — simpler and safer (avoids racing an in-flight push) at the cost of a delay before storage is actually freed.

## Likely follow-ups
- "10,000 nodes need to pull the same new image within seconds during a deploy — how do you avoid melting the registry?" → regional caching/CDN in front of blob storage, plus registries at massive fleet scale often use peer-to-peer layer distribution between nodes (nodes that already pulled a layer serve it to others) to avoid a thundering herd on origin.
- "How do you handle a client pushing a manifest that references a layer digest that doesn't actually exist?" → reject the manifest upload at that point (validate all referenced digests exist before accepting the manifest) — the manifest write is the atomicity boundary, so it must fail closed on missing dependencies.
