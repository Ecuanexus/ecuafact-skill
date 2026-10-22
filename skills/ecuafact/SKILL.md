---
name: ecuafact
description: Integrates the Ecuafact electronic invoicing API for Ecuador's tax authority (SRI). Covers issuing invoices (factura 01), purchase settlements (liquidacion de compra 03), credit notes (nota de credito 04), debit notes (nota de debito 05), delivery guides (guia de remision 06) and withholding receipts (comprobante de retencion 07); operation tracking; issued documents; RIDE (PDF) and XML downloads; email resend; issuer profile; error catalog and limits; SRI registry lookups (taxpayer, establishments, ID validation, access key decoding); signed webhooks (Ecuafact-Signature and Standard Webhooks); SDKs, CLI, Postman collection and MCP server. Use when integrating e-invoicing in Ecuador, issuing or querying SRI documents, validating a RUC or cedula, or decoding an access key. Triggers: Ecuador e-invoicing, electronic invoice Ecuador, SRI, RUC, cedula, access key, factura, nota de credito, nota de debito, retencion, guia de remision, liquidacion de compra, comprobante electronico, clave de acceso, X-Api-Key, Idempotency-Key, RIDE, webhook, mynexusapi, ecuafact.
---

# Ecuafact electronic invoicing (SRI, Ecuador)

Ecuafact is a REST API for issuing and querying electronic tax documents (comprobantes electronicos) with Ecuador's tax authority, the **SRI**. Your system sends the document as JSON, and the platform validates it, signs it, transmits it to the SRI and generates the RIDE (the PDF representation).

The API contract is in Spanish: field names, routes, error codes, messages and catalog values stay exactly as written here (`infoTributaria`, `codDoc`, `estadoAutorizacion`, `autorizado`...). Never translate them in code or payloads. Reply to the user in the language they write in.

Rules for the assistant:

- Do not invent fields, codes or limits. If something is not in this skill or in the guides, say so and point to the documentation.
- Confirm with the user before issuing a document or resending an email; both have real effects.
- Label test examples with "Ambiente de pruebas: sin validez fiscal".
- Never show or write the API Key or the webhook secret; use the environment variables `ECUAFACT_API_KEY` and `ECUAFACT_WEBHOOK_SECRET`.
- Use only test data: the issuer and any buyer or supplier with a cedula use `0123456789` (type `05`), with a RUC `0123456789001` (type `04`); consumidor final is type `07` with `9999999999999`, up to an `importeTotal` of 50.00. These values are fictitious and fail the modulo-10 check, so a validation lookup returns `valido:false` for them; in your own sandbox, use the identification and RUC that `GET /v1/contexto` returns.

## How to use this skill

This skill is a multi-file package in the **Agent Skills** format (`SKILL.md` plus resources in `reference/`). It works in any assistant that loads Agent Skills, such as Claude Code, Cursor or GitHub Copilot, reading the resources on demand. Start here and open a resource only when the task needs it.

| Resource | Read it when you... |
|---|---|
| `reference/codigos-sri.md` | need an exact SRI code (identification type, VAT rate, payment method, withholding) |
| `reference/emision/factura-01.md` | issue or validate an **invoice** (`01`) |
| `reference/emision/liquidacion-03.md` | issue a **purchase settlement** (`03`) |
| `reference/emision/nota-credito-04.md` | issue a **credit note** (`04`) |
| `reference/emision/nota-debito-05.md` | issue a **debit note** (`05`) |
| `reference/emision/guia-remision-06.md` | issue a **delivery guide** (`06`) |
| `reference/emision/retencion-07.md` | issue a **withholding receipt** (`07`) |
| `reference/consultas.md` | query context, operations, issued documents, quota, catalogs, RIDE/XML, email resend or the issuer profile |
| `reference/consultas-sri.md` | query the SRI registry (taxpayer, establishments, identification, access key) |
| `reference/webhooks.md` | configure webhooks or verify their signature |
| `reference/errores-limites.md` | handle errors, rate limits, quota or support |
| `reference/herramientas.md` | use the SDKs, the `ecuafact` CLI, the Postman collection or the MCP server |

## If your assistant has internet access

