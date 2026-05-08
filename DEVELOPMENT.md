# Development Guide

## Quick Start

```bash
# Clone repo
git clone https://github.com/your-org/continuum.git
cd continuum

# Install dependencies
pnpm install
cd frontend && pnpm install && cd ..

# Start local validator
solana-test-validator

# Build and deploy programs (new terminal)
cd programs
anchor build
anchor deploy

# Start frontend (new terminal)
cd frontend
cp .env.example .env.local
pnpm dev
```

Visit http://localhost:3000

## Project Structure

```
continuum/
├── frontend/              # Next.js 15 app
│   ├── app/              # Pages (App Router) — pools, portfolio, mint UI
│   ├── components/       # React components
│   ├── hooks/            # useDlmmPool, useProgram, etc.
│   ├── lib/              # Utils, constants, Solana client, IDLs
│   └── types/            # TypeScript types
├── keeper/               # Rust keeper bot (peg-enforcement loop)
│   ├── src/              # Main source — arb, dlmm, liquidity_seeder, oracle, etc.
│   └── key/              # CLP authority keypair
├── programs/             # Anchor Solana programs
│   ├── mint-redeem/      # Paired mint/redeem at NAV + keeper-only single-side ixns
│   ├── oracle/           # Pyth/Hermes + TWAP + risk state
│   ├── clp/              # Capital & Liquidity Provider (Meteora DLMM proxy)
│   ├── governance/       # Scaffolded — listing/parameter votes
│   ├── faucet/           # Devnet cUSDC drip
│   └── registry/         # Market registry
├── scripts/              # Operator TypeScript scripts (bun run)
├── docs/                 # Architecture, API, Operations, Deployment guides
└── tests/                # Anchor integration tests
```

**Programs that don't exist:** older drafts (e.g. `FINAL_STATUS.md`, `IMPLEMENTATION_STATUS.md`) referenced separate `funding`, `amm`, `margin`, and `insurance` programs. Those were a design pivot that was never built — the live system is paired-token + Meteora DLMM only, with no funding rate, no in-house AMM, no margin, no separate insurance fund (the global CLP vault absorbs losses).

## Development Workflow

### 1. Solana Programs (Rust/Anchor)

**Build**:
```bash
cd programs
anchor build
```

**Test**:
```bash
anchor test
```

**Deploy Locally**:
```bash
# Terminal 1: Start validator
solana-test-validator

# Terminal 2: Deploy
anchor deploy
```

**Update Program ID**:
After first build, update program IDs:
```bash
anchor keys list
```

Copy IDs to:
- `Anchor.toml` (programs.localnet)
- `programs/*/src/lib.rs` (declare_id!)
- `frontend/.env.local`

### 2. Frontend (Next.js)

**Dev Server**:
```bash
cd frontend
pnpm dev
```

**Type Check**:
```bash
pnpm type-check
```

**Lint**:
```bash
pnpm lint
```

**Build**:
```bash
pnpm build
```

### 3. Testing

**Unit Tests (Rust)**:
```bash
cd programs/mint-redeem
cargo test
```

**Integration Tests (TypeScript)**:
```bash
cd programs
anchor test
```

**Frontend Tests** (add later):
```bash
cd frontend
pnpm test
```

## Common Tasks

### Add a New Market

1. **Create SPL Tokens**:
```bash
# Create Long mint
spl-token create-token

# Create Short mint
spl-token create-token

# Create vaults
spl-token create-account <LONG_MINT>
spl-token create-account <SHORT_MINT>
```

2. **Initialize Market**:
```typescript
// In a script or frontend
await program.methods
  .initializeMarket(
    "TSLA",              // symbol
    { pyth: {} },        // oracle type
    initialLPrice,
    initialSPrice,
    5,                   // 0.05% mint fee
    5                    // 0.05% redeem fee
  )
  .accounts({
    market,
    authority,
    longMint,
    shortMint,
    collateralMint,
    collateralVault,
    oracle,
  })
  .rpc();
```

3. **Update Frontend Constants**:
```typescript
// frontend/lib/constants.ts
export const ASSETS: Asset[] = [
  // ...existing assets
  {
    id: "tsla",
    name: "Tesla",
    symbol: "TSLA",
    category: AssetCategory.EQUITY,
    description: "Electric vehicles",
    oracleAddress: "PYTH_TSLA_ADDRESS",
    longMint: "YOUR_LONG_MINT",
    shortMint: "YOUR_SHORT_MINT",
    isActive: true,
  },
];
```

### Debug Program Logs

