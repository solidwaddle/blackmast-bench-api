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

**Scopes** are attached per key. Today's two scopes:

- `opportunities:read` — required for `GET /opportunities*`
- `bids:write` — required for `POST /bids`

Most contractors get both. If you need something not in your key's scope, request a scope expansion.

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
