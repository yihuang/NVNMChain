# Tempo Multi-Node Production Deployment

## Architecture

```
┌────────────────────────┐       ┌────────────────────────┐
│  Validator A           │       │  Validator B           │
│                        │       │                        │
│  ┌──────────────────┐  │       │  ┌──────────────────┐  │
│  │ Consensus Engine │──┼───────┼──│ Consensus Engine │  │
│  │ (commonware)     │  │       │  │ (commonware)     │  │
│  └────────┬─────────┘  │       │  └────────┬─────────┘  │
│           │            │       │           │            │
│  ┌────────▼─────────┐  │       │  ┌────────▼─────────┐  │
│  │ Execution Layer  │  │       │  │ Execution Layer  │  │
│  │ (reth SDK)       │  │       │  │ (reth SDK)       │  │
│  └──────────────────┘  │       │  └──────────────────┘  │
└────────────────────────┘       └────────────────────────┘
         │                              │
         └─────────── P2P ──────────────┘
```

Each node runs two layers in one process:

- **Execution layer** — Reth SDK-based EVM (Oracle-compatible EVM at Osaka hardfork with Tempo extensions). Handles JSON-RPC, transaction pool, block building, state.
- **Consensus layer** — Commonware Simplex BFT with BLS threshold signatures. DKG-based validator set rotation, sub-second finality.

## Prerequisites

