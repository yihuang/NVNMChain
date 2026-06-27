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
  -a 50000 \                                    # genesis-funded accounts
  --chain-id 42432 \                            # unique chain ID
  --validators 10.0.0.1:8000,10.0.0.2:8000     # comma-separated validator P2P addrs
  --t0-time 0 --t1-time ... --t8-time <TS>      # hardfork activation timestamps
  --gas-limit 500000000                         # block gas limit
  --epoch-length 302400                         # blocks per epoch (≈7d at 0.5s blocks)
  --seed <SEED>                                 # deterministic key generation (dev only)
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
  --ws --ws.addr 0.0.0.0 --ws.port 8545 \
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
tempo node --follow wss://validator-a.example.com:8545 \
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
