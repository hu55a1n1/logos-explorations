# logos-explorations

Research and prototypes for Logos RFP-021 (trustless Ethereum state attestation on LEZ).

## Quick start

Prerequisites: Rust (rustup), `just`, and for the LEZ runs RISC Zero via `rzup`
(`rzup install rust`, `rzup install cpp`, `r0vm` 3.0.x).

```console
just -f eth-lc/justfile test       # light-client crates: unit, recorded mainnet and Sepolia chains
just -f rfp-021/e1/justfile e1b 1  # a real mainnet update as a private LEZ v0.4 transaction (dev mode)
```

## Contents

| Path | What |
|---|---|
| [`eth-lc/`](eth-lc/README.md) | Ethereum light-client crates for RISC Zero guests: sync protocol (Capella through Gloas), execution anchor, account, storage and log claims. |
| [`rfp-021/e1/`](rfp-021/e1/README.md) | E1: a sync-committee update proven in LEZ's private mode, with its gas, proving time and memory. |
