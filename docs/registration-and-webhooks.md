# Registration, operations and webhooks

## Why registration is asynchronous

The .ma registry (ANRT) does not always create a name at once. For some requests it answers "action pending" and
examines the name first (ANRT decision ANRT/DG/N°02/2024, art. 32: notably names that conflict with its list of
reserved terms). While it examines, the name is held at the registry and cannot be changed. ANRT then approves it
(the name goes live) or refuses it (the name disappears).

So `POST /domains` does not wait. It answers `202` with an **operation**; the result comes later.

## Operation lifecycle

```text
                   ┌──────────────► active    (registered)
 pending ──────────┼──────────────► refused   (ANRT refused; charge returned)
 (processing,      └──────────────► failed    (technical failure; charge returned)
  registry_review)
```

| Field | Meaning |
| --- | --- |
| `status` | `pending` until the end, then `active`, `refused` or `failed`. Final states never change. |
| `pending_reason` | `processing` (being ordered), `registry_review` (ANRT examines the name), `transfer_pending` (transfer under way). `null` when finished. |
| `documents_required` | `true` while ANRT waits for the examination form and the holder's documents. |
| `documents_deadline` | Last day to send them to Golzak, when known; `null` otherwise. The date in Golzak's e-mail is always the reference. |
| `charge` | What was taken from your credit. |
| `refund` | After `refused` or `failed`: `credited` (back on your credit) or `manual_review` (Golzak returns it by hand). |
| `error` | Why it was refused or failed, in the [error format](errors.md). |

Renewals and transfers return operations too. They end `active` or `failed`; only registrations can be `refused`.

## When ANRT asks for documents

When an operation enters `registry_review`:

1. You receive the webhook `operation.pending` with `documents_required: true` (and `documents_deadline` when known).
2. Golzak e-mails you the official ANRT examination form and the documents needed for your client (for an individual,
   an identity document; for a company, its registration documents; and proof of the right to the name).
3. Get them from your client and **send them to Golzak by reply to that e-mail, or in a support ticket, before
   the deadline in that e-mail**. Golzak files them with ANRT. API v1 has no document upload.

ANRT's decision gives the registrar 3 business days from the request to file the documents, after which the request
is cancelled, and gives ANRT 5 business days from receiving them to decide (art. 32). That deadline already
includes Golzak's own margin.

**How long it really takes.** Do not count on the decision's figures. In the cases Golzak has seen, the registry's
answer came between 3 and 18 calendar days after the request. Plan for up to **3 weeks**, show your client "pending registry
review", and react to the webhook.

## Webhooks

Golzak POSTs an event to every webhook endpoint of your account (set on developers.golzak.com, each with its own
signing secret) whenever an operation enters `registry_review` or ends. Endpoints belong to the account, so an
operation started with a key you have since revoked is still reported.

| Event `type` | When |
| --- | --- |
| `operation.pending` | The operation entered `registry_review`: documents required. |
| `operation.active` | Done. The domain is active (registered, renewed or transferred in). This includes automatic renewals (`initiator: "auto_renew"`). |
| `operation.refused` | ANRT refused the registration. |
| `operation.failed` | The operation could not be done. |

Body:

```json
{
  "id": "evt_01JA7XB2C3D4E5F6G7H8J9K0LM",
  "type": "operation.active",
  "created_at": "2026-10-09T08:00:00Z",
  "data": { "operation": { "id": "op_01JA7X3M9Q2W8E5R6T7Y8U9I0P", "type": "register", "initiator": "api", "status": "active", "domain": "atlas-tours.ma" } }
}
```

(`operation` is shortened here; you receive the full object, as from `GET /operations/{id}`.)

### Verify the signature

Each delivery has three headers:

| Header | Content |
| --- | --- |
| `Golzak-Webhook-Id` | Event ID; the same on every retry of that event. |
| `Golzak-Webhook-Timestamp` | Unix time (seconds) of signing. |
| `Golzak-Signature` | `v1=` + lower-case hex of `HMAC-SHA256(webhook_secret, timestamp + "." + raw_body)`. |

Reject the delivery if the timestamp is more than 300 seconds from your clock or the signature does not match.
Compute the HMAC over the **raw** body bytes, before any JSON parsing, and compare in constant time.

Node.js:

```js
import crypto from "node:crypto";

export function verifyGolzakWebhook(rawBody, headers, secret) {
  const ts = headers["golzak-webhook-timestamp"];
  const sig = headers["golzak-signature"] ?? "";
  if (!/^\d{10}$/.test(ts) || Math.abs(Date.now() / 1000 - Number(ts)) > 300) return false;
  const expected = "v1=" + crypto.createHmac("sha256", secret).update(`${ts}.`).update(rawBody).digest("hex");
  return sig.length === expected.length && crypto.timingSafeEqual(Buffer.from(sig), Buffer.from(expected));
}
```

Python:

```python
import hashlib, hmac, time

def verify_golzak_webhook(raw_body: bytes, headers: dict, secret: str) -> bool:
    ts = headers.get("golzak-webhook-timestamp", "")
    sig = headers.get("golzak-signature", "")
    if not (ts.isdigit() and len(ts) == 10) or abs(time.time() - int(ts)) > 300:
        return False
    expected = "v1=" + hmac.new(secret.encode(), ts.encode() + b"." + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(sig, expected)
```

### Delivery rules

- Store the event durably first, then answer any `2xx` within 10 seconds; do slow work after answering. A `2xx`
  ends the retries, so never acknowledge an event you have not stored.
- Anything else, or no answer, is retried with increasing delays for 24 hours.
- An event can arrive twice or out of order. Deduplicate on `Golzak-Webhook-Id`. Keep the newest state by the
  operation's `updated_at`, or re-read `GET /operations/{id}`.
- Webhooks are a convenience, not the only source of truth: if you missed some, read `GET /operations/{id}` for each
  operation still `pending` in your system (for example every 15 minutes).

## Polling instead of webhooks

`GET /operations/{id}` is always available. Poll no more than once a minute per operation; during `registry_review`,
once an hour is enough.
