# PROPOSAL-0001: Fork Tempo into a Sovereign Stablecoin EVM L1

- **Status:** Draft for review
- **Date:** 2026-06-25
- **Author:** Platform / Protocol Engineering
- **Implements:** [ADR-0001](ADR-0001-evm-chain-implementation.md) (Reth + Commonware, Tempo-bootstrapped)
- **Validated by:** [SPIKE-0001](SPIKE-0001-reth-commonware-tempo.md) (run first; this proposal assumes a GO)
- **Grounding research:** [tempo-as-blueprint.md](docs/architecture/tempo-as-blueprint.md) · [commonware-reth-glue.md](docs/architecture/commonware-reth-glue.md) · [stablecoin-gas-mechanisms.md](docs/architecture/stablecoin-gas-mechanisms.md) · [precompiles-turing-incomplete-modules.md](docs/architecture/precompiles-turing-incomplete-modules.md)

---

## 1. Proposal in one paragraph

Fork **`github.com/tempoxyz/tempo`** (Apache-2.0/MIT) and harden it into our own sovereign stablecoin L1. Because our product is *also* a stablecoin chain with gas paid in stablecoins, Tempo is an unusually close starting point: we **keep** the consensus↔execution glue, the fee model, and most precompiles; we **strip** Stripe-/product-specific coupling; we **build** our own validator-selection/consensus policy, genesis, governance, domain precompiles, and the decentralization + audit work that Tempo has not yet done. This is closer to *"adopt, re-own, harden, decentralize"* than *"rewrite."* The dominant cost is **ownership and assurance** (auditing, tracking two ALPHA-ish upstreams, growing the validator set), not greenfield engineering.

## 2. Goals & non-goals

**Goals**
- A sovereign EVM L1 that is 1:1 with the latest Ethereum fork for tooling (R1), exposes custom Turing-incomplete modules as precompiles (R2), runs our chosen consensus over our validator set (R3), is actively maintained by us (R4), and charges gas in any stablecoin with no required native token (R5).
- A maintainable fork we can track against upstream Reth and Commonware.

**Non-goals (initially)**
- Re-implementing consensus or the EVM (we use Commonware Simplex and Reth/revm as-is).
- Tempo's payments-product features we don't need (Stripe integrations, hosted fee-payer/indexer services, private "zones" subchains unless wanted).
- Day-one large permissionless validator set — we start permissioned/small and decentralize on a roadmap (mirrors Tempo, but we own the roadmap).

## 3. Source & licensing

- **Upstream:** `github.com/tempoxyz/tempo` — Rust workspace, ~79% Rust / ~15% Solidity, Apache-2.0/MIT. Reference commit from research: `9f2041c` (2026-06-24); **pin a specific commit** at fork time.
- **Transitive upstreams:** `paradigmxyz/reth` (pinned rev), `commonwarexyz/*` (`consensus::simplex`, `consensus::threshold_simplex`, `cryptography::bls12381`, `p2p`, `storage`, `runtime`). Both Apache-2.0/MIT.
- **License obligation:** preserve Apache-2.0/MIT notices; our additions can be our own license-compatible terms. (Note: this is permissive — unlike Arbitrum/BeaconKit BUSL — so a fork is unencumbered.)

## 4. Architecture we inherit (and keep)

From [tempo-as-blueprint.md](docs/architecture/tempo-as-blueprint.md), verified against the repo:

- **Reth used as a library** via the SDK node builder: `TempoNode` implements Reth's `Node`/`NodeTypes` with custom `ExecutorBuilder`, `ConsensusBuilder`, `PayloadBuilder`, `PoolBuilder`, `PayloadValidatorBuilder`.
- **Consensus = Commonware Simplex** (leader-based, notarize→finalize, deterministic no-reorg finality) plus **BLS12-381 + per-epoch DKG** (the `dkg`/`epoch` actors), driving execution **in-process via Engine-API data types** (`ForkchoiceState`, `PayloadAttributes`, `ConsensusEngineHandle`), with the external Engine-API RPC disabled (`NoopEngineApiBuilder`).
- **Two-actor split:** an `application` actor (propose/verify payloads) and an `executor` actor (apply finalized forkchoice). We keep this separation.
- **Protocol features as native-Rust precompiles** injected via a custom `PrecompilesMap` in the EVM config.
- **Custom EIP-2718 tx type `0x76`** carrying account-abstraction + fee-token fields.

We do **not** change this architecture. Our work sits in well-bounded places within it.

## 5. Work breakdown — keep / adapt / strip / build, by crate

Tempo's `crates/` and our disposition for each:

