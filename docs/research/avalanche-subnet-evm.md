# Avalanche Subnet-EVM / Coreth (Custom Avalanche L1 framework)

**Projects covered:** [`ava-labs/subnet-evm`](https://github.com/ava-labs/subnet-evm), [`ava-labs/coreth`](https://github.com/ava-labs/coreth), [`ava-labs/precompile-evm`](https://github.com/ava-labs/precompile-evm), built on [`ava-labs/avalanchego`](https://github.com/ava-labs/avalanchego).
**Last verified:** 2026-06-24

---

## Executive summary

Subnet-EVM is Ava Labs' production EVM virtual machine for building custom Avalanche L1s (the networks formerly called "subnets"). It is a simplified fork of Coreth (the C-Chain VM) that runs as a plugin inside an AvalancheGo node and exposes a near-identical-to-Ethereum execution and JSON-RPC surface, while adding Avalanche's signature feature: **stateful precompiles** — native Go modules surfaced to the EVM as fixed-address contracts (fee manager, deployer/transaction allowlists, native minter, reward manager, Warp messenger), plus a `precompilegen` toolchain (and the `precompile-evm` library repo) for writing your own. Consensus is provided by AvalancheGo's leaderless, sub-second-finality **Snowman** protocol (the Snowball/Snowflake/Slush family), and validator-set management is configurable as **PoA or PoS** via the on-chain ValidatorManager contracts introduced with the Avalanche9000/Etna upgrade (Dec 2024) that renamed subnets to L1s. The code is actively maintained by Ava Labs but **as of late 2025 the standalone subnet-evm and coreth repos were folded into the AvalancheGo monorepo** (subnet-evm repo archived Dec 16, 2025; last standalone tag v0.8.0, Nov 5, 2025). The single most important compatibility caveat for a "latest Ethereum mainnet" comparison: **Subnet-EVM targets the Cancun hardfork and does NOT yet support Prague/Pectra**, so it trails Ethereum mainnet by one hardfork.

---

## Fit against requirements

### 1. 1:1 compatibility with the LATEST Ethereum mainnet — **Partial (Strong on tooling, one hardfork behind)**
Subnet-EVM implements a standards EVM and is "compatible with almost all Ethereum tooling, including Foundry, Remix, Hardhat, MetaMask, ethers.js." JSON-RPC is the standard go-ethereum `eth` namespace plus Avalanche extensions. **However it targets the Cancun fork and does not yet support Pectra/Prague** — Ava Labs explicitly tells developers to set Solidity `evmVersion: cancun`. There are also intentional block-format deviations (BlockGasCost surcharge, millisecond timestamps as of Granite, Avalanche-specific dynamic-fee fields). So it is excellent for existing tooling but is **not** literally 1:1 with the latest Ethereum mainnet hardfork.

### 2. Turing-incomplete extension modules exposed as EVM precompiles — **Strong**
This is exactly Subnet-EVM's flagship capability. The **stateful precompile** system lets you implement arbitrary native Go logic, give it access to EVM state, and surface it at a fixed `0x02000...` contract address callable from Solidity. Six battle-tested built-ins ship (allowlists, native minter, fee manager, reward manager, Warp). Custom precompiles are scaffolded with the `precompilegen` codegen tool from an ABI, and the `precompile-evm` repo lets you register them **without forking** subnet-evm. Best-in-class for this criterion.

### 3. Consensus support (BFT / DPoS / PoS / PoA) — **Strong**
Snowman consensus is a novel leaderless, metastable BFT-family protocol with sub-second probabilistic finality. Validator-set governance is pluggable via on-chain ValidatorManager contracts: **PoA** (PoAManager, admin-controlled permissioned set), **PoS** (NativeTokenStakingManager / ERC20TokenStakingManager — permissionless, with delegation = effectively DPoS-style). All four requested models are achievable. Caveat: it is *Snowman/Snowball*, not classical PBFT/Tendermint or a pluggable consensus engine — you get Ava Labs' consensus, configured PoA-or-PoS, not a choice of consensus algorithm.

### 4. Actively maintained — **Strong (with a structural change to note)**
Maintained by Ava Labs, the original Avalanche team. Frequent releases tracking AvalancheGo (Fortuna Mar 2025, Granite Nov 2025). Latest standalone subnet-evm tag **v0.8.0 (Nov 5, 2025)**; `precompile-evm` v0.4.0 (Dec 10, 2025); AvalancheGo at **v1.14.x (Granite, 2026)**. **Important:** subnet-evm and coreth have been consolidated into the AvalancheGo monorepo and the standalone subnet-evm repo was archived (read-only) on Dec 16, 2025 — development continues, but in avalanchego, not the old repo.

---

## High-level architecture

- **Language:** Go (requires Go 1.24.9+). **License:** the EVM/precompile code is LGPL-3.0 (inherited go-ethereum lineage; precompile-evm is LGPL-3.0); BSD-3 components exist in the broader stack.
- **VM plugin model:** AvalancheGo is the node/networking/consensus engine. EVM logic lives in a VM plugin. Subnet-EVM is a "simplified version of Coreth" — Coreth runs the Primary Network **C-Chain**; Subnet-EVM runs **custom L1 chains**. Each L1 chain is one instance of the VM plugin, launched and validated by a subset of nodes.
- **Avalanche multi-chain structure:** the Primary Network has the P-Chain (platform/staking/validator registry), X-Chain (assets), and C-Chain (Coreth EVM). Custom L1s are additional chains validated by their own validator sets.
- **block.Accept/Reject** hooks integrate the EVM with Snowman's decision process (a block is final once Accepted — no reorgs).
- **Genesis & upgrade config:** chain behavior, fees, and which precompiles are active are set in `genesis.json` `chainConfig`, and precompiles can also be turned on/off later via timestamped network-upgrade config — no hard fork of the binary required.

## Ethereum compatibility detail

- Implements the **Cancun** EVM. **Pectra/Prague not yet supported** (as of v0.8.0 / Granite, 2026). Developers must pin Solidity `evmVersion=cancun`.
- JSON-RPC: standard go-ethereum `eth`, `net`, `web3`, `debug`, `txpool` namespaces (only `eth` enabled by default), plus Avalanche-specific APIs (e.g. `eth_getChainConfig`/fee-config-at-block, `avax`-prefixed methods on Coreth). "Subnet-EVM APIs are identical to Coreth C-Chain APIs except for the avax-prefixed ones."
- EIP-1559 dynamic fees supported, but with Avalanche's own fee mechanics (ACP-176 dynamic gas limits via Fortuna; ACP-226 dynamic minimum block times and millisecond block timestamps via Granite). Block header includes `BaseFee` and `BlockGasCost`.
- **Finality:** unlike Ethereum's probabilistic-then-finalized model, accepted blocks are immediately final (no reorgs); `safe`/`finalized` tags behave accordingly. Tooling that assumes PoW/PoS reorg windows generally still works.
- secp256r1 precompile (ACP-204) added in Granite for biometric/passkey use cases.

## Stateful precompiles (the extension system)

A stateful precompile is native Go that (a) lives at a reserved address (`0x0200000000000000000000000000000000000000`–`...0005` for the built-ins), (b) can read/write EVM state, and (c) is invoked like a normal contract from Solidity — faster and cheaper than equivalent Solidity. Built-in precompiles:

| Precompile | Purpose |
|---|---|
| **Contract Deployer AllowList** | Restrict who may deploy contracts |
| **Transaction AllowList** | Restrict who may submit transactions (permissioned chain) |
| **Native Minter** | Mint/burn the native gas token to authorized addresses post-launch |
| **Fee Manager** | Configure the dynamic-fee parameters on-chain |
| **Reward Manager** | Direct where transaction-fee rewards go (validators/address/burn) |
| **Warp Messenger** | Avalanche Interchain Messaging (ICM/Teleporter) — cross-L1 native messaging |

**AllowList interface** (shared by allowlist-style precompiles) defines three roles: **Admin** (manage other addresses + enabled), **Manager** (manage enabled, added in later versions), **Enabled** (use the function), and disabled = no access.

**Writing your own:**
1. Define the Solidity interface + ABI in `contracts/`.
2. Run `./scripts/generate_precompile.sh --abi <abi> --out <dir>` which installs and runs `precompilegen` to scaffold a self-registering Go precompile module.
3. Register in `plugin/main.go` (via the `precompile/modules` package) and build with `./scripts/build.sh`.
- The **`precompile-evm`** repo (LGPL-3.0, v0.4.0 Dec 2025) imports subnet-evm as a library so you can ship custom precompiles **without forking**; `hello-world-example` branch + Avalanche Academy "Customizing the EVM" course are the canonical tutorials.

## Consensus & validators

- **Snowman**: linearized form of Avalanche consensus for total-order chains (EVM needs linear ordering). Built on Slush → Snowflake → Snowball. Leaderless: each node repeatedly samples k≈20 random validators (stake-weighted) and flips preference after α agreement, finalizing after β consecutive successes. Transitive voting (a vote for a block votes for all ancestors).
- **Properties:** BFT-family but not classical PBFT; combines classical + Nakamoto ideas. **Sub-second, irreversible (probabilistic but practically immediate) finality**; safety holds while a sufficient majority of bonded AVAX is honest. Scales to thousands of validators without all-to-all messaging.
- **Validator management (post-Etna / ACP-77):** L1 validators no longer must validate the Primary Network and pay a continuous P-Chain fee (~1.3 AVAX/month initially) instead of bonding 2000 AVAX. Validator set is governed by the **ValidatorManager** smart contracts (in `ava-labs/icm-contracts`):
  - **PoA:** `PoAManager` — only the owner/admin initiates add/remove/weight changes (permissioned validator set). Ideal for enterprise/consortium chains.
  - **PoS:** `NativeTokenStakingManager` and `ERC20TokenStakingManager` (both permissionless, support delegation → DPoS-like, uptime-based rewards, staking fees).
  - Changes are committed to the P-Chain via signed Warp/ICM messages quorum-signed by the current validator set.

## Maintenance, releases & the subnet→L1 rename

- **Maintainer:** Ava Labs.
- **Avalanche9000 / Etna upgrade (mainnet Dec 16, 2024):** largest protocol change in Avalanche history; 7 ACPs, headlined by **ACP-77** — renamed "subnets" to **Avalanche L1s**, decoupled them from Primary Network validation, replaced large AVAX bonds with a small continuous fee, and introduced the on-chain validator-manager model. Drastically lowered the cost/complexity of launching a sovereign L1.
- **Fortuna (AvalancheGo v1.13.0, Mar 24 2025 / mainnet Apr 8 2025):** ACP-176 C-Chain fee overhaul, dynamic EVM gas limits.
- **Granite (AvalancheGo v1.14.0, released Nov 5 2025 / mainnet Nov 19 2025; subnet-evm v0.8.0):** ACP-181 P-Chain epoched views for ICM, ACP-204 secp256r1 precompile, ACP-226 dynamic minimum block times + millisecond block timestamps. v1.14.2 ("Granite.2", 2026) = benchlist redesign, optional/backwards-compatible.
- **Repo consolidation:** subnet-evm and coreth moved into the AvalancheGo monorepo; **subnet-evm repo archived (read-only) Dec 16, 2025**, last standalone tag **v0.8.0**. File issues/PRs in avalanchego now. `precompile-evm` remains its own repo (v0.4.0, Dec 10 2025).
- Version mapping: subnet-evm v0.8.0 ↔ AvalancheGo v1.14.0 (plugin/protocol v44); v0.7.9 ↔ v1.13.5 (v43).

## Notable production L1s

- **Beam** (gaming; Merit Circle DAO; BEAM gas; >4.5M wallets, 100+ studios as of Q1 2026)
- **Dexalot** (on-chain order-book DEX; DEXALOT gas; MEV-resistant deterministic execution)
- **DeFi Kingdoms** (gaming/DeFi; JEWEL gas; one of the earliest subnets)
- **GUN / Gunzilla** (Off the Grid AAA shooter; GUN gas)
- Plus institutional/permissioned pilots. Most active L1s (Beam, Dexalot) migrated to the L1 model in Q1 after Etna.

## Security / audit posture

- Inherits the heavily-reviewed go-ethereum EVM lineage; Coreth/C-Chain has run Avalanche mainnet value since 2020. AvalancheGo and the ICM/validator-manager contracts undergo Ava Labs internal review and third-party audits, and major upgrades (Etna, Granite) ship via the public ACP (Avalanche Community Proposal) process with testnet (Fuji) activation before mainnet. Specific per-release audit reports were not enumerated in the sources reviewed; consult the ACP repo and Ava Labs audit disclosures for a given release before production use.

## Frank pros/cons vs the 4 criteria

**Pros**
- Best-in-class stateful-precompile system for adding native Turing-incomplete modules as EVM contracts (criterion 2).
- Excellent existing-tooling compatibility (MetaMask/Hardhat/Foundry/ethers) and fast, final, no-reorg execution.
- Genuine PoA *and* PoS/DPoS validator models via swappable on-chain ValidatorManager; sub-second BFT finality (criterion 3).
- Mature, production-proven (Beam, Dexalot, etc.), actively developed by Ava Labs, frequent ACP-driven upgrades (criterion 4).
- Post-Etna L1 model makes launching a sovereign chain cheap and fast.

**Cons**
- **One hardfork behind mainnet: Cancun, not Pectra/Prague** — the key gap against "1:1 with latest Ethereum mainnet" (criterion 1).
- Several intentional block-format/fee deviations from vanilla Ethereum (BlockGasCost, ms timestamps, Avalanche fee mechanics) — almost always transparent to tooling but not literally identical.
- Consensus is fixed to Snowman (not a pluggable engine; not classical PBFT/Tendermint). You configure PoA/PoS *on top of* Snowman, you don't swap consensus.
- Structural churn: standalone subnet-evm/coreth repos archived and merged into the AvalancheGo monorepo (late 2025) — existing integrations pinning the old repo must migrate.
- Custom precompiles require Go and rebuilding the VM binary (the precompile-evm repo softens but does not remove this).

---

## Sources

- [ava-labs/subnet-evm (GitHub)](https://github.com/ava-labs/subnet-evm)
- [subnet-evm Releases](https://github.com/ava-labs/subnet-evm/releases)
- [subnet-evm RELEASES.md](https://github.com/ava-labs/subnet-evm/blob/master/RELEASES.md)
- [subnet-evm README](https://github.com/ava-labs/subnet-evm/blob/master/README.md)
- [ava-labs/precompile-evm (GitHub)](https://github.com/ava-labs/precompile-evm)
- [ava-labs/avalanchego Releases](https://github.com/ava-labs/avalanchego/releases)
- [AvalancheGo v1.14.2 (Granite.2) release](https://github.com/ava-labs/avalanchego/releases/tag/v1.14.2)
- [AvalancheGo v1.13.0 (Fortuna) release](https://github.com/ava-labs/avalanchego/releases/tag/v1.13.0)
- [EVM L1 Customization (Builder Hub)](https://build.avax.network/docs/avalanche-l1s/evm-configuration/evm-l1-customization)
- [Native Minter precompile](https://build.avax.network/docs/avalanche-l1s/precompiles/native-minter)
- [Fee Manager precompile](https://build.avax.network/docs/avalanche-l1s/precompiles/fee-manager)
- [Generating Your Precompile](https://build.avax.network/docs/virtual-machines/custom-precompiles/create-precompile)
- [Custom Precompiles intro](https://docs.avax.network/evm-l1s/custom-precompiles/introduction)
- [Customizing the EVM with Stateful Precompiles (Aaron Buchwald, Medium)](https://medium.com/avalancheavax/customizing-the-evm-with-stateful-precompiles-f44a34f39efd)
- [Snowman Consensus (Builder Hub)](https://build.avax.network/docs/primary-network/avalanche-consensus)
- [Validator Manager Contracts](https://build.avax.network/docs/avalanche-l1s/validator-manager/contract)
- [PoA vs PoS](https://build.avax.network/docs/avalanche-l1s/validator-manager/poa-vs-pos)
- [Proof of Authority (Academy)](https://build.avax.network/academy/l1-validator-management/03-deploy-validator-manager/00-proof-of-authority)
- [ava-labs/icm-contracts validator-manager](https://github.com/ava-labs/icm-contracts/tree/main/contracts/validator-manager)
- [Validator Rewards and Staking Mechanisms](https://build.avax.network/blog/staking-and-validator-management)
- [Etna: Enhancing the Sovereignty of Avalanche L1 Networks](https://www.avax.network/about/blog/etna-enhancing-the-sovereignty-of-avalanche-l1-networks)
- [Avalanche9000 Upgrade (Academy)](https://build.avax.network/academy/avalanche-l1/avalanche-fundamentals/03-multi-chain-architecture-intro/03a-etna-upgrade)
- [What to Expect After the Etna Upgrade](https://build.avax.network/blog/etna-changes)
- [Avalanche9000 goes live (The Block)](https://www.theblock.co/post/331101/anticipated-avalanche9000-upgrade-goes-live-reducing-costs-and-making-it-easier-to-launch-avalanche-subnets)
- [Avalanche Granite Upgrade (Builder Hub)](https://build.avax.network/blog/granite-upgrade)
- [Subnet-EVM RPC reference](https://build.avax.network/docs/rpcs/subnet-evm)
- [Subnet-EVM API reference](https://docs.avax.network/reference/subnet-evm/api)
- [Gunzilla launches AAA shooter as an Avalanche L1](https://www.avax.network/about/blog/gunzilla-launches-aaa-shooter-as-an-avalanche-l1)
- [What Is Avalanche? AVAX, L1s, and Subnets in 2026](https://eco.com/support/en/articles/12168599-what-is-avalanche-avax-l1s-and-subnets-in-2026)
- [Nansen — Avalanche Q1 2026 report](https://nansen.ai/post/avalanche-q1-2026-report)
