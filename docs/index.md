# Golzak Domains API

Register and manage domains for your clients through Golzak, from your own systems or from an AI agent.

- **Extensions in v1:** `.ma`, `.co.ma`, `.net.ma`, `.org.ma`. Other `.ma` extensions (`.ac.ma`, `.gov.ma`,
  `.press.ma`) are handled by hand: contact Golzak. More extensions may come in later versions; `GET /prices` always
  lists the ones you can order.
- **Prices:** yours, from your client group at Golzak, read live on every call (`GET /check`, `GET /prices`).
- **Payment:** your prepaid Golzak credit. No credit, no order.
- **Registration is asynchronous:** the registry (ANRT) may examine a name before it creates it. You get an
  operation, then a signed webhook when it ends.
- **Safe by default:** every paid call can be dry-run, every `POST` is idempotent, nothing renews or charges unless you
  ask, and keys only work from your servers' IP addresses.

## Building with an AI agent?

Give your agent the [AI integration guide](ai-integration-guide.md) and the [OpenAPI file](openapi.yaml), or point it
at [llms.txt](llms.txt). Together they hold every rule, example and error the agent needs; it should not have to guess
anything.

## Read in this order

1. [Quick start](quick-start.md): first calls in five minutes.
2. [Workflows](workflows.md): every task as numbered steps (register, review, renew, transfer in and out, keys).
3. [Authentication](authentication.md): API keys, IP allowlist, idempotency, rate limits, daily cap.
4. [Holder rules](holder-rules.md): who owns the domain and which legal ID to send. Read this before registering.
5. [Registration, operations and webhooks](registration-and-webhooks.md): the asynchronous lifecycle.
6. [Expiry, renewal and transfers out](lifecycle.md): auto-renew, grace period, moving a name away.
7. [Pricing and credit](pricing-and-credit.md).
8. [Errors](errors.md): every error code, what it means, whether to retry.
9. [Limits](limits.md): every number, and what the API does not do.
10. [Edge cases](edge-cases.md): timeouts, races, duplicates, unusual holders, missed webhooks.
11. [Sandbox and testing](sandbox.md).
12. [AI integration guide](ai-integration-guide.md): everything an AI coding agent needs, with worked examples.

The [API reference](reference.html) is generated from the OpenAPI file, which you can [download](openapi.yaml) and feed
to a code generator.

## Base URL

```text
https://api.golzak.com/domains/v1
```

All of Golzak's APIs share the host `api.golzak.com`; each product has its own prefix and version. Paths in these docs
(`GET /check`, `POST /domains`, …) are relative to this base URL: `GET /check` is
`GET https://api.golzak.com/domains/v1/check`.

## Versions

This is version 1 of the Domains API. These docs are published per version at `developers.golzak.com/domains/v1/`;
`developers.golzak.com/domains/latest/` always points to the newest. Within v1, changes only add things (new optional
fields, new endpoints, new error codes); anything that would break an integration goes to v2. See the
[changelog](changelog.md).
