# Workflows

Every task the API supports, as numbered steps. Paths are relative to `https://api.golzak.com/domains/v1`; every call
sends `Authorization: Bearer <API key>`. Each `POST` also sends a new `Idempotency-Key` that you store **before** the
call and reuse if you retry it.

> **Rule for every workflow:** on `429`, `500`, `502` or a network timeout, retry the **same** request with the
> **same** `Idempotency-Key` (after `Retry-After` when given). On any other `4xx`, do not retry: show `detail` and fix
> the cause. See [Errors](errors.md).

## 1. Register a domain for a client

1. `GET /check?name=atlas-tours.ma`
   - `available: false` → stop; offer another name.
   - `422 tld_manual_only` / `tld_not_supported` → stop; the extension is not available through the API.
2. `GET /prices` → check that the number of years you want is a key of the `register` table; read the price.
3. Collect the holder's data from your client. Ask, do not guess, whether the holder is a person or a company and
   whether they are Moroccan; this decides `legal_id` ([Holder rules](holder-rules.md)).
4. `POST /domains?dry_run=true` with the full body.
   - `200` → shows `would_charge` and `credit_after`. Show the price to your client if you need their approval.
   - Any error → fix it and repeat this step. Nothing has been ordered.
5. `POST /domains` with the same body and a **new** `Idempotency-Key` → `202` with an operation.
6. Store the operation `id`. Show your client "registration in progress".
7. Wait for the result (webhook, or `GET /operations/{id}`; see workflow 9):
   - `active` → done. `GET /domains/{name}` gives `expires_at`.
   - `pending` with `pending_reason: registry_review` → go to workflow 2.
   - `refused` → workflow 3.
   - `failed` → read `error.code`; the charge is in `refund`. Fix the cause and start again from step 1.

## 2. A registration held for ANRT review

1. You receive `operation.pending` with `documents_required: true` (and `documents_deadline` when known).
2. Golzak e-mails your account the official ANRT form and the list of documents for this holder.
3. Tell your client at once. Collect the documents.
4. Reply to Golzak's e-mail (or open a ticket) with the documents **before the deadline in that e-mail**.
5. Wait. Plan for up to 3 weeks. The operation ends `active` or `refused`.

> **Note:** do not order the same name again while it is under review: the registry already holds it for your
> client. Wait for the operation's result.

## 3. A registration refused by ANRT

1. The operation is `refused`, `error.code: registry_refused`.
2. `refund.status` is `credited` (the charge is back on your credit) or `manual_review` (Golzak returns it by hand).
3. Tell your client. The name is free again at the registry. Before ordering it again, ask Golzak why ANRT refused it:
   an order that does not address the reason (for example, missing proof of the holder's right to the name) is likely
   to be refused again.

## 4. Renew a domain by hand

1. `GET /domains/{name}` → note `expires_at` and `status` (`active` or `grace_period` can be renewed).
2. `GET /prices` → check that `years` is in the `renew` table and that the new expiry stays within 5 years of today.
3. `POST /domains/{name}/renew?dry_run=true` with `{"years": 1}` → check the charge.
4. `POST /domains/{name}/renew` with a new `Idempotency-Key` → `202` operation.
5. Wait for `active`; `GET /domains/{name}` shows the new `expires_at`.

## 5. Turn automatic renewal on or off

1. Make sure your account's auto-renew cap on developers.golzak.com covers your renew prices.
2. `PUT /domains/{name}/auto-renew` with `{"enabled": true}` (or `false`) → `200` with the domain, `auto_renew` updated.
3. Before each expiry Golzak creates a renew operation with `initiator: "auto_renew"`; you get `operation.active` or
   `operation.failed`.
4. On `operation.failed` for an automatic renewal: top up credit or raise the cap, then renew by hand (workflow 4)
   before `expires_at`.

## 6. Change the nameservers

1. `PUT /domains/{name}/nameservers` with `{"nameservers": ["ns1.example-dns.com", "ns2.example-dns.com"]}`.
2. `200` → the registry has the new list. DNS caches may take up to 48 hours to follow.
3. `409 domain_status_prohibits` → the name is under ANRT review, in redemption or otherwise not changeable; retry once
   it is `active`. In `grace_period` the nameservers can still change; the registry freezes everything else
   ([ANRT decision 02/2024](https://www.anrt.ma/sites/default/files/2024-06/decision_02-2024_-_ndd_fr_.pdf), art. 33.3).

## 7. Move a domain from another registrar to Golzak (transfer in)

1. Ask the holder to get the EPP code from the current registrar and to have the name **unlocked** there.
2. `GET /prices` → read the transfer price.
3. `POST /transfers?dry_run=true` with `{"name": "...", "epp_code": "..."}` → check the charge.
4. `POST /transfers` with a new `Idempotency-Key` → `202` operation, `pending_reason: transfer_pending`.
5. Wait: usually about 2 business days.
   - `active` → the name is in your account, `auto_renew: false`, `transfer_locked: true`.
   - `failed` with `transfer_not_completed` (after 10 business days) or another code → the charge is refunded; check
     the lock and the code with the holder and the current registrar, then start again.

## 8. A client leaves: move a domain to another registrar (transfer out)

The name must be `active`: the lock and the EPP code answer `409 domain_status_prohibits` in any other status (in
`grace_period`, renew it first).

1. `PUT /domains/{name}/transfer-lock` with `{"locked": false}`.
2. `GET /domains/{name}/epp-code` → give `epp_code` to the **holder only**. Each call makes a new code and cancels the
   previous one: call it once.
3. The holder gives the code to the new registrar, which requests the transfer.
4. Once the name has left, `GET /domains/{name}` shows `status: "transferred_away"`.
5. If the holder changes their mind, lock it again: `PUT /domains/{name}/transfer-lock` with `{"locked": true}`.

## 9. Follow operations reliably (webhooks + reconciliation)

1. Add a webhook endpoint on developers.golzak.com; store its signing secret on your server.
2. For each delivery: verify the signature on the raw body, store the event (deduplicate on `Golzak-Webhook-Id`),
   answer `200`, then process it. See [Registration and webhooks](registration-and-webhooks.md#webhooks).
3. Every 15 minutes, for each operation still `pending` in your database, call `GET /operations/{id}` and update it if
   `updated_at` is newer. This catches any webhook you missed.

## 10. Keep enough credit

1. `GET /balance` daily (or before large batches).
2. If `credit` is low, pay Golzak to top up; the new credit shows in `GET /balance`.
3. If `spent_today` is close to `daily_spend_cap`, wait for `cap_resets_at` or raise the cap on developers.golzak.com.

## 11. Rotate an API key without downtime

1. On developers.golzak.com, create a new key with the same IP allowlist.
2. Deploy it to your servers; check one `GET /balance` with it.
3. Revoke the old key. Webhooks and auto-renewals are not affected: they belong to your account.

## 12. Show your clients their domains

1. `GET /domains?limit=100`; then `GET /domains?limit=100&cursor=<next_cursor>` until `next_cursor` is `null`.
2. For details of one name (nameservers, lock, auto-renew): `GET /domains/{name}`.
3. Flag every domain whose `expires_at` is within 30 days and `auto_renew` is `false`.
