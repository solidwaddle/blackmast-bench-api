# Blackmast Bench API

Public REST API for Blackmast bench contractors to integrate their own toolchains directly into the Blackmast portal.

**Base URL:**
```
https://espejknkeeayjnzyrejw.supabase.co/functions/v1/contractor-api
```

**Status:** Early access. Endpoints below are stable; more will land as needed. Watch this repo for changes.

---

## Auth

Every request must include an `Authorization: Bearer <key>` header. Keys are issued by Mario on request — DM him or email `mvaldes@blackmastco.com`. Keys look like:

```
bm_pat_jX3z9Q...
```

The plaintext is shown to you exactly once when the key is issued. Store it somewhere secure (1Password, env file, etc.) — there's no recovery path. If you lose it, ask for a fresh one.

**Scopes** are attached per key. Current scopes:

- `opportunities:read` — required for `GET /opportunities*`
- `bids:write` — required for `POST /bids` and the bid attachment routes
- `active_solicitations:read` — required for `GET /active-solicitations*` (the curated go-now lane)

Most contractors get all three. If you need something not in your key's scope, request a scope expansion — no need to rotate your key, we'll just add it.

---

## Endpoints

### `GET /opportunities`

Returns the current open opportunity pool — slim projection (one row per opp, ~12 columns, no descriptions). Sorted by `win_likelihood_score DESC`, then `due_date ASC`.

**Query params (all optional):**

| Param | Type | Default | Meaning |
|---|---|---|---|
| `limit` | int | 50 | Max rows to return. Cap 200. |
| `source` | string | (any) | Filter to one ingest source: `sam_gov`, `bidmatch`, `usa_spending`, etc. |
| `naics` | string | (any) | Exact 6-digit NAICS code. |
| `include_past_due` | bool | false | Pass `true` to include opps whose due date has passed. |

**Example:**

```bash
curl -s \
  -H "Authorization: Bearer bm_pat_xxxxxxxx" \
  "https://espejknkeeayjnzyrejw.supabase.co/functions/v1/contractor-api/opportunities?limit=20&source=bidmatch"
```

**Response:**

```json
{
  "ok": true,
  "data": {
    "count": 20,
    "opportunities": [
      {
        "id": "1f9b07f0-...",
        "source": "bidmatch",
        "external_id": "bm-rfq-12345",
        "title": "Emergency Management Software Platform Services",
        "agency": "AlamedaCounty, CA",
        "naics": "541511",
        "notice_type": "BidMatch - State",
        "due_date": "2026-06-25",
        "posted_date": "2026-06-11",
        "set_aside": null,
        "place": "CA",
        "url": "https://...",
        "win_likelihood_score": 78
      }
      // ... 19 more
    ]
  }
}
```

---

### `GET /opportunities/:id`

Single opportunity, full payload including the `description` field.

**Path param:** `id` must be a UUID (the `id` field from `GET /opportunities`).

**Example:**

```bash
curl -s \
  -H "Authorization: Bearer bm_pat_xxxxxxxx" \
  "https://espejknkeeayjnzyrejw.supabase.co/functions/v1/contractor-api/opportunities/1f9b07f0-1234-1234-1234-abc..."
```

**Response shape:** same as a listing row, plus `description`, `value_low`, `value_high`.

---

### `POST /bids`

Create or update a bid draft for a specific opportunity. Each opportunity can have at most one draft — calling `POST /bids` for the same `opportunity_id` updates the existing draft.

**Request body (JSON):**

| Field | Type | Required | Notes |
|---|---|---|---|
| `opportunity_id` | uuid | yes | The `id` from `GET /opportunities`. |
| `status` | string | no (default `drafting`) | One of `drafting`, `review`, `submitted`, `withdrawn`. |
| `sections` | object | no | Free-form JSON. Suggested keys: `executive_summary`, `technical_approach`, `management_approach`, `past_performance_lead`, `win_themes` (array), `pricing_notes`, `risk_mitigation`, `compliance_matrix` (array of `{requirement, section, status}`). |
| `notes` | string | no | Free-text notes for Mario. |

