# E1: light-client update in LEZ's proven (private) mode

Question: can a sync-committee update run as a **private actor** of the light-client
program, proven by the submitter, and advance the program's **public** instance?
If yes, what does the chain charge, and what does the submitter pay in proving?

Answer so far: **yes**. The sequencer charges a flat private-transaction fee plus a
small public part, whatever the private part computed. Details and gaps below.

## The mechanism under test

```mermaid
sequenceDiagram
    participant S as Submitter (private account)
    participant P as LC program, private actor (proven)
    participant I as LC program, public instance actor
    S->>P: Update { material }  (root call, hidden)
    Note over P: full check: BLS, branches, handoff
    P->>I: Advance { prev_store, next_store }  (published boundary message)
    Note over I: accept only from own program<br/>and only if prev_store == current
    I-->>I: store := next_store
```

- The same deployed program handles both messages. The privacy circuit proves the private one.
- The sequencer verifies the circuit receipt (flat charge), then runs only `Advance`.

## Results

LEZ `v0.4.0-rc1` (`1806e873`). RISC Zero 3.0.5 guests, `r0vm` 3.0.6. Apple M3 Max, 16 cores, 64 GB.
E1b runs `eth-lc-core` (`../../eth-lc`), written from consensus-specs `v1.7.0-beta.4`, on recorded mainnet periods 1902 to 1905.
The program pins the mainnet preset and takes the chain identity from `Init`, so the same image serves mainnet, Sepolia and Hoodi. The operator verifies the identity with `eth_lc_host::checkpoint::verify_instance_config`.

| Run | What | Result |
|---|---|---|
| E1a, dev | toy update, 12,609,529 private cycles | settles; public part **9,547** cycles |
| E1a, dev | `Advance` from another program | refused: "advance must come from this program" |
| E1a, dev | `Advance` on a stale store | refused: "stale store"; store unchanged |
| E1a, dev | blinded sender | still recognised as its program; settles |
| E1a, dev | same transaction replayed | refused by LEZ (commitment already seen) |
| E1a, **real proof** | toy update, 12.6M cycles | settles with a verified receipt; **895 s** CPU, **13.0 GB** peak, proof **224,813 B** |
| E1b, dev | **real mainnet update** with committee handoff (period 1902 to 1903, step 1) | settles; private **14,058,921** cycles, public part **19,959** cycles; input 52,218 B stays off chain |
| E1b, dev | guest wire and `Update` window, step 1 (`just e1b-window`) | settles at `stale_after_ms - 1`; refused at `stale_after_ms`; one trailing input byte refused. The README guest example (`just consumer-check` in `eth-lc`) passes the same cases, and its `Attest` leaves the timestamp window open |
| E1b, dev | two instances under one program, steps 1 and 2 (`just e1b-instances`) | an update advances only its own instance; step 1's input sent to the other instance is refused, and neither instance moves |
| E1b, **real proof** | the same real update | settles with a verified receipt; **1,107 s** CPU, **13.1 GB** peak, transaction **228,355 B** |
| E1b, dev | the same real update **executed directly** (what a sequencer would run on the public path) | **255 ms** mean of 5. For scale: LEZ's fee model prices verifying a private-transaction receipt at about 12 ms (`PRIVATE_VERIFY_GAS`, M2 Pro calibration) |
| E1c, dev | toy update through the block fee path (`settle_transaction`) | exec gas **419,311** = 409,764 flat + 9,547; storage gas **224,063**; **4.19%** of a block's exec, **22.4%** of its storage; base fee 5,146,992 at the floor price |
| E1c, dev | two operators race the same update | loser lands as a **charged revert**: 5,470,616 (its full reserve), store unchanged |
| E1c, **real proofs** | both E1c cases, tiny private burn | identical gas and fees with real receipts; 3 proofs in 699 s (**about 233 s per private transaction** of fixed circuit overhead) |

### Against the public multi-step path