The current documentation wins whenever it differs from this skill. The most direct way to read it is the public **documentation MCP server**, which needs no credentials: `https://docsapi.ecuafact.com/mcp` (Streamable HTTP). Connect it with `claude mcp add --transport http ecuafact-docs https://docsapi.ecuafact.com/mcp` or your client's equivalent. All its tools are read-only:

| Tool | Use it to |
|---|---|
| `buscar_documentacion` | find the guide for a topic (`consulta`, optional `maximo`) |
| `leer_guia` | read a guide or one section of it (`slug`, optional `seccion`) |
| `explicar_error` | explain an error `codigo` from the catalog (for example `302`) |
| `ejemplo_comprobante` | get a valid JSON example by type (`01`, `03`, `04`, `05`, `06`, `07`) |
| `consultar_referencia` | read the OpenAPI contract of an endpoint (`ruta`, optional `metodo`) |
| `listar_guias` | list every guide with its slug |

Without MCP, fetch these over HTTP:

- `https://docsapi.ecuafact.com/llms.txt`, the documentation index for LLMs.
- `https://docsapi.ecuafact.com/llms-full.txt`, the whole documentation in one file.
- `https://docsapi.ecuafact.com/v1/guias/<slug>.md`, one guide in Markdown (for example `emision-01.md`).

## Environments

| Environment | Base URL | Use |
|---|---|---|
| Test (sandbox) | `https://staging-api.mynexusapi.com` | Develop and test. No fiscal validity. Free sign-up: `https://docsapi.ecuafact.com/v1/guias/sandbox.md` |
| Production | `https://api.mynexusapi.com` | Issue real documents to the SRI. |

The URL defines the environment, and each environment has its own API Key. Do not send `ambiente` in the body (`422`). The contract, fields and error codes are the same in both.

`infoTributaria.secuencial` is optional: if you omit it, send `""` or send zeros (`000000000`), the platform assigns the next number in the series, in both environments. A number already used by a document that is in process, authorized or rejected by the SRI returns `409` (`codigo` `401`); only a number from an operation that ended in `document.failed` can be reused. There is no `claveAcceso` field in the request; the platform assigns the access key, and any unknown field returns `422`.

## Authentication and headers

```bash
curl -s "https://staging-api.mynexusapi.com/v1/contexto" \
  -H "X-Api-Key: $ECUAFACT_API_KEY"
```

- The API Key identifies your integration and covers every taxpayer authorized for it. Do **not** map ERP users or permissions onto it.
- Call the API only from your backend. Besides the key, the API checks the **source IP**: register your server's public egress IP in the Customer Portal under **Cuenta > IP permitidas**. An invalid key or an unregistered IP returns `401` (`codigo` `101`).
- First call: `GET /v1/contexto` to list your taxpayers. `identificacion` (the first 10 digits of the RUC) goes in the routes; `identificacionCompleta` (the 13-digit RUC) goes in `infoTributaria.ruc`.
- RUC mismatch: if the first 10 digits of `infoTributaria.ruc` differ from the route's `{identificacion}`, the API answers `422` (`codigo` `302`). If the prefix matches but the last 3 digits do not belong to the enabled taxpayer, it answers `403` (`codigo` `103`).

Optional request headers:

| Header | Rule |
|---|---|
| `X-Api-Version` | A date `yyyy-mm-dd`. The current version is `2026-10-03`, and every response returns it in this header. Omitting it is recommended; a value that does not match a published version returns `400` (`codigo` `306`), checked before authentication. |
| `X-Correlation-Id` | Your own trace ID: 1 to 64 characters from `A-Za-z0-9._-`. If you omit it or it is invalid, the API generates one (an invalid value is replaced, never rejected). Every response returns it; store it for support. |
| `X-Additional-Info` | Printable ASCII (0x20 to 0x7E), up to 256 characters, sent once: integrator text (for example `MiERP/4.2 sucursal=001`) that helps support locate the request. It does not change the response, is not stored with the document, is not returned and never rejects the request. No personal data or secrets. The MCP server does not forward it. |

## Lifecycle and idempotency

