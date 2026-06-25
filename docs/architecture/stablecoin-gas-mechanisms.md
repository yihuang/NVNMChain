# Paying Gas in Any Stablecoin on an EVM L1: Mechanisms and Stack Comparison

**Last verified: 2026-06-25**

Research input for the ADR on a sovereign EVM L1 with a HARD requirement: *gas payable in any stablecoin*, ideally with no (or optional) native gas token and seamless UX.

---

## Executive Summary

There are two fundamentally different ways to let users pay gas in stablecoins, and they are not equivalent:

1. **Protocol-native (enshrined) fee tokens** — the chain's *execution client and consensus layer* are modified so that a transaction can denominate and pay its gas in an ERC-20/stablecoin directly. The node debits the token, settles validators, and there is no app-layer relayer. **Tempo** (no native token at all) and **Celo** (CELO + allowlisted ERC-20s) are the two production examples. This delivers the seamless, "any stablecoin," no-native-token UX the requirement asks for — but only if you control the L1 client.

2. **App-layer account abstraction (ERC-4337 paymasters, + EIP-7702)** — works on *any* EVM chain with no protocol changes. A paymaster contract fronts the native gas; the user repays in stablecoin (e.g. signing an EIP-2612 permit). Requires off-chain bundler/paymaster infrastructure and a relayer trust relationship, and the chain still has a native gas token under the hood. Good as a portability fallback, not a true "no native token" design.

For a **sovereign L1 with a hard requirement**, the enshrined approach is the correct architecture, and **Tempo's model (Reth SDK + a Fee Manager precompile + a fixed-rate Fee AMM, no native token) is the closest off-the-shelf precedent that is open-source (Apache/MIT) and built on a forkable stack.** Celo's L2 fee-currency code (CIP-64 + FeeCurrencyDirectory) is the most battle-tested precedent but is OP-Stack-coupled. Cosmos EVM does **not** support charging EVM gas directly in arbitrary ERC-20s — it charges a native denom and relies on Cosmos-layer (IBC/Osmosis) fee abstraction, which is less seamless for an EVM-first product.

---

## 1. Tempo (Stripe / Paradigm) — Enshrined Stablecoin Gas (primary focus)

