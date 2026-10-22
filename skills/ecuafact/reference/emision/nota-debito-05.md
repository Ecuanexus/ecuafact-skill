# Debit note / Nota de debito (`codDoc` = `05`)

Charges interest, surcharges or upward adjustments on an already issued invoice. Uses the **`motivos`** collection (not `detalles`). Unlike the credit note, it uses header-level `impuestos` (not `totalConImpuestos`) and `valorTotal` (not `valorModificacion`).

Common structure (`origenReferencia`, `referenciaExterna`, `infoTributaria` with `codDoc` `"05"`, `info`, `motivos`, `infoAdicional` up to 20) is the same as the invoice. `infoTributaria` rules: `ruc` is the issuer's full 13-digit RUC and its first 10 digits must equal the route `{identificacion}` (different prefix -> `422`/`302`; same prefix, different last 3 digits -> `403`/`103`); `estab` and `ptoEmi` up to 3 digits, left-padded with zeros; `secuencial` optional (omit it or send all zeros to auto-assign the next number of the debit-note series; a used number returns `409`/`401`). Do not send `claveAcceso`, `ambiente`, `tipoEmision`, `version` or `formato` (`422`, `codigo` `302`). A `202` means received, not authorized.

## `info` fields

| Field | Type | Required | Rule |
|---|---|---|---|
| `fechaEmision` | string | Yes | `dd/MM/yyyy`, today's date (the authorized document carries the processing date) |
| `dirEstablecimiento` | string | No | Up to 500 characters |
| `tipoIdentificacionComprador` | string | Yes | `04` RUC, `05` cedula (national ID), `06` pasaporte (passport), `08` identificacion del exterior (foreign ID). `07` (consumidor final) is not allowed |
| `identificacionComprador` | string | Yes | With `04`: 13 digits; `05`: 10; other types: up to 20 characters |
| `razonSocialComprador` | string | Yes | Up to 300 characters |
| `direccionComprador` | string | No | Up to 300 characters |
| `telefono` | string | No | Up to 40 characters |
| `correo` | string | No | Buyer's email, up to 500 characters. If omitted, an `infoAdicional` entry named `email` or `correo` is used |
| `codDocModificado` | string | Yes | Type of the modified document, 2 digits (usually `"01"`) |
| `numDocModificado` | string | Yes | `ddd-ddd-ddddddddd` |
| `fechaEmisionDocSustento` | string | Yes | Date of the modified document, `dd/MM/yyyy` |
| `totalSinImpuestos` | number | Yes | Sum of `motivos[].valor` (equal to `impuestos[0].baseImponible`) |
| `impuestos` | array | Yes | Exactly one IVA element (`codigo` `"2"`), no ICE: `codigo`, `codigoPorcentaje`, `tarifa` (required; `0` with `codigoPorcentaje` `0`, `6` or `7`, otherwise > 0), `baseImponible`, `valor` (sum of each motivo's IVA, each rounded to 2 decimals half up, then summed) |
| `valorTotal` | number | Yes | `totalSinImpuestos` + IVA |
| `moneda` | string | No | `DOLAR` or `USD` |
| `pagos` | array | Yes | 1 to 20; sum of `total` = `valorTotal`. `formaPago` and `total` required; `plazo` (integer >= 0) and `unidadTiempo` optional (`unidadTiempo` required if `plazo` > 0) |

Do not send `detalles`, `totalConImpuestos`, `valorModificacion`, `totalDescuento`, `propina` or `motivo` (singular): they return `422`. The API does not look up the modified document: copy its data from your record.

## `motivos`

1 to 1000 items. All motivos share the single IVA rate declared in `info.impuestos`.

| Field | Type | Required | Description |
|---|---|---|---|
| `razon` | string | Yes | Description of the charge, up to 300 characters. The first `razon` is used as the document's reason |
| `valor` | number | Yes | Charge amount **excluding VAT**, >= 0, up to 2 decimals |

Amounts are JSON numbers with at most 2 decimals; the API does not round for you.

## Example

Test environment: no fiscal validity. Charge of 30.00 + IVA 15% (4.50) = 34.50.

```json
{
  "origenReferencia": "MiERP",
  "referenciaExterna": "ND-2026-0002",
  "infoTributaria": { "ruc": "0123456789001", "codDoc": "05", "estab": "002", "ptoEmi": "001" },
  "info": {
    "fechaEmision": "07/10/2026",
    "tipoIdentificacionComprador": "05",
    "identificacionComprador": "0123456789",
    "razonSocialComprador": "CLIENTE DE PRUEBA",
    "codDocModificado": "01",
    "numDocModificado": "002-001-000000123",
    "fechaEmisionDocSustento": "01/10/2026",
    "totalSinImpuestos": 30.00,
    "impuestos": [ { "codigo": "2", "codigoPorcentaje": "4", "tarifa": 15, "baseImponible": 30.00, "valor": 4.50 } ],
    "valorTotal": 34.50,
    "pagos": [ { "formaPago": "01", "total": 34.50 } ]
  },
  "motivos": [ { "razon": "INTERESES POR MORA", "valor": 30.00 } ]
}
```

Downloadable example: `https://docsapi.ecuafact.com/downloads/v1/ejemplos/example-05.json`. Full guide: `https://docsapi.ecuafact.com/v1/guias/emision-05.md`.
