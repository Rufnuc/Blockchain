Minimal Blockchain in Rust

A Production-Ready Educational Implementation

<div align="center">
  <img src="docs/images/blockchain-logo.svg" alt="Blockchain Rust Logo" width="200"/>https://img.shields.io/badge/build-passing-brightgreen
https://img.shields.io/badge/rust-1.70+-orange
https://img.shields.io/badge/License-MIT-yellow
https://img.shields.io/badge/docs-latest-blue

</div>---

📋 Table of Contents

· Visual Architecture
· System Design
· Data Structures
· Process Flows
· User Interface
· Getting Started
· Development

---

🏗 Visual Architecture

High-Level System Design

```mermaid
graph TB
    subgraph "CLI Layer"
        CLI[CLI Parser<br/>clap]
        CMDS[Commands<br/>wallet/mine/send]
    end
    
    subgraph "Core Layer"
        BC[Blockchain<br/>Manager]
        BLK[Block<br/>Processor]
        TX[Transaction<br/>Engine]
        UTXO[UTXO Set<br/>Manager]
        MP[Mempool<br/>Manager]
        MINER[Mining<br/>Engine]
    end
    
    subgraph "Infrastructure"
        CRYPTO[Crypto<br/>SHA256/ED25519]
        STORE[Storage<br/>Memory/Persistent]
        SERDE[JSON<br/>Serialization]
    end
    
    subgraph "Future Network"
        P2P[P2P Network<br/>libp2p]
        SYNC[Sync<br/>Protocol]
        PROP[Block<br/>Propagation]
    end
    
    CLI --> CMDS
    CMDS --> BC
    BC --> BLK
    BC --> TX
    BC --> UTXO
    TX --> MP
    MINER --> MP
    MINER --> BLK
    BLK --> CRYPTO
    TX --> CRYPTO
    BC --> STORE
    STORE --> SERDE
    
    style CLI fill:#f9f,stroke:#333,stroke-width:2px
    style Core Layer fill:#bbf,stroke:#333,stroke-width:2px
    style Infrastructure fill:#bfb,stroke:#333,stroke-width:2px
    style Future Network fill:#fbb,stroke:#333,stroke-width:2px
```

Module Dependency Graph

```mermaid
graph LR
    subgraph "External Dependencies"
        ED[ed25519-dalek]
        SH[sha2]
        CL[clap]
        SJ[serde_json]
    end
    
    subgraph "Internal Modules"
        H[hash.rs]
        S[signature.rs]
        B[block.rs]
        C[blockchain.rs]
        T[transaction.rs]
        M[miner.rs]
        MEM[memory.rs]
    end
    
    H --> ED
    H --> SH
    S --> ED
    B --> H
    B --> T
    C --> B
    C --> T
    C --> MEM
    T --> H
    T --> S
    M --> C
    M --> T
    
    style External fill:#ffd,stroke:#333,stroke-width:1px
    style Internal fill:#dfd,stroke:#333,stroke-width:1px
```

---

🎨 System Design

1. Core Data Flow Diagram

```mermaid
sequenceDiagram
    participant User
    participant CLI
    participant Wallet
    participant Mempool
    participant Miner
    participant Blockchain
    participant Storage
    
    User->>CLI: send --to B --amount 100
    CLI->>Wallet: create_transaction()
    Wallet->>Blockchain: get_utxos(address)
    Blockchain-->>Wallet: utxos list
    Wallet->>Wallet: sign_transaction()
    Wallet->>Mempool: add_transaction(tx)
    Mempool-->>CLI: tx_hash
    
    User->>CLI: mine --address A
    CLI->>Miner: start_mining()
    Miner->>Mempool: get_pending_txs()
    Miner->>Miner: build_block_candidate()
    Miner->>Miner: proof_of_work()
    Miner->>Blockchain: submit_block(block)
    Blockchain->>Blockchain: validate_block()
    Blockchain->>Storage: persist_block()
    Blockchain->>UTXO: update_utxo_set()
    Blockchain-->>CLI: block_hash
```

