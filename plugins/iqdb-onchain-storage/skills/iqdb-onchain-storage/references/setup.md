# Environment Setup

## Prerequisites

- **Node.js 18+** — `node --version`
- **Solana CLI** — `solana --version`
- **A Solana wallet** — Keypair JSON file

## Install Solana CLI

```bash
# macOS/Linux
sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"

# Windows — download installer and run with admin privileges
# https://release.anza.xyz/stable/solana-install-init-x86_64-pc-windows-msvc.exe
```

Add to PATH: `export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"`

## Configure for Devnet

```bash
solana config set --url https://api.devnet.solana.com
solana-keygen new --no-bip39-passphrase -o ~/.config/solana/id.json
solana config set --keypair ~/.config/solana/id.json
solana airdrop 2
```

If airdrop fails (rate-limited), use the web faucet at faucet.solana.com.

## Install Dependencies

```bash
npm install @iqlabsteam/iqdb @coral-xyz/anchor @solana/web3.js hanlock
```

**Important:** Set `"type": "commonjs"` in package.json (or use `.cjs` file extension). The IQDB SDK uses dynamic `require()` internally and does not support ESM.

## Project Setup

```javascript
// setup.cjs
// Set NETWORK_URL before importing to ensure reads use the same RPC as writes
const RPC = process.env.ANCHOR_PROVIDER_URL || 'https://api.devnet.solana.com';
process.env.NETWORK_URL = RPC;

const { createIQDB } = require('@iqlabsteam/iqdb');

const iqdb = createIQDB();

async function init() {
  await iqdb.ensureRoot();
  console.log('IQDB ready.');
}

module.exports = { iqdb, init };
```

### Required Environment Variables

```bash
ANCHOR_WALLET=/path/to/keypair.json        # Required — Solana keypair for signing
ANCHOR_PROVIDER_URL=https://api.devnet.solana.com  # Required — RPC endpoint
```

**RPC Note:** The standard `api.devnet.solana.com` aggressively rate-limits. The SDK includes a built-in Helius devnet RPC as fallback. For production use, get a dedicated RPC from Helius, Alchemy, or QuickNode.

## Mainnet Checklist

Before switching to mainnet:

1. Change RPC URL to mainnet-beta endpoint
2. Use a funded wallet (real SOL)
3. Test all operations on devnet first
4. Budget for rent costs (~0.012 SOL per 100 tables)
5. Use a paid RPC provider — free endpoints will rate-limit under load
6. Row data must stay under ~100 bytes per write
