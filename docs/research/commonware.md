# Commonware — Evaluation for a Sovereign EVM Chain

**Last verified: 2026-06-24**

## Executive summary

Commonware (the "Commonware Library," by Commonware, Inc., founded by Patrick O'Grady, ex-VP of Engineering at Ava Labs / Avalanche) is a Rust monorepo of low-level, composable blockchain **primitives** — consensus, authenticated P2P, a deterministic async runtime, authenticated storage (QMDB/ADB/MMR), cryptography, codec, broadcast, and deployment tooling. It explicitly markets itself as the "anti-framework": a toolbox for assembling specialized blockchains rather than a turnkey chain or SDK. Crucially for an EVM use case, **Commonware ships no EVM and no execution layer of its own.** It deliberately leaves execution to the developer. Its consensus is its standout strength — Simplex / threshold-simplex / Minimmit are modern, sub-second-finality BFT protocols with succinct BLS12-381 certificates and dynamic validator reconfiguration. The reference proof point that an EVM chain *can* be built on it is **Tempo** (the Stripe/Paradigm stablecoin L1, mainnet March 2026), which pairs Commonware's Simplex BFT consensus with **Reth** (Paradigm's revm-based Ethereum execution client) for full EVM compatibility. So building an EVM chain on Commonware is entirely feasible and proven at scale, but EVM compatibility is **100% do-it-yourself**: you supply and wire in revm/Reth yourself; Commonware provides the consensus, networking, storage, and runtime around it.

## Fit against requirements

| # | Requirement | Verdict | Why |
|---|-------------|---------|-----|
| 1 | 1:1 latest-Ethereum-mainnet compatibility (all EVM tooling works) | **Weak (out of the box) / Partial-to-Strong (DIY)** | Commonware has no EVM. You must integrate your own execution layer (revm or Reth). If you embed Reth — as Tempo does — you inherit Reth's near-complete, continuously updated EVM/mainnet compatibility, so all Ethereum tooling works. But none of that ships with Commonware; it is integration work you own. |
| 2 | Add Turing-incomplete modules exposed as EVM precompiles | **Strong (in principle), but DIY** | Because you assemble the execution layer yourself, you have total freedom to inject custom precompiles. With revm/Reth this is a well-supported pattern (custom precompile registration). Commonware imposes no constraints. The flexibility is maximal; the work is yours. |
| 3 | Consensus: BFT / PoS / DPoS / PoA | **Strong** | This is Commonware's core competency. Production-grade Simplex, threshold-simplex, and Minimmit BFT protocols with sub-second finality, succinct certificates, and reconfigurable validator sets. PoS/DPoS/PoA are stake/identity weighting layers you build on top — the primitives support dynamic validator sets and DKG/resharing. |
| 4 | Actively maintained | **Strong** | Monthly releases (latest v2026.5.0, May 28 2026), 93% test coverage, 1500 daily benchmarks, well-funded ($9M seed from Haun Ventures & Dragonfly; $25M strategic from Tempo, Nov 2025), real production user (Tempo mainnet 2026) and active testnet (Alto). Note: it is still a young, fast-moving library — APIs change. |

## What Commonware is (and is not)

- **Is:** A library of independent Rust primitives ("17 primitives and 50+ dialects") for building decentralized systems — consensus, P2P, runtime, storage, crypto, codec, broadcast, deployer. MIT/Apache-2.0 dual-licensed. ~99% Rust.
- **Is not:** A blockchain framework, an SDK like Cosmos SDK, or an EVM. There is no "import the chain" path. You compose primitives into your own node binary and supply your own state-transition / execution logic.
- **Philosophy ("anti-framework"):** Each primitive solves one problem, has minimal dependencies, is trait/interface-driven, and prescribes no usage. The bet is that specialized, unbundled chains (custom networking + consensus + storage + execution) beat generalized frameworks.

## Architecture — the crate suite

All crates are Rust, published on crates.io (`commonware-*`) and documented on docs.rs. Key primitives:

- **runtime** — Deterministic, async task runtime with configurable scheduling; supports a deterministic simulation mode for reproducible testing (a major reliability differentiator).
- **p2p** — Authenticated, encrypted peer-to-peer communication (handshake-authenticated peers).
- **consensus** — BFT ordering primitives (see Consensus section): `simplex`, `threshold_simplex`, `minimmit`, plus `marshal` (ordered delivery of finalized blocks), `aggregation` (quorum certificates over a synchronized sequencer), and `ordered_broadcast` (reliable broadcast across reconfigurable participants).
- **storage** — Authenticated data structures: **QMDB** (Quick Merkle Database, productionized with LayerZero), the **ADB** family (`adb::any`, `adb::current`), **MMR** (Merkle Mountain Range, including grafting), and journaled logs. These give you Merkle-proof state commitments — but they are *Commonware's* commitment scheme, not Ethereum's MPT/state trie. (Relevant to criterion 1: if you use Reth for execution you'd use Reth's state trie, not QMDB, unless you bridge them.)
- **cryptography** — Key generation, signing, verification; BLS12-381 (threshold sigs, DKG/resharing), ed25519, VRFs.
- **codec** — Deterministic serialization; **conformance** asserts encoding stability over time.
- **broadcast**, **stream**, **resolver**, **collector** — Data dissemination, transport, key-based resolution, response collection.
- **coding** — Erasure/recovery coding.
- **deployer** — Multi-cloud infrastructure deployment for nodes.
- **glue / math / parallel / actor** — Default cross-primitive constructions, math objects, parallel fold operations, safe concurrent component coordination.

Commonware also promotes **Decoupled State Machine Replication (DSMR)** — separating ordering/consensus from execution so the chain can absorb load spikes. This decoupling is exactly what makes plugging in an external EVM execution layer natural.

## EVM compatibility (Criterion 1) — candid assessment

**There is no EVM in Commonware.** The library stops at consensus + ordering + storage + runtime. The examples (`alto`, `battleware`, `chat`, `bridge`, etc.) are custom state machines, **not EVM chains**. Alto, the flagship example/benchmark chain, has a minimal bespoke state machine and is explicitly **ALPHA** ("not yet recommended for production use"); it has no EVM.

To get 1:1 latest-mainnet EVM compatibility you must bring your own execution layer. The proven path:

- **Tempo** (Stripe + Paradigm payments L1, mainnet **March 18, 2026**; $500M Series A at $5B valuation) is built on **Commonware for consensus (Simplex BFT) + Reth for EVM execution**. Reth is Paradigm's production Ethereum execution client built on **revm**, tracking Ethereum mainnet closely. Tempo is "forked from Ethereum and amended," is fully EVM-compatible, and runs standard Solidity/Foundry tooling. Tempo invested $25M into Commonware and is now also a contributor.

So the realistic recipe for your sovereign EVM chain is: **Commonware (consensus/P2P/runtime) + Reth or a standalone revm integration (execution + Ethereum-faithful state trie + JSON-RPC) glued together via DSMR**, where Commonware orders transactions/blocks and revm/Reth executes them. With Reth you inherit excellent, continuously-updated mainnet compatibility and the full Ethereum tooling ecosystem (wallets, indexers, explorers, JSON-RPC). The cost is that **all of this glue is your engineering**, and you must keep the EVM layer current with Ethereum hardforks yourself (Reth does the heavy lifting, but the integration is yours).

Verdict: out-of-the-box EVM compatibility = **Weak**; achievable compatibility once you integrate revm/Reth = **Strong**, with the integration being substantial, proven-at-scale (Tempo) work.

## Precompiles / Turing-incomplete modules (Criterion 2)

Because you own the execution layer, you have full freedom here. Commonware places no restrictions on what executes — it only orders opaque messages. If you adopt **revm/Reth**, custom precompiles are a first-class, well-trodden pattern: you register additional precompiled contracts at fixed addresses that expose your Turing-incomplete native modules to Solidity contracts as EVM calls. This is exactly the flexibility the requirement asks for.

The flip side, again, is that **Commonware contributes nothing to the precompile mechanism itself** — that flexibility comes from revm, not Commonware. Commonware's contribution is that its decoupled, non-prescriptive design never gets in your way. Verdict: **Strong flexibility, fully DIY.**

## Consensus (Criterion 3) — Commonware's strongest area

Documented in `commonware-consensus` (docs.rs) and the official blogs. Core dialects:

- **Simplex** — "Simple and fast BFT agreement inspired by Simplex Consensus." Standard BFT safety: tolerates f Byzantine faults of 3f+1 validators (i.e. < 1/3). Two-round propose-and-vote with fast finality. This is the protocol Tempo uses in production.
- **threshold-simplex** — Simplex-like BFT that natively embeds **BLS12-381 threshold cryptography**. When 2f+1 messages are collected in a view, it recovers a threshold signature, emitting a **succinct ~240-byte consensus certificate** per finalized view (verifiable with a single BLS verification). Uses **lazy verification** (verify only on quorum; bisect to find a bad signature if needed). Requires a **DKG** to establish the shared secret, and supports **resharing**, so the network keeps the **same BLS public key across arbitrary validator-set changes (reconfiguration)** — extremely useful for light clients and cross-chain certificate verification. Embeds a VRF for leader election. Tolerance f of 3f+1.
- **Minimmit** — A newer propose-and-vote construction optimized to **minimize block time** (advances views at a ~40% quorum). Reported ~50–130ms block time and ~100–250ms finality depending on validator topology. Tradeoff: tolerates **< 20%** Byzantine replicas (lower than the 1/3 BFT norm) in exchange for speed. Formal spec + correctness proof on arXiv (Aug 2025); flagged by the authors as "not yet peer-reviewed or fully implemented."
- **marshal** — Ordered delivery of finalized blocks (post-consensus sequencing).
- **aggregation** — Recovers quorum certificates over an externally synchronized sequencer of items.
- **ordered_broadcast** — Ordered, reliable broadcast across **reconfigurable** participants.

**Finality:** Deterministic BFT finality (not probabilistic). Alto benchmarks ~200ms block time / ~300ms finality; Minimmit targets even lower.

**PoS / DPoS / PoA layering:** Commonware provides the *agreement* engine over a validator set; it does not prescribe **how that set is chosen or weighted**. That stake/identity layer is yours to build:
- **PoA** — A fixed/permissioned validator set: the simplest layering; just configure the participant set and rotate it via the reconfiguration/resharing support.
- **PoS / DPoS** — Implement stake accounting (typically inside your execution layer / staking contracts), then feed the resulting weighted or elected validator set into the consensus participant set each epoch. threshold-simplex's resharing and ordered_broadcast's reconfigurable participants are designed for exactly this kind of dynamic membership.

Verdict: **Strong.** Modern, fast, formally-described BFT with dynamic membership and succinct certificates. PoS/DPoS/PoA economics are a thin (but DIY) layer on top.

## Maintenance, team, funding, maturity (Criterion 4)

- **Maintainer / team:** Commonware, Inc., founded by **Patrick O'Grady** (ex-VP of Engineering at Ava Labs/Avalanche; previously led Coinbase's Rosetta; Stanford CS). Contributors include Roberto Bayardo, Lucas Meier, Guru Vamsi Policharla, Ben Clabby, Brendan Chou.
- **Funding:** ~$9M seed led by **Haun Ventures** and **Dragonfly** (2024); **$25M strategic** from **Tempo** (Nov 2025). Coinbase Ventures co-led the "Route 66" initiative (May 2026). Collaboration with **LayerZero** on QMDB.
- **Release cadence:** Roughly **monthly**. Recent: v2026.5.0 (May 28 2026), v2026.4.0 (Apr 14 2026), v2026.3.0 (Mar 9 2026), v2026.2.0 (Feb 5 2026); earlier v0.0.x scheme through Jan 2026. 16 releases total. ~582 GitHub stars.
- **Quality signals:** 93% test coverage, 1500 daily benchmarks, deterministic-runtime simulation testing, formal (Quint/arXiv) specs for Minimmit.
- **Production / chains built on it:** **Tempo** (Stripe/Paradigm stablecoin L1, mainnet March 2026, EVM via Reth) is the marquee production user and contributor. **Alto** is the project's own benchmark/testnet chain (ALPHA, no EVM). **Battleware**, **bridge**, etc. are demo examples.
- **Maturity:** The consensus and storage primitives are battle-tested enough that Tempo runs them in production. But the library overall is **young and fast-moving** — APIs evolve release-to-release, several primitives are labelled with varying stability, and Alto is explicitly alpha. Expect to track upstream changes.

Verdict: **Strong** — actively and professionally maintained, well-funded, with a real production deployment — tempered by its youth and churn.

## Overall pros / cons for a sovereign EVM chain

**Pros**
- Best-in-class, modern BFT consensus (Simplex/threshold-simplex/Minimmit) with sub-second deterministic finality, succinct BLS certificates, and dynamic validator reconfiguration — ideal foundation for PoA/PoS/DPoS.
- Total architectural freedom (anti-framework): you control execution, so precompiles and custom modules are unconstrained.
- DSMR decoupling makes bolting on an external EVM execution layer (revm/Reth) natural — and **proven at scale by Tempo**.
- Excellent engineering hygiene (deterministic-sim testing, 93% coverage, daily benchmarks), strong team and funding, monthly releases.
- Permissive MIT/Apache-2.0 license; pure Rust.

**Cons**
- **No EVM, no execution layer.** EVM compatibility, the Ethereum state trie, JSON-RPC, mempool semantics, and hardfork tracking are all DIY — you must integrate and maintain revm/Reth yourself.
- Far more upfront engineering than a turnkey stack (e.g. an OP Stack / Reth-based rollup framework or a Cosmos-EVM SDK). You are assembling a node, not configuring one.
- Storage primitives (QMDB/ADB/MMR) are Commonware's own commitment scheme, not Ethereum-compatible — if you use Reth you'd use Reth's trie and largely not use these.
- Young, churning APIs; smaller community than mature frameworks; Alto is alpha and not an EVM reference.
- Best fit for teams with deep Rust/distributed-systems expertise who want maximum control, not for teams wanting a chain "out of the box."

**Bottom line:** Commonware is an outstanding *consensus + infrastructure* foundation and a credible base for a high-performance sovereign EVM chain — but only if you treat it as the consensus/networking/runtime layer and pair it with revm/Reth for execution (the Tempo model). It is a build-your-own-blockchain toolkit, not a turnkey EVM stack. Choose it for control and consensus quality; avoid it if you need EVM compatibility to come for free.

## Sources

- [Commonware official site](https://commonware.xyz/)
- [Commonware monorepo (GitHub)](https://github.com/commonwarexyz/monorepo)
- [Commonware GitHub org](https://github.com/commonwarexyz)
- [Commonware monorepo releases](https://github.com/commonwarexyz/monorepo/releases)
- [Anti-Framework Philosophy (DeepWiki)](https://deepwiki.com/commonwarexyz/monorepo/1.1-anti-framework-philosophy)
- [consensus/README (GitHub)](https://github.com/commonwarexyz/monorepo/blob/main/consensus/README.md)
- [commonware-consensus on docs.rs](https://docs.rs/commonware-consensus/latest/commonware_consensus/)
- [threshold_simplex on docs.rs](https://docs.rs/commonware-consensus/latest/commonware_consensus/threshold_simplex/index.html)
- [simplex on docs.rs](https://docs.rs/commonware-consensus/latest/commonware_consensus/simplex/index.html)
- [Blog: Many-to-Many Interoperability with Threshold Simplex](https://commonware.xyz/blogs/threshold-simplex)
- [Blog: Minimmit — Fast Finality with Even Faster Blocks](https://commonware.xyz/blogs/minimmit)
- [Minimmit paper (arXiv 2508.10862)](https://arxiv.org/html/2508.10862v7)
- [Blog: Once a Validator, Not Always a Validator (reshare)](https://commonware.xyz/blogs/reshare)
- [Blog: Introducing Commonware](https://commonware.xyz/blogs/introducing-commonware)
- [Blog: The Simplest Database You Need (adb-any)](https://commonware.xyz/blogs/adb-any)
- [Blog: QMDB All The Things](https://commonware.xyz/blogs/qmdb)
- [Blog: Grafting Trees to Prove Current State (adb-current)](https://commonware.xyz/blogs/adb-current)
- [commonware-storage on docs.rs](https://docs.rs/commonware-storage)
- [QMDB paper (arXiv 2501.05262)](https://arxiv.org/html/2501.05262v2)
- [Alto site](https://alto.commonware.xyz/)
- [Alto repo (GitHub)](https://github.com/commonwarexyz/alto)
- [Inside Commonware (Decipher Media)](https://medium.com/decipher-media/inside-commonware-50c58211953c)
- [Patrick O'Grady — personal site](https://patrickogrady.xyz/)
- [The Block: Former Ava Labs lead raises $9M for anti-framework tools](https://www.theblock.co/post/330443/former-ava-labs-engineering-lead-raises-9-million-to-build-anti-framework-blockchain-developer-tools)
- [Fortune: Stripe-backed Tempo leads $25M raise for Commonware](https://fortune.com/2025/11/07/stripe-paradigm-tempo-commonware-strategic-investment-25-million/)
- [Tempo official site](https://tempo.xyz/)
- [Analysis: Stripe's Tempo (lex.substack.com)](https://lex.substack.com/p/analysis-stripes-tempo-is-building)
- [Tempo Review 2026 (Crypto Adventure)](https://cryptoadventure.com/tempo-review-2026-stripe-and-paradigms-payments-first-stablecoin-chain/)
