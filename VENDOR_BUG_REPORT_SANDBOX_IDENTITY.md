# Sandbox test-identity matching fails for token-bound identities

**Partner ID:** 8711 · **Environment:** sandbox (`testapi.smileidentity.com`) · **Product:** `biometric_kyc`
**Reported:** 2026-08-18 · **Reproduced with `curl` alone** — no SDK involved

---

## Summary

When the identity fields are bound into the auth token at `POST /v3/token`, the sandbox's
test-identity matcher never matches, and every submission is rejected with:

> In sandbox mode, requests must use a recognized test identity. Please check the documentation for
> valid test identities.

Sending the **same** identity as plaintext `user_details` in the request body instead is accepted
immediately. The identity values are byte-identical in both cases; the only difference is where they
travel.

We believe the matcher compares against the raw `pii_`-prefixed vault references rather than the
values they resolve to, so a token-bound identity cannot match by construction.

## Impact

Token binding is the security control that makes the mobile flow safe — per your own documentation
for `POST /v3/token`:

> Downstream v3 endpoints inject these fields server-side, **overriding any matching fields in the
> request body**, so data bound to the token cannot be altered by the client.

That is exactly why we bind: the app forwards a server-authoritative identity and cannot assert a
different person. But because binding makes sandbox verification impossible, we currently cannot
test the flow we intend to ship. The only way to get a green sandbox run is to stop binding — which
is the configuration we specifically do **not** want in production.

Production is unaffected (test-identity matching only runs in sandbox), so this blocks testing
rather than live traffic.

---

## Reproduction

Three submissions. Everything is held constant — same ID number, images, consent, callback URL,
partner and product. The **only** variable is where the identity travels.

Test identity used, from your Biometric KYC table (expected outcome `clear`):

| Last Name | Given Names | Email |
|---|---|---|
| `Clearwater` | `Amina Fatou` | `amina.clearwater@example.com` |

### Variant A — identity bound into the token → **400**

```bash
# Mint with the identity bound
curl -X POST https://testapi.smileidentity.com/v3/token \
  -H "smileid-partner-id: 8711" -H "smileid-api-key: $KEY" \
  -F 'product=biometric_kyc' \
  -F 'payload={"country":"NG","id_type":"NIN_V2","id_number":"00088855500",
               "given_names":"Amina Fatou","last_name":"Clearwater",
               "email":"amina.clearwater@example.com"}'

# Submit — no user_details in the body, because the token binds them
curl -X POST https://testapi.smileidentity.com/v3/biometric_kyc \
  -H "smileid-token: $TOKEN_A" -H "smileid-partner-id: 8711" \
  -F "selfie_image=@selfie.jpg;type=image/jpeg" \
  -F "liveness_images=@liveness.jpg;type=image/jpeg" \
  -F 'consent={"granted":true,"granted_at":"2026-08-18T05:00:00Z","notice_language":"EN",
               "notice_privacy_policy_url":"https://earnberries.com/privacy"}' \
  -F "callback_url=https://staging.rivabit.com/webhooks/smile-id/v3"
```

```json
HTTP 400
{"message":"In sandbox mode, requests must use a recognized test identity. Please check the documentation for valid test identities.","status":"Bad Request"}
```

The token's `payload` claim for this request:

```json
{
  "country": "NG",
  "id_type": "NIN_V2",
  "email":       "pii_01krejnyg8e05abg2d0mv8q30x",
  "given_names": "pii_01krejnyg0e4f80wb5wqpkz7dt",
  "id_number":   "pii_01m09jyhmfexts4gz6nq05w77m",
  "last_name":   "pii_01krejnyj6e04rwpcmfn8z5de4"
}
```

### Variant B — nothing bound, identity in the body → **202**

Same ID number, same images, same consent, same callback URL.

```bash
curl -X POST https://testapi.smileidentity.com/v3/token \
  -H "smileid-partner-id: 8711" -H "smileid-api-key: $KEY" \
  -F 'product=biometric_kyc'          # no payload — nothing bound

curl -X POST https://testapi.smileidentity.com/v3/biometric_kyc \
  -H "smileid-token: $TOKEN_B" -H "smileid-partner-id: 8711" \
  -F "selfie_image=@selfie.jpg;type=image/jpeg" \
  -F "liveness_images=@liveness.jpg;type=image/jpeg" \
  -F 'consent={...}' \
  -F 'user_details={"given_names":"Amina Fatou","last_name":"Clearwater","email":"amina.clearwater@example.com"}' \
  -F "country=NG" -F "id_type=NIN_V2" -F "id_number=00088855500" \
  -F "callback_url=https://staging.rivabit.com/webhooks/smile-id/v3"
```

