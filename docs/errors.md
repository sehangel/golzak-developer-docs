# Errors

Errors use the problem-details format of [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html), media type
`application/problem+json`, with one extra member, `code`:

```json
{
  "type": "https://developers.golzak.com/domains/v1/errors#insufficient_credit",
  "title": "Insufficient credit",
  "status": 402,
  "code": "insufficient_credit",
  "detail": "Top up your Golzak account credit, then retry with a new Idempotency-Key. Nothing was ordered.",
  "request_id": "req_01JA7X3M9Q2W8E5R6T7Y8U9I0P"
}
```

**Branch on `code`.** Codes are stable for the whole of v1: never renamed or removed, only added. **A code you do not
know is not a bug:** handle it by its HTTP status (`4xx`: show `detail`, do not retry; `5xx`: retry with the same
`Idempotency-Key`). The same goes for other lists that v1 may extend (domain statuses, `pending_reason`, operation
types, webhook event types): treat an unknown value as unknown, never as an error. `title` and
`detail` are for humans and can change. `validation_failed` adds `errors[]`, one entry per broken rule, each with
`field`, `code` and `message`.

> **Retry rule in one line:** `429`, `500`, `502` and network timeouts → retry with the **same** `Idempotency-Key`;
> every other error → do not retry as is; fix the cause, and send a **new** key for the corrected request.

## Order of checks on a POST

A registration is checked in this order; the first failing step answers, and nothing is ordered:

1. API key and IP (`401`, `403`), rate limit (`429`), `Idempotency-Key` present (`428`).
2. Request syntax (`400 invalid_request`) and host-name syntax (`400 invalid_domain_name`).
3. Extension (`422 tld_manual_only`, `tld_not_supported`), then one label plus the extension
   (`400 invalid_domain_name`).
4. Body rules, holder rules, years, nameservers (`422 validation_failed`, all broken rules at once), then
   `422 holder_is_reseller`.
5. The domain's state, except on a dry run: another operation still running on it (`409 operation_in_progress`),
   then an earlier order of it still being checked (`409 previous_order_unresolved`).
6. Availability (`409 domain_not_available`).
7. Credit (`402 insufficient_credit`), then daily cap (`403 daily_spend_cap_reached`).

## All codes

