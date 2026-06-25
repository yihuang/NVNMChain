# Turing-Incomplete Native Modules Exposed as EVM Precompiles — A Technical Reference

**Last verified: 2026-06-25**

> **Scope.** The concept, mechanics, and design discipline of exposing *bounded, deterministic*
> native (non-EVM-bytecode) modules to Solidity as precompiles, and how this works across the
> candidate EVM stacks. Written for a team building a **sovereign stablecoin EVM L1 on
> Reth + revm + Commonware (Tempo-bootstrapped)**, so the Reth/revm path is weighted most heavily
> and the others are treated as contrast.
>
> **Sourcing.** Reuses already-verified facts from five companion research files in this directory
> (cited inline as `[glue]`, `[tempo]`, `[cosmos]`, `[avax]`, `[frontier]`), supplemented by
> primary-source verification of revm/Reth, the standard Ethereum precompile set, and Arbitrum's
> precompile restrictions on 2026-06-25. Where an exact API name could not be confirmed against
> current source, it is flagged **[UNVERIFIED]** or **[version-sensitive]** rather than guessed.

---

## Executive summary

- A **precompile** is native code (compiled Rust/Go, not EVM bytecode) mounted at a fixed 20-byte
  address. Solidity reaches it with an ordinary `CALL`/`STATICCALL` carrying ABI-encoded calldata;
  the node intercepts the address, runs the native function, charges gas, and returns ABI-encoded
  bytes (or reverts). Ethereum's own cryptographic primitives — `ecrecover` (`0x01`) … the
  BLS12-381 suite (`0x0b`–`0x11`, Prague) — are precompiles.

- **"Turing-incomplete" is a property of the *exposed function*, not the implementation language.**
  The implementation is Turing-complete Rust/Go, but the function you expose must be **bounded**
  (its work is capped by a gas charge computed before/independent of attacker-controlled iteration),
  **total** (it always halts), and **deterministic** (a pure function of `(input [+ committed
  state])` with no wall-clock, RNG, floats, I/O, or map-iteration nondeterminism). This is what makes
  a precompile safe for **gas-metering/DoS**, **consensus determinism** (identical result on every
  node and every replay), and gives it its **native-performance** edge over equivalent Solidity.

- **Stateless vs stateful.** Stateless precompiles are pure `input → output` (the Ethereum set;
  Tempo's signature verifier for secp256k1/P256/WebAuthn). Stateful precompiles read/mutate committed
  state via the host's state interface (Avalanche's "stateful precompiles"; Cosmos module precompiles
  over keepers; Tempo's FeeManager/TIP-20 moving balances). Stateful + reentrant is the **highest-risk**
  category — every disclosed precompile CVE below is stateful.

- **Reth/revm (your path).** Precompiles are `fn(&[u8], u64, u64) -> PrecompileResult` (input,
  gas_limit, reservoir) registered in a `Precompiles` map keyed by address, injected via
  `ConfigureEvm` → `EvmFactory` and revm's `PrecompilesMap` (`from_static(EthPrecompiles…)` +
  `set_precompile_lookup(...)`), and run inside the deterministic `BlockExecutor`. Tempo proves the
  full pattern, including stateful precompiles via `DynPrecompile::new_stateful` inside a `StorageCtx`
  `[glue][tempo]`.

- **Design rules that matter most:** gas must over-charge the worst case; forbid all nondeterminism;
  treat state-mutating precompiles as reentrancy-attack surface (Arbitrum *forbids* anything beyond
  view/`eth_call`-safe to protect fraud proofs); **gate every add/change behind a hardfork activated
  at a height**; allocate addresses in a distinct app range away from Ethereum's reserved low range
  and future-EIP collisions; and remember that a precompile change is a **coordinated node upgrade**,
  unlike an upgradeable system contract.

- **Recommendation for this L1:** expose as precompiles the *enshrined, hot-path, hard-to-express-in-EVM*
  modules — stablecoin **fee manager / fee-AMM**, **signature verification** (P256/WebAuthn), **oracle
  price reads**, **validator-config** read surface, and any **custom hash / ZK-verify**. Keep
  *governance-mutable, business-policy* logic — KYC/compliance allowlist *rules*, token issuance
  parameters, upgrade admin — as **upgradeable system/predeploy contracts** so they evolve without a
  node release.

---

## 1. What a precompile is

A **precompile** (precompiled contract) is a function implemented in the client's native language and
mounted at a **fixed, well-known address** in the EVM address space. From a contract's point of view it
is indistinguishable from a normal contract: you `CALL` or `STATICCALL` the address with ABI-encoded
calldata and read back ABI-encoded return data. The difference is entirely under the hood — instead of
loading account bytecode and running it on the EVM interpreter, the client **intercepts the address**,
runs compiled native code, deducts a gas amount the code declares, and returns bytes (or signals a
revert/failure).

