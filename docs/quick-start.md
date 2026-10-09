# Quick start

You need an **API key**. When the API opens, sign in to [developers.golzak.com](https://developers.golzak.com) with
your Golzak account (the reseller account Golzak set up for your contract), create a key with your servers' IP
addresses, and add a webhook endpoint (you get its signing secret). See [Authentication](authentication.md#api-key).
Keep both secrets on your server, never in a browser or a mobile app.

```bash
export GOLZAK_KEY='<your API key>'
export API='https://api.golzak.com/domains/v1'
```

## 1. Check a name and your price

```bash
curl -s "$API/check?name=atlas-tours.ma" -H "Authorization: Bearer $GOLZAK_KEY"
```

```json
{
  "name": "atlas-tours.ma",
  "extension": ".ma",
  "available": true,
  "price": { "operation": "register", "years": 1, "amount": "<your price, read live>", "currency": "<your account currency>" }
}
```

Money amounts in these docs are placeholders: real figures come from your client group and are read live.

## 2. Check your credit

```bash
curl -s "$API/balance" -H "Authorization: Bearer $GOLZAK_KEY"
```

## 3. Try the registration without ordering

`dry_run=true` runs every check (name, holder rules, availability, price, credit, daily cap) and orders nothing.

```bash
curl -s -X POST "$API/domains?dry_run=true" \
  -H "Authorization: Bearer $GOLZAK_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "atlas-tours.ma",
    "years": 1,
    "holder": {
      "type": "individual",
      "moroccan": true,
      "legal_id": "<CIN number>",
      "first_name": "Salma",
      "last_name": "Bennani",
      "email": "salma.bennani@example.com",
      "phone": "+212.612345678",
      "address": { "line1": "12 Rue Example", "city": "Casablanca", "postal_code": "20000", "country": "MA" }
    },
    "nameservers": ["ns1.example-dns.com", "ns2.example-dns.com"]
  }'
```

The holder is **your client**, and `legal_id` depends on who they are: read the [holder rules](holder-rules.md).

## 4. Register

Same call without `dry_run`. Generate **one** `Idempotency-Key` per registration and reuse it if you retry.

```json
{
  "id": "op_01JA7X3M9Q2W8E5R6T7Y8U9I0P",
  "type": "register",
  "initiator": "api",
  "status": "pending",
  "pending_reason": "processing",
  "domain": "atlas-tours.ma"
}
```

(Shortened; the full operation object is in the [reference](reference.html).)

## 5. Wait for the result

Your webhook receives `operation.active` or `operation.refused`, or you read `GET /operations/{id}`. If the registry
examines the name you first get `operation.pending` with `documents_required: true`: see
[Registration, operations and webhooks](registration-and-webhooks.md).
