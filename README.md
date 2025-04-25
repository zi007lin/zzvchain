# ZZVChain
*Zero‑trust, Ultra‑low‑latency, **V**erified DAG Ledger Framework*

> **Repository name suggestion:** `zzvchain`
>
> (You can later add language‑specific sub‑repos such as `zzvchain-java`, `zzvchain-rust`, etc. if you decide to split the codebase.)

---

**Last updated:** 2025-04-25

---

## Table of Contents
1. [Background](#background)
2. [Key Features](#key-features)
3. [High‑Level Architecture](#high-level-architecture)
4. [Quick Start](#quick-start)
5. [Module Breakdown](#module-breakdown)
6. [Roadmap](#roadmap)
7. [Contributing](#contributing)
8. [License](#license)
9. [Contact](#contact)

---

## Background
`ZZVChain` merges proven, production‑grade components—**Kafka** for log durability, a **vector‑timestamp DAG (ZetaGraph)** for fast quorum finality, and a **genesis‑anchored zero‑trust layer**—into a single modular consensus framework. Nodes are compiled as native images with **GraalVM** to achieve sub‑millisecond commit latency while exposing **gRPC‑native APIs over port 443**.

## Key Features
| Capability | Description |
|------------|-------------|
| **Zero‑trust** | Mutual TLS everywhere; genesis root cert anchors the network. |
| **Ultra‑low‑latency** | Native images + lock‑free queues target < 1 ms end‑to‑end. |
| **Verified DAG** | Vector DAG edges ensure causal ordering and fast finality without classic PBFT overhead. |
| **Kafka durability** | Each event is persisted to an append‑only log before DAG consensus—no data loss. |
| **Modular** | Pluggable transports (Kafka, RabbitMQ), consensus engines, and ledger stores. |
| **gRPC‑first** | Uniform API surface; HTTP/2 multiplexing over 443 for easier firewall traversal. |

## High‑Level Architecture
```
┌──────────────┐      gRPC      ┌──────────────┐
│  Client SDK  │──────────────▶│  zzv-gateway │
└──────────────┘               └──────┬───────┘
                                      │ events
                                      ▼
                                ┌──────────────┐
                                │   Kafka ✔    │
                                └──────┬───────┘
                                 stream│durable
                                      ▼
                         ┌─────────────┴─────────────┐
                         │  zzv-graph  │  zzv-quorum │
                         └─────────────┬─────────────┘
                                       │finalized DAG
                                       ▼
                                ┌──────────────┐
                                │ zzv-ledger   │
                                └──────────────┘
```
*Fig 1 – Simplified data flow (events travel left→right)*

A more detailed diagram with component internals lives in `./docs/architecture/diagram.svg` (to be committed).

## Quick Start
```bash
# Clone monorepo
git clone https://github.com/zi007lin/zzvchain.git
cd zzvchain

# Build core services (requires GraalVM 21 + Docker)
./gradlew buildNative

# Launch a 3‑node demo network
./scripts/devnet-up.sh

# Send a test transaction
./scripts/send-demo-tx.sh
```
The dev‑net publishes gRPC reflection; use `grpcurl -plaintext localhost:7443 list` to explore APIs.

## Module Breakdown
| Directory | Description |
|-----------|-------------|
| `zzv-graph` | Vector‑timestamp DAG engine (Rust) |
| `zzv-quorum` | Quorum & finality service (Kotlin) |
| `zzv-ledger` | Ledger persistence + Merkle proofs (Java) |
| `zzv-gateway` | gRPC/REST ingress, auth, rate‑limiting (Go) |
| `zzv-genesis` | Bootstrap & certificate authority (Python) |
| `cli/zzvctl` | Cross‑platform CLI (Rust) |

## Roadmap
- **Q2 2025** – Publish alpha dev‑net & public SDKs (TS/Python/Go)
- **Q3 2025** – Pluggable consensus adapters (Raft, HotStuff)
- **Q4 2025** – Main‑net beta with cross‑region KRaft cluster
- **Early 2026** – OpenTelemetry deep‑link + Grafana dashboards

## Contributing
We welcome PRs for bug fixes, docs, and new transports. Please open an issue first for major changes. All contributors are required to sign the [DCO](docs/DCO.txt).

## License
`ZZVChain` is released under the **Apache License 2.0**. See `LICENSE` for details.

## Contact
| Role | Handle |
|------|--------|
| Maintainer | **Zi Lin (@zi007lin)** |
| Core repo | <https://github.com/zi007lin/zzvchain> |
| Discord | `#zzvchain` on zk‑infra |