| Component | Requirement |
|---|---|
| OS | Linux (x86_64 or arm64), macOS for dev |
| CPU | 8+ cores (16+ recommended for validators) |
| RAM | 32 GiB (64 GiB+ recommended for archive nodes) |
| Storage | NVMe SSD, 1 TiB+ (see [storage estimation](#storage-estimation)) |
| Network | 100+ Mbps, <50ms RTT between validators |
| Rust | stable (MSRV tracked in `rust-toolchain.toml`) |
| Tools | `just`, `nu`, `cargo` |

## Binary

```bash
# Build with production profile
cargo build --profile maxperf -p tempo

# Optional features
#   --features js-tracer    JavaScript-based debug tracing
#   --features jemalloc     jemalloc allocator (recommended for production)
#   --features asm-keccak   assembly-optimized keccak (significant perf gain)
#   --features otlp         OpenTelemetry logs/metrics export

# Binary location
./target/maxperf/tempo
```

## Configuration

### 1. Generate Genesis

One-time process run by the genesis coordinator. Creates `genesis.json` and per-validator key material.

```bash
cargo run --profile maxperf -p tempo-xtask -- generate-genesis \
  --output /etc/tempo/genesis \
  -a 50000 \
  --chain-id 42432 \
  --validators 10.0.0.1:8000,10.0.0.2:8000 \
  --t0-time <TS0> --t1-time <TS1> --t2-time <TS2> --t3-time <TS3> \
  --t4-time <TS4> --t5-time <TS5> --t6-time <TS6> --t7-time <TS7> --t8-time <TS8> \
  --gas-limit 500000000 \
  --epoch-length 302400 \
  --seed <SEED>
```

Output structure:

```
/etc/tempo/genesis/
├── genesis.json              # chain genesis (share with every node)
├── 10.0.0.1:8000/
│   ├── signing.key           # ed25519 p2p identity key (SECRET)
│   └── signing.share         # bls12-381 threshold signing share (SECRET)
├── 10.0.0.2:8000/
│   ├── signing.key
│   └── signing.share
└── ...
```

`genesis.json` embeds the initial DKG outcome in `extra_data`. Each `--validators` entry gets a unique ed25519 keypair and BLS share from the initial DKG ceremony. **Treat `signing.key` and `signing.share` as secrets equivalent to a validator private key.**

### 2. Validator Key Security

Two key types:

| Key | Format | Purpose | Security |
|---|---|---|---|
| Ed25519 signing key (`signing.key`) | raw 32-byte hex, or age-encrypted | P2P identity + consensus messages | **Option A:** raw hex on disk with restrictive permissions. **Option B:** age-encrypted with passphrase via `--consensus.secret`. Prefer a FIFO (`mkfifo`) or process substitution (`<(echo $PASSPHRASE)`) to avoid passphrase on disk |
| BLS signing share (`signing.share`) | CBOR-encoded share | Block notarization/finalization | Must be distributed out-of-band per validator; never shared between validators |

Encrypted signing key workflow:

```bash
# One-time encryption
age -p -o signing.key.age signing.key

# Node startup (passphrase via FIFO)
mkfifo /run/tempo/consensus-secret.fifo
echo "$PASSPHRASE" > /run/tempo/consensus-secret.fifo &  # writer
tempo node --consensus.signing-key signing.key.age \
           --consensus.secret /run/tempo/consensus-secret.fifo
```

### 3. Node CLI Reference

#### Required Arguments

```
tempo node \
  --chain /etc/tempo/genesis/genesis.json \
  --datadir /var/lib/tempo \
  --consensus.signing-key /etc/tempo/validator/signing.key \
  --consensus.signing-share /etc/tempo/validator/signing.share \
  --consensus.listen-address 0.0.0.0:8000
```

#### Consensus Arguments

| Argument | Default | Description |
|---|---|---|
| `--consensus.signing-key <PATH>` | — | Ed25519 P2P identity key (required unless `--dev` or `--follow`) |
| `--consensus.secret <PATH>` | — | Passphrase file/FIFO for age-encrypted signing key |
| `--consensus.signing-share <PATH>` | — | BLS threshold signing share (required unless `--dev` or `--follow`) |
| `--consensus.listen-address <ADDR>` | `127.0.0.1:8000` | P2P listen socket |
| `--consensus.metrics-address <ADDR>` | `127.0.0.1:8001` | Consensus metrics HTTP endpoint |
| `--consensus.target-block-time <DUR>` | `550ms` | Target block interval (healthy network) |
| `--consensus.wait-for-proposal <DUR>` | `1200ms` | Max wait for leader proposal before view timeout |
| `--consensus.wait-for-notarizations <DUR>` | `2s` | Max wait for quorum notarizations |
| `--consensus.network-budget <DUR>` | `50ms` | Time reserved for proposal propagation |
| `--consensus.worker-threads <N>` | `3` | Consensus worker threads |
| `--consensus.message-backlog <N>` | `16384` | Channel buffer depth |
| `--consensus.mailbox-size <N>` | `16384` | Actor mailbox capacity |
| `--consensus.views-to-track <N>` | `256` | Activity timeout in views |
| `--consensus.inactive-views-until-leader-skip <N>` | `32` | Views inactive before skip |
| `--consensus.synchrony-bound <DUR>` | `5s` | Maximum clock skew tolerance |
| `--consensus.max-message-size-bytes <N>` | MAX_RLP_BLOCK | Max P2P message size |
| `--consensus.backfill-frequency <N>` | `8` | Blocks/sec rate limit for backfill |
| `--consensus.time-to-build-subblock <DUR>` | `100ms` | Subblock construction timeout |
| `--consensus.subblock-broadcast-interval <DUR>` | `50ms` | Subblock rebroadcast interval |
| `--consensus.fcu-heartbeat-interval <DUR>` | `5m` | Forkchoice state heartbeat |
| `--consensus.finalized-blocks-retention <N>` | `4096` | In-memory finalized blocks count |
| `--consensus.datadir <PATH>` | `<datadir>/<chain>/consensus` | Consensus data directory |
| `--consensus.network-identity <KEY>` | — | BLS network identity key (post-DKG rotation) |
| `--consensus.network-identity-from-epoch <N>` | — | Epoch for network identity activation |

#### Network Security Arguments

| Argument | Default | Description |
|---|---|---|
| `--consensus.bypass-ip-check` | `false` | Disable IP-based connection filtering. **Only on trusted networks** |
| `--consensus.allow-private-ips` | `false` | Allow RFC1918 connections. Required for private/cloud VPC |
| `--consensus.allow-dns` | `true` | Allow DNS-based ingress addresses |
| `--consensus.max-concurrent-handshakes <N>` | `512` | Max concurrent TLS handshakes |
| `--consensus.connection-per-peer-min-period <DUR>` | `60s` | Min interval between reconnects to the same peer |
| `--consensus.handshake-per-ip-min-period <DUR>` | `5s` | Rate-limit handshakes per source IP |
| `--consensus.handshake-per-subnet-min-period <DUR>` | `15ms` | Rate-limit handshakes per /24 |
| `--consensus.time-to-unblock-byzantine-peer <DUR>` | `4h` | Ban duration for misbehaving peers |
| `--consensus.wait-before-peers-redial <DUR>` | `1s` | Retry delay for peer dials |
| `--consensus.wait-before-peers-reping <DUR>` | `50s` | Liveness ping interval |
| `--consensus.wait-before-peers-discovery <DUR>` | `60s` | Peer discovery query interval |

#### Execution Layer Arguments

Reth-standard arguments apply. Key overrides:

| Argument | Default | Description |
|---|---|---|
| `--http.api` | `safe` | RPC namespaces. Production: `eth,net,web3,txpool,debug,trace,tempo,operator` |
| `--ws.api` | `safe` | WebSocket namespaces |
| `--auth-ip-blacklist` | `*` | Engine API blacklist; override to allow CL connections |
| `--builder.gaslimit` | N/A | Block gas limit (must match genesis) |
| `--builder.deadline` | `12` | Block building deadline in seconds |
| `--txpool.aa-valid-after-max-secs` | `3600` | Max `valid_after` offset for AA transactions |
| `--txpool.max-tempo-authorizations` | `16` | Max authorizations per AA transaction |

#### Faucet (Optional — dev/test only)

```bash
--faucet.enabled \
--faucet.private-key <HEX> \
--faucet.amount <U256> \
--faucet.address <TOKEN_ADDRESS>
```

Exposes `tempo_fundAddress` RPC. **Not for production.**

### 4. Config File

#### Execution Layer: reth TOML Config

Reth-based flags (database, RPC, P2P, mempool, tracing) support a TOML config file via `--config`:

```toml
# /etc/tempo/config.toml
[peers]
trusted_only = true

[rpc]
eth_apis = ["eth", "net", "web3", "txpool", "debug", "trace", "tempo", "operator"]

[http]
addr = "0.0.0.0"
port = 8545

[ws]
addr = "0.0.0.0"
port = 8546

[metrics]
addr = "0.0.0.0"
port = 9001

[builder]
deadline = 3

[stages]
[stages.types]
full = { pruning = true }
```

Usage:

```bash
tempo node --config /etc/tempo/config.toml \
  --chain /etc/tempo/genesis.json \
  --consensus.signing-key /etc/tempo/signing.key \
  --consensus.signing-share /etc/tempo/signing.share \
  --consensus.listen-address 0.0.0.0:8000
```

Most reth flags also accept environment variables prefixed with `RETH_` or `TEMPO_` (check `--help` per flag). Known Tempo-specific env vars:

| Env var | Equivalent flag |
|---|---|
| `TEMPO_FOLLOW` | `--follow` |
| `TEMPO_BOOTNODES_ENDPOINT` | `--bootnodes-endpoint` |

#### Consensus Layer: CLI Flags Only

Tempo-specific `--consensus.*` flags have **no config file** and **no environment variable** equivalents beyond those listed above. They must be passed as CLI arguments.

Workaround — store flags in a shell wrapper:

```bash
# /usr/local/bin/tempo-validator.sh
#!/bin/bash
exec /usr/local/bin/tempo node \
  --config /etc/tempo/config.toml \
  --chain /etc/tempo/genesis.json \
  --consensus.signing-key /etc/tempo/signing.key \
  --consensus.signing-share /etc/tempo/signing.share \
  --consensus.listen-address 0.0.0.0:8000 \
  --consensus.target-block-time 550ms \
  --consensus.wait-for-proposal 1200ms \
  --consensus.allow-private-ips \
  "$@"
```

## Deployment Procedure

### Bootstrap Phase

1. **Generate genesis** on a secure offline machine. Distribute `genesis.json` and per-validator key material to each operator out-of-band.

2. **Configure bootnodes** (optional). Gen 0 validators discover each other through the validator set embedded in genesis. For dynamic discovery:

```bash
# Host a bootnode endpoint returning JSON:
# {"42432": ["enode://<NODE_ID>@10.0.0.1:30303", ...]}

tempo node --bootnodes-endpoint https://bootnodes.example.com/nodes.json
```

### First Validator Start

```bash
# Systemd unit example: /etc/systemd/system/tempo.service
[Unit]
Description=Tempo Validator Node
After=network.target

[Service]
Type=simple
User=tempo
ExecStart=/usr/local/bin/tempo node \
  --chain /etc/tempo/genesis.json \
  --datadir /var/lib/tempo \
  --http --http.addr 0.0.0.0 --http.port 8545 \
  --http.api eth,net,web3,txpool,debug,trace,tempo,operator \
  --ws --ws.addr 0.0.0.0 --ws.port 8546 \
  --consensus.signing-key /etc/tempo/signing.key \
  --consensus.signing-share /etc/tempo/signing.share \
  --consensus.listen-address 0.0.0.0:8000 \
  --consensus.metrics-address 0.0.0.0:8001 \
  --consensus.target-block-time 550ms \
  --consensus.wait-for-proposal 1200ms \
  --consensus.allow-private-ips \
  --metrics 0.0.0.0:9001 \
  --log.file.directory /var/log/tempo

Restart=on-failure
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
```

### Subsequent Validators

Start remaining validators with their unique `signing.key` and `signing.share`. Nodes discover each other via the genesis-embedded peer list and begin the consensus view-change protocol. The first block is proposed within seconds of the first supermajority being online.

### Followers (Read-Only Nodes)

Sync from an upstream validator without participating in consensus:

```bash
tempo node --follow wss://validator-a.example.com:8546 \
  --chain /etc/tempo/genesis.json \
  --datadir /var/lib/tempo \
  --consensus.signing-key /etc/tempo/signing.key \
  --http --http.api eth,net,web3,debug,trace
```

`--consensus.signing-key` is still needed for P2P identity (network requirements), but the node does not sign notarizations.

## Validator Lifecycle

Validator set is managed on-chain via `ValidatorConfigV2` precompile (`0xCCCCCCCC...0001`). Only the contract owner can add/remove validators.

### Add a Validator

```bash
# Owner calls addValidator
cast send 0xCCCCCCCC00000000000000000000000000000001 \
  "addValidator(address,bytes32,string,string,address,bytes)" \
  <ADDR> <PUBKEY> <INGRESS> <EGRESS> <FEE_RECIPIENT> <SIG> \
  --rpc-url http://localhost:8545 --private-key <OWNER_KEY>
```

The new node is included as a dealer in the next DKG ceremony (which starts at the next epoch boundary). After DKG completes, the new node holds a BLS share and participates as a player in the following epoch.

### Remove a Validator

```bash
# Owner or the validator itself calls deactivateValidator
cast send 0xCCCCCCCC00000000000000000000000000000001 \
  "deactivateValidator(uint64)" <INDEX> \
  --rpc-url http://localhost:8545 --private-key <KEY>
```

Deactivation is effective at the next epoch boundary after the DKG reshare.

## Consensus Network Configuration

### Timing Budgets

The consensus layer uses a pipelined view-change protocol where each view has a leader, a proposal window, and a notarization window. These parameters must be tuned together:

```
┌──────────────────────────────────────────────────► time
│
├── Proposal Window ──┤├── Notarization Window ──┤├── Next View
│  wait-for-proposal   │  wait-for-notarizations   │
│                      │                           │
│◄── network-budget ──►│                           │
│◄──── target-block-time ─────────────────────────►│
```

| Scenario | `target-block-time` | `wait-for-proposal` | `wait-for-notarizations` | `network-budget` |
|---|---|---|---|---|
| Same-region (≤5ms RTT) | `500ms` | `1200ms` | `1000ms` | `50ms` |
| Cross-region (50ms RTT) | `2s` | `4s` | `3s` | `200ms` |
| Geo-distributed (100ms RTT) | `4s` | `8s` | `6s` | `500ms` |

**Rules of thumb:**

- `wait-for-proposal` should be ≥ `target-block-time × 2` to account for network jitter on the leader's proposal broadcast.
- `wait-for-notarizations` should be ≥ `target-block-time × 1.5` to give followers time to propagate notarizations.
- `network-budget` should be `2 × P95 RTT` between any two validators — this is the time reserved for proposal propagation before the local build deadline.
- `target-block-time` minimum is constrained by: `time-to-build-subblock (100ms) + network-budget + 2 × P95 RTT`. Running below this causes view timeouts in healthy conditions.
- `synchrony-bound` must be ≥ `target-block-time` and at least `2s` to tolerate NTP skew. Tighten only with hardware PTP clocks.

### Subblock Construction

Subblocks are partial proposals that the next leader collects before assembling the final block:

| Argument | Default | Recommendation |
|---|---|---|
| `--consensus.time-to-build-subblock` | `100ms` | Keep at `100ms` for most deployments. Reduce to `50ms` only at sub-`500ms` block times |
| `--consensus.subblock-broadcast-interval` | `50ms` | Keep at `50ms`. Each subblock is rebroadcast at this interval to the next proposer |

### Epoch Configuration

The chain's epoch length is set in genesis (`--epoch-length`). Epoch boundaries trigger DKG reshare:

| Epoch length | At 0.5s blocks | At 2s blocks | Use case |
|---|---|---|---|
| `302_400` | ~7 days | ~7 days | Default. Balances DKG overhead against validator set reactivity |
| `86_400` | ~12 hours | ~2 days | Frequent node churn; smaller sets converge faster |
| `1_209_600` | ~7 days | ~28 days | Stable validator set; minimize DKG ceremony frequency |

The DKG ceremony runs in the background over 3–5 views (~1–25 seconds depending on block time). It does not halt block production during reshare.

### Worker and Channel Tuning

| Argument | Default | Recommendation |
|---|---|---|
| `--consensus.worker-threads` | `3` | `4`–`8` for high-throughput. Each handles: network I/O, crypto, storage. Monitor CPU; if `tempo_consensus_view` lags behind wall clock, increase |
| `--consensus.message-backlog` | `16384` | Sufficient for all tested loads. Increase to `32768` for 10k+ validator sets |
| `--consensus.mailbox-size` | `16384` | Match to `message-backlog` |
| `--consensus.deque-size` | `10` | Increase to `20`–`50` for high-latency networks with frequent backfill |
| `--consensus.views-to-track` | `256` | At `550ms` block time this is ~140s of activity window. Increase for slower networks |
| `--consensus.inactive-views-until-leader-skip` | `32` | At `550ms` this is ~17s before a lagging validator is skipped as leader. Increase for geo-deployed networks |

### Network Security Hardening

Tempo's consensus P2P layer uses commonware with TLS-authenticated connections. Production defaults are aggressive:

| Argument | Default | Effect |
|---|---|---|
| `--consensus.bypass-ip-check` | `false` | **Must remain `false` in production.** Rejects handshakes from IPs not matching the validator's on-chain ingress/egress |
| `--consensus.allow-dns` | `true` | Allows DNS hostnames in ingress/egress fields — useful behind load balancers |
| `--consensus.allow-private-ips` | `false` | Enable for VPC-only deployments where validators communicate over RFC1918 addresses |
| `--consensus.handshake-per-ip-min-period` | `5s` | Limits handshake attempts per source IP. Tighten to `30s` if under volumetric attack |
| `--consensus.max-concurrent-handshakes` | `512` | Caps TLS handshake concurrency. Reduce to `128` under resource constraints |
| `--consensus.time-to-unblock-byzantine-peer` | `4h` | Ban duration for misbehaving or unresponsive peers |
| `--consensus.connection-per-peer-min-period` | `60s` | Minimum interval between reconnect attempts to the same peer |
| `--consensus.handshake-per-subnet-min-period` | `15ms` | Per-/24 rate limit. Leave at `15ms` to avoid collateral blocking in shared infrastructure |

### Validator IP Obfuscation

Tempo does not have a built-in sentry proxy. To hide a validator's real IP:

1. **Set ingress/egress to a load balancer IP or DNS name** when calling `addValidator` or `setIpAddresses` on the `ValidatorConfigV2` precompile.
2. **Place the validator in a private subnet** — it outbound-connects to peers using their on-chain addresses. The LB forwards incoming consensus connections.
3. **Restrict the validator's firewall** to only accept connections from the LB's source IP range.
4. **Verify** with `--consensus.bypass-ip-check false` — if the on-chain ingress doesn't match the LB's source IP, commonware rejects the handshake.

```
Internet ──► Load Balancer ──► Validator (P2P port 8000)
              (ingress in          │
               ValidatorConfigV2)  │ outbound peers dialed
                                   ▼ directly from on-chain
                               Peer A, Peer B, ...
```

### Leader Selection and View-Timeouts

Consensus uses a VRF-based leader election over the validator BLS public key set. Every view has a deterministic leader per round. If the leader fails to propose within `wait-for-proposal`, validators broadcast a nullify and advance to the next view.

High view-change rates indicate network or configuration issues:

| Symptom | Likely cause | Fix |
|---|---|---|
| Frequent `tempo_consensus_view` increments without proposals | `wait-for-proposal` too short for network RTT | Increase by 2× P95 RTT |
| Stuttering block production but low view changes | `network-budget` too small; proposal returns late | Increase `network-budget` |
| Validator frequently skipped as leader (`inactive-views-until-leader-skip` exhausted) | Node is slower than peers or under-resourced | Check CPU, disk IO, or increase `inactive-views-until-leader-skip` |
| Spikes in view changes during DKG epochs | DKG ceremony competing with proposal CPU | Increase `worker-threads` temporarily |

## Fee Tokens and Fee AMM

### Built-in Fee Tokens

Gen 0 genesis creates four USD-denominated TIP-20 tokens with deterministic vanity addresses:

| Token | Address | Role |
|---|---|---|
| **PathUSD** | `0x20C0000000000000000000000000000000000000` | Default fee token. Used as fallback when no user/validator preference is set |
| **AlphaUSD** | `0x20C0000000000000000000000000000000000001` | Extra token; created unless `--no-extra-tokens` |
| **BetaUSD** | `0x20C0000000000000000000000000000000000002` | Same |
| **ThetaUSD** | `0x20C0000000000000000000000000000000000003` | Same |

All built-in tokens are initially minted to every genesis account with `u64::MAX` balance. `quoteToken` of every token points to PathUSD, enabling two-hop fee routing through a single PathUSD pair instead of requiring every `(userToken, validatorToken)` pair to have direct liquidity.

AlphaUSD/BetaUSD/ThetaUSD are demonstration tokens. **For production**, supply your own stablecoin TIP-20 tokens via `TIP20Factory` and disable the built-in extras with `--no-extra-tokens`.

### Custom Fee Tokens

TIP-20 tokens are created through the `TIP20Factory` precompile (`0x20FC0000...`). A token only qualifies as a fee token if it has a verified USD currency (`validate_usd_currency`).

```solidity
// Call TIP20Factory to create a new TIP-20 token
ITIP20Factory(0x20FC000000000000000000000000000000000000).createToken(
    "USD Coin",        // name
    "USDC",            // symbol
    6,                 // decimals
    1000000000000      // initial supply
)
```

The returned address is deterministic from the factory's CREATE2 logic (derived from salt = `tokenId`). USD currency verification requires:

1. The token's `currency()` matches one of the recognized stablecoin codes (USD, EUR, JPY, etc.)
2. Decimals ≤ 18
3. The token is registered in `TIP20Factory` (i.e., created through the factory)

### Fee AMM (TIPFeeAMM)

The Fee AMM is a constant-product stablecoin AMM co-located with the `TipFeeManager` precompile (`0xFEEC0000...`). It automatically converts user-paid fees to each validator's preferred token.

#### How Fee Payment Works

```
User Transaction
    │
    ▼
1. collect_fee_pre_tx(user_token, gas_price, gas_limit)
    ├── UserToken == ValidatorToken? → direct transfer (no swap)
    └── UserToken ≠ ValidatorToken? → AMM swap through TIPFeeAMM
         ├── Direct pool (userToken, validatorToken) has liquidity
         │   → swap and transfer
         └── No direct pool
             → two-hop: userToken → quoteToken → validatorToken

2. [EVM Execution]

3. collect_fee_post_tx(actual_spending) → refund unused fee
4. Validator calls distributeFees() → claims accumulated fees
```

#### Set Up Liquidity Pools

Each `(userToken, validatorToken)` pair needs a liquidity pool. Liquidity is provided via `mint` on the precompile:

```solidity
// Approve token transfer first
ITIP20(userToken).approve(0xFEEC..., amount);

// Add liquidity
ITIPFeeAMM(0xFEEC000000000000000000000000000000000000).mint(
    address userToken,
    address validatorToken,
    uint256 amountValidatorToken,    // amount of validatorToken to add
    address to                        // receives LP tokens
);
```

The AMM uses a constant-product formula with a 0.25% fee (`FEE_BPS = 25`). Key parameters:

| Parameter | Value |
|---|---|
| `M` (reserve ratio numerator) | `1` |
| `N` (reserve ratio denominator) | `1` |
| `SCALE` | `1e18` |
| `MIN_LIQUIDITY` | `1000` |

#### User Token Preference

Each user can set their preferred fee token (defaults to PathUSD if unset):

```bash
cast send 0xFEEC000000000000000000000000000000000000 \
  "setUserToken(address)" <TOKEN_ADDRESS> \
  --rpc-url <RPC_URL> --private-key <USER_KEY>
#
# Read current preference
cast call 0xFEEC000000000000000000000000000000000000 \
  "userTokens(address)" <USER_ADDRESS> \
  --rpc-url <RPC_URL>
```

#### Validator Token Preference

Each validator sets their preferred fee token (where they receive fees). Defaults to PathUSD if unset:

```bash
cast send 0xFEEC000000000000000000000000000000000000 \
  "setValidatorToken(address)" <TOKEN_ADDRESS> \
  --rpc-url <RPC_URL> --private-key <VALIDATOR_KEY>
#
# Read current preference
cast call 0xFEEC000000000000000000000000000000000000 \
  "validatorTokens(address)" <VALIDATOR_ADDRESS> \
  --rpc-url <RPC_URL>
```

Tokens must be:
- Registered in `TIP20Factory` (created through the factory)
- USD-denominated (validated by the TIP-20 token's `currency()`)

#### Distribute Fees

Validators (or any caller) can trigger fee distribution:

```bash
cast send 0xFEEC000000000000000000000000000000000000 \
  "distributeFees(address,address)" <VALIDATOR> <TOKEN> \
  --rpc-url <RPC_URL> --private-key <KEY>
```

This transfers all accumulated fees for `(validator, token)` to the validator's address. Accumulated fees are viewable via:

```bash
cast call 0xFEEC000000000000000000000000000000000000 \
  "collectedFees(address,address)" <VALIDATOR> <TOKEN> \
  --rpc-url <RPC_URL>
```

#### Two-Hop Routing (TIP-1033, T5+)

When the direct `(userToken, validatorToken)` pool has insufficient liquidity, the Fee AMM falls back to a two-hop route through `userToken.quoteToken()`. This means only `(userToken, quoteToken)` and `(quoteToken, validatorToken)` pools need liquidity, not every `(N×M)` pair.

In genesis, all built-in tokens have `quoteToken = PathUSD`, so:

```
USDC (user) → PathUSD → EURC (validator)
```

requires only:
- `(USDC, PathUSD)` pool — has liquidity from `--no-pairwise-liquidity` remaining enabled
- `(PathUSD, EURC)` pool — must be created separately via `mint`

### Deploying a Custom Stablecoin Chain

Minimal genesis command for a production chain with a single custom fee token:

```bash
cargo run --profile maxperf -p tempo-xtask -- generate-genesis \
  --output /etc/tempo/genesis \
  --no-extra-tokens \
  --deployment-gas-token \
  --deployment-gas-token-admin <ADMIN_ADDRESS> \
  -a <ACCOUNT_COUNT> \
  --chain-id <CHAIN_ID> \
  --validators <VALIDATOR_LIST> \
  --epoch-length 302400
```

This creates only PathUSD (for system operations) and the deployment gas token `DONOTUSE` (for genesis account funding). All users and validators set their own fee tokens at runtime via `setUserToken`/`setValidatorToken`, and liquidity providers add AMM pools on demand via `mint`.

## Cross-Chain Bridging

Tempo has no built-in bridge. All cross-chain transfers must be routed through external bridge protocols (LayerZero, Hyperlane, Wormhole, CCIP). The protocol provides the primitives for safe bridge integration.

### TIP-20 Bridge Primitives

| Primitive | TIP | Purpose |
|---|---|---|
| `burnAt(from, amount)` | TIP-1006 | Burn tokens from any address. Requires `BURN_AT_ROLE`. Designed for ERC-7802 (`crosschainBurn`) compatibility |
| `mint(to, amount)` / `mintWithMemo(to, amount, memo)` | TIP-20 | Mint tokens on destination. Requires `ISSUER_ROLE` |
| `setLogoURI(uri)` | TIP-20 | Set token metadata for bridge frontends |

`burnAt` is not yet implemented (status: Backlog). Current workaround: use the existing `burn(amount)` (self-burn, requires `ISSUER_ROLE`) or `burnBlocked(from, amount)` (requires `BURN_BLOCKED_ROLE`, only works on policy-blocked addresses).

### Replay Protection

Every signed object in the protocol binds to `chainId`:

| Component | Binding | Replay prevention |
|---|---|---|
| TempoTransaction `chain_id` field | `encode_for_signing` includes chainId | Cross-chain tx replay blocked at signature verification |
| KeyAuthorization `chain_id` | T1C+: exact match required. Pre-T1C: `0` = wildcard allowed | AdminKey replay across chains |
| ValidatorConfigV2 `addValidatorMessage` | `keccak256(chainId \|\| contractAddress \|\| ...)` | Cross-chain validator registration replay |

### Bridge Architecture

A standard setup uses **PathUSD** (`0x20C00000...`) as the canonical bridge token:

```
Tempo Chain A                     Tempo Chain B
┌─────────────────┐              ┌─────────────────┐
│  Bridge Contract │  lock/burn  │  Bridge Contract │
│  (e.g. OFT)      │◄──────────►│  (e.g. OFT)      │
│                  │  mint/unlock│                  │
│  holds PathUSD   │  relay      │  mints PathUSD   │
└─────────────────┘              └─────────────────┘
         │                              │
         └────────── Relayer ───────────┘
               (oracle, light client,
                multisig, or ZK proof)
```

### Integration Steps for Each Bridge Protocol

**1. Deploy the bridge's token wrapper on Tempo**

Each bridge protocol requires a wrapper contract around the TIP-20 token:

```solidity
// Example: LayerZero OFT adapter wrapping PathUSD
contract OFTAdapter is OFT {
    TIP20 public immutable pathUSD;

    function _debit(address from, uint256 amount, bytes memory) internal override returns (uint256) {
        // burnAt(from, amount) — requires BURN_AT_ROLE on PathUSD
        pathUSD.burnAt(from, amount);
        return amount;
    }

    function _credit(address to, uint256 amount, bytes memory) internal override {
        pathUSD.mint(to, amount);
    }
}
```

**2. Create liquidity for bridged tokens on the Fee AMM**

When bridged tokens arrive on a Tempo chain, they need Fee AMM liquidity for users to pay gas:

```bash
# Add PathUSD → native token pool
cast send 0xFEEC000000000000000000000000000000000000 \
  "mint(address,address,uint256,address)" \
  0x20C0000000000000000000000000000000000000 \  # source (PathUSD)
  0x20C0000000000000000000000000000000000000 \  # target (same = no swap needed)
  <AMOUNT> \
  <LP_RECIPIENT> \
  --rpc-url <RPC_URL> --private-key <KEY>
```

**3. Set the bridged token as default fee token** (optional)

If the bridged stablecoin should be the canonical fee token:

```bash
# Set for all validators
cast send 0xFEEC000000000000000000000000000000000000 \
  "setValidatorToken(address)" <BRIDGED_TOKEN> \
  --rpc-url <RPC_URL> --private-key <VALIDATOR_KEY>

# Optional: set as default user token
cast send 0xFEEC000000000000000000000000000000000000 \
  "setUserToken(address)" <BRIDGED_TOKEN> \
  --rpc-url <RPC_URL> --private-key <USER_KEY>
```

**4. Configure the bridge relayer**

The relayer monitors Tempo for `Transfer(burner, address(0), amount)` or custom `BurnAt` events, and mints on the destination. Standard bridge protocols handle this; Tempo provides no native relayer.

### Recommended Bridge Protocols

Tempo is EVM-compatible (Osaka hardfork target), so any EVM bridge protocol works:

| Protocol | Integration path | Notes |
|---|---|---|
| LayerZero | `OFT` | Deploy `OFTAdapter` wrapping TIP-20. Uses UltraLight Node + oracles |
| Hyperlane | `HypERC20` | Deploy `HypERC20` with `mailbox` pointing to Tempo's Hyperlane mailbox contract |
| Wormhole | NFT/Bridge | Integrate via Wormhole's token bridge. Requires deploying bridge contracts on Tempo |
| CCIP | TokenPool | Deploy a `TokenPool` that calls `burnAt`/`mint` |
| IBC (Composable) | csr-token | Requires a light client contract for Tempo on the counterparty |

### Bridging Validator Set (Interchain Security)

Tempo has no IBC or ICS. Each chain has its own `ValidatorConfigV2` contract and independent validator set. Cross-chain validator rotation is not possible through the protocol — validators must be added/removed independently on each chain.

The `chainId` embedded in `addValidatorMessage` and `rotateValidatorMessage` prevents a validator registration from one Tempo chain from being replayed on another.

## Monitoring

### Consensus Metrics

Tempo exposes Prometheus metrics at `--consensus.metrics-address`:

```
tempo_consensus_view
tempo_consensus_epoch
tempo_consensus_proposals_total
tempo_consensus_notarizations_total
tempo_consensus_finalizations_total
tempo_dkg_manager_ceremony_players
tempo_dkg_manager_ceremony_dealers
tempo_epoch_manager_how_often_signer_total
```

Execution layer metrics at `--metrics` (Reth-standard).

### Health Checks

```bash
# Node is producing blocks
cast block-number --rpc-url http://localhost:8545

# Node is participating in consensus (returns latest consensus state)
curl http://localhost:8545 -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"consensus_getLatest","params":[],"id":1}'

# Peer count
cast rpc net_peerCount --rpc-url http://localhost:8545
```

### Logs

Structured JSON logs at `--log.file.directory` with configurable level via `RUST_LOG`:

```bash
RUST_LOG=info,tempo_consensus=debug,tempo_execution=warn
```

### Telemetry (OTLP)

```bash
# Enable OpenTelemetry metrics and logs
tempo node --telemetry \
  --telemetry.logs-otlp-url http://otel-collector:4318/v1/traces \
  --telemetry.metrics-prometheus-url http://victoriametrics:8428/api/v1/write
```

## Disaster Recovery

### State Resync

A validator that loses state can recover by replaying blocks from genesis from a peer:

```bash
# Backup current data first
mv /var/lib/tempo /var/lib/tempo.corrupt

# Start fresh; the node will backfill from peers
systemctl start tempo
```

### Consensus Data Corruption

Tempo maintains separate storage for consensus (BLS shares, DKG state) at `--consensus.datadir`. If only consensus data is corrupt but execution state is intact:

```bash
rm -rf /var/lib/tempo/consensus
# On restart, the node re-derives consensus state from the last finalized block
```

### Key Compromise

1. Owner calls `ValidatorConfigV2.rotateValidator()` on-chain to replace the compromised identity
2. Generate a new `signing.key` and distribute securely
3. Restart the node with new key material

## Storage Estimation

| Component | Growth Rate (est.) |
|---|---|
| Execution state (reth) | ~20 GiB/day at 0.5s blocks |
| Consensus archive | ~2 GiB/day |
| Pruned full node | ~1 TiB/year |
| Archive node | ~15 TiB/year |

Pruning is handled by reth's built-in pipeline; consensus data prunes when `--consensus.finalized-blocks-retention` fills.

## Hardfork Schedule

Hardforks are time-activated in genesis (T0–T8). Design activation timestamps when configuring a new chain:

| Fork | Key Changes |
|---|---|
| T0 — Genesis | TIP-20 tokens, Fee AMM, ValidatorConfig, NonceManager |
| T1 | Base fee model, EIP-1559-style fee market |
| T1.A | Tempo transactions (AA), keychain accounts |
| T1.B | Fee sponsorship, 2D nonces |
| T1.C | Journal checkpoints for precompile atomicity |
| T2 | EIP-712 permit on TIP-20 |
| T3 | Virtual addresses, signature verifier precompile |
| T4 | Refined gas accounting, SSTORE refund propagation |
| T5 | TIP20ChannelReserve (payment channels) |
| T6 | Receive policy guard |
| T7 | TIP-1060 storage credits |
| T8 | Sequential storage gas accounting |

Set T0–T8 to the same timestamp for immediate activation on a new chain, or stagger for phased rollout.