1. **Send** the document and receive `202` with an `idOperacion`.
2. The platform validates, signs and transmits it to the SRI. `estado` moves through `en_cola`, `enviando` and `enviado`; it can also be `resultado_desconocido`, and the final states are `rechazado` and `cancelado`.
3. **Track** it with `GET /v1/operaciones/{idOperacion}` or by webhook (`document.authorized`, `document.rejected`, `document.failed`). Webhooks are the primary mechanism; polling is the fallback.
4. The SRI **authorizes**: `estadoAutorizacion` is `no_disponible`, `pendiente`, `autorizado` (the only fiscally valid result), `error` (rejected by the SRI) or `desconocido`.
5. Once `autorizado`, download the RIDE or the XML with the `claveAcceso` returned by the tracking call.

The `202` is **not** the SRI authorization; the fiscal result arrives by webhook or through the operation.

Idempotency: the `Idempotency-Key` header is required when issuing. It has 1 to 128 characters from `A-Za-z0-9-_.:/`; a missing key returns `400`, and a key that breaks the format returns `400` (`codigo` `301`). Store `idOperacion`, `Idempotency-Key` and `claveAcceso` next to your document. When you retry (for example after a timeout), send the **same** key with the **same** content: nothing is duplicated and you get the same operation back (`codigo` `201`). The same key with different content returns `409` (`codigo` `403`). For a different document, including the correction of a rejected one, use a new key and a new `referenciaExterna`. Keys are remembered for 7 days per integration and environment.

## Issuing

```
POST /v1/contribuyentes/{identificacion}/comprobantes
Content-Type: application/json
X-Api-Key: <API Key>
Idempotency-Key: <stable intent key>
```

- The body cannot exceed **2 MiB** (`413`, `codigo` `305`) or 32 nesting levels (`422`, `codigo` `302`).
- `Content-Type: application/xml` returns `501` (`codigo` `005`): send JSON.
- Every type shares the same structure; what changes is `infoTributaria.codDoc`, the fields of `info` and the collection.

| Key | Content |
|---|---|
| `origenReferencia` | Source system (1 to 64 characters; ASCII letters and digits plus `-` `_` `.` `:` `/`) |
| `referenciaExterna` | Stable reference from your system (1 to 128, same alphabet) |
| `infoTributaria` | `ruc` (13), `codDoc`, `estab` (3, with leading zeros), `ptoEmi` (3, with leading zeros), `secuencial` (9, optional) |
| `info` | Header of the type; only the fields of that `codDoc` |
| `detalles` / `motivos` / `destinatarios` / `docsSustento` | Collection for the type |
| `infoAdicional` | Up to 20 `{nombre, valor}` objects |

