# Errors, limits and support

Full guides: `https://docsapi.ecuafact.com/v1/guias/errores.md`, `https://docsapi.ecuafact.com/v1/guias/limites.md` and `https://docsapi.ecuafact.com/v1/guias/limite-de-request.md`.

## Error format

```json
{ "codigo": "301", "mensaje": "Solicitud invalida.", "errores": ["Idempotency-Key: La cabecera Idempotency-Key es obligatoria y debe ser ASCII estable (maximo 128 caracteres)."] }
```

- `codigo`: stable, three digits; the first digit is the category (`0` system, `1` access, `2` success, `3` validation, `4` conflict, `5` state). Decide with the pair HTTP status + `codigo`, never with `mensaje`.
- `errores` (list) is always present: on validation errors (`3xx`) it holds one item, the first problem found, often prefixed with the field path (`$.infoTributaria.estab: ...`; some paths appear with English names such as `items.code`). It is informational and may change: log or display it, never compare it in code. For every other code it is `[]`.
- Successful queries (`200`) carry no `codigo`; the `2xx` codes appear only in the `202` body of an issuance.
- The `codigo` is not the HTTP status: `409` with `codigo` `401` means "duplicate document", not an authentication error.
- Some responses have no JSON error body because the request never reached an endpoint (they still carry `X-Correlation-Id`): `404` (the route does not exist: typo, missing `/v1`, or an `{idOperacion}` that is not a UUID), `405` (HTTP method not allowed on the route), `415` (body sent without `Content-Type: application/json`).

## Request and response headers

| Header | Rule |
|---|---|
| `X-Api-Key` | Required on every `/v1` route. Missing or invalid: `401` (`101`) |
| `Idempotency-Key` | Required on issuance. 1 to 128 characters; only unaccented ASCII letters (`A-Z`, `a-z`), digits (`0-9`) and `-` `_` `.` `:` `/`. Missing or invalid: `400` (`301`). Remembered for 7 days per integration and environment |
| `X-Api-Version` | Optional. If sent, it must be the current contract version, `2026-10-03`; any other value returns `400` (`306`) before authentication. Recommended: do not send it, and log the `X-Api-Version` value that comes back in every `/v1` response |
| `X-Correlation-Id` | Returned on every response, success or error: store it, it is the first thing support asks for. Optional on the request: if you send one with 1 to 64 characters from `A-Z`, `a-z`, `0-9`, `.`, `_`, `-`, the API uses it and returns the same value (for example your order number); otherwise the API generates a new one. Never an error |
| `X-Additional-Info` | Optional free text (for example `MiERP/4.2 sucursal=norte lote=42`) that lets support find your request even without the `X-Correlation-Id`. One header per request, printable ASCII only (0x20 to 0x7E; no accents, `ñ` or line breaks), at most 256 characters. If it does not comply the request is not rejected, but support cannot use it to locate the request. It is not stored with the document nor returned in the response. Never include personal data or secrets |

## Code catalog

