# SRI codes

Catalogs you need to build a document. Confirm the current values with `GET /v1/catalogos/{nombre}`
(`tipos-identificacion`, `formas-pago`, `tipos-comprobante`, `tipos-impuesto`, `tarifas-iva`, `tarifas-ice`); if a table here and the endpoint differ, the endpoint wins. Withholding and support codes, states and countries are not served by the endpoint: use the tables here. Full guide: `https://docsapi.ecuafact.com/v1/guias/catalogos.md`.

## Document types (`infoTributaria.codDoc`)

Also used in `info.codDocModificado` (credit and debit notes) for the type of the modified document. Which types your API Key can issue depends on your plan; see `tiposComprobante` in `GET /v1/contexto`.

| Code | Document |
|---|---|
| `01` | Factura (invoice) |
| `03` | Liquidacion de compra (purchase settlement) |
| `04` | Nota de credito (credit note) |
| `05` | Nota de debito (debit note) |
| `06` | Guia de remision (shipping guide) |
| `07` | Comprobante de retencion (withholding voucher) |

## Identification types

For `tipoIdentificacionComprador` (invoice and notes), `tipoIdentificacionProveedor` (purchase settlement), `tipoIdentificacionTransportista` (shipping guide) and `tipoIdentificacionSujetoRetenido` (withholding). Any other code returns `422`.

| Code | Type | Identification rule |
|---|---|---|
| `04` | RUC | 13 digits |
| `05` | Cedula (national ID) | 10 digits |
| `06` | Pasaporte (passport) | Up to 20 characters |
| `07` | Venta a consumidor final (final consumer) | `9999999999999`; invoice only and with `importeTotal` <= 50.00; `razonSocialComprador` `CONSUMIDOR FINAL` |
| `08` | Identificacion del exterior (foreign ID) | Up to 20 characters |

- Purchase settlement, credit and debit notes and withholding do not accept `07`: the counterparty must be identified. An invoice above 50.00 must use `04`, `05`, `06` or `08`.
- Shipping guide: the recipient has no type; `identificacionDestinatario` has 10 digits (cedula) or 13 (RUC).

## Taxes

Tax code (`impuestos[].codigo`, `totalConImpuestos[].codigo`, `impuestosDocSustento[].codImpuestoDocSustento`):

| Code | Tax | Supported |
|---|---|---|
| `2` | IVA (VAT) | Yes. Each line carries exactly one IVA |
| `3` | ICE (excise) | Yes, percentage-based only and at most one per line. The debit note does not accept ICE |
| `5` | IRBPNR | No: returns `422` |

IVA (`codigoPorcentaje`). `tarifa` is the percentage (`15`, not `0.15`). With `0`, `6` and `7` the `tarifa` must be `0`; with any other `codigoPorcentaje` it must be greater than 0 (otherwise `422`).

| codigoPorcentaje | Rate | `tarifa` | In `GET /v1/catalogos/tarifas-iva` |
|---|---|---|---|
| `0` | 0% | `0` | Yes |
| `4` | 15% (general rate) | `15` | Yes |
| `5` | 5% (reduced rate) | `5` | Yes |
| `6` | No objeto de impuesto (not subject to IVA) | `0` | Yes |
| `7` | Exento de IVA (exempt) | `0` | Yes |
| `8` | 8% (differentiated, for periods the SRI sets) | `8` | Yes |
| `2` | 12% (only for fiscal periods when it applied) | `12` | No |
| `3` | 14% (only for fiscal periods when it applied) | `14` | No |
| `10` | 13% (special SRI rate) | `13` | No |

To invoice, use the rates returned by `GET /v1/catalogos/tarifas-iva`; the other rows are reference only. The IVA `valor` is (`cantidad` x `precioUnitario` - `descuento`) x `tarifa` / 100, computed on the unrounded base and rounded to 2 decimals at the end, so `baseImponible` x `tarifa` / 100 can differ by one cent. A line's IVA base includes that line's ICE. In `totalConImpuestos` the `tarifa` is optional, but if sent it must match the lines with the same `codigoPorcentaje`.

ICE (`codigo` `3`): `codigoPorcentaje` is the SRI product code (up to 4 digits); get the current list with `GET /v1/catalogos/tarifas-ice`. Percentage ICE only (fixed per-unit ICE is not supported): `valor` = line base x `tarifa` / 100, and that value is added to the IVA base of the same line.

## Payment methods (`pagos[].formaPago`)

| Code | Payment method |
|---|---|
| `01` | Sin utilizacion del sistema financiero (cash) |
| `15` | Compensacion de deudas (debt offset) |
| `16` | Tarjeta de debito (debit card) |
| `17` | Dinero electronico (electronic money) |
| `18` | Tarjeta prepago (prepaid card) |
| `19` | Tarjeta de credito (credit card) |
| `20` | Otros con utilizacion del sistema financiero (transfer, check) |
| `21` | Endoso de titulos (endorsement of securities) |

Each document with payments carries 1 to 20 payment methods (invoice and purchase settlement), and the sum of `pagos[].total` must equal the document total. `plazo` is optional (integer >= 0). `unidadTiempo` is free text up to 20 characters (usual values: `dias`, `meses`, `anios`), required when `plazo` > 0; if you omit it with `plazo` 0, `dias` is used.

