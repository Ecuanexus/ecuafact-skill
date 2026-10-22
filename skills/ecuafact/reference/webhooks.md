# Webhooks

The platform reports the outcome of each issuance with a `POST` (`Content-Type: application/json; charset=utf-8`) to the destination URL configured in the Client Portal (**Integracion > Eventos**). The destination's signing secret is shown only once, when the destination is created or its secret is rotated. Full guides: `https://docsapi.ecuafact.com/v1/guias/webhooks.md` and `https://docsapi.ecuafact.com/v1/guias/webhooks-firma.md`.

A destination receives events for every taxpayer in the integration; match each event to your record by `idOperacion`. Each environment (staging and production) has its own portal, destinations and secrets, and the body does not say which environment sent it, so use a different URL per environment. The URL must be `https://` on port 443, with no credentials, resolving to a public IP; redirects are not followed.

## Signatures

Every delivery carries the same signed payload in two schemes. Verify **one** of them; both cover the same raw body and use the same destination secret, but the HMAC key is derived differently.

| Scheme | Headers | HMAC key | Signed string | Encoding |
|---|---|---|---|---|
| Ecuafact | `Ecuafact-Signature: t=<epoch_seconds>,v1=<hmac_sha256_hex>` | UTF-8 bytes of the secret text, **not** Base64-decoded | `t + "." + <raw body>` | lowercase hex (64 chars) |
| Standard Webhooks (`https://www.standardwebhooks.com`) | `webhook-id`, `webhook-timestamp`, `webhook-signature` | Base64 **decoded** secret (strip a `whsec_` prefix first) | `webhook-id + "." + webhook-timestamp + "." + <raw body>` | Base64, sent as `v1,<base64>` |

Rules that apply to both schemes:

- Sign over the **raw body** exactly as received (UTF-8 bytes). Frameworks that parse the body (ASP.NET Core model binding, `express.json()`, Flask `get_json()`, Spring POJOs) break the signature; read `Request.Body`, `express.raw({ type: 'application/json' })`, `request.get_data()`, `php://input` or `@RequestBody byte[]`. Never rebuild the JSON.
- Reject the delivery with `401` if a header is missing, the timestamp differs from your clock by more than the tolerance (recommended **300 seconds**; never below 60), or the signature does not match. Compare in **constant time**.
- `t` and `webhook-timestamp` are the same value and are re-generated on every retry, so the tolerance never rejects a legitimate retry. To know when the event happened, use `occurredAtUtc` from the body, not `t`.
- If your server also rejects events by the age of `occurredAtUtc`, allow at least **72 hours**, because retries can arrive up to about 71 hours after the event.
- Only parse the JSON after the signature verifies.

### Ecuafact-Signature

Split the header on commas, trim each part, and read `t` and `v1` by name (ignore any other part). `t` must be an integer. The secret is a 44-character string that looks like Base64, but you use its text as-is: decoding it gives a wrong signature.

```python
import hashlib, hmac, time

def verify_ecuafact(secret: str, header: str, raw_body: bytes, now=None, tolerance=300) -> bool:
    parts = dict(p.strip().split("=", 1) for p in header.split(",") if "=" in p)
    t, v1 = parts.get("t", ""), parts.get("v1", "")
    if not t.isdigit() or not v1:
        return False
    now = int(time.time()) if now is None else now
    if abs(now - int(t)) > tolerance:
        return False
    expected = hmac.new(secret.encode("utf-8"), t.encode("ascii") + b"." + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected.encode("ascii"), v1.lower().encode("utf-8"))

# secret = os.environ["ECUAFACT_WEBHOOK_SECRET"]; header = request.headers["Ecuafact-Signature"]
```

### Standard Webhooks

Every delivery carries both signatures; verifying either `Ecuafact-Signature` or the Standard Webhooks headers is enough.

`webhook-id` is the event identifier and stays the same on every retry and every manual **Reenviar**. `webhook-signature` is a **space-separated list** of signatures, each `v1,<base64>`; only one is sent today, but read the whole list and accept the delivery if any entry matches.

```python
import base64, hashlib, hmac, time

def verify_standard(secret: str, msg_id: str, timestamp: str, signatures: str, raw_body: bytes,
                    now=None, tolerance=300) -> bool:
    if not msg_id or not timestamp.isdigit() or not signatures:
        return False
    now = int(time.time()) if now is None else now
    if abs(now - int(timestamp)) > tolerance:
        return False
    if secret.startswith("whsec_"):
        secret = secret[len("whsec_"):]
    key = base64.b64decode(secret)
    signed = f"{msg_id}.{timestamp}.".encode("utf-8") + raw_body
    expected = base64.b64encode(hmac.new(key, signed, hashlib.sha256).digest()).decode("ascii")
    return any(s.startswith("v1,") and hmac.compare_digest(s[3:].encode("utf-8"), expected.encode("ascii")) for s in signatures.split(" "))

# verify_standard(secret, headers["webhook-id"], headers["webhook-timestamp"], headers["webhook-signature"], raw_body)
```

