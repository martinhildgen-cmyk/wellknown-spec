# Support Card specification

**Status:** Draft 0.2 (in progress); cards declare "version": "0.1". First draft 7 September 2026. Open standard, CC0. Feedback: [open an issue](https://github.com/martinhildgen-cmyk/wellknown-spec/issues).

## 1. Purpose

Software products have support channels for humans: chat widgets, email, help centers. AI agents that use these products on behalf of humans (coding agents, browser agents, workflow automations, custom agents) have none. When an agent hits an error it cannot understand, it retries, gives up, or pastes a useless message into the human chat.

A **support card** is a small JSON document published at a well-known URL that tells any agent where and how to get help for a product: a support endpoint it can call (MCP or plain HTTP), the tools that endpoint exposes, the levels of principal verification the vendor accepts, and a human fallback.

The card is vendor-neutral. Wellknown hosts cards and support servers, but any vendor can publish a card pointing at its own implementation.

## 2. Discovery

A product publishes its card at:

```
https://{product-domain}/.well-known/support.json
```

Rules:

- `Content-Type: application/json; charset=utf-8`.
- Must be served over HTTPS, with `Access-Control-Allow-Origin: *`.
- May redirect (301/302/307/308) to a hosted card on another origin, for example `https://{slug}.wellknown.support/.well-known/support.json`. Clients must follow up to three redirects.
- Should be cacheable. A card served from the vendor's own domain should use `Cache-Control: max-age=3600` or longer. A provider that hosts cards may use a shorter lifetime on purpose, because a hosted card changes when the product it describes changes (its name, its escalation channel, its verification): Wellknown serves hosted cards with `max-age=300`.

Additional discovery hints, all optional:

- HTTP response header on any page of the product: `Link: <https://example.com/.well-known/support.json>; rel="support-card"`.
- HTML: `<link rel="support-card" href="/.well-known/support.json">`.
- In an MCP server card (`/.well-known/mcp/server.json`), a `support` field holding the card URL.
- In `llms.txt`, a line `Support card: https://example.com/.well-known/support.json`.
- In an OpenAPI document, `info.x-support-card`.

## 3. Document format

Top-level fields. See `support-card.schema.json` for the normative schema.

| Field | Required | Description |
|---|---|---|
| `version` | yes | Spec version, currently `"0.1"` |
| `product` | yes | Object: `name`, `url`, optional `docs`, `status_url`, `changelog`, `openapi`, `llms_txt`, `mcp_server` |
| `support` | yes | Object with at least one of `mcp` or `http` |
| `support.mcp` | no | Object: `url` (streamable HTTP endpoint), `transport` (`"streamable-http"`), optional `auth` (`"none"`, `"bearer"`, `"oauth"`) |
| `support.http` | no | Object: `base_url` exposing `POST /ask`, `POST /escalate`, `GET /status`, optional `POST /act` |
| `tools` | yes | Array of tool names available: any of `ask`, `act`, `escalate`, `status` |
| `verification` | yes | Object: `levels` (array of integers 0 to 3 accepted), optional `introspection_url` (level 1), `oauth` (level 2: `authorization_url`, `token_url`, `scopes`), `web_bot_auth` (level 3: `directory_url`) |
| `actions` | no | Array of declared actions for `act`: `id`, `description`, `params` (JSON Schema), `min_level`, `confirmation` (`"none"`, `"principal"`, `"vendor"`), optional `rate_limit` |
| `policies` | no | Object: `rate_limit` (`{ "requests": integer, "per": ... }`, where `per` is `"minute"`, `"hour"` or `"day"`, per agent identity; section 7 gives the default), `languages`, `human_hours` (ISO 8601 recurring interval or free text), `data_retention_days` |
| `receipts` | no | Object: `enabled`, `url_template` (with `{receipt_id}`), `delivery` (an array of `"inline"`, `"email"`, `"webhook"`; `"email"` only when `verification.levels` accepts level 1 or above, section 6) |
| `contact` | yes | Human fallback: `email` and/or `url` |
| `updated` | yes | ISO 8601 date of last change |
| `provider` | no | Who hosts the support endpoint: `name`, `url`, optional `notice` |

`provider.notice` is one line, at most 300 characters, in which a provider hosting the card for a product says the card is not the vendor's own: for example an unofficial demo built from the product's public documentation, or a product whose vendor has not confirmed it yet. Agents should relay it with the product name and must not present answers or escalations from such a card as coming from the vendor. A vendor's own card carries no notice.

A minimal card:

```json
{
  "version": "0.1",
  "product": { "name": "Acme API", "url": "https://acme.dev", "docs": "https://docs.acme.dev" },
  "support": { "mcp": { "url": "https://acme.wellknown.support/mcp", "transport": "streamable-http", "auth": "none" } },
  "tools": ["ask", "escalate", "status"],
  "verification": { "levels": [0] },
  "contact": { "email": "support@acme.dev" },
  "updated": "2026-09-07"
}
```

## 4. Tools

The support endpoint exposes up to four tools. Names, inputs and outputs are normative; MCP servers declare them as MCP tools with the JSON Schemas below; HTTP servers accept the same JSON body on `POST {base_url}/{tool}` (and `GET {base_url}/status`).

### 4.1 `ask`

Answer a question from the vendor's documentation, with citations.

Input:

```json
{
  "question": "string, required",
  "context": {
    "error": "string, optional: error message or code received",
    "endpoint": "string, optional: URL, method or tool name involved",
    "sdk": "string, optional: SDK or client and version",
    "attempts": "integer, optional: how many times the agent already retried"
  },
  "principal": { "level": 0, "token": "optional, see section 5", "email": "optional, where the human principal can be reached" },
  "conversation_id": "string, optional: continue a previous conversation"
}
```

Output:

```json
{
  "conversation_id": "string",
  "answer": "string, short, in the language of the question",
  "citations": [ { "url": "string", "title": "string", "quote": "string" } ],
  "confidence": "number 0 to 1",
  "next": "one of retry | fix_request | read | escalate | wait",
  "receipt": { "id": "string", "url": "string" }
}
```

Rules: an answer without at least one citation must set `confidence` below 0.5 and `next` to `escalate`. Servers must not invent product behaviour that is not in the indexed sources.

### 4.2 `act`

Execute one of the actions declared in `actions`.

Input:

```json
{
  "action": "string, one of the declared action ids",
  "params": { },
  "principal": { "level": 1, "token": "required at the action's min_level or above" },
  "conversation_id": "string, optional",
  "idempotency_key": "string, recommended"
}
```

Output:

```json
{
  "status": "one of done | pending_confirmation | refused | failed",
  "result": { },
  "message": "string",
  "receipt": { "id": "string", "url": "string" }
}
```

Rules: an action not present in `actions` is refused with `status: refused`, whatever the request text says. `pending_confirmation` means a human (principal or vendor) must confirm; the agent polls `status` with the `receipt.id`.

### 4.3 `escalate`

Hand the conversation to a human at the vendor.

Input:

```json
{
  "summary": "string, required, one paragraph",
  "transcript": [ { "role": "agent | support", "content": "string" } ],
  "principal": { "level": 0, "token": "optional", "email": "optional, where the human principal can be reached" },
  "urgency": "one of low | normal | high",
  "conversation_id": "string, optional"
}
```

Output:

```json
{
  "ticket_id": "string",
  "eta": "ISO 8601 duration, e.g. PT4H, or null",
  "status_url": "string",
  "receipt": { "id": "string", "url": "string" }
}
```

### 4.4 `status`

Input: `{ "ticket_id": "optional", "receipt_id": "optional" }`. Without arguments, returns the product's component status. With a ticket id, returns its state (`open`, `answered`, `closed`) and any human reply addressed to the agent. With a receipt id, returns the receipt under `receipt` (the fields of section 6), plus its state while an `act` awaits confirmation; never a human reply, since a receipt is meant to be shown and shared.

## 5. Principal verification

The **principal** is the human or organisation the agent works for. Vendors decide which levels they accept and declare them in `verification.levels`.

| Level | Proof carried in `principal.token` | Typical rights |
|---|---|---|
| 0 | none | `ask`, `status`, `escalate` without account data |
| 1 | the customer's own API key for the product; the support server validates it against `verification.introspection_url` (`POST`, body `{ "token": "..." }`, response `{ "active": true, "account_id": "...", "email": "..." }`) | `act` on that account, identified escalation |
| 2 | OAuth 2.1 access token obtained by the agent through the vendor's authorization server with scopes `support:read` and/or `support:act` | as level 1, scoped and revocable |
| 3 | HTTP Message Signature (RFC 9421) following the Web Bot Auth profile, with the agent's key published in a directory referenced by `verification.web_bot_auth.directory_url`, plus a level 1 or 2 token binding the principal | high-stakes actions, listing in the agent directory |

Servers must never grant more than the rights of the highest verified level and must log the level with every receipt.

## 6. Receipts

A receipt is the proof that a conversation happened and how it ended. It exists so that the human principal learns that their agent got help, and where.

Fields: `id`, `product`, `provider`, `tool`, `outcome` (`resolved`, `escalated`, `refused`, `failed`), `created_at`, `principal_level`, `summary` (one line, no secrets), `url`.

Retention: a server may purge a receipt once its retention period is over. A purged receipt keeps its other fields and its `url` keeps answering; its `summary` becomes a fixed retention notice, and the optional `purged_at` (ISO 8601) gives the date of the purge. `purged_at` is null or absent until then.

Delivery: always `inline` in tool outputs, and in the error body of a call that failed after the server accepted it (outcome `failed`, section 7); `email` to `principal.email` when given and the principal is verified (level 1 or above, section 5), never at level 0, where any agent could name any address; `webhook` when the vendor configured one. The public receipt page must not expose the transcript, only the summary and outcome.

## 7. Errors and limits

- HTTP servers use standard status codes; MCP servers return tool errors with `isError: true` and a JSON body `{ "code": "...", "message": "...", "retry_after": seconds }`. When the call was accepted before it failed (`internal`), the body may also carry `"receipt": { "id": "...", "url": "..." }`, a receipt with outcome `failed` (section 6).
- Codes: `rate_limited`, `verification_required`, `verification_failed`, `action_unknown`, `action_refused`, `index_not_ready`, `invalid_request`, `internal`.
- `invalid_request`: the input does not match the tool's schema (HTTP 400), or names an endpoint, ticket or receipt the server does not know (HTTP 404). Fix the request before retrying.
- Default rate limit if not declared: 60 requests per minute per agent identity.
- Card size: a card must not exceed 64 KiB. Validators may reject larger documents before parsing. URLs are limited to 2048 characters, `actions` to 64 entries, each action's `params` schema to 32 top-level properties.

## 8. Security considerations

- Treat every field an agent sends as untrusted text. Answers come from indexed sources, never from instructions embedded in the question.
- Actions are an allowlist. No natural-language fallback.
- Introspection and OAuth endpoints must be on the vendor's own origin, declared in the card, and validated with TLS.
- Receipts and public pages must not contain tokens, keys, or transcripts.
- Vendors should rotate the card's `updated` date whenever endpoints change; clients should re-fetch the card after any `verification_failed` error.

## 9. Relation to other standards

- **MCP** (Model Context Protocol): the support server is an ordinary MCP server over streamable HTTP; `tools/list` exposes the tools above. The card is complementary to the MCP server card at `/.well-known/mcp/server.json`, which describes the product's own tools, not its support.
- **llms.txt**: describes documentation for language models; the card describes where to get help when the documentation was not enough.
- **Web Bot Auth** (IETF): provides the level 3 identity proof.
- **OpenAPI**: `info.x-support-card` links an API description to its card.

## 10. Versioning

Cards declare `version`. Minor versions add optional fields only. Breaking changes bump the major version and the well-known path stays the same.

## 11. Changelog

- **0.2 (draft)**: optional `provider.notice` (section 3), one line in which a hosting provider says the card is not the vendor's own: an unofficial demo built from public documentation, or a product its vendor has not confirmed yet. An optional field only, cards keep `"version": "0.1"`; validators that predate it refuse a card carrying it, since the schema allows no unknown property, so only hosted cards that need it carry it.
- **0.2 (draft)**: added the `invalid_request` error code, section 7, for inputs that fail validation and for unknown endpoints, tickets or receipts; servers already returned it. An addition only (section 10): no change to the card format, cards keep `"version": "0.1"`.
- **0.2 (draft)**: the `receipt` field of the `ask` and `escalate` outputs (sections 4.1 and 4.3) may be absent while a server does not implement receipts (section 6). This only relaxes the output contract.
- **0.2 (draft)**: receipts, from the first implementation (section 6). `ask` accepts `principal.email`, as `escalate` already did, so the receipt of an answered question can reach the human (sections 4.1 and 6). An error body may carry the `failed` receipt of a call the server accepted (sections 6 and 7). `status` with a `receipt_id` returns the receipt's fields under `receipt` and never a human reply, which stays behind the `ticket_id`: a receipt is meant to be shared (section 4.4). Optional fields only: no change to the card format, cards keep `"version": "0.1"`.
- **0.2 (draft)**: email delivery of a receipt requires a verified principal, level 1 or above (section 6). At level 0 the receipt is inline only: an unverified `principal.email` could be anyone's, and mailing it would turn the support server into a relay. Servers do not store an unverified address for receipts.
- **0.2 (draft)**: receipts may be purged once a retention period is over (section 6). The receipt and its page remain, the `summary` becomes a retention notice, and the optional `purged_at` field gives the date. An optional field only: no change to the card format, cards keep `"version": "0.1"`.
- **0.1, 7 September 2026**: initial draft. Schema fix the same day: the conditional rules (`act` requires `actions`, level 1 requires `introspection_url`, level 2 `oauth`, level 3 `web_bot_auth`) now only apply when the triggering field is present, so a card missing `tools` or `levels` reports the missing field rather than the derived requirement. Also added upper bounds (URL and email length, `actions` count, `params` size) and a 64 KiB document limit, section 7. No change to the card format.

## 12. Reference implementation

`@wellknown/spec` is the reference implementation: this schema, generated TypeScript types, a `validateSupportCard` function and a `wellknown-spec validate` command. It lives in `packages/spec` of the Wellknown repository and is not published on npm yet. Until it is, the validator at [wellknown.support/validate](https://wellknown.support/validate) runs the same code on any card, pasted or fetched from its URL, with the discovery checks of section 2.
