# Queries

Every endpoint here requires `X-Api-Key` and counts toward the per-minute request limit; none of them consumes document quota (`cupo`). Paths with `{identificacion}` take the `identificacion` value (10 digits for a RUC) returned by `GET /v1/contexto`. Putting the 13-digit RUC in the path returns `403` (`codigo` `103`).

## Context

```
GET /v1/contexto
```

Call this first. It returns your integration and the taxpayers (`contribuyentes`) your API Key can operate in the environment of the URL you call. Use `contribuyentes[].identificacion` in paths and `identificacionCompleta` (13-digit RUC) in `infoTributaria.ruc`. Response `200` inside `datos`: `idCliente` (UUID; include it in support requests), `nombre`, and `contribuyentes[]` with `identificacion`, `identificacionCompleta`, `puedeEmitir`, `puedeRecibir` and `tiposComprobante` (enabled `codDoc` values). `contribuyentes` is `[]` when nothing is enabled in that environment. A newly enabled taxpayer or document type can take up to a minute to appear. Guide: `https://docsapi.ecuafact.com/v1/guias/consulta-contexto.md`.

## Track an operation

```
GET /v1/operaciones/{idOperacion}
```

`idOperacion` is the UUID returned by the issuance (also in `urlEstado`, the `Location` header and the webhook). If it does not exist, belongs to another integration or was issued in the other environment: `404` (`codigo` `502`). If it is not a UUID: `404` with no body. Response `200` inside `datos`:

```json
{
  "datos": {
    "idOperacion": "3f1c8c1e-6f2a-4b7e-9c3d-2a1b0c9d8e7f",
    "uid": "0123456789001-01-1-001-001-000000123",
    "ambiente": 1,
    "codDoc": "01",
    "estado": "enviado",
    "claveAcceso": "0710202601012345678900110010010000001231234567811",
    "codigoError": null,
    "fechaCreacion": "2026-10-07T15:04:05Z",
    "fechaActualizacion": "2026-10-07T15:04:20Z",
    "estadoAutorizacion": "autorizado",
    "fechaAutorizacion": "2026-10-07T15:04:18+00:00",
    "fechaConsulta": "2026-10-07T15:04:21Z",
    "codigoErrorConsulta": null
  }
}
```

- `ambiente`: `1` test, `2` production.
- `uid`: `{ruc}-{codDoc}-{ambiente}-{estab}-{ptoEmi}-{secuencial}` (not a UUID); while the operation awaits registration it can be `contingencia-{idOperacion}`.
- `claveAcceso`: `null` until assigned. Store it: you need it for the RIDE, the XML and the email.
- `codigoError` and `codigoErrorConsulta`: only `null` or `error_servicio`. The API does not return the detailed reason for an SRI rejection; check the document against the rules of its type and, if you cannot find the cause, contact support with the `idOperacion`.
- `fechaAutorizacion` can be `null` even when `estadoAutorizacion` is `autorizado`: never use it to decide.
- `estado` and `estadoAutorizacion`: values in `reference/codigos-sri.md`. Final when `estadoAutorizacion` is `autorizado` or `error`, or `estado` is `rechazado` or `cancelado`; any other combination means keep waiting.

The webhook is the primary mechanism; polling is the fallback. If you poll, wait between attempts (polling more often than every 5 seconds does not speed up the SRI result), increase the delay and set a cap. Reaching the cap without a final state is not an error: keep the document as pending and check again later.

## Issued documents (paginated)

```
GET /v1/contribuyentes/{identificacion}/comprobantes/emitidos
```

| Parameter | Rule |
|---|---|
| `desde`, `hasta` | `yyyy-MM-dd`, applied to the issue date. `desde` not before year 2000 and not after `hasta`; maximum range 366 days. No dates: last 30 days up to today (Ecuador time). Only `hasta`: the 30 days before it. Only `desde`: `hasta` is today, so a `desde` older than 366 days fails; send both dates for old periods |
| `codDoc` | Filter by type (`01`, `03`, `04`, `05`, `06`, `07`). A type not enabled for the taxpayer returns `403` (`103`) |
| `buscar` | Text matched against access key, sequential, counterparty identification or counterparty name; up to 100 characters, no control characters |
| `pagina` | 1 to 10000 (default 1) |
| `tamanoPagina` | 1 to 100 (default 20) |

Out of range: `400` (`codigo` `303`). Wrong type (for example `pagina=abc` or a date not in `yyyy-MM-dd`): `400` (`301`). Response in `datos`: `comprobantes[]` (`claveAcceso`, `numeroDocumento`, `codDoc`, `fechaEmision` in `dd/MM/yyyy`, `identificacionContraparte`, `razonSocialContraparte`, `total`, `estadoAutorizacion`, `fechaAutorizacion`, `codigoError`, `fechaExpress` (always `null`; ignore it), `idExpressDocument`), `pagina`, `tamanoPagina` and `hayMas` (if `true`, request `pagina + 1`). Rows are ordered newest first; only documents that already have an access key appear. When paging through a period, always fix `hasta` so new issuances do not shift pages. Use `tamanoPagina=100` to save requests. If `pagina` reaches 10000 with `hayMas` still `true`, split the date range.

## Usage and quota

```
GET /v1/consumo
```