| Code | HTTP | Meaning | What to do | Retry |
|---|---|---|---|---|
| `001` | 500 | Internal error (on RIDE/XML/email/profile it can also be a communication failure with the service behind the endpoint) | Retry with exponential backoff; if it persists, report the `X-Correlation-Id` | Yes, same `Idempotency-Key` |
| `002` | 503 | Service temporarily unavailable | Wait for `Retry-After` (5 s) and retry | Yes, same `Idempotency-Key` |
| `003` | 502 | An external service returned an error in a synchronous call (SRI queries, issued documents, catalogs, RIDE or XML download, issuer profile, email resend) | Retry with backoff; if it persists, report the `X-Correlation-Id` | Yes |
| `004` | 504 | An external service did not answer in time (SRI queries, issued documents, catalogs, issuer profile) | Retry with backoff | Yes |
| `005` | 501 | Not implemented: issuance sent with `Content-Type` `application/xml` or `text/xml` | Send the document structure as JSON with `Content-Type: application/json` | No |
| `101` | 401 | Missing `X-Api-Key`, API Key unknown, revoked or expired, **or** source IP not registered | Check the API Key and register the IP in the Client Portal, **Cuenta > IP permitidas** | No |
| `102` | 403 | The API could not identify the source IP (rare) | Call the API URL directly, without intermediate proxies | No |
| `103` | 403 | Taxpayer or document type not enabled for your integration, 13-digit RUC used in the path, `infoTributaria.ruc` with the right 10-digit prefix but a different 3-digit suffix, or plan missing or expired | Check `GET /v1/contexto`; use `identificacion` in the path and `identificacionCompleta` in `ruc`; renew an expired plan | No |
| `104` | 429 | Request limit exceeded: your integration's per-minute limit, 20 failed authentications per minute from your IP, or the source of an SRI query throttled requests | Wait at least `Retry-After` and retry the same request | Yes, after `Retry-After` (same `Idempotency-Key`) |
| `200` | 202 | New issuance received | Store `idOperacion` and track the operation | - |
| `201` | 202 | Idempotent retry: same `Idempotency-Key` (or same `referenciaExterna`) with the same body; returns the original operation. If that operation is `cancelado`, it is returned as is and nothing is issued | Track the same `idOperacion`; to issue after `cancelado`, send a new `Idempotency-Key` and a new `referenciaExterna` | - |
| `301` | 400 | Invalid request: malformed JSON, `Idempotency-Key` missing or invalid, path or query parameter of the wrong type, invalid email recipient | Read `errores` and fix | No |
| `302` | 422 | Invalid document (`Comprobante invalido.`): missing field, wrong format, totals or taxes that do not add up, fields of another type inside `info`, unknown or repeated field, you sent `version`, `formato`, `ambiente` or `tipoEmision`, or `infoTributaria.ruc` does not start with the path `identificacion`. Consumes neither quota nor sequential | Read `errores` (it carries the field path), fix the document and resend | No |
| `302` | 400 | Email resend of a document not yet authorized | Wait for `estadoAutorizacion` = `autorizado` | No |
| `303` | 400 | Query filter out of range (dates, `pagina`, `tamanoPagina`, `buscar`, `codDoc` or `max`) | Adjust the filters | No |
| `305` | 413 | Body too large: over 2 MiB on issuance or 1 MiB on other `POST`/`PUT` | Reduce the body | No |
| `306` | 400 | `X-Api-Version` with a value other than the current version `2026-10-03` | Send `2026-10-03` or omit the header | No |
| `401` | 409 | Duplicate document: same taxpayer, type, `estab`, `ptoEmi` and `secuencial` already in process, authorized or rejected by the SRI. Only the number of an operation that ended in `document.failed` can be reused | Do not resend; find it in issued documents. For a different document use another `secuencial` or omit it | No |
| `403` | 409 | Idempotency or reference conflict: same `Idempotency-Key` (or same `referenciaExterna` with the same `origenReferencia`) with a different body, or the `referenciaExterna` already belongs to another document of your integration | If it is a retry, resend the exact original body; if it is another issuance, use a new key and a new reference | No |
| `501` | 429 | Quota exhausted (`Cupo agotado.`): no documents left in plan or packages | Check `GET /v1/consumo` and increase the quota. Waiting for `Retry-After` does not restore it | No, until quota is increased |
| `502` | 404 | Resource not found (`idOperacion`, access key, catalog, RIDE not available yet, or no plan in this environment) | Check the identifier and the environment URL; for a just-authorized RIDE, retry later | No (RIDE: yes, later) |

Two catalog codes are never returned by the public API: `304` (422, invalid environment) and `503` (409, reconciliation required). If one arrives, do not resend the document; store the `X-Correlation-Id`, check the operation and contact support.

Treat an unknown `codigo` by its HTTP status: fix on `4xx`, retry on `5xx`.

## 429: rate limit (`104`) versus quota (`501`)

Both come back as HTTP `429`; always tell them apart by `codigo`.

- `429` + `104` is the request rate limit. The request was not processed (nothing was issued), so retry the same request after waiting **at least** `Retry-After` seconds (1 to 60, the rest of the current clock minute); on issuance resend the same body with the same `Idempotency-Key`. When your IP hit 20 failed authentications, fix the API Key or register the IP before retrying.
- `429` + `501` is document quota exhaustion and only happens on issuance. It also carries `Retry-After` (60), but waiting does not restore quota: do not retry in a loop; stop and get the quota increased (check `GET /v1/consumo`).

## Issuance errors

After authentication, `POST /v1/contribuyentes/{identificacion}/comprobantes` checks in this order and returns the **first** failure; none of these consumes quota.

