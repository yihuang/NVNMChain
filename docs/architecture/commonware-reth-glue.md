# Building a Sovereign EVM L1 on Commonware + Reth/revm: An Engineering Map of the Glue

**Last verified: 2026-06-25**

> Scope: the *concrete technical mechanics* of wiring Commonware (consensus + runtime +
> p2p + storage) to Reth/revm (EVM execution). This is the integration layer ("the glue"),
> not a tutorial on either component in isolation.
>
> Primary evidence comes from three sources verified on 2026-06-25:
> 1. The Commonware monorepo (`github.com/commonwarexyz/monorepo`), `consensus/`, `runtime/`,
>    `p2p/`, `glue/` crates, read at source level.
> 2. The **Tempo** L1 (`github.com/tempoxyz/tempo`) — a production EVM payments chain that is
>    *exactly* this architecture (Reth SDK + Commonware Simplex). Tempo is the single best
>    reference implementation of the glue and is cited throughout with file paths.
> 3. Reth docs (`reth.rs`) and `paradigmxyz/reth` (Tempo pins rev `c0a9431`).
>
> Where an API name could not be confirmed against source, it is flagged **[UNVERIFIED]**.

---

## Executive summary

- **Commonware gives you ordered agreement over opaque digests, not a blockchain.** Consensus
  agrees on a 32-byte `Digest` per view; your application supplies the bytes behind it and all
  execution semantics. You implement a small set of traits — chiefly **`Automaton`**
  (`propose`/`verify`), **`Relay`** (`broadcast`), and **`Reporter`** (`report`) — and Commonware
  drives them. Finalized, *totally-ordered* blocks are delivered to you by the **`marshal`** actor
  via a `Reporter<Activity = Update<Block>>` callback. This is the hand-off boundary.

- **Reth is best embedded via the SDK, not driven over JSON-RPC Engine API.** Tempo embeds Reth
  through the Reth **node builder** (custom `Node` impl), *disables* the external beacon Engine API
  (`NoopEngineApiBuilder`), and drives execution **in-process** through the engine-tree handle
  **`ConsensusEngineHandle<T>`** — calling `.new_payload(...)` to execute/validate and issuing
  **forkchoice updates** to canonicalize/finalize. This keeps the engine-API *semantics* (newPayload
  + forkchoiceUpdated + getPayload) but avoids the localhost JSON-RPC round-trip and gives you direct
  access to Reth's payload builder, tx-pool, EVM config, and providers.

- **Ownership split.** Reth owns: the mempool (tx-pool), payload building, revm execution, state &
  state-root, the canonical store/DB, and all `eth_*` JSON-RPC. Commonware owns: leader election,
  block proposal ordering decision, voting/notarization/finalization, threshold signatures, p2p block
  dissemination/backfill (`marshal` + `resolver`), and the validator set view. *You* write the actor
  that bolts the two together (the Tempo `consensus` crate is this layer).

- **Custom precompiles** are injected through Reth's EVM config: implement `ConfigureEvm`, expose an
  `EvmFactory`, and override the precompile set via revm's **`PrecompilesMap`** (Tempo uses
  `PrecompilesMap::from_static(EthPrecompiles…)` then `set_precompile_lookup(...)` to add address-keyed
  custom precompiles). Because they run inside the deterministic `BlockExecutor`/revm path, they are
  consensus-safe as long as they are pure functions of `(input, caller, committed state)`.

- **Build-vs-free:** You get for free — EVM execution, mempool, payload building, state, `eth_*` RPC,
  block sync/backfill, BFT consensus with sub-second finality, threshold BLS certs, and a deterministic
  async runtime. You must build — the consensus↔Reth actor glue, the `Automaton`/`Relay`/`Reporter`
  impls, the block type & codec, genesis, the staking/validator-selection module feeding the Commonware
  `Supervisor`, validator-set reconfiguration/DKG plumbing, governance/upgrades, and any bridging/DA.

---

## 1. Commonware consensus interface

### 1.1 The application-side traits (verified in `consensus/src/lib.rs`)

Commonware's design philosophy ("the anti-framework") is that consensus agrees on opaque blobs and
delegates everything block-shaped to interfaces you implement. The core traits:

