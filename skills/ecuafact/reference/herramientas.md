# Developer tooling

Besides the HTTP API, Ecuafact publishes a CLI, a Postman collection, five official SDKs and two MCP servers. Every guide named here is available at `https://docsapi.ecuafact.com/v1/guias/<slug>` (append `.md` for Markdown), for example `https://docsapi.ecuafact.com/v1/guias/cli-referencia.md`. All tools are previews; if a version or flag below differs from the live guide, the guide wins.

## CLI `ecuafact`

The CLI issues, tracks and downloads documents from the terminal or a script, without writing HTTP code. Guides: `https://docsapi.ecuafact.com/v1/guias/cli` (install and quick start), `https://docsapi.ecuafact.com/v1/guias/cli-emision` (issuing and batches), `https://docsapi.ecuafact.com/v1/guias/cli-csv-excel` (CSV and Excel templates), `https://docsapi.ecuafact.com/v1/guias/cli-vigilar` (folder watching) and `https://docsapi.ecuafact.com/v1/guias/cli-referencia` (every option and exit code).

### Install

| Manager | Install | Update / uninstall |
|---|---|---|
| npm | `npm install -g @ecuafact/cli@preview` | repeat the install / `npm uninstall -g @ecuafact/cli` |
| Scoop (Windows) | `scoop bucket add ecuafact https://github.com/Ecuanexus/ecuafact-scoop` then `scoop install ecuafact/ecuafact-preview` | `scoop update ecuafact-preview` / `scoop uninstall ecuafact-preview` |
| Homebrew (macOS, Linux) | `brew install ecuanexus/ecuafact/ecuafact-preview` | `brew upgrade ecuafact-preview` / `brew uninstall ecuafact-preview` |

Check the version with `ecuafact --version`. Before uninstalling, remove any folder-watch services with `ecuafact vigilar desinstalar --nombre <nombre>`.

### Environments and authentication

- Two environments: `sandbox` (testing, no fiscal validity) and `produccion`. **`produccion` is the default** for a new profile, so set `ecuafact config poner entorno sandbox` or pass `--entorno sandbox` before testing. Each environment has its own API Key.
- `ecuafact login` prompts for the API Key without echoing it and stores it for the current profile and environment; `ecuafact sesion` shows which key source is active (without showing it) and `ecuafact logout` deletes it. In scripts use `login --desde-stdin` (`echo "$ECUAFACT_KEY_SANDBOX" | ecuafact login --desde-stdin --entorno sandbox`).
- On servers and CI, set `ECUAFACT_API_KEY`; it takes precedence over the stored key. Never pass the API Key as a command argument (it ends up in shell history).
- The machine's public IP must be registered in the portal (**Cuenta > IP permitidas**). `ecuafact doctor` checks connection, key, IP and quota.
- In `produccion`, commands that issue or change data ask for confirmation; in scripts add `--confirmar produccion`.
- Precedence for each setting: command option, `ECUAFACT_*` variable, `ecuafact.json` in the current or a parent folder (never holds the API Key), profile (`config poner`), default.
- When not attached to a terminal, the CLI writes JSON to stdout and JSON errors to stderr (`error`, `codigoSalida`, `sugerencia`, and `estadoHttp`, `codigo`, `idCorrelacion` when the API returned the error). Exit codes: `0` ok, `1` general/network/server, `2` usage, `3` auth/IP/permission, `4` validation, `5` SRI rejected, `6` rate limit or no quota, `7` partial, `8` not found or conflict. Exit `0` from `emitir` includes documents still `enviado` or `en_proceso`; use `--esperar` or `--reporte` before treating them as authorized.
- `ecuafact schema [comando]` describes the CLI in JSON for scripts and AI agents.

### Main commands

