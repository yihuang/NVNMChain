# Cosmos EVM (`cosmos/evm`) — Full-Stack EVM Chain Implementation Research

**Last verified: 2026-06-24**

## Executive Summary

Cosmos EVM (the [`cosmos/evm`](https://github.com/cosmos/evm) repository) is the canonical, officially-maintained module for adding a fully spec-compliant Ethereum Virtual Machine to any Cosmos SDK chain. It is the open-sourced descendant of the Ethermint → Evmos → evmOS lineage: the Interchain Foundation funded Tharsis Labs to open-source the evmOS codebase, and it now lives under the `cosmos/` org maintained by Cosmos Labs (a subsidiary of the Interchain Foundation, a.k.a. Interchain Labs), with significant core contributions from B-Harvest, Cronos, and MANTRA. Architecturally it is a clean two-layer system: **CometBFT** (Tendermint-lineage BFT) provides consensus and instant finality at the consensus layer, while a **Cosmos-maintained fork of go-ethereum** provides the execution layer, wired into the Cosmos SDK via ABCI as the `x/vm` (and supporting) modules. It targets the **Prague** EVM (Pectra-era), exposes a near-complete Ethereum JSON-RPC, and works with standard tooling (MetaMask, Hardhat, Foundry, ethers.js). Its standout differentiator is **stateful precompiles** that expose deterministic, non-Turing-complete Cosmos SDK modules (staking, bank, distribution, governance, IBC/ICS-20, slashing, ICS-02, ERC20/WERC20) to Solidity contracts at fixed addresses. It is actively maintained (latest release **v0.7.0**, 2026-05-05; `main` pushed 2026-06-23) but still pre-v1.0 (v1.0 targeted ~Q2 2026 after audit), and has had two **critical** precompile-related security advisories in the past year, including one that caused a real $7M loss on Saga — so the precompile system, while powerful, is also where its highest historical risk has concentrated.

---

## Fit Against Requirements

### Criterion 1 — 1:1 compatibility with the latest Ethereum mainnet: **STRONG (with minor, mostly-benign deviations)**

It targets the **Prague hard fork** (the execution half of Pectra), supports all standard opcodes/EIPs/ERC interfaces up to Prague, supports EIP-1559, EIP-2930, and **EIP-7702 account abstraction** (added natively in v0.5.0), and ships a near-complete Ethereum JSON-RPC so MetaMask/Hardhat/Foundry/ethers.js work out of the box. Deviations are real but mostly do not break tooling: the EIP-1559 base fee is **distributed to validators rather than burned** (and `eth_gasPrice` returns 0 by design — clients must use 1559 fee fields), there is **no EIP-4844 blob support** (deliberately excluded), accounts use a **unified dual-representation** (one secp256k1 key → both a `0x` and a `cosmos1...` bech32 address), and there are **no uncles, no reorgs, and ~2s instant finality**. Verdict is Strong rather than Perfect because of the base-fee/`eth_gasPrice` quirk, missing blob txs, and the dual-address model, none of which typically break MetaMask/Foundry but can trip up tooling that assumes burn semantics, blob support, or pure-Ethereum account identity.

### Criterion 2 — Add Turing-incomplete modules exposed as EVM contracts via precompiles: **STRONG (this is its signature capability)**

This is exactly what Cosmos EVM is built for. Cosmos SDK modules are deterministic, bounded (non-Turing-complete) Go state machines, and the precompile system wraps them behind a Solidity ABI at fixed addresses so EVM contracts can call into them as if calling a normal contract — `delegate()`, `vote()`, `withdrawRewards()`, `transfer()` IBC, etc. — atomically within a single EVM transaction. Out of the box it ships staking, distribution, bank, governance, slashing, ICS-20 (IBC transfer), ICS-02 (IBC client), ERC20/WERC20, plus stateless P256 and Bech32 helpers. Developers add a **custom** precompile by implementing the geth-style stateful precompile interface (`Address()`, `RequiredGas()`, `Run()`) on top of `cmn.Precompile`, holding a reference to their module's keeper, and registering it in the active-precompiles map. Verdict Strong; the only caveats are that adding a precompile is a chain-level (genesis/upgrade) operation, not something a smart-contract author can do at runtime, and that this surface is where the project's critical CVEs have occurred.

### Criterion 3 — Consensus support (BFT, PoS, DPoS, PoA): **STRONG for BFT/PoS/DPoS; PARTIAL/configurable for PoA**

Consensus is **CometBFT** (Tendermint-lineage BFT), giving deterministic single-block (~2s) instant finality with up to 1/3 Byzantine tolerance — so **BFT is native**. **PoS/DPoS is the default**: the Cosmos SDK `x/staking` module bonds the native token to a stake-ranked, capped (`MaxValidators`) active validator set with delegation — which is precisely the DPoS model. **PoA is achievable but not a turnkey mode**: because the chain builder owns the whole stack, PoA is implemented by constraining the validator set to a fixed/permissioned group (gating who can validate, small/fixed set, governance restrictions) and optionally using the permissioned-EVM feature; there is no single "PoA switch." Verdict Strong on BFT/PoS/DPoS, Partial on PoA only because it requires custom configuration rather than a built-in flag.

### Criterion 4 — Actively maintained: **STRONG**

Latest published release **v0.7.0 on 2026-05-05**; `main` last pushed **2026-06-23** (the day before this report). Release cadence is roughly one minor every 1–3 months (v0.3.0 Jul 2025 → v0.7.0 May 2026, ~7 minors in ~10 months) plus patches as needed, with ~419 commits in the trailing year. Maintained by Cosmos Labs / Interchain Foundation with named contributions from B-Harvest, Cronos, and MANTRA, and a stated roadmap toward a v1.0 in ~Q2 2026 after audit. Verdict Strong. The only asterisk is that it is still **pre-v1.0** and explicitly warns of breaking changes across 0.x releases.

---

## Architecture

Cosmos EVM is a **two-layer, single-binary** design, the standard Cosmos pattern:

- **Consensus layer — CometBFT.** Tendermint-lineage Byzantine-fault-tolerant consensus. Orders transactions, produces blocks, and gives deterministic single-block finality. Communicates with the application via ABCI.
- **Execution / application layer — Cosmos SDK + EVM modules.** The Cosmos SDK hosts the application state machine. Cosmos EVM plugs in as a set of modules (the EVM module / `x/vm`, fee-market, ERC20, precompiles, etc.) that embed a **fork of go-ethereum** as the EVM execution engine. EVM state and Cosmos module state live in the same multistore and commit atomically per block.

Other architectural facts:
- **Implementation language:** Go (Go toolchain `go 1.25.9` on `main`).
- **EVM engine:** Cosmos-maintained geth fork — `go.mod` declares `github.com/ethereum/go-ethereum v1.16.8` but **replaces** it with **`github.com/cosmos/go-ethereum v1.17.2-cosmos-0`** (based on upstream `release/1.17`). v0.7.0 bumped this from the 1.16 line to 1.17.
- **Core dependency versions (from `go.mod` on `main`):** Cosmos SDK **v0.54.3** (+ `store/v2 v2.0.0`), CometBFT **v0.39.3**, IBC-go **v11.0.0**.
- **License:** **Apache-2.0** (single license).
- **Self-description:** "a plug-and-play solution that adds EVM compatibility and customizability to your Cosmos SDK chain."
- **Performance work in v0.7.0:** application-side mempool ("Krakatoa"), **BlockSTM parallel execution**, OpenTelemetry tracing, and a JSON-RPC filter-lifecycle overhaul.

### Lineage / history
Ethermint → Evmos → **evmOS** → **cosmos/evm**. The Interchain Foundation funded **Tharsis Labs** to open-source the evmOS codebase; the repo was created on GitHub **2025-03-18**, first GitHub release (v0.3.0) **2025-07-25**, and it is positioned as the *canonical* Cosmos EVM going forward.

---

## Ethereum Compatibility (detailed)

| Aspect | Cosmos EVM behavior |
|---|---|
| **EVM hardfork target** | **Prague** (Pectra-era). Supports all standard opcodes/EIPs/ERC interfaces up to Prague. |
| **EIP-1559** | Supported, but **base fee goes to validators, not burned**; `eth_gasPrice` returns 0 (use 1559 fee fields); base fee can be disabled via `NoBaseFee`; configurable chain-wide min-gas-price floor. |
| **EIP-2930 (access lists)** | Supported. |
| **EIP-7702 (set-code / EOA "AA")** | **Supported** (native, added v0.5.0). Implemented as `SetCodeTx` with authorization lists, standard nonce validation, delegation-proxy bytecode, standard gas costs, EIP-712-compatible auth signatures. |
| **EIP-4844 (blob txs)** | **Not supported** (deliberately excluded; blobs target L2 rollups). `eth_blobBaseFee` stubbed. |
| **EIP-4399 (PREVRANDAO)** | Not supported (no PoW/beacon randomness; uses CometBFT). |
| **Tx types** | Legacy, EIP-2930, EIP-1559, EIP-7702. No blob (4844) txs. |
| **Account model** | **Unified dual-representation**: one **secp256k1** key yields both a `0x…` Ethereum address and a `cosmos1…` bech32 address. **EIP-155** replay protection / chain-ID supported. |
| **Finality / blocks** | ~1–2s block time; **instant single-block finality, no reorgs** (CometBFT BFT). No uncles/ommers. |
| **JSON-RPC** | Near-complete Ethereum JSON-RPC over HTTP + WS. Default namespaces `eth_`, `net_`, `web3_`; configurable `debug_` (partial), `txpool_`, `personal_`; plus a Cosmos-specific `cosmos` namespace. Not implemented/N/A: `engine_`, `admin_`, `clique_`, `les_`, `miner_` (no PoW/beacon engine). Uncle methods return `0x0`/`null`; `eth_mining` always false; `eth_hashrate` `0x0`. |
| **Tooling** | MetaMask, Rabby, Hardhat, Foundry, Remix, ethers.js, web3.js work out of the box per docs; contracts deploy as standard Ethereum bytecode. |

---

## Precompiles / EVM Extensions (detailed)

Cosmos EVM exposes Cosmos SDK modules to the EVM as **precompiled contracts** at fixed addresses, implemented as native Go rather than EVM bytecode.

| Precompile | Address | Type | Function |
|---|---|---|---|
| **P256** | `0x…0100` | Stateless | secp256r1 (P-256) signature verification |
| **Bech32** | `0x…0400` | Stateless | Convert between hex and bech32 address formats |
| **Staking** | `0x…0800` | Stateful | Delegate / undelegate / redelegate, validator ops |
| **Distribution** | `0x…0801` | Stateful | Withdraw staking rewards, community pool |
| **ICS20** | `0x…0802` | Stateful | IBC (ICS-20) cross-chain token transfers |
| **Bank** | `0x…0804` | Stateful | Native Cosmos token balances / supply |
| **Governance** | `0x…0805` | Stateful | On-chain proposals and voting |
| **Slashing** | `0x…0806` | Stateful | Validator slashing / jailing / unjailing |
| **ICS02** | `0x…0807` | Stateful | IBC light-client (ICS-02) queries / client-router (v0.7.0) |
| **ERC20** | Dynamic (per token) | Stateful | Standard ERC-20 interface over native Cosmos tokens |
| **WERC20** | Dynamic (per token) | Stateful | Wrapped-native (WETH-style deposit/withdraw) |

(Note: `vesting` and `evidence` precompiles existed in older Evmos code but are **not** in the current canonical `cosmos/evm` set; `0x…0803` is unused. ERC20/WERC20 register one instance per token pair, so they have no single fixed address.)

### How stateful precompiles work
- Each stateful precompile holds a reference to the relevant module **keeper** (staking keeper, bank keeper, etc.).
- A Solidity call to the precompile address routes out of the EVM into native Go: it decodes the ABI input, builds the corresponding Cosmos SDK message/query, and invokes the keeper against the multistore — **within the same transaction**, so EVM and Cosmos state changes commit atomically.
- **Gas metering across both worlds:** `RequiredGas` computes a base cost from the **Cosmos SDK gas config** (flat cost + per-byte cost × input length, keyed off the method ID, distinguishing tx vs query), and the EVM `suppliedGas`/`remainingGas` accounting is reconciled with the SDK gas meter.

### Adding a custom precompile
Build on `cmn.Precompile` (`github.com/cosmos/evm/precompiles/common`) and implement the geth-style stateful interface:
- `Address() common.Address` — fixed registration address.
- `RequiredGas(input []byte) uint64` — cost from the SDK gas config.
- `Run(evm *vm.EVM, contract *vm.Contract, readOnly bool) ([]byte, error)` — decode ABI input, dispatch to a method, call the module keeper, ABI-encode the result.

Then register/activate it in the chain's active-precompiles map. This is a **chain-builder** operation (genesis/upgrade), not something a contract author does at runtime. Conceptually this is exactly the requested capability: deterministic, bounded (non-Turing-complete) native modules surfaced to EVM contracts behind a Solidity ABI.

---

## Consensus (detailed)

- **Engine:** **CometBFT** (Tendermint Core successor), BFT with up to 1/3 Byzantine/offline tolerance.
- **Finality:** **single-block instant/deterministic finality (~2s), no reorganizations** — docs: "Transactions are final after one block (~2 seconds) with no reorganizations." Contrasts with Ethereum L1's probabilistic finality.
- **PoS / DPoS (default):** Cosmos SDK `x/staking` — token holders bond to validators; top `MaxValidators` by bonded stake form the active set; non-validators delegate. At each block end, `x/staking` returns the updated validator set to CometBFT. This is effectively **DPoS by default**.
- **PoA (configurable):** Achieved by constraining the validator set to a fixed/permissioned group (gating validator membership, fixed/small set, governance constraints), optionally combined with the **permissioned EVM** (whitelist who may deploy/call). No dedicated "PoA mode" flag — it's a configuration of the same `x/staking` + CometBFT machinery.

---

## Maintenance, Governance & Production Use

### Release history (published releases, newest first)
| Tag | Published | Notes |
|---|---|---|
| **v0.7.0** | **2026-05-05** | Krakatoa app-side mempool; BlockSTM parallel execution; ICS-02 client-router precompile; OpenTelemetry tracing; JSON-RPC filter overhaul; 2026.1 deps |
| v0.6.0 | 2026-03-02 | Removed IBC Transfer wrapper (must use precompile); StateDB passed into internal EVM calls; **fix for ASA-2026-002** |
| v0.5.1 | 2025-11-28 | IBC middleware sender-verification, ERC20 event, Ledger coin-type-60 fixes |
| v0.5.0 | 2025-10-21 | **EIP-7702 support**; JSON-RPC aligned to go-ethereum 1.16.3; IBC OnRecvPacket supports 0x recipients; config overhaul |
| v0.4.2 / v0.3.2 | 2025-10-21 | patch line |
| v0.4.1 | 2025-08-14 | |
| v0.3.1 | 2025-08-05 | |
| v0.3.0 | 2025-07-25 | first GitHub release |

There are also `v1.0.0-rc0/rc1/rc2` git tags (May–Jun 2025) and `v0.7.0-beta.0`, but **v1.0.0 has not been released** — the stable line remains on 0.x, with v1.0 targeted ~Q2 2026 after a security audit and benchmarking.

### Maintenance signals
- Repo **not archived**; `main` last pushed **2026-06-23**; ~419 commits in the trailing 52 weeks; bursty cadence with clusters around releases.
- **Maintainer:** Cosmos Labs (Interchain Foundation subsidiary / Interchain Labs), with core contributions from **B-Harvest, Cronos, and MANTRA**.
- **License:** Apache-2.0.

### Production / adopting chains
Reported users include **MANTRA Chain** (first L1 purpose-built for RWAs running CosmWasm + a fully spec-compliant EVM side-by-side, via the Interchain Labs Cosmos EVM module), **Injective** (launched native EVM mainnet), **Sei** (transitioning to EVM-only by mid-2026 under SIP-3), plus **Ripple/XRPL EVM sidechain, Mezo, KiiChain, Ondo, Stable, and Telegram/TAC**. (Note: Injective and Sei use their own EVM variants/heavily-customized stacks; the canonical `cosmos/evm` module is the shared upstream the ecosystem is converging on.)

### Security / audit posture
- **Pre-v1.0**, undergoing audit ahead of a v1.0 release; breaking changes expected across 0.x.
- **ASA-2025-004 / GHSA-mjfq-3qr2-6g84 (Critical, May 2025) — "Partial Precompile State Writes."** A low EVM call-gas-limit could make a stateful precompile execute partway and error **without reverting state already written** (e.g. distribution precompile transferring rewards without zeroing claimable; also a chain-halt vector). Affected Cosmos EVM 0.1.0 / Evmos >13.0.0. Fixed by wrapping precompile execution atomically (`RunAtomic` / `RevertMultiStore`).
- **ASA-2026-002 / GHSA-54gx-3cgr-7mfm (Critical, Mar 2026).** Incorrect state handling during **nested EVM execution paths involving the ICS20 precompile** allowed tokens to be reused within a single transaction — caused a **$7M loss on the Saga network**. Affected versions with the ICS20 precompile **prior to v0.6.0**; **patched in v0.6.0**. Cosmos Labs coordinated with 15 affected chains before disclosure.
- Both critical advisories are **precompile-related**, reinforcing that the precompile system is both the key feature and the principal risk surface.

---

## Frank Pros / Cons vs. the 4 Criteria

**Pros**
- Genuinely high Ethereum compatibility at the **Prague/Pectra** level including EIP-7702; standard tooling works out of the box.
- Best-in-class realization of "expose native non-Turing-complete modules as EVM contracts" via stateful precompiles, with a clean keeper-based extension model.
- Native BFT with instant finality; PoS/DPoS by default and PoA achievable — full consensus flexibility from one stack.
- Actively maintained by the Interchain Foundation with real production adopters (MANTRA, Injective, XRPL EVM, etc.).

**Cons**
- Still **pre-v1.0** with explicit breaking-change warnings; v1.0 only targeted ~Q2 2026.
- **Two critical precompile CVEs in ~12 months**, one with a real $7M loss — the headline feature is also the historical risk concentration.
- Deviations from pure Ethereum: base fee to validators (not burned), `eth_gasPrice`=0, no EIP-4844 blobs, dual-address account model — generally benign but can surprise some tooling/assumptions.
- PoA requires manual configuration rather than a turnkey mode.
- Custom precompiles are a chain-level (genesis/upgrade) capability, not a runtime/contract-author one.

---

## Sources

- [github.com/cosmos/evm (repo + README)](https://github.com/cosmos/evm)
- [cosmos/evm README on main](https://github.com/cosmos/evm/blob/main/README.md)
- [cosmos/evm releases](https://github.com/cosmos/evm/releases)
- [cosmos/evm v0.7.0 release](https://github.com/cosmos/evm/releases/tag/v0.7.0)
- [cosmos/evm CHANGELOG](https://github.com/cosmos/evm/blob/main/CHANGELOG.md)
- [cosmos/evm go.mod (deps & geth fork)](https://raw.githubusercontent.com/cosmos/evm/main/go.mod)
- [cosmos/go-ethereum v1.17.2-cosmos-0](https://github.com/cosmos/go-ethereum/tree/v1.17.2-cosmos-0)
- [Cosmos EVM docs — Overview](https://docs.cosmos.network/evm/latest/)
- [Cosmos EVM docs — EVM compatibility](https://docs.cosmos.network/evm/latest/documentation/evm-compatibility)
- [Cosmos EVM docs — Ethereum JSON-RPC methods](https://docs.cosmos.network/evm/latest/api-reference/ethereum-json-rpc/methods)
- [Cosmos EVM docs — EIP-7702](https://docs.cosmos.network/evm/next/documentation/evm-compatibility/eip-7702)
- [Cosmos EVM docs — Precompiles overview](https://docs.cosmos.network/evm/latest/documentation/smart-contracts/precompiles/overview)
- [Cosmos EVM docs — Precompiles reference](https://docs.cosmos.network/evm/next/documentation/smart-contracts/precompiles)
- [Cosmos EVM docs — Concepts / consensus overview](https://docs.cosmos.network/evm/next/documentation/concepts/overview)
- [Cosmos EVM docs — Integrate (converting a Cosmos SDK chain)](https://evm.cosmos.network/integrate)
- [Cosmos EVM docs — Changelog / release notes](https://docs.cosmos.network/evm/latest/changelog/release-notes)
- [Cosmos SDK x/staking README](https://github.com/cosmos/cosmos-sdk/blob/main/x/staking/README.md)
- [pkg.go.dev — cosmos/evm precompiles/common](https://pkg.go.dev/github.com/cosmos/evm/precompiles/common)
- [pkg.go.dev — cosmos/evm precompiles/staking](https://pkg.go.dev/github.com/cosmos/evm/precompiles/staking)
- [Security advisory GHSA-mjfq-3qr2-6g84 (ASA-2025-004, partial precompile state writes)](https://github.com/advisories/GHSA-mjfq-3qr2-6g84)
- [Security advisory GHSA-54gx-3cgr-7mfm (ASA-2026-002, ICS20 nested-execution)](https://github.com/cosmos/evm/security/advisories/GHSA-54gx-3cgr-7mfm)
- [CertiK — EVM–Cosmos Convergence Research, Part 2](https://www.certik.com/blog/evm-cosmos-convergence-research-from-security-base-part-2)
- [CertiK — EVM–Cosmos Convergence Research, Part 3](https://www.certik.com/blog/evm-cosmos-convergence-research-from-security-base-part-3)
- [Cosmos Labs — ICF funds open-sourcing of Cosmos EVM](https://www.cosmoslabs.io/blog/interchain-foundation-funds-tharsis-labs-to-publish-evmos-open-source-strengthening-ethereum-cosmos-support-and-interoperability)
- [The Cosmos Stack Roadmap for 2026](https://cosmos.network/blog/the-cosmos-stack-roadmap-2026)
- [Blockworks — Cosmos gets a canonical EVM](https://blockworks.com/news/cosmos-canonical-evm)
- [The Block — Injective rolls out native EVM support](https://www.theblock.co/post/378418/injective-rolls-out-native-evm-support-on-its-high-performance-cosmos-based-chain)
- [MANTRA Chain — EVM Module docs](https://docs.mantrachain.io/developers/architecture/evm-module)