| HTTP | `code` | Meaning | What to do | Retry? |
| --- | --- | --- | --- | --- |
| 400 | <a id="invalid_request"></a>`invalid_request` | Malformed JSON, unknown field, bad query parameter. | Fix the request. | No |
| 400 | <a id="invalid_domain_name"></a>`invalid_domain_name` | Not a lower-case ASCII host name, or not exactly one label plus an accepted extension (e.g. `a.b.ma`, or a bare `co.ma`). | Send e.g. `example.ma`, `example.co.ma`. | No |
| 401 | <a id="invalid_api_key"></a>`invalid_api_key` | Key missing, unknown or revoked. | Check the `Authorization` header. | No |
| 403 | <a id="ip_not_allowed"></a>`ip_not_allowed` | The key is limited to other IP addresses. | Call from an allowed address, or ask Golzak. | No |
| 403 | <a id="account_suspended"></a>`account_suspended` | Your reseller account is suspended. | Contact Golzak. | No |
| 403 | <a id="daily_spend_cap_reached"></a>`daily_spend_cap_reached` | The order would exceed the key's daily cap. Nothing ordered. | Wait for `cap_resets_at`, or ask Golzak. | After `cap_resets_at`, new key |
| 402 | <a id="insufficient_credit"></a>`insufficient_credit` | Credit does not cover the charge. Nothing ordered. | Top up; retry with a new key. | After top-up, new key |
| 404 | <a id="route_not_found"></a>`route_not_found` | The method and path are not an endpoint of the Domains API v1. | Check the path against the reference; paths start with `/domains/v1`. | No |
| 404 | <a id="domain_not_found"></a>`domain_not_found` | No such domain in your account. | Check the name; list with `GET /domains`. | No |
| 404 | <a id="operation_not_found"></a>`operation_not_found` | No such operation in your account. | Check the ID. | No |
| 409 | <a id="domain_not_available"></a>`domain_not_available` | The name is already registered. Nothing ordered. | Suggest another name. | No |
| 409 | <a id="operation_in_progress"></a>`operation_in_progress` | Another operation is still running on this domain. | Wait for it to end. | After it ends |
| 409 | <a id="previous_order_unresolved"></a>`previous_order_unresolved` | An earlier order for this domain did not complete cleanly (usually a `failed` operation; a request that never answered leaves none), and Golzak is still checking its billing. New orders for the domain wait until then: usually within the hour; at least a day when the billing system never answered the earlier order; longer when Golzak must settle it by hand. Nothing was ordered. | Retry later with a new key; contact Golzak if it lasts. | Later, new key |
| 409 | <a id="domain_status_prohibits"></a>`domain_status_prohibits` | The domain's state forbids this change (e.g. under ANRT review, expired). | Wait, or contact Golzak. | When the status allows |
| 409 | <a id="idempotency_key_in_use"></a>`idempotency_key_in_use` | A call with this key is still running. | Retry the same call shortly. | Yes, same key, after a few seconds |
| 422 | <a id="idempotency_key_reused"></a>`idempotency_key_reused` | Same key, different body. | Use a new key for a new action. | No (new key for a new action) |
| 428 | <a id="idempotency_key_required"></a>`idempotency_key_required` | POST without `Idempotency-Key`. | Send one (a UUID). | Yes, with a key |
| 422 | <a id="tld_not_supported"></a>`tld_not_supported` | This API version does not offer the extension. | In v1: `.ma`, `.co.ma`, `.net.ma`, `.org.ma` (`GET /prices` lists them). | No |
| 422 | <a id="tld_manual_only"></a>`tld_manual_only` | `.ac.ma`, `.gov.ma`, `.press.ma`: handled by hand. | Contact Golzak. | No |
| 422 | <a id="validation_failed"></a>`validation_failed` | One or more rules broken; see `errors[]`. Nothing ordered. | Fix every listed field. | After fixing, new key |
| 422 | <a id="holder_is_reseller"></a>`holder_is_reseller` | The holder is your own company. | Register in your client's name. | No |
| 422 | <a id="registry_refused"></a>`registry_refused` | (In an operation's `error`.) ANRT refused the name. | Tell your client; the charge is refunded. | No |
| 422 | <a id="transfer_not_completed"></a>`transfer_not_completed` | (In an operation's `error`.) The transfer did not complete within 10 business days. | Check the lock and the EPP code with the current registrar; order again. The charge is refunded. | New order, new key |
| 502 | <a id="registry_error"></a>`registry_error` | The registry rejected the command (e.g. bad nameserver). | Read `detail`; fix and retry with a new key. | After fixing, new key |
| 429 | <a id="rate_limited"></a>`rate_limited` | Too many requests. | Wait `Retry-After` seconds. | Yes, after `Retry-After`, same key |
| 502 | <a id="upstream_unavailable"></a>`upstream_unavailable` | Golzak's billing system or the registry did not answer. | Retry with the **same** key. | Yes, same key |
| 503 | <a id="orders_disabled"></a>`orders_disabled` | Orders are switched off for a while (maintenance or an incident); reads and dry runs still work. Nothing was ordered. | Retry later. | Yes, same key, later |
| 500 | <a id="internal_error"></a>`internal_error` | Bug on Golzak's side. | Retry with the same key; report the `request_id`. | Yes, same key |

## Field error codes in `validation_failed`

| `errors[].code` | Meaning |
| --- | --- |
| `required` | The field is missing or empty. |
| `invalid_value` | Not one of the allowed values (e.g. `holder.type`). |
| `invalid_format` | Wrong format (phone, country, host name, e-mail), or a control character in text. |
| `too_long` | Longer than the field's limit (255 characters for most text fields). |
| `years_not_offered` | `years` is not in your price table for this operation. |
| `exceeds_max_validity` | The name would be valid for more than 5 years from today (ANRT decision 02/2024 art. 33.1). |
| `too_few` / `too_many` | Nameserver count outside 2 to 5. |
| `duplicate` | The same nameserver twice. |

The holder-specific messages are listed in [Holder rules](holder-rules.md#what-each-mistake-returns).
