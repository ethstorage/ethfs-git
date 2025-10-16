# Decentralized Git on EthStorage

## 1. Vision: A Fully Verifiable Open-Source Infrastructure

Modern open-source collaboration relies heavily on centralized services like GitHub and npm. While Git itself is decentralized, **the hosting and distribution layers are not**—a single point of failure or compromise can affect the entire software supply chain.  

Our goal is to build a **fully decentralized GitHub**:  
- where repositories are **hosted on decentralized storage** (EthStorage),  
- refs and commit histories are **anchored on-chain**,  
- and users can **clone, push, and verify code** with the same Git commands, but backed by Ethereum’s trust guarantees.  

This aligns with Vitalik’s call for *“full-stack openness and verifiability”*, ensuring that every layer — from code to deployment — is transparent and independently reproducible.

## 2. Core Principle — How Decentralized Git Works

### 2.1 Git’s Data Model

**Git objects** are the atomic units of a repository:
 - **Commit** objects record a snapshot of the project and link to a specific tree and parent commits.
 - **Tree** objects represent **directory structures** — they map file names to subdirectories (trees) or files (blobs).
 - **Blob** objects contain the actual **file contents** (the data of each versioned file).

All of these objects are content-addressed and linked by cryptographic hashes, forming a tree structure — the foundation of Git’s verifiability.

A centralized Git service like GitHub essentially provides:
 - A mapping of refs (e.g., refs/heads/main → commit hash), and
 - A storage backend for Git objects.

### 2.2 Packfiles

For efficiency, Git bundles many of these objects into a **packfile** — a compact, binary container that stores multiple objects (often delta-compressed).
Packfiles are how Git transmits data between repositories and also how it optimizes on-disk storage.

### 2.3 Decentralizing the Stack

To decentralize this:
- **Refs** move on-chain (managed by smart contracts).  
- **Objects (packfiles)** move to **EthStorage**, Ethereum’s native decentralized blob storage.  
- **Git clients** interact with these through a thin remote helper that speaks the standard Git protocol but resolves to on-chain contracts instead of a centralized server.

So instead of:
```
https://github.com/user/repo.git
```
we have:
```
eths://dehub.eth/vitalik-blog
```
where:
- `dehub.eth` is an ENS-resolved **DeHub contract**,  
- `vitalik-blog` is a **repo** registered on-chain,  
- and the code itself lives on EthStorage, verifiable and permanent.

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
Each push corresponds to a packfile stored in EthStorage.

The `packfileHash` is recorded in the `updateRefs()` call, ensuring full data-chain verifiability.

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
1. `git clone eths://dehub.eth/vitalik-blog`  
2. Client resolves `dehub.eth` → DeHub contract → repo address.  
3. Fetches current refs and packfileHashs.  
4. Downloads pack files from EthStorage and reconstructs the repository locally.

## 6. Roadmap

| Phase | Milestone |
|-------|------------|
| **Phase 1 – Contract Launch** | Deploy DeHub (registry + factory) and Repo base implementation. Support ENS name resolution (`dehub.eth`). |
| **Phase 2 – EthStorage Integration** | Push/fetch Git objects through EthStorage packfile hashes. |
| **Phase 3 – Git Helper Integration** | Release `git-remote-eths` plugin to enable `git push` / `git clone` directly. |
| **Phase 4 – Governance and Permissions** | Multi-sig / DAO-controlled writer sets, repo ownership transfer, and organization namespaces. |
---

## 7. Outlook

This design creates a **verifiable, permanent, and open developer infrastructure**:
- Every commit is cryptographically anchored to Ethereum.  
- Every byte of code is permanently stored and retrievable from EthStorage.  
- No single party controls code distribution or versioning.  

Over time, this model can extend beyond code to packages, data models, or even front-end assets — forming the foundation of **a decentralized, verifiable developer stack**.
