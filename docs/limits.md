# Limits and what the API does not do

Every number and boundary of the Domains API v1 in one place. If a value here and the [API reference](reference.html)
ever differ, the reference wins.

## Numbers

| What | Limit | What happens past it |
| --- | --- | --- |
| Requests per API key | 60 per minute unless your contract says otherwise; read `RateLimit-Limit` | `429 rate_limited`; wait `Retry-After` seconds |
| Daily spending per API key | The cap you set for the key, per UTC day (00:00–24:00 UTC) | `403 daily_spend_cap_reached`; nothing ordered |
| Daily spending by automatic renewals | The account's auto-renew cap, per UTC day | The renewal operation ends `failed` |
| `years` on register and renew | 1 to 5, and only the counts listed in your price table (`GET /prices`) | `422 validation_failed`, `years_not_offered` |
| Total validity of a name | 5 years from today (ANRT decision 02/2024, art. 33.1) | `422 validation_failed`, `exceeds_max_validity` |
| Nameservers | 2 to 5, all different, lower-case host names | `422 validation_failed`, `too_few` / `too_many` / `duplicate` / `invalid_format` |
| Domain name | One label of 1–63 characters + an accepted extension; lower-case ASCII letters, digits, hyphen; no hyphen at either end | `400 invalid_domain_name` |
| Holder text fields | 255 characters (254 for `email`); no control characters | `422 validation_failed`, `too_long` / `invalid_format` |
| `Idempotency-Key` | 1–255 characters; remembered 24 hours | `428` if missing on POST; `422 idempotency_key_reused` if reused with another body |
| List page size (`limit`) | 1–100, default 50 | `400 invalid_request` |
| Webhook answer time | 10 seconds | Treated as failed and retried |
| Webhook retries | For 24 hours, with increasing delays | Dropped after 24 hours: read `GET /operations/{id}` |
| Webhook timestamp tolerance | 300 seconds | Reject the delivery yourself |
| Transfer in | Must complete within 10 business days | Operation ends `failed`, `transfer_not_completed`, refunded |
| Documents for an ANRT review | Before the deadline in Golzak's e-mail | ANRT cancels the request; operation ends `refused` |

## Accepted extensions in v1

`.ma`, `.co.ma`, `.net.ma`, `.org.ma`. `GET /prices` always lists the extensions you can order.

| Extension | Answer |
| --- | --- |
| `.ac.ma`, `.gov.ma`, `.press.ma` | `422 tld_manual_only`: Golzak registers them by hand; contact Golzak. |
| Anything else (`.com`, `.fr`, …) | `422 tld_not_supported`. |

## What v1 does not do

Plan your integration around these. Each one is a deliberate limit of version 1, not a bug.

| Not in v1 | What to do instead |
| --- | --- |
| A test or sandbox environment | Use `?dry_run=true` and the error cases, which cost nothing ([Sandbox and testing](sandbox.md)). |
| Internationalized names (IDN, e.g. Arabic script) | Contact Golzak. |
| Changing the holder (owner) or the holder's contact details of a registered name | Contact Golzak. |
| Uploading the documents ANRT asks for | Reply to Golzak's e-mail or open a ticket before the deadline in Golzak's e-mail. |
| Deleting or cancelling a registered name | Turn auto-renew off and let it expire, or contact Golzak. |
| Restoring a name in `redemption_period` | Contact Golzak at once. |
| DNS hosting (zone records) | Use any DNS provider and set its nameservers with `PUT /domains/{name}/nameservers`. |
| DNSSEC | Contact Golzak. |
| Calls from a browser (CORS) | Call the API from your server only; keys must never reach a browser. |
| Webhook management through the API | Manage endpoints on developers.golzak.com. |

## Timing you can expect

| Action | Usual time | Notes |
| --- | --- | --- |
| A read (`GET`) | Under a second | |
| Registration of a name ANRT does not examine | Minutes | The operation goes `pending` → `active`. |
| Registration under ANRT review | Plan for up to 3 weeks | Observed 3 to 18 calendar days; see [Registration and webhooks](registration-and-webhooks.md#when-anrt-asks-for-documents). |
| Renewal | Minutes | |
| Transfer in | About 2 business days | Longer if the current registrar opposes (ANRT then decides within 3 more business days). |
| Nameserver change, lock change | Seconds (synchronous) | DNS caches around the world may take up to 48 hours to follow a nameserver change. |
