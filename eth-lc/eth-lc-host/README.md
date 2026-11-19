# eth-lc-host

Host-side support for `eth-lc-core`.

## Quick start

```console
cd eth-lc
just verify-checkpoint                    # check a new mainnet instance's config and checkpoint
just verify-checkpoint --network sepolia  # the same on Sepolia
just vectors                              # re-record the mainnet and Sepolia test data
```

## Modules

| Module or binary | Use |
|---|---|
| `forks` | Light-client containers per fork layout (Capella, Deneb, Electra and Fulu, Gloas), from Beacon API JSON or SSZ, converted to core types, and the Gloas upgrade functions of `specs/gloas/light-client/fork.md`. A Capella or Deneb layout becomes the core variant of the fork at its own slot; a Gloas layout becomes a Gloas header at any slot from Capella on. |
| `chain_dir` | `ChainDir::load(dir)`: a recorded bootstrap, period updates, a finality update and RLP execution headers; `update_input` builds the guest input, with the RLP header a Gloas-variant finalized header needs. |
| `hints` | `key_hints(committee, bits)`: the uncompressed aggregate and non-signer keys. |
| `checkpoint` | `stall_status`; `verify_instance_config` (chain config against Beacon API and execution endpoints) and `verify_new_checkpoint` (config, then checkpoint, with the RLP header of a Gloas bootstrap) for new instances. |
| `execution` | RLP execution headers from JSON-RPC block objects, checked by Keccak-256. |
| `evidence` | `eth_getProof` results and receipt trie proofs as `Evidence`. |
| `synthetic` | Test-key committees, consistent branches and signed updates for any supported fork. |
| `fetch_chain` | Records a mainnet or Sepolia chain-dir; `verify-checkpoint` checks a config and a checkpoint against the endpoints. |
| `fetch_evidence` | Records claim evidence at the finalized block of the latest finality update. |
