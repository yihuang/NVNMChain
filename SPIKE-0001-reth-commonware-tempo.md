# SPIKE-0001: Validate the Reth + Commonware (Tempo-bootstrapped) stack

- **Status:** Proposed
- **Date:** 2026-06-25
- **Timebox:** 3 weeks (15 working days), hard stop
- **Owner:** Protocol / Platform Engineering lead
- **Team:** 2–3 Rust engineers (≥1 with EVM/Reth or revm exposure), part-time devops
- **Decision it informs:** [ADR-0001](ADR-0001-evm-chain-implementation.md) §9 — flip from **Proposed** to **Accepted** (or fall back to Cosmos EVM)
- **Background reports:** [tempo-as-blueprint.md](docs/architecture/tempo-as-blueprint.md) · [commonware-reth-glue.md](docs/architecture/commonware-reth-glue.md) · [stablecoin-gas-mechanisms.md](docs/architecture/stablecoin-gas-mechanisms.md)

---

## 1. Why this spike exists

ADR-0001 recommends building our sovereign EVM L1 on **Reth (execution) + Commonware Simplex (consensus)**, bootstrapped from Stripe's open-source **Tempo** node (`github.com/tempoxyz/tempo`, Apache-2.0/MIT). The recommendation hinges on three claims that are currently *evidence-light* and must be proven before we commit:

1. **R5 is real and forkable** — we can run a devnet where users pay gas **in a stablecoin with no native gas token**, using Tempo's `0x76` tx + FeeManager precompile + Fee AMM.
2. **The ownership cost is bounded** — forking Tempo and standing up a multi-node devnet is a *measured* effort (days, not quarters), so the "you own the stack forever" risk is acceptable.
3. **The maturity risk is survivable** — Commonware (ALPHA) and the unaudited Tempo reference behave well enough on a small validator set to justify a build decision.

**Central question:** *Can a 2–3 person team stand up a ≥4-validator EVM devnet on the forked Tempo stack, pay gas end-to-end in a stablecoin with no native token, add one custom precompile, and swap the validator-selection logic — within 3 weeks?* A clean "yes" → Accept the ADR. A "no / only with major effort" → fall back to Cosmos EVM + paymasters.

## 2. Explicitly out of scope

