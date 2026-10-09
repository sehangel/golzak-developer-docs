# Expiry, renewal and transfers out

## Renewal

A name stays registered until its `expires_at` (`GET /domains/{name}`). Renew it before that date.

- **By hand:** `POST /domains/{name}/renew` with `years`. You pay your renew price from your credit, like a
  registration. The new `expires_at` shows on the domain once the operation is `active`.
- **Automatically:** `PUT /domains/{name}/auto-renew` with `{"enabled": true}`. Golzak then renews the name for one
  year shortly before `expires_at`, at your renew price, from your credit. The renewal appears as an operation with
  `initiator: "auto_renew"` and is reported by webhook. It counts against your account's daily cap for automatic
  renewals (set on developers.golzak.com), never against a key's cap, so revoking a key does not stop auto-renewals.
  If your credit or that cap does not allow it, the operation ends `failed` (webhook `operation.failed`): renew by
  hand before `expires_at`.
- **Default: off.** Every domain registered or transferred in through the API starts with `auto_renew: false`, so
  nothing is ever charged without a call from you.
- **Limit:** a name can never be valid for more than 5 years from today (ANRT decision ANRT/DG/N°02/2024, art. 33.1).
  A renewal that would pass that limit answers `422 validation_failed` with `exceeds_max_validity`.

ANRT requires the registrar to remind the holder of the expiry date at least 30 days before it (same decision,
art. 33.2). Golzak e-mails these reminders to your account; pass them on to your client.

## After `expires_at`

| Domain `status` | What it means | What you can do |
| --- | --- | --- |
| `grace_period` | Expired. The registry freezes the name; only its nameservers can still change (art. 33.3). | Renew through the API until the grace period ends. Its length per extension is `grace_period_days` in `GET /prices`. |
| `redemption_period` | The grace period is over and Golzak has asked the registry to delete the name. | Not renewable through the API. Contact Golzak at once: Golzak can restore it during `redemption_period_days` (`GET /prices`), at the restore price Golzak quotes. |
| `expired` | Deleted, or about to be. | Nothing. Once the registry deletes it, the name becomes free for anyone to register. |

The lengths of both periods are read live from Golzak's billing system; never hard-code them.

## Moving a name to another registrar

The holder may move their name to another registrar (ANRT decision 02/2024, art. 35) while it is `active`: under ANRT
review, in `grace_period` (renew it first) or later, the lock and the EPP code answer `409 domain_status_prohibits`.
Golzak locks every name against transfer after registration and after a transfer in, so a move takes two calls:

1. `PUT /domains/{name}/transfer-lock` with `{"locked": false}`.
2. `GET /domains/{name}/epp-code`: give the code **to the holder only**. Each call sets a new code at the registry and
   cancels the previous one.

The holder gives the code to the new registrar, which requests the transfer. Under art. 35 the losing registrar has
two business days to accept or oppose it; without an answer the transfer completes automatically. ANRT also requires
the registrar to hand over the code and lift the lock within two business days of the holder's request: the two calls
above do it at once.

If the move is abandoned, lock the name again with `{"locked": true}`. Once the name has left, it shows
`status: "transferred_away"`.

## Moving a name to Golzak

`POST /transfers` with the name and the EPP code from the current registrar. Before ordering, make sure the name is
unlocked there and is not expired. The current registrar has two business days to answer (art. 35); if ANRT has to
decide an opposition, it takes up to three more business days. If the transfer has not completed 10 business days
after your request, the operation ends `failed` with `transfer_not_completed` and the charge is returned.

A transferred name keeps its remaining validity and its holder data (art. 35). It arrives with `auto_renew: false`
and `transfer_locked: true`.

Source: [ANRT decision ANRT/DG/N°02/2024](https://www.anrt.ma/sites/default/files/2024-06/decision_02-2024_-_ndd_fr_.pdf)
(in French), articles 33 and 35.
