# Changelog

## v1.0.0

First published specification of the Golzak Domains API, version 1, at `https://api.golzak.com/domains/v1`. The API
itself is not open yet: Golzak announces go-live to its resellers.

- Availability, prices, registration (asynchronous, with ANRT review), renewal, automatic renewal (off by default),
  nameservers, transfer lock, EPP code, transfers in, credit balance, operations, signed webhooks.
- `.ma` holder rules enforced by the API; idempotency keys on every `POST`; dry runs on every paid call; per-key rate
  limits, daily spending caps and required IP allowlists; RFC 9457 errors with stable codes.
