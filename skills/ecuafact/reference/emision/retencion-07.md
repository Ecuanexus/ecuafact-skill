# Withholding receipt / Comprobante de retencion (`codDoc` = `07`)

Certifies to the supplier the income tax (renta), IVA or ISD you withheld when paying them. Uses `docsSustento` (exactly one support document) instead of `detalles`; to withhold on several invoices, issue one receipt per invoice.

Common structure (`origenReferencia`, `referenciaExterna`, `infoTributaria` with `codDoc` `"07"`, `info`, `docsSustento`, `infoAdicional` up to 20) is the same as the invoice. `infoTributaria` rules: `ruc` is the full 13-digit RUC of the issuer (the withholding agent) and its first 10 digits must equal the route `{identificacion}` (different prefix -> `422`/`302`; same prefix, different last 3 digits -> `403`/`103`); `estab` and `ptoEmi` up to 3 digits, left-padded with zeros; `secuencial` optional (omit it or send all zeros to auto-assign the next number of the withholding series; a used number returns `409`/`401`). Do not send `claveAcceso`, `ambiente`, `tipoEmision`, `version` or `formato` (`422`, `codigo` `302`). Do not send `detalles`, `motivos` or `destinatarios` (`422`). A `202` means received, not authorized.

## `info` fields

| Field | Type | Required | Rule |
|---|---|---|---|
| `fechaEmision` | string | Yes | `dd/MM/yyyy`, from 01/01/2000. The withholding is the one type whose authorized date is not overwritten with the processing date |
| `dirEstablecimiento` | string | No | Up to 500 characters |
| `tipoIdentificacionSujetoRetenido` | string | Yes | `04` RUC, `05` cedula (national ID), `06` pasaporte (passport), `08` identificacion del exterior (foreign ID). `07` (consumidor final) is not allowed |
| `identificacionSujetoRetenido` | string | Yes | With `04`: 13 digits; `05`: 10; other types: up to 20 characters |
| `razonSocialSujetoRetenido` | string | Yes | Up to 300 characters |
| `tipoSujetoRetenido` | string | No | Classification of the withheld party, when the SRI requires it (foreign ID) |
| `periodoFiscal` | string | Yes | Declaration month and year, `MM/yyyy` (e.g. `"10/2026"`) |
| `parteRel` | string | Yes | `SI` (related parties) or `NO` |
| `telefono` | string | No | Up to 40 characters |
| `correo` | string | No | Supplier's email, up to 500 characters. If omitted, an `infoAdicional` entry named `email` or `correo` is used |

`info` carries no `moneda`, `pagos` or totals (`422`): amounts go inside `docsSustento`. Reimbursement, dividend and foreign-payment fields (`tipoRegi`, `paisEfecPago`, `aplicConvDobTrib`, `pagExtSujRetNorLeg`, `pagoRegFis`, `reembolsos`, `dividendos`, `compraCajBanano`, ...) are accepted but not processed: do not send them.

## `docsSustento` (exactly one)

| Field | Type | Required | Description |
|---|---|---|---|
| `codSustento` | string | Yes | Support code, 2 digits (documented accepted values `01` to `07`); `10` (dividends) is rejected |
| `codDocSustento` | string | Yes | Support document type, 2 digits (documented accepted values `01` to `05`, e.g. `01` factura, `03` liquidacion de compra); `41` (reimbursement) is rejected |
| `numDocSustento` | string | Yes | Supplier document number: 15 digits (`002001000000123`) or `ddd-ddd-ddddddddd` |
| `fechaEmisionDocSustento` | string | Yes | `dd/MM/yyyy` |
| `fechaRegistroContable` | string | Yes | Date you booked the document, `dd/MM/yyyy` |
| `numAutDocSustento` | string | No | Access key of the electronic support document; if sent, exactly 49 digits |
| `pagoLocExt` | string | Yes | Only `01` (local payment). `02` returns `422` |
| `totalSinImpuestos` | number | Yes | Support document base without taxes |
| `importeTotal` | number | Yes | `totalSinImpuestos` + sum of `impuestosDocSustento[].valorImpuesto` |
| `impuestosDocSustento` | array | Yes | 1 or 2 items, as they appear on the supplier's document: `codImpuestoDocSustento` (`2` IVA, `3` ICE), `codigoPorcentaje`, `baseImponible`, `tarifa`, `valorImpuesto` |
| `retenciones` | array | Yes | 1 to 1000: `codigo` (`1` renta, `2` IVA, `6` ISD), `codigoRetencion` (up to 8 characters), `baseImponible`, `porcentajeRetener` (0 to 100), `valorRetenido` = round(base x percentage / 100, 2) |
| `pagos` | array | Yes | 1 to 20; sum of `total` = `importeTotal`. `formaPago` and `total` required; `plazo`/`unidadTiempo` optional (`unidadTiempo` required if `plazo` > 0) |

Bases: for renta, the base excluding VAT; for IVA, the IVA amount of the support document; for ISD (`codigoRetencion` `4580`, 5%), the amount paid. IVA withholding codes: `9` 10%, `10` 20%, `1` 30%, `11` 50%, `2` 70%, `3` 100%. More codes and percentages in `reference/codigos-sri.md`. Banana income codes `338`, `340`, `341`, `342`, `342A` and `342B` return `422`. The API checks `valorRetenido` = base x percentage / 100 but not that the percentage matches the code: use the current SRI percentage, or the SRI may reject the document. Amounts are JSON numbers with at most 2 decimals; the API does not round for you. SRI rules require issuing the withholding within 5 days of receiving the supplier's document; the API does not check this.

## Example (renta 303 professional fees, 10%)

Test environment: no fiscal validity.

```json
{
  "origenReferencia": "MiERP",
  "referenciaExterna": "RET-2026-0001",
  "infoTributaria": { "ruc": "0123456789001", "codDoc": "07", "estab": "002", "ptoEmi": "001" },
  "info": {
    "fechaEmision": "07/10/2026",
    "tipoIdentificacionSujetoRetenido": "05",
    "identificacionSujetoRetenido": "0123456789",
    "razonSocialSujetoRetenido": "PROVEEDOR DE PRUEBA",
    "periodoFiscal": "10/2026",
    "parteRel": "NO"
  },
  "docsSustento": [
    {
      "codSustento": "01",
      "codDocSustento": "01",
      "numDocSustento": "001001000000123",
      "fechaEmisionDocSustento": "07/10/2026",
      "fechaRegistroContable": "07/10/2026",
      "numAutDocSustento": "0710202601012345678900110010010000001231234567811",
      "pagoLocExt": "01",
      "totalSinImpuestos": 100.00,
      "importeTotal": 115.00,
      "impuestosDocSustento": [
        { "codImpuestoDocSustento": "2", "codigoPorcentaje": "4", "baseImponible": 100.00, "tarifa": 15, "valorImpuesto": 15.00 }
      ],
      "retenciones": [
        { "codigo": "1", "codigoRetencion": "303", "baseImponible": 100.00, "porcentajeRetener": 10, "valorRetenido": 10.00 }
      ],
      "pagos": [ { "formaPago": "01", "total": 115.00 } ]
    }
  ]
}
```

Downloadable example: `https://docsapi.ecuafact.com/downloads/v1/ejemplos/example-07.json`. Full guide: `https://docsapi.ecuafact.com/v1/guias/emision-07.md`.