| Command | Purpose |
|---|---|
| `ecuafact nuevo <tipo> [--archivo <archivo>]` | Create a sample document (`factura`, `liquidacion`, `nota-credito`/`nc`, `nota-debito`/`nd`, `guia`, `retencion`, or `01`-`07`) |
| `ecuafact validar <entradas...> [--en-linea]` | Validate without issuing. Locally it checks structure, formats, lines, `totalSinImpuestos` against the lines and that payments add up to `importeTotal`; with `--en-linea` each document is also validated by the API with every issuing rule (line VAT, header taxes, totals, RUC and type enabled). No quota used |
| `ecuafact emitir <entradas...>` | Issue `.json` (object or array), `.jsonl`, folders (`-r` recursive) or stdin (`-`). Key options: `--esperar`, `--descargar ride,xml` (implies `--esperar`), `--dry-run` (validates each document in the API and issues nothing), `--reporte <csv/json>`, `--plantilla <archivo>` for CSV/Excel, `--continuar-con-errores`, `--concurrencia <1-10>` |
| `ecuafact lote listar` / `lote estado <lote>` / `lote reanudar <lote> [--esperar]` / `lote borrar <lote>` | Every `emitir` run is a batch; resume an interrupted one in the same environment without duplicating |
| `ecuafact estado <idOperacion>` | Operation status |
| `ecuafact comprobantes listar [--desde] [--hasta] [--tipo] [--todas]` | List issued documents (default last 30 days) |
| `ecuafact descargar [claveAcceso...] [--tipo ride,xml] [-d <carpeta>]` | Download RIDE (default) and/or XML as `<claveAcceso>.pdf` / `.xml`; without keys, `--desde`/`--hasta` downloads every authorized document in the period |
| `ecuafact correo reenviar <claveAcceso> --a <correos>` | Resend the email (up to 20 addresses) |
| `ecuafact vigilar <carpeta>` | Issue every file that appears in a folder, moving it to `procesados/` or `errores/` (with `<archivo>.error.json`). `--una-vez` for cron; `vigilar instalar <carpeta> --nombre <nombre>` installs it as a system service (`--usuario` for a user service on Linux/macOS), `vigilar estado` / `vigilar desinstalar --nombre <nombre>` |
| `ecuafact plantilla nueva factura` / `plantilla probar <archivo> --plantilla <archivo>` | Create and dry-run a CSV/Excel mapping template (the CLI does not compute totals; the file must carry them) |
| `ecuafact cuenta`, `consumo`, `perfil-emisor ver/editar/logo` | Integration context, quota, issuer profile |
| `ecuafact ruc`, `identificacion validar`, `clave decodificar`, `establecimientos`, `catalogos` | SRI lookups |
| `ecuafact api <metodo> <ruta>` | Call any API route with the stored key (pass a stable `--idempotency-key` when issuing) |

### Bulk issuing from CSV or Excel

These three commands create a sample template, dry-run it and issue in the test environment:

```bash
ecuafact plantilla nueva factura
ecuafact plantilla probar ejemplo-factura.csv --plantilla mapeo-factura.json
ecuafact --entorno sandbox emitir ejemplo-factura.csv --plantilla mapeo-factura.json --esperar --reporte resultado.csv
```

1. `plantilla nueva factura` writes the template `mapeo-factura.json` and a sample `ejemplo-factura.csv`. Put the issuer RUC in `infoTributaria.ruc` and rename the columns to match your file.
2. `plantilla probar` prints the JSON of every document the file would produce, without sending it; each one should say `"validacionLocal": "sin errores"`.
3. `emitir` sends them and writes the result of each row to `resultado.csv`.

The template is JSON. Inside `documento`, each key is a document field and each value is a column (`"{columna}"`, optionally typed as `{fecha:fecha}`, `{importe_total:numero}` or `{establecimiento:digitos3}`) or a fixed value. `lineas.coleccion` (for example `detalles`) with `lineas.campos` maps one line per row. `agruparPor` names the column that groups rows into one document (without it, each row is a document); `separador` (CSV: `","`, `";"` or `"tab"`, detected by default) and `hoja` (Excel sheet, first by default) are optional. The CLI does not compute totals, so the file must carry them. Full guide: `https://docsapi.ecuafact.com/v1/guias/cli-csv-excel`.

The CLI recognizes a document by `origenReferencia` + `referenciaExterna`, so re-running the same file does not duplicate it; give a corrected or new document a new `referenciaExterna`.

### Webhook commands

Both commands are local tests of your endpoint and do not call the API. They read the destination secret from `ECUAFACT_WEBHOOK_SECRET` or from stdin with `--secreto-desde-stdin`.

- `ecuafact webhooks probar --a <url>` sends a signed sample event with the same headers as a real delivery (`Ecuafact-Signature` and the Standard Webhooks headers). Options: `--evento` (`document.authorized` default, `document.rejected`, `document.failed`, `operation.requires_attention`), `--motivo` (for `document.failed`, default `entrega_no_posible`), `--clave-acceso`, `--id-operacion` (new one per send by default; reuse it to test deduplication), `--firma-invalida` (your endpoint should answer `401`).
- `ecuafact webhooks verificar` checks a received signature, either `--firma "t=...,v1=..."` **or** `--webhook-id`, `--webhook-timestamp` and `--webhook-signature` together (the two schemes cannot be mixed), plus `--cuerpo @archivo` (or `-` for stdin), `--tolerancia <segundos>` (default 300) and `--ahora <epoch>` to check an old event. The output's `esquema` field says which scheme was verified.

