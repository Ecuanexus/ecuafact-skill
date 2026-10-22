# SRI registry queries

Every endpoint here requires `X-Api-Key`, **does not consume document quota** and counts toward the per-minute request limit. Responses are **flat** (no `datos` wrapper). They behave the same in test and production. The registry lookups (taxpayer by identification, by name, establishments) read the real public SRI registry (`catastro`), so the test RUC `0123456789001` returns `404` (`codigo` `502`) there; use a real RUC to see data. Guides: `https://docsapi.ecuafact.com/v1/guias/consulta-contribuyente.md` and the siblings (`consulta-contribuyente-nombre`, `consulta-establecimientos`, `consulta-identificacion`, `consulta-clave-acceso`).

Common errors: `400` (`codigo` `303`) for an invalid filter or out-of-range value, `400` (`301`) for a parameter of the wrong type or a format the source rejects, `404` (`502`) when the SRI has no data, `502`/`504` (`003`/`004`) when the source fails or times out (retry with backoff, for example 5, 15 and 60 seconds), `503` (`002`) when the query service is temporarily down (honor `Retry-After`), `500` (`001`) on an internal error, and `429` (`104`) when your limit is exceeded or the source throttles requests (honor `Retry-After`).

## Validate an identification number

```
GET /v1/consultas/identificacion/validar/{numero}
```

Checks the format offline, without calling the SRI: national ID (`cedula`) is 10 digits with a valid modulo 10 check digit; a natural-person RUC is a valid cedula followed by an establishment code such as `001`; a company RUC (private or public) must have the company RUC structure (modulo 11 check digit not required); a passport is alphanumeric, 5 to 20 characters. It does **not** confirm the person or company exists (use the taxpayer lookup for that). It returns `200` even when the number is invalid: always check `valido`, never the HTTP status. Characters that cannot be validated: `400` (`301`). Leading and trailing spaces are trimmed.

```json
{
  "entrada": "0123456789001",
  "normalizado": "0123456789001",
  "valido": true,
  "tipo": "RucPersonaNatural",
  "longitud": 13,
  "esSoloDigitos": true,
  "cedulaBaseDelRuc": "0123456789",
  "mensaje": "Formato de RUC de persona natural aceptado; confirme su registro en el SRI."
}
```

Illustrative values: this is the shape for a valid real natural-person RUC. The documentation test values `0123456789` and `0123456789001` fail modulo 10, so the real response for them has `valido` `false`. Store `normalizado`. `cedulaBaseDelRuc` is `null` for other types. Do not program against `mensaje`. `tipo` values, with the matching `tipoIdentificacionComprador` code in parentheses: `Desconocido` (treat as invalid), `Cedula` (`05`), `RucPersonaNatural` (`04`, includes `cedulaBaseDelRuc`), `RucSociedadPrivada` (`04`), `RucSociedadPublica` (`04`), `Pasaporte` (`06`).

## Taxpayer by identification

```
GET /v1/consultas/contribuyentes/{identificacion}
```

Registry data for a cedula (10 digits), RUC (13 digits) or passport: `razonSocial`, `nombreCompleto` (use it when `razonSocial` is `null`), `estado` (compare against `ACTIVO`; treat any other text as not active), `regimen`, `clase`, main economic activity, accounting/withholding-agent/special-taxpayer flags, activity dates, `representantesLegales`, `establecimientos` and `estadoTributario`. Any field can be `null`; tolerate it. Optional query parameters: `fusionar` (`true` by default merges all sources; `false` returns the first source that answers, faster but possibly with fewer fields; any value other than `true`/`false` returns `400` (`301`)) and `fuente` (`Todas` by default, or `SriCatastro`; case-insensitive; any other value returns `400` (`303`)). Not in the registry: `404` (`502`). On `504` (`004`) you can try `fusionar=false`.

## Taxpayers by name

```
GET /v1/consultas/contribuyentes?apellidos=...&nombres=...&clase=...&max=...
```

| Parameter | Rule |
|---|---|
| `apellidos` | Required. People: surnames (`PEREZ LOPEZ`). Companies: the **beginning** of the legal name (a fragment from the middle or end may not match). Missing or empty: `400` (`303`) |
| `nombres` | Optional; narrows the search for people |
| `clase` | `PersonaNatural` (default), `Sociedad` or `SociedadPublica`; case-insensitive. Other value: `400` (`303`). Use `Sociedad` for a private company |
| `max` | 1 to 30 (default 10). Out of range: `400` (`303`); not an integer: `400` (`301`) |

URL-encode the values (space as `%20`). Response: flat array of `{ identificacion, tipoIdentificacion, nombreCompleto, clase, estado, fuenteOrigen }`; any field can be `null`. No matches can come back as `200` with `[]` or as `404` (`502`), depending on the source: treat both as "no results". Several people can share a name, so let a human confirm the match instead of picking the buyer automatically. Do not call on every keystroke.

## Establishments of a RUC

```
GET /v1/consultas/establecimientos/{ruc}?filtro=...
```

`ruc`: 13 digits. `filtro` (case-insensitive): `Todos` (default), `SoloActivos`, `SoloCerrados`, `SoloMatriz`, `SoloSucursales`; other value: `400` (`303`). Flat response: `ruc`, `filtro` (canonical name), `cantidad` and `establecimientos[]` (`numero`, the 3-digit code used in `infoTributaria.estab`; `nombreComercial`; `tipo`; `direccionCompleta`; `estado` `ABIERTO`/`CERRADO`; `esMatriz`). To identify the head office use `esMatriz`, not `tipo`. RUC not in the registry or without establishments: `404` (`502`).

## Decode an access key

```
GET /v1/consultas/comprobantes/decodificar/{claveAcceso}
```

Checks the 49 digits and the modulo 11 check digit and returns the segments. It does **not** prove fiscal authorization or that the document exists at the SRI; to know the status of a document you issued, use operation tracking. Wrong length, other characters or wrong check digit: `400` (`codigo` `301`). On `200`, still confirm `valido` is `true` before using the parts.

```json
{
  "entrada": "0710202601012345678900110010010000001231234567811",
  "normalizada": "0710202601012345678900110010010000001231234567811",
  "valido": true,
  "longitud": 49,
  "fechaEmision": "2026-10-07",
  "tipo": "Factura",
  "rucEmisor": "0123456789001",
  "ambiente": "Pruebas",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": "000000123",
  "codigoNumerico": "12345678",
  "tipoEmision": "1",
  "tipoEmisionDescripcion": "Normal",
  "digitoVerificador": "1",
  "mensaje": "Clave de acceso decodificada correctamente."
}
```

`fechaEmision` comes as `yyyy-MM-dd`; convert it to `dd/MM/yyyy` before using it in a document body (for example to reference the modified document in a credit note). `ambiente` is `Pruebas` (digit 1, no fiscal validity) or `Produccion` (digit 2). Key layout: positions 1-8 date `ddMMyyyy`, 9-10 `codDoc`, 11-23 issuer RUC, 24 environment, 25-27 `estab`, 28-30 `ptoEmi`, 31-39 sequential, 40-47 numeric code, 48 emission type (`1` normal), 49 check digit.
