# Pricing and credit

## Where your prices come from

Your account at Golzak is in its own **client group**, set up when your contract is signed. The group's discount
gives your reseller price for every extension and operation. Prices are **read live** from Golzak's billing system on
every call: they are never written in these docs or in the API's code.

| To know | Call |
| --- | --- |
| Your 1-year registration price for one name | `GET /check?name=...` → `price` |
| Every price: register, renew, transfer, per number of years | `GET /prices` |

The year counts listed in `GET /prices` are the only ones you can order; another value answers `422 validation_failed`
with `code: years_not_offered`. Transfer prices come from the same place as the others.

A price can change between your quote and your order. The charge is always the price at the moment of the order,
returned in the operation's `charge`. Use `?dry_run=true` right before ordering to see it.

> **Tax:** prices are before tax. When Golzak's billing system adds tax for your account (for example Moroccan VAT
> for a company registered in Morocco), the operation's `charge` is the invoice total, tax included, and the credit
> check uses that total. A dry run's `would_charge` shows the price before tax.

## Prepaid credit

You pay from the **credit balance** of your Golzak account.

- **Top up** by paying Golzak (bank transfer or another method agreed in your contract); Golzak adds it to your credit.
- **Each registration, renewal or transfer** takes its price from the credit when the API accepts the order.
- **Not enough credit:** `402 insufficient_credit`. Nothing is ordered and nothing is charged. Top up, then retry
  with a **new** `Idempotency-Key`.
- **Daily spending cap:** a key can also have a cap per UTC day: `403 daily_spend_cap_reached` (see
  [Authentication](authentication.md#daily-spending-cap)).
- **Refunds:** a registration refused by ANRT, or an operation that failed, has its charge returned; the operation's
  `refund.status` says `credited` (back on your credit) or `manual_review` (Golzak returns it by hand).

```bash
curl -s "$API/balance" -H "Authorization: Bearer $GOLZAK_KEY"
```

```json
{
  "credit": { "amount": "<your balance>", "currency": "<your account currency>" },
  "daily_spend_cap": { "amount": "<cap set for this key>", "currency": "<your account currency>" },
  "spent_today": { "amount": "<spent since 00:00 UTC>", "currency": "<your account currency>" },
  "cap_resets_at": "2026-10-09T00:00:00Z"
}
```

## Money format

Amounts are **strings** holding a decimal number (e.g. two decimals), with an ISO 4217 `currency`: your account's
currency. Parse them with a decimal type, never a float.
