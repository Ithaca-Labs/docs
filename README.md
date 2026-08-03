# openx402 documentation

Mintlify documentation for [openx402](https://github.com/Ithaca-Labs/openx402).

## Local development

Install the Mintlify CLI and start the preview from this directory:

```bash
npm install -g mint
mint dev
```

## Validate changes

```bash
mint validate
mint broken-links --check-anchors --check-redirects
mint a11y
```

Site configuration lives in `docs.json`. Every published page is listed in its `navigation` section.