Because there is no bytecode, a precompile can do things EVM bytecode *cannot do cheaply or at all*:
big-integer modular exponentiation, elliptic-curve pairings, alternative signature schemes, native
hash functions — anything that would be ruinously expensive (or impossible within block gas limits) as
hand-rolled Solidity/Yul.

### The canonical Ethereum precompiles (mainnet, post-Prague)

Verified against the standard precompile set and go-ethereum/revm layout (addresses are left-padded to
20 bytes; e.g. `0x01` = `0x00…0001`):

| Addr | Name | Introduced | What it does |
|---|---|---|---|
| `0x01` | **ecRecover** | Frontier | Recover secp256k1 signer address from a signature |
| `0x02` | **SHA-256** | Frontier | SHA2-256 hash |
| `0x03` | **RIPEMD-160** | Frontier | RIPEMD-160 hash |
| `0x04` | **identity** | Frontier | Returns input unchanged (datacopy) |
| `0x05` | **modexp** | Byzantium (EIP-198) | Arbitrary-precision modular exponentiation |
| `0x06` | **ecAdd** (alt_bn128/BN254) | Byzantium (EIP-196) | BN254 point addition |
| `0x07` | **ecMul** (alt_bn128) | Byzantium (EIP-196) | BN254 scalar multiplication |
| `0x08` | **ecPairing** (alt_bn128) | Byzantium (EIP-197) | BN254 pairing check (zk-SNARK verification) |
| `0x09` | **blake2f** | Istanbul (EIP-152) | BLAKE2b compression-function round |
| `0x0a` | **point evaluation (KZG)** | Cancun (EIP-4844) | Verify a KZG opening (blob commitment) |
| `0x0b`–`0x11` | **BLS12-381 suite** | Prague (EIP-2537) | G1 add (`0x0b`), G1 MSM (`0x0c`), G2 add (`0x0d`), G2 MSM (`0x0e`), pairing (`0x0f`), map-FP-to-G1 (`0x10`), map-FP2-to-G2 (`0x11`) |

(Note: revm's `secp256r1`/P-256 verify precompile, EIP-7212, is implemented in the crate but its
mainnet-activation address/fork status varies; treat its address as fork-configured, not a fixed mainnet
slot. **[version-sensitive]**)

Every one of these is **stateless** — a pure function of its input — and every one is the canonical
illustration of a Turing-incomplete native module: bounded work, fixed gas formula, deterministic output.