```bash
ecuafact webhooks verificar --webhook-id 9b2e4f7a-1c3d-4e5f-8a6b-7c8d9e0f1a2b --webhook-timestamp 1791385446 --webhook-signature "v1,6YgtHWzoIeuArJTPh8Oq8lpg//3ldAAsy1IRzD5eiG0=" --cuerpo @evento.json --ahora 1791385446
```

## Postman collection

Guide: `https://docsapi.ecuafact.com/v1/guias/colecciones`. The environment points to staging (`https://staging-api.mynexusapi.com`).

- Collection (Postman v2.1, importable in Bruno and Newman): `https://docsapi.ecuafact.com/downloads/v1/Ecuafact-API-v1.postman_collection.json`
- Environment: `https://docsapi.ecuafact.com/downloads/v1/Ecuafact-API.postman_environment.json`
- Also published: OpenAPI 3.1 at `https://docsapi.ecuafact.com/api/openapi.json` and one sample request per type at `https://docsapi.ecuafact.com/downloads/v1/ejemplos/example-<01|03|04|05|06|07>.json`.

Environment variables: `baseUrl` (staging), `baseUrlProduccion`, `apiKey` (empty; mark it secret), `identificacion`, `ruc`, `estab`, `ptoEmi` (template ships `002` / `001`), `claveAcceso` (sample, for decoding), `idOperacion` and `idOperacion01`-`idOperacion07` (filled when issuing), `claveAccesoEmitida` (filled by **02.1** once the operation is authorized), `correoDestino` and `nombreCatalogo`. Without the environment selected, authenticated requests return `401`.

| Folder | Contents |
|---|---|
| `00` | Health (`GET /salud`, no key), context, quota |
| `01` | One issuance per type; each stores its `idOperacion` and consumes sandbox quota |
| `02` | View operation (stores `claveAccesoEmitida` when authorized) |
| `03` | Issued documents |
| `04` | Error cases: 401, 400 without `Idempotency-Key`, 422, 429 quota (only if really exhausted), 501 XML |
| `05` | SRI lookups and catalog (chosen by `nombreCatalogo`); no quota |
| `06` | RIDE, XML download and email resend, using `claveAccesoEmitida` |
| `07` | Read profile, update profile, change logo |

**Warning:** **06.3** sends a real email to `correoDestino` on every run, and **07.2** and **07.3** change the real issuer profile of the environment. Each issuance request generates a new `Idempotency-Key` and `referenciaExterna`, so re-sending issues a new document.

Newman (`npm install -g newman`), passing the key through an environment variable and limiting the run with `--folder` to avoid the side effects above:

```bash
newman run Ecuafact-API-v1.postman_collection.json \
  -e Ecuafact-API.postman_environment.json \
  --env-var "apiKey=$ECUAFACT_API_KEY" \
  --env-var "identificacion=0123456789" \
  --env-var "ruc=0123456789001" \
  --folder "00 · Inicio y diagnostico" --folder "05 · Consultas SRI"
```

`--folder` takes the folder's full name as it appears in the collection (for example `00 · Inicio y diagnostico`, `01 · Emitir comprobantes`, `02 · Seguimiento de operacion`) and can be repeated.

## SDKs

Guide: `https://docsapi.ecuafact.com/v1/guias/sdk`, plus one guide per language (`https://docsapi.ecuafact.com/v1/guias/sdk-<lang>`, in the last column). All five are pre-release, so the install command must request it explicitly.

| Language | Package | Install | Version | Minimum runtime | Guide |
|---|---|---|---|---|---|
| .NET | `Ecuafact.Sdk` (+ `Ecuafact.Sdk.Core` for DI) | `dotnet add package Ecuafact.Sdk --prerelease` (and `dotnet add package Ecuafact.Sdk.Core --prerelease`) | `1.0.0-preview.4` | `netstandard2.0` / .NET Framework 4.6.1 | `sdk-dotnet` |
| Node.js | `@ecuafact/sdk` | `npm install @ecuafact/sdk@beta` | `1.0.0-beta.4` | Node.js 18 (ES module) | `sdk-node` |
| Python | `ecuafact` | `pip install "ecuafact>=1.0.0b5"` (not `pip install --pre`) | `1.0.0b5` | Python 3.10 | `sdk-python` |
| Java | `com.ecuafact:sdk` (Maven Central) | Maven/Gradle dependency `com.ecuafact:sdk:1.0.0-beta4` | `1.0.0-beta4` | Java 17 | `sdk-java` |
| PHP | `ecuafact/sdk` | `composer require "ecuafact/sdk:^1.0@beta"` | `v1.0.0-beta4` | PHP 8.2 with `ext-curl` | `sdk-php` |

The Postman collection and the sandbox environment are also in the public workspace `https://www.postman.com/ecuafact/ecuafact-api` (fork it to get updates).