When `status = "submitted"` the API also sets `submitted_at` to the current time.

**Example:**

```bash
curl -s -X POST \
  -H "Authorization: Bearer bm_pat_xxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "opportunity_id": "1f9b07f0-1234-1234-1234-abc...",
    "status": "review",
    "sections": {
      "executive_summary": "Blackmast proposes a cloud-native emergency management platform...",
      "technical_approach": "We will deploy a modular architecture on AWS GovCloud...",
      "win_themes": ["proven NDR experience", "rapid 90-day rollout", "deep AlamedaCounty stakeholder map"],
      "compliance_matrix": [
        {"requirement": "Section L.3.2 Tech Approach", "section": "3.1", "status": "drafted"}
      ]
    },
    "notes": "Holding on pricing until I confirm hosting tier with vendor."
  }' \
  "https://espejknkeeayjnzyrejw.supabase.co/functions/v1/contractor-api/bids"
```

**Response:**

```json
{
  "ok": true,
  "data": {
    "id": "abcd1234-...",
    "opportunity_id": "1f9b07f0-...",
    "status": "review",
    "created_by": "your-user-uuid",
    "created_via": "contractor_api",
    "submitted_at": null,
    "created_at": "2026-06-19T19:32:11.000Z",
    "updated_at": "2026-06-19T19:32:11.000Z"
  }
}
```

---

### Attachments

Attach supporting docs (PDFs, Word, images, etc.) to a specific bid. All attachment routes are scoped to an opportunity — you must have already created a draft via `POST /bids` for that opportunity first.

Uploads are JSON+base64 (no multipart). Practical file ceiling is **~4.5MB** because Supabase edge functions cap request bodies at 6MB and base64 inflates the payload ~33%. If you need bigger files, open an issue and we'll add a signed-upload-URL flow.

All attachment routes require the `bids:write` scope (same scope as `POST /bids`).

---

#### `POST /bids/:opportunity_id/attachments`

Upload a single attachment to the bid for `:opportunity_id`.

**Request body (JSON):**

| Field | Type | Required | Notes |
|---|---|---|---|
| `filename` | string | yes | Original filename. Sanitized internally to `[A-Za-z0-9._-]`. |
| `content_type` | string | yes | MIME type, e.g. `application/pdf`. |
| `content_base64` | string | yes | File bytes encoded as base64 (no `data:` prefix). |
| `notes` | string | no | Free text. |

**Example:**

```bash
# encode a local PDF then upload
B64=$(base64 -i ./past_performance.pdf)
curl -s -X POST \
  -H "Authorization: Bearer bm_pat_xxxxxxxx" \
  -H "Content-Type: application/json" \
  -d "{
    \"filename\": \"past_performance.pdf\",
    \"content_type\": \"application/pdf\",
    \"content_base64\": \"$B64\",
    \"notes\": \"NDR project recap for AlamedaCounty bid\"
  }" \
  "https://espejknkeeayjnzyrejw.supabase.co/functions/v1/contractor-api/bids/1f9b07f0-1234-1234-1234-abc.../attachments"
```

**Response:**

```json
{
  "ok": true,
  "data": {
    "id": "8b91ab10-...",
    "bid_draft_id": "abcd1234-...",
    "filename": "past_performance.pdf",
    "content_type": "application/pdf",
    "size_bytes": 412339,
    "uploaded_via": "contractor_api",
    "notes": "NDR project recap for AlamedaCounty bid",
    "created_at": "2026-06-22T16:14:02.000Z"
  }
}
```

Errors specific to upload:
- `404` — no bid draft exists for `:opportunity_id`. Call `POST /bids` first.
- `413` — file too large (decoded bytes exceed 5MB).
- `400` — `content_base64` is not valid base64 or decoded to zero bytes.

---

#### `GET /bids/:opportunity_id/attachments`

