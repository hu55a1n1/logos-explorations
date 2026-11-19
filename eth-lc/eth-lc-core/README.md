# eth-lc-core

Ethereum light-client core for RISC Zero guests. It implements the
consensus-specs light-client sync protocol at tag `v1.7.0-beta.4` for
Capella through Gloas, anchors one execution block per finalized header, and
checks claims about accounts, storage slots and receipt logs against that
anchor.

The crate is `no_std` + `alloc` with `default-features = false`. Guests use
that form.

## API overview

| Item | Use |
|---|---|
| `InstanceConfig` (`ChainBinding`, `FinalityParams`) | The instance configuration (proposal 4.1): checkpoint, chain identity, fork schedule, `min_participants`, `weak_subjectivity_period`. |
| `ConsensusState`, `Committee`, `FinalisedHeader`, `Event` | The program's accounts and events (proposal 4.2 to 4.4, 4.8). Each is written once. |
| `instance::{init, hand_off, advance}` | The transitions that write those accounts. `hand_off` takes an update that installs the next committee without a new block (a new instance's first update). Seeds: `InstanceConfig::seed`, `instance::{state_seed, committee_seed, header_seed}`. |
| `ChainConfig`, `check_profile` | The parameters the sync rules read: an `InstanceConfig` with the pinned preset (`ChainConfig::from_instance`). Never stored. The chain identity and finality parameters pass structural checks. |
| `ChainConfig::mainnet()`, `sepolia()`, `hoodi()` | Constructors with each chain's values, cross-checked against recorded Beacon API responses. They are conveniences: any chain of the pinned preset passes the same checks. |
| `initialize_store(cfg, &BootstrapInput)` | `initialize_light_client_store` plus the config check and the execution anchor. The only way to anchor a store. |
| `apply_update(cfg, &store, &UpdateInput)` | Guest entry: one strict store transition. |
| `get_lc_execution_root(cfg, &header)` | The spec's execution root of a header, at the fork of its own slot. |
| `validate_light_client_update(..., current_slot)` | The full spec check, read-only. For parity tests and off-chain callers. |
| `stale_after_ms(cfg, &store)` | End of the update window: the time at which the store stalls and its committees' keys expire. |
| `claim::check_claim(&header, &claim)` | Checks a `Claim` (facts, each a `Proof` and `Condition`s) against a `FinalisedHeader` and returns the `Statement` (proposal 4.5 to 4.7). |
| `FinalisedHeader::from_store(config, instance, &store, signers, period)` | The header account of a store's finalized header. |

Inputs carry the full signing committee and `KeyHints` (uncompressed keys of
the committee aggregate and of each non-signer). `eth-lc-host` builds them.

## Trust model

- Safety rests on two inputs: the `Init` anchor (the chain config and the
  trusted checkpoint behind `initialize_store`), and an honest supermajority of
  each sync committee the store accepts as signer.
- `initialize_store` and `apply_update` pin the preset (mainnet preset: 12 s
  slots, 32 slots per epoch, 256 epochs per sync committee period, 512
  members) and check the chain identity for structure only: nonzero chain id
  and genesis validators root, one fork entry each from phase0 through Gloas,
  phase0 at epoch 0, non-decreasing epochs, Capella scheduled, distinct fork
  versions. The same program therefore serves any chain of that preset. The
  identity values are part of the trust anchor: verify them against the chain
  with `eth_lc_host::checkpoint::verify_instance_config`. The state bytes keep
  the config, and `Advance` keeps it byte-identical.
- The core checks every header without the LEZ sequencer timestamp. The
  timestamp sets the end of the update window, which is the expiry of
  committee keys. A late or stuck timestamp keeps a finished committee's
  update includable, so the expiry holds only while the clock moves forward.
- The store accepts an update signed by the next committee only when its
  finalized header is also in the next period. That update rotates the
  committees. This is a stronger assumption than the spec, and it adds a short
  finality gap at each period boundary (about two epochs).
- `apply_update` stores only finalized headers from updates with at least two
  thirds participation and a finality proof.
- A stall is fatal for an instance. Each instance keeps its first anchor for
  life. Recovery is a new instance with a fresh `Init` (see "Stalls and
  migration" in the workspace README).
- Hinted keys skip the per-key subgroup check. A hinted key is on the curve and
  compresses to the committed bytes, so it is the committed key. The committed
  bytes chain by hash to the trusted checkpoint. The participant key must
  differ from the identity point.

## Guest example (LEZ v0.4 actor)

`Init` anchors an empty instance. `Update` runs on a private actor of the
program and sends `Advance` to the public instance. `Attest` checks a claim
against the instance's finalised header and emits the statement. This
example keeps one instance account holding `(InstanceConfig, Store)`; the
proposal's account layout (state, committee and header PDAs) is
`eth_lc_core::instance::{init, hand_off, advance}`.

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use eth_lc_core::claim::{check_claim, Claim};
use eth_lc_core::{
    apply_update, initialize_store, stale_after_ms, BootstrapInput, ChainConfig, Event, FinalisedHeader,
    InstanceConfig, Store, UpdateInput,
};
use lee_core::account::Actor;
use lee_core::program::{ProgramEvent, ReceiveInput, Response};

#[derive(BorshSerialize, BorshDeserialize)]
pub enum LcMsg {
    /// Instance actor: anchors an empty instance. The instance must authorize it.
    Init { config: InstanceConfig, input: BootstrapInput },
    /// Private actor: `input` is the instance state followed by `borsh(UpdateInput)`.
    Update { instance: Actor, input: Vec<u8> },
    /// Instance actor: replaces its state, only from this program and only on top of `prev`.
    Advance { prev: Vec<u8>, next: Vec<u8> },
    /// Instance actor: checks `claim` against the finalised header and emits the statement.
    Attest { claim: Claim },
}

/// Event selector: the first 8 bytes of `sha256("eth-lc::Event")`.
pub const EVENT: [u8; 8] = [0x67, 0xa6, 0x3c, 0xa3, 0x1e, 0x9d, 0xfc, 0xbc];

// A panic aborts the guest, which refuses the message: the `expect` calls are the refusal paths.
fn handle(input: &ReceiveInput) -> Response {
    match borsh::from_slice::<LcMsg>(&input.message).expect("lc message") {
        LcMsg::Init { config, input: bootstrap } => {
            assert!(input.is_authorized, "init must be authorized");
            assert!(input.pre_state.as_ref().is_empty(), "instance already anchored");
            assert_eq!(config.checkpoint, bootstrap.trusted_block_root, "checkpoint");
            let store = initialize_store(&ChainConfig::from_instance(&config), &bootstrap).expect("bootstrap verifies");
            Response::set_state(borsh::to_vec(&(config, store)).expect("state encodes"))
        }
        LcMsg::Update { instance, input } => {
            let mut rest = input.as_slice();
            let (config, store) = <(InstanceConfig, Store)>::deserialize(&mut rest).expect("instance state");
            let prev = input[..input.len() - rest.len()].to_vec();
            let update: UpdateInput = borsh::from_slice(rest).expect("update input");
            let cfg = ChainConfig::from_instance(&config);
            let next_store = apply_update(&cfg, &store, &update).expect("the update verifies");
            let next = borsh::to_vec(&(&config, &next_store)).expect("state encodes");
            let cfg_len = borsh::to_vec(&config).expect("config encodes").len();
            assert_eq!(next[..cfg_len], prev[..cfg_len], "config changed");
            Response::keep_state()
                .send_sync(instance, &LcMsg::Advance { prev, next })
                .timestamp_window(..stale_after_ms(&cfg, &store))
        }
        LcMsg::Advance { prev, next } => {
            assert!(input.from_own_program(), "advance must come from this program");
            assert_eq!(input.pre_state.as_ref(), prev.as_slice(), "stale store");
            Response::set_state(next)
        }
        LcMsg::Attest { claim } => {
            let (config, store): (InstanceConfig, Store) = borsh::from_slice(input.pre_state.as_ref()).expect("instance state");
            let instance: [u8; 32] = input.receiver.account_id.as_ref().try_into().expect("32-byte id");
            let period = ChainConfig::from_instance(&config).period_at_slot(store.finalized.slot);
            // This one-account example keeps no signer count: it records 0.
            let header = FinalisedHeader::from_store(&config, instance, &store, 0, period);
            let statement = check_claim(&header, &claim).expect("claim holds");
            Response::keep_state().event(ProgramEvent {
                selector: EVENT,
                data: borsh::to_vec(&Event::Attested { statement }).expect("event encodes"),
            })
        }
    }
}

lee_core::actor_entrypoint!(raw handle);
```

Notes:

- The `Update` response sets an upper timestamp bound at `stale_after_ms`. LEZ
  then refuses updates on a stale instance. `Attest` leaves its window unbounded, so a
  stalled instance keeps serving claims against the headers it recorded.
- Execution block numbers can repeat at Gloas (two beacon slots can name the
  same parent block). The program keeps one header per execution block
  number: `advance` refuses a later slot at the same number (`NoProgress`),
  and `hand_off` takes the committee such an update carries.
- A Gloas-variant header needs the RLP execution header of its
  `execution_block_hash` in the input, at any slot. A Gloas-layout header at
  a pre-Gloas slot (the Gloas upgrade of an older header,
  `specs/gloas/light-client/fork.md`) names its own payload block, proven at
  gindex 412 (Capella) or 812 (Deneb through Fulu). Its anchor is the same
  block that the older layout anchors.

## Claims

- State trie key `keccak256(address)`; storage key `keccak256(key)`; receipt key `rlp(tx_index)`.
- Absent accounts read as nonce 0, balance 0, the empty storage root and
  `keccak256("")`. Absent slots read as 0. An absent receipt fails the claim.
- Receipt types: legacy, `0x01`, `0x02`, `0x03`, `0x04`. Other type bytes fail
  with `Error::UnknownReceiptType`. A later receipt type gets support after its
  EIP text is checked.
- A `Proof` carries the values it proves (the account, the storage value, the
  receipt). Each must equal what its trie proof shows
  (`Error::InvalidInclusionProof`).
- A condition reads a subject its fact carries. Any other subject, an absent
  topic or data word, and a packed field past 32 bytes fail with
  `Error::MalformedInput`. Comparisons are unsigned big-endian.
- A log of a reverted transaction fails with `Error::ConditionUnsatisfied`.
- Fact IDs: `SHA-256("eth_lc/statement/v1" ‖ chain_id ‖ kind ‖ fields)`,
  integers big-endian (`chain_id`, indices, `nonce`, `len(data)` as 8 bytes;
  `kind`, `n_topics` as 1 byte). Storage and account facts are `Class::State`,
  logs are `Class::Occurrence`.
- A `Statement` is plain data. It is a result for the program that ran
  `check_claim`. A consumer that needs authentication re-checks the claim
  against a header it holds itself.

## Stalls and migration

See the workspace README: a stalled instance stays readable, and recovery is a
new instance anchored at a chain config and a checkpoint that
`eth_lc_host::checkpoint::verify_new_checkpoint` accepts.

## Features

| Feature | Use |
|---|---|
| `std` (default) | `std::error::Error` for `Error`. Guests turn it off. |
| `minimal-profile` | The minimal preset as a second pinned preset, and `ChainConfig::minimal`, for conformance tests. |
| `spec-oracle` | Exposes the spec transition (`oracle::apply_light_client_update`) to host tests. |

The crate refuses `minimal-profile` and `spec-oracle` on `target_os = "zkvm"`.