The official Standard Webhooks libraries (`npm install standardwebhooks`, `pip install standardwebhooks`) accept the secret as the portal shows it or with the `whsec_` prefix and decode it for you: `new Webhook(secret).verify(rawBody, headers)` returns the parsed event or throws `WebhookVerificationError`. They check the timestamp against the current time.

The official Ecuafact SDKs also ship a verifier for `Ecuafact-Signature` (for example `ecuafact.verify(secret, header, body, now=None, tolerance_seconds=300)` in Python); they return `false` instead of throwing, accept uppercase or lowercase `v1`, compare in constant time and accept a list of secrets.

### Test vectors

| Data | Value |
|---|---|
| Secret | `c2VjcmV0by1kZS1wcnVlYmFzLW5vLXVzYXItMzJieXQ=` |
| Raw body (one line, no trailing newline) | `{"idOperacion":"3f1c8c1e-6f2a-4b7e-9c3d-2a1b0c9d8e7f","estado":"authorized","claveAcceso":"0710202601012345678900110010010000001231234567811","eventType":"document.authorized","resourceId":"3f1c8c1e-6f2a-4b7e-9c3d-2a1b0c9d8e7f","occurredAtUtc":"2026-10-07T15:04:05.1234567Z"}` |
| `Ecuafact-Signature` | `t=1791385446,v1=a1ae78d763238168f09290d33a7647b3b3c685ff96ef3b60be203e453d772568` |
| `webhook-id` / `webhook-timestamp` | `9b2e4f7a-1c3d-4e5f-8a6b-7c8d9e0f1a2b` / `1791385446` |
| `webhook-signature` | `v1,6YgtHWzoIeuArJTPh8Oq8lpg//3ldAAsy1IRzD5eiG0=` |

Pass `now=1791385446` when testing with these vectors. Typical wrong results: `b6e6ac50...` for `v1` means you Base64-decoded the secret for `Ecuafact-Signature`; `v1,PVfUF7A2...` means you used the undecoded secret text for Standard Webhooks.

## Body

```json
{
  "idOperacion": "3f1c8c1e-6f2a-4b7e-9c3d-2a1b0c9d8e7f",
  "estado": "authorized",
  "claveAcceso": "0710202601012345678900110010010000001231234567811",
  "eventType": "document.authorized",
  "resourceId": "3f1c8c1e-6f2a-4b7e-9c3d-2a1b0c9d8e7f",
  "occurredAtUtc": "2026-10-07T15:04:05.1234567Z"
}
```

The real body arrives as compact single-line JSON. All values are strings, key order is not guaranteed, and you must not reject an event because it contains a field you do not recognize.

| Field | Present in | Description |
|---|---|---|
| `eventType` | All | Event type |
| `resourceId` | All | For `document.*` it is the `idOperacion`; for the test event it is the destination id |
| `occurredAtUtc` | All | UTC, ISO 8601 with 7 decimals; does not change on retries |
| `idOperacion` | `document.*` | Operation returned by the issuance call |
| `estado` | `document.*` | `authorized`, `rejected` or `failed`. This is not `estadoAutorizacion`: `authorized` corresponds to `autorizado` and `rejected` to `error` |
| `claveAcceso` | `document.authorized` and `document.rejected`; in `document.failed` only if already assigned | 49 digits |
| `motivo` | `document.failed` | See the event table. Treat an unknown value as a rejection to review |
| `codigoError` | `document.failed`, when there is an associated error | Always `error_servicio` |
| `probe` | Test event | Always `"true"` |

## Events and what to do

| Event / `motivo` | What happened | What to do |
|---|---|---|
| `document.authorized` | The SRI authorized the document | Mark it authorized, store `claveAcceso`, download the RIDE if needed |
| `document.rejected` | Fiscal rejection by the SRI | Fix the document and issue it as a **new** document: **new** `secuencial` (or omit it to get the next one), new `Idempotency-Key` and new `referenciaExterna`. The rejected number cannot be reused (`409` with `codigo` `401`) |
| `document.failed` / `estructura_invalida` | The fiscal service rejected the structure before sending it to the SRI | Fix it and issue again with the **same** number, new `Idempotency-Key` and new `referenciaExterna` |
| `document.failed` / `suscripcion_inactiva` | The issuer has no active subscription in the fiscal service; nothing was sent to the SRI | Fix the subscription and issue again with the same number, new key and new reference. The attempt's quota is consumed |
| `document.failed` / `plan_agotado` | The issuer's plan in the fiscal service reached its limit | Extend or renew the plan and issue again with the same number, new key and new reference |
| `document.failed` / `emisor_no_vinculado` | The issuer's connection to the SRI is not configured in the fiscal service | Complete the link and issue again with the same number, new key and new reference |
| `document.failed` / `entrega_no_posible` | The platform could not deliver the document to the fiscal service and gave up; the document was not created | Check `GET /v1/operaciones/{idOperacion}`; if `estadoAutorizacion` is not `autorizado`, issue again with the same number, new key and new reference |
| `document.failed` / `resultado_desconocido` | Several attempts without confirming the result; **not final** while the platform keeps trying | **Do not issue again** (you could duplicate the document). Wait for a later `document.authorized` or `document.rejected`, or check the operation; if it arrives after send retries are exhausted, contact support with the `idOperacion` |
| `operation.requires_attention` | Test event from the portal's **Probar** button (`probe` = `"true"`) | Respond `2xx` |

