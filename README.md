# Skill de Ecuafact para asistentes de IA

Skill oficial de la [API de facturación electrónica de Ecuafact](https://docsapi.ecuafact.com) (SRI, Ecuador) para Claude Code, GitHub Copilot, Cursor y cualquier agente compatible con el estándar **Agent Skills**. Le enseña al asistente el contrato de la API para que escriba integraciones correctas sin inventar campos ni códigos: emisión de los seis comprobantes (factura, liquidación de compra, nota de crédito, nota de débito, guía de remisión y retención), validación sin emitir, seguimiento, RIDE y XML, consultas al catastro del SRI, webhooks firmados, SDK, CLI, colección de Postman y servidor MCP.

*English: official Agent Skill for the Ecuafact e-invoicing API for Ecuador's tax authority (SRI). The instructions are in English; the API contract (fields, routes, codes, catalog values) stays in Spanish, and the assistant answers in the user's language.*

## Instalación

Con la CLI de skills:

```bash
npx skills add Ecuanexus/ecuafact-skill
```

O a mano: copia la carpeta `skills/ecuafact` en la carpeta de skills de tu asistente, de modo que queden `ecuafact/SKILL.md` y `ecuafact/reference/...`.

| Asistente | Carpeta |
|---|---|
| Claude Code (este proyecto) | `.claude/skills/ecuafact/` |
| Claude Code (todos tus proyectos) | `~/.claude/skills/ecuafact/` |
| GitHub Copilot en VS Code | `.github/skills/ecuafact/` |
| Otros agentes compatibles | La carpeta de skills que indique tu herramienta |

Para comprobar que lo usa, pregúntale: "Con el skill de ecuafact, ¿qué cabeceras necesito para emitir una factura?". Debe responder `X-Api-Key`, `Idempotency-Key` y `Content-Type: application/json`, y mencionar que la respuesta es `202 Accepted`.

## Contenido

| Archivo | Para qué |
|---|---|
| `skills/ecuafact/SKILL.md` | Ambientes, autenticación, ciclo de vida, emisión, validación sin emitir, endpoints y mapa de recursos |
| `reference/emision/*.md` | Campos y ejemplo de cada tipo de comprobante |
| `reference/consultas.md`, `consultas-sri.md` | Consultas de la cuenta y del catastro del SRI |
| `reference/webhooks.md` | Eventos, firmas (`Ecuafact-Signature` y Standard Webhooks) y reintentos |
| `reference/errores-limites.md`, `codigos-sri.md` | Errores con qué hacer, límites y códigos del SRI |
| `reference/herramientas.md` | SDK (.NET, Node.js, Python, Java, PHP), CLI `ecuafact`, Postman y servidores MCP |

La documentación vigente manda si difiere del skill. Si tu asistente tiene acceso a internet, conéctalo también al servidor MCP público de la documentación: `https://docsapi.ecuafact.com/mcp`.

## Enlaces

- Documentación: https://docsapi.ecuafact.com
- Guía del skill: https://docsapi.ecuafact.com/v1/guias/skill
- Servidor MCP de la cuenta: https://docsapi.ecuafact.com/v1/guias/mcp
- Changelog: https://docsapi.ecuafact.com/v1/guias/changelog

## Licencia

MIT. Ver [LICENSE](LICENSE).