To protect the timebox, this spike will **not**: implement final tokenomics or staking economics; design the production validator set or DKG ceremony; build bridging/DA; do a security audit (we *identify* the audit surface, we don't audit); choose the final fee-eligibility governance model (we prototype one and document the other); optimize performance; or write production-grade code. **Throwaway code is expected and acceptable.**

## 3. Exit criteria (the questions we MUST answer)

Each has a binary or measured answer captured in the spike report (§7).

| # | Question | Evidence required |
|---|---|---|
| E1 | Does the forked Tempo node build and run a **≥4-validator** devnet producing blocks? | Running devnet; block height advancing; validators independent processes/containers. |
| E2 | **R5 proof:** can an account with **zero native balance** send a tx and pay gas in a stablecoin via the `0x76` tx type? | Tx hash, receipt, before/after stablecoin balances, FeeManager state change. |
| E3 | Does **standard Ethereum tooling** work unmodified against the devnet RPC? | MetaMask connect + send; Foundry `cast`/`forge` deploy+call; ethers/viem script; a block explorer or indexer syncing. |
| E4 | What is the **observed finality latency** of Commonware Simplex on the devnet? | Measured ms from tx submit → final, over ≥100 txs, p50/p95. |
| E5 | Can we **add one custom precompile** (a trivial Turing-incomplete module) and call it from Solidity? | Deployed contract calling the precompile; deterministic result; appears in execution. |
| E6 | Can we **replace the validator-selection source** (the `Supervisor` input) with our own logic (e.g. a static PoA set, then a stake-ranked set)? | Validator set driven by our module; a membership change takes effect. |
| E7 | What is the **realistic integration cost & maintenance burden**? | Effort log (person-days per workstream); list of Tempo couplings we had to cut; upstream-version pinning notes. |
| E8 | What is the **audit/risk surface** we'd be taking on? | Enumerated: forked glue, fee module, custom precompiles, Commonware ALPHA APIs touched. |

## 4. Workstreams

Run **WS-A** and **WS-B** in parallel from Day 1; **WS-C** and **WS-D** start once a single node runs.

### WS-A — Bootstrap & devnet (owner: eng 1) → answers E1, E7
1. Fork `tempoxyz/tempo`; record the pinned Reth rev and `commonware-*` versions it uses. Reproduce the documented build.
2. Run a **single** node from genesis; get blocks producing.
3. Scale to a **≥4-validator** devnet (containers or local processes); confirm independent validators reach consensus and the chain makes progress under a node restart.
4. Capture: build time, dependency surprises, how genesis/validator config is expressed, the run book.

### WS-B — R5 / stablecoin gas (owner: eng 2) → answers E2, E7
1. Map Tempo's fee path in source: the `0x76` tx type, FeeManager precompile (`0xfeec…`), Fee AMM, and the TIP-20/USD eligibility check (cross-check against [stablecoin-gas-mechanisms.md](docs/architecture/stablecoin-gas-mechanisms.md)).
2. Stand up a test stablecoin token recognised by the fee module (mint to a test account).
3. **The R5 proof:** from an account with **0 native balance**, submit a `0x76` tx paying fees in the stablecoin; verify it executes, the stablecoin balance decreases by the fee, and the validator/FeeManager receives the proceeds.
4. Document what governs token eligibility and what a **Tempo-model vs Celo-model** choice would change (don't implement Celo; just scope it).

### WS-C — Tooling compatibility (owner: eng 1/3, after first node) → answers E3
Run the **compatibility matrix** against the devnet RPC: MetaMask (add network, send), Foundry (`cast send`, `forge create` + call), an ethers v6 **and** viem script, and point one indexer/explorer (e.g. Blockscout) at it. Note the `0x76`-tx UX wrinkle: standard wallets emit type-2 txs — record whether gas-in-stablecoin requires a custom signer/SDK and what "normal ETH-style" txs do on this chain.

### WS-D — Extensibility probes (owner: eng 2/3) → answers E5, E6
1. **Custom precompile:** add a trivial deterministic precompile (e.g. a fixed-point helper) via the `ConfigureEvm`/`EvmFactory`/`PrecompilesMap` path Tempo uses; call it from a deployed Solidity contract.
2. **Validator-selection swap:** replace the input feeding Commonware's `Supervisor` with our own minimal module — first a **static PoA** set, then a **stake-ranked** set read from a contract/precompile — and show a membership change taking effect. (Verify the exact `Supervisor`/`ThresholdSupervisor` trait names against current source — flagged version-sensitive in the glue report.)

### WS-E — Synthesis (owner: lead, Days 13–15) → answers E4, E7, E8
Measure finality latency (E4), consolidate the effort log and coupling/maintenance notes (E7), enumerate the audit surface (E8), write the spike report, and run the go/no-go.

## 5. Timeline (15 working days)

| Days | WS-A | WS-B | WS-C | WS-D | Milestone |
|---|---|---|---|---|---|
| 1–2 | Fork, build, single node | Map fee path in source | — | — | **M1:** one node producing blocks |
| 3–5 | 4-validator devnet | Stablecoin minted & recognised | Begin tooling matrix | — | **M2:** 4-node devnet + RPC reachable |
| 6–8 | Harden run book; restart/liveness test | **R5 end-to-end proof (E2)** | Finish tooling matrix (E3) | Custom precompile (E5) | **M3:** R5 proven; tooling green |
| 9–11 | Measure finality (E4) | Eligibility model write-up | Indexer sync | Validator-selection swap (E6) | **M4:** extensibility proven |
| 12–13 | Effort log freeze | — | — | — | Buffer / overflow |
| 14–15 | — | — | — | — | **M5:** report + **go/no-go** |

Days 12–13 are deliberate buffer for the inevitable ALPHA-dependency / build surprises.

## 6. Environment & prerequisites

- 4–6 small cloud VMs or a single host running containers; a private network between validators.
- Rust toolchain matching Tempo's pinned version; Foundry; Node (ethers/viem); a Blockscout (or similar) instance.
- Read access to `tempoxyz/tempo`, `paradigmxyz/reth`, `commonware-xyz/monorepo`, and the Commonware `alto` example (the glue is "modelled after alto").
- A scratch git repo for the fork + throwaway probe code.

## 7. Deliverable: SPIKE-0001 report

A short report (and a 30-min demo) structured as:
1. **Verdict:** Go / No-Go / Conditional, one paragraph.
2. **Exit-criteria table (E1–E8)** with the concrete evidence/measurements.
3. **Effort log:** person-days per workstream → the real integration-cost number for ADR §5.
4. **Coupling cuts:** what Tempo payments code we removed and how cleanly.
5. **Audit surface (E8):** the components a production build must get audited.
6. **Recommendation:** confirm ADR-0001 primary, or invoke the Cosmos-EVM fallback, with reasons.

## 8. Go / No-Go decision gate

Apply after the report. **GO (Accept ADR-0001 primary)** requires **all** of:
- E1 (4-node devnet), **E2 (R5 proven end-to-end)**, and E3 (standard tooling works) are **YES**; and
- E5 + E6 are **YES** (extensibility for our R2 + consensus needs is demonstrated); and
- E7 effort is within tolerance — **rough guide: ≤ ~25 person-days to first working devnet**, extrapolating to a credible team plan; and
- E4 finality is acceptable for the product (target sub-second p50, define the ceiling beforehand).

**CONDITIONAL GO:** R5 + tooling pass but extensibility or effort is heavier than hoped → Accept with a larger staffing plan and a phased roadmap.

**NO-GO (fall back to Cosmos EVM + ERC-4337/EIP-7702 paymasters):** if **E2 cannot be demonstrated**, the stack won't build/stabilise on a 4-node set within the timebox, or the effort/maturity (Commonware ALPHA churn, unaudited reference) reads as unmanageable for our team. Per ADR §7, the fallback trades the seamless "no native token" UX for a turnkey, upstream-maintained stack.

## 9. Risks to the spike itself

| Risk | Mitigation |
|---|---|
| ALPHA Commonware / Reth-pin build breakage eats days | Pin to exactly Tempo's revisions; don't upgrade during the spike; Days 12–13 buffer. |
| Tempo too payments-coupled to run "generic" quickly | First milestone is "run Tempo *as-is*"; only then strip — don't refactor before it runs. |
| `0x76` tx signing has no off-the-shelf wallet | Use Tempo's own client/SDK or a scripted signer for the R5 proof; treat wallet UX as a separate later question. |
| Version-sensitive API names (`Supervisor`, `threshold_simplex` path, `Automaton::genesis`) drift from the glue report | Verify against current source on Day 1; the glue report flagged these as needing confirmation. |
| Scope creep into production concerns | Enforce §2 out-of-scope; throwaway code only; lead guards the timebox. |

---
This plan operationalises [ADR-0001](ADR-0001-evm-chain-implementation.md) §9 step 1.
