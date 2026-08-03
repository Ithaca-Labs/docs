# Product brief

## Description

openx402 is a permissively licensed, self-hostable x402 v2 facilitator for Stellar. It gives developers a complete path to sell and buy paid HTTP or MCP resources, sponsor Stellar network fees without custodying seller funds, publish Bazaar metadata, discover cataloged resources, and optionally expose discovery and guarded paid execution to agents.

## Primary audience

The primary readers are TypeScript developers integrating paid resources and infrastructure operators deploying the facilitator. Secondary readers are buyer-client developers and agent builders using Bazaar discovery or the MCP server.

## Jobs to be done

- Complete a paid request on Stellar testnet with the hosted facilitator.
- Add x402 payment protection and Bazaar metadata to an HTTP or MCP seller.
- Build a buyer that validates, signs, retries, and verifies settlement safely.
- Browse and search the Bazaar catalog directly or through MCP.
- Self-host and operate the facilitator, PostgreSQL, and optional MCP server.
- Understand the protocol boundaries, security invariants, and current release status.

## Motivation

openx402 exists so teams can control their infrastructure and discovery quality while payment correctness remains fixed. Its core deployment needs one facilitator process and one PostgreSQL database; MCP is optional. The facilitator is non-custodial, supports exact Stellar settlement, includes a proposed Stellar `upto` scheme, and treats seller metadata as untrusted input.

Assumption to verify: the public docs should prioritize the hosted Stellar testnet path, then self-hosting. Pubnet remains an operator-only, gated path until the release requirements documented by the project are complete.
