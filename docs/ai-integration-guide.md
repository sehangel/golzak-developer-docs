# AI integration guide

This page is written for an **AI coding agent** (and the developer guiding it) that must build a reseller's
integration with the Golzak Domains API without help. It is self-contained: with this page and the
[OpenAPI file](openapi.yaml) you have everything. When this page and the OpenAPI file disagree, the OpenAPI file wins.

> **For the agent:** do not invent behaviour. Every rule you need is on this page, in the OpenAPI file, or on the pages
> linked here: [Workflows](workflows.md) (step-by-step tasks), [Limits](limits.md) (every number and what the API
> does not do), [Edge cases](edge-cases.md) (timeouts, races, duplicates) and [Errors](errors.md) (every code, whether
> to retry). If something is still unclear, stop and ask the developer instead of guessing.

## 1. Facts to hold on to

1. Base URL `https://api.golzak.com/domains/v1`; every path in the docs is relative to it. JSON in, JSON out, UTF-8.
2. Auth: `Authorization: Bearer <API key>` on every call, key created on developers.golzak.com. Server-side only.
3. v1 offers only `.ma`, `.co.ma`, `.net.ma`, `.org.ma`; `.ac.ma`, `.gov.ma`, `.press.ma` → `tld_manual_only`.
   Later versions may add extensions: read the list from `GET /prices`, don't hard-code it.
   Anything else → `tld_not_supported`. Names: one lower-case ASCII label + extension.
4. **Never hard-code a price.** Read `GET /prices` or `GET /check`. Every amount in these docs is a placeholder.
   Amounts are decimal strings; parse with a decimal type.
5. Payment is the reseller's prepaid credit. `402 insufficient_credit` means nothing was ordered.
6. Every POST needs an `Idempotency-Key`. Create it once per intended action, **persist it before sending**, and
   reuse it on every retry of that action.
7. Registration, renewal and transfer are **asynchronous**: `202` + operation. Final states: `active`, `refused`
   (registration only), `failed`. Listen to webhooks; poll `GET /operations/{id}` as a fallback.
8. The holder is the reseller's **client**; `legal_id` depends on `type` × `moroccan` (table in §4).
9. Errors are RFC 9457 problem details. Branch on `code` (and `errors[].field` + `errors[].code`), never on text.
10. No sandbox: test with `?dry_run=true` and with error cases, which never cost anything.
11. v1 only adds things. Treat an unknown error `code`, domain status, `pending_reason`, operation `type` or webhook
    event `type` as unknown, never as a crash: handle errors by HTTP status, acknowledge unknown events with `200`.
    Only operation `status` is a closed set (`pending`, `active`, `refused`, `failed`).
12. Domains start with `auto_renew: false`. Renew with `POST /domains/{name}/renew`, or enable auto-renew per domain
    (`PUT /domains/{name}/auto-renew`). `years` is at most 5, and a name never passes 5 years of validity.
13. To let a holder leave: `PUT /domains/{name}/transfer-lock` `{"locked": false}`, then `GET /domains/{name}/epp-code`
    (each call makes a new code). See [Expiry, renewal and transfers out](lifecycle.md).
14. API keys only work from the IP addresses listed for them. A `403 ip_not_allowed` in development means the
    machine's public IP is not on the key's allowlist.

### Endpoints at a glance

| Call | Purpose | Sync? | Costs money | Needs `Idempotency-Key` |
| --- | --- | --- | --- | --- |
| `GET /check?name=` | Availability + 1-year price | yes | no | no |
| `GET /prices` | Every price, allowed years, grace/redemption days | yes | no | no |
| `GET /balance` | Credit, cap, spent today | yes | no | no |
| `POST /domains` | Register | no (`202` + operation) | yes | **yes** |
| `GET /domains` | List your domains (paginated) | yes | no | no |
| `GET /domains/{name}` | One domain: status, expiry, nameservers, lock, auto-renew | yes | no | no |
| `POST /domains/{name}/renew` | Renew | no (`202` + operation) | yes | **yes** |
| `PUT /domains/{name}/nameservers` | Replace nameservers | yes | no | optional |
| `PUT /domains/{name}/transfer-lock` | Lock / unlock against transfer | yes | no | optional |
| `PUT /domains/{name}/auto-renew` | Turn automatic renewal on / off | yes | not now (later renewals do) | optional |
| `GET /domains/{name}/epp-code` | New transfer code (cancels the previous one) | yes | no | no |
| `POST /transfers` | Transfer a name in | no (`202` + operation) | yes | **yes** |
| `GET /operations/{id}` | State of an asynchronous operation | yes | no | no |