| | Public, 3 transactions (Proposal #224 on v0.3) | Proven, 1 transaction (E1) |
|---|---|---|
| Exec gas per update | about 12.6M (126% of one block, over 3 blocks) | 419,311 to 429,723 (4.2% to 4.3% of one block) |
| Storage gas per update | update material, about 52 KB | 224,063 (the receipt) |
| Base fee at floor prices | about 101M | about 5.1M (**about 20x less**) |
| Job state across transactions | yes | none |
| Submitter hardware | LEZ gas only | a prover: **1,107 s, 13.1 GB** on a 16-core laptop CPU, or **30.2 s** on an RTX 4090, for a real update (E1b) |

Public-path figures come from Proposal #224's published measurements, not from this experiment.

### Findings worth keeping

1. Proving cost splits into a fixed part (about 233 s for any private transaction on this CPU) and the update itself (about 874 s more for the E1b update, 14.1M cycles). On an RTX 4090 the same E1b update proves in **30.2 s** (30.3 s on a cold run), about 37x the laptop CPU. Host RSS was 758 MB. Tessera's about 30x was a different program on an L4, not this update.
2. A canonical sender publishes the submitter's private account ID in the boundary message. Blind it (the test does).
3. A losing racer pays its whole reserve. The update daemon must re-check the instance immediately before submitting (R.9) and keep its reserve tight.
4. LEZ refuses `test-utils` in release builds without debug assertions. `chain_state` tests run with `CARGO_PROFILE_RELEASE_DEBUG_ASSERTIONS=true`.

## What E1 does **not** establish yet

| Gap | Why it matters | How to close it |
|---|---|---|
| **Run on a standalone sequencer node** | Settlement code is covered (`settle_transaction`, the code the sequencer runs). Untested: RPC submission, mempool admission, deployment through the program loader, the node's rules on which public accounts a private transaction may touch. | Needs a Bedrock node. LEZ's resolver found **no prebuilt binary** for the pinned revision and the source build **failed to link** here (Rust 1.99, `rust-lld`). Next try: `bedrock/tools/resolve_bedrock_node_in_docker.sh`, then an integration test modelled on `integration_tests/tests/async_payouts.rs`. |
| GPU proving time | Sets realistic operator cost and update latency. | Measured: E1b step 1, **30.2 s** on an RTX 4090 (24 GB, CUDA 12.4, `r0vm` 3.0.6 built with `cuda`). |
| Remote proving (Bonsai, Boundless) | Lets operators without proving hardware submit. A prover is liveness-only: it cannot forge an update. | Split the proof. The heavy program proof (about 874 s, public inputs only) goes remote. The privacy-circuit proof (about 233 s, needs the submitter's private keys) stays local. LEZ proves both through one `default_prover()`, so the split needs a small LEZ change or a daemon-side path. Open checks: Bonsai availability, and whether Boundless receipts work as circuit assumptions. Fallback: a throwaway fee-only private account. Planned as E1d (needs an API key). |
| Lowest-spec machine | P.4 asks for the declared minimum. | Repeat `e1b-prove` on a 16 GB machine. |
| Testnet 0.4 | Delivery target; not released (end of November per roadmap). | Re-run on release. |

## Reproduce

Prerequisites: Rust (rustup), `just`, RISC Zero via `rzup` (`rzup install rust`, `rzup install cpp`, `r0vm` 3.0.x).

```console
cd rfp-021/e1
just            # list recipes
just e1a        # dev mode, about 1 min after the first LEZ build
just e1b 1      # dev mode, real mainnet update step 1 (steps 0 to 3)
just e1c        # dev mode, block fee path and racing copy
just e1a-prove  # real proof, about 15 min CPU
just e1b-prove 1
just e1c-prove  # real proofs with a small private burn (fee is flat in it)
```

Everything builds under `.work/` (ignored).

## Layout

| Path | Content |
|---|---|
| `justfile` | all recipes |
| `lez-patch/e1.patch` | the LEZ changes: `lc_spike` guest, `lee` tests (E1a, E1b), `chain_state` tests (E1c) |
| `e1b-guest/` | E1b guest: `eth-lc-core` as a LEZ actor, accelerated `blst` |
| `e1b-input/` | E1b inputs from the recorded mainnet chain-dir (`../../eth-lc/vectors/mainnet`) |
| `e1b-pack/` | packs the guest ELF with RISC Zero's v1compat kernel (the form LEZ loads) |

## Build notes

- The E1b guest builds with a plain `cargo build` and the rzup toolchains. risc0-build drops its `bigint2` signal once `CC_*` is overridden.
- macOS `ar` silently drops RISC-V objects (an empty `libblst.a`). The recipe sets `AR_riscv32im_risc0_zkvm_elf` to the RISC Zero toolchain's `ar`.
- Real proving goes through the prebuilt `r0vm` (no `prove` feature), because building RISC Zero's Metal kernels needs Xcode.
