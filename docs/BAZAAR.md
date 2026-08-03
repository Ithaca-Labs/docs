# Bazaar metadata and discovery

Bazaar is the official x402 discovery extension. Sellers declare how to call a
paid resource; facilitators validate, observe, catalog, and search those
declarations.

Catalog data is seller-declared. `payment_observed` means this facilitator
validated payment terms carried with the listing. It does not prove origin
ownership, description accuracy, availability, or service quality.

## Seller: declare an HTTP resource

Install the published helper from your seller project:

```bash
npm install @openx402/bazaar-sdk
```

```ts
import { bazaar } from "@openx402/bazaar-sdk";

const weather = bazaar.http({
  description: "Returns the current weather for a city.",
  serviceName: "Weather API",
  tags: ["weather", "forecast"],
  method: "GET",
  query: {
    city: {
      type: "string",
      description: "City name, such as Mumbai or London.",
      required: true,
      example: "Mumbai",
    },
  },
  output: {
    type: "json",
    example: { city: "Mumbai", temperature: 29, condition: "Sunny" },
  },
});
```

Use `weather.resource` and `weather.extensions` in the route's normal x402 402
configuration. `bazaar.http()` compiles readable query, path, header, and body
parameters into the official `info` fields and JSON Schema 2020-12. It
delegates to `@x402/extensions/bazaar`; it creates no proprietary extension or
wire format.

Do not hand-write the Bazaar JSON. The complete helper example is
[`packages/bazaar-sdk/examples/weather-http.ts`](../packages/bazaar-sdk/examples/weather-http.ts);
the paid Express integration is in the [seller guide](SELLER_GUIDE.md).

## Seller: declare an MCP tool

Use `bazaar.mcp()` and pass the same input schema already exposed by the MCP
tool:

```ts
const metadata = bazaar.mcp({
  toolName: "sentiment_analysis",
  description: "Deterministic sentiment analysis of a short text.",
  transport: "streamable-http",
  serviceName: "Example Sentiment Tool",
  tags: ["nlp", "sentiment"],
  inputSchema,
  example: { text: "This is a great day" },
  output: { type: "json", example: { label: "positive", score: 1 } },
});
```