Every paid `POST` accepts `?dry_run=true`: all checks, no order, `200` with the would-be charge.

## 2. What to build

| Component | Responsibility |
| --- | --- |
| API client | One function per endpoint; adds the bearer key, `Idempotency-Key`, parses problem details into a typed error with `code`. Retries `429` (after `Retry-After`), `502`, `500` and network errors with the **same** idempotency key, exponential backoff, at most ~5 tries. Never retries other `4xx`. |
| Holder mapper | Converts the reseller's client record into `holder`. Asks the end user for `moroccan` and the right document number (§4) instead of guessing. |
| Operation store | Table: `operation_id`, `idempotency_key`, `domain`, `type`, `status`, `pending_reason`, `documents_required`, `documents_deadline`, `updated_at`, raw JSON. |
| Webhook endpoint | Verifies the signature on the raw body, stores the event durably (deduplicated on `Golzak-Webhook-Id`), then answers `200`; processing (upsert the operation if `updated_at` is newer) happens after. |
| Reconciler | Every 15 minutes: `GET /operations/{id}` for each local operation still `pending`. |
| Expiry watcher | Daily: for each domain with `auto_renew: false` and `expires_at` within 30 days, alert staff or renew. |
| Notifier | Tells the reseller's staff when `documents_required` becomes true (they must send documents to Golzak by e-mail before `documents_deadline`) and when an operation ends. |

## 3. The registration flow, step by step

```text
1. GET /check?name=N                 → available? price (1 year)
2. GET /prices                       → allowed years, price for the chosen years
3. build holder (§4); validate locally (optional, the API validates anyway)
4. key = uuid4(); store {key, N, "register"}
5. POST /domains?dry_run=true        (Idempotency-Key: new uuid) → 200 would_charge, or an error to show
6. POST /domains                     (Idempotency-Key: key)      → 202 operation op_…
7. store op_…; show "pending"
8. webhook operation.pending         → documents_required: tell staff, deadline
9. webhook operation.active          → done; GET /domains/N for expiry
   webhook operation.refused         → tell the client; refund in operation.refund
   webhook operation.failed          → show operation.error.detail; refund in operation.refund
```

A dry run consumes its own idempotency key; use a different key for the real order.

## 4. Holder: decide `legal_id`

```python
def legal_id_kind(holder_type: str, moroccan: bool) -> str:
    if holder_type == "individual":
        return "CIN" if moroccan else "passport number"
    if holder_type == "company":
        return "company tax ID (ICE / RC)" if moroccan else "company registration number or tax ID"
    raise ValueError("holder.type must be individual or company")
```

| `type` | `moroccan` | `legal_id` | Extra required |
| --- | --- | --- | --- |
| `individual` | `true` | CIN | – |
| `individual` | `false` | passport number | – |
| `company` | `true` | company tax ID (ICE / RC) | `organization` |
| `company` | `false` | company registration number or tax ID | `organization` |

Always required: `type`, `moroccan`, `legal_id`, `first_name`, `last_name` (for a company: its contact person),
`email`, `phone` as `+CC.NUMBER`, `address.line1`, `address.city`, `address.country` (ISO two letters, upper case).
`moroccan` is the holder's nationality (individual) or country of incorporation (company), **not** the address
country: ask, don't infer. Do not validate the ID's format yourself; send it as written on the document.

## 5. Worked examples: the four holder cases

Every request below is `POST /domains` with headers:

