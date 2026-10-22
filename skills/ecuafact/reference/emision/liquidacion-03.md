# Purchase settlement / Liquidacion de compra (`codDoc` = `03`)

Issued by the buyer (your taxpayer) when it **purchases** goods or services from a supplier that cannot issue an invoice (e.g. a person without RUC, or a foreign supplier without domicile in Ecuador). Lines, taxes, totals and payments work exactly as in the invoice (`reference/emision/factura-01.md`); what changes is the counterparty (supplier, `...Proveedor` fields) and the fields that do not apply.

Common structure (`origenReferencia`, `referenciaExterna`, `infoTributaria` with `codDoc` `"03"`, `info`, `detalles`, `infoAdicional` up to 20) is the same as the invoice. `infoTributaria` rules: `ruc` is your full 13-digit RUC (the `identificacionCompleta` from `GET /v1/contexto`) and its first 10 digits must equal the route `{identificacion}` (different prefix -> `422`/`302`; same prefix, different last 3 digits -> `403`/`103`); `estab` and `ptoEmi` up to 3 digits, left-padded with zeros; `secuencial` optional (omit it or send all zeros to auto-assign the next number of the settlement series; a used number returns `409`/`401`). Do not send `claveAcceso`, `ambiente`, `tipoEmision`, `version` or `formato` (`422`, `codigo` `302`). A `202` means received, not authorized.

## `info` fields

| Field | Type | Required | Rule |
|---|---|---|---|
| `fechaEmision` | string | Yes | `dd/MM/yyyy`, today's date (the authorized document carries the processing date) |
| `dirEstablecimiento` | string | No | Up to 500 characters |
| `tipoIdentificacionProveedor` | string | Yes | `04` RUC, `05` cedula (national ID), `06` pasaporte (passport), `08` identificacion del exterior (foreign ID). `07` (consumidor final) is not allowed |
| `identificacionProveedor` | string | Yes | With `04`: 13 digits; `05`: 10 digits; `06`/`08`: up to 20 characters |
| `razonSocialProveedor` | string | Yes | Up to 300 characters |
| `direccionProveedor` | string | No | Up to 300 characters |
| `telefono` | string | No | Up to 40 characters |
| `correo` | string | No | Supplier's email, up to 500 characters. If omitted, an `infoAdicional` entry named `email` or `correo` is used |
| `totalSinImpuestos` | number | Yes | Sum of `detalles[].precioTotalSinImpuesto` |
| `totalDescuento` | number | Yes | Sum of `detalles[].descuento`; send `0` if none |
| `totalConImpuestos` | array | Yes | 1 or 2 rows, one per `codigo` + `codigoPorcentaje` pair, with `baseImponible` and `valor` summed from the lines; `tarifa` optional (if sent, must equal the lines' `tarifa`) |
| `importeTotal` | number | Yes | `totalSinImpuestos` + IVA + ICE |
| `moneda` | string | Yes | `DOLAR` or `USD` |
| `pagos` | array | Yes | 1 to 20; sum of `total` = `importeTotal`. `formaPago` and `total` required; `plazo`/`unidadTiempo` optional (`unidadTiempo` required if `plazo` > 0) |

Do not send `propina` (not even `0`), `guiaRemision`, `placa` or any `...Comprador` field: they return `422` (`Campo incompatible con codDoc.`).

`detalles`: same as the invoice: `codigoPrincipal` (up to 25), `codigoAuxiliar?` (up to 25), `descripcion` (up to 300), `cantidad` (> 0, up to 6 decimals), `precioUnitario` (>= 0, excluding VAT, up to 6 decimals), `descuento` (amount, up to 2 decimals), `precioTotalSinImpuesto`, `impuestos` (exactly one IVA and at most one ICE), `detallesAdicionales?` (up to 3 per line). 1 to 1000 lines. `codigoInterno`/`codigoAdicional` return `422`; `unidadMedida` and `precioSinSubsidio` are accepted but not transmitted.

Amounts are JSON numbers with at most 2 decimals; codes are strings. If you must also withhold taxes on this purchase, issue a withholding receipt (`codDoc` `07`) with this settlement as support document (`codDocSustento` `03`).

## Example

Test environment: no fiscal validity.

```json
{
  "origenReferencia": "MiERP",
  "referenciaExterna": "LIQ-2026-0007",
  "infoTributaria": { "ruc": "0123456789001", "codDoc": "03", "estab": "002", "ptoEmi": "001" },
  "info": {
    "fechaEmision": "07/10/2026",
    "dirEstablecimiento": "AV. PRINCIPAL 123",
    "tipoIdentificacionProveedor": "05",
    "identificacionProveedor": "0123456789",
    "razonSocialProveedor": "PROVEEDOR DE PRUEBA",
    "direccionProveedor": "CALLE 1 NORTE",
    "totalSinImpuestos": 100.00,
    "totalDescuento": 0,
    "totalConImpuestos": [ { "codigo": "2", "codigoPorcentaje": "4", "baseImponible": 100.00, "valor": 15.00 } ],
    "importeTotal": 115.00,
    "moneda": "DOLAR",
    "pagos": [ { "formaPago": "01", "total": 115.00 } ]
  },
  "detalles": [
    {
      "codigoPrincipal": "COMPRA-001",
      "descripcion": "Compra de bienes",
      "cantidad": 1,
      "precioUnitario": 100.00,
      "descuento": 0,
      "precioTotalSinImpuesto": 100.00,
      "impuestos": [ { "codigo": "2", "codigoPorcentaje": "4", "tarifa": 15, "baseImponible": 100.00, "valor": 15.00 } ]
    }
  ]
}
```

Downloadable example: `https://docsapi.ecuafact.com/downloads/v1/ejemplos/example-03.json`. Full guide: `https://docsapi.ecuafact.com/v1/guias/emision-03.md`.