**`Automaton`** — drives proposal and verification. This is the central trait.
```rust
// consensus/src/lib.rs  (verified)
pub trait Automaton: Clone + Send + 'static {
    type Context;          // engine-supplied metadata (proposer, view, parent, epoch, height)
    type Digest: Digest;   // 32-byte hash of your payload

    fn propose(&mut self, context: Self::Context)
        -> impl Future<Output = oneshot::Receiver<Self::Digest>> + Send;

    fn verify(&mut self, context: Self::Context, payload: Self::Digest)
        -> impl Future<Output = oneshot::Receiver<bool>> + Send;
}
```
Both methods are *two-stage*: the `async fn` returns immediately with a `oneshot::Receiver`; you
complete the receiver later when your (possibly slow) build/execute finishes. This lets consensus
proceed while execution runs. `propose` commits the local proposer to later verifying the same
`(context, payload)`. `verify` must return `false` only for *permanently* invalid payloads.

> **Note on `genesis`:** docs.rs lists only `propose`/`verify` on `Automaton`, but Tempo's
> `impl Automaton` also defines `async fn genesis(&mut self, epoch) -> Digest`
> (`crates/consensus/src/consensus/application/ingress.rs`). Treat `genesis` as present in the
> current source even though the published rustdoc trait page omitted it — confirm against the exact
> Commonware revision you pin. **[Partially verified — version-sensitive]**

**`CertifiableAutomaton`** (extends `Automaton`) — adds a `certify(round, payload)` step ("is this
verified payload *safe to commit*") used by the Simplex `Engine` before finalization:
```rust
pub trait CertifiableAutomaton: Automaton {
    fn certify(&mut self, round: Round, payload: Self::Digest)
        -> impl Future<Output = oneshot::Receiver<bool>> + Send;
}
```
The Simplex `Engine` is generic over `A: CertifiableAutomaton`, so this is the trait you actually wire.

**`Relay`** — broadcasts the payload bytes behind a digest to peers.
```rust
pub trait Relay: Clone + Send + 'static {
    type Digest: Digest;
    type PublicKey: PublicKey;
    type Plan: Send;                              // how/where to broadcast
    fn broadcast(&mut self, payload: Self::Digest, plan: Self::Plan) -> Feedback;
}
```
In a "standard" chain the proposer pushes the whole block to each validator (`marshal::standard` over
`broadcast::buffered`); `Relay` is how the application tells the network layer to disseminate it.

**`Reporter`** — receives activity from consensus (votes, notarizations, finalizations, faults). This
is also the channel through which `marshal` delivers finalized blocks.
```rust
pub trait Reporter: Clone + Send + 'static {
    type Activity;
    fn report(&mut self, activity: Self::Activity) -> Feedback;
}
```
`Activity` is concretized by the consensus module: in raw Simplex it is `Activity<S, D>` (votes /
notarize / finalize / fault — use it for rewards & slashing); for the marshal actor it is
`Update<Block>` (see §1.3).

**Supporting traits:** `Monitor` (`subscribe()` to observe the in-progress view/index), `Block`
(`Heightable + Codec + Digestible + parent()`), `CertifiableBlock` (adds `context()`), and the
higher-level `Application<E>` trait (`propose(context, ancestry) -> Option<Block>` /
`verify(context, ancestry) -> bool`) which is a *batteries-included* alternative to hand-wiring
`Automaton`+`Relay`+`Reporter` for "a stream of epoched blocks." All verified in `consensus/src/lib.rs`.

### 1.2 The Simplex / threshold-simplex Engine

The consensus engine you instantiate (verified in `consensus/src/simplex/engine.rs` and
`.../config.rs`):
```rust
pub struct Engine<E, S, L, B, D, A, R, F, T> { … }   // E=runtime ctx, S=scheme, L=elector,
                                                      // B=blocker, D=digest, A=automaton,
                                                      // R=relay, F=reporter, T=strategy
impl Engine { pub fn new(context: E, cfg: Config<…>) -> Self; }
```
`Config` fields (verified, abridged): `scheme`, `elector`, `blocker`, `automaton`, `relay`,
`reporter`, `strategy`, `epoch`, `floor` (`Genesis | Finalized` certified root), `leader_timeout`,
`certification_timeout`, `activity_timeout`, `skip_timeout`, `fetch_timeout`, `forwarding`,
plus journal/buffer knobs. You construct it with the runtime context and then `.start()` it on the
runtime's spawner (the engine runs as a set of actors: Batcher, Voter, Resolver per the threshold
docs).