## Withholdings (`docsSustento[].retenciones`)

| `codigo` | Tax withheld | Withholding base |
|---|---|---|
| `1` | Impuesto a la renta (income tax) | Purchase base without IVA |
| `2` | IVA | IVA amount of the purchase |
| `6` | Impuesto a la salida de divisas (ISD, currency outflow tax) | Amount paid; `codigoRetencion` `4580` (5%) |

Any other `codigo` returns `422`. `codigoRetencion` accepts up to 8 characters (letters, digits and `-` `_` `.` `:` `/`). The API checks `valorRetenido` = round(`baseImponible` x `porcentajeRetener` / 100, 2), but not that the percentage matches the code: if it does not, the SRI may refuse to authorize the document.

IVA withheld (`codigo` = `2`):

| codigoRetencion | porcentajeRetener |
|---|---|
| `9` | 10 |
| `10` | 20 |
| `1` | 30 |
| `11` | 50 |
| `2` | 70 |
| `3` | 100 |

Income tax (`codigo` = `1`), frequently used rows with their reference percentage (the SRI updates the table by resolution; confirm the current value in the SRI table):

| codigoRetencion | Concept | Percentage |
|---|---|---|
| `303` | Professional fees | 10 |
| `307` | Services where labor predominates | 2 |
| `310` | Private passenger transport or cargo transport | 1 |
| `312` | Transfer of tangible movable goods | 1.75 |
| `320` | Lease of real estate | 10 |
| `322` | Insurance and reinsurance | 1.75 |
| `332` | Payments not subject to withholding | 0 |
| `343` | Other withholdings at 1% | 1 |
| `3440` | Other withholdings at 2.75% | 2.75 |

The banana income-tax codes `338`, `340`, `341`, `342`, `342A` and `342B` return `422`.

Support documents (`docsSustento[]`):

| Field | Rule |
|---|---|
| `codSustento` | `01` to `07` (IVA tax credit or income-tax cost/expense for general items, fixed assets, travel expenses or inventory). `10` (dividends) returns `422` |
| `codDocSustento` | `01` factura, `02` nota o boleta de venta, `03` liquidacion de compra, `04` nota de credito, `05` nota de debito. `41` (reimbursement) returns `422` |
| `numDocSustento` | 15 digits without dashes (`001001000000123`); `ddd-ddd-ddddddddd` is also accepted |
| `numAutDocSustento` | Optional; if sent, the 49-digit access key of the electronic support document |
| `pagoLocExt` | Only `01` (local payment); foreign payment is not supported |

## States

`estado` of `GET /v1/operaciones/{idOperacion}` (delivery of the document, not the fiscal result):

| State | Meaning | Final |
|---|---|---|
| `en_cola` | Received, waiting to be sent (also while a temporarily failed send is retried) | No |
| `enviando` | Being delivered for fiscal processing | No |
| `enviado` | Delivered; the SRI result is in `estadoAutorizacion` | No |
| `resultado_desconocido` | Delivery could not be confirmed; the platform keeps verifying and retrying while send retries remain. Do not reissue | No |
| `rechazado` | Rejected during fiscal processing, before an SRI result or by the SRI. Fix and resend | Yes |
| `cancelado` | Cancelled before sending; the reserved quota was released | Yes |
| `desconocido` | Value not recognized; should not appear. Treat as not final and contact support with the `idOperacion` if it persists | No |

`estadoAutorizacion` (in tracking and in the issued-documents list):

| Value | Meaning |
|---|---|
| `no_disponible` | No SRI result yet |
| `pendiente` | The SRI is processing the document |
| `autorizado` | The SRI authorized the document (the only value with fiscal validity). Final |
| `error` | The SRI rejected the document (not authorized). Final. Fix it and reissue with a new `Idempotency-Key` |
| `desconocido` | Value not recognized; should not appear. Treat as not final; keep checking and contact support if it persists |

Webhooks do not carry `estadoAutorizacion`; their `estado` is `authorized` (equivalent to `autorizado`) or `rejected` (equivalent to `error`). The SRI's own portals use `PPR` (processing, = `pendiente`), `AUT` (authorized, = `autorizado`) and `NAT` (not authorized, = `error`); the API does not return these acronyms.

Never call a merely received document "authorized": the issuance `202` only means received.

## Other tables

- `moneda`: only `DOLAR` or `USD` (other values `422`). Required in invoice, purchase settlement and credit note; optional in debit note; not sent in shipping guide or withholding (`422`).
- Environment (SRI code, position 24 of the access key): `1` test (`https://staging-api.mynexusapi.com`), `2` production (`https://api.mynexusapi.com`). Do not send `ambiente` or `tipoEmision` (`422`): the URL defines the environment and emission is always normal (`1`).
- Countries (`paisOrigen`, `paisDestino`, `paisAdquisicion` in an export invoice): the SRI 3-digit numeric country code; Ecuador is `593`. The full list is in the catalogs guide.
- `detalles[].unidadMedida`: free text (usual values `Kilos`, `Litros`, `Unidades`, `Cajas`); the SRI publishes no closed catalog, and the value is not included in the document sent to the SRI.
