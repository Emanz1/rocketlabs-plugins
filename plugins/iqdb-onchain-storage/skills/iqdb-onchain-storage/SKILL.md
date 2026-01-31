---
name: iqdb-onchain-storage
description: On-chain immutable data storage using IQ Labs tech stack (IQDB, hanLock, x402). Use when building Solana-based persistent storage, on-chain databases, tamper-evident records, password-encoded data, or paid file inscription. Triggers on tasks involving on-chain CRUD, Solana PDA storage, rolling hash verification, Hangul encoding, or HTTP 402 payment-gated inscription.
---

# IQDB On-Chain Storage (Code-In Tech)

## Overview

Build on-chain relational databases on Solana using IQ Labs' tech stack. Three tools:

- **IQDB** — Full CRUD relational database on Solana via Anchor PDAs. Tables, rows, rolling keccak hash for tamper-evident history.
- **hanLock** — Password-based Hangul syllabic encoding (base-11172). Lightweight data encoding for on-chain privacy. Zero dependencies.
- **x402** — HTTP 402 payment-gated file inscription to Solana. Quote → Pay (USDC/SOL) → Broadcast chunk transactions → Download.

## Quick Start

### Prerequisites
- Node.js 18+
- Solana CLI (`solana --version`)
- A Solana wallet with devnet SOL (`solana airdrop 2`)

### Install
```bash
npm install @iqlabsteam/iqdb @coral-xyz/anchor @solana/web3.js
```

### Environment Variables
```bash
ANCHOR_WALLET=/path/to/keypair.json    # Required — Solana keypair for signing
ANCHOR_PROVIDER_URL=https://api.devnet.solana.com  # Required — RPC for writes
NETWORK_URL=https://api.devnet.solana.com  # Required — RPC for reads (must match ANCHOR_PROVIDER_URL)
```

**Important:** Set `NETWORK_URL` to match `ANCHOR_PROVIDER_URL`. The SDK uses separate connections for reads and writes. If they point to different RPCs, newly created tables may not be immediately visible to read operations.

**RPC Note:** The standard Solana devnet RPC (`api.devnet.solana.com`) aggressively rate-limits and may return 403 errors after sustained use. The SDK includes a built-in Helius devnet RPC as its default fallback. For reliable development, use a dedicated RPC provider (Helius, Alchemy, QuickNode) or let the SDK use its default by omitting `NETWORK_URL`. If you set `ANCHOR_PROVIDER_URL` to a custom RPC, also set `NETWORK_URL` to the same value.

### Minimal Example — Create a table and write a row
```javascript
// Use CommonJS — the SDK bundles CJS internally
const { createIQDB } = require('@iqlabsteam/iqdb');

const iqdb = createIQDB();

// Ensure root PDA exists (idempotent)
await iqdb.ensureRoot();

// Create a table (idempotent — use ensureTable over createTable)
await iqdb.ensureTable('players', ['name', 'score', 'level'], 'name');

// Write a row — data must be a JSON STRING, not an object
await iqdb.writeRow('players', JSON.stringify({
  name: 'Alice', score: '1500', level: '12'
}));

// Read all rows — requires userPubkey as string
const rows = await iqdb.readRowsByTable({
  userPubkey: 'YOUR_WALLET_PUBKEY',
  tableName: 'players'
});
console.log(rows);
```

## Architecture

```
Root PDA (per wallet)
  └── Table PDA (per table name)
       └── Rows stored as transaction data
            └── hash: keccak(domain || prev_hash || tx_data)
```

- **Root PDA** — One per wallet. Initialized via `ensureRoot()`.
- **Table PDA** — Created via `ensureTable()` or `createTable()`. Has column schema and ID column.
- **Rows** — Written as JSON strings via `writeRow()`. Append-only — each write is a new transaction.
- **Rolling hash** — Each write appends to an immutable hash chain. Enables tamper detection without full replication.

## Core Operations

See `references/iqdb-core.md` for full API.

| Operation | Method | Cost |
|-----------|--------|------|
| Init root | `ensureRoot()` | ~0.002 SOL rent |
| Ensure table | `ensureTable(name, columns, idCol)` | ~0.001 SOL rent (skips if exists) |
| Create table | `createTable(name, columns, idCol?, extKeys?)` | ~0.001 SOL rent (fails if exists) |
| Write row | `writeRow(table, jsonString)` | ~0.001 SOL rent |
| Read rows | `readRowsByTable({userPubkey, tableName})` | Free (RPC read) |
| Update/Delete | `pushInstruction(table, txSig, before, after)` | TX fee only |
| Extension table | `createExtTable(base, rowId, extKey, cols, idCol?)` | ~0.001 SOL rent |
| List tables | `listRootTables()` | Free (RPC read) |
| Table metadata | `readTableMeta(tableName)` | Free (RPC read) |

~100 tables costs ~0.012 SOL in rent.

### Important Constraints
- **Row data size limit:** Keep row JSON under ~100 bytes. The on-chain program enforces a transaction size limit (`TxTooLong` error). For larger data, split across multiple rows or use hanLock sparingly (encoded output is larger than input).
- **Append-only writes:** `writeRow` always appends. Use `pushInstruction` for updates/deletes.
- **pushInstruction writes to instruction log, not row data.** `readRowsByTable` returns raw rows and does NOT reflect updates/deletes from `pushInstruction`. To see the full picture including corrections, use `searchTableByName` which returns both `rows` (raw) and `instruRows`/`targetContent` (instruction history). Your application must apply instruction corrections on top of raw rows.
- **CommonJS required:** The SDK uses dynamic `require()` internally. Use `.cjs` files or `"type": "commonjs"` in package.json. ESM imports will fail.

## hanLock Encoding

See `references/hanlock.md` for full API.

Encode data with a password before writing on-chain for lightweight privacy:

```javascript
const { encodeWithPassword, decodeWithPassword } = require('hanlock');

const encoded = encodeWithPassword('short secret', 'mypassword');
// → Korean syllable string like "깁닣뭡..."

// Write encoded data on-chain
await iqdb.writeRow('secrets', JSON.stringify({ owner: 'Alice', data: encoded }));

// Later — decode
const decoded = decodeWithPassword(encoded, 'mypassword');
// → 'short secret'
```

**Note:** hanLock encoding expands data size (~3x). Keep input short to stay within the on-chain row size limit.

## x402 Payment Flow

See `references/x402-payments.md` for full API.

Payment-gated file inscription to Solana:

1. **Quote** — `POST /quote` with file metadata → get price in USDC/SOL
2. **Pay** — Send payment transaction to provided address
3. **Inscribe** — `POST /inscribe` with payment proof → file chunked into Solana transactions
4. **Download** — `GET /download/:txId` → reconstruct file from on-chain chunks

## Use Cases

- **Discord RPG Bot** — On-chain character persistence, provable item ownership, immutable game state
- **Governance** — Tamper-evident proposal/vote storage with rolling hash audit trail
- **Compliance logs** — Verifiable edit history for call center records
- **Paid storage** — Monetize data inscription via x402

## References

- `references/iqdb-core.md` — Full IQDB SDK API reference
- `references/hanlock.md` — hanLock encoding/decoding reference
- `references/x402-payments.md` — x402 payment inscription flow
- `references/setup.md` — Environment setup guide