- **simplex** = Simplex BFT over a pluggable signing `Scheme`. Attributable schemes available:
  `ed25519`, `bls12381_multisig`, `secp256r1`.
- **threshold_simplex / bls12381_threshold scheme** = same protocol with **BLS12-381 threshold
  signatures** (2f+1 of 3f+1 quorum), yielding **succinct consensus certificates** and an embedded
  **VRF beacon** (leader election + post-facto randomness). Two sub-variants: `standard` (cert = vote
  sig only) and `vrf` (vote sig + round sig). Threshold sigs are *non-attributable* (a quorum can forge
  any member's partial), so you cannot use individual partials as fraud evidence — slashing must key off
  the `Activity` reports instead.
  - **API-name caveat:** the published `docs.rs/.../threshold_simplex/` URL returned 404 on 2026-06-25;
    the threshold variant is now organized as a **`Scheme`** under `simplex/scheme/bls12381_threshold/`
    rather than a separate `threshold_simplex` top-level module in current source. Tempo uses
    `Scheme<PublicKey, MinSig>` (the `MinSig` BLS variant). **[Module path version-sensitive — verify
    against your pinned rev.]**

**Minimmit:** a newer, *responsive* leader-based protocol (`n ≥ 5f+1`, single-round finalize on `L=n−f`
votes, `t+2Δ` happy-path latency). As of 2026-06-25 it exists **only as a specification**
(`pipeline/minimmit/minimmit.md`, arXiv 2508.10862) — **no consensus module implements it yet**.
Build on `simplex`/threshold today. **[Verified: spec only, not implemented.]**

### 1.3 How an ordered block reaches the application: the `marshal` actor

Raw Simplex only tells you a *digest* was finalized in a view. **`marshal`** ("ordered delivery of
finalized blocks") turns that into a totally-ordered stream of actual blocks. It is the real
consensus→application hand-off. Verified in `consensus/src/marshal/`:

- `core::Actor<…>` is the unified actor. It (1) receives uncertified blocks from broadcast, (2)
  receives notarizations/finalizations from consensus, (3) reconstructs total order, (4) backfills
  missing blocks via `resolver` (`commonware_resolver::p2p::Engine`).
- It delivers to a `Reporter` whose `Activity = Update<Block, A>`:
  ```rust
  pub enum Update<B: Block, A: Acknowledgement = Exact> {
      Tip(Round, Height, B::Digest),   // new finalized tip
      Block(B, A),                     // a finalized block + ack handle (in order)
  }
  ```
  Delivery is **at-least-once and in order** — your reporter must tolerate duplicates and `ack` each
  block when processed.
- **Persistence is your responsibility** via two traits you implement (verified in
  `marshal/store.rs`): `Certificates` (store/get/prune `Finalization` certs by height/digest) and
  `Blocks` (store/get/prune finalized blocks, plus `missing_items`/`next_gap` for backfill). Tempo's
  `Blocks` impl is a **`Hybrid`** store: marshal's own cache *plus a fallback into Reth's
  `BlockchainProvider`** — i.e. Reth's DB *is* the durable block store. (`crates/consensus/src/alias.rs`.)
- A `Start` enum / sync floor lets you resume from an arbitrary height instead of genesis (state sync).
  Tempo floors marshal at `max(marshal_stored_height, reth_finalized_height)` on boot
  (`alias.rs` lines ~170–185).

### 1.4 Finality signalling

Finality is **deterministic and single-shot** (no reorgs in Simplex): a block is final once a
`Finalization` threshold certificate exists for its view. You learn of it two ways: the raw
`Reporter<Activity = Activity<S,D>>` (finalize events, for rewards) and — the one you act on — the
**`marshal` `Update::Block`** in-order delivery. In Tempo, receiving a finalized block triggers a
forkchoice update to Reth marking that block hash `finalized` (§3).

### 1.5 Validator-set management & reconfiguration (DKG / resharing)

- Consensus is generic over a **`Supervisor`/`ThresholdSupervisor`** that, per epoch, returns the
  participant set and (for threshold schemes) the polynomial/share. The *source of truth* for "who is a
  validator this epoch" is **your application**, not Commonware. (Trait surface verified at a high level;
  exact `Supervisor` method names — e.g. `participants(epoch)` / `is_participant` / `leader` — were
  **[UNVERIFIED]** at source line level in this pass; confirm in `consensus/src/types.rs` and the simplex
  `elector` module for your pinned rev.)
- **Reconfiguration / DKG / resharing** is demonstrated in `examples/reshare/`: validators run a DKG at
  epoch boundaries, surfacing results through an `UpdateCallBack` (`Update::Success { output, share }` /
  `Update::Failure`) and a `PostUpdate { Continue | Stop }`. Pattern: bootstrap on a non-threshold scheme
  (ed25519), run DKG during a designated epoch, then transition to threshold consensus with the rotated
  key; reshare each subsequent epoch. Requires all validators online during a short synchrony window at
  the boundary.
- **Tempo's concrete approach (verified):** the validator set and DKG outcome are **on-chain artifacts**.
  `crates/dkg-onchain-artifacts` defines `OnchainDkgOutcome { epoch, output: Output<MinSig, PublicKey>,
  next_players, is_next_full_dkg }` with `dealers()/players()/next_players()/sharing()/network_identity()`.
  `crates/validator-config` defines `ValidatorConfig { chain_id, validator_address, public_key, ingress,
  egress }` with `check_add_validator_signature` / `check_rotate_validator_signature`. A `VALIDATOR_CONFIG`
  precompile (see §4) makes the set a contract-readable, governance-mutable on-chain object that the
  Commonware `Supervisor` reads each epoch. **This is the recommended template for a PoS/DPoS validator
  set.**

---

## 2. Reth as an embedded execution engine — the two options

### Option (a) — Standard Engine API (Reth as a black-box EL over JSON-RPC)
Treat Reth as a stock execution client and speak the beacon **Engine API**:
`engine_forkchoiceUpdatedV* ` (set head/safe/finalized + optionally start a build job),
`engine_getPayloadV*` (collect the built block), `engine_newPayloadV*` (submit a peer's block for
execution/validation). Your consensus process is the "consensus layer" client.

- **Pros:** clean process isolation; Reth upgrades independently; you can target *any* Engine-API EL
  (this is exactly what **Seismic's "Summit"** consensus client does — "works with any EVM client that
  implements the Engine API," built on Commonware primitives); simplest to reason about.
- **Cons:** JSON-RPC/IPC serialization on every block (latency tax that fights sub-second finality);
  you cannot reach inside to add precompiles, customize the pool, or share the execution cache with the
  builder; the Engine API surface is Ethereum-PoS-shaped (assumes a CL with slots/attestations) and you
  must adapt it.

### Option (b) — Embed via the Reth SDK / node builder (Tempo's choice)
Build a custom node with the Reth **node builder** and drive execution **in-process**. Verified in
Tempo `crates/node/src/node.rs`: a custom `Node` impl using Reth's `NodeBuilder`/add-ons,
`PayloadBuilderBuilder`, `PoolBuilder`, a custom `engine_validator_builder()` → `TempoEngineValidator`,
and crucially **`NoopEngineApiBuilder`** — the *external* beacon Engine-API RPC is **switched off**.
Instead, the consensus actor holds an in-process **`reth_node_builder::ConsensusEngineHandle<T>`**
(verified import + use at `crates/consensus/src/consensus/application/actor.rs:38, 975, 1016`) and calls:
- `engine_handle.new_payload(execution_data).await` — execute & validate a block (the in-process
  equivalent of `engine_newPayload`), returning a `PayloadStatus`.
- **forkchoice updates** via the executor actor (`crates/consensus/src/executor/`) which maintains a
  `reth …::ForkchoiceState { head_block_hash, safe_block_hash, finalized_block_hash }` and sends FCUs to
  canonicalize the head and mark blocks finalized; a periodic "forkchoice update heartbeat" keeps Reth's
  view fresh.
- payload building via the executor mailbox: `canonicalize_and_build(height, digest, attributes)
  -> oneshot::Receiver<TempoBuiltPayload>` (the in-process equivalent of FCU-with-attributes +
  `getPayload`).

- **Pros:** no RPC tax (direct in-process calls); full access to inject precompiles via `ConfigureEvm`,
  a custom tx-pool, custom payload attributes (`TempoPayloadAttributes`), and an execution-cache shared
  between validation and the builder; you can keep Engine-API *semantics* (newPayload/FCU/getPayload)
  without the wire format. This is the path that hits ~0.5s finality.
- **Cons:** tighter coupling — you pin a Reth git rev (Tempo pins `c0a9431`) and ride its (fast-moving,
  pre-2.0-era SDK) API churn; more surface to maintain; you re-implement the orchestration that the
  beacon-API would otherwise structure for you.

**Recommendation for a custom-consensus L1:** **Option (b)** — embed via the SDK and use
`ConsensusEngineHandle` in-process. It is the only way to get both sub-second finality *and* custom
precompiles/pool, and Tempo proves it works at scale. Keep `new_payload`/forkchoice/build as your
internal contract so you *could* fall back to true Engine-API if you ever want EL/CL separation.

---

## 3. Block production loop (concrete sequence)

Ownership, verified against Tempo:

| Concern | Owner |
|---|---|
| Mempool / tx-pool | **Reth** (Tempo wraps it: `TempoTransactionPool`, plus an `AA2dPool` for 2D account-abstraction nonces — `crates/transaction-pool`) |
| Leader election / who proposes when | **Commonware** (Simplex elector + VRF beacon) |
| *Which* txs / payload bytes | **Reth payload builder** (selects from its pool), invoked by the glue |
| Ordering decision (this digest, this view, this parent) | **Commonware** consensus |
| EVM execution + state + state root | **Reth/revm** (`BlockExecutor` inside `ConfigureEvm`) |
| Vote / notarize / finalize / certificates | **Commonware** |
| Block dissemination + backfill | **Commonware** (`Relay` + `marshal` + `resolver`) |
| Durable block/state store | **Reth DB** (marshal uses it as fallback `Blocks` store) |

**Happy-path sequence (leader's node), verified to Tempo file/method level:**

1. **Commonware** elects this node leader for the view and calls
   `Automaton::propose(context)` (`consensus/application/ingress.rs`).
2. The application actor (`consensus/application/actor.rs::propose`) resolves the parent block (via
   `marshal` / `subscribe`), builds `TempoPayloadAttributes` (proposer, ms-precision timestamp, DKG
   extra-data, consensus context, build-time budget), then calls
   `executor.canonicalize_and_build(parent_height, parent_digest, attrs)`.
3. The **executor actor** issues a **forkchoice-update-with-attributes** to Reth and Reth's **payload
   builder** pulls txs from its pool, **executes them in revm** (`TempoEvmConfig` /
   `BlockExecutor`), computes the **state root**, and returns a `TempoBuiltPayload`.
4. The application turns the payload into a Commonware `Block`
   (`Block::from_execution_block_with_encoded_cache`), persists it via `marshal.proposed(round, block)`,
   and resolves the `propose` oneshot with the block's `Digest`.
5. **Commonware** broadcasts the proposal (`Relay::broadcast`) and runs voting. Other validators get
   `Automaton::verify` → `verify_block(...)` which calls `engine_handle.new_payload(execution_data)` to
   **re-execute/validate** the block in their own Reth and returns the `oneshot<bool>`.
6. On ≥2f+1 notarize then finalize votes, Commonware recovers a **`Finalization`** threshold cert.
7. **`marshal`** delivers the finalized block in order as `Update::Block` to the executor's
   `Reporter::report`.
8. The **executor actor** sends a **forkchoice update** marking that block hash as `head` and
   `finalized` → Reth canonicalizes and the state becomes queryable over `eth_*`. No reorg is possible.

State root is computed *during* execution (step 3/5), not after finalization — finalization only commits
an already-executed, already-state-rooted block. This is why both proposer and verifier run full revm.

---

## 4. Custom precompiles (Turing-incomplete, deterministic, consensus-safe)

The injection point is Reth's EVM configuration. Verified API surface (Tempo `crates/evm`,
`crates/revm`, `crates/precompiles`; cross-checked with `reth_evm` rustdocs):

- Implement **`ConfigureEvm`** (verified `impl ConfigureEvm for TempoEvmConfig` in `crates/evm/src/lib.rs`):
  associated types `Primitives`, `Error`, `NextBlockEnvCtx`, `BlockExecutorFactory`, `BlockAssembler`;
  methods `block_executor_factory()`, `block_assembler()`, `evm_env(header)`, `next_evm_env(parent, attrs)`,
  `context_for_block(...)`, `context_for_next_block(...)`.
- `TempoEvmConfig` wraps `EthEvmConfig<TempoChainSpec, TempoEvmFactory>` — i.e. it reuses Reth's Ethereum
  EVM config but swaps in a custom **`EvmFactory`** (`TempoEvmFactory`, from the `tempo-revm` crate).
  `EvmFactory` is "the type responsible for creating EVM instances given a certain input" and is where the
  precompile set is chosen.
- **Precompile registration via revm `PrecompilesMap`** (verified `crates/precompiles/src/lib.rs`):
  ```rust
  pub fn tempo_precompiles(cfg, actions, non_creditable_slots) -> PrecompilesMap {
      let mut precompiles =
          PrecompilesMap::from_static(EthPrecompiles::new(spec).precompiles); // start from std EVM set
      extend_tempo_precompiles(&mut precompiles, …);
      precompiles
  }
  // address-keyed dynamic lookup for the custom ones:
  precompiles.set_precompile_lookup(move |address: &Address| {
      if address.is_tip20() { Some(TIP20Token::create_precompile(*address, &env)) }
      else if *address == TIP403_REGISTRY_ADDRESS { Some(TIP403Registry::create_precompile(&env)) }
      …
  });
  ```
  Individual precompiles are built with a `tempo_precompile!` macro that wraps a closure in
  `DynPrecompile::new_stateful(PrecompileId::Custom(id), |input| …)`, rejects non-direct calls, and runs the
  body inside a `StorageCtx` so the precompile can read/write committed account storage deterministically.
- **Confirmed current API names:** `ConfigureEvm`, `EvmFactory`, `EthEvmConfig`, `EthPrecompiles`,
  `PrecompilesMap`, `PrecompilesMap::from_static`, `set_precompile_lookup`, `DynPrecompile`,
  `PrecompileId::Custom` — all present in Tempo at Reth rev `c0a9431` / current revm.
  **Not confirmed:** that `PrecompilesMap` / `set_precompile_lookup` have these exact names in *older* Reth
  releases — they are revm/Reth-version-specific. Pin and verify.

**Determinism / consensus-safety rules:** precompiles run inside the same `BlockExecutor`/revm path on
both proposer and every verifier, so they are consensus-safe *iff* their output is a pure function of
`(call input, caller, committed state, chain spec)`. Forbid: wall-clock/system time, RNG not derived from
the VRF beacon, filesystem/network I/O, floating-point with platform-dependent rounding, iteration over
unordered maps. Charge deterministic, spec-versioned gas (Tempo gates precompiles by `TempoHardfork` so the
active set changes only at coordinated forks — `SYSTEM_PRECOMPILES` table). Keep them Turing-incomplete (no
unbounded loops over attacker-controlled size without gas metering) so execution cost is bounded and
fraud-checkable.

---

## 5. Build-vs-free checklist (with effort estimates)

Effort is relative engineer-effort for a small team, assuming you fork/study Tempo: **S** ≈ days,
**M** ≈ 1–3 weeks, **L** ≈ 1–2 months, **XL** ≈ multi-month.

### You get for (mostly) free
| Capability | Source | Notes |
|---|---|---|
| EVM execution / revm | Reth | Stock; you only customize via `ConfigureEvm`. |
| Mempool / tx-pool | Reth | Reusable; light wrapping for custom tx types (**S–M**). |
| Payload building | Reth payload builder | Driven via FCU-with-attributes / `canonicalize_and_build`. |
| State, state root, DB, pruning, snapshots | Reth | Also serves as marshal's durable block store. |
| `eth_*` JSON-RPC | Reth RPC add-on | Out of the box; bespoke methods are extra (**S–M**). |
| BFT consensus + sub-second deterministic finality | Commonware `simplex` | Configure, don't build. |
| Threshold BLS certs + VRF beacon | Commonware `bls12381_threshold` | Free succinct certs + randomness. |
| Ordered finalized-block delivery + backfill | Commonware `marshal` + `resolver` | You implement `Blocks`/`Certificates` stores (**M**). |
| Authenticated p2p (Sender/Receiver, blocker) | Commonware `p2p` | Configure channels. |
| Deterministic async runtime (Clock/Spawner/Storage/Metrics) | Commonware `runtime` | Also enables deterministic sim-testing. |

### You must build yourself
| Capability | Effort | Notes |
|---|---|---|
| **Consensus↔Reth glue actor** (the whole Tempo `consensus` crate) | **L–XL** | `Automaton`/`CertifiableAutomaton`/`Relay`/`Reporter` impls, executor actor doing FCU/`new_payload`/build, marshal wiring. The core of the project. |
| **Block type + codec + digest** implementing Commonware `Block` | **M** | Must bridge Reth's block to Commonware `Heightable+Codec+Digestible`. |
| **`Blocks` / `Certificates` stores** (Reth-backed hybrid) | **M** | Tempo's `Hybrid` over `BlockchainProvider` is the template. |
| **Genesis** (chain spec + initial validator set + initial threshold key) | **M** | Reth chain spec + Commonware `floor=Genesis` + DKG/initial-share bootstrap. |
| **Staking / validator-selection module (PoS/DPoS/PoA)** feeding `Supervisor` | **L** | On-chain registry (precompile/contract) → epoch participant set + weights → Commonware `Supervisor`. Tempo: `validator-config` precompile. |
| **Validator-set reconfiguration / DKG / resharing** | **L–XL** | `examples/reshare` pattern + on-chain `OnchainDkgOutcome`; the trickiest correctness/liveness area. |
| **Rewards & slashing** off the `Reporter<Activity>` stream | **M–L** | Threshold sigs are non-attributable, so slashing must use `Activity` reports, not partial sigs. |
| **Governance / upgrades / hardfork gating** | **M–L** | Tempo gates features by `TempoHardfork`; coordinate fork activation across consensus + EVM. |
| **Custom precompiles** | **M each** | Per §4; mostly mechanical once the first one exists. |
| **Bridging** (to Ethereum / other chains) | **XL** | Not provided. Commonware has a `bridge` example/crate for cross-chain *threshold-cert* verification, but a full token bridge is a separate project. |
| **Data availability** (if you need DA beyond full-replication p2p) | **L–XL** | Commonware replicates blocks to validators; external DA layer is your build. |
| **Bespoke RPC / indexer / explorer** | **M–L** | Beyond `eth_*`. |

---

## 6. Proposed component / architecture diagram

```
                          ┌──────────────────────────────────────────────────────────┐
                          │                  YOUR NODE (single process)                │
                          │                                                            │
   peers ───p2p────►  ┌───────────────────────── COMMONWARE ──────────────────────┐   │
   (validators)       │  runtime: Spawner / Clock / Storage / Metrics (det. async) │   │
                      │                                                            │   │
                      │  p2p: authenticated Sender/Receiver, Blocker              │   │
                      │                                                            │   │
                      │  consensus::simplex::Engine<A,R,F,…>  (BLS threshold)      │   │
                      │     ├─ elects leader (VRF beacon)                          │   │
                      │     ├─ Automaton::propose / verify / certify  ◄────┐       │   │
                      │     ├─ Relay::broadcast (block dissemination)      │       │   │
                      │     └─ Finalization threshold certs                │       │   │
                      │                                                    │       │   │
                      │  marshal::core::Actor  ── ordered finalized ───┐   │       │   │
                      │     (+ resolver backfill)   Update::Block(B)   │   │       │   │
                      │  Supervisor ◄── epoch validator set + share     │  │       │   │
                      └─────────────────────────────────────────────────┼──┼──────┘   │
                                                                         │  │          │
   ┌─────────────────────── THE GLUE (you build; cf. Tempo `consensus` crate) ──────┐  │
   │  application actor:  impl Automaton/CertifiableAutomaton/Relay              │  │  │
   │  executor actor:     holds reth ConsensusEngineHandle<T>                    │  │  │
   │     propose():  canonicalize_and_build(parent,attrs) ──► getPayload         │  │  │
   │     verify():   engine.new_payload(execution_data)  ──► PayloadStatus       │  │  │
   │     finalize(): forkchoice update {head, safe, finalized}                   │  │  │
   │     Reporter<Update<Block>> ──► forkchoice(finalized) ; Blocks/Certs store  │◄─┘  │
   └────────────────────────────────────────┬────────────────────────────────────┘   │
                                             │ in-process (no JSON-RPC)                 │
   ┌───────────────────────── RETH SDK (embedded; Engine API RPC = Noop) ────────────┐ │
   │  tx-pool (mempool)  ──►  payload builder  ──►  BlockExecutor (revm)             │ │
   │  ConfigureEvm + EvmFactory + PrecompilesMap (your custom precompiles)           │ │
   │  state / state-root / MDBX DB  ◄── also marshal's durable Blocks store          │ │
   │  eth_* JSON-RPC  ──────────────────────────────────────────────► dApps/wallets │ │
   └─────────────────────────────────────────────────────────────────────────────────┘ │
                          └──────────────────────────────────────────────────────────┘
```

---

## Sources

- [Commonware consensus crate (docs.rs)](https://docs.rs/commonware-consensus) — Automaton, Relay, Reporter, Monitor, Block, Application, modules (simplex/threshold_simplex/aggregation/marshal/ordered_broadcast).
- [`Automaton` trait (docs.rs)](https://docs.rs/commonware-consensus/latest/commonware_consensus/trait.Automaton.html) — propose/verify signatures.
- [`Reporter` trait (docs.rs)](https://docs.rs/commonware-consensus/latest/commonware_consensus/trait.Reporter.html) — Activity / report.
- [`CertifiableAutomaton` (docs.rs)](https://docs.rs/commonware-consensus/latest/commonware_consensus/trait.CertifiableAutomaton.html)
- [simplex module (docs.rs)](https://docs.rs/commonware-consensus/latest/commonware_consensus/simplex/index.html)
- [Commonware monorepo (GitHub)](https://github.com/commonwarexyz/monorepo) — `consensus/`, `runtime/`, `p2p/`, `glue/`, `examples/reshare/`, `pipeline/minimmit/minimmit.md` (read at source level on 2026-06-25).
- [commonware.xyz — the Anti-Framework](https://commonware.xyz/blogs/commonware-the-anti-framework)
- [commonware.xyz — Many-to-Many Interoperability with Threshold Simplex](https://commonware.xyz/blogs/threshold-simplex)
- [Alto — minimal Commonware blockchain](https://alto.commonware.xyz/)
- [Tempo (GitHub) — Reth SDK + Commonware Simplex EVM L1](https://github.com/tempoxyz/tempo) — reference implementation; crates `consensus`, `evm`, `revm`, `precompiles`, `payload`, `transaction-pool`, `node`, `validator-config`, `dkg-onchain-artifacts` (read at source level on 2026-06-25).
- [Tempo docs — Performance / architecture](https://docs.tempo.xyz/learn/tempo/performance)
- [Seismic "Summit" — Commonware-based consensus client over Engine API (GitHub)](https://github.com/SeismicSystems/summit)
- [Reth for Developers — SDK](https://reth.rs/sdk/)
- [Reth — EVM Component (SDK)](https://reth.rs/sdk/node-components/evm/)
- [`reth_evm` (rustdoc)](https://reth.rs/docs/reth_evm/index.html) — ConfigureEvm / EvmFactory / BlockExecutor(Factory) / precompiles helpers.
- [`reth_node_builder` (rustdoc)](https://reth.rs/docs/reth_node_builder/index.html) — NodeBuilder, ConsensusEngineHandle.
- [paradigmxyz/reth (GitHub)](https://github.com/paradigmxyz/reth) — Tempo pins rev `c0a9431`.
- [Decipher Media — Inside Commonware](https://medium.com/decipher-media/inside-commonware-50c58211953c)

---

### Confidence & caveats
- **High confidence (read at source):** Automaton/Relay/Reporter/Monitor/Block/Application trait shapes;
  marshal `Update`/`Blocks`/`Certificates`; Tempo's embedded-SDK approach, `ConsensusEngineHandle.new_payload`,
  forkchoice-update finalization, `ConfigureEvm`/`EvmFactory`/`PrecompilesMap` precompile injection, custom
  tx-pool, on-chain validator-config + `OnchainDkgOutcome`.
- **Version-sensitive / flagged:** the `threshold_simplex` *module path* (now organized as a `Scheme` under
  `simplex/scheme/bls12381_threshold`; the standalone docs.rs page 404'd); `Automaton::genesis` (present in
  Tempo source, omitted on the docs.rs trait page); exact `Supervisor`/`ThresholdSupervisor` method names
  (not line-verified this pass). Pin a specific Commonware + Reth revision and re-verify these before coding.
- Commonware consensus is self-described **ALPHA** — expect breaking changes.
```