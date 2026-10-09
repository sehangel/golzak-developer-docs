# Sandbox and testing

**There is no reseller sandbox in v1.** Every key works on the real registry, and every order is real and paid.

To build and test safely:

- **Read-only calls are free to repeat:** `GET /check`, `GET /prices`, `GET /balance`, `GET /domains`,
  `GET /operations/{id}`.
- **`?dry_run=true`** on `POST /domains`, `POST /domains/{name}/renew` and `POST /transfers` runs every check
  (name, holder rules, availability, price, credit, daily cap) and returns the charge it would make, without
  ordering anything. Use it in your tests and right before a real order.
- **Errors without cost:** a wrong holder, an unaccepted extension, a missing `Idempotency-Key` or an unknown domain
  never order anything, so you can test all your error handling for free.
- **Webhooks:** ask Golzak to send a test event to your URL to check your signature code. You can also compute a
  signature yourself with the [recipe](registration-and-webhooks.md#verify-the-signature) and post it to your own
  endpoint.
- **First real order:** do it with a name your client actually wants, after a successful dry run.

Before the API opens, Golzak validates the full registration path (payment from credit, holder sent to the registry,
pending and refusal handling) on the .ma registry's test environment. A reseller sandbox may come in a later version;
it will be announced in the [changelog](changelog.md).
