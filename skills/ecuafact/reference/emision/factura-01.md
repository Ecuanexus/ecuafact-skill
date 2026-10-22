# Invoice / Factura (`codDoc` = `01`)

Sale of goods or services to a buyer. Endpoint: `POST /v1/contribuyentes/{identificacion}/comprobantes` with `Content-Type: application/json`, `X-Api-Key` and `Idempotency-Key`. `{identificacion}` is `contribuyentes[].identificacion` from `GET /v1/contexto` (for a RUC, its first 10 digits; test data: `0123456789`), never the 13-digit RUC. Response: `202 Accepted` with `idOperacion`. A 202 means the document was received (admitted), not authorized: follow the operation until `estadoAutorizacion` is `autorizado` or `error`.

## Structure

| Key | Type | Required | Description |
|---|---|---|---|
| `origenReferencia` | string | Yes | Name of your system, 1 to 64 characters (ASCII letters and digits plus `-` `_` `.` `:` `/`) |
| `referenciaExterna` | string | Yes | Your internal document number, 1 to 128 characters, same alphabet |
| `infoTributaria` | object | Yes | See below |
| `info` | object | Yes | Header (see below) |
| `detalles` | array | Yes | 1 to 1000 lines |
| `infoAdicional` | array | No | Up to 20 `{nombre, valor}` objects, each value 1 to 300 characters |

`infoTributaria`:

| Field | Required | Rule |
|---|---|---|
| `ruc` | Yes | Issuer's full 13-digit RUC: the `identificacionCompleta` from `GET /v1/contexto` (test data: `0123456789001`). Its first 10 digits must equal the route `{identificacion}`. If they differ -> `422` (`codigo` `302`, `$.infoTributaria.ruc: El RUC no corresponde al contribuyente del contexto.`). Same 10-digit prefix but different last 3 digits (e.g. `0123456789002`) -> `403` (`codigo` `103`) |
| `codDoc` | Yes | `"01"` |
| `estab` | Yes | Establishment, digits only, up to 3, left-padded with zeros (`"2"` is read as `"002"`). Send `"002"` |
| `ptoEmi` | Yes | Emission point, digits only, up to 3, left-padded with zeros (`"001"`) |
| `secuencial` | No | Digits only, up to 9 (1 to 999999999). Omit it, or send `""` or all zeros (`"000000000"`), and the platform assigns the next number in the series. A number already used by a document that is in process, authorized or rejected by the SRI returns `409` (`codigo` `401`); only the number of an operation that ended in `document.failed` can be reused |

Do not send `claveAcceso`, `ambiente`, `tipoEmision`, `version` or `formato`: the platform assigns the access key, the URL defines the environment, and any unknown or repeated field returns `422` (`codigo` `302`).

## `info` fields