```http
Authorization: Bearer <API key>
Idempotency-Key: <new uuid per registration>
Content-Type: application/json
```

### 5.1 Moroccan individual → CIN

```json
{
  "name": "atlas-tours.ma",
  "years": 1,
  "holder": {
    "type": "individual",
    "moroccan": true,
    "legal_id": "<CIN number>",
    "first_name": "Salma",
    "last_name": "Bennani",
    "email": "salma.bennani@example.com",
    "phone": "+212.612345678",
    "address": { "line1": "12 Rue Example", "city": "Casablanca", "postal_code": "20000", "country": "MA" }
  },
  "nameservers": ["ns1.example-dns.com", "ns2.example-dns.com"]
}
```

Answer `202 Accepted`, header `Location: /v1/operations/op_01JA7X3M9Q2W8E5R6T7Y8U9I0P`:

```json
{
  "id": "op_01JA7X3M9Q2W8E5R6T7Y8U9I0P",
  "type": "register",
  "initiator": "api",
  "status": "pending",
  "pending_reason": "processing",
  "domain": "atlas-tours.ma",
  "years": 1,
  "charge": { "amount": "<your price, read live>", "currency": "<your account currency>" },
  "refund": null,
  "documents_required": false,
  "documents_deadline": null,
  "error": null,
  "created_at": "2026-10-08T09:00:00Z",
  "updated_at": "2026-10-08T09:00:00Z",
  "completed_at": null
}
```

### 5.2 Non-Moroccan individual → passport number

```json
{
  "name": "kemal-yilmaz.ma",
  "years": 1,
  "holder": {
    "type": "individual",
    "moroccan": false,
    "legal_id": "<passport number>",
    "first_name": "Kemal",
    "last_name": "Yilmaz",
    "email": "kemal@example.com",
    "phone": "+90.5321234567",
    "address": { "line1": "Example Cd. 5", "city": "Istanbul", "postal_code": "34000", "country": "TR" }
  },
  "nameservers": ["ns1.example-dns.com", "ns2.example-dns.com"]
}
```

Answer: `202` with an operation like 5.1 (`domain: "kemal-yilmaz.ma"`).

### 5.3 Moroccan company → company tax ID (ICE / RC)

```json
{
  "name": "atlas-export.co.ma",
  "years": 2,
  "holder": {
    "type": "company",
    "moroccan": true,
    "legal_id": "<ICE or RC number>",
    "organization": "Atlas Export SARL",
    "first_name": "Youssef",
    "last_name": "Alaoui",
    "email": "contact@atlas-export.example",
    "phone": "+212.522123456",
    "address": { "line1": "4 Boulevard Example", "city": "Rabat", "postal_code": "10000", "country": "MA" }
  },
  "nameservers": ["ns1.example-dns.com", "ns2.example-dns.com"]
}
```

Answer: `202` with an operation like 5.1 (`domain: "atlas-export.co.ma"`, `years: 2`), provided `2` is in your
`register` price table.

### 5.4 Foreign company → registration number or tax ID

```json
{
  "name": "anatolia-trade.ma",
  "years": 1,
  "holder": {
    "type": "company",
    "moroccan": false,
    "legal_id": "<company registration number or tax ID>",
    "organization": "Anatolia Trade A.S.",
    "first_name": "Elif",
    "last_name": "Demir",
    "email": "it@anatolia-trade.example",
    "phone": "+90.2121234567",
    "address": { "line1": "Example Sk. 10", "city": "Istanbul", "country": "TR" }
  },
  "nameservers": ["ns1.example-dns.com", "ns2.example-dns.com"]
}
```

Answer: `202` with an operation like 5.1 (`domain: "anatolia-trade.ma"`).

## 6. Worked examples: what each wrong case returns

Every error below is `application/problem+json`, and **nothing is ordered or charged**. `request_id` is omitted
for brevity.

### 6.1 Moroccan individual without a CIN

Body as 5.1 with `"legal_id": ""` (or the field missing). Answer `422`:

