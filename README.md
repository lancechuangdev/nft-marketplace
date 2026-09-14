# NFT Marketplace

A full-stack NFT marketplace built around an EVM order book, event indexers, and a Next.js frontend.

## Projects

- `EasySwapContract` — upgradeable escrow-based ERC-721 marketplace contracts.
- `NFTOrderBook` — standalone Hardhat order book with escrowed and EIP-712 signed orders.
- `EasySwapSync` — Go service that synchronizes EasySwap events to MySQL and Redis.
- `NFTOrderBookIndexer` — Go indexer for order, activity, item, and collection state.
- `nft-market-fe` — Next.js marketplace UI.

Each directory contains more detailed setup and architecture notes.

## Prerequisites

- Go versions declared by each module (`1.21` and `1.25.6`)
- Node.js with npm (and pnpm for the frontend)
- MySQL and Redis for the sync/indexer services
- An EVM RPC endpoint for blockchain-backed services

## Quick start

Install and test the Solidity projects:

```bash
cd EasySwapContract && npm install && npx hardhat test
cd ../NFTOrderBook && npm install && npm test
```

Run the Go tests from the repository root:

```bash
(cd EasySwapSync && go test ./...)
(cd NFTOrderBookIndexer && go test ./...)
```

Start the frontend:

```bash
cd nft-market-fe
pnpm install
pnpm dev
```

Copy the relevant config template before running a Go service; see its local README for database migrations and runtime commands.
