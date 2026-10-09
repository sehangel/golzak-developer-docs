# Edge cases

Questions developers and AI agents run into, with the exact answer. If yours is missing, it is a gap in these docs:
tell Golzak.

## Network and retries

### My `POST` timed out. Was the domain ordered?

Maybe. Send the **same** request again with the **same** `Idempotency-Key`. If the first one went through, you get
its answer again (`Idempotent-Replayed: true`) and nothing is ordered twice. If it did not, it runs now. Never send a
new key to "try again": that can order twice.

### I got `409 idempotency_key_in_use`

The first request with that key is still running. Wait a few seconds and retry with the same key.

### I got `422 idempotency_key_reused`

You reused a key with a different body, usually a bug: one key per intended action. Create a new key for the new
action.

### I retried after 25 hours with the same key

Keys are remembered for 24 hours. After that the key is treated as new, and the request runs again. Do not retry an
order that old: first check `GET /operations/{id}` or `GET /domains/{name}`.

## Availability and price

### `GET /check` said available, but `POST /domains` answers `409 domain_not_available`

Someone registered it in between, or the registry has it reserved. `GET /check` is a hint read from WHOIS, not a
reservation. Offer your client another name.

### The price changed between my quote and the order

Prices are read live from your client group. The charge is the price at the moment of the order, shown in the
operation's `charge`. Run `?dry_run=true` just before ordering to see the exact charge.

### My credit was enough in the dry run, but the order answers `402 insufficient_credit`

Another order spent credit in between, maybe from another of your systems or an automatic renewal. Nothing was
ordered. Top up and retry with a **new** `Idempotency-Key`.

### Two of my servers order at the same moment

Orders of one account are checked one after the other, so credit can never be spent twice: one order passes, the other
answers `402 insufficient_credit` if the credit is not enough for both.

## Holder

### The holder is Moroccan but lives abroad

`moroccan: true`, `legal_id` = CIN, and the foreign address. `moroccan` is nationality (or country of incorporation),
not the address.

### The holder is a foreign company with a Moroccan branch

If the branch is registered in Morocco and holds the name, it is a Moroccan company (`moroccan: true`, ICE / RC).
Otherwise `moroccan: false` with the foreign registration number or tax ID. When you are unsure which applies, ask
Golzak before ordering: ANRT checks the documents against what you declared.

### The holder is an association or a sole trader

The API has two types, `individual` and `company`. An association or any registered organization is `company`. A sole
trader registering in their own name is `individual`.

### My client wants the domain in my company's name

Not possible: the holder must be your client (`422 holder_is_reseller`). Register it in your client's name.

### Can I change the holder or the holder's e-mail after registration?

Not through API v1. Contact Golzak.

## Registration under ANRT review

### The operation has been `pending` with `registry_review` for two weeks

Normal: observed answers took 3 to 18 calendar days. Check that the documents were sent before the deadline in
Golzak's e-mail.
Keep waiting for the webhook; there is nothing to call.

### I want to change the name servers, the lock or get the EPP code while the name is under review

Not yet: the registry refuses changes to a name it holds (observed on ANRT's test registry, 2026-10-08: `2304 Object
status prohibits operation`). Until the name is active, these calls answer `409` `domain_status_prohibits` and nothing
is sent. Repeat the call after the `operation.active` webhook.

### The deadline passed and I did not send the documents

ANRT cancels the request; the operation ends `refused` and the charge is refunded (`refund`).

## Webhooks

### A webhook arrived before the `202` answer of my `POST`

Possible under load. Store events even for operation IDs you do not know yet, and match them when the `202` arrives;
or, on an unknown ID, call `GET /operations/{id}`.

### The same webhook arrived twice

Expected: deliveries are retried until you answer `2xx`. Deduplicate on `Golzak-Webhook-Id`.

### Events arrived out of order (`operation.active` before `operation.pending`)

Keep the state with the newest `updated_at`. A final state (`active`, `refused`, `failed`) never changes back.

### My endpoint was down for a day

Deliveries are retried for 24 hours. After that, your 15-minute reconciliation (`GET /operations/{id}` for every
operation still `pending`) brings you up to date.

### I get an event type I do not know

Answer `200` and ignore it. New event types may be added in v1.

## Domains and renewals

### A domain shows `grace_period`

It has expired but can still be renewed through the API until the grace period ends (`grace_period_days` in
`GET /prices`). Renew now.

### A domain shows `redemption_period`

The API can no longer renew it. Contact Golzak at once to restore it.

### Auto-renew is on but the domain was not renewed

The automatic renewal failed: you received `operation.failed` with the reason (usually credit or the auto-renew cap).
Fix it and renew by hand before `expires_at`.

### I asked for 5 years on a name that already has 2 years left

`422 validation_failed` with `exceeds_max_validity`: a name can never be valid for more than 5 years from today. Renew
for 3 years at most.

### `GET /domains/{name}` answers `404` for a name I can see in WHOIS

The name exists, but not in your account. The API only shows your own domains.

## Unknown values

### A response has a field I do not know

Ignore it. v1 may add fields.

### An error `code` I do not know

Handle it by its HTTP status: `4xx` → show `detail`, do not retry; `5xx` → retry with the same `Idempotency-Key`.