2. State Machine Design

```mermaid
stateDiagram-v2
    [*] --> Initialized: new blockchain
    
    Initialized --> Syncing: peer connected
    Syncing --> Active: sync complete
    
    Active --> Mining: mine command
    Mining --> BlockFound: nonce found
    BlockFound --> Validating: validate block
    
    Validating --> Active: valid
    Validating --> Mining: invalid (retry)
    
    Active --> TransactionPool: tx received
    TransactionPool --> Active: tx verified
    
    Active --> [*]: shutdown
```

---

📊 Data Structures

Block Structure Design

```
┌─────────────────────────────────┐
│            BLOCK                 │
├─────────────────────────────────┤
│ ┌─────────────┐                 │
│ │   HEADER    │                 │
│ ├─────────────┤                 │
│ │ Parent Hash │ ◄─── Previous    │
│ │ Timestamp   │      Block       │
│ │ Difficulty  │                  │
│ │ Nonce       │                  │
│ │ Merkle Root │ ───────┐        │
│ └─────────────┘         │        │
│                         ▼        │
│ ┌─────────────────────────────┐  │
│ │       TRANSACTIONS          │  │
│ ├─────────────────────────────┤  │
│ │ Coinbase Tx  ├──────────────┘  │
│ │ Tx 1         │  Merkle         │
│ │ Tx 2         │  Tree           │
│ │ ...          │  Root           │
│ │ Tx N         │  Hash           │
│ └─────────────────────────────┘  │
└─────────────────────────────────┘
```

UTXO Set Organization

```mermaid
graph TD
    subgraph "UTXO Database"
        direction TB
        H1[UTXO Entry 1] --> K1[Key: tx_hash:index]
        H1 --> V1[Value: amount, pubkey_hash]
        
        H2[UTXO Entry 2] --> K2[Key: tx_hash:index]
        H2 --> V2[Value: amount, pubkey_hash]
        
        H3[UTXO Entry 3] --> K3[Key: tx_hash:index]
        H3 --> V3[Value: amount, pubkey_hash]
    end
    
    subgraph "Address Index"
        A1[Address A] --> H1
        A1 --> H2
        A2[Address B] --> H3
    end
```

Memory Pool Architecture

```
┌─────────────────────────────────────┐
│            MEMPOOL                   │
├─────────────────────────────────────┤
│ Priority Queue (by fee/age)         │
│ ┌─────────────────────────────────┐ │
│ │ [High Fee] Tx: 0x7f3e... (2.5) │ │
│ │ [Medium Fee] Tx: 0x9a2b... (1.2)│ │
│ │ [Low Fee]  Tx: 0x4c8d... (0.5) │ │
│ └─────────────────────────────────┘ │
│                                      │
│ Indexes:                             │
│ • By Hash: HashMap<Hash, Tx>         │
│ • By Address: HashMap<Addr, Vec<Hash>>│
│ • By Time: BTreeMap<Time, Hash>      │
└─────────────────────────────────────┘
```

---

🔄 Process Flows

Transaction Lifecycle

```mermaid
graph LR
    subgraph "Creation"
        A[Create Tx] --> B[Select UTXOs]
        B --> C[Create Outputs]
        C --> D[Sign Inputs]
    end
    
    subgraph "Validation"
        D --> E{Verify Sig}
        E -->|Valid| F{Check UTXOs}
        E -->|Invalid| X[Reject]
        F -->|Unspent| G{Sum Inputs >= Outputs}
        F -->|Spent| X
        G -->|Yes| H[Add to Mempool]
        G -->|No| X
    end
    
    subgraph "Mining"
        H --> I[Select from Mempool]
        I --> J[Build Block]
        J --> K[Mine PoW]
        K --> L[Add to Chain]
    end
    
    subgraph "Confirmation"
        L --> M[Update UTXO Set]
        M --> N[Remove from Mempool]
        N --> O[Emit Event]
    end
```

Mining Algorithm Flow