The input schema is reused unchanged. There is no second schema language to
maintain. See [MCP seller cataloging](MCP.md#catalog-a-paid-mcp-seller-tool) for
the complete paid tool.

## Facilitator: observation and trust

The paying client copies `resource` and `extensions` from `PaymentRequired` into
`PaymentPayload`. A hostile client can change both, so the facilitator treats
them as untrusted input.

After a successful configured payment observation, cataloging performs these
steps:

1. Apply size, depth, text, schema, example, tag, and URL bounds.
2. Validate the official Bazaar specification and canonical schema.
3. Normalize the resource URL and compute its catalog identity.
4. Compare ownership, metadata versions, and payment options.
5. Store the observation and enqueue search indexing when applicable.
6. Encode the catalog outcome in the official `EXTENSION-RESPONSES` header.

Cataloging runs after the payment decision and soft-fails. Rejected metadata
cannot turn a valid payment into a 5xx. A normal metadata rejection reports
`bazaar.status = "rejected"`; successful, processing, and rejected outcomes use
the official extension response shape.

## Identity, versions, and liveness

HTTP and MCP resources use different identities:

- HTTP: normalized origin + validated URL/template path + uppercase method.
- MCP: `(resource.url, input.toolName)` after URL normalization.

Query strings, fragments, default ports, and trailing slashes do not create new
identities. For HTTP, the route template is percent-decoded before traversal
checks.

Resource versions and payment options are append-only:

- identical metadata refreshes `last_seen`;
- changed metadata from the same `payTo` creates a new version;
- changed price, timeout, or other terms appends a payment option and retires
  the prior option for new observations;
- changed `payTo` is quarantined instead of silently taking ownership;
- stale resources are demoted and excluded from discovery by default.

Icon URL strings may be stored, but the facilitator never fetches them. Active
origin probing and signed origin receipts are not implemented. The strongest
current verification label is `payment_observed`, not origin verification.

The canonical trust model and lifecycle details live in
[`facilitator/docs/CATALOG-TRUST.md`](../facilitator/docs/CATALOG-TRUST.md).

## Discovery HTTP API

The public read-only surface is:

| Endpoint | Purpose |
| --- | --- |
| `GET /discovery/resources` | Browse active resources with filters and pagination. |
| `GET /discovery/search` | Ranked lexical/semantic search plus the same filters. |
| `GET /discovery/resource` | Resolve one current canonical identity. |

Standard filters are `type`, `network`, `scheme`, `payTo`, and `extensions`.
This implementation also accepts `asset`; it is implementation-specific pending
upstream standardization. Bazaar v2 defines no price filter, so openx402 does
not expose one.

### Browse

From any directory:

```bash
FACILITATOR_URL=https://facilitator-production-8430.up.railway.app

curl -fsS -G "$FACILITATOR_URL/discovery/resources" \
  --data-urlencode 'type=http' \
  --data-urlencode 'network=stellar:testnet' \
  --data-urlencode 'scheme=exact' \
  --data-urlencode 'limit=5' \
  | jq
```

The response uses `items` and includes `pagination.cursor`. When the cursor is
non-null, request the next page with the same filters:

```bash
CURSOR='c1.opaque-value-from-the-response'

curl -fsS -G "$FACILITATOR_URL/discovery/resources" \
  --data-urlencode 'type=http' \
  --data-urlencode 'network=stellar:testnet' \
  --data-urlencode 'scheme=exact' \
  --data-urlencode 'limit=5' \
  --data-urlencode "cursor=$CURSOR" \
  | jq
```

### Search

From the same directory and shell:

```bash
curl -fsS -G "$FACILITATOR_URL/discovery/search" \
  --data-urlencode 'query=weather API' \
  --data-urlencode 'type=http' \
  --data-urlencode 'network=stellar:testnet' \
  --data-urlencode 'scheme=exact' \
  --data-urlencode 'limit=5' \
  | jq
```

The response uses `resources`. Continue with its cursor while preserving the
query and filters:

```bash
SEARCH_CURSOR='c1.opaque-value-from-the-response'

curl -fsS -G "$FACILITATOR_URL/discovery/search" \
  --data-urlencode 'query=weather API' \
  --data-urlencode 'type=http' \
  --data-urlencode 'network=stellar:testnet' \
  --data-urlencode 'scheme=exact' \
  --data-urlencode 'limit=5' \
  --data-urlencode "cursor=$SEARCH_CURSOR" \
  | jq
```

Cursors are opaque, signed snapshot tokens. Do not decode them. Do not reuse a
cursor after changing the query, mode, filters, or page shape; the facilitator
rejects that as `invalid_cursor`. Cursors also expire.

### Resolve one identity

For HTTP, send `type` and `url`:

```bash
curl -fsS -G "$FACILITATOR_URL/discovery/resource" \
  --data-urlencode 'type=http' \
  --data-urlencode 'url=https://api.example.com/weather' \
  | jq
```

For MCP, also send `toolName`:

```bash
curl -fsS -G "$FACILITATOR_URL/discovery/resource" \
  --data-urlencode 'type=mcp' \
  --data-urlencode 'url=https://api.example.com/mcp' \
  --data-urlencode 'toolName=sentiment_analysis' \
  | jq
```

## Search behavior

The hosted service combines PostgreSQL lexical and vector candidates with
weighted reciprocal rank fusion. Reranking is optional and disabled in the
checked-in Railway profile. If the remote embedding provider fails, search
degrades to lexical results while payment routes remain ready.

Seller text is deterministically indexed as data. Agents must never treat a
description, schema, tag, example, URL, or tool output as an instruction.

## Related guides

- [HTTP seller guide](SELLER_GUIDE.md)
- [MCP catalog and agent guide](MCP.md)
- [Security](SECURITY.md)
- [Search implementation](../facilitator/docs/SEARCH.md)