List metadata for all attachments on a bid. No file contents.

**Response:**

```json
{
  "ok": true,
  "data": {
    "count": 2,
    "attachments": [
      {
        "id": "8b91ab10-...",
        "filename": "past_performance.pdf",
        "content_type": "application/pdf",
        "size_bytes": 412339,
        "uploaded_via": "contractor_api",
        "notes": "NDR project recap",
        "created_at": "2026-06-22T16:14:02.000Z"
      }
    ]
  }
}
```

If no bid draft exists yet, returns `{ "ok": true, "data": { "count": 0, "attachments": [] } }` — easier to fan out than a 404.

---

#### `GET /bids/:opportunity_id/attachments/:attachment_id`

Get one attachment plus a **24-hour signed download URL**. The URL is the only way to fetch the file bytes — the bucket is private.

**Response:**

```json
{
  "ok": true,
  "data": {
    "id": "8b91ab10-...",
    "filename": "past_performance.pdf",
    "content_type": "application/pdf",
    "size_bytes": 412339,
    "uploaded_via": "contractor_api",
    "notes": "NDR project recap",
    "created_at": "2026-06-22T16:14:02.000Z",
    "download_url": "https://espejknkeeayjnzyrejw.supabase.co/storage/v1/object/sign/bid-attachments/...",
    "expires_in_seconds": 86400
  }
}
```

The `download_url` is a vanilla HTTPS link — no auth header needed to fetch it. Re-call this endpoint to mint a fresh URL whenever the old one expires.

---

#### `DELETE /bids/:opportunity_id/attachments/:attachment_id`

Remove an attachment. Deletes both the Storage object and the metadata row. Not reversible.

**Response:**

```json
{ "ok": true, "data": { "id": "8b91ab10-...", "deleted": true } }
```

---

### Active Solicitations

The curated go-now lane. These are solicitations Mario hand-picks as ready-to-bid right now — they live in a separate table from the scraped `/opportunities` pool and are not duplicated across both. Status visibility is enforced server-side: bench contractors only see rows that are `published` (or `closed`, if you opt in).

Both routes require the `active_solicitations:read` scope. Ask for a scope expansion if your key doesn't have it.

---

#### `GET /active-solicitations`

List the open curated lane.

**Query params (all optional):**

| Param | Type | Default | Meaning |
|---|---|---|---|
| `limit` | int | 50 | Max rows to return. Cap 200. |
| `naics` | string | (any) | Exact 6-digit NAICS code filter. |
| `priority` | `urgent` \| `standard` | (any) | Filter to one priority. |
| `include_closed` | bool | false | Include solicitations whose status is `closed` (e.g. for post-mortems). |
| `match` | `mine` \| `all` | `mine` | `mine` filters to solicitations whose NAICS matches your NAICS preferences (exact or 4-digit prefix). `all` returns the full set. |

If your account has no NAICS preferences set, `match=mine` falls through to `all` so you don't get an empty list while you're ramping up.

**Default ordering:** `priority='urgent'` first, then `due_date` ascending, then `published_at` descending.

**Example:**

```bash
curl -s \
  -H "Authorization: Bearer bm_pat_xxxxxxxx" \
  "https://espejknkeeayjnzyrejw.supabase.co/functions/v1/contractor-api/active-solicitations?limit=20&priority=urgent"
```

**Response:**

```json
{
  "ok": true,
  "data": {
    "count": 3,
    "active_solicitations": [
      {
        "id": "a1b2c3d4-...",
        "title": "Emergency Management Software Platform Services",
        "agency": "AlamedaCounty, CA",
        "naics": "541511",
        "set_aside": null,
        "notice_type": "RFP",
        "place": "CA",
        "due_date": "2026-07-08",
        "posted_date": "2026-06-22",
        "source_url": "https://sam.gov/...",
        "estimated_value_min": 250000,
        "estimated_value_max": 500000,
        "priority": "urgent",
        "status": "published",
        "published_at": "2026-06-22T18:00:00.000Z",
        "closed_at": null,
        "attachments": [
          {
            "id": "f00dcafe-...",
            "filename": "RFP_SOW.pdf",
            "content_type": "application/pdf",
            "size_bytes": 318204,
            "download_url": "https://espejknkeeayjnzyrejw.supabase.co/storage/v1/object/sign/active-solicitation-files/...",
            "expires_in_seconds": 3600
          }
          // ... more files, in display order
        ]
      }
      // ...
    ]
  }
}
```