Sources: [EIP-2537 (BLS12-381)](https://eips.ethereum.org/EIPS/eip-2537),
[go-ethereum contracts.go](https://github.com/ethereum/go-ethereum/blob/master/core/vm/contracts.go),
[EVM Codes — precompiles](https://www.evm.codes/precompiled),
[RareSkills — Ethereum precompiles](https://rareskills.io/post/solidity-precompiles).

---

## 2. Why "Turing-incomplete" matters

The implementation language is Turing-complete (Rust, Go). The *contract* the precompile makes with the
chain is not: the **exposed function must be bounded, total, and deterministic**. Concretely:

1. **It must declare and charge a gas cost that bounds its work.** A precompile cannot contain an
   unbounded loop whose iteration count is driven by caller input without a *proportional, pre-declared*
   gas charge. Ethereum's precompiles encode this directly: `modexp`, `ecPairing`, the BLS MSM ops all
   compute gas as a function of input size/word-count *first*, so cost scales with — and over-bounds —
   work. The discipline is "Turing-incomplete" in the sense that **execution cost is statically
   bounded by the gas you charge**; there is no way for a caller to make it run arbitrarily long for a
   fixed fee.

2. **It must halt.** No "maybe-non-terminating" code paths. Total functions only.

3. **It must be a pure function of its inputs** — for a stateless precompile, `output = f(input)`; for a
   stateful one, `output, Δstate = f(input, caller, committed state, chainspec)`. Nothing else may
   influence the result.

These three properties buy three distinct things:

- **(a) Gas-metering / DoS safety.** If cost is bounded by a pre-charged gas formula, an attacker
  cannot craft an input that consumes disproportionate node CPU/memory for cheap gas. The gas formula
  *is* the DoS bound. (This is precisely the failure mode behind historical "gas-mispricing" issues
  on EVM chains — when a precompile's real cost outran its charged gas.)

- **(b) Consensus determinism.** Every validator re-executes every block; a node re-executes on replay
  and on state-sync. The precompile **must return bit-identical results everywhere, every time**. That
  forbids: wall-clock/system time, RNG not derived from an in-protocol beacon, filesystem/network I/O,
  floating-point with platform-dependent rounding, iteration over unordered maps/hashsets, and any
  uninitialized-memory or pointer-address leakage. On a Commonware+Reth chain a precompile runs inside
  the same revm `BlockExecutor` path on proposer and every verifier, so it is consensus-safe *iff* it
  is this kind of pure function `[glue]`.

- **(c) Native performance.** A pairing check or P256 verification in compiled Rust is orders of
  magnitude faster and cheaper than the equivalent EVM bytecode — often the *only* feasible way to do
  the operation within block gas limits. This is the entire reason precompiles exist.

A useful mental model: **the precompile body is written in a Turing-complete language, but you are
contractually promising the chain that the function it exposes behaves like a primitive recursive /
total function with a known cost bound.** Breaking that promise breaks DoS-safety or consensus.

---

## 3. Stateless vs stateful precompiles

| | **Stateless** | **Stateful** |
|---|---|---|
| Signature | `output = f(input)` | `output, Δstate = f(input, caller, committed state)` |
| Examples | All Ethereum standard precompiles; **Tempo signature verifier** (secp256k1/P256/WebAuthn) `[tempo]`; Cosmos P256 (`0x…0100`) and Bech32 (`0x…0400`) `[cosmos]` | **Avalanche** "stateful precompiles" (native minter, fee manager) `[avax]`; **Cosmos** module precompiles over keepers (staking, bank, ICS-20…) `[cosmos]`; **Tempo** FeeManager/TIP-20 moving balances `[tempo]` |
| Determinism source | Input only | Input + the *committed* pre-state of the block, which is itself agreed by consensus |
| Risk surface | Low — same as any pure crypto primitive | High — can read/mutate balances and storage, can be re-entered, can interleave with EVM calls |

**Stateless** is the simplest and safest: it cannot touch the world, so the only correctness concerns
are gas pricing and the determinism rules of §2.

**Stateful** precompiles are the powerful, dangerous category. They are given a handle to the host's
committed state (revm's `StorageCtx`/journaled state `[glue]`, Cosmos keepers + `StateDB` `[cosmos]`,
Avalanche's state-access interface `[avax]`) and can both **read** (oracle price, allowlist membership,
validator set) and **mutate** (transfer balances, set storage). They remain Turing-incomplete in the
§2 sense — bounded, total, deterministic — but they add three hazards:

1. **Reentrancy / nested-execution.** A stateful precompile that itself triggers further EVM execution
   (or is re-entered mid-call) can violate the checks-effects-interactions assumptions of the calling
   contract or of the precompile itself. **This is exactly how Cosmos EVM's ASA-2026-002 worked:**
   incorrect state handling during *nested EVM execution paths involving the ICS-20 precompile* let
   tokens be reused within a single transaction — a **$7M loss on Saga**, patched in v0.6.0 `[cosmos]`.

2. **Partial state writes on error.** If a stateful precompile writes state and then errors without
   atomically reverting, it leaves committed garbage. Cosmos EVM's ASA-2025-004 ("Partial Precompile
   State Writes") was exactly this — fixed by wrapping precompile execution atomically
   (`RunAtomic`/`RevertMultiStore`) `[cosmos]`. Tempo mitigates structurally by running each precompile
   body inside a `StorageCtx` so state effects are journaled with the surrounding EVM frame `[glue]`.

3. **Read-only enforcement.** Under `STATICCALL` a stateful precompile must refuse to mutate (honour the
   `readOnly`/`is_static` flag), exactly as Solidity `view` is enforced. Tempo's precompiles also
   *reject non-direct calls* in certain cases to constrain the call surface `[glue]`.

**Both critical Cosmos EVM advisories in the past ~12 months were precompile bugs** `[cosmos]`, and
Moonbeam has shipped urgent patches for custom-precompile bugs `[frontier]`. The lesson: stateful
precompiles concentrate risk; treat them like privileged kernel modules.

---

## 4. Mechanics per stack

### 4.1 revm / Reth — PRIMARY

**The precompile function signature.** In current revm the precompile function alias is (verified on
docs.rs, `revm-precompile` 41.x):

```rust
// revm_precompile::interface  (verified 2026-06-25)
pub type PrecompileFn = fn(&[u8], u64, u64) -> PrecompileResult;
//                          input  gas    reservoir
```

The third `u64` ("reservoir") is newer than the classic two-argument form (`fn(&[u8], u64) ->
PrecompileResult`) you'll see in older revm and in most blog posts — **pin and check the arity against
your revm rev**. `PrecompileResult` resolves to `Ok(PrecompileOutput)` or a fatal `PrecompileError`;
`PrecompileOutput` is the "rich execution output with gas accounting" carrying the **gas used**, the
**return bytes**, and (re-added after a brief removal) a **`gas_refunded`** field. A precompile signals
"out of gas" / failure through the result type rather than by panicking.
([revm-precompile docs](https://docs.rs/revm-precompile/latest/revm_precompile/),
[interface module](https://docs.rs/revm-precompile/41.0.0/revm_precompile/interface/index.html).)

**The registry.** revm exposes a `Precompiles` struct: an address→precompile map (`inner`) plus the set
of active addresses (`addresses`), with `get`/`contains` lookups by `Address`. Reth re-exports it and
wraps it in a mutable **`PrecompilesMap`** so a chain can start from the standard Ethereum set and add/
override entries. ([reth precompile module](https://reth.rs/docs/reth/revm/revm/precompile/index.html).)

**Injecting custom ones (the Tempo pattern, source-verified in `[glue]`/`[tempo]`).** The injection
point is Reth's EVM configuration:

1. Implement **`ConfigureEvm`** for your config type (Tempo: `impl ConfigureEvm for TempoEvmConfig`),
   which carries the `BlockExecutorFactory`, `BlockAssembler`, and EVM-env construction.
2. `TempoEvmConfig` wraps `EthEvmConfig<…, TempoEvmFactory>` — it reuses Reth's Ethereum EVM config but
   swaps in a custom **`EvmFactory`** ("the type responsible for creating EVM instances"), which is
   where the precompile set is chosen.
3. Build the set with revm's **`PrecompilesMap`**: start from the standard set
   (`PrecompilesMap::from_static(EthPrecompiles::new(spec).precompiles)`), then register address-keyed
   custom precompiles via **`set_precompile_lookup(|address| …)`**.
4. Individual precompiles are wrapped with **`DynPrecompile::new_stateful(PrecompileId::Custom(id),
   |input| …)`** (Tempo's `tempo_precompile!` macro), running the body inside a **`StorageCtx`** so it
   can read/write committed account storage deterministically, and rejecting disallowed call shapes.
5. Everything runs inside the deterministic **`BlockExecutor`** on both proposer and verifiers, so it is
   consensus-safe as a pure function of `(input, caller, committed state, chainspec)` `[glue]`.

**Confirmed current API names** (Tempo @ Reth rev `c0a9431` / current revm): `ConfigureEvm`,
`EvmFactory`, `EthEvmConfig`, `EthPrecompiles`, `PrecompilesMap`, `PrecompilesMap::from_static`,
`set_precompile_lookup`, `DynPrecompile`, `PrecompileId::Custom` `[glue]`. **Not confirmed:** that these
exact names exist in *older* Reth/revm releases — they are revm/Reth-version-specific; pin and verify.
The 3-arg `PrecompileFn` "reservoir" parameter is verified against `revm-precompile` 41.x docs but its
presence in the exact rev Tempo pins was not line-checked this pass **[version-sensitive]**.

#### Illustrative code (labelled — NOT line-verified; shapes match `[glue]`/docs)

```rust
// ILLUSTRATIVE — pin your revm/Reth rev and verify exact arities & names.

use revm::precompile::{PrecompileResult, PrecompileOutput, PrecompileError};

// A stateless precompile: pure input -> output, with an UP-FRONT gas charge.
fn price_decimals_precompile(input: &[u8], gas_limit: u64 /*, reservoir: u64 */) -> PrecompileResult {
    const GAS: u64 = 2_000;                 // bounded, pre-declared cost (§2)
    if gas_limit < GAS {
        return Err(PrecompileError::OutOfGas);
    }
    // deterministic, total, bounded work only — no clock/RNG/IO/floats (§2)
    let out = do_bounded_pure_work(input);  // e.g. fixed-size decode + table lookup
    Ok(PrecompileOutput::new(GAS, out.into()))
}

// Registration via Reth's ConfigureEvm -> EvmFactory -> PrecompilesMap (Tempo pattern, [glue]):
fn my_precompiles(spec: Spec) -> PrecompilesMap {
    let mut p = PrecompilesMap::from_static(EthPrecompiles::new(spec).precompiles); // std set first
    p.set_precompile_lookup(move |addr: &Address| {
        if *addr == ORACLE_PRICE_ADDR {
            // stateful variant wraps a closure with access to committed state:
            Some(DynPrecompile::new_stateful(PrecompileId::Custom(/*id*/ 0x01), |ctx, input| {
                // read committed storage via ctx (StorageCtx), charge gas, return bytes
                oracle_read(ctx, input)
            }))
        } else {
            None
        }
    });
    p
}
```

Sources: [reth_evm rustdoc](https://reth.rs/docs/reth_evm/index.html),
[Reth SDK — EVM component](https://reth.rs/sdk/node-components/evm/),
[revm releases](https://github.com/bluealloy/revm/releases); mechanics cross-checked against
`commonware-reth-glue.md` and `tempo-as-blueprint.md` in this directory.

### 4.2 Cosmos EVM (`cosmos/evm`)

The geth-style stateful precompile interface (verified in `[cosmos]`):

```go
// implement on top of cmn.Precompile (github.com/cosmos/evm/precompiles/common)
Address() common.Address                                          // fixed registration address
RequiredGas(input []byte) uint64                                  // cost computed BEFORE Run
Run(evm *vm.EVM, contract *vm.Contract, readOnly bool) ([]byte, error)
```

Key mechanics:

- **`RequiredGas` is computed from the input *before* `Run` executes** — cost derives from the Cosmos
  SDK gas config (flat cost + per-byte × input length, keyed off the method ID, tx vs query). This is
  the §2 "bounded, pre-charged" discipline made explicit at the interface level: the EVM knows the cost
  before it commits to running the work `[cosmos]`.
- **State access via keepers.** Each stateful precompile holds a reference to its module's keeper
  (staking, bank, distribution, gov, slashing, ICS-20, ICS-02, ERC20/WERC20). `Run` decodes the ABI
  input, builds the corresponding SDK message/query, invokes the keeper against the multistore **within
  the same EVM transaction**, and ABI-encodes the result. EVM and Cosmos state commit atomically
  `[cosmos]`.
- **Registration is a chain-level (genesis/upgrade) operation**, not something a contract author can do
  at runtime `[cosmos]`.
- **Cautionary CVEs.** ASA-2025-004 (partial precompile state writes; fixed via atomic
  `RunAtomic`/`RevertMultiStore`) and **ASA-2026-002 (ICS-20 nested-execution/reentrancy → $7M Saga
  loss; patched v0.6.0)** are both precompile bugs — the canonical warning that stateful precompiles are
  the principal risk surface `[cosmos]`.

### 4.3 Avalanche Subnet-EVM

- **Stateful precompiles**: native Go at a reserved address (built-ins occupy
  `0x0200000000000000000000000000000000000000`–`…0005`), with access to EVM state, callable from
  Solidity like a normal contract `[avax]`.
- **`precompilegen` from an ABI**: define the Solidity interface + ABI, run
  `./scripts/generate_precompile.sh --abi <abi> --out <dir>` to scaffold a self-registering Go module,
  register it in `plugin/main.go` (via `precompile/modules`), rebuild. The **`precompile-evm`** library
  repo lets you ship custom precompiles **without forking** subnet-evm `[avax]`.
- **Activation via ChainConfig/upgrade.** Which precompiles are active is set in `genesis.json`
  `chainConfig`, and they can be turned **on/off later via timestamped network-upgrade config** — no
  binary hard-fork required (an explicit, height/timestamp-gated consensus change) `[avax]`.
- **Built-ins:** Contract Deployer AllowList, Transaction AllowList, Native Minter, Fee Manager, Reward
  Manager, Warp Messenger (Interchain Messaging). The AllowList family defines Admin/Manager/Enabled
  roles `[avax]`.

### 4.4 Frontier / Moonbeam (Substrate)

- A Frontier precompile implements an `execute`-style entry returning the gas used, registered in the
  runtime's `PrecompileSet` at a chosen address `[frontier]`.
- **`precompile-utils` + the `#[precompile]` proc-macro** (Moonbeam) remove the boilerplate of ABI
  encode/decode, selector matching, gas costing, and error handling: define the Solidity interface,
  annotate Rust methods with `#[precompile]`, call into the target pallet's dispatchable/storage with
  gas/weight accounting, register at an address `[frontier]`.
- **`pallet-evm-precompile-dispatch`** is the generic bridge: an EVM contract can dispatch an arbitrary
  Substrate runtime `Call` (any pallet extrinsic) from inside the EVM through a single precompile
  address — the broadest "reach the native runtime from Solidity" mechanism of any stack here
  `[frontier]`.
- Built-in precompiles include the standard Ethereum set plus `modexp`, `bn128`, `blake2`, `ed25519`,
  `curve25519`, `bls12377`, `bls12381`, `bw6761`, and `dispatch` `[frontier]`. Custom precompiles have
  been a real audit burden — Moonbeam shipped urgent patches `[frontier]`.

---

## 5. Design rules & pitfalls

These are the load-bearing rules. Most production precompile incidents trace to violating one of them.

1. **Gas must over-estimate the worst case.** Compute the charge from input size / iteration count
   *before* (or independently of) doing the work, and round up. The gas formula is your DoS bound (§2a).
   Cosmos's `RequiredGas`-before-`Run` and Ethereum's size-keyed `modexp`/MSM formulas are the model.
   A precompile whose real cost can exceed its charged gas is a node-DoS / chain-halt vector.

2. **Strict determinism — no exceptions.** No wall-clock/system time, no RNG except an in-protocol
   beacon value, no filesystem/network I/O, no floats with platform-dependent rounding, no iteration
   over unordered maps, no pointer/uninitialized-memory leakage. Must be bit-identical on every node and
   every replay/state-sync (§2b) `[glue]`.

3. **Reentrancy / call-back hazards.** State-mutating precompiles are the highest-risk category. If a
   precompile triggers further EVM execution or can be re-entered, audit it as hard as a Solidity
   protocol contract. **Arbitrum's stance is the strongest signal:** custom precompiles there are
   restricted to **view/`eth_call`-safe** behaviour — *"this approach will break the block validation if
   you call them from other contracts or add non-view/pure methods"* — because state-touching custom
   precompiles would corrupt the deterministic re-execution that fraud proofs rely on
   ([Arbitrum — customize precompile](https://docs.arbitrum.io/launch-arbitrum-chain/customize-your-chain/customize-precompile)).
   On a BFT chain (Commonware) you don't have Arbitrum's fraud-proof constraint, but the determinism
   requirement is identical — and the Cosmos ICS-20 reentrancy CVE shows the cost of getting it wrong
   `[cosmos]`. Make precompile state effects **atomic** (journaled with the EVM frame, reverted on
   error) and honour `STATICCALL`/`readOnly`.

4. **Hardfork/upgrade-gating.** Enabling, disabling, or changing the semantics or gas of a precompile is
   a **consensus change** and must be activated at a specific block height/timestamp, identically across
   all nodes. Tempo gates its active precompile set by `TempoHardfork` (a `SYSTEM_PRECOMPILES` table that
   changes only at coordinated forks) `[glue][tempo]`; Avalanche uses timestamped upgrade config
   `[avax]`; Cosmos via chain upgrades `[cosmos]`. Never "just deploy" a precompile change — schedule it.

5. **Address allocation.** Do **not** collide with Ethereum's reserved low range (`0x01`–`0x11` today,
   and *future* EIP precompiles will keep climbing — leave headroom). Pick a distinct, documented
   application range. Real examples: **Tempo** uses high vanity-ish slots — FeeManager `0xfeec…`, TIP-20
   Factory `0x20FC…`, Stablecoin DEX `0xdec0…`, Signature Verifier `0x5165…`, Account Keychain
   `0xAAAA…`, plus a `VALIDATOR_CONFIG` precompile `[tempo][glue]`; **Avalanche** uses the
   `0x0200…00xx` block `[avax]`; **Cosmos** uses `0x…08xx` for modules and `0x…01xx`/`0x…04xx` for
   stateless helpers `[cosmos]`; **Arbitrum** uses the `0x64…` range. Document the range in your chain
   spec and reserve it.

6. **Upgradeability = coordinated node upgrades.** A precompile lives in the node binary. Changing it
   means shipping a new binary to every validator and activating at a fork height. This is heavier than
   upgrading a contract (see §6) and is the main reason to reserve precompiles for *stable, enshrined*
   logic and push *frequently-changing policy* into upgradeable system contracts.

7. **Test the determinism, not just the happy path.** Differential-test the precompile across
   architectures; fuzz for inputs that blow the gas bound; replay historical blocks to confirm identical
   output. Commonware's deterministic simulation runtime makes this tractable for your stack `[glue]`.

---

## 6. Precompiles vs alternatives

There are three distinct extensibility axes. They are complementary, not mutually exclusive.

| | **(a) Precompile** | **(b) System / predeploy contract** | **(c) Alternative VM** |
|---|---|---|---|
| What it is | Native (Rust/Go) code at a fixed address | **Upgradeable EVM bytecode** at a fixed address (deployed at genesis / via upgrade) | A second VM alongside the EVM (WASM/RISC-V) |
| Lives in | The **node binary** | **Chain state** (it's just account code) | The node binary (new VM engine) |
| Can do non-EVM things (pairings, alt-sigs, native state)? | **Yes** | **No** — limited to EVM-expressible logic | **Yes** (compute-heavy, multi-language) |
| Performance | **Native, fastest** | EVM-speed | Near-native (WASM/JIT) |
| To change it | **Coordinated node upgrade + hardfork gate** | **On-chain upgrade** (proxy/governance) — no node release | Node upgrade |
| Consensus-change risk | High (node-level) | Low (state-level) | High (node-level) |
| Reentrancy/determinism burden | High for stateful | Same as any contract | High (must be deterministic + fraud-provable) |
| Examples | Ethereum crypto precompiles; Cosmos/Avalanche/Tempo modules | **OP-Stack predeploys** (`L1Block`, `GasPriceOracle`); **BeaconKit / beacon-deposit** system contracts; OpenZeppelin upgradeable proxies | **Arbitrum Stylus** (WASM, MultiVM); **PolkaVM/PVM** in `pallet-revive` `[frontier]` |

**When to use which:**

- **Precompile** when the logic is (i) *enshrined* protocol behaviour, (ii) on a hot path where
  EVM-cost is prohibitive, (iii) needs primitives EVM can't express (new curves, hashes, signature
  schemes, direct native-state access), and (iv) changes *rarely* and only at coordinated forks. Crypto
  primitives, fee engines, signature verifiers, validator-config reads.

- **System / predeploy contract** when the logic is *expressible in EVM* and you want to **upgrade it
  without a node release** — business policy, parameters, registries, admin. This is the OP-Stack /
  BeaconKit approach: ship EVM bytecode at a fixed address, upgrade it via governance/proxy. Lower
  consensus risk; no validator coordination to change behaviour. The trade-off is EVM-speed and
  EVM-expressible-only.

- **Alternative VM** when you need a *general* high-performance or multi-language compute surface for
  *user* code, not a fixed set of protocol primitives — Stylus (WASM) lets users *deploy their own*
  "precompile-like" libraries (e.g. a novel ZK curve) as ordinary contracts, sidestepping the node-
  upgrade requirement entirely; PolkaVM is Polkadot's RISC-V analogue `[frontier]`. This is a different
  axis: it democratizes native-speed extensibility to users rather than reserving it to chain operators.

For a sovereign EVM L1, the pragmatic split is: **precompiles for the handful of enshrined primitives,
upgradeable system contracts for everything policy-shaped, and an alternative VM only if you actually
need a general non-EVM compute surface (most stablecoin L1s do not).**

Sources: [Arbitrum Stylus — gentle intro](https://docs.arbitrum.io/stylus/gentle-introduction),
[Arbitrum — customize precompile](https://docs.arbitrum.io/launch-arbitrum-chain/customize-your-chain/customize-precompile),
plus `polkadot-frontier.md` (PolkaVM/`pallet-revive`) in this directory.

---

## 7. Recommended modules for THIS project (a sovereign stablecoin L1)

Built on Reth + revm + Commonware (Tempo-bootstrapped). The decision for each module is **precompile**
(enshrined, hot, hard-in-EVM, rarely-changing) vs **upgradeable system contract** (policy, parameters,
frequently-changing).

| Module | Recommendation | Why | Stateful? | Reference |
|---|---|---|---|---|
| **Stablecoin fee manager / fee-AMM (R5)** | **Precompile** | Runs on every tx / at block end; moves balances and batches fee→preferred-token swaps; cost-sensitive hot path; benefits from native speed and MEV-safe batch ordering | **Yes (high-risk)** — journal effects, make atomic, honour staticcall | Tempo FeeManager `0xfeec…` + block-end batch swap `[tempo][glue]` |
| **Signature verification (P256 / WebAuthn / secp256k1)** for smart accounts | **Precompile** | Pure crypto, impossible-to-cheap in EVM, needed for passkey/AA flows; the textbook precompile | **No** (stateless) | Tempo signature verifier `0x5165…` `[tempo]`; EIP-7212 P256; revm `secp256r1` |
| **Oracle / price reads** | **Precompile (read-only)** | Hot read path; deterministic read of committed price state; keep it `view`-only (no mutation) so it's the low-risk stateful kind | Stateful **read-only** | revm `DynPrecompile::new_stateful` over `StorageCtx`, read-only `[glue]` |
| **KYC / compliance allowlist (RWA)** — *enforcement check* | **Precompile (read-only)** for the hot membership check | Every transfer may need an `isAllowed(addr)` read; native read is cheap and deterministic | Stateful **read-only** | cf. Avalanche AllowList roles `[avax]`; Cosmos ERC20 hooks `[cosmos]` |
| **KYC / compliance allowlist (RWA)** — *the rules / admin* | **Upgradeable system contract** | The *policy* (who can add, jurisdiction logic, parameters) changes often and is EVM-expressible; upgrade via governance without a node release | n/a | OP-Stack predeploy pattern; OZ upgradeable proxy |
| **Staking / validator-config** — *read surface for the Supervisor* | **Precompile (read-only)** | Commonware's `Supervisor` reads the epoch validator set; a precompile makes the set a contract-readable on-chain object | Stateful **read-only** | Tempo `VALIDATOR_CONFIG` precompile `[glue]` |
| **Staking / validator-config** — *registration & governance rules* | **Upgradeable system contract** | Onboarding rules, slashing parameters, signatures — policy that evolves; EVM-expressible | n/a | Tempo `ValidatorConfig` / on-chain DKG artifacts pattern `[glue]` |
| **Custom hashing / ZK-verify** (if your RWA/compliance design needs it) | **Precompile** | Pure crypto, prohibitive in EVM; classic precompile | **No** (stateless) | BN254/BLS standard precompiles; custom curve as precompile |
| **Token issuance parameters, fee schedules, upgrade admin** | **Upgradeable system contract** | Pure policy/parameters; must change without forking the node | n/a | system-contract / governance |

**Rules of thumb for this L1:**

- **Anything that moves balances or mutates state inside a precompile (fee manager, mint/burn) is the
  highest-risk surface** — give it the most audit, make its state effects atomic and journaled, honour
  `STATICCALL`, and watch for the ICS-20-style nested-execution reentrancy that cost Saga $7M `[cosmos]`.
- **Prefer read-only stateful precompiles** (oracle reads, allowlist checks, validator-set reads) over
  mutating ones wherever you can — they get most of the benefit at a fraction of the risk.
- **Keep policy in upgradeable contracts.** Compliance rules, parameters, and admin will change far more
  often than you want to ship validator binaries. Reserve precompiles for the stable, enshrined,
  performance-critical primitives, and gate every precompile change behind a Commonware-coordinated
  hardfork at a height `[glue][tempo]`.
- **Reserve a documented address range** away from `0x01`–`0x11` and future-EIP headroom; follow Tempo's
  high-slot convention so a future Ethereum precompile EIP can never collide `[tempo]`.

---

## Sources

Primary / verified 2026-06-25:

- [revm-precompile (docs.rs)](https://docs.rs/revm-precompile/latest/revm_precompile/) — precompile crate layout.
- [revm-precompile `interface` module (docs.rs)](https://docs.rs/revm-precompile/41.0.0/revm_precompile/interface/index.html) — `PrecompileFn` = `fn(&[u8], u64, u64) -> PrecompileResult` (input, gas, reservoir); `PrecompileResult` / `PrecompileOutput` / `PrecompileError`.
- [reth — `precompile` module (rustdoc)](https://reth.rs/docs/reth/revm/revm/precompile/index.html) — `Precompiles` (`inner`/`addresses`, `get`/`contains`), `PrecompileId`.
- [reth_evm (rustdoc)](https://reth.rs/docs/reth_evm/index.html) — `ConfigureEvm` / `EvmFactory` / `BlockExecutor`.
- [Reth SDK — EVM component](https://reth.rs/sdk/node-components/evm/).
- [bluealloy/revm releases](https://github.com/bluealloy/revm/releases) — `PrecompileOutput::gas_refunded` re-added.
- [EIP-2537 — BLS12-381 precompiles](https://eips.ethereum.org/EIPS/eip-2537) and [execution-spec tests](https://ethereum.github.io/execution-spec-tests/main/tests/prague/eip2537_bls_12_381_precompiles/).
- [go-ethereum core/vm/contracts.go](https://github.com/ethereum/go-ethereum/blob/master/core/vm/contracts.go) — canonical precompile table.
- [EVM Codes — precompiled contracts](https://www.evm.codes/precompiled); [RareSkills — precompiles](https://rareskills.io/post/solidity-precompiles).
- [Arbitrum — customize your chain's precompiles](https://docs.arbitrum.io/launch-arbitrum-chain/customize-your-chain/customize-precompile) — view/`eth_call`-only restriction.
- [Arbitrum Stylus — gentle introduction](https://docs.arbitrum.io/stylus/gentle-introduction) — WASM MultiVM, user-deployable "custom precompiles".

Companion research files in this directory (already source-verified; cited inline):

- `commonware-reth-glue.md` `[glue]` — revm/Reth precompile injection (`ConfigureEvm`/`EvmFactory`/`PrecompilesMap`/`DynPrecompile::new_stateful`/`StorageCtx`), Tempo source paths, determinism rules, hardfork gating.
- `tempo-as-blueprint.md` `[tempo]` — Tempo precompile addresses (`0xfeec…`/`0x20FC…`/`0xdec0…`/`0x5165…`/`0xAAAA…`), fee-AMM, signature verifier, no-native-token model.
- `cosmos-evm.md` `[cosmos]` — `Address()`/`RequiredGas()`/`Run()` interface, keeper-based state, ASA-2025-004 & ASA-2026-002 ($7M Saga) CVEs.
- `avalanche-subnet-evm.md` `[avax]` — stateful precompiles, `precompilegen`, ChainConfig/upgrade activation, built-ins.
- `polkadot-frontier.md` `[frontier]` — `precompile-utils`/`#[precompile]`, `pallet-evm-precompile-dispatch`, PolkaVM/`pallet-revive`, custom-precompile audit history.

---

**Last verified: 2026-06-25**
