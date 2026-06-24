# Reth & the Reth SDK — Research Findings

**Project:** Reth (Rust Ethereum) — Paradigm's modular Ethereum execution-layer client and node-building SDK
**Repository:** [github.com/paradigmxyz/reth](https://github.com/paradigmxyz/reth)
**Docs:** [reth.rs](https://reth.rs/) · [paradigm.xyz](https://www.paradigm.xyz/)
**Last verified:** 2026-06-24

---

## Executive Summary

Reth is a production-ready, Apache/MIT-licensed Ethereum **execution-layer** client written in Rust by Paradigm, built on `revm` (the Rust EVM), Alloy, and Foundry. It reached its first production release (1.0) in June 2024 after a Sigma Prime audit, and shipped **Reth 2.0 on 8 April 2026** with a redesigned tiered storage engine ("Storage V2"), ~1.7 Gigagas/s execution, and sub-2ms state-root computation; the latest tagged release is **v2.3.0 (10 June 2026)**. It tracks the latest Ethereum mainnet hardforks (Cancun, Prague/**Pectra** live May 2025, **Fusaka**/Osaka live Dec 2025, and ongoing **Glamsterdam/Amsterdam** EIP-7928 Block Access List work targeted for 2026), and is fully JSON-RPC and Engine-API compatible, so existing Ethereum tooling works unchanged. Its standout strength for this evaluation is the **Reth SDK** ("Reth as a library"): a NodeBuilder + component model that lets you inject **custom precompiles** and a custom EVM configuration via `revm`, plus **ExEx (Execution Extensions)** for post-execution logic — exactly the "Turing-incomplete modules exposed as EVM contracts via precompiles" pattern the requirements ask for. The main caveat is that **Reth is execution-only and consensus-agnostic**: it implements no consensus algorithm itself; PoS comes from an external Ethereum consensus client over the Engine API, and other models (PoA/DPoS/BFT) must be supplied by the surrounding stack (e.g. OP Stack via op-reth, or a custom consensus component).

---

## Fit Against Requirements

### 1. 1:1 compatibility with the latest Ethereum mainnet — **STRONG**
Reth is a *real Ethereum mainnet execution client*, not an EVM-alike. It is built directly on `revm` and is run in production on Ethereum mainnet (including by the Ethereum Foundation validator and a network-wide validator rollout in 2026), with full JSON-RPC and Engine-API compatibility. It supports the current mainnet hardfork set — Cancun, Prague/Pectra (EIP-7702 etc., live May 2025), and Fusaka/Osaka (PeerDAS, live Dec 2025) — and v2.2/v2.3 are already implementing the next fork (Glamsterdam/Amsterdam, EIP-7928 Block Access Lists). All standard Ethereum tooling (wallets, indexers, Foundry, RPC clients) works against it unchanged. This is the strongest possible answer to "1:1 mainnet compatibility."

### 2. Custom Turing-incomplete modules exposed as EVM contracts via precompiles — **STRONG**
This is a core, first-class capability. Through the Reth SDK's NodeBuilder and component model, you replace/extend the EVM component (`ConfigureEvm` / `EvmConfig` / `EvmFactory`) and inject **custom precompiles** into `revm`'s precompile map (`PrecompilesMap`), giving native-speed, deterministic (Turing-incomplete) functions callable at fixed addresses like ordinary precompiled contracts. Real deployments do exactly this: SP1-Reth adds keccak/secp256k1/sha256 precompiles; WeaveVM adds Arweave-data precompiles via a custom `ExecutorBuilder`. Separately, **ExEx (Execution Extensions)** provide reorg-aware post-execution hooks for indexers/bridges/rollups. Verdict is Strong with one nuance: precompiles run *inside* a custom build of the node (you compile your precompiles/ExEx into the binary), which is the intended model but means you are operating a customized client, not configuring a hosted chain.

### 3. Consensus support (BFT / DPoS / PoS / PoA) — **PARTIAL (by design)**
Reth is an **execution-layer client** and is deliberately **consensus-agnostic** — it implements no consensus algorithm on its own. For Ethereum **PoS**, consensus is provided by a separate CL client (Lighthouse, Prysm, Teku, Nimbus, Lodestar) over the **Engine API**; Reth is compatible with all of them. For L2s, the OP Stack supplies consensus/derivation via `op-node` paired with **op-reth**. The SDK exposes a pluggable **Consensus component**, and projects use Reth under custom consensus (e.g. BNB Chain's bsc-reth). So Reth *pairs with* PoS/PoA/BFT/DPoS engines but does **not itself implement** them — you must bring or build the consensus layer. Honest verdict: Partial. It is an excellent, neutral execution substrate for any consensus, but it is not a turnkey BFT/DPoS/PoA chain on its own.

### 4. Actively maintained — **STRONG**
Maintained by Paradigm plus a large open-source ecosystem (Optimism/Base, BNB Chain, and many forks). Monthly-ish cadence: v2.0.0 (8 Apr 2026), v2.1.0 (20 Apr 2026), v2.2.0 (30 Apr 2026), v2.3.0 (10 Jun 2026). Very active development, heavy production adoption (Base deprecated Geth for Reth; Optimism is sunsetting op-geth; growing Ethereum mainnet share), and a public roadmap (AOT/JIT execution via `revmc`, Gigagas scaling). Clearly and strongly maintained.

---

## Architecture

Reth follows **Erigon's "staged sync"** pipeline design and stores data in **MDBX** plus **append-only Static Files**. It is decomposed into many focused, individually reusable crates:

- **Execution:** `revm` (Rust EVM) is the execution engine; Reth wraps it with block executors and EVM configuration traits.
- **Sync:** a staged pipeline (`stages/api`, `stages/stages`) — headers, bodies, sender recovery, execution, merkle/trie, etc.
- **Storage:** MDBX for hot/hashed state plus Static Files for cold/append-only history. **Storage V2** (default in Reth 2.0) splits hot/cold data, stores only hashed state in MDBX, and moves historical changesets to Static Files — enabling a "minimal mode" archive-capable node under ~300 GB and ~240 GB footprints at ~1.7 Ggas/s.
- **Networking:** full devp2p/P2P stack.
- **RPC:** multi-transport JSON-RPC (HTTP/WS/IPC), Ethereum-compatible.
- **Consensus:** a *component interface* (validation/fork-choice integration via Engine API), not a consensus algorithm.

Performance highlights in 2.0: state-root via a **Sparse Trie Cache** at ~1–2 ms/block, ~40 ms standard block persistence (vs 8.4 s in V1 for large blocks → ~400 ms), and parallel-stream snapshot sync (~10-minute syncs). Built with Rust, Alloy, revm, and Foundry. Licensed **Apache-2.0 OR MIT**.

## Compatibility (Ethereum mainnet & tooling)

- **Real mainnet client**, production-ready since 1.0 (June 2024, audited by Sigma Prime); "suitable for mission-critical environments such as staking."
- **Hardforks:** Cancun (Dencun), **Prague/Pectra** (live mainnet 7 May 2025, incl. EIP-7702), **Fusaka/Osaka** (live mainnet 3 Dec 2025, PeerDAS). v2.2/v2.3 ship groundwork + correctness for **Glamsterdam/Amsterdam** (EIP-7928 Block Access Lists), targeted ~2026.
- **JSON-RPC:** full Ethereum-compatible RPC; all standard tooling (Foundry, wallets, indexers, explorers) works unchanged.
- **Engine API:** compatible with all Ethereum CL clients, so it slots directly into the standard EL/CL split.
- **Multi-chain:** can sync Ethereum and other EVM chains; OP Stack support is delivered via **op-reth** (moved to `ethereum-optimism/optimism`), and forks exist for BNB Chain (bsc-reth) and others.

## Custom Precompiles & Custom EVM (the key SDK capability)

The **Reth SDK** ("use Reth node components as libraries") centers on the **NodeBuilder** pattern, letting you customize every component: Network, **Consensus**, **EVM**, RPC, Transaction Pool, plus the type system and DB layer.

**How custom precompiles are injected:**
1. Implement/extend the EVM configuration — Reth exposes EVM configuration traits (`ConfigureEvm` / `EvmConfig`, with an `EvmFactory` that constructs the `revm` instance per block).
2. When building the `revm` EVM, modify its **precompiles** — `revm` exposes a precompile registry (in Reth surfaced as a **`PrecompilesMap`** / `with_precompiles`-style API) where you register additional precompiled contracts at chosen addresses. Each precompile is a deterministic, **Turing-incomplete** Rust function `(input bytes, gas) -> (output bytes, gas_used)`.
3. Wire this through a custom **`ExecutorBuilder`/EVM component** in the NodeBuilder so the customized EVM (with your precompiles) is used when the node bootstraps.
4. Compile and run your customized Reth binary. Because precompiles are compiled into the node, all custom chain participants must run the same build.

**Real-world precedents:** SP1-Reth (Succinct) adds keccak/secp256k1/sha256 precompiles to build a type-1 zkEVM; WeaveVM forks Reth and registers Arweave-data-access precompiles via a custom `ExecutorBuilder`. This is precisely the "extra Turing-incomplete modules exposed as EVM contracts through precompiles" pattern in the requirements.

**ExEx (Execution Extensions) — complementary:** ExExes are async tasks compiled into the same binary that receive **reorg-aware notifications** (block committed/reverted/reorged) with full access to blocks, txs, receipts, and state changes — ideal for rollups, bridges, and indexers. ExEx is *post-execution* infrastructure (it observes/derives state); **precompiles** are *in-EVM* logic (they add callable contract functions). Together they cover both "add native functions to the VM" and "build infrastructure on top of execution." A **Remote ExEx** option streams data out to a separate process.

## Consensus

Reth does **not** implement consensus. It is consensus-agnostic and exposes consensus as a pluggable component:

- **PoS (Ethereum mainnet):** external CL client (Lighthouse/Prysm/Teku/Nimbus/Lodestar) drives fork-choice and block production via the **Engine API**; Reth executes. (Sigma Prime even prototyped "Fullhouse" — Lighthouse + Reth in one binary.)
- **L2 / rollups:** OP Stack uses **op-node** (derivation/sequencing) + **op-reth** (execution); other stacks plug their own derivation/consensus.
- **PoA / DPoS / BFT:** not provided by Reth itself; supplied by the surrounding stack or a custom Consensus component (e.g. BNB Chain's bsc-reth runs Reth under BSC's consensus). Reth makes an excellent neutral execution engine for any of these, but is **not** a turnkey BFT/DPoS/PoA node.

## Maintenance & Adoption

- **Maintainer:** Paradigm + broad open-source ecosystem (Optimism/Base, BNB Chain, MegaETH, Succinct, etc.).
- **Releases:** 1.0 (Jun 2024) → 2.0 (8 Apr 2026) → 2.1 (20 Apr 2026) → 2.2 (30 Apr 2026) → **2.3.0 (10 Jun 2026, latest)**. Frequent, roughly monthly point releases.
- **Adoption:** Base fully moved off Geth to Reth and invests in OP Stack integration; Optimism sunsetting op-geth (~May 2026); growing Ethereum mainnet client share (~mid-single-digit % and rising); EF Foundation validator migrated (Feb 2026) with broader validator rollout in 2026; BNB Chain reports ~40% faster sync / ~24% lower execution latency vs Geth.
- **Roadmap:** AOT/JIT execution via **revmc**, continued Gigagas scaling, full parallel execution post-Glamsterdam (BALs).

---

## Pros / Cons vs the 4 Criteria

**Pros**
- Genuine, audited, production mainnet client → unbeatable Ethereum 1:1 compatibility and tooling support (Criterion 1).
- Best-in-class extensibility for custom precompiles + custom EVM + ExEx via a clean library/SDK model (Criterion 2).
- Consensus-agnostic component model → drops into any consensus stack (PoS/PoA/BFT/DPoS) cleanly (Criterion 3, as a substrate).
- Very actively maintained by Paradigm with major production users (Criterion 4).
- High performance (Gigagas-class), modular Rust crates, permissive Apache/MIT license.

**Cons / Caveats**
- **Execution-only:** implements no consensus — you must bring/build BFT/DPoS/PoA/PoS yourself (Criterion 3 is Partial). It is not a full-stack chain by itself.
- Custom precompiles/ExEx require **compiling and operating a customized Reth binary**; all nodes must run the same build (operational coordination cost).
- SDK is powerful but lower-level (Rust, NodeBuilder, EVM traits) — steeper learning curve than a config-driven chain framework.
- For non-Ethereum consensus you typically adopt an external framework (OP Stack) or fork, rather than a turnkey Reth feature.

---

## Sources

- [Reth GitHub repository](https://github.com/paradigmxyz/reth)
- [Reth releases (GitHub)](https://github.com/paradigmxyz/reth/releases)
- [Reth README](https://github.com/paradigmxyz/reth/blob/main/README.md)
- [Reth documentation (reth.rs)](https://reth.rs/)
- [Reth SDK overview (reth.rs)](https://reth.rs/sdk/)
- [Execution Extensions (ExEx) overview — reth.rs](https://reth.rs/exex/overview/)
- [Remote Execution Extensions — reth.rs](https://reth.rs/exex/remote/)
- [Running Reth on Ethereum Mainnet — reth.rs](https://reth.rs/run/ethereum/)
- [Releasing Reth 2.0! — Paradigm (Apr 2026)](https://www.paradigm.xyz/2026/04/releasing-reth-2-0)
- [Releasing Reth 1.0! — Paradigm (Jun 2024)](https://www.paradigm.xyz/2024/06/reth-prod)
- [Reth Execution Extensions — Paradigm (May 2024)](https://www.paradigm.xyz/2024/05/reth-exex)
- [Introducing Reth — Paradigm (Dec 2022)](https://www.paradigm.xyz/2022/12/reth)
- [Paradigm releases production-ready Reth 1.0 — The Block](https://www.theblock.co/post/302158/paradigm-reth)
- [What's New in Paradigm's Reth 2.0 — Bankless](https://www.bankless.com/read/whats-new-in-paradigms-reth-2-0)
- [How to Build and Deploy Reth ExExs — QuickNode](https://www.quicknode.com/guides/infrastructure/how-to-use-reth-exex)
- [Introducing SP1 Reth (custom precompiles) — Succinct](https://blog.succinct.xyz/sp1-reth/)
- [Unlocking Arweave with EVM Precompiles in Reth — WeaveVM](https://blog.wvm.dev/unlocking-arweave-with-evm-precompiles-in-reth/)
- [Scaling Base With Reth — Base Engineering Blog](https://blog.base.dev/scaling-base-with-reth)
- [op-reth — ethereum-optimism/op-reth](https://github.com/ethereum-optimism/op-reth)
- [OP Stack execution client configuration — Optimism docs](https://docs.optimism.io/node-operators/guides/configuration/execution-clients)
- [BSC Reth Client — BNB Chain Blog](https://www.bnbchain.org/en/blog/bsc-reth-client-the-next-evolution-of-bsc-infrastructure)
- [bnb-chain/reth (bsc-reth)](https://github.com/bnb-chain/reth)
- ["Fullhouse": Lighthouse + Reth in a single binary — Sigma Prime](https://blog.sigmaprime.io/fullhouse.html)
- [Why 0G migrated validators from Geth to Reth — 0G](https://0g.ai/blog/geth-to-reth-validator-migration)
- [Reth SDK Comprehensive Technical Analysis — HackMD](https://hackmd.io/@divergence/claude-reth-research)
- [Pectra upgrade — ethereum.org](https://ethereum.org/roadmap/pectra/)
- [Fusaka Mainnet Announcement — Ethereum Foundation Blog](https://blog.ethereum.org/2025/11/06/fusaka-mainnet-announcement)
- [Protocol Priorities Update 2026 — Ethereum Foundation Blog](https://blog.ethereum.org/2026/02/18/protocol-priorities-update-2026)
- [From Pectra to Fusaka: how Ethereum's protocol changed in 2025 — The Block](https://www.theblock.co/post/383451/how-ethereums-protocol-changed-2025)

---

*Last verified: 2026-06-24*