Tempo is a payments-first EVM L1 incubated by Stripe and Paradigm, built on the **Reth SDK** (Paradigm's Rust execution client) with **Commonware "Simplex" consensus** (sub-second finality). All code is open source under **Apache/MIT** licenses. As of mid-2026 it is in testnet; the validator set launches **permissioned/small** (core team + partners) with a roadmap to permissionless PoS. ([Paradigm](https://www.paradigm.xyz/2025/09/tempo-payments-first-blockchain), [Tempo GitHub](https://github.com/tempoxyz/tempo), [Performance docs](https://docs.tempo.xyz/learn/tempo/performance))

### No native gas token
Tempo has **no native token**. Per the fee spec: *"Transaction fees are paid directly in USD-denominated stablecoins."* A direct consequence stated in the docs: because there is no native value, **`BALANCE` and `CALLVALUE` are unaffected by the fee mechanics** — value transfer between accounts happens via TIP-20 token transfers, not native value. ([Fee spec](https://docs.tempo.xyz/protocol/fees/spec-fee))

### Fee denomination and the fixed base fee
- Gas-price fields (`max_priority_fee_per_gas`, `max_fee_per_gas`) are denominated in **attodollars (10⁻¹⁸ USD) per gas**, i.e. abstract USD units, not a token.
- Tempo uses a **fixed base fee, NOT the dynamic EIP-1559 base-fee curve** (current mainnet figure cited: `2×10¹⁰` attodollars/gas). Priority fees still exist for ordering. ([Fee spec](https://docs.tempo.xyz/protocol/fees/spec-fee))
- This is closer to a Solana-style fixed gas price than to Ethereum's congestion-driven base fee. (Corroborated by independent analysis: [Jeon, Medium, Jan 2026](https://medium.com/@organmo/tempo-architecture-analysis-2-stablecoin-gas-and-the-payment-only-lane-134f2150b9ae).)

### The custom EIP-2718 transaction type `0x76`
The transaction struct (from the spec) is:

```
TempoTransaction {
  chain_id, max_priority_fee_per_gas, max_fee_per_gas, gas_limit,
  calls: Vec<Call>,                 // batch; must be non-empty
  access_list,
  nonce_key: U256, nonce: u64,      // 2D / parallelizable nonces
  fee_token: Option<Address>,       // <-- chosen fee stablecoin
  fee_payer_signature: Option<Signature>,  // <-- sponsorship
  valid_before, valid_after,        // time bounds
  key_authorization, aa_authorization_list  // passkey / AA auth
}
```

- The **`fee_token` field carries the chosen fee stablecoin as an optional address.** When `Some(addr)` it overrides account-level and validator-level preferences. Validation requires it to be a **valid USD-denominated TIP-20 token with sufficient balance/liquidity**.
- **Fee sponsorship** is native via `fee_payer_signature`; senders sign with magic byte `0x76`, fee payers with `0x78` (domain separation). The sponsor commits to a specific sender + fee token.
- RLP layout is `0x76 || rlp([...])` with absent optionals encoded as `0x80`. ([Transaction spec](https://docs.tempo.xyz/protocol/transactions/spec-tempo-transaction), [Transactions overview](https://docs.tempo.xyz/protocol/transactions))

### The Fee Manager precompile
A precompiled contract, **FeeManager at `0xfeec000000000000000000000000000000000000`**, handles fees within each transaction. It:
- collects and refunds fees per transaction;
- executes fee **swaps** immediately (via the Fee AMM) when the user's token ≠ the validator's token;
- stores **per-user** (`setUserToken`) and **per-validator** (`setValidatorToken`) fee-token preferences;
- accumulates fees for validators to claim via **`distributeFees()`**.

Fallback order if no preference is set at tx / account / contract level: the protocol defaults the user's fee token to **pathUSD**. `setValidatorToken` requires a USD TIP-20. ([Fee search results / Fee spec](https://docs.tempo.xyz/protocol/fees/spec-fee))

### The Stablecoin DEX / "Fee AMM" — how pricing/conversion works
This is the load-bearing detail and the docs are explicit: the Fee AMM uses **fixed/pegged exchange rates, NOT a constant-product market AMM and NOT an oracle.** (An early third-party summary describing it as a 0.3%-LP constant-product DEX is inaccurate; the authoritative spec is below.)

- Each AMM converts `userToken → validatorToken` at a **fixed `0.9970` rate** (`m = 9970 / 10000`); `amountOut = amountIn * 9970 / 10000`. The `0.0030` spread accrues to liquidity providers.
- Rebalancing the other direction (`validatorToken → userToken`) runs at a fixed **`0.9985`** (`n = 9985`).
- If a direct pair lacks liquidity, it does a **two-hop route via `userToken.quoteToken()`**; the validator then receives `floor(floor(spend·9970/10000)·9970/10000)`.
- Fixed rates are a deliberate **MEV-elimination** choice — there is no market price to sandwich. LPs supply validator-token liquidity and earn the spread. ([Fee AMM spec](https://docs.tempo.xyz/protocol/fees/spec-fee-amm), [Managing fee liquidity](https://docs.tempo.xyz/guide/stablecoin-dex/managing-fee-liquidity))

### What validators actually receive
The fee spec: *"All fees collected in blocks the validator proposes will be automatically converted to the chosen token (if needed) and transferred to the validator's account."* Net of the spread, a validator receives **0.9970 of its preferred stablecoin per 1.0 user token**, claimable via `distributeFees()`. So validators are paid in a *settled* stablecoin of their choosing, not whatever the user happened to hold. ([Fee spec](https://docs.tempo.xyz/protocol/fees/spec-fee))

### Which stablecoins are eligible, and who governs the list
- **Eligibility rule:** any **TIP-20 token whose `currency == "USD"`** can be used to pay gas. TIP-20 is a precompile-backed token standard extending ERC-20 (memos, reward distribution, transfer policies). ([TIP-20 overview](https://docs.tempo.xyz/protocol/tip20/overview), [TIP-20 spec](https://docs.tempo.xyz/protocol/tip20/spec))
- **Creation is permissionless at the protocol level:** the `TIP20Factory` precompile (`0x20Fc000000000000000000000000000000000000`) lets *any* caller deploy a TIP-20; the `currency` is fixed at creation and immutable. There is **no on-chain whitelist or governance body** approving fee-eligible tokens in the spec — eligibility is the structural `currency=="USD"` rule plus the existence of Fee AMM liquidity. Per-token admin (mint/burn, e.g. for pathUSD) is held by the token's own `admin` (recommended to be a multisig on mainnet). So "governance" of the gas-token list is effectively *currency-tag + liquidity*, not a curated allowlist. ([TIP-20 spec](https://docs.tempo.xyz/protocol/tip20/spec), [Native stablecoins](https://docs.tempo.xyz/learn/tempo/native-stablecoins))
- **pathUSD** is the first/native stablecoin and the universal fallback gas + quote token.

### Reusable/forkable, or payments-coupled?
- **In favor of reuse:** open-source Apache/MIT; built on the **Reth SDK**, which is explicitly a modular toolkit for building custom chains; the fee logic lives in well-isolated precompiles (FeeManager, FeeAMM, TIP20Factory) with documented addresses and interfaces (`IFeeAMM`, `IFeeManager`, `IStablecoinDEX`).
- **Against clean reuse:** the model is **tightly coupled to TIP-20** (gas tokens *must* be TIP-20 precompile tokens, not arbitrary externally-deployed ERC-20s), to **pathUSD** as fallback, to a **fixed base fee**, and to the **`0x76` tx type + Tempo's signing domains**. Adopting "Tempo's fee module" effectively means adopting Tempo's token standard and tx envelope. It is forkable *as a whole stack*, less so as a drop-in module for a different EVM client.

> Candor: Tempo is pre-mainnet (testnet as of mid-2026); the spec warns features may change. The exact governance/operator model for who runs validators and curates pathUSD at scale is not fully public.

---

## 2. Celo — Fee Currencies (key precedent)

Celo has supported paying gas in allowlisted ERC-20s (cUSD, USDC, USDT, etc.) since its L1 days, and **this survived its March 2025 migration to an Ethereum L2** (OP-Stack-based; L1 froze at block 31,056,500 on 2025-03-26). It is implemented in a **modified op-geth/node**, not in app-layer paymasters. ([L2 migration spec](https://specs.celo.org/l2_migration.html), [ERC-20 tx fees](https://docs.celo.org/what-is-celo/about-celo-l1/protocol/transaction/erc20-transaction-fees))

How it works now ([Fee abstraction spec](https://specs.celo.org/fee_abstraction.html), [Fee abstraction docs](https://docs.celo.org/developer/fee-abstraction)):
- A transaction sets a **`feeCurrency` field (CIP-64 tx type)** naming the ERC-20 that covers `maxFeePerGas`/`maxPriorityFeePerGas`.
- **`FeeCurrencyDirectory`** (replaced the old `FeeCurrencyWhitelist`) is the **governance-curated registry** of eligible tokens. Each entry stores: token address, an **oracle address** (`IOracle`) giving the CELO↔token exchange rate, and a **token-specific intrinsic-gas** value (extra gas for the debit/credit token calls).
- Tokens must implement **`IFeeCurrency`** (debit/credit hooks the node calls). Tokens with <18 decimals (USDC/USDT @ 6) register via a **`FeeCurrencyAdapter`** that normalizes to 18-decimal precision.
- RPC support: `eth_estimateGas`, `eth_gasPrice`, `eth_maxPriorityFeePerGas` accept an optional `feeCurrency`.
- **Block-space governance:** per-currency caps via `celo.feecurrency.limits` (e.g. cUSD ≤ 90%) and a `celo.feecurrency.default` for unlisted tokens; CELO is uncapped.

Differences vs Tempo: Celo **still has a native token (CELO)** and uses **oracles** to price the ERC-20 against it (vs Tempo's no-native-token, abstract-USD, fixed-rate model). Celo's allowlist is **explicitly governance-curated**; Tempo's is structural (`currency==USD` + liquidity). Celo's eligibility extends to *arbitrary* ERC-20s once allowlisted; Tempo requires its TIP-20 standard.

> Candor: the L2 spec confirms fee-currency survived migration but does not spell out exactly how the sequencer monetizes ERC-20 fees post-migration; this is handled inside the node's debit/credit path.

---

## 3. Cosmos EVM — Fee Abstraction Options

A Cosmos SDK + EVM chain (the `cosmos/evm` framework, formerly Evmos) does **not** let the EVM charge gas directly in arbitrary ERC-20 stablecoins. ([Cosmos EVM gas & fees](https://docs.cosmos.network/evm/), [x/erc20 module](https://docs.cosmos.network/evm/latest/documentation/cosmos-sdk/modules/erc20))

- **`x/feemarket`** implements EIP-1559 dynamic fees, but fees are paid in the **chain's native fee denom**; base fee is distributed to validators/delegators (not burned).
- **`x/erc20`** only bridges representation — it auto-wraps Cosmos bank coins/IBC tokens as ERC-20s and converts bidirectionally. It **does not enable ERC-20 gas payment**; gas is still the native denom. (Confirmed: "ERC-20 stablecoins cannot be used directly for transaction fees.")
- **Fee abstraction (Osmosis-style `x/feeabstraction` / FAM):** lets users pay in *IBC* tokens by **swapping them for the native fee denom on Osmosis** (price via an IBC `async-icq` channel). This is a **Cosmos-layer** mechanism — it works on Cosmos-SDK transactions, swaps to the native token under the hood, and is not an EVM-native "gas in stablecoin" path. ([Fee Abstraction Module overview](https://medium.com/@notional-ventures/the-fee-abstraction-module-653838089515))

**Candor on seamlessness:** For an EVM-first product this is the weakest fit. Gas is conceptually always the native staking token; "stablecoin gas" is an *abstraction/swap* layer (IBC + DEX) rather than the enshrined "denominate the tx in the stablecoin" model of Tempo/Celo. You cannot ship "no native token, pay any stablecoin directly" with stock Cosmos EVM without significant custom work in the AnteHandler / fee deduction path.

---

## 4. Generic EVM — ERC-4337 Paymasters & EIP-7702

Works on **any** EVM chain with **no protocol changes** — Avalanche Subnet-EVM, Reth-based chains, OP-Stack L2s, etc. ([ERC-4337 EIP](https://eips.ethereum.org/EIPS/eip-4337), [ERC-4337 docs](https://docs.erc4337.io/core-standards/erc-4337.html))

How stablecoin gas works: a **Paymaster** contract agrees to pay the native gas to the `EntryPoint`; the user repays in stablecoin. **Circle Paymaster** is the canonical USDC example — the user signs an **EIP-2612 permit** granting the paymaster a small USDC allowance, the app builds a `UserOperation`, a **bundler** submits it. Live on Arbitrum, Avalanche, Base, Ethereum, OP Mainnet, Polygon PoS, Unichain; supports **ERC-4337 SCAs (EntryPoint v0.7/v0.8)** *and* **EOAs via EIP-7702**. Pimlico and Biconomy run comparable stablecoin paymasters. ([Circle Paymaster](https://www.circle.com/paymaster), [Gelato](https://gelato.cloud/paymaster-bundler))

**EIP-7702's role:** lets a plain EOA *temporarily* delegate to ERC-4337-compatible smart-account code, so existing EOAs (not just new smart wallets) can route through the same bundler+paymaster infra and pay gas in stablecoin.

**Downsides for a "hard requirement":**
- **Not truly native:** the chain still has a native gas token; stablecoin gas is a UX veneer that depends on a willing paymaster having native-token reserves.
- **Infra burden:** you must run or rent **bundlers + paymaster service + EntryPoint**; the alt-mempool is real but **bundler operation is concentrated** among a few operators.
- **Relayer/paymaster trust + liquidity:** paymaster must be funded and online; it sets the spread; it's a censorship and liveness chokepoint.
- **Smart-account risk:** SCAs add contract attack surface; a buggy validation function is a buggy wallet.
- **UX friction & coverage:** "any stablecoin" is only what the paymaster chooses to accept; per-token permit flows; higher per-tx gas overhead.

---

## 5. Comparison & Recommendation

### Comparison table

| Dimension | **Tempo** (enshrined, no native token) | **Celo** (enshrined, native CELO + ERC-20) | **Cosmos EVM** (native denom + abstraction) | **ERC-4337 paymasters + 7702** (app layer) |
|---|---|---|---|---|
| Where it lives | Execution client (Reth SDK) precompiles + `0x76` tx | Modified op-geth node, CIP-64 tx | AnteHandler / x/feemarket + IBC swap | Smart contracts + off-chain bundler/paymaster |
| Native gas token? | **None** | Yes (CELO) | Yes (staking denom) | Yes (chain's native) |
| UX seamlessness | **Highest** — tx denominates in stablecoin natively | High — `feeCurrency` field, native | Low–medium for EVM users (swap under hood) | Medium — permit + UserOp, depends on paymaster |
| "Any stablecoin" coverage | Any **USD TIP-20** (+ Fee AMM liquidity); permissionless creation | Any ERC-20 **once governance-allowlisted** | Any IBC token w/ Osmosis route (not EVM ERC-20 gas) | Only what the paymaster accepts |
| Pricing/conversion | **Fixed-rate Fee AMM** (0.9970/0.9985), no oracle, MEV-free | **Oracle** (token↔CELO) per directory entry | DEX swap price via Osmosis (IBC ICQ) | Paymaster-set rate/spread |
| What validators get | Their chosen settled stablecoin (net 0.9970) | Value via node debit/credit (CELO-equivalent) | Native fee denom | Native token (paymaster fronts it) |
| Infra burden on you | Run the L1 (Reth+Commonware); maintain Fee AMM liquidity | Run OP-Stack L2 w/ Celo node mods | Run Cosmos chain; integrate Osmosis FAM | Run/rent bundler+paymaster; fund reserves |
| Validator economics | Settled in preferred stablecoin; LPs earn spread | Standard, oracle-priced | Standard staking-token fees | Unchanged (native) |
| Security surface | Precompiles + fixed rates (small, MEV-resistant) + permissioned validators (initially) | Oracle dependency; node debit/credit hooks | IBC + cross-chain oracle (ICQ) trust | Paymaster solvency/liveness, bundler centralization, SCA bugs |
| Maturity | **Testnet** (mid-2026), Stripe/Paradigm-backed | **Production** (L2 since Mar 2025), years of L1 history | Production framework; abstraction is Cosmos-layer | Production, widely deployed |
| Forkability for a sovereign L1 | Apache/MIT; **forkable whole-stack** but TIP-20-coupled | OP-Stack-coupled; reusable if you adopt OP-Stack | Cosmos SDK; reusable but not EVM-native gas | N/A (drop-in on any chain) |

### Recommendation

For a sovereign EVM L1 where **gas-in-any-stablecoin is a hard, no-native-token, seamless-UX requirement**, build on an **enshrined fee-token model in the execution client** — do **not** rely on ERC-4337 paymasters as the primary mechanism (keep them as an optional portability fallback for external tooling).

Concretely, ranked:

1. **Best fit / closest precedent: the Tempo model (Reth SDK + Fee Manager precompile + fixed-rate Fee AMM, no native token).** It is the only open-source, production-track stack that *natively* delivers "no native gas token, pay in any (USD) stablecoin, validators settled in their chosen stablecoin, MEV-free fixed-rate conversion." Because it's Apache/MIT on the modular **Reth SDK**, you can fork it or re-implement the precompile pattern. Cost: you inherit/adopt the **TIP-20 token standard, the `0x76` tx type, and a fixed base fee**, and you must seed/maintain **Fee AMM liquidity** per validator token. If "any stablecoin" can mean "any USD-tagged token on our standard," this is the cleanest path.

2. **Strongest battle-tested reference design: Celo's CIP-64 + `FeeCurrencyDirectory` + adapters + oracles.** If you prefer **arbitrary ERC-20** fee currencies (not a bespoke token standard) and are willing to keep a (possibly nominal) native token and run an **OP-Stack** L2/L1, port Celo's node modifications. It is the most mature implementation of protocol-level ERC-20 gas. Trade-off: oracle dependency and OP-Stack coupling; not "zero native token."

3. **Avoid as the primary mechanism: stock Cosmos EVM.** It cannot charge EVM gas directly in arbitrary ERC-20s; stablecoin gas is a Cosmos-layer IBC/Osmosis swap abstraction. Choosing Cosmos for this requirement means writing significant custom fee-deduction logic anyway, losing the EVM-native ergonomics.

4. **ERC-4337 paymasters (+ EIP-7702): keep as a complementary fallback, not the spec.** Useful for letting existing EOAs/wallets pay in stablecoin without bespoke client support, and for chains you don't control. But it keeps a native gas token, adds bundler/paymaster infra and a relayer trust/liveness chokepoint, and limits "any stablecoin" to what a paymaster accepts — so it does not satisfy the *no-native-token, seamless* part of the hard requirement on its own.

**Bottom line for the ADR:** target an **enshrined execution-client fee module on a Reth-based stack, following Tempo's Fee-Manager-precompile + fixed-rate Fee-AMM pattern (no native token)**, with **Celo's FeeCurrencyDirectory/adapter/oracle design as the reference** if you need arbitrary-ERC-20 (rather than a custom token standard) coverage. Offer ERC-4337/7702 paymasters as an optional compatibility layer.

---

## Sources

**Tempo**
- [Tempo: The Blockchain Designed for Payments — Paradigm](https://www.paradigm.xyz/2025/09/tempo-payments-first-blockchain)
- [tempoxyz/tempo — GitHub](https://github.com/tempoxyz/tempo)
- [Tempo Fee Specification](https://docs.tempo.xyz/protocol/fees/spec-fee)
- [Tempo Fee AMM Specification](https://docs.tempo.xyz/protocol/fees/spec-fee-amm)
- [Managing Fee Liquidity](https://docs.tempo.xyz/guide/stablecoin-dex/managing-fee-liquidity)
- [Tempo Transaction (0x76) Spec](https://docs.tempo.xyz/protocol/transactions/spec-tempo-transaction)
- [Transactions overview](https://docs.tempo.xyz/protocol/transactions)
- [TIP-20 Overview](https://docs.tempo.xyz/protocol/tip20/overview) · [TIP-20 Spec](https://docs.tempo.xyz/protocol/tip20/spec) · [Native stablecoins](https://docs.tempo.xyz/learn/tempo/native-stablecoins)
- [Performance / architecture](https://docs.tempo.xyz/learn/tempo/performance)
- [Architecture Analysis (2): Stablecoin Gas — Seungmin Jeon, Medium (Jan 2026)](https://medium.com/@organmo/tempo-architecture-analysis-2-stablecoin-gas-and-the-payment-only-lane-134f2150b9ae)

**Celo**
- [Fee Abstraction — Celo Specification](https://specs.celo.org/fee_abstraction.html)
- [L2 Migration — Celo Specification](https://specs.celo.org/l2_migration.html)
- [Paying for Gas with Tokens — Celo Docs](https://docs.celo.org/what-is-celo/about-celo-l1/protocol/transaction/erc20-transaction-fees)
- [Implementing Fee Abstraction in Wallets — Celo Docs](https://docs.celo.org/developer/fee-abstraction)

**Cosmos EVM**
- [Cosmos EVM docs (gas & fees)](https://docs.cosmos.network/evm/)
- [cosmos/evm — GitHub](https://github.com/cosmos/evm)
- [x/erc20 module](https://docs.cosmos.network/evm/latest/documentation/cosmos-sdk/modules/erc20)
- [The Fee Abstraction Module — Notional Ventures](https://medium.com/@notional-ventures/the-fee-abstraction-module-653838089515)

**ERC-4337 / EIP-7702**
- [ERC-4337 EIP](https://eips.ethereum.org/EIPS/eip-4337) · [ERC-4337 docs](https://docs.erc4337.io/core-standards/erc-4337.html)
- [Circle Paymaster (pay gas in USDC)](https://www.circle.com/paymaster)
- [Gelato Paymaster & Bundler](https://gelato.cloud/paymaster-bundler)