| Field | Type | Required | Rule |
|---|---|---|---|
| `fechaEmision` | string | Yes | `dd/MM/yyyy`. Send today's date: the authorized document carries the date it is processed, even if you send another |
| `dirEstablecimiento` | string | No | Up to 500 characters |
| `tipoIdentificacionComprador` | string | Yes | `04` RUC, `05` cedula (national ID), `06` pasaporte (passport), `07` consumidor final (final consumer; only with `importeTotal` <= 50.00), `08` identificacion del exterior (foreign ID) |
| `identificacionComprador` | string | Yes | With `04`: 13 digits; `05`: 10 digits; `07`: exactly `9999999999999`; `06`/`08`: up to 20 characters |
| `razonSocialComprador` | string | Yes | Up to 300 characters. With `07`: `CONSUMIDOR FINAL` |
| `direccionComprador` | string | No | Up to 300 characters |
| `telefono` | string | No | Up to 40 characters |
| `correo` | string | No | Buyer's email, up to 500 characters. If omitted, an `infoAdicional` entry named `email` or `correo` is used |
| `guiaRemision` | string | No | Number of the related delivery guide, `ddd-ddd-ddddddddd` |
| `placa` | string | No | Vehicle plate, up to 20 characters |
| `totalSinImpuestos` | number | Yes | Sum of `detalles[].precioTotalSinImpuesto` (after discounts) |
| `totalDescuento` | number | Yes | Sum of `detalles[].descuento`; send `0` if none |
| `totalConImpuestos` | array | Yes | 1 or 2 rows, exactly one per `codigo` + `codigoPorcentaje` pair present in the lines: `codigo`, `codigoPorcentaje`, `baseImponible` and `valor` summed from the lines (do not recompute `valor` on the summed base); `tarifa` optional (if sent, must equal the lines' `tarifa`) |
| `propina` | number | Yes | Tip; send `0` if none. Added to the total, no VAT |
| `importeTotal` | number | Yes | `totalSinImpuestos` + IVA + ICE + `propina` |
| `moneda` | string | Yes | `DOLAR` or `USD` |
| `pagos` | array | Yes | 1 to 20; the sum of `total` equals `importeTotal`. Each payment: `formaPago` (2-digit code string, e.g. `"01"` cash, `"19"` credit card, `"20"` bank transfer) and `total` required; `plazo` (integer >= 0) and `unidadTiempo` optional. `unidadTiempo` is required when `plazo` > 0 (free text up to 20 characters; usual values `dias`, `meses`, `anios`); if omitted with `plazo` 0, `dias` is assumed |

Export fields (`comercioExterior`, `incoTermFactura`, `lugarIncoTerm`, `paisOrigen`, `puertoEmbarque`, `puertoDestino`, `paisDestino`, `paisAdquisicion`, `incoTermTotalSinImpuestos`) are accepted but never reach the authorized document: do not use them. Fields such as `contribuyenteEspecial`, `obligadoContabilidad`, `totalSubsidio`, `retenciones`, `reembolsos` or fields of other document types (`...Proveedor`, `motivo`) return `422` (`Campo incompatible con codDoc.`).

## `detalles`

| Field | Type | Required | Description |
|---|---|---|---|
| `codigoPrincipal` | string | Yes | Product or service code, up to 25 characters |
| `codigoAuxiliar` | string | No | Up to 25 characters |
| `descripcion` | string | Yes | Up to 300 characters |
| `cantidad` | number | Yes | Greater than 0, up to 6 decimals |
| `precioUnitario` | number | Yes | Greater than or equal to 0, excluding VAT, up to 6 decimals |
| `descuento` | number | Yes | Line discount amount in dollars (not a percentage), up to 2 decimals; send `0` if none |
| `precioTotalSinImpuesto` | number | Yes | `cantidad x precioUnitario - descuento`, rounded to 2 decimals |
| `impuestos` | array | Yes | Exactly one IVA (`codigo` `"2"`) and at most one percentage ICE (`codigo` `"3"`), each with `codigo`, `codigoPorcentaje`, `tarifa`, `baseImponible`, `valor`. Other taxes such as IRBPNR (`"5"`) return `422` |
| `detallesAdicionales` | array | No | Up to 3 `{nombre, valor}` per line |

Do not use `codigoInterno` or `codigoAdicional` in an invoice (`422`, `Campo no permitido para este tipo de comprobante.`). `unidadMedida` and `precioSinSubsidio` are accepted but not transmitted.

Per-line calculation: `valor` = base x `tarifa` / 100, computed on the unrounded base (`cantidad x precioUnitario - descuento`) and rounded to 2 decimals (half up). The IVA `baseImponible` is `precioTotalSinImpuesto` plus the line's ICE. `tarifa` is a percentage (`15`, not `0.15`); with IVA `codigoPorcentaje` `0`, `6` or `7` it must be `0`, with other IVA rates greater than 0. Send the exact catalog pair (`"4"` with `15`): the API does not check that `codigoPorcentaje` and `tarifa` match, and the SRI may reject a mismatched pair later.

Format rules: amounts are JSON numbers with at most 2 decimals (`52.44`, never `"52,44"`); codes and identifications are JSON strings (`"01"`, `"0123456789"`); omitted required numbers are not treated as zero, so send `0` explicitly. The API never rounds or corrects values: any total off by one cent returns `422`, and each `422` reports only the first problem found.

## Example (IVA 15%)

Test environment: no fiscal validity. Uses the published test data (issuer `0123456789001`, route `identificacion` `0123456789`, buyer with cedula `0123456789`).

```json
{
  "origenReferencia": "MiERP",
  "referenciaExterna": "FACTURA-2026-0001",
  "infoTributaria": { "ruc": "0123456789001", "codDoc": "01", "estab": "002", "ptoEmi": "001" },
  "info": {
    "fechaEmision": "07/10/2026",
    "dirEstablecimiento": "AV. PRINCIPAL 123",
    "tipoIdentificacionComprador": "05",
    "identificacionComprador": "0123456789",
    "razonSocialComprador": "CLIENTE DE PRUEBA",
    "totalSinImpuestos": 100.00,
    "totalDescuento": 0,
    "totalConImpuestos": [ { "codigo": "2", "codigoPorcentaje": "4", "baseImponible": 100.00, "valor": 15.00, "tarifa": 15 } ],
    "propina": 0,
    "importeTotal": 115.00,
    "moneda": "DOLAR",
    "pagos": [ { "formaPago": "01", "total": 115.00 } ]
  },
  "detalles": [
    {
      "codigoPrincipal": "SERV-001",
      "descripcion": "Servicio de ejemplo",
      "cantidad": 1,
      "precioUnitario": 100.00,
      "descuento": 0,
      "precioTotalSinImpuesto": 100.00,
      "impuestos": [ { "codigo": "2", "codigoPorcentaje": "4", "tarifa": 15, "baseImponible": 100.00, "valor": 15.00 } ]
    }
  ],
  "infoAdicional": [ { "nombre": "Orden", "valor": "OC-1001" } ]
}
```

Send it with `Idempotency-Key: FACTURA-2026-0001` and reuse the same key on every retry of this document. No `secuencial` is sent, so the next number of series 002-001 is assigned; read it from the last segment of `uid` in the response.

Downloadable example: `https://docsapi.ecuafact.com/downloads/v1/ejemplos/example-01.json`. Full guide: `https://docsapi.ecuafact.com/v1/guias/emision-01.md`.