Cross-cutting rules: amounts are JSON decimal numbers (dot separator, no quotes); codes and identifications are strings; dates are `dd/MM/yyyy` (send today's date in `fechaEmision`; the withholding receipt `07` is the only type that keeps the date you send); fiscal periods are `MM/yyyy`. The platform assigns the 49-digit **access key**; do not send it. Do not send `ambiente`, `tipoEmision`, `version` or `formato` either (`422`). Per-type fields and examples are in `reference/emision/*`.

**Validate before issuing.** `POST /v1/contribuyentes/{identificacion}/comprobantes/validar` takes the same body and applies the same rules as issuing (structure, formats, line and header taxes, totals, payments, RUC and document type enabled) without issuing, without `Idempotency-Key` and without using quota. It answers `200` with `{"codigo":"200","mensaje":"Comprobante valido.","errores":[]}`, or exactly the `422`/`302` or `403`/`103` that issuing would return. It does not check quota, plan validity or duplicates, and it counts toward the per-minute rate limit. It is optional, since issuing applies the same rules and a `422` uses no quota. It pays off when you build documents yourself (from files, spreadsheets or a user's description), especially in batches or when portal confirmation is on: validate, fix what fails, and only then ask the user to confirm. With the CLI, `ecuafact validar <archivo> --en-linea` and `emitir --dry-run` use it; on MCP, call `validar_comprobante`.

`202 Accepted` response:

```json
{
  "codigo": "200",
  "mensaje": "Operacion exitosa.",
  "idOperacion": "3f1c8c1e-6f2a-4b7e-9c3d-2a1b0c9d8e7f",
  "urlEstado": "/v1/operaciones/3f1c8c1e-6f2a-4b7e-9c3d-2a1b0c9d8e7f",
  "uid": "0123456789001-01-1-002-001-000000124"
}
```

`codigo` `200` means a new issuance and `201` an idempotent retry (HTTP `202` in both cases). `urlEstado` and the `Location` header are **relative** paths: prepend the base URL. `uid` is `{ruc}-{codDoc}-{ambiente}-{estab}-{ptoEmi}-{secuencial}`. Store the `X-Correlation-Id` response header as well.

```bash
# Ambiente de pruebas: sin validez fiscal
curl -s -X POST "https://staging-api.mynexusapi.com/v1/contribuyentes/0123456789/comprobantes" \
  -H "X-Api-Key: $ECUAFACT_API_KEY" \
  -H "Idempotency-Key: FACTURA-2026-0001" \
  -H "Content-Type: application/json" \
  -d @comprobante.json
# 202 Accepted -> track with GET /v1/operaciones/{idOperacion}
```

## Endpoints

Issuing and queries:

| Method | Route | Purpose |
|---|---|---|
| `POST` | `/v1/contribuyentes/{identificacion}/comprobantes` | Issue any document type |
| `POST` | `/v1/contribuyentes/{identificacion}/comprobantes/validar` | Validate a document with the issuing rules, without issuing it |
| `GET` | `/v1/contexto` | Enabled taxpayers (first call) |
| `GET` | `/v1/operaciones/{idOperacion}` | Status of an operation |
| `GET` | `/v1/contribuyentes/{identificacion}/comprobantes/emitidos` | Issued documents (paginated) |
| `GET` | `/v1/consumo` | Available quota |
| `GET` | `/v1/catalogos/{nombre}` | SRI catalog |
| `GET` | `/v1/contribuyentes/{identificacion}/comprobantes/{claveAcceso}/ride` | RIDE (PDF) |
| `GET` | `/v1/contribuyentes/{identificacion}/comprobantes/{claveAcceso}/xml` | XML |
| `POST` | `/v1/contribuyentes/{identificacion}/comprobantes/{claveAcceso}/correo` | Resend PDF and XML by email (up to 20 recipients) |
| `GET` | `/v1/contribuyentes/{identificacion}/perfil` | Issuer profile |
| `PUT` | `/v1/contribuyentes/{identificacion}/perfil` | Update the profile |
| `POST` | `/v1/contribuyentes/{identificacion}/perfil/logo` | RIDE logo (multipart, PNG/JPEG up to 512 KB) |

SRI registry lookups:

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/v1/consultas/identificacion/validar/{numero}` | Validate the format of a cedula, RUC or passport |
| `GET` | `/v1/consultas/contribuyentes/{identificacion}` | Taxpayer data |
| `GET` | `/v1/consultas/contribuyentes?apellidos=...&nombres=...` | Search taxpayers by name |
| `GET` | `/v1/consultas/establecimientos/{ruc}?filtro=...` | Establishments of a RUC |
| `GET` | `/v1/consultas/comprobantes/decodificar/{claveAcceso}` | Decode an access key |

## Webhooks in brief

Each delivery carries `Ecuafact-Signature: t=<epoch>,v1=<hmac_sha256_hex>` and the Standard Webhooks headers `webhook-id`, `webhook-timestamp` and `webhook-signature`; verifying either one is enough. Discard duplicates by `eventType` + `resourceId`, or by `webhook-id`. Failed deliveries are retried after 1 min, 5 min, 30 min, 2 h, 8 h, 24 h and 36 h, so an event can arrive about 71 hours after it happened: if you reject old events by `occurredAtUtc`, allow at least 72 hours. Details and verification code in `reference/webhooks.md`.

## Tools: SDKs, CLI, Postman and MCP

All packages are pre-release, so install them with the pre-release flag or tag:

| Language | Install |
|---|---|
| .NET | `dotnet add package Ecuafact.Sdk --prerelease` |
| Node.js | `npm install @ecuafact/sdk@beta` |
| Python | `pip install "ecuafact>=1.0.0b5"` |
| Java | `com.ecuafact:sdk:1.0.0-beta4` |
| PHP | `composer require "ecuafact/sdk:^1.0@beta"` |

- **CLI** `ecuafact` (`npm install -g @ecuafact/cli@preview`, Scoop or Homebrew): issues one or many documents from JSON, JSONL, CSV or Excel, validates without using quota, tracks operations, downloads RIDE/XML, watches folders and tests webhooks. Prefer the CLI when the user works from the terminal, scripts or CI, or wants to issue in bulk from CSV or Excel without writing code (`ecuafact plantilla nueva factura`, `plantilla probar`, then `emitir --plantilla`; recipe in `reference/herramientas.md`). If the companion skill `ecuafact-cli` is installed, use it; otherwise `ecuafact schema` describes every command in JSON.
- **Postman collection** with folders `00` (start and diagnostics) to `07` (issuer profile); folders `01`, `06` and `07` have real effects.
- **Two MCP servers.** The account server acts on the user's behalf (it issues documents and queries their account, with OAuth). The public documentation server only reads the documentation and cannot issue anything.

Details for each tool in `reference/herramientas.md`.

## Account MCP server and portal confirmation

- Package of this skill: `https://docsapi.ecuafact.com/downloads/v1/skills/ecuafact.zip`.
- MCP server (Streamable HTTP, OAuth 2.1 with `Authorization: Bearer`; the user approves permissions and RUCs in the Customer Portal, bots use `client_credentials`; the API Key does not work on MCP): test `https://mcp-staging.mynexusapi.com/mcp`, production `https://mcp.mynexusapi.com/mcp`. Tools: `obtener_contexto`, `obtener_consumo`, `consultar_operacion`, `listar_emitidos`, `descargar_archivo`, `leer_catalogo`, `validar_identificacion`, `consultar_contribuyente`, `buscar_contribuyentes`, `listar_establecimientos`, `decodificar_clave_acceso`, `validar_comprobante` (validates without issuing; same permission as issuing), `emitir_comprobante` (requires `confirmar=true` and `idempotencyKey`) and `reenviar_correo` (requires `confirmar=true`, up to 5 recipients), plus the six documentation tools of the public server. Only the tools for the permissions granted to the agent appear. To connect Claude or Claude Desktop: **Configuración > Conectores > Agregar conector personalizado**, enter the server URL, press **Conectar** and authorize permissions and RUCs in the Customer Portal (steps for ChatGPT and Claude Code in the guide). See `https://docsapi.ecuafact.com/v1/guias/mcp.md`.
- Portal confirmation: the checkbox **Pedir mi confirmación para emitir y enviar correos** is checked by default when the user grants issuing or email permissions. While it is on, `emitir_comprobante` and `reenviar_correo` first answer `Pendiente de aprobacion: ...` with a link. **Nothing was executed**: show the link to the user, wait until they say they approved, and repeat the call with the same data (same `idempotencyKey` and `estructura`, or the same `claveAcceso` and recipients). If you change any value, a new approval is needed. If the answer says the user rejected it, do not retry without asking them what to change.

## Canonical documentation

Each guide lives at `https://docsapi.ecuafact.com/v1/guias/<slug>`; add `.md` for Markdown.

- Getting started: [/v1/guias/introduccion](https://docsapi.ecuafact.com/v1/guias/introduccion), [/v1/guias/primeros-pasos](https://docsapi.ecuafact.com/v1/guias/primeros-pasos), [/v1/guias/sandbox](https://docsapi.ecuafact.com/v1/guias/sandbox), [/v1/guias/datos-de-prueba](https://docsapi.ecuafact.com/v1/guias/datos-de-prueba), [/v1/guias/ciclo-de-vida](https://docsapi.ecuafact.com/v1/guias/ciclo-de-vida), [/v1/guias/autenticacion](https://docsapi.ecuafact.com/v1/guias/autenticacion), [/v1/guias/limite-de-request](https://docsapi.ecuafact.com/v1/guias/limite-de-request), [/v1/guias/ambientes](https://docsapi.ecuafact.com/v1/guias/ambientes), [/v1/guias/buenas-practicas](https://docsapi.ecuafact.com/v1/guias/buenas-practicas)
- Issuing: [/v1/guias/emision](https://docsapi.ecuafact.com/v1/guias/emision), [/v1/guias/emision-01](https://docsapi.ecuafact.com/v1/guias/emision-01), [/v1/guias/emision-03](https://docsapi.ecuafact.com/v1/guias/emision-03), [/v1/guias/emision-04](https://docsapi.ecuafact.com/v1/guias/emision-04), [/v1/guias/emision-05](https://docsapi.ecuafact.com/v1/guias/emision-05), [/v1/guias/emision-06](https://docsapi.ecuafact.com/v1/guias/emision-06), [/v1/guias/emision-07](https://docsapi.ecuafact.com/v1/guias/emision-07), [/v1/guias/consulta-ride](https://docsapi.ecuafact.com/v1/guias/consulta-ride), [/v1/guias/consulta-xml](https://docsapi.ecuafact.com/v1/guias/consulta-xml), [/v1/guias/reenviar-correo](https://docsapi.ecuafact.com/v1/guias/reenviar-correo), [/v1/guias/perfil-emisor](https://docsapi.ecuafact.com/v1/guias/perfil-emisor)
- Queries: [/v1/guias/consulta-contexto](https://docsapi.ecuafact.com/v1/guias/consulta-contexto), [/v1/guias/consulta-emitidos](https://docsapi.ecuafact.com/v1/guias/consulta-emitidos), [/v1/guias/consulta-catalogos](https://docsapi.ecuafact.com/v1/guias/consulta-catalogos), [/v1/guias/consulta-operacion](https://docsapi.ecuafact.com/v1/guias/consulta-operacion), [/v1/guias/consumo](https://docsapi.ecuafact.com/v1/guias/consumo)
- SRI lookups: [/v1/guias/consulta-contribuyente](https://docsapi.ecuafact.com/v1/guias/consulta-contribuyente), [/v1/guias/consulta-contribuyente-nombre](https://docsapi.ecuafact.com/v1/guias/consulta-contribuyente-nombre), [/v1/guias/consulta-establecimientos](https://docsapi.ecuafact.com/v1/guias/consulta-establecimientos), [/v1/guias/consulta-identificacion](https://docsapi.ecuafact.com/v1/guias/consulta-identificacion), [/v1/guias/consulta-clave-acceso](https://docsapi.ecuafact.com/v1/guias/consulta-clave-acceso)
- Webhooks: [/v1/guias/webhooks](https://docsapi.ecuafact.com/v1/guias/webhooks), [/v1/guias/webhooks-firma](https://docsapi.ecuafact.com/v1/guias/webhooks-firma)
- SDKs: [/v1/guias/sdk](https://docsapi.ecuafact.com/v1/guias/sdk), [/v1/guias/sdk-dotnet](https://docsapi.ecuafact.com/v1/guias/sdk-dotnet), [/v1/guias/sdk-node](https://docsapi.ecuafact.com/v1/guias/sdk-node), [/v1/guias/sdk-python](https://docsapi.ecuafact.com/v1/guias/sdk-python), [/v1/guias/sdk-java](https://docsapi.ecuafact.com/v1/guias/sdk-java), [/v1/guias/sdk-php](https://docsapi.ecuafact.com/v1/guias/sdk-php)
- CLI: [/v1/guias/cli](https://docsapi.ecuafact.com/v1/guias/cli), [/v1/guias/cli-emision](https://docsapi.ecuafact.com/v1/guias/cli-emision), [/v1/guias/cli-csv-excel](https://docsapi.ecuafact.com/v1/guias/cli-csv-excel), [/v1/guias/cli-vigilar](https://docsapi.ecuafact.com/v1/guias/cli-vigilar), [/v1/guias/cli-referencia](https://docsapi.ecuafact.com/v1/guias/cli-referencia)
- Assistants: [/v1/guias/mcp](https://docsapi.ecuafact.com/v1/guias/mcp), [/v1/guias/skill](https://docsapi.ecuafact.com/v1/guias/skill), [/v1/guias/agentes](https://docsapi.ecuafact.com/v1/guias/agentes)
- Resources: [/v1/guias/colecciones](https://docsapi.ecuafact.com/v1/guias/colecciones), [/v1/guias/changelog](https://docsapi.ecuafact.com/v1/guias/changelog)
- Reference: [/v1/guias/respuestas](https://docsapi.ecuafact.com/v1/guias/respuestas), [/v1/guias/errores](https://docsapi.ecuafact.com/v1/guias/errores), [/v1/guias/catalogos](https://docsapi.ecuafact.com/v1/guias/catalogos), [/v1/guias/limites](https://docsapi.ecuafact.com/v1/guias/limites), [/v1/guias/glosario](https://docsapi.ecuafact.com/v1/guias/glosario), [/v1/guias/faq](https://docsapi.ecuafact.com/v1/guias/faq), [/v1/guias/versionado](https://docsapi.ecuafact.com/v1/guias/versionado)