Ready-to-run examples in .NET, Node.js, Python, PHP and Java, both with plain HTTP and with the SDK (one console script per task and a small web app with signed webhooks), are in `https://github.com/Ecuanexus/ecuafact-api-ejemplos`. Point users there when they want working code for their language.

Every SDK can validate a document without issuing it (`POST .../comprobantes/validar`): `ValidarAsync` (.NET), `validar`/`validarEn` (Node, PHP, Java), `validar`/`validar_en` (Python), and `validar` on the per-taxpayer view `para(...)`. They return a result with `valido`, `codigo`, `mensaje` and `errores`: a `422`/`302` comes back as `valido` = false instead of an exception, and any other error throws as in other calls.

Without `@beta`, npm installs an older beta and Composer fails looking for a stable release. The SDKs read `ECUAFACT_API_KEY` and `ECUAFACT_BASE_URL` (base URL without `/v1`; there is no environment option, the URL decides), generate and reuse the `Idempotency-Key`, retry `408`, `425`, `500`, `502`, `503`, `504` and network errors (3 attempts by default, honoring `Retry-After`), never retry quota exhaustion (`429` `codigo` `501`), do not retry rate limiting (`codigo` `104`) unless enabled, and include a webhook verifier for `Ecuafact-Signature` that accepts several secrets. Since 2026-10-21 the default per-attempt timeout is 30 seconds.

## MCP servers

Guide: `https://docsapi.ecuafact.com/v1/guias/mcp`. Both use the Streamable HTTP transport.

| Server | URL | Auth |
|---|---|---|
| Account, staging | `https://mcp-staging.mynexusapi.com/mcp` (authorized at `https://staging-portal.mynexusapi.com`) | OAuth 2.1 |
| Account, production | `https://mcp.mynexusapi.com/mcp` (authorized at `https://portal.mynexusapi.com`) | OAuth 2.1 |
| Documentation | `https://docsapi.ecuafact.com/mcp` | None (public, read-only) |

- **Account server.** Access is OAuth 2.1, not the API Key, and no IP registration is needed. A person connects with the URL and approves permissions and RUCs in the portal; an unattended system uses a **bot** (`client_credentials`, 15-minute tokens, no refresh token). Scopes: `cuenta:leer` (always), `comprobantes:leer`, `sri:consultar`, `comprobantes:emitir`, `comprobantes:enviar`, `offline_access`. Each tool returns the API JSON as-is. Start with `obtener_contexto`. Staging accounts and agents do not work in production and vice versa. Claude Code: `claude mcp add --transport http ecuafact https://mcp-staging.mynexusapi.com/mcp`, then `/mcp` > **Authenticate**.
- **Documentation server.** Public, no account or credentials, 300 queries per minute per IP. Tools: `listar_guias`, `buscar_documentacion`, `leer_guia`, `explicar_error`, `ejemplo_comprobante`, `consultar_referencia` (the account server includes the same six under `cuenta:leer`). Claude Code: `claude mcp add --transport http ecuafact-docs https://docsapi.ecuafact.com/mcp`. It is listed in the official MCP Registry (`registry.modelcontextprotocol.io`) as `com.ecuafact.docsapi/documentacion`.

### Issuing through the account MCP

Optional validation: `validar_comprobante` (`identificacion`, `estructura`) applies the issuing rules without issuing or using quota and answers `Comprobante valido.` or the error issuing would return. It comes with the issuing permission and is announced as read-only. `emitir_comprobante` applies the same rules, so use it when you build documents yourself (batches, files) or when portal confirmation is on, to avoid asking the user to approve documents that would fail.

`emitir_comprobante` requires `confirmar` = `true` and a stable `idempotencyKey` (reuse it on retries); `reenviar_correo` also requires `confirmar` = `true`. Show the user the document or the recipients and get their confirmation before sending `confirmar=true`. A `202` means received, not authorized: follow up with `consultar_operacion`.

**Portal confirmation.** If the agent was connected with **Pedir mi confirmacion para emitir y enviar correos** checked (the default when emit or email permissions are granted), the first call to `emitir_comprobante` or `reenviar_correo` does not execute and returns `Pendiente de aprobacion: ...` with a portal link that expires in 30 minutes. The agent must:

1. Show the link to the user and ask them to open it and press **Aprobar** (or **Rechazar**).
2. Wait for the user to say they approved.
3. Repeat the call with **exactly the same data**: same `idempotencyKey` and `estructura`, or same `claveAcceso` and recipients. Any change (total, item, recipient) requires a new approval. After approval the agent has 30 minutes to repeat the call; an issuance approval covers retries with the same `idempotencyKey`, an email approval covers a single send. If the user rejected, the tool answers `El usuario rechazo ... en el portal`; ask what to change.