```json
{
  "type": "https://developers.golzak.com/domains/v1/errors#validation_failed",
  "title": "Validation failed",
  "status": 422,
  "code": "validation_failed",
  "detail": "The request breaks 1 rule. Nothing was ordered.",
  "errors": [
    { "field": "holder.legal_id", "code": "required", "message": "A Moroccan individual needs their CIN number in holder.legal_id." }
  ]
}
```

### 6.2 Non-Moroccan individual without a passport number

```json
{ "field": "holder.legal_id", "code": "required", "message": "A non-Moroccan individual needs their passport number in holder.legal_id." }
```

(Same `422 validation_failed` envelope as 6.1; only the `errors[]` entry differs.)

### 6.3 Moroccan company without its tax ID

```json
{ "field": "holder.legal_id", "code": "required", "message": "A Moroccan company needs its company tax ID (ICE / RC) in holder.legal_id." }
```

### 6.4 Foreign company without registration number or tax ID

```json
{ "field": "holder.legal_id", "code": "required", "message": "A foreign company needs its company registration number or tax ID in holder.legal_id." }
```

### 6.5 Company without `organization`, and several mistakes at once

Body as 5.3 without `organization`, with `"phone": "0522123456"` and without `moroccan`. Answer `422`, every rule
at once:

```json
{
  "type": "https://developers.golzak.com/domains/v1/errors#validation_failed",
  "title": "Validation failed",
  "status": 422,
  "code": "validation_failed",
  "detail": "The request breaks 3 rules. Nothing was ordered.",
  "errors": [
    { "field": "holder.moroccan", "code": "required", "message": "holder.moroccan must be true or false." },
    { "field": "holder.organization", "code": "required", "message": "A company holder needs its legal name in holder.organization." },
    { "field": "holder.phone", "code": "invalid_format", "message": "Write the phone as +<country code>.<number>, e.g. +212.612345678." }
  ]
}
```

### 6.6 Unknown holder type

`"type": "person"` → `422 validation_failed`:

```json
{ "field": "holder.type", "code": "invalid_value", "message": "holder.type must be individual or company." }
```

### 6.7 The holder is the reseller itself

Holder e-mail or organization equal to your own account's → `422`:

```json
{
  "type": "https://developers.golzak.com/domains/v1/errors#holder_is_reseller",
  "title": "Holder is the reseller",
  "status": 422,
  "code": "holder_is_reseller",
  "detail": "The holder must be your client, not your own company. Register the name in your client's name."
}
```

### 6.8 Extension handled by hand

`"name": "ministere.gov.ma"` (also `.ac.ma`, `.press.ma`) → `422`:

```json
{
  "type": "https://developers.golzak.com/domains/v1/errors#tld_manual_only",
  "title": "Extension handled by hand",
  "status": 422,
  "code": "tld_manual_only",
  "detail": ".gov.ma, .ac.ma and .press.ma are not available through the API. Contact Golzak to register one."
}
```

### 6.9 Not a .ma extension

`"name": "atlas-tours.com"` → `422`, `"code": "tld_not_supported"`,
`"detail": "This API version registers .ma, .co.ma, .net.ma and .org.ma only."`.

### 6.10 Not enough credit

Valid body, credit below the price → `402`:

```json
{
  "type": "https://developers.golzak.com/domains/v1/errors#insufficient_credit",
  "title": "Insufficient credit",
  "status": 402,
  "code": "insufficient_credit",
  "detail": "Top up your Golzak account credit, then retry with a new Idempotency-Key. Nothing was ordered."
}
```

### 6.11 Name already registered

`409`, `"code": "domain_not_available"`, `"detail": "atlas-tours.ma is already registered. Nothing was ordered."`.

### 6.12 Years not in the price table

`"years": 7` when your `register` table has no `7` → `422 validation_failed` with
`{ "field": "years", "code": "years_not_offered", "message": "Order one of the year counts listed in GET /prices." }`.

### 6.13 Missing idempotency key

`428`, `"code": "idempotency_key_required"`.

## 7. Following the operation

### What to do in each state

