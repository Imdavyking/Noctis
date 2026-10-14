# Noctis — Private, Auditable USDC Payments on Arc

Private payments infrastructure for [Arc](https://arc.network), Circle's stablecoin-native L1.

## Problem

Arc is designed for real economic activity — payroll, B2B settlement, treasury flows, FX — denominated in USDC/EURC. Most of that activity is sensitive: no business wants its full balance and every counterparty visible on a public ledger. But regulated finance can't just go dark either — auditors and regulators need selective, provable access to transaction history.

Arc's own design goal is "opt-in configurable privacy: balances and transactions can be selectively shielded while supporting compliance with relevant obligations." Shielded Arc is a concrete implementation of that goal for USDC.

## What it does

Shielded Arc lets users deposit USDC into a shielded pool, transact privately using zero-knowledge notes, and selectively disclose specific transactions to an auditor — without revealing their full balance or history to anyone else.

- **Deposit**: USDC is locked in the pool and minted as a fixed-denomination private note.
- **Private transfer**: Notes move between users via ZK proofs; nobody on-chain learns sender, receiver, or amount beyond the fixed denomination.
- **Withdraw**: Notes are redeemed back to USDC by proving ownership without revealing which deposit they came from.
- **Selective disclosure**: The note owner can generate a proof that reveals a specific transaction to a designated auditor key, without exposing anything else.

## Why fixed denomination

Notes are issued in fixed sizes (e.g. $1, $10, $100 USDC) rather than arbitrary amounts. This keeps every note in a denomination indistinguishable from every other note of that size, which is what makes the anonymity set meaningful — arbitrary amounts leak information through amount correlation alone.

## Architecture

```
┌─────────────────────┐        ┌──────────────────────────┐
│   Client (Flutter /  │        │   Arc (EVM-compatible)    │
│   web wallet)        │        │                            │
│                       │        │  ┌──────────────────────┐ │
│  ZkProofBridge ───────┼───────▶│  │ ShieldedPool.sol      │ │
│  (Noir + bb.js,       │        │  │  - deposit()          │ │
│   UltraHonk proving)  │        │  │  - transferPrivate()  │ │
│                       │        │  │  - withdraw()         │ │
│  VaultNoteStore       │        │  │  - Merkle root store  │ │
│  PrivateVaultClient   │        │  │  - nullifier set      │ │
│  Auditor log          │        │  └──────────┬───────────┘ │
│  (encrypted           │        │             │              │
│   ciphertexts)        │        │  ┌──────────▼───────────┐ │
└─────────────────────┘         │  │ UltraHonkVerifier.sol │ │
                                  │  └───────────────────────┘ │
                                  └──────────────────────────┘
```

### Circuits (Noir / UltraHonk)

- **Note commitment**: `commitment = Poseidon2(denomination, owner_pubkey, blinding)`
- **Membership proof**: note commitment is a leaf in the current Merkle root maintained on-chain
- **Nullifier**: `nullifier = Poseidon2(commitment, owner_privkey)`, checked against the on-chain nullifier set to prevent double-spend
- **View key (BabyJubJub)**: derives a view key per note for the owner to scan and decrypt notes addressed to them
- **`note_link`**: a per-note selective disclosure circuit — proves a specific note belongs to a specific transaction, revealing only what's disclosed and nothing about the rest of the owner's notes

### On-chain (Arc, EVM-compatible)

- `ShieldedPool.sol` — deposit/withdraw/transferPrivate, Merkle root history, nullifier set
- `UltraHonkVerifier.sol` — on-chain verification of Noir-generated proofs
- Deposit/withdrawal events log `depositor: address` / `recipient: address` for the public entry/exit points; everything between stays shielded

### Client

- Flutter wallet, `ZkProofBridge` over WebView/bb.js for local UltraHonk proving (requires COOP/COEP headers for `SharedArrayBuffer`)
- `VaultNoteStore` — local note management
- `PrivateVaultClient` — deposit/transfer/withdraw flows
- Auditor-facing log of encrypted ciphertexts, decryptable only with the relevant disclosure proof

## Compliance model

This isn't anonymity-by-default — it's privacy-by-default with disclosure-on-demand:

1. Every note has an owner-controlled view key.
2. The owner can generate a `note_link` proof for any single transaction, revealing it to a designated auditor public key.
3. The auditor can verify the disclosure without needing pool-wide visibility or cooperation from anyone else.
4. Nothing about undisclosed notes is revealed by a disclosure proof.

This maps to standard travel-rule / audit-request workflows: a regulator or counterparty requests proof of a specific transaction, the user proves it, nothing else leaks.

## Why Arc

- EVM compatibility means the Noir/UltraHonk verifier and note logic port directly from prior work — no new proving system.
- Arc's sub-second finality makes shielded transfers feel instant despite the extra proving step.
- USDC-native fees mean users transact in the same asset they're moving, no separate gas token.
- Arc's own stated design goal — selective shielding with compliance — is exactly what this project implements.

## Status

Prototype adapted from an existing shielded-note wallet built on Soroban; this hackathon build is the EVM/Arc port of the contracts and verifier, plus the disclosure flow.

## Roadmap

- [ ] Deploy `ShieldedPool.sol` + `UltraHonkVerifier.sol` to Arc testnet
- [ ] Port Noir circuits (commitment, membership, nullifier, note_link) — no logic changes expected
- [ ] Wire Flutter client to Arc RPC
- [ ] Auditor disclosure demo (single-transaction proof, out-of-band verification)
- [ ] Multi-denomination pool support ($1 / $10 / $100)

## Disclaimer

Prototype built for hackathon purposes. Not audited. Do not use with real funds.