**Flat** object (no `datos`): `limiteDocumentos`, `reservados`, `consumidos`, `disponibles` (totals for plan plus packages), `vigenteHasta` (`9999-12-31T00:00:00Z` means no plan, packages only), and the package breakdown `limiteExtra`, `reservadosExtra`, `consumidosExtra`, `disponiblesExtra`. `disponibles` = `limiteDocumentos - reservados - consumidos`, and is 0 if the plan is not current. Quota is reserved when an issuance is accepted (`202` with `codigo` `200`) and consumed when it resolves (authorized or rejected by the SRI, or `estado` `rechazado` before reaching the SRI); only a `cancelado` operation releases it. Idempotent retries (`201`) do not reserve again. Each environment has its own quota. No plan in this environment: `404` (`codigo` `502`). Out of quota, the issuance (not this query) returns `429` (`codigo` `501`); an expired plan returns `403` (`103`) even with package balance left. If the plan allows overage, `disponibles` can go negative.

## Catalogs

```
GET /v1/catalogos/{nombre}
```

`nombre` (case-insensitive): `tipos-identificacion`, `formas-pago`, `tipos-comprobante`, `tipos-impuesto`, `tarifas-iva`, `tarifas-ice`. Any other value: `404` (`codigo` `502`). No taxpayer enabled for your API Key in this environment: `403` (`103`). Response in `datos`: rows with `codigo` (string; keeps leading zeros), `nombre` and `tarifa` (number in IVA and ICE; `null` elsewhere). For IVA, `codigo` goes in `codigoPorcentaje` and `tarifa` is the percentage. Catalogs change rarely: read them at startup or once a day and cache them.

## RIDE (PDF) and XML

```
GET /v1/contribuyentes/{identificacion}/comprobantes/{claveAcceso}/ride
GET /v1/contribuyentes/{identificacion}/comprobantes/{claveAcceso}/xml
```

- Use the real 49-digit `claveAcceso` from tracking, the webhook or the issued-documents list. A key without 49 digits, with a wrong check digit or all zeros: `400` (`codigo` `301`).
- Both are available once the document has been delivered for fiscal processing and has an access key. Before that (`en_cola`, `enviando` or no `claveAcceso`): `404` (`codigo` `502`); retry later.
- RIDE: `application/pdf` (printed representation), binary body; save it as a file (suggested name `{claveAcceso}.pdf`). The PDF can be served before authorization, but only the RIDE of an `autorizado` document has fiscal validity. Download it after `estadoAutorizacion` is `autorizado`.
- XML: `application/xml`, `Content-Disposition: attachment` with file name `{claveAcceso}.xml`. Save it byte for byte; reformatting or re-encoding invalidates the electronic signature. If the final XML is not ready yet, the API returns `200` with a provisional XML and the header `X-Contenido: provisional`; that body is not the document and has no fiscal validity: detect it by the header (not by the content) and download again later. The final XML comes without `X-Contenido`.
- Non-200 responses carry a JSON error body; check the status before saving the body as a file.

## Resend email

```
POST /v1/contribuyentes/{identificacion}/comprobantes/{claveAcceso}/correo
Content-Type: application/json
```

```json
{ "destinatario": "cliente@ejemplo.com,contador@ejemplo.com" }
```

1 to 20 distinct comma-separated recipients; attaches the PDF and XML. Spaces around addresses and empty entries are ignored; duplicates are removed case-insensitively. Each address: at most 200 characters, no inner spaces, exactly one `@`, at least 1 character before it, at least 3 after it and a dot in the domain. If any address is invalid, no email is sent. Only works when the document is **authorized**; otherwise (pending or rejected) `400` (`codigo` `302`), and a rejected document can never be resent. Invalid or empty recipient, more than 20 recipients, invalid access key, malformed JSON or an unknown field: `400` (`codigo` `301`). Missing `Content-Type: application/json`: `415` with no body. Mail service failure: `502` (`003`). Response `200` in `datos`: `claveAcceso` and `destinatario` (normalized list). `200` means the send was accepted, not delivered. Every call sends the email again: do not retry a call that already returned `200`; retry only on network errors without a response or on `502`/`500`. Does not consume quota.

## Issuer profile

```
GET  /v1/contribuyentes/{identificacion}/perfil
PUT  /v1/contribuyentes/{identificacion}/perfil
POST /v1/contribuyentes/{identificacion}/perfil/logo
```

- All three return the same shape in `datos` (after any change): `identificacion` (the full 13-digit RUC), `razonSocial`, `nombreComercial`, `direccionMatriz`, `direccionEstablecimiento`, `codigoEstablecimiento`, `codigoPuntoEmision`, `moneda`, `obligadoContabilidad`, `correo`, `telefono`, `ciudad`, `provincia`, `pais`, `logo` (a reference, not a downloadable URL; `null` if none).
- `PUT` (`application/json`) changes only the fields sent: `nombreComercial`, `direccionMatriz`, `correo`, `telefono`, `ciudad`, `provincia`. An omitted or `null` field keeps its value; an empty string `""` is a new value. Any other field (for example `razonSocial` or `logo`) returns `400` (`codigo` `301`) and nothing changes.
- `POST .../logo` uses `multipart/form-data` with a single file field `logo`; the part's content type must be exactly `image/png` or `image/jpeg` (the API checks that, not the file extension; `application/octet-stream` fails), size greater than 0 and up to 512 KB. Invalid file: `400` (`codigo` `301`). The new logo replaces the previous one.
- Whole body over 1 MiB: `413` (`305`). Wrong `Content-Type` (not `application/json` on `PUT`, not `multipart/form-data` on the logo): `415` with no body. Profile not found: `404` (`502`); contact support. On `504` (`004`) or `500` (`001`) read the profile to check whether the change was applied before retrying.