The list payload omits `description` to keep it slim. Use the detail endpoint to get the full body.

#### Attachments

Curated solicitations can carry supporting files (the SOW, amendments, Q&A, attachments, etc.), uploaded by Mario in the portal. Both `GET /active-solicitations` and `GET /active-solicitations/:id` return an **`attachments`** array on each solicitation, in display order:

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | Attachment id. |
| `filename` | string | Original filename. |
| `content_type` | string | MIME type, e.g. `application/pdf`. |
| `size_bytes` | int | File size. |
| `download_url` | string | **Signed HTTPS link, valid 1 hour.** No auth header needed to fetch it — the bucket (`active-solicitation-files`) is private. |
| `expires_in_seconds` | int | Always `3600`. |

Solicitations with no files return `"attachments": []`.

Because `download_url`s expire after **one hour**, treat them as ephemeral: fetch the bytes shortly after the API call, or re-request the solicitation to mint fresh URLs. Don't cache or share the signed URL itself — re-call the endpoint instead.

> **Breaking change (2026-06-30):** the old `attachments_url` field (a single nullable string, previously always `null`) has been **removed**. Replace any reference to `attachments_url` with the `attachments[]` array above.

---

#### `GET /active-solicitations/:id`

Single curated solicitation, full payload including `description`.

**Path param:** `id` must be a UUID (the `id` field from the list endpoint).

**Returns 404 if:**
- The id doesn't exist, or
- The solicitation's status is not `published` or `closed` (drafts are not visible to bench contractors)

**Example:**

```bash
curl -s \
  -H "Authorization: Bearer bm_pat_xxxxxxxx" \
  "https://espejknkeeayjnzyrejw.supabase.co/functions/v1/contractor-api/active-solicitations/a1b2c3d4-..."
```

**Response shape:** same as a list row, plus `description`. The `attachments[]` array is included here too, with freshly minted 1-hour signed `download_url`s on each call.

---

## Errors

All errors come back as `{ "ok": false, "error": "<human-readable message>" }` with an appropriate HTTP status.

| Status | Meaning |
|---|---|
| `400` | Bad request — usually a missing/invalid field. |
| `401` | Auth failed — missing/wrong/revoked key. |
| `403` | Your key doesn't have the required scope. |
| `404` | Route or referenced object (e.g. `opportunity_id`) doesn't exist. |
| `500` | Server-side error. Open an issue here with the response body. |

---

## Rate limiting

No hard limit today. Be reasonable. If you're going to fan out across the full opp pool (1000+ rows), batch on your end and respect a few-hundred-ms delay between requests so we don't have to add formal limits.

---

## Versioning

Today this is a v0 API. Breaking changes will be communicated here at least 7 days in advance. If you're shipping this to production, pin to a specific commit of this README and re-read on each bump.

---

## What's missing (intentional)

- **`POST /opportunities`** — pushing your own opp discoveries into the pool. Coming after we have a review queue so Mario can approve before they enter the main pool.
- **Bulk endpoints** — if you need to push 50 bids at once, do them one at a time for now. Open an issue if you actually hit a use case that needs bulk.
- **Webhooks** — no push notifications from the portal to your tools yet. If this becomes a real need, open an issue.

---

## Support

- **Bug or weird response:** open an issue in this repo
- **Need a key, scope expansion, or revoke:** DM Mario or email `mvaldes@blackmastco.com`
- **Want to add a contractor to the bench:** Mario does onboarding
