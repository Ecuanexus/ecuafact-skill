# Credit note / Nota de credito (`codDoc` = `04`)

Refunds, discounts or cancels, fully or partially, an already authorized invoice. It references the modified invoice. Amounts are positive (they are the credited value); negative amounts return `422` (`Numero fuera de rango.`). Lines use **`codigoInterno`** (not `codigoPrincipal`). The note has no payments.

Common structure (`origenReferencia`, `referenciaExterna`, `infoTributaria` with `codDoc` `"04"`, `info`, `detalles` with 1 to 1000 lines, `infoAdicional` up to 20) is the same as the invoice. `infoTributaria` rules: `ruc` is the issuer's full 13-digit RUC and its first 10 digits must equal the route `{identificacion}` (different prefix -> `422`/`302`; same prefix, different last 3 digits -> `403`/`103`); `estab` and `ptoEmi` up to 3 digits, left-padded with zeros (may differ from the invoice's); `secuencial` optional (omit it or send all zeros to auto-assign; credit notes have their own series, independent of invoices; a used number returns `409`/`401`). Do not send `claveAcceso`, `ambiente`, `tipoEmision`, `version` or `formato` (`422`, `codigo` `302`). A `202` means received, not authorized: record the credit in your system only when `estadoAutorizacion` is `autorizado`.

## `info` fields

| Field | Type | Required | Rule |
|---|---|---|---|
| `fechaEmision` | string | Yes | `dd/MM/yyyy`, today's date (the authorized document carries the processing date) |
| `dirEstablecimiento` | string | No | Up to 500 characters |
| `tipoIdentificacionComprador` | string | Yes | `04` RUC, `05` cedula (national ID), `06` pasaporte (passport), `08` identificacion del exterior (foreign ID). `07` (consumidor final) is not allowed |
| `identificacionComprador` | string | Yes | The buyer of the original invoice. With `04`: 13 digits; `05`: 10; `06`/`08`: up to 20 characters |
| `razonSocialComprador` | string | Yes | Up to 300 characters |
| `direccionComprador` | string | No | Up to 300 characters |
| `telefono` | string | No | Up to 40 characters |
| `correo` | string | No | Buyer's email, up to 500 characters. If omitted, an `infoAdicional` entry named `email` or `correo` is used |
| `codDocModificado` | string | Yes | Type of the modified document (usually `"01"`), up to 2 digits, left-padded with zeros |
| `numDocModificado` | string | Yes | `ddd-ddd-ddddddddd` (e.g. `"002-001-000000123"`; segments 4, 5 and 6 of the invoice's `uid`). Short segments are zero-padded (`"2-1-123"` -> `"002-001-000000123"`); 10 to 15 digits without hyphens are also accepted. Only the sequential (`"123"`) is rejected (`Indique establecimiento y punto de emision; no se infieren.`) |
| `fechaEmisionDocSustento` | string | Yes | Date of the original invoice, `dd/MM/yyyy` |
| `totalDocumentoSustento` | number | No (recommended) | `importeTotal` of the original invoice. If sent, `valorModificacion` cannot exceed it |
| `totalSinImpuestos` | number | Yes | Sum of `detalles[].precioTotalSinImpuesto` |
| `totalConImpuestos` | array | Yes | 1 or 2 rows, one per `codigo` + `codigoPorcentaje` pair, with `baseImponible` and `valor` summed from the lines; `tarifa` optional |
| `valorModificacion` | number | Yes | `totalSinImpuestos` + IVA + ICE: the credited total (replaces `importeTotal`) |
| `moneda` | string | Yes | `DOLAR` or `USD` |
| `motivo` | string | Yes | Reason for the note, up to 300 characters (e.g. `DEVOLUCION`). Line breaks become spaces |

Do not send `totalDescuento`, `propina`, `importeTotal` or `pagos` (`422`, `Campo incompatible con codDoc.`, even if they are `0`).

The API does not look up or validate the modified invoice: copy these fields from your own record. It also does not track the cumulative balance of several notes against one invoice, nor check that the invoice exists, is authorized, has the same buyer or an earlier date. Check those in your system before emitting.

## `detalles`

`codigoInterno` (required, up to 25; use the invoice's product code), `codigoAdicional?` (up to 25), `descripcion` (up to 300), `cantidad` (> 0, up to 6 decimals), `precioUnitario` (>= 0, excluding VAT, up to 6 decimals), `descuento` (amount, up to 2 decimals), `precioTotalSinImpuesto`, `impuestos` (exactly one IVA and at most one ICE; use the same rate the line had on the invoice), `detallesAdicionales?` (up to 3 per line). Do not send `codigoPrincipal` or `codigoAuxiliar` (`422`). Line and tax calculations are the same as the invoice. Amounts are JSON numbers with at most 2 decimals.

## Example

Test environment: no fiscal validity. Partial refund of 20.00 + IVA 15% on invoice 002-001-000000123 (total 115.00).

```json
{
  "origenReferencia": "MiERP",
  "referenciaExterna": "NC-2026-0003",
  "infoTributaria": { "ruc": "0123456789001", "codDoc": "04", "estab": "002", "ptoEmi": "001" },
  "info": {
    "fechaEmision": "07/10/2026",
    "tipoIdentificacionComprador": "05",
    "identificacionComprador": "0123456789",
    "razonSocialComprador": "CLIENTE DE PRUEBA",
    "codDocModificado": "01",
    "numDocModificado": "002-001-000000123",
    "fechaEmisionDocSustento": "01/10/2026",
    "totalDocumentoSustento": 115.00,
    "totalSinImpuestos": 20.00,
    "totalConImpuestos": [ { "codigo": "2", "codigoPorcentaje": "4", "baseImponible": 20.00, "valor": 3.00 } ],
    "valorModificacion": 23.00,
    "moneda": "DOLAR",
    "motivo": "DEVOLUCION PARCIAL"
  },
  "detalles": [
    {
      "codigoInterno": "SERV-001",
      "descripcion": "Devolucion de servicio",
      "cantidad": 1,
      "precioUnitario": 20.00,
      "descuento": 0,
      "precioTotalSinImpuesto": 20.00,
      "impuestos": [ { "codigo": "2", "codigoPorcentaje": "4", "tarifa": 15, "baseImponible": 20.00, "valor": 3.00 } ]
    }
  ]
}
```

Downloadable example: `https://docsapi.ecuafact.com/downloads/v1/ejemplos/example-04.json`. Full guide: `https://docsapi.ecuafact.com/v1/guias/emision-04.md`.
