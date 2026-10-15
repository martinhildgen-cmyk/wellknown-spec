# Contributing

The specification is a draft. Feedback from people who publish MCP servers, APIs, or build agents is what turns 0.1 into 0.2.

## How to propose a change

1. [Open an issue](https://github.com/martinhildgen-cmyk/wellknown-spec/issues) describing the situation the spec does not cover: what the agent tried, what the vendor could not express in the card, what broke. Real cases beat hypotheticals.
2. For a concrete proposal, open a pull request that changes, in this order: `support-card.md` (with a line in the changelog, section 11), `support-card.schema.json`, and the examples. Every example must validate against the schema: paste it into [the validator](https://wellknown.support/validate), or run any JSON Schema draft 2020-12 validator against `support-card.schema.json`.
3. Minor versions only add optional fields. Anything that breaks an existing valid card is a major version and needs a migration note.

## Principles that proposals are judged against

- **The agent is a client, not a threat model to ignore.** Everything an agent sends is untrusted text; nothing in the card may require a server to execute instructions found in a question.
- **Actions are an allowlist.** `act` executes only what the card declares. No natural-language fallback.
- **Vendor-neutral.** Nothing in the spec may require a specific host, including Wellknown.
- **Small.** A minimal card fits on one screen. A field that most vendors will leave empty probably does not belong at the top level.
- **Receipts carry no secrets.** Public pages and receipts never contain tokens, keys or transcripts.

## Open questions for 0.2

These are the issues we want opinions on. Each becomes a GitHub issue at publication.

1. **Multiple products per domain.** A company with several APIs under one domain: one card with a `products` array, or one card per subpath (`/.well-known/support.json?product=`)? Current draft: one card per domain, one product.
2. **`status` payload.** The component-status shape is unspecified. Adopt the statuspage.io summary shape, or a minimal `{ components: [{ name, status }] }`?
3. **Escalation reply channel.** When the human answers, how does the agent learn? Polling `status` is specified; should the card also allow a callback URL the agent registers per conversation?
4. **Level 3 details.** Web Bot Auth is still an IETF draft. Which signature components are required, and how is the principal bound to the agent key?
5. **Language negotiation.** `policies.languages` declares what the vendor supports; should `ask` accept an explicit `language` hint, or always follow the question?
6. **Discovery through DNS.** A `_support` TXT record as an alternative to the well-known path for products that cannot serve files on their apex domain?

## Licence

By contributing you agree that your contribution is released under CC0 1.0, like the rest of this repository.
