# Authentication, idempotency and limits

## API key

Every call sends your key as a bearer token:

```http
Authorization: Bearer <your API key>
```

- **Get a key on [developers.golzak.com](https://developers.golzak.com)** (key management opens with the API). Sign in
  with your **Golzak account**: the
  single sign-in for every Golzak service, the same e-mail and password as on the Golzak client portal. Golzak makes
  it a reseller account when your contract is signed. Only a reseller account can create keys. You see the key
  **once**, when it is created: it starts with `gzk_live_`. Golzak stores only a hash of it and can never show it
  again.
- One key belongs to one reseller account. Everything you do with it is billed to that account's credit, and you only
  ever see that account's domains. An account can hold several keys (e.g. one per system), each with its own name,
  daily spending cap and IP allowlist.
- **Webhooks and automatic renewals belong to the account, not to a key.** Webhook endpoints and the daily cap for
  automatic renewals are set once per account on developers.golzak.com, so revoking or rotating a key never loses a
  webhook or stops an auto-renewal.
- **Every key has an IP allowlist, and it is required:** the public IPv4/IPv6 addresses or CIDR ranges of the servers
  that call the API. A call from any other address answers `403 ip_not_allowed`, so a leaked key is useless elsewhere.
  Edit the list on developers.golzak.com; changes apply at once.
- A missing, wrong or revoked key answers `401 invalid_api_key`.
- **Keys do not expire** unless you give one an expiry date when you create it (you get e-mail reminders before it).
  developers.golzak.com shows when and from which IP each key was last used; a key unused for 90 days is flagged and
  you are e-mailed.
- **Rotate** without downtime: create a new key, deploy it, then revoke the old one on developers.golzak.com.
- If a key leaks, revoke it at once on developers.golzak.com; the next call with it is refused.

## Idempotency keys

Every `POST` needs an `Idempotency-Key` header (`428 idempotency_key_required` without one). Use a new random value,
such as a UUID v4, for each action you intend, and store it with the action **before** you send the request.

| You send | You get |
| --- | --- |
| A new key | The call runs normally. |
| The same key and the same body (a retry) | The first answer again, with `Idempotent-Replayed: true`. Nothing is ordered twice. |
| The same key while the first call still runs | `409 idempotency_key_in_use`. Wait and retry. |
| The same key with a different body | `422 idempotency_key_reused`. Use a new key for a new action. |

Keys are kept for 24 hours. After a timeout or a `502`, always retry with the **same** key: that is what makes a retry
safe. `PUT /domains/{name}/nameservers` accepts an `Idempotency-Key` too (optional).

## Rate limits

Each key has a request rate limit (60 requests per minute unless your contract says otherwise). Every answer carries:

| Header | Meaning |
| --- | --- |
| `RateLimit-Limit` | Requests allowed per window |
| `RateLimit-Remaining` | Requests left in this window |
| `RateLimit-Reset` | Seconds until the window resets |

Over the limit you get `429 rate_limited` with `Retry-After` (seconds). Wait that long, then retry.

## Daily spending cap

Golzak can set a **daily spending cap** on a key: the most it may spend per UTC day (00:00 to 24:00 UTC). An order
that would go over it answers `403 daily_spend_cap_reached` and is not placed. `GET /balance` shows the cap,
`spent_today` and when the cap resets. Refunds do not lower `spent_today`. To change the cap, ask Golzak.

## Request IDs

Every answer has a `Request-Id` header, also in error bodies as `request_id`. Quote it when you contact Golzak.