| `status` | `pending_reason` | Meaning | Your code does |
| --- | --- | --- | --- |
| `pending` | `processing` | Being ordered. | Nothing; wait. |
| `pending` | `registry_review` | ANRT examines the name. | Alert a human: documents to Golzak before the deadline in Golzak's e-mail. |
| `pending` | `transfer_pending` | Waiting for the losing registrar. | Nothing; wait (about 2 business days). |
| `pending` | any other value | A reason added later in v1. | Treat as "still pending". |
| `active` | `null` | Done. | Mark done; refresh the domain with `GET /domains/{name}`. |
| `refused` | `null` | ANRT refused the registration. | Tell the client; show `refund.status`. |
| `failed` | `null` | Could not be done. | Show `error.detail`; show `refund.status`; offer to retry with a new key. |

Final states (`active`, `refused`, `failed`) never change. Ignore any later event that would move an operation out of a
final state.

Held for ANRT examination (`GET /operations/{id}` or webhook `operation.pending`):

```json
{
  "id": "op_01JA7X3M9Q2W8E5R6T7Y8U9I0P",
  "type": "register",
  "initiator": "api",
  "status": "pending",
  "pending_reason": "registry_review",
  "domain": "atlas-tours.ma",
  "documents_required": true,
  "documents_deadline": null
}
```

(Shortened.) Tell the reseller's staff at once: Golzak has e-mailed them the ANRT form and document list, and the
documents must reach Golzak (reply to that e-mail or a ticket) before the deadline written in it
(`documents_deadline` repeats it when known). There is no upload endpoint in v1. Expect the registry's answer to
take up to about **3 weeks**.

Refused: `status: "refused"`, `error.code: "registry_refused"`, `refund: { "status": "credited", ... }` or
`"manual_review"`.

## 8. Webhook receiver (Node.js, Express)

```js
import express from "express";
import crypto from "node:crypto";

const app = express();
const SECRET = process.env.GOLZAK_WEBHOOK_SECRET;

app.post("/golzak/webhook", express.raw({ type: "application/json" }), async (req, res) => {
  const ts = req.get("Golzak-Webhook-Timestamp") ?? "";
  const sig = req.get("Golzak-Signature") ?? "";
  const fresh = /^\d{10}$/.test(ts) && Math.abs(Date.now() / 1000 - Number(ts)) <= 300;
  const expected = "v1=" + crypto.createHmac("sha256", SECRET).update(`${ts}.`).update(req.body).digest("hex");
  const ok = fresh && sig.length === expected.length && crypto.timingSafeEqual(Buffer.from(sig), Buffer.from(expected));
  if (!ok) return res.sendStatus(400);

  const eventId = req.get("Golzak-Webhook-Id");
  const event = JSON.parse(req.body.toString("utf8"));
  try {
    await saveEventOnce(eventId, event);             // your code: durable inbox, dedupe on eventId (fast)
  } catch {
    return res.sendStatus(500);                      // not stored: let Golzak retry
  }
  res.sendStatus(200);                               // acknowledge only what is stored
  processInboxLater(eventId);                        // your code: upsert the operation if updated_at is newer
});
```

## 9. Acceptance checklist for the integration

- [ ] No price, currency or year count is hard-coded; all come from `GET /prices` / `GET /check`.
- [ ] Every POST sends an `Idempotency-Key` that is persisted before the request and reused on retry.
- [ ] `429`, `500`, `502` and timeouts are retried with the same key; other `4xx` are shown, not retried.
- [ ] The four holder cases of §5 produce the right `legal_id`; `moroccan` is asked, not inferred from the address.
- [ ] A dry run of each §6 error case shows the right message to the user and orders nothing.
- [ ] Webhook signatures are verified on the raw body; bad signatures get `400`; events are deduplicated.
- [ ] `documents_required: true` alerts a human, with `documents_deadline` when it is not `null`.
- [ ] A reconciler polls operations still `pending`.
- [ ] `refused` and `failed` show the refund status to the reseller.
- [ ] Expiring domains are renewed or flagged before `expires_at`; nothing assumes auto-renew is on.