| Order | Code | HTTP | When | What to do |
|---|---|---|---|---|
| 1 | `305` | 413 | Body over 2 MiB | Reduce the document |
| 2 | `005` | 501 | `Content-Type` is XML | Send JSON |
| 3 | - | 415 | `Content-Type` is not JSON (no body) | Add `Content-Type: application/json` |
| 4 | `301` | 400 | Malformed JSON | Fix the JSON |
| 5 | `302` | 422 | Fiscal rule, structure, unknown field, JSON depth over 32, or `infoTributaria.ruc` whose first 10 digits differ from the path `identificacion` (`errores`: `$.infoTributaria.ruc: El RUC no corresponde al contribuyente del contexto.`) | Read `errores` and fix; use `identificacionCompleta` from `GET /v1/contexto` as `ruc` |
| 6 | `301` | 400 | `Idempotency-Key` missing, longer than 128 or with characters outside the allowed set | Fix the header |
| 7 | `103` | 403 | Taxpayer, document type or plan not enabled, or `infoTributaria.ruc` with the same 10-digit prefix but different last 3 digits (for example `0123456789002`) | Use `identificacionCompleta` from `GET /v1/contexto`; check `tiposComprobante` and the plan |
| 8 | `403` | 409 | Same `Idempotency-Key` or `referenciaExterna` with different content | Resend the original content, or use a new key and reference |
| 9 | `401` | 409 | Number (`estab`-`ptoEmi`-`secuencial`) already used: in process, authorized or rejected by the SRI | Look up the existing document or omit `secuencial` |
| 10 | `501` | 429 | Quota exhausted | Increase your quota; do not retry |
| - | `200` | 202 | New issuance accepted (reserves one document) | Track the operation |
| - | `201` | 202 | Idempotent retry (nothing new reserved) | Track the same operation |
| - | `002` / `001` | 503 / 500 | Temporary failure | Retry with the same `Idempotency-Key` |

A `202` means **received**, not authorized. An SRI rejection arrives later (tracking with `estadoAutorizacion` `error`, or the `document.rejected` webhook) and is not an HTTP error.

## Retry rules

- Retry with the same body and the same `Idempotency-Key`, with exponential backoff plus jitter: network timeouts or drops with no response, `500` (`001`), `502` (`003`), `503` (`002`, honor `Retry-After`), `504` (`004`).
- Retry after waiting at least `Retry-After`: `429` (`104`).
- Never retry unchanged: `429` (`501`), `400`, `401`, `403`, `404`, `413`, `415`, `422`, `409` (`401`: the number exists, find the document; `403`: resend the original body or use a new key for a new issuance). Exception: a `404` (`502`) on a just-authorized RIDE can be retried later.
- On a `GET` query, retrying simply means repeating it.

## Limits

| Limit | Value | Error |
|---|---|---|
| Requests per integration (all your API Keys share the counter; every authenticated `/v1` request counts, success or error) | 200 per minute by default; 60 in a sandbox. Fixed clock-minute window (hh:mm:00 to hh:mm:59), not sliding. The value that applies to you is `X-RateLimit-Limit` | `429`, `codigo` `104` |
| Failed authentications (`401`/`101`) per IP | 20 per minute; once reached, **every** request from that IP gets `429` until the minute ends, even with a valid key. A successful authentication before reaching 20 resets the counter | `429`, `codigo` `104` |
| Issuance body | 2 MiB (2,097,152 bytes) | `413`, `codigo` `305` |
| Body of other `POST`/`PUT` | 1 MiB (1,048,576 bytes) | `413`, `codigo` `305` |
| JSON depth | 32 levels | Issuance: `422`/`302`; others: `400`/`301` |
| `origenReferencia` | Required, 1 to 64 characters from the `Idempotency-Key` character set | `422`, `codigo` `302` |
| `referenciaExterna` | Required, 1 to 128 characters from the `Idempotency-Key` character set | `422`, `codigo` `302` |
| `detalles` | 1 to 1000 lines; up to 3 `detallesAdicionales` per line | `422`, `codigo` `302` |
| `info.pagos` | 1 to 20 in invoice and purchase settlement | `422`, `codigo` `302` |
| `infoAdicional` | Up to 20 `{ "nombre", "valor" }` objects | `422`, `codigo` `302` |

Rate-limit headers: `X-RateLimit-Limit` and `X-RateLimit-Remaining` on responses to authenticated requests (not on `401`; `X-RateLimit-Limit` is 20 on the failed-authentication `429`), `X-RateLimit-Reset` (Unix epoch seconds, only on `429` with `104`), `Retry-After` (on every `429` and `503`).

- If `X-RateLimit-Remaining` reaches 0, do not send the next request until `X-RateLimit-Reset`.
- Spread bulk issuance across the minute (for example 3 per second instead of 200 at once), cap concurrency, and use webhooks instead of polling operations.
- Document quota is separate from the request limit (`501` vs `104`).

## How to request support

Report: the `X-Correlation-Id` (most important), the `X-Additional-Info` value if you sent one, the approximate date and time, the full request URL (it shows the environment), the HTTP status, `codigo`, `mensaje` and `errores`, the `idOperacion` if the issuance returned `202`, and the body you sent **without** the API Key. Never include the API Key or the webhook secret in a ticket, email or chat.