| Crate | Role | Disposition | Notes |
|---|---|---|---|
| `consensus` | The glue: Simplex engine + actors (application, executor, marshal, epoch, dkg, feed, follow) | **KEEP + ADAPT** | Keep wholesale; adapt the validator-set source (see §6.1) and rebrand. Highest-value asset. |
| `node` | `TempoNode` Reth-SDK node def + `engine.rs` | **KEEP + RENAME** | Re-point to our chainspec; keep `NoopEngineApiBuilder` in-process pattern. |
| `evm` | `TempoEvmConfig`, custom revm `Evm`, block assembly/validation | **KEEP + ADAPT** | Keep; adjust precompile set + block rules; decide native-token behavior (§6.2). |
| `revm` | revm extensions/context | **KEEP** | Plumbing; track upstream revm. |
| `payload*` | `PayloadBuilder`/`Attributes`/`BuiltPayload` | **KEEP** | The build path consensus calls. |
| `precompiles` (+ `-macros`) | Fee Manager, TIP-20 Factory, Stablecoin DEX, Signature Verifier, Account Keychain, Policy Registry… | **KEEP useful + STRIP unneeded + BUILD ours** | This is most of our R2/R5 value. See §6.2, §6.3. |
| `primitives`, `alloy`, `chainspec` | `TempoTxType (0x76)`, envelope, signatures, chain spec | **KEEP + ADAPT** | Our chain ID/params; decide whether to keep `0x76` (§6.4). |
| `transaction-pool` | `TempoPoolBuilder`, AA tx validation | **KEEP** | Mempool support for `0x76`. |
| `contracts` | Precompile **addresses** + Solidity interfaces | **KEEP + REASSIGN** | Re-document our address map (§6.3). |
| `faucet`, `tempo-sidecar`, `zones` | Faucet, sidecar services, private subchains | **STRIP (optional)** | Product/ops; keep only if we want them. |

Companion org repos (`tempo-go`, `pytempo`, `wallet-rs`, `accounts`, `tempo-std`, `mpp-specs`): **adopt selectively** for SDKs/tooling.

## 6. Detailed work areas

