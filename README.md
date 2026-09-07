# Support Card

**An open standard for the support desk of AI agents.** A product publishes a small JSON document at `https://{domain}/.well-known/support.json` that tells any AI agent where and how to get help: a support endpoint (MCP or plain HTTP), the tools it exposes (`ask`, `act`, `escalate`, `status`), the levels of principal verification the vendor accepts, and a human fallback.

Status: **Draft 0.1**, 7 September 2026. Licence: CC0 1.0, no rights reserved. Maintained by [Wellknown](https://wellknown.support); vendor-neutral by design, any product can publish a card pointing at its own implementation.

## Why

Agents (coding assistants, browser agents, workflow automations) now use products on behalf of humans. When one of them hits an error it cannot understand, it has nowhere to ask: it retries in a loop, pastes a useless message into the human chat, or gives up. Every support product goes vendor → human. Nothing goes agent → vendor. This card is that missing entry point, in the same spirit as `robots.txt`, `security.txt` and `llms.txt`.

## The card, in one glance

```json
{
  "version": "0.1",
  "product": { "name": "Acme API", "url": "https://acme.dev", "docs": "https://docs.acme.dev" },
  "support": { "mcp": { "url": "https://acme.wellknown.support/mcp", "transport": "streamable-http" } },
  "tools": ["ask", "escalate", "status"],
  "verification": { "levels": [0] },
  "contact": { "email": "support@acme.dev" },
  "updated": "2026-09-07"
}
```

An agent fetches the card, opens the MCP endpoint, and calls `ask` with the error it hit. It gets a short answer grounded in the vendor's documentation, citations, a confidence score and a next step (`retry`, `fix_request`, `read`, `escalate`, `wait`). If that is not enough, `escalate` hands the full context to a human and returns a ticket id the agent can poll with `status`. Every closed conversation produces a **receipt** for the agent's human principal.

## Contents

| File | What |
|---|---|
| [support-card.md](support-card.md) | The specification: discovery, document format, the four tools, principal verification levels 0 to 3, receipts, errors, security |
| [support-card.schema.json](support-card.schema.json) | Normative JSON Schema (draft 2020-12) |
| [examples/](examples/) | Example cards: minimal, a self-hosted API vendor, Wellknown's own |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to propose changes, open questions for 0.2 |

## Tools

- **Validator**: [wellknown.support/validate](https://wellknown.support/validate) checks any card against the schema and the discovery rules (HTTPS, content type, CORS, caching, redirects).
- **npm**: `npm install @wellknown/spec` gives you the schema, TypeScript types, `validateSupportCard()` and a `wellknown-spec validate` command.
- **Hosted implementation**: [Wellknown](https://wellknown.support) hosts cards and the MCP support server for products that do not want to run their own.

## Publish a card in three steps

1. Write the JSON (start from [examples/minimal.support.json](examples/minimal.support.json)).
2. Serve it at `/.well-known/support.json` over HTTPS with `Content-Type: application/json` and `Access-Control-Allow-Origin: *`, or redirect that path to a hosted card.
3. Reference it from your `llms.txt` (`Support card: https://…`), your MCP server card, or a `Link: <…>; rel="support-card"` header.

## Relation to other standards

The card complements MCP (the support endpoint is an ordinary MCP server), `llms.txt` (documentation for models; the card is where to go when documentation was not enough), Web Bot Auth (level 3 agent identity) and OpenAPI (`info.x-support-card`). See section 9 of the specification.

## Feedback

Open an issue. Changes to the card format or the tool contracts go through the process in [CONTRIBUTING.md](CONTRIBUTING.md).
