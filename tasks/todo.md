# Todo

## Plan

- [x] Read the Mintlify skill and required product, configuration, navigation, component, and CLI references.
- [x] Audit the copied openx402 docs, starter site, source branding, and repository facts.
- [x] Define the product brief and documentation information architecture.
- [x] Replace starter content and organize all copied guides as production MDX pages.
- [x] Apply openx402 branding, navigation, metadata, redirects, and repository documentation.
- [x] Fix standalone links and MDX compatibility issues without changing technical meaning.
- [x] Add dedicated API, configuration, seller SDK, embedded facilitator, protocol, search, and evaluation pages.
- [x] Add analytics API, MCP tool reference, storage/recovery, CLI, catalog lifecycle, and SDK export reference pages.
- [x] Document the full evaluation tracks, handwritten-v2 design, calibration, isolation, metrics, and release gates.
- [x] Publish two development evaluation tables: embedding quality and embedding latency.
- [x] Add the dedicated proposed Upto guide with contract, hooks, smart-account, server, and evidence paths.
- [x] Embed the four checked-in evaluation charts with captions and exploratory-status notes.
- [x] Link configuration and self-hosting from the quickstart and operator paths.
- [x] Add a top-level OpenAPI API Reference tab with interactive Try it support.
- [x] Reorder navigation around getting started, build, discover, protocol, operate, and reference tasks.
- [x] Validate configuration, pages, navigation, links, and basic MDX structure locally.
- [x] Review the final diff for completeness and unrelated changes.

## Files likely touched

- `.mintlify/product-brief.md`
- `docs.json`
- `index.mdx`
- `quickstart.mdx`
- `guides/*.mdx`
- `concepts/*.mdx`
- `operations/*.mdx`
- `logo/*.svg`
- `favicon.svg`
- `README.md`
- `tasks/todo.md`

## Verification

- [ ] `mint validate` (run in the Mintlify environment; CLI is unavailable here)
- [ ] `mint broken-links --check-anchors --check-external --check-redirects` (run in the Mintlify environment)
- [ ] `mint a11y` (run in the Mintlify environment)
- [x] JSON, navigation coverage, frontmatter, fence, internal-link, and evaluation-image checks
- [ ] Local preview returned HTTP 200 for every navigation page (requires Mintlify CLI)
- [x] Git diff and status review

## Validation note

The local `mint` executable is not installed in this workspace, so strict
Mintlify validation, accessibility, external-link checks, and the preview
server still need to run in the Mintlify environment. Repository-local checks
confirm all 28 navigation pages exist, have frontmatter, have balanced code
fences, have no missing site-route links, and reference four existing
evaluation images.

## Product boundaries documented

- Stellar `exact` is the shipped testnet path.
- Stellar `upto` is implemented with testnet evidence but remains upstream- and
  audit-gated.
- Pubnet is packaged and fail-closed, not claimed as live evidence.
- Search bakeoff values are development evidence; release gates remain separate.

## Review

### Changed

- Replaced starter pages and configuration with an openx402 documentation site.
- Organized the guides into quickstart, build, discovery, protocol, operations, and reference routes.
- Added route-level analytics, MCP tool schemas, storage/recovery, maintenance CLI, catalog lifecycle, SDK export, and full evaluation workflow references.
- Added a dedicated proposed Upto navigation group and a build guide covering the temporary client/server package, Soroban contract algorithm, settlement hook ABI, OZ context rules, configuration, and evidence boundary.
- Added evaluation chart assets and embedded them in the evaluation report.
- Added product context, frontmatter, Mintlify components, redirects, accessible brand colors, and openx402 assets.
- Replaced repository-relative links with valid site routes or public source links.

### Verified

- Navigation targets, redirects, frontmatter, internal links, and basic MDX checks passed locally.
- Mintlify strict build, external links, accessibility, and local preview remain unverified until the CLI runs.

### Risks

- Pubnet remains intentionally documented as gated and not production-ready.

### Follow-ups

- Commit and push only when requested.
