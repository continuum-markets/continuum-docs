# Continuum Docs

Developer documentation for [Continuum.Markets](https://continuum.markets) — 24/7 synthetic exposure to real-world assets on Solana.

This repo is the source for [docs.continuum.markets](https://docs.continuum.markets). Built with [Mintlify](https://mintlify.com).

## Local preview

```bash
npm i -g mint
mint dev
```

The site reloads on save. Default port: `http://localhost:3000`.

## Structure

```
continuum-docs/
├── docs.json              # Mintlify navigation + theme
├── get-started/           # Introduction, quickstart, devnet info
├── concepts/              # Mental model: NAV, paired tokens, oracle, keeper, CLP
├── flows/                 # Core user flows (mint, redeem, trade, read state)
├── programs/              # Per-program reference (PDAs, instructions, errors)
├── build/                 # SDK setup, examples, CPI, composability
├── markets/               # Live markets, listing flow, asset roadmap
├── keeper/                # Keeper / market-making overview
├── reference/             # Glossary, error catalogue, addresses, FAQ
└── snippets/              # Reusable MDX fragments
```

## Conventions

- Every page has frontmatter: `title`, `description`. Mintlify renders the title as `<h1>`.
- Solana addresses are in inline code; the canonical devnet address registry is in [`reference/addresses.mdx`](./reference/addresses.mdx).
- "cUSDC" refers to the **devnet** USDC mint at `B1c5xBYkp7AAemYhcu4VuH4CU4sPJDDuG2iuv6ts38uE`. Mainnet uses real USDC.
- Code examples use TypeScript with `@coral-xyz/anchor` v0.32 unless otherwise noted.

## Contributing

PRs welcome. Keep examples copy-pasteable — assume the reader has a fresh Solana CLI install and a wallet on devnet.

For doc-system questions (Mintlify config, navigation, theming) see the [Mintlify docs](https://mintlify.com/docs).
