# Decentralized Git on EthStorage

## 1. Vision: A Fully Verifiable Open-Source Infrastructure

Modern open-source collaboration relies heavily on centralized services like GitHub and npm. While Git itself is decentralized, **the hosting and distribution layers are not**—a single point of failure or compromise can affect the entire software supply chain.  

Our goal is to build a **fully decentralized GitHub**:  
- **Git branch information** — such as which commit a branch (e.g., main) currently points to — is recorded and updated by a on-chain Git contract to ensure a transparent and tamper-proof history.
- **Git objects** are stored as **packfiles** — compact binary bundles containing all commits, trees, and file contents — on EthStorage, Ethereum’s decentralized storage layer for blobs. Users upload data through blob-carrying transactions to the EthStorage L1 contract with a storage fee.  Then, the EthStorage off-chain storage network (a.k.a., a storage L2) will permanently preserve these blobs and submit storage proofs to the EthStorage L1 contract.    
- **Users can clone and push** using the same Git commands — now backed by Ethereum’s trust guarantees.  

This aligns with Vitalik’s call for *“full-stack openness and verifiability”*, ensuring that every layer — from code to deployment — is transparent and independently reproducible.

## 2. Core Principle — How Decentralized Git Works

### 2.1 Git’s Local Data Model

 - A **commit** object records a snapshot of the project (represented by a tree); Except for the initial commit, each commit has one or more parent commits (in the case of merges).
 - A **tree** object represents **directory structure** — it maps names to either subdirectories (trees) or files (blobs).
 - A **blob** object contains the actual **file content** - the data of each versioned file.
 - A **ref** (reference) is a human-readable pointer that maps a branch name (e.g., refs/heads/main) to a specific commit’s object ID (OID). When you make a new commit, Git updates the ref to point from the old commit (oldOid) to the new one (newOid)

All of the Git's objects are **content-addressed** — each is identified by a 20-byte SHA-1 (or SHA-256) **object ID** (OID). These OIDs form a cryptographic chain linking every commit to its parents, ensuring the entire project history is tamper-evident and verifiable — the foundation of Git’s integrity model.

