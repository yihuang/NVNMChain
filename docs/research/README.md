# EVM Chain Implementations — Research Index & Comparison

**Last verified: 2026-06-24**

Deep research into full-stack EVM chain implementations (consensus + EVM execution + precompiles), evaluated against four requirements:

1. **1:1 compatibility with the latest Ethereum mainnet** — so all existing tooling (MetaMask, Hardhat, Foundry, ethers.js, etc.) works flawlessly.
2. **Turing-incomplete custom modules exposed as EVM contracts via precompiles** — ability to add native functions and surface them to Solidity at fixed addresses.
3. **Pluggable consensus** — BFT, DPoS, PoS, and PoA.
4. **Actively maintained** — current releases, healthy cadence, identifiable maintainer.

Each project has a dedicated report in this directory with full detail and sources.

---

## Comparison matrix

Legend: 🟢 Strong · 🟡 Partial · 🔴 Weak / not the model · ⚫ Dead/EOL

| Project | (1) Latest-mainnet 1:1 | (2) Custom precompile modules | (3) BFT/DPoS/PoS/PoA | (4) Maintained | Category |
|---|---|---|---|---|---|
| **[Cosmos EVM](cosmos-evm.md)** | 🟢 Prague/Pectra; minor deviations (base-fee→validators, no blobs) | 🟢 Signature feature — SDK modules (staking/bank/IBC/gov) as stateful precompiles | 🟢 CometBFT BFT + PoS/DPoS; 🟡 PoA manual | 🟢 v0.7.0 (May 2026); **pre-v1.0** | Sovereign L1 SDK |
| **[Avalanche Subnet-EVM](avalanche-subnet-evm.md)** | 🟡 Cancun (no Pectra yet) | 🟢 Stateful precompiles + `precompilegen`, no fork needed | 🟢 Snowman BFT; PoA/PoS/DPoS via ValidatorManager | 🟢 Active; standalone repo merged into AvalancheGo monorepo (Dec 2025) | Sovereign L1 (Avalanche) |
| **[Polkadot Frontier](polkadot-frontier.md)** | 🟢 Prague (but EVM *emulation* over Substrate; gas/account deviations) | 🟢 Pallets-as-precompiles (Moonbeam `precompile-utils`) | 🟢 Aura(PoA)/BABE+GRANDPA(PoS+BFT)/NPoS(DPoS)/PoW | 🟢 Active; **caveat:** Parity shifting to REVM + `pallet-revive` (Polkadot Hub) | Substrate parachain/L1 |
| **[Reth SDK](reth-sdk.md)** | 🟢 Best fidelity — Fusaka live, Glamsterdam in progress | 🟢 revm precompile injection via NodeBuilder/`ConfigureEvm` + ExEx | 🟡 Execution-only; consensus-agnostic (bring your own) | 🟢 v2.3.0 (Jun 2026); heavy production use | Execution client / SDK |
| **[Commonware](commonware.md)** | 🔴 No EVM out of the box (DIY; pair with Reth/revm) | 🟢 Full control if paired with revm — but you build it | 🟢 Standout: Simplex/Minimmit BFT; PoS/DPoS/PoA as thin layers | 🟢 v2026.5.0 (May 2026); young, fast-churning APIs | Blockchain primitives toolkit |
| **[Tempo (Stripe)](tempo-stripe.md)** | 🟡 Reth/revm on Osaka; payments deviations (no native gas token, custom tx type 0x76, TIP-20) | 🟡 Reth/revm precompiles proven (TIP-20, Fee Manager, DEX), but third parties can't add their own on the live chain | 🟡 Commonware Simplex BFT over a **permissioned/PoA** validator set; PoS roadmapped/undated | 🟢 Mainnet live 18 Mar 2026; $500M Series A | Payments L1 (network, not a framework) |
| **[Hyperledger Besu](hyperledger-besu.md)** | 🟢 Real mainnet client; Fusaka/PeerDAS (Dec 2025) | 🔴 No plugin hook for precompiles — requires custom build/fork | 🟢 QBFT/IBFT2 (BFT-PoA); 🔴 no DPoS; PoS only as mainnet EL | 🟢 26.6.1 (Jun 2026), ~monthly | Enterprise client |
| **[OP Stack](op-stack.md)** | 🟢 EVM-equivalent (op-geth/op-reth); Isthmus=Pectra, Jovian=Fusaka | 🟡 Predeploys configurable; true precompiles need a fork | 🔴 Rollup — centralized sequencer; security from L1 PoS + fault proofs | 🟢 Active; **caveats:** op-geth EOL 31 May 2026; Base leaving shared stack | L2/L3 rollup framework |
| **[Arbitrum Orbit](arbitrum-orbit.md)** | 🟢 Geth-fork EVM-equivalent; ArbOS 51 "Dia" (Fusaka parts, Jan 2026) | 🟢 ArbOS precompiles + Stylus (WASM); custom precompiles limited by fraud-proof determinism | 🔴 Rollup/AnyTrust — sequencer + BoLD fraud proofs / DAC | 🟢 Nitro v3.11.0 (Jun 2026); BUSL-1.1 license | L2/L3 rollup framework |
| **[Polygon Edge / CDK](polygon-edge-cdk.md)** | 🔴 Edge frozen pre-Dencun; 🟡 CDK zkEVM near-Type-2 (deviations) | 🔴 Hardcoded precompiles; CDK also needs prover/ROM changes | 🟢 Edge IBFT PoA/PoS (BFT); 🔴 CDK central sequencer | ⚫ **Edge archived Dec 2024**; 🟢 CDK active | Edge=L1; CDK=ZK-L2 |
| **[GoQuorum](goquorum.md)** | 🔴 geth fork capped at **Berlin** — no EIP-1559/Merge | 🟡 Standard geth precompile pattern, but fork-and-recompile only | 🟡 QBFT/IBFT/Raft/Clique; 🔴 no PoS/DPoS | ⚫ **Repo archived 5 Jun 2026 — EOL** | Enterprise client (dead) |
| **[Berachain BeaconKit](berachain-beaconkit.md)** | 🟢 EVM-identical (unmodified EL); Fusaka via v1.4.1 (Jun 2026) | 🔴 Custom precompiles deliberately removed; native logic via system contracts | 🟡 CometBFT BFT + PoS/Proof-of-Liquidity; 🔴 no DPoS/PoA | 🟢 Active; BUSL-1.1 | Consensus-client framework |