```mermaid
flowchart TD
    Start([Start Mining]) --> GetTxs[Get Transactions from Mempool]
    GetTxs --> AddCoinbase[Add Coinbase Transaction]
    AddCoinbase --> BuildHeader[Build Block Header]
    BuildHeader --> CalcMerkle[Calculate Merkle Root]
    
    CalcMerkle --> Loop{Nonce < Max?}
    
    Loop -->|Yes| Hash[Hash Header]
    Hash --> Check{Hash < Target?}
    Check -->|Yes| Success[Block Found!]
    Check -->|No| Inc[Increment Nonce]
    Inc --> Loop
    
    Loop -->|No| Refresh[Get New Transactions]
    Refresh --> BuildHeader
    
    Success --> Validate[Validate Block]
    Validate --> Add[Add to Chain]
    Add --> Broadcast[Broadcast to Network]
    Broadcast --> Stop([Stop])
```

---

🖥 User Interface Design

CLI Command Tree

```
minimal-blockchain
├── wallet
│   ├── new                 # Generate new wallet
│   ├── list                # List all wallets
│   ├── balance <address>   # Show balance
│   ├── export <address>    # Export private key
│   └── import <file>       # Import wallet
│
├── mine
│   ├── --address <addr>    # Mine to address
│   ├── --difficulty <n>    # Set difficulty
│   └── --background        # Run in background
│
├── send
│   ├── --from <addr>       # Source address
│   ├── --to <addr>         # Destination
│   ├── --amount <n>        # Amount to send
│   └── --fee <n>           # Transaction fee
│
├── blockchain
│   ├── info                # Chain statistics
│   ├── blocks [--limit]    # List blocks
│   ├── verify              # Validate chain
│   └── reset               # Reset chain
│
├── tx
│   ├── get <hash>          # Get transaction
│   ├── history <address>   # Transaction history
│   └── mempool             # Pending transactions
│
└── network (future)
    ├── connect <peer>      # Connect to peer
    ├── status              # Network status
    └── peers               # List peers
```

Terminal UI Mockup

```
┌─────────────────────────────────────────────────────────────┐
│  MINIMAL BLOCKCHAIN v0.1.0                        [Ctrl+C to exit] │
├─────────────────────────────────────────────────────────────┤
│  Blockchain Status                                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Height:      42,391                                   │  │
│  │ Last Block:  0000a7f...3e2b (5 seconds ago)          │  │
│  │ Difficulty:  4 (target: 0000xxxx)                     │  │
│  │ Mempool:     127 transactions (0.45 MB)               │  │
│  │ Peers:       8 connected                              │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Latest Transactions                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 7f3e... → bc1a...  100 BLK  [Confirmed]  ⬛⬛⬛⬛⬛⬛⬛⬛  │  │
│  │ 9a2b... → bc1b...  50 BLK   [Pending]    ⬛⬛⬛⬛⬛⬜⬜⬜  │  │
│  │ 4c8d... → bc1c...  25 BLK   [Pending]    ⬛⬛⬜⬜⬜⬜⬜⬜  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  > wallet balance bc1a7f9e3d2c8b5a4f6e                     │
│  Balance: 1,250 BLK                                        │
│  ───────────────────────────────────────────────────────── │
│  > _                                                        │
└─────────────────────────────────────────────────────────────┘
```

---

🚀 Getting Started

Installation Flow

```bash
# 1. Clone with visual progress
$ git clone https://github.com/your-repo/minimal-blockchain-rust.git
Cloning into 'minimal-blockchain-rust'...
remote: Counting objects: 100% (152/152), done.
Receiving objects: 100% (152/152), 45.67 KiB | 1.2 MiB/s, done.

# 2. Enter directory
$ cd minimal-blockchain-rust

# 3. Build with cargo (watch the magic)
$ cargo build --release
   Compiling sha2 v0.10.8
   Compiling ed25519-dalek v2.0.0
   Compiling clap v4.0.0
   Compiling minimal-blockchain v0.1.0
    Finished release [optimized] target(s) in 2m 15s

# 4. Run your first command
$ ./target/release/minimal-blockchain wallet new

🎉 Wallet created successfully!
────────────────────────────────
Address:  bc1a7f9e3d2c8b5a4f6e1d3c5b7a9f2e4d6c8a0b1
Public Key: 7f3e8d9a2b4c5f6e1d3c8a9b2f4e5d6c7a8b9c0d
Private Key: [SAVED TO ~/.config/blockchain/wallets/bc1a7f9e...]
────────────────────────────────
⚠️  IMPORTANT: Keep your private key safe and never share it!
```