For efficiency, Git bundles related objects (commits, trees, and blobs) into a **packfile**, a compact binary format that delta-compresses objects relative to one another. When pushing or fetching, Git determines the difference between the local and remote repositories, then packs all missing objects (from the common ancestor commit up to the latest commit) into a single packfile for transmission. For a deeper look at Git’s smart transfer protocol, see the Git [documentation](https://git-scm.com/book/en/v2/Git-Internals-Transfer-Protocols).

A centralized Git service like GitHub essentially provides:
- A mapping of refs (e.g., `refs/heads/main → commit hash`), and  
- A storage backend for Git objects and packfiles.

### 2.2 Git’s On-Chain Data Model

Git’s on-chain data model separates **logical state** (branch references and commit mappings) from **data persistence** (packfiles). The **Ethereum L1 contract** records which commit a branch currently points to and stores the hash of the corresponding packfile. Meanwhile, **EthStorage L2** permanently stores the packfile data itself — uploaded via blob-carrying transactions and retrievable by content hash.

Push and fetch operations thus span both layers:
 - **Push** updates refs on-chain and uploads new packfiles to EthStorage.
 - **Fetch** (or clone) reads refs and packfile hashes from the contract, then retrieves the corresponding packfiles from EthStorage to reconstruct the repository locally.

| Git Concept | On-chain Equivalent | Ethereum L1 (Contract Layer) | EthStorage L2 (Blob Storage) |
|--------------|--------------------|----------------|----------------|
| **Ref (e.g., refs/heads/main)** | Smart contract variable recording the current **commit OID (`newOid`)** | ✅ Stores branch → commit mapping | ❌ |
| **Packfile (objects delta)** | Compact binary bundle containing commits, trees, and blobs | ❌ | ✅ Permanently stores packfiles as content-addressed blobs |
| **Push (update refs)** | Uploads new packfile and updates on-chain refs | ✅ Calls updateRefs(oldOid, newOid, packfileHash) | ✅ Uploads packfile via blob-carry transaction |
| **Fetch / Clone** | Reads refs and downloads packfiles to reconstruct the repo | ✅ Reads refs & packfileHash from contract | ✅ Downloads packfiles by hash |

Thus:
- **Refs and updates** are verifiable on-chain.  
- **Objects** are stored as immutable blobs on EthStorage.  
- **Integrity** is guaranteed by cryptographic hashes linking the two layers.

### 2.3 End-to-End Workflow (Clone & Push)

A decentralized Git remote looks like this:

```
ethfs://dehub.eth/vitalik-blog
```

- `ethfs://` denotes the EthStorage-based Git protocol that connects Git to smart contracts and decentralized storage.  
- `dehub.eth` is an ENS-resolved **DeHub registry contract** that manages repositories on-chain.  
- `vitalik-blog` is a **Repo contract** deployed from DeHub.  
- The code itself resides as packfiles in EthStorage.  

#### Clone

1. User runs `git clone ethfs://dehub.eth/vitalik-blog`.  
2. Git calls the [git remote helper](#how-git-remote-helper-works) binary `git-remote-ethfs`.  
3. The helper resolves `dehub.eth` → DeHub contract → Repo contract, fetches branch refs, and obtains corresponding `packfileHash`s.  
4. It downloads packfiles from EthStorage using these hashes and reconstructs the full repository locally.

#### Push

1. Git computes the delta between local and remote and generates a packfile.  
2. The **git remote helper** submits this packfile via a blob-carrying transaction.  
3. EthStorage nodes permanently store the blob and submit proofs to the L1 contract.  
4. The helper then calls `updateRefs()` on the Repo contract with the new commit hash and `packfileHash`.  

#### Why This Matters

This model preserves Git’s local logic (objects and refs) but replaces the trusted central server with verifiable on-chain coordination and decentralized storage — **same Git, new trust model**.

#### How Git Remote Helper Works

When you run standard Git commands like:

```bash
git clone ethfs://dehub.eth/vitalik-blog
git push ethfs://dehub.eth/vitalik-blog
```

Git automatically call a helper binary named:

```bash
git-remote-ethfs
```

The helper communicates with Git over a simple stdin/stdout protocol:
 - list → list refs
 - fetch → download pack(s)
 - push → upload pack(s)

It then translates these operations into backend actions:
 - calling smart contracts to update refs and upload packfiles in blobs, and
 - reading/writing packfiles to EthStorage.

## 3. Architecture Overview

| Layer | Responsibility |
|-------|----------------|
| **DeHub (Factory + Registry Contract)** | Registers and manages all repositories. Deploys lightweight Repo contracts with deterministic addresses (`CREATE2`). |
| **Repo Contract** | Stores refs (`refs/heads/main → commit hash`), enforces update rules (fast-forward, permissions), and emits verifiable push events. |
| **EthStorage** | Stores Git pack files (commits, trees, blobs) as permanent, content-addressed blobs. |
| **Client (Git Remote Helper)** | Integrates with native Git CLI. When you run `git push` or `git clone`, it uploads/fetches packs from EthStorage and calls `updateRefs()` on the Repo contract. |

## 4. Smart Contract Design

### 4.1. DeHub Contract
- Acts as the global registry:  
  `createRepo(name, owner)` → deploys a new Repo contract using minimal proxy (EIP-1167).  
- Maps repository names to their contract addresses and emits `RepoCreated` events.  
- Manages ownership, access control, and optional DAO-based governance.

### 4.2. Repo Contract
- Maintains the mapping of Git refs to commit hashes:
  ```solidity
  mapping(bytes => bytes20) public refs;
  ```
- Supports atomic multi-ref updates:
  ```solidity
  function updateRefs(Update[] calldata updates) external;
  ```
  where `Update` includes `{ name, oldOid, newOid, packfileHash }`.
- Verifies fast-forward by checking `oldOid` consistency.
- Emits `RefUpdated(ref, oldOid, newOid, packfileHash, sender)` events for indexing.
- Optional writer/maintainer list for permission control.

### 4.3. EthStorage Integration
Each push produces a Git packfile representing the delta between the local and remote state.

The client uploads this packfile through a blob-carry transaction, after which EthStorage nodes permanently store the blob as part of the Ethereum data layer.

Later, the Git remote helper retrieves the same packfile directly from EthStorage using its packfileHash, reconstructing the repository state locally.

## 5. Example Workflow

### Creating a Repo
```solidity
DeHub.createRepo("vitalik-blog", 0xUserAddress);
```

### Pushing a Commit
1. The client uploads new pack file to EthStorage.  
2. Computes `packfileHash`.  
3. Calls:
   ```solidity
   Repo.updateRefs("refs/heads/main", oldOid, newOid, packfileHash);
   ```
4. Contract emits `RefUpdated`, anchoring the commit.

### Cloning
1. `git clone ethfs://dehub.eth/vitalik-blog`  
2. Client resolves `dehub.eth` → DeHub contract → repo address.  
3. Fetches current refs and packfileHashs.  
4. Downloads pack files from EthStorage and reconstructs the repository locally.

## 6. Roadmap

| Phase | Milestone |
|-------|------------|
| **Phase 1 – Contract Launch** | Deploy DeHub (registry + factory) and Repo base implementation. Support ENS name resolution (`dehub.eth`). |
| **Phase 2 – EthStorage Integration** | Push/fetch Git objects through EthStorage packfile hashes. |
| **Phase 3 – Git Helper Integration** | Release `git-remote-ethfs` plugin to enable `git push` / `git clone` directly. |
| **Phase 4 – Governance and Permissions** | Multi-sig / DAO-controlled writer sets, repo ownership transfer, and organization namespaces. |

## 7. Outlook

This design creates a **verifiable, permanent, and open developer infrastructure**:
- Every commit is cryptographically anchored to Ethereum.  
- Every byte of code is permanently stored and retrievable from EthStorage.  
- No single party controls code distribution or versioning.  

Over time, this model can extend beyond code to packages, data models, or even front-end assets — forming the foundation of **a decentralized, verifiable developer stack**.
