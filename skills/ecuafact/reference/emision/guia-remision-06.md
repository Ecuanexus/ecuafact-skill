# Delivery guide / Guia de remision (`codDoc` = `06`)

Backs the transport of goods: who carries them, from where, to whom and what. Issue it **before** the goods leave. It **carries no amounts** and uses `destinatarios` (exactly one recipient) instead of top-level `detalles`; to deliver to several recipients, issue one guide per recipient.

Common structure (`origenReferencia`, `referenciaExterna`, `infoTributaria` with `codDoc` `"06"`, `info`, `destinatarios`, `infoAdicional` up to 20) is the same as the invoice. `infoTributaria` rules: `ruc` is the issuer's full 13-digit RUC and its first 10 digits must equal the route `{identificacion}` (different prefix -> `422`/`302`; same prefix, different last 3 digits -> `403`/`103`); `estab` and `ptoEmi` up to 3 digits, left-padded with zeros; `secuencial` optional (omit it or send all zeros to auto-assign the next number of the guide series; a used number returns `409`/`401`). Do not send `claveAcceso`, `ambiente`, `tipoEmision`, `version` or `formato` (`422`, `codigo` `302`). Do not send top-level `detalles`, `motivos` or `docsSustento` (`422`). A `202` means received, not authorized.

The guide has no `fechaEmision`: `fechaIniTransporte` acts as the document date. Sending `fechaEmision`, `moneda`, `pagos` or any total returns `422`.

## `info` fields

| Field | Type | Required | Rule |
|---|---|---|---|
| `dirEstablecimiento` | string | No | Up to 500 characters |
| `dirPartida` | string | Yes | Departure address, up to 300 characters |
| `razonSocialTransportista` | string | Yes | Carrier name, up to 300 characters |
| `tipoIdentificacionTransportista` | string | Yes | `04` RUC, `05` cedula (national ID), `06` pasaporte (passport), `08` identificacion del exterior (foreign ID) |
| `rucTransportista` | string | Yes | Carrier ID despite the name; follows the type: 13 digits (`04`), 10 (`05`), up to 20 characters (`06`/`08`) |
| `fechaIniTransporte` | string | Yes | Transport start, `dd/MM/yyyy`; also the guide's issue date |
| `fechaFinTransporte` | string | Yes | Transport end, `dd/MM/yyyy`, equal to or later than `fechaIniTransporte` |
| `placa` | string | Yes | Vehicle plate, up to 20 characters |
| `telefono` | string | No | Recipient's phone, up to 40 characters |
| `correo` | string | No | Recipient's email, up to 500 characters. If omitted, an `infoAdicional` entry named `email` or `correo` is used |

## `destinatarios` (exactly one)

| Field | Type | Required | Description |
|---|---|---|---|
| `identificacionDestinatario` | string | Yes | Exactly 10 or 13 digits (cedula or RUC); the type is inferred from the length. There is no type field; passports and foreign IDs are rejected |
| `razonSocialDestinatario` | string | Yes | Up to 300 characters |
| `dirDestinatario` | string | Yes | Delivery address, up to 300 characters |
| `motivoTraslado` | string | Yes | Reason for the transfer (e.g. `VENTA`, `TRASLADO ENTRE ESTABLECIMIENTOS`, `DEVOLUCION`), up to 300 characters |
| `docAduaneroUnico` | string | No | Customs document number, up to 300 characters |
| `codEstabDestino` | string | No | Destination establishment code, digits only, up to 3 |
| `ruta` | string | No | Route, up to 300 characters; cannot be only spaces |
| `detalles` | array | Yes | 1 to 1000 lines without amounts or taxes (see below) |

The recipient does not accept `codDocSustento`, `numDocSustento`, `numAutDocSustento` or `fechaEmisionDocSustento` (`422`).

`destinatarios[].detalles`: `codigoInterno` (required, up to 25), `codigoAdicional?`, `descripcion` (up to 300), `cantidad` (> 0), `detallesAdicionales?` (up to 3 per line, e.g. lot or serial). `codigoPrincipal` and `codigoAuxiliar` return `422`; do not send `precioUnitario`, `descuento`, `precioTotalSinImpuesto` or `impuestos` either.

## Example

Test environment: no fiscal validity.

```json
{
  "origenReferencia": "MiERP",
  "referenciaExterna": "GR-2026-0004",
  "infoTributaria": { "ruc": "0123456789001", "codDoc": "06", "estab": "002", "ptoEmi": "001" },
  "info": {
    "dirEstablecimiento": "AV. PRINCIPAL 123",
    "dirPartida": "BODEGA CENTRAL",
    "razonSocialTransportista": "TRANSPORTISTA DE PRUEBA",
    "tipoIdentificacionTransportista": "05",
    "rucTransportista": "0123456789",
    "fechaIniTransporte": "07/10/2026",
    "fechaFinTransporte": "08/10/2026",
    "placa": "PBA1234"
  },
  "destinatarios": [
    {
      "identificacionDestinatario": "0123456789001",
      "razonSocialDestinatario": "DESTINATARIO DE PRUEBA",
      "dirDestinatario": "SUCURSAL NORTE",
      "motivoTraslado": "VENTA",
      "codEstabDestino": "001",
      "ruta": "QUITO - GUAYAQUIL",
      "detalles": [
        { "codigoInterno": "PROD-001", "descripcion": "Caja de producto", "cantidad": 10 }
      ]
    }
  ]
}
```

Downloadable example: `https://docsapi.ecuafact.com/downloads/v1/ejemplos/example-06.json`. Full guide: `https://docsapi.ecuafact.com/v1/guias/emision-06.md`.
