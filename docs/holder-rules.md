# Holder rules

The **holder** (registrant) is the person or company that owns the domain. The API enforces these rules on every
registration; a request that breaks one is refused with `422 validation_failed` and nothing is ordered.

## The holder is your client

The holder must be **your client**: the person or company that will own and use the name. Never your own company,
and never a registrar's staff member. The API refuses a holder whose e-mail or organization name is your own
account's (`422 holder_is_reseller`); your reseller contract covers the rest.

## Who the holder is: `type` and `moroccan`

| Field | Values | Meaning |
| --- | --- | --- |
| `holder.type` | `individual` or `company` | A person, or a company / organization. |
| `holder.moroccan` | `true` or `false` | A Moroccan individual (Moroccan nationality), or a company registered in Morocco. |

`moroccan` decides which identity document is asked for. It is **not** taken from the address: a Moroccan citizen
living in Istanbul is `moroccan: true` with a Turkish address.

## Which legal ID: `legal_id`

`holder.legal_id` is **always required**. What it must contain depends on the case. This is the same text Golzak's
.ma registration form shows today:

| `type` | `moroccan` | `legal_id` must be |
| --- | --- | --- |
| `individual` | `true` | the holder's **CIN** (Moroccan national identity card number) |
| `individual` | `false` | the holder's **passport number** |
| `company` | `true` | the **company tax ID** (ICE / RC) |
| `company` | `false` | the **company registration number or tax ID** |

The API does not check the number's format; the registry (ANRT) checks the documents. Send the number exactly as
written on the document. Leading and trailing spaces are removed.

Every text field of the holder is limited to 255 characters (254 for `email`) and may not contain control characters
(line breaks, tabs, etc.). These are input-safety limits of the API, not ANRT rules: too long answers `too_long`, a
control character answers `invalid_format`.

## Other required data

| Field | Rule |
| --- | --- |
| `first_name`, `last_name` | Required. For a company: the contact person. |
| `organization` | Required when `type` is `company` (the company's legal name). Optional for an individual. |
| `email` | Required. The holder's own e-mail. |
| `phone` | Required, written `+<country code>.<number>`, e.g. `+212.612345678` (the EPP format of RFC 5733). |
| `address.line1`, `address.city`, `address.country` | Required. `country` is the ISO 3166-1 two-letter code, upper case (`MA`, `TR`). |
| `address.line2`, `address.state`, `address.postal_code` | Optional. |

## What each mistake returns

All mistakes are reported together in one `422 validation_failed`, one entry per broken rule in `errors[]`. Branch on
`errors[].field` and `errors[].code`; `message` is for humans.

| Mistake | `field` | `code` | `message` |
| --- | --- | --- | --- |
| Moroccan individual, no `legal_id` | `holder.legal_id` | `required` | A Moroccan individual needs their CIN number in holder.legal_id. |
| Non-Moroccan individual, no `legal_id` | `holder.legal_id` | `required` | A non-Moroccan individual needs their passport number in holder.legal_id. |
| Moroccan company, no `legal_id` | `holder.legal_id` | `required` | A Moroccan company needs its company tax ID (ICE / RC) in holder.legal_id. |
| Foreign company, no `legal_id` | `holder.legal_id` | `required` | A foreign company needs its company registration number or tax ID in holder.legal_id. |
| Company without `organization` | `holder.organization` | `required` | A company holder needs its legal name in holder.organization. |
| `type` missing or not `individual` / `company` | `holder.type` | `required` / `invalid_value` | holder.type must be individual or company. |
| `moroccan` missing or not a boolean | `holder.moroccan` | `required` / `invalid_value` | holder.moroccan must be true or false. |
| Phone not `+CC.NUMBER` | `holder.phone` | `invalid_format` | Write the phone as +<country code>.<number>, e.g. +212.612345678. |
| Country not a two-letter code | `holder.address.country` | `invalid_format` | Use the ISO 3166-1 two-letter country code, e.g. MA. |
| Missing name, e-mail, line1 or city | that field | `required` | holder.\<field\> is required. |

The holder is your own company: `422 holder_is_reseller` (not a field error).

Worked requests and answers for every case are in the [AI integration guide](ai-integration-guide.md).

## Extensions and their purpose (ANRT)

ANRT's decision ANRT/DG/N°02/2024 (art. 5.2) describes the descriptive extensions: `.net.ma` notably for Internet
service providers, `.org.ma` for associations, foundations and similar organizations, `.co.ma` for commercial
activities ([official text, in French](https://www.anrt.ma/sites/default/files/2024-06/decision_02-2024_-_ndd_fr_.pdf)).
The API does not enforce these purposes; choose the extension that fits your client.

## When ANRT examines a name

ANRT examines some requests before creating the name, notably when the name conflicts with its list of reserved terms
(same decision, art. 32). Then it needs an examination form and proof of the holder's right to the name. Golzak
e-mails you the form and the list of documents; the operation shows `documents_required: true` and a
the deadline in its e-mail. See [Registration, operations and webhooks](registration-and-webhooks.md).

## Transfers

A transfer keeps the holder registered at the registry; `POST /transfers` takes no holder.