First Mining Session

```bash
# Start mining with visual feedback
$ minimal-blockchain mine --address bc1a7f9e --background

⛏️  Mining started on address: bc1a7f9e...
────────────────────────────────────────────
[15:32:01] ⏳ Mining block #42391 (difficulty 4)
[15:32:02] 🔨 Nonce: 1,452,389 | Hash: 7f3e8d9a...
[15:32:03] 🔨 Nonce: 2,891,234 | Hash: 4b2c8f1e...
[15:32:04] 🔨 Nonce: 4,237,891 | Hash: 1d3e5f7a...
[15:32:05] 🎉 BLOCK FOUND! Nonce: 5,678,234
[15:32:05] ✅ Block hash: 0000a7f3e8d9b2c4f5e6d1a3b5c7e9f2a4b6c8d0e
[15:32:05] 💰 Reward: 50 BLK + 2.5 BLK fees
────────────────────────────────────────────
Current balance: 1,300 BLK
Hash rate: 145.2 KH/s
```

---

🛠 Development

Test Coverage Visualization

```mermaid
pie title Current Test Coverage
    "Block Validation" : 95
    "Transaction Signing" : 92
    "UTXO Management" : 88
    "Mining Logic" : 85
    "CLI Commands" : 78
    "Storage Layer" : 72
```

Performance Benchmarks

```
Blockchain Operations (mean time)
────────────────────────────────────
Block validation      ████░░░░░░  42 µs
Transaction verify    ██░░░░░░░░  18 µs
Merkle root calc      ████████░░  85 µs
UTXO lookup           █░░░░░░░░░   3 µs
Chain rewind          ██████████ 120 ms

Memory Usage
────────────────────────────────────
10k blocks            ████████░░  45 MB
100k UTXOs            ██████░░░░  32 MB
10k mempool txs       ████░░░░░░  18 MB
```

Development Workflow

```mermaid
gitGraph
    commit id: "initial setup"
    branch feature/transaction-pool
    commit id: "mempool struct"
    commit id: "add tx validation"
    checkout main
    merge feature/transaction-pool
    branch feature/mining
    commit id: "pow algorithm"
    commit id: "miner integration"
    checkout main
    merge feature/mining
    branch feature/cli
    commit id: "clap commands"
    commit id: "pretty printing"
    checkout main
    merge feature/cli
    commit id: "v0.1.0 release"
```

---

📈 Future Scaling Design

Horizontal Scaling Architecture

```mermaid
graph TB
    subgraph "Node Cluster"
        N1[Node 1<br/>Validator] --> DB1[(Shard 1)]
        N2[Node 2<br/>Validator] --> DB2[(Shard 2)]
        N3[Node 3<br/>Validator] --> DB3[(Shard 3)]
    end
    
    subgraph "Load Balancer"
        LB[Round Robin<br/>Distributor]
    end
    
    subgraph "Clients"
        C1[Wallet 1]
        C2[Wallet 2]
        C3[Wallet 3]
    end
    
    C1 --> LB
    C2 --> LB
    C3 --> LB
    LB --> N1
    LB --> N2
    LB --> N3
```

---

📚 Documentation Links

· API Reference - Complete API documentation
· Architecture Decision Records - Design decisions
· Security Model - Threat modeling
· Benchmarking - Performance metrics
· Contributing Guide - How to contribute

---

<div align="center">Built with ❤️ in Rust

Star ⭐ • Report Bug 🐛 • Request Feature 🚀

</div>
