# Polkadot Frontier (pallet-evm + pallet-ethereum)

**Project:** [`polkadot-evm/frontier`](https://github.com/polkadot-evm/frontier) — the Ethereum / EVM compatibility layer for Substrate and Polkadot.
**Last verified:** 2026-06-24

---

## Executive summary

Frontier is an **EVM emulation layer for the Substrate / Polkadot SDK**, implemented in Rust and shipped as a set of FRAME pallets (`pallet-evm`, `pallet-ethereum`) plus an Ethereum-compatible JSON-RPC client (`fc-rpc`, the "Eth RPC"). It lets a Substrate chain present itself to the outside world as an Ethereum node so that unmodified Ethereum tooling (MetaMask, Hardhat, Foundry, ethers.js) and unmodified Solidity dapps work. It is the technology that powers production EVM parachains such as **Moonbeam, Moonriver and Astar**. Frontier is genuinely actively maintained (commits in June 2026, including EIP-7702 and BLS12-381 precompile work) and tracks Ethereum hardforks reasonably closely through the `rust-ethereum/evm` interpreter (formerly SputnikVM), which now exposes a **Prague** preset. Its standout strength is the **precompile system**: Substrate pallets and custom Rust logic can be exposed to Solidity as precompiled contracts (the Moonbeam `precompile-utils` pattern), which is exactly the "add Turing-incomplete modules and expose them as EVM contracts" capability. Because consensus is provided by Substrate, Frontier is **consensus-agnostic** and inherits PoA (Aura), PoS/finality (BABE+GRANDPA), NPoS, Nimbus/DPoS-style collation, or PoW depending on the host chain. The major strategic caveat in 2026: Parity has moved EVM compatibility *into the core SDK* via **REVM + `pallet-revive`** for the **Polkadot Hub** (launched on Kusama late-2025, Polkadot enactment ~27 Jan 2026), which is the officially blessed forward path and a partial competitor to Frontier — though Frontier remains the de-facto choice for sovereign/standalone EVM chains and for the large existing parachain installed base.

---

## Fit against requirements

| # | Requirement | Verdict | Why |
|---|-------------|---------|-----|
| 1 | 1:1 compatibility with latest Ethereum mainnet (all tooling works) | **Strong (with caveats)** | Full Eth JSON-RPC, EIP-1559, and the EVM interpreter now reaches **Prague** (incl. EIP-7702 work landing mid-2026). Standard tooling works. But it is an *emulation* layer — block/account model and gas are mapped onto Substrate, producing known deviations (see below). |
| 2 | Add Turing-incomplete modules and expose them as EVM contracts via precompiles | **Strong** | This is Frontier's signature feature. Native Substrate pallets and custom Rust are exposed to Solidity through precompiles; the Moonbeam `precompile-utils` crate + `#[precompile]` proc-macro make this ergonomic, and `pallet-evm-precompile-dispatch` can dispatch arbitrary runtime calls from the EVM. |
| 3 | Consensus support: BFT, DPoS, PoS, PoA | **Strong** | Frontier is consensus-agnostic; Substrate supplies Aura (PoA), BABE+GRANDPA (PoS block-production + BFT finality), NPoS (staking), Nimbus (collator/DPoS-style), and PoW. Any of these can host Frontier unchanged. |
| 4 | Actively maintained | **Strong (with strategic caveat)** | Active commits June 2026, tracks polkadot-sdk `stable*` releases, dual-licensed, community + Parity stewardship under the `polkadot-evm` org. Caveat: Parity's strategic EVM bet has shifted to REVM/`pallet-revive`/Polkadot Hub, so Frontier's long-term centrality is uncertain. |

---

## Architecture

Frontier is **not a standalone chain**. It is a bundle of components dropped into a Substrate / Polkadot-SDK runtime:

- **`pallet-evm`** — the core EVM execution environment. Runs EVM bytecode, maps EVM accounts/addresses/balances onto Substrate's account model, and applies a gas configuration (hardfork preset). Can be used on its own ("EVM-execution-only" mode) but then *Ethereum RPCs are not available* and dapp frontends must use Substrate APIs.
- **`pallet-ethereum`** — sits on top of `pallet-evm` to provide **Ethereum block emulation** and to **validate Ethereum-encoded (RLP-signed) transactions**. This is what makes the chain look like Ethereum: Ethereum-format transactions with native Ethereum signing are accepted, and an Ethereum block is synthesized. Block construction supports a "post-block" strategy (Ethereum block generated after runtime execution) and a "pre-block / wrapper block" strategy used for migrations.
- **Eth RPC client layer (`fc-rpc`, `fc-mapping-sync`, `fc-db`)** — the node-side services that expose the standard Ethereum JSON-RPC endpoints (`eth_*`, `net_*`, `web3_*`) and maintain the mapping between Substrate block hashes and emulated Ethereum block/transaction hashes.
- **EVM interpreter** — the [`rust-ethereum/evm`](https://github.com/rust-ethereum/evm) crate (the project historically known as **SputnikVM**, since rewritten). It is a customizable pure-Rust EVM interpreter that passes the Ethereum test suite. Frontier does **not** use `revm`; `revm` is the engine Parity chose for the *separate* `pallet-revive` path.

**Language:** Rust (~84% of the repo; the rest is TypeScript integration tests and Solidity test fixtures).
**License:** Dual-licensed **Apache-2.0 / GPL-3.0**.
**Forkless upgrades:** because it is a FRAME pallet set, EVM support (and its config) can be added, removed, or upgraded at runtime via Substrate's on-chain runtime-upgrade mechanism.

---

## Ethereum compatibility (Criterion 1)

**EVM version / hardforks.** The underlying `rust-ethereum/evm` interpreter defines presets up to and including **Prague**: Frontier, Homestead, Tangerine Whistle, Spurious Dragon, Byzantium, Petersburg, Istanbul, Berlin, **London**, **Shanghai**, **Cancun**, **Prague**. Historically Frontier shipped a default **London** gas configuration; recent (2026) work is bringing it current — e.g. June 2026 commits add **EIP-7702 (set-code / delegation) metering** in the storage meter and **BLS12-381 G1/G2 subgroup checks** on the multiexp precompiles (Prague-era EIPs). The actual hardfork active on any given chain is a runtime configuration choice.

**EIP-1559.** Supported (fee-market with base fee + priority fee); `pallet-base-fee` provides the base-fee logic, and the RPC surfaces `maxPriorityFeePerGas` / `maxFeePerGas` style transactions.

**JSON-RPC.** Frontier exposes the standard Ethereum RPC API via its `fc-rpc` client, which is why MetaMask, Hardhat, Foundry, ethers.js and Web3.js connect without modification. Debug/trace RPC (`debug_*`, `txpool_*`, tracing) is available through additional Frontier/Moonbeam tracing modules.

**Known deviations / emulation quirks (important):**
- **Account model mapping.** EVM 20-byte addresses are mapped onto Substrate's 32-byte `AccountId` via a configurable mapping (`HashedAddressMapping` / `IdentityAddressMapping`). Some chains expose a single unified account; others keep EVM and Substrate accounts distinct, which can confuse balance/nonce expectations.
- **Block emulation.** Ethereum blocks are *synthesized* from Substrate blocks. Block time, finality, and the meaning of "block" follow the host chain's consensus, not Ethereum's. Some block fields are emulated/placeholder; opcodes like `DIFFICULTY`/`PREVRANDAO`, `BASEFEE`, and `BLOCKHASH` behave per the host chain's semantics.
- **Gas ↔ Weight.** EVM gas must be reconciled with Substrate's two-dimensional Weight (ref-time + proof-size). Gas limits and pricing are mapped through a `GasWeightMapping`; proof-size limits (PoV) on parachains can cause transactions that would succeed on Ethereum to hit Substrate resource limits.
- **Finality semantics.** "Confirmed" depends on the host consensus (e.g. GRANDPA finality on Polkadot), not Ethereum's probabilistic/finality model — tooling that assumes Ethereum confirmation depth may need tuning.
- **No native blob transactions / KZG (EIP-4844 data blobs)** in the emulation sense — blob-carrying L1 semantics don't map onto a parachain.

Net: extremely good *application-level* compatibility (contracts and tooling work), but it is an emulation, so infrastructure-level assumptions (account identity, gas, block/finality) differ from real Ethereum.

---

## Precompiles — exposing Substrate pallets as EVM contracts (Criterion 2)

This is Frontier's strongest differentiator and directly satisfies the "add Turing-incomplete modules and expose them as EVM contracts" requirement.

**Built-in precompiles** (`frame/evm/precompile/*`): `simple` (ECRecover, SHA256, RIPEMD160, Identity — the standard Ethereum precompiles), `modexp`, `bn128`, `blake2`, `sha3fips`, `ed25519`, `curve25519`, `bls12377`, `bls12381`, `bw6761`, and crucially **`dispatch`** (`pallet-evm-precompile-dispatch`).

**`pallet-evm-precompile-dispatch`** is the generic bridge: it lets an EVM contract dispatch an arbitrary Substrate runtime `Call` (a pallet extrinsic) from inside the EVM, so any FRAME pallet's functionality can be reached from Solidity through a single precompile address.

**Custom / pallet precompiles (the Moonbeam pattern).** Moonbeam (which co-created Frontier) established the production-grade pattern: write a Rust precompile that wraps a pallet and presents a clean **Solidity interface** at a fixed address. Solidity developers then `interface`-call it like any contract. Moonbeam exposes pallets this way for **staking, governance/democracy, collective, proxy, identity/registrar, preimage, randomness, XC-20 (ERC-20 view of Substrate assets), X-Tokens / XCM transactor (cross-chain), batch (multi-call), and call-permit (gasless/EIP-712 meta-tx)** — i.e. Turing-incomplete native modules surfaced as EVM contracts.

**`precompile-utils` crate + `#[precompile]` proc-macro.** Moonbeam's `precompile-utils` removes the boilerplate of Solidity ABI encoding/decoding, function selector matching, gas costing, and error handling. A developer adding a custom precompile typically:
1. Defines the Solidity interface (`.sol`) for the functions to expose.
2. Writes a Rust struct and annotates methods with `#[precompile]` (matching selectors to the Solidity ABI), using `precompile-utils` helpers to read inputs and return values.
3. Calls into the target pallet's dispatchable / storage from the precompile body, applying gas/weight accounting.
4. Registers the precompile at a chosen address in the runtime's `PrecompileSet`.

This gives a clean, well-trodden path to extend the EVM with arbitrary native functionality without writing it in (gas-expensive, Turing-complete) Solidity. (Note: custom precompiles are powerful and have been a source of security advisories — Moonbeam has shipped urgent patches for custom-precompile bugs — so they require careful auditing.)

---

## Consensus support (Criterion 3)

Frontier itself contains **no consensus** — it is consensus-agnostic and relies entirely on the host Substrate/Polkadot-SDK runtime. Available options:

- **PoA — Aura.** Round-robin authority block production; the standard choice for permissioned / consortium chains and many dev/solo chains. Frontier's own template node uses an Aura+GRANDPA setup.
- **PoS + BFT finality — BABE + GRANDPA.** BABE provides (VRF-based) slot block production; **GRANDPA** is the **Byzantine-fault-tolerant finality gadget**. Together they deliver the PoS + BFT combination.
- **NPoS / DPoS-style staking.** Substrate's `pallet-staking` (Nominated Proof-of-Stake) and Moonbeam's `parachain-staking` implement delegated/nominated staking — the DPoS-style requirement.
- **Nimbus (parachain collation).** Moonbeam's Nimbus framework selects collators and moves authorship checks into the runtime; used by Frontier-based parachains. Finality on a parachain is ultimately provided by the **Polkadot relay chain (GRANDPA)** via shared security.
- **PoW.** Substrate also ships PoW consensus modules; Frontier can run on a PoW chain (less common in production).

Because all of these are pluggable Substrate components, a Frontier chain can be configured for **BFT, PoS, DPoS/NPoS, or PoA** without changing Frontier itself. This is a clear strength versus monolithic EVM clients with a single fixed consensus.

---

## Maintenance & strategic status (Criterion 4)

**Active maintenance — confirmed current (June 2026):**
- Repo `polkadot-evm/frontier`, default branch `master`; commits as recent as **2026-06-18**, repo pushed-to **2026-06-24**.
- Recent substantive work (not just dependabot): **"Meter EIP-7702 delegation writes in StorageMeter" (#1901, 2026-06-11)** and **"Enforce subgroup checks on BLS12-381 G1/G2 multiexp precompiles" (#1900, 2026-06)** — i.e. tracking Prague-era EVM behavior.
- ~616 stars / 544 forks / ~149 open issues / 36+ release tags; per-pallet semver tags (e.g. `pallet-evm-v5.0.0`) plus an umbrella `v0.2.1`.
- Dual Apache-2.0 / GPL-3.0 license.

**Compatibility tracking.** Frontier maintains branches/releases that track **polkadot-sdk `stable*`** lines (the SDK moved to a calendar-style `stableYYMM` release cadence — e.g. `stable2509` Oct 2025, `stable2603` in 2026). Chain teams pick the Frontier revision matching their polkadot-sdk version.

**Governance.** Originally Parity-led and co-developed with Moonbeam; it now lives under the community **`polkadot-evm` GitHub org**, and as Parity decentralizes there is movement toward an on-chain collective stewarding Frontier independently rather than as a pure Parity product.

**Strategic caveat — REVM / `pallet-revive` / Polkadot Hub.** The biggest 2025-2026 development: Parity has embedded an EVM engine (**REVM**) directly into the Polkadot SDK as a first-class module alongside FRAME/Cumulus/XCM, exposed through **`pallet-revive`** with a **dual-VM** design — **REVM** for Ethereum-semantics Solidity, and **PolkaVM (PVM)**, a RISC-V register-based VM for higher-compute / multi-language workloads, with **Revive** coordinating gas-weight accounting, shared state, and a unified ETH-RPC. This is the **Polkadot Hub** smart-contract story: EVM + PVM went live on **Kusama in late 2025**, and the **Polkadot** enactment (originally targeted 20 Jan 2026) was rescheduled to **~27 Jan 2026**. Infra work includes gas-to-weight mapping, Ethereum-style block storage in `pallet-revive`, 18-decimal DOT alignment, and Anvil-compatible local nodes plus Hardhat/Foundry integration. The 2026 roadmap adds a PVM JIT, more source languages, and gas payable in any asset.

**Implication for Frontier:** Parity's *official* forward investment for EVM-on-Polkadot is now REVM/`pallet-revive`, not Frontier. However: (a) Frontier remains independently maintained and is the path for **sovereign / standalone EVM chains and appchains** that aren't Polkadot Hub; (b) the **large installed base** (Moonbeam, Moonriver, Astar and others) runs on Frontier and continues to drive its development; (c) `pallet-revive` targets the shared Polkadot Hub rather than replacing the Frontier-style "own-runtime EVM parachain" model. So Frontier is not deprecated, but its strategic centrality within *Polkadot itself* has been partly superseded.

---

## Notable production chains

- **Moonbeam / Moonriver** — the flagship Frontier-based EVM parachains; source of the `precompile-utils` and Nimbus tooling; full Ethereum compatibility with native Substrate precompiles.
- **Astar** — major EVM (Frontier) + Wasm (ink!) parachain.
- Numerous other parachains and sovereign Substrate chains embed Frontier for EVM support.

---

## Frank pros / cons

**Pros**
- True Ethereum tooling compatibility (MetaMask/Hardhat/Foundry/ethers) on a Substrate chain; battle-tested in production (Moonbeam, Astar).
- Best-in-class **extensibility via precompiles** — native pallets and custom Rust exposed as Solidity contracts; `dispatch` precompile reaches arbitrary runtime calls.
- **Consensus-agnostic**: PoA / PoS / BFT / DPoS-NPoS / PoW all available from Substrate, plus forkless runtime upgrades.
- Genuinely active, EIP-current (Prague, EIP-7702/BLS12-381 in 2026), permissively dual-licensed.

**Cons**
- It is an **emulation layer**, not a real Ethereum client: account-model mapping, gas↔Weight (incl. PoV proof-size), block synthesis, and finality semantics deviate from L1 and can trip infrastructure-level assumptions.
- Uses the `rust-ethereum/evm` interpreter (not `revm`); hardfork parity can lag mainnet, and integrating Frontier + matching polkadot-sdk versions is non-trivial engineering.
- Custom precompiles are powerful but a real **security-audit burden** (history of urgent patches).
- **Strategic risk:** Parity's official EVM future is REVM/`pallet-revive`/Polkadot Hub, so a team choosing Frontier should weigh long-term direction vs. building on the Hub.

---

## Sources

- [polkadot-evm/frontier — GitHub repository](https://github.com/polkadot-evm/frontier)
- [Frontier README (master)](https://raw.githubusercontent.com/polkadot-evm/frontier/master/README.md)
- [Frontier documentation site](https://polkadot-evm.github.io/frontier/)
- [Frontier overview docs](https://polkadot-evm.github.io/frontier/overview.html)
- [frame/evm source (pallet-evm)](https://github.com/polkadot-evm/frontier/tree/master/frame/evm)
- [frame/ethereum source (pallet-ethereum)](https://github.com/polkadot-evm/frontier/tree/master/frame/ethereum)
- [frame/evm/precompile — built-in precompiles](https://github.com/polkadot-evm/frontier/tree/master/frame/evm/precompile)
- [Frontier commits API (activity June 2026, EIP-7702 #1901, BLS12-381 #1900)](https://api.github.com/repos/polkadot-evm/frontier/commits)
- [rust-ethereum/evm — EVM interpreter (formerly SputnikVM)](https://github.com/rust-ethereum/evm)
- [rust-ethereum/evm hardfork config presets (up to Prague)](https://github.com/rust-ethereum/evm/blob/master/src/standard/config.rs)
- [Polkadot EVM org](https://github.com/polkadot-evm)
- [Does FrontierEVM support EIP-1559? — issue #1411](https://github.com/polkadot-evm/frontier/issues/1411)
- [HAS_PRECOMPILE and EVM_VERSION precompiles — issue #548](https://github.com/polkadot-evm/frontier/issues/548)
- [Moonbeam — Solidity Precompiles docs](https://docs.moonbeam.network/builders/pallets-precompiles/precompiles/)
- [Moonbeam — `#[precompile]` proc-macro / precompile-utils refactor (PR #1783)](https://github.com/moonbeam-foundation/moonbeam/pull/1783)
- [Moonbeam — urgent security patch for custom precompiles](https://moonbeam.network/blog/urgent-security-patch-for-custom-precompiles/)
- [Moonbeam — Consensus & Finality (Nimbus, GRANDPA)](https://docs.moonbeam.network/learn/features/consensus/)
- [moonbeam-foundation/nimbus — consensus framework](https://github.com/moonbeam-foundation/nimbus)
- [Polkadot Wiki — Smart Contracts on Polkadot](https://wiki.polkadot.com/learn/learn-smart-contracts/)
- [Polkadot Developer Docs — Polkadot Hub Smart Contracts](https://docs.polkadot.com/reference/polkadot-hub/smart-contracts/)
- [Polkadot Developer Docs — Precompiles](https://docs.polkadot.com/smart-contracts/precompiles/)
- [Polkadot Developer Docs — PolkaVM Design](https://docs.polkadot.com/polkadot-protocol/smart-contract-basics/polkavm-design/)
- [Polkadot Forum — Smart Contracts on Polkadot Hub: Progress Update](https://forum.polkadot.network/t/smart-contracts-on-polkadot-hub-progress-update/14596)
- [Polkadot Forum — "Revive" Smart Contracts: Status Update (Jan 2026 enactment ~27 Jan)](https://forum.polkadot.network/t/revive-smart-contracts-status-update/16366)
- [Parity — What is Polkadot Hub?](https://www.parity.io/blog/what-is-polkadot-hub)
- [OpenGuild — Dual-VM architecture of Polkadot Hub (REVM + PVM)](https://openguild.wtf/blog/polkadot/polkadot-introducing-about-dual-vm-architecture-polkadot-hub)
- [OpenGuild — From Substrate to Polkadot SDK (REVM embedded in SDK)](https://openguild.wtf/blog/polkadot/polkadot-from-substrate-to-polkadot-sdk)
- [paritytech/polkadot-sdk releases (stable cadence, e.g. stable2603)](https://github.com/paritytech/polkadot-sdk/releases)

---

*Last verified: 2026-06-24*