### 6.1 Consensus & validator model (R3) — *the main "build"*
Commonware Simplex and the glue are kept. Tempo runs a **small permissioned PoA set with per-epoch DKG**. Our work:
- **Validator-selection module.** Replace/extend the source that feeds Commonware's `Supervisor`/`ThresholdSupervisor` (Tempo uses an on-chain `OnchainDkgOutcome` + a validator-config precompile). Implement our consensus family:
  - **PoA:** static or governance-managed set (Tempo's model, smallest effort).
  - **PoS/DPoS:** a staking module/precompile that ranks validators by stake and feeds the active set + weights into the epoch DKG. *(This is net-new; M–L effort.)*
- **Epoch & DKG/resharing.** Keep the `epoch`/`dkg` actors; validate resharing on validator-set changes; this is non-trivial to operate — budget testing.
- **Liveness tuning.** Expose and tune the engine knobs (`time_to_propose`, `time_to_collect_notarizations`, `proposal_return_budget`, `fcu_heartbeat_interval`, `views_until_leader_skip`).
- **Verify version-sensitive API names** (`Supervisor`/`ThresholdSupervisor`, `threshold_simplex` module path, `Automaton::genesis`) against the pinned Commonware rev on day one.

### 6.2 Token & fee model (R5) — *keep, decide two things*
Keep the `0x76` `fee_token` envelope + FeeManager precompile (`0xfeec…`) + fixed-rate Fee AMM. Two decisions:
- **Eligibility model:** **Tempo (TIP-20 + structural USD eligibility + fixed-rate MEV-free AMM)** = lowest fork friction; or **Celo `FeeCurrencyDirectory` + per-token oracle + decimal adapters** = supports arbitrary existing ERC-20 USDC/USDT but adds oracle + governance surface. (See [stablecoin-gas-mechanisms.md](docs/architecture/stablecoin-gas-mechanisms.md).)
- **Native gas token: none or optional.** Tempo has none (`BALANCE`/`CALLVALUE` return 0). Keep "none" for the seamless stablecoin-only UX, *or* re-introduce an optional native token if we want it for staking/MEV/incentives — which interacts with the PoS choice in §6.1.

### 6.3 Precompiles (R2) — *keep, strip, extend*
- **Keep:** Fee Manager, Signature Verifier (secp256k1/P256/WebAuthn — needed for AA), TIP-20 Factory + Stablecoin DEX (if we keep the TIP-20 model), Account Keychain.
- **Strip:** any Stripe-/product-specific precompiles we don't need.
- **Build (our domain modules):** e.g. for an RWA/compliance chain — a **compliance/KYC allowlist** precompile, **oracle/price read**, **validator-config read**, custom hashing/ZK-verify if relevant. Follow the gas/determinism rules and the precompile-vs-system-contract guidance in [precompiles-turing-incomplete-modules.md](docs/architecture/precompiles-turing-incomplete-modules.md): prefer read-only stateful precompiles; put governance-tunable policy in upgradeable system contracts; **hardfork-gate every precompile change** behind a Commonware-coordinated upgrade.
- **Re-document the address map** in `crates/contracts` so our precompiles don't collide with Ethereum's reserved range or future EIPs.

### 6.4 Custom tx type `0x76` (R1 trade-off)
Tempo's `0x76` carries fee-token + AA fields, but it is **non-standard** — generic wallets emit type-2 txs and won't produce `0x76` without a custom signer/SDK. Decide:
- **Keep `0x76`** for full AA + stablecoin-fee UX, and ship signer support (adopt `tempo-go`/`wallet-rs`/`pytempo`); or
- **Also accept standard type-2 txs** (with a default fee-token path) so unmodified MetaMask works out of the box, reserving `0x76` for advanced flows.
This is the main R1 tooling-compatibility decision; recommend supporting **both** so standard tooling works while AA users get `0x76`.

### 6.5 Genesis, chainspec, RPC, tooling (R1)
- New chain ID, genesis, and chainspec in `crates/chainspec`/`primitives`.
- Keep Reth's standard `eth_*` JSON-RPC; run the **compatibility matrix** (MetaMask, Hardhat, Foundry, ethers, viem, a Blockscout-style indexer) — carried over from SPIKE-0001 §WS-C.
- Confirm the target hardfork (Tempo targets **Osaka/Fusaka-era**); decide our fork target and `evmVersion` guidance for contract authors.

### 6.6 Governance, upgrades, ops
- **Upgrade mechanism:** precompile/consensus changes are hardfork-gated (coordinated node upgrade); policy/parameter changes go through upgradeable system contracts. Define the on-chain governance surface.
- **Ops:** validator runbooks, key management, monitoring, the genesis ceremony (Tempo's are not published — we build ours).
- **Decentralization roadmap:** start permissioned (4–N nodes), expand the set, exercise DKG resharing.

### 6.7 Strip / rebrand
- Remove Stripe-specific integrations, branding, `faucet`/`sidecar`/`zones` if unwanted, and dormant/feature-flagged paths (subblocks, some BAL/TIP-1016 work) — *or* keep them disabled but documented. **Distinguish live from experimental code** (the repo carries hardfork-gated dormant paths).

## 7. Phases, milestones & effort

Effort in engineer-weeks (EW), assuming a **3–4 person Rust team** (≥1 with Reth/revm depth, ≥1 with consensus/crypto depth). Ranges reflect ALPHA-upstream and audit uncertainty.

| Phase | Scope | Exit criteria | Effort |
|---|---|---|---|
| **P0 — Validate** | [SPIKE-0001](SPIKE-0001-reth-commonware-tempo.md): fork builds, ≥4-node devnet, R5 proof, finality measured | GO decision | 3 wks (timeboxed) |
| **P1 — Foundation** | Fork, pin commits, rebrand, reproduce devnet in our CI; chain ID/genesis/chainspec; strip obvious product code | Branded devnet producing blocks under our chainspec | 4–6 EW |
| **P2 — Consensus & validators (R3)** | Validator-selection module for our consensus family (PoA first; PoS/DPoS if chosen); epoch/DKG validated; liveness tuned | Validator-set changes + DKG resharing work on devnet | 8–14 EW |
| **P3 — Token & fee model (R5)** | Decide TIP-20 vs Celo + native-token; adapt fee module; end-to-end stablecoin gas hardened | Gas paid in ≥2 stablecoins, no native balance, audited swap math | 6–10 EW |
| **P4 — Precompiles (R2)** | Strip unneeded; build our domain precompiles; finalize address map; system-contract split | Our modules callable from Solidity; determinism tests pass | 6–12 EW |
| **P5 — Tooling & RPC (R1)** | `0x76` + type-2 support decision; signer/SDK; full compatibility matrix; indexer | MetaMask/Hardhat/Foundry/ethers/viem/indexer all green | 4–8 EW |
| **P6 — Governance, ops, public testnet** | Upgrade/governance surface; runbooks; monitoring; multi-org testnet | Public testnet with external validators | 8–12 EW |
| **P7 — Audit & mainnet readiness** | Independent audit (glue + fee module + every precompile + consensus integration); fix; grow validator set; mainnet | Audit closed; mainnet genesis | 10–16 EW + external audit (calendar + $$) |

**Indicative total to mainnet:** ~**46–78 engineer-weeks** of build (≈ **3–6 months** wall-clock for a 3–4 person team running phases with overlap), **plus** external audit calendar time and an ongoing maintenance commitment. P0–P5 (a credible internal testnet) is the bulk of the value and lands first.

## 8. Dependency & upstream-tracking strategy

- **Pin** Reth and `commonware-*` to Tempo's revisions initially; never upgrade mid-phase.
- Maintain a **thin fork**: keep our changes in clearly separated crates/modules so rebasing onto newer Reth/Tempo is mechanical. Track upstream Tempo too — they will keep improving the glue.
- **CI that builds against the pinned toolchain** and a scheduled job that attempts upstream bumps in a branch, surfacing breakage early.
- Treat **Commonware ALPHA churn** as a standing cost; design to Simplex (Minimmit is spec-only); consider contributing fixes upstream.

## 9. Testing & assurance

- **Deterministic runtime:** use Commonware's deterministic test runtime + `Pacer` pattern (as Tempo does) for reproducible consensus e2e tests.
- **Devnet matrix:** ≥4 validators, restart/liveness, partition, DKG-resharing, validator add/remove.
- **EVM conformance:** Ethereum execution-spec tests against our fork; the tooling compatibility matrix.
- **Fee-module fuzzing:** swap math, `0x76` decoding, fee accounting; de-peg/liquidity edge cases.
- **Audit (P7):** independent audit of the forked glue, fee module, every custom precompile, and the consensus↔execution integration **before mainnet** — Tempo itself is unaudited, so we cannot rely on its assurance.

## 10. Risks & mitigations (fork-specific)

| Risk | Severity | Mitigation |
|---|---|---|
| Copying unaudited Tempo bugs | High | Full independent audit (P7); don't treat upstream as vetted; track Tempo security fixes. |
| Commonware ALPHA API churn | High | Pin; thin fork; design to Simplex; upstream contributions; scheduled bump branch. |
| Validator-selection/DKG is net-new and subtle | Med–High | Start PoA (Tempo's proven path); stage PoS/DPoS; heavy DKG-resharing testing. |
| `0x76` breaks generic wallets | Medium | Support standard type-2 txs too (§6.4); ship signer SDKs. |
| Distinguishing live vs dormant code | Medium | Pin a commit; audit feature flags/hardfork gates; document what we enable. |
| Reth SDK surface evolves | Medium | Thin fork; CI bump branch; lean on Reth SDK docs + Tempo/Alto as references. |
| Perpetual ownership burden | Med–High | Staff a dedicated platform team; budget recurring merge/maintenance. |
| Performance below vendor claims at our validator count/hardware | Medium | Benchmark our own set; tune liveness knobs; don't assume ~0.6s finality without measuring. |

## 11. Decision points for stakeholders

1. **Consensus family (§6.1):** PoA only, or PoS/DPoS? (drives P2 effort and the native-token decision)
2. **Fee eligibility (§6.2):** Tempo TIP-20 model vs. Celo arbitrary-ERC-20 model?
3. **Native gas token (§6.2):** none (seamless stablecoin-only) vs. optional (for staking/incentives)?
4. **Tx types (§6.4):** keep `0x76` only, or also accept standard type-2 for out-of-the-box tooling?
5. **Domain precompiles (§6.3):** which native modules do we need (e.g. compliance/KYC/oracle for RWA)?
6. **Scope of strip (§6.7):** keep or drop `zones`/`faucet`/`sidecar` and the dormant throughput features?

## 12. Recommendation & next step

Proceed in this order: **run SPIKE-0001 (P0) → resolve the §11 decision points → execute P1–P7.** The fork is high-leverage because our product overlaps Tempo's (stablecoin gas, AA, no native token), so we keep most of the hardest engineering (the glue, fee model, precompile framework) and concentrate net-new work on the validator/consensus model, our domain precompiles, decentralization, and assurance. The defining commitment is **ownership**: we adopt two fast-moving upstreams and an unaudited reference, so the audit and maintenance plan (§8–§9) is not optional — it is the core of the proposal.

---
**Last verified:** 2026-06-25. Repo facts from [tempo-as-blueprint.md](docs/architecture/tempo-as-blueprint.md) (inspected at commit `9f2041c`).