```bash
# Terminal 1: Start validator with logs
solana-test-validator -l

# Terminal 2: Watch logs
solana logs

# Terminal 3: Run transactions
anchor test
```

### Reset Local State

```bash
# Stop validator
# Delete test ledger
rm -rf test-ledger/

# Restart validator
solana-test-validator
```

## Code Style

### Rust
```rust
// Use explicit error handling
require!(amount > 0, ErrorCode::InvalidAmount);

// Document public functions
/// Mint Long or Short tokens by depositing USDC
pub fn mint_tokens(ctx: Context<MintTokens>, amount: u64) -> Result<()> {
    // ...
}

// Use checked math
let result = a.checked_mul(b).ok_or(ErrorCode::Overflow)?;
```

### TypeScript
```typescript
// Use explicit types
const mintTokens = async (
  amount: number,
  tokenType: "long" | "short"
): Promise<string> => {
  // ...
};

// Handle errors gracefully
try {
  await mintTokens(100, "long");
} catch (err) {
  console.error("Mint failed:", err);
  toast.error("Transaction failed");
}
```

### React
```tsx
// Prefer function components
export function TradingInterface() {
  const [amount, setAmount] = useState("");
  
  return (
    <Card>
      {/* ... */}
    </Card>
  );
}

// Use custom hooks for Solana interactions
const { mint, redeem } = useMintRedeem();
```

## Environment Variables

### Frontend (.env.local)
```bash
# Network
NEXT_PUBLIC_SOLANA_NETWORK=devnet
NEXT_PUBLIC_SOLANA_RPC_URL=https://api.devnet.solana.com

# Program IDs (from anchor keys list)
NEXT_PUBLIC_MINT_REDEEM_PROGRAM_ID=...
NEXT_PUBLIC_FUNDING_PROGRAM_ID=...
NEXT_PUBLIC_AMM_PROGRAM_ID=...
NEXT_PUBLIC_MARGIN_PROGRAM_ID=...
NEXT_PUBLIC_INSURANCE_PROGRAM_ID=...
NEXT_PUBLIC_ORACLE_PROGRAM_ID=...
NEXT_PUBLIC_GOVERNANCE_PROGRAM_ID=...

# Tokens
NEXT_PUBLIC_USDC_MINT=4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU

# Services
REDIS_URL=redis://localhost:6379
FILEBASE_API_KEY=...
FILEBASE_SECRET_KEY=...
```

## Debugging

### Solana Programs

**Enable Detailed Logs**:
```rust
#[cfg(feature = "debug")]
msg!("Current price: {}, NAV: {}", oracle_price, nav);
```

**Use solana-test-validator with logging**:
```bash
solana-test-validator --log
```

**Check account data**:
```bash
solana account <PUBKEY> --output json
```

### Frontend

**Check Wallet Connection**:
```typescript
import { useWallet } from "@solana/wallet-adapter-react";

const { connected, publicKey } = useWallet();
console.log("Wallet:", connected, publicKey?.toString());
```

**Inspect Transactions**:
```typescript
const tx = await connection.sendTransaction(transaction, [signer]);
console.log("Transaction signature:", tx);
console.log("Explorer:", `https://explorer.solana.com/tx/${tx}?cluster=devnet`);
```

## Performance Tips

### Solana Programs
- Use PDAs (Program Derived Addresses) to minimize account creation
- Batch operations when possible
- Avoid unnecessary account deserializations

### Frontend
- Use React Query for caching on-chain data
- Debounce user input (especially for price updates)
- Lazy load heavy components
- Use Next.js Image component for optimized images

## Common Errors

### "Program failed to complete"
- Check compute budget (may need to increase)
- Verify all required accounts are passed
- Check account ownership and signatures

### "Custom program error: 0x0"
- Look at program logs for the actual error
- Often a require!() check failed
- Use anchor error codes for better debugging

### "Transaction too large"
- Break into multiple transactions
- Use lookup tables for frequently used accounts
- Reduce number of accounts per instruction

## Resources

- [Anchor Docs](https://www.anchor-lang.com/)
- [Solana Cookbook](https://solanacookbook.com/)
- [Pyth Docs](https://docs.pyth.network/)
- [Next.js Docs](https://nextjs.org/docs)
- [Shadcn/UI](https://ui.shadcn.com/)

## Getting Help

- **Discord**: [discord.gg/continuum](https://discord.gg/continuum) - #dev-help channel
- **GitHub Discussions**: Ask questions, share ideas
- **Stack Overflow**: Tag `solana` + `anchor-solana`