```json
HTTP 202
{"created_at":"2026-08-18T05:00:26.445Z","job_id":"job_01m09kw0qbee7txsq7r0zg56f3",
 "message":"Request accepted and queued for processing.","status":"accepted",
 "user_id":"user_01m09kw0qcer5vdt6fnf1m7j0s"}
```

### Variant C — bound **and** also sent in the body → **400**

Variant A's token, with `user_details` added to the body as well.

```json
HTTP 400
{"message":"In sandbox mode, requests must use a recognized test identity. …","status":"Bad Request"}
```

This is the important one: **supplying the plaintext alongside does not help.** The token binding
overrides the body as documented, and the matcher then fails anyway — so there is no client-side
workaround while the identity is bound.

---

## The bound references are correct

We verified the binding is not the problem before concluding the matcher is. Minting a token with
the test identity typed out literally returns **byte-identical** references to the ones our backend's
token carried:

```
email        pii_01krejnyg8e05abg2d0mv8q30x
given_names  pii_01krejnyg0e4f80wb5wqpkz7dt
last_name    pii_01krejnyj6e04rwpcmfn8z5de4
```

As a control, binding a *different* test identity (`Rashid Omar` / `Dangerfield` /
`rashid.dangerfield@example.com`) returns a different set of references, while an unchanged
`id_number` keeps its reference. So the vault is content-addressed — the same value always yields
the same reference — and `POST /v3/token` is genuinely reading the payload we send.

In other words: the correct identity **was** bound, and it still did not match.

The reference ULIDs also carry their creation time. The `Clearwater` references date from
2026-05-12 and the `Dangerfield` ones from 2026-05-15 — presumably when your test identities were
first seeded. That is consistent with content-addressing and is not a staleness problem on our side.

---

## What we think is happening

`external_docs`-side we can only observe behaviour, but the pattern fits a matcher that reads
`payload.given_names` / `payload.last_name` / `payload.email` **before** the vault references are
resolved — comparing the literal `pii_…` strings against the test-identity table, which can never
match. Variant C suggests the token values take precedence at that point, so the body's plaintext is
already discarded by the time matching runs.

## What we would like

1. **Confirmation** of whether sandbox test-identity matching is expected to resolve `pii_`
   references. If it is, this is a bug; if it is not, please document that sandbox testing requires
   an unbound token, because the two features are currently mutually exclusive.
2. **A fix**, so a token-bound identity can be tested in sandbox — this is the configuration we
   intend to run in production, and today it is the one configuration we cannot verify.
3. **Interim guidance**: is there a supported way to exercise a bound identity in sandbox, e.g. via
   `sandbox_result`, or partner-level allowlisting of our test identities?

---

## Environment

| | |
|---|---|
| Partner ID | 8711 |
| Environment | sandbox — `https://testapi.smileidentity.com` |
| Product | `biometric_kyc` |
| Reproduced with | `curl` only (also observed via SDK, below) |
| Mobile SDK | `@smileid/usesmileid` 12.0.2, React Native / Expo |
| Host | Expo SDK 57, React Native 0.86.2, Android 15 |

The same behaviour occurs through the SDK. With our backend's token (identity bound) the submission
returns the 400 above; swapping in an unbound token minted on the device — everything else identical
— returns 202 and the verification completes end to end, webhook included
(`job_01m09k00jnervaw4ced78c3b6q`).

For the record, this partner account previously hit the `UseSmileIDBridge` packaging issue, which
**12.0.2 resolved** — the module now publishes `android/`, `ios/` and `expo-module.config.json` and
links correctly on device. Thank you for that fix; this report is unrelated to it.

---

## Minor, unrelated observation

The SDK builds its request URL from the token's `api_url` claim, which already ends in `/v3`, and
then appends `/v3/biometric_kyc` — producing a double slash:

```
POST https://testapi.smileidentity.com//v3/biometric_kyc
```

The API tolerates it, so this is cosmetic, but you may want to normalise the join.