---

## Verdict & recommendations

**No single off-the-shelf project hits all four requirements at 🟢.** The fundamental tension is requirement (1) "1:1 with *latest* mainnet" vs. requirement (2) "add native modules as precompiles" — the projects that bend the EVM to expose native modules (Cosmos EVM, Frontier, Avalanche) accept small deviations or fork lag, while the projects that stay perfectly mainnet-faithful (Besu, Reth, BeaconKit) make custom precompiles hard or off-limits.

Ranked for the stated use case (**sovereign EVM L1, pluggable BFT/DPoS/PoS/PoA, native modules as precompiles, current mainnet**):

1. **Cosmos EVM** — best turnkey all-rounder. Precompile-exposed native modules are its core design, CometBFT gives BFT + PoS/DPoS, and it tracks Prague/Pectra. Watch: pre-v1.0, two critical precompile CVEs in the past year, PoA only via manual permissioning.
2. **Avalanche Subnet-EVM** — equally strong on precompiles and now on consensus (PoA/PoS/DPoS via on-chain ValidatorManager post-Etna), with leaderless BFT finality. Main gap: trails mainnet at Cancun (no Pectra yet).
3. **Reth SDK + Commonware** — unbeatable mainnet fidelity and clean revm precompile injection, with Commonware Simplex BFT for sub-second finality. Best-in-class performance and full control. **Tempo (Stripe + Paradigm) is the live proof of this exact stack AND its node is open source** (`github.com/tempoxyz/tempo`, Apache-2.0/MIT) — its `crates/consensus` is a copyable reference for the consensus↔Reth glue, so this is now a **fork-strip-extend** exercise, not greenfield. You still own the stack forever (every Reth/Commonware bump is yours) and Commonware consensus is ALPHA. See [`tempo-as-blueprint.md`](../architecture/tempo-as-blueprint.md) and [`commonware-reth-glue.md`](../architecture/commonware-reth-glue.md).
4. **Polkadot Frontier** — richest consensus menu and mature pallets-as-precompiles, but it's an emulation layer (not literal 1:1) and Parity's strategic EVM investment is moving to REVM/`pallet-revive`.
5. **Hyperledger Besu** — pick only if mainnet fidelity + BFT/PoA permissioning matter more than extensibility; custom precompiles require a maintained fork, and there's no DPoS.

**Not recommended for this use case:** OP Stack and Arbitrum Orbit (L2 rollup frameworks — wrong consensus category; security derives from L1, not pluggable validator consensus); Polygon Edge and GoQuorum (both archived/EOL); BeaconKit if custom precompiles are a hard requirement (intentionally traded away).

---

## Also considered (not given full reports)

- **Monad, Sei v2, Kaia** — high-performance EVM **L1s**, not reusable frameworks for launching your own chain.
- **Ethermint, Polaris, Kava EVM** — superseded by Cosmos EVM / BeaconKit respectively.
- **Erigon, Nethermind, Geth** — mainnet execution clients; like Reth, execution-only and consensus-agnostic (Reth chosen as the representative modular SDK).

## Reports in this directory

- [cosmos-evm.md](cosmos-evm.md)
- [avalanche-subnet-evm.md](avalanche-subnet-evm.md)
- [polkadot-frontier.md](polkadot-frontier.md)
- [reth-sdk.md](reth-sdk.md)
- [commonware.md](commonware.md)
- [tempo-stripe.md](tempo-stripe.md)
- [tempo-as-blueprint.md](../architecture/tempo-as-blueprint.md) — how Tempo wires Commonware↔Reth (glue, open-source)
- [commonware-reth-glue.md](../architecture/commonware-reth-glue.md) — engineering map of the integration
- [stablecoin-gas-mechanisms.md](../architecture/stablecoin-gas-mechanisms.md) — paying gas in any stablecoin (enshrined vs paymaster)
- [precompiles-turing-incomplete-modules.md](../architecture/precompiles-turing-incomplete-modules.md) — R2 deep-dive: native modules as precompiles
- [ADR-0001-evm-chain-implementation.md](../../ADR-0001-evm-chain-implementation.md) — **the decision record**
- [SPIKE-0001-reth-commonware-tempo.md](../../SPIKE-0001-reth-commonware-tempo.md) — **3-week validation spike + go/no-go gate**
- [PROPOSAL-0001-fork-tempo.md](../../PROPOSAL-0001-fork-tempo.md) — **the fork-Tempo implementation proposal (keep/adapt/strip/build, phases, effort)**
- [ISSUE-TREE-fork-tempo.md](../../ISSUE-TREE-fork-tempo.md) — **epic → issue breakdown with IDs, dependencies, estimates, acceptance criteria**
- [hyperledger-besu.md](hyperledger-besu.md)
- [op-stack.md](op-stack.md)
- [arbitrum-orbit.md](arbitrum-orbit.md)
- [polygon-edge-cdk.md](polygon-edge-cdk.md)
- [goquorum.md](goquorum.md)
- [berachain-beaconkit.md](berachain-beaconkit.md)
