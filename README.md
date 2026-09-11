# Ship Bytes publications

Production content for Ship Bytes. The `master` branch is imported Tuesday and Friday at approximately 06:00 Europe/Stockholm, with checks at 07:00, 08:00 and 09:00 if nothing was published.

One issue per `issues/YYYY/YYYY-MM-DD.json`, with `schema_version: 1`, a unique slug, title, subject, published_date, intro and nonempty stories. Stories require unique slug, title, byte, summary, why_it_matters, source_name, source_url and source_type (`external` or `shipbytes`). External stories require a valid primary-source URL. No fake production stories.

The [server schema](https://github.com/robotman4/shipbytes-server/blob/master/docs/publication.schema.json) and [workflow documentation](https://github.com/robotman4/shipbytes-server/blob/master/docs/publications.md) are authoritative. The database controls publication and email state; editing already-published JSON never resends the newsletter.

## Issue and story images

Images are staged in Dropbox; GitHub contains JSON metadata only for new images. Use the same optional `image` object on an issue and on each story. Stories without an image remain valid and use their issue cover on the full story page; story listings show an image only when that story has its own.

Configured public folder: `/ShipBytes`

```text
https://www.dropbox.com/scl/fo/ry05g9ow61zsroacmfdi1/AJcsXkGtzq3J8ONf0f0rJR8?rlkey=t1aod2y9bdclzg5y0d9y08tk8&dl=0
```

Upload and verify images before committing their references. Paths are relative to `/ShipBytes`, without a leading slash or `ShipBytes/` prefix. For an issue:

```json
"image": {
  "provider": "dropbox",
  "shared_folder_url": "https://www.dropbox.com/scl/fo/ry05g9ow61zsroacmfdi1/AJcsXkGtzq3J8ONf0f0rJR8?rlkey=t1aod2y9bdclzg5y0d9y08tk8&dl=0",
  "path": "2026/2026-09-11/cover.jpg",
  "type": "generated",
  "alt": "Illustration of a connected cargo vessel with satellite, cybersecurity, navigation and data overlays.",
  "credit": "Ship Bytes",
  "source_url": null,
  "usage": "generated"
}
```

Each story can contain the same object with its own path, such as `2026/2026-09-11/nexuswave-bv-e27-approval.jpg`, and a descriptive alt text. Do not reference a story image until that file exists. Do not call generated illustrations documentary photographs. Generated images are labelled as illustrations on the website.

`type` is `generated`, `licensed`, `source`, or `own`. `path`, `type`, and `alt` are required. `source` requires `source_url` and nonempty `usage` describing permission/licensing. Generated images default credit to `Ship Bytes` and usage to `generated`. `source_url` is attribution metadata, never a download address. Dropbox images additionally require `shared_folder_url`. The publisher rejects any folder other than `DROPBOX_SHARED_FOLDER_URL` in server configuration. Paths reject absolute paths, traversal, URL schemes, backslashes and encoded path segments.

### Transport and configuration

The dedicated `AssetResolver` supports two Dropbox transports without changing JSON:

* Without credentials, download the public folder as a ZIP using Dropbox's documented `dl=1` parameter. Read only the exact referenced member; never extract the archive to disk. One import downloads the archive at most once, only if an image needs importing. The compressed/decompressed HTTP response is capped at 32 MiB and each selected member at 2 MiB. Large archives fail explicitly instead of consuming unlimited resources.
* For a growing archive, set `DROPBOX_APP_KEY` and `DROPBOX_APP_SECRET` in the private server `.env`. The provider uses Dropbox's official `sharing/get_shared_link_file` API with app authentication, the shared-folder URL and `path: "/" + relative_path`. It downloads only the requested file. Configure both credentials together and recreate the app container. Never put credentials in issue JSON, Git or logs.

Dropbox does not document an unauthenticated child-file API for shared-folder links. We do not invent child-link tokens or scrape preview pages. Sources: [force downloads](https://help.dropbox.com/share/force-download) and [official shared-link API specification](https://github.com/dropbox/dropbox-api-spec/blob/main/sharing.stone).

Downloads follow up to five redirects, checking each HTTPS destination against Dropbox download hosts before connecting. Authentication is removed on redirects. Network, HTTP, missing member, size, checksum and image decoding failures are publication errors for explicit Dropbox references. The publisher prepares all required images before sending any newsletter. Existing published images remain visible if a refresh fails.

### Processing, caching and updates

The importer validates actual image content with Pillow: nonanimated JPEG, PNG or WebP, at most 2,097,152 bytes and 4 million pixels. It preserves the original and creates 1200x630 and 600x315 JPEGs with Lanczos resizing and white padding when needed. JPEG quality starts at 92, may reduce to 88 or 85 to aim below 250 KB, and never drops below 85 solely to meet a byte target. Large high-quality derivatives are allowed. Preferred cover ratio: 1.91:1.

Dropbox derivatives are immutable, content-addressed local files:

```text
/data/media/issues/{issue-slug}/{source-sha256}/source.jpg
/data/media/issues/{issue-slug}/{source-sha256}/cover-1200.jpg
/data/media/issues/{issue-slug}/{source-sha256}/cover-600.jpg
/data/media/stories/{story-slug}/{source-sha256}/cover-1200.jpg
/data/media/stories/{story-slug}/{source-sha256}/cover-600.jpg
```

The original extension follows decoded format. Public URLs use `/media/...` on Ship Bytes, never Dropbox. Issue covers remain in the archive, issue pages and newsletters (600px version). Stories use their own images on story pages, social metadata and web listings. Previously sent newsletter HTML is not rewritten.

SQLite records the normalized full reference, attribution, local paths and image type on both issues and stories. Unchanged references with existing local derivatives are reused without downloading or processing. For an intentional change, use a new path or add/change optional `sha256` (64 lowercase hex characters of the original file); a supplied hash is verified. Replacing bytes at the same Dropbox path alone does not silently refresh the site. To explicitly recheck the current references:

```sh
docker compose --env-file .env --env-file .release.env exec -T app \
  python -m shipbytes.publish_repository /publications --refresh-images
```

This still publishes eligible unpublished issues; already-published issues receive image-only updates and are never emailed again. Their text, story membership, publication time and Broadcast ID remain unchanged. Changed images receive new URLs; old files are retained for historical links. Repeated identical refreshes reuse the same file paths and do not rewrite matching files. Omitting an image in older JSON preserves existing media.

Legacy `image` objects without `provider` (or with `provider: "repository"`) still use repository-relative `assets/...` paths. They are never interpreted as Dropbox paths. Legacy optional broken/missing files retain their prior fallback behavior. Do not add new image binaries to Git. Remove a legacy file only after its Dropbox replacement is imported and verified.

The media directory defaults to `media` beside SQLite (`/data/media` in production). Back up and restore it with the database. `MEDIA_DIRECTORY` remains available for development. The default logo stays at `/static/brand/shipbytes-default-transparent.png`.