To reissue "with the same number", resend the corrected document with the same `estab` and `ptoEmi` and set `infoTributaria.secuencial` to the previous operation's sequence number (last part of its `uid`). Reusing the old `Idempotency-Key` or `referenciaExterna` returns the original operation (`201`) or `409` with `codigo` `403` and does not issue. The accepted resend (`202`, `codigo` `200`) reuses the **same operation** (`idOperacion`), and since at most one event per `eventType` is generated per operation, confirm the final result through the operation endpoint.

`document.failed` is **not** an SRI rejection. Respond `2xx` and ignore any `eventType` you do not recognize, so unknown types do not count as failures or pause the destination.

## Delivery guarantees, retries and pausing

- **At least once.** Deduplicate with the unique key `eventType` + `resourceId` (retries and re-sends keep it; `t` and the signature change). `webhook-id` is also stable across retries and **Reenviar**, so it works as a dedup key too. A single operation can legitimately receive two different events: `document.failed` with `resultado_desconocido` and later `document.authorized` or `document.rejected`; process both, the final one replaces the `failed`. A table with a composite primary key, inserted in the same transaction that applies the effect, is enough.
- **No ordering.** To know the current state, call `GET /v1/operaciones/{idOperacion}` instead of trusting the last event received.
- **Response.** Any `2xx` within **15 seconds** counts as delivered (your response body is ignored); redirects are not followed. Persist the event, respond quickly and process in the background. If your own save fails, respond `500` so the delivery is retried.
- **Retried failures:** no response (timeout, DNS failure, connection refused, TLS error), `408`, `429` or any `5xx`. If you send `Retry-After`, the platform waits at least that long, up to 60 minutes.
- **Rejected by your server (final, no retries, does not count toward pausing):** any other `4xx` (400, 401, 403, 404, 410, 422...) or any `3xx`. Fix your endpoint and use **Reenviar** in the portal. For a temporary problem on your side (database down, deployment), respond `503` with `Retry-After`, never a `4xx`.
- **Retry ladder:** up to **8 attempts** in total, waiting 1 min, 5 min, 30 min, 2 h, 8 h, 24 h and 36 h between attempts (each wait randomized to 50-100 % of its value), so an event is retried for at most about **71 hours**. After 8 failed attempts the delivery is **Agotado** (exhausted); **Reenviar** forces a new attempt. Every retry carries the same body, the same `occurredAtUtc` and the same `webhook-id`, with a new signature (new `t` and `webhook-timestamp`).
- **Pausing.** After **8 consecutive counted failures** (no response, `408`, `429`, `5xx`) across all events of the destination, it becomes **Pausado**. Rejections (`3xx`, other `4xx`) do not count. A later `2xx` resets the counter but does **not** resume the destination: use **Reanudar** in the portal. While paused, new events are neither delivered nor queued, and after resuming only new events arrive, so reconcile using the operation endpoint or the issued-documents query (`comprobantes emitidos`). A clock drifting beyond the tolerance rejects every delivery and can lead to a pause.

## Secret rotation

**Rotar secreto** shows the new secret once, and from then on **every** delivery (including retries of older events) is signed **only** with the new secret; no second signature with the old one is sent. To rotate without losing deliveries:

1. Make your server accept a **list of secrets** (valid if any matches). With a Standard Webhooks library, create one verifier per active secret.
2. Rotate, add the new secret to the list without removing the old one, and deploy as soon as possible.
3. Deliveries that fail verification in between are retried (1, 5, 30 minutes...); check **Entregas** for HTTP 200.
4. Once deliveries verify with the new secret, remove the old one. If the secret leaked, remove it as soon as the new one is deployed.

## Testing

- The portal's **Probar** sends `operation.requires_attention` to that destination; confirm in **Entregas** that it shows **Entregado** and HTTP 200.
- Locally, sign a fake delivery with `ecuafact webhooks probar --a http://localhost:3000/webhooks/ecuafact` (reads `ECUAFACT_WEBHOOK_SECRET`; `--evento document.rejected`, `--firma-invalida`), or check a signature with `ecuafact webhooks verificar`. See `reference/herramientas.md`.
- To receive real events on your machine, expose the port with a tunnel (`ngrok http 3000`, `cloudflared tunnel --url http://localhost:3000`) and create a destination in the staging portal. Free tunnel URLs change on restart: create a new destination and disable the old one, or it will accumulate failures and pause.
