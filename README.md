# Ship Bytes publications

Production content for Ship Bytes. The `master` branch is imported Tuesday and Friday at approximately 06:00 Europe/Stockholm, with checks at 07:00, 08:00 and 09:00 if nothing was published.

One issue per `issues/YYYY/YYYY-MM-DD.json`, with `schema_version: 1`, a unique slug, title, subject, published_date, intro and nonempty stories. Stories require unique slug, title, byte, summary, why_it_matters, source_name, source_url and source_type (`external` or `shipbytes`). External stories require a valid primary-source URL. No fake production stories.

The [server schema](https://github.com/robotman4/shipbytes-server/blob/master/docs/publication.schema.json) and [workflow documentation](https://github.com/robotman4/shipbytes-server/blob/master/docs/publications.md) are authoritative. The database controls publication and email state; editing already-published JSON never resends the newsletter.

## Issue and story images

The canonical image contract is **`image.url`**. Issue covers and individual stories use the same optional image object. Upload and verify an image before committing its URL. Git contains metadata, not image binaries.

```json
"image": {
  "url": "https://www.dropbox.com/scl/fo/<folder-token>/<token>/2026/2026-09-11?dl=1&preview=example-story.jpg&rlkey=<shared-link-key>",
  "type": "generated",
  "alt": "Illustration of a connected cargo vessel at sea.",
  "credit": "Ship Bytes",
  "source_url": null,
  "usage": "generated"
}
```

Use the actual tested public download URL, including its required query parameters. Do not invent Dropbox tokens or resolve paths from a shared folder. The canonical real example is `issues/2026/2026-09-11.json` on `robotman4/shipbytes-publications` master, containing one cover and six story images.

* `url` is the transport/download reference. HTTP and HTTPS on standard ports are supported. No credentials, fragments or unsupported schemes are accepted.
* `source_url` is provenance or attribution, never the download address.
* `type` is `generated`, `licensed`, `source`, or `own`. `alt` is required. `source` requires a valid `source_url` and nonempty `usage` describing permission/licensing.
* Generated images default credit to `Ship Bytes` and usage to `generated`. They are illustrations, not documentary photography, and are labelled as such on the site.
* Historical stories without an image remain valid. Their detail pages fall back to the issue cover; listings only display a story image when one exists.

### Download and validation

The reusable remote fetcher performs GET requests and follows up to five redirects, validating every destination before contacting it. This supports Dropbox public URLs that redirect through a file-specific link to `*.dropboxusercontent.com`. No Dropbox API credentials, folder ZIP downloads or provider/path resolution are used. The temporary `provider/shared_folder_url/path` contract is no longer accepted.

Each request resolves the hostname, rejects private, loopback, link-local, reserved and multicast addresses, then pins the connection to a validated public IP. The original HTTP Host and TLS SNI are retained, preventing a second DNS lookup from rebinding the request to a private address. Redirects receive the same checks. Proxy environment variables are ignored. Connection timeout is 10 seconds, read timeout 30 seconds, with a bounded transfer time and 2 MiB (2,097,152 bytes) input limit.

Pillow decodes the actual bytes. A valid JPEG returned as `application/binary` is accepted; an HTML error page returned as `image/jpeg` is rejected. Accepted inputs are nonanimated JPEG, PNG and WebP, at most 4 million decoded pixels. Downloads stay in bounded memory, so failed downloads leave no temporary files. Persistent file writes use cleaned-up temporary files and atomic replacement.

Explicit remote-image failures stop import before sending email. All required images are prepared before any publication sends. Logs identify the publication, story where applicable, failure category, remote hostname and a short URL fingerprint. Query strings and response bodies are omitted to avoid leaking shared-link keys or signed URL tokens. Failed image refreshes preserve existing published database records and visible images.

### Local media and derivatives

Dropbox is transport only. The source is preserved under `/data/media`, alongside high-quality 1200x630 and 600x315 JPEG derivatives. Images are resized with Lanczos and padded white to fit when required. Preferred ratio is 1.91:1. JPEG quality starts at 92 and may reduce to 88 or 85 to approach 250 KB; quality never drops below 85 solely to hit that target. Larger derivatives are allowed. Each derivative is encoded directly from the decoded original, never from another compressed derivative.

Remote-image paths include the SHA-256 of the original bytes:

```text
/data/media/issues/{issue-slug}/{source-sha256}/source.jpg
/data/media/issues/{issue-slug}/{source-sha256}/cover-1200.jpg
/data/media/issues/{issue-slug}/{source-sha256}/cover-600.jpg
/data/media/stories/{story-slug}/{source-sha256}/cover-1200.jpg
/data/media/stories/{story-slug}/{source-sha256}/cover-600.jpg
```

The original extension follows decoded format. Websites use `/media/...` URLs on Ship Bytes. New newsletters use the issue's locally hosted 600px cover; social metadata uses the 1200px version. Story detail pages and web listings display their own image. Rendering never fetches from Dropbox. Existing sent newsletters remain unchanged.

### Idempotency and image updates

The existing `0005` Alembic migration adds nullable Story image fields and a normalized JSON `image_reference` on both issues and stories. It already supports the final URL metadata, so no destructive replacement model or redundant migration is needed. Local paths, alt text, credit, type, source_url and usage are preserved.

An unchanged reference with existing derivatives is reused without downloading or processing. Changing `image.url` triggers a fresh download; different bytes produce new content-addressed paths. Identical bytes reuse the same files. Optional `sha256` (64 lowercase hex characters) pins and verifies an expected source checksum.

Replacing bytes behind the same URL alone does not silently refresh published media. Use a changed URL/hash or explicitly refresh:

```sh
docker compose --env-file .env --env-file .release.env exec -T app \
  python -m shipbytes.publish_repository /publications --refresh-images
```

The command still publishes eligible unpublished issues. Already-published issues receive image-only updates: text, story membership, publication time and Broadcast ID are unchanged, and no email is resent. Old media files remain available for historical links. Omitting an image in historical JSON preserves existing media.

### Legacy local images and backups

Legacy image objects with `path: "assets/..."` instead of `url` remain valid. Paths are repository-relative and never reinterpreted as URLs. Traversal, absolute paths and symlinks are rejected. Do not supply both `url` and `path`. Legacy optional missing/corrupt assets retain their fallback behavior; remote references are required once explicitly supplied.

The persistent media directory defaults to `media` beside SQLite (`/data/media` in production), and `MEDIA_DIRECTORY` can override it for development. Back up and restore media with the database. The default logo stays at `/static/brand/shipbytes-default-transparent.png`.
