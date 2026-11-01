# Module 01: VeChainThor Foundations

This module introduces the VeChainThor blockchain from zero: what it is, how its consensus works after the Hayabusa upgrade, its dual-token model, and the networks every developer needs to know before writing a single line of code.

## Table of Contents

1. [What Is VeChainThor?](#1-what-is-vechainthor)
2. [Consensus: From PoA 2.0 to Delegated Proof of Stake](#2-consensus-from-poa-20-to-delegated-proof-of-stake)
3. [The Dual-Token Model: VET and VTHO](#3-the-dual-token-model-vet-and-vtho)
4. [VTHO After Hayabusa: Generation and Burn](#4-vtho-after-hayabusa-generation-and-burn)
5. [Networks: Mainnet, Testnet, and Solo](#5-networks-mainnet-testnet-and-solo)
6. [VeChain Renaissance Timeline](#6-vechain-renaissance-timeline)
7. [What This Means for vechain_flutter_sdk](#7-what-this-means-for-vechain_flutter_sdk)
8. [Summary](#summary)
9. [Sources](#sources)

## 1. What Is VeChainThor?

### What It Is

VeChainThor is the public layer 1 blockchain of the VeChain ecosystem. It is EVM-compatible: it uses `0x`-prefixed addresses like Ethereum and runs Solidity smart contracts. On top of that, it adds native features that Ethereum does not have.

### Why It Matters

VeChainThor was designed for enterprises and real-world applications: predictable costs, transactions that bundle several operations, and the ability for a third party to pay gas on behalf of the user. For a mobile app, this translates into an experience much closer to a Web2 app.

### Technical Details

- Native token: **VET** (18 decimals).
- Gas token: **VTHO** (18 decimals, ERC-20 interface).
- Block time: **10 seconds**.
- **Deterministic finality** (VIP-220): a finalized block cannot be reverted, even under large-scale network partitioning.
- SLIP-44 coin type: **818**.
- Block explorer: `explore.vechain.org`.

> **Simple example:** on Ethereum, approving a token and then swapping it takes two transactions. On VeChainThor, both can travel as two *clauses* inside a single transaction: either both succeed or neither does.

## 2. Consensus: From PoA 2.0 to Delegated Proof of Stake

### In Short

Consensus is the mechanism nodes use to agree on which blocks are valid. VeChainThor ran on Proof of Authority (PoA 2.0), with authorized nodes, until December 2025.

### What Changed

The **Hayabusa** hard fork moved the network to **Delegated Proof of Stake (dPoS)**. Any VET holder can now help secure the network without running a node, by delegating stake to a validator.

> **Note:** some pages of the official documentation still describe the consensus as PoA 2.0. Since the Hayabusa mainnet activation on December 2, 2025, the consensus is dPoS.

### How It Works

- **Validators** must stake a minimum of **25 million VET**. Up to 101 can be active; when the set is full, new candidates wait in a first-in, first-out queue.
- **Delegators** stake VET through **StarGate**, which mints an NFT representing that stake. The NFT is delegated to a validator and also acts as a voting instrument for on-chain governance.
- **Block proposal:** a validator's chance of proposing a block depends on its own stake plus the VET delegated to it.
- **Downtime:** a validator that produces no blocks for seven consecutive days is removed by the protocol.

> **Simple example:** think of a validator as a team and delegators as its sponsors. The more sponsorship a team receives, the more chances it gets to play (propose blocks), and the prize is shared between the team and its sponsors.

## 3. The Dual-Token Model: VET and VTHO

### The Concept

VeChainThor separates **value** from **cost of use**:

- **VET** is the value token: it is transferred, staked, and grants governance rights.
- **VTHO** is the "energy": it is spent to pay transaction gas.

### Why Two Tokens

On Ethereum, ETH is both a store of value and the gas token, so when its price rises, so does the cost of using the network. Splitting the two roles aims to keep operating costs more stable and predictable, which is key for businesses and consumer apps.

### Technical Details

- VTHO contract address: `0x0000000000000000000000000000456E65726779`. Fun fact: `456E65726779` is "Energy" in hexadecimal.
- VTHO exposes an ERC-20 interface, so it is queried and transferred like any other token.

## 4. VTHO After Hayabusa: Generation and Burn

### What Changed

Before Hayabusa, any VET automatically generated VTHO just by sitting in a wallet. **Since Hayabusa, VET that is not staked no longer generates VTHO.** VTHO is now issued as block rewards only to active validators and delegators.

### Why It Matters

The goal is to reward those who secure the network and eliminate idle VTHO generation. For developers, this means a new app user will usually **have no VTHO to pay for gas**. This turns fee delegation (Module 05) from a nice-to-have into a near-essential feature for consumer apps.

### Mechanics

- **Issuance** follows a curve derived from the total VET staked.
- **Block reward split:** 30% to the validator and 70% to its delegators, proportional to their effective stake (staked VET multiplied by their NFT tier multiplier). A validator with no delegators keeps 100%.
- **Burn:** since Galactica, the **base fee** of every transaction is fully burned; the **priority fee** goes to the validator that produced the block.
- **Cycles:** delegator rewards accrue over the validator's 7, 15, or 30 day cycle and become claimable at the end of each cycle.

## 5. Networks: Mainnet, Testnet, and Solo

### What a Network Is

A network is an independent instance of the blockchain. Separate networks are used for production, testing, and local development.

### Why It Matters

A transaction signed for testnet **cannot** be executed on mainnet. This is guaranteed by the *chain tag*, the last byte of the genesis block ID, which is included in every transaction to prevent cross-chain replay attacks.

### Network Parameters

| Network | Purpose | Chain Tag | Genesis Block ID |
|---|---|---|---|
| Mainnet | Production, real value | `0x4a` | `0x00000000851caf3cfdb6e899cf5958bfb1ac3413d346d43539627e6be7ec1b4a` |
| Testnet | Public testing, valueless tokens | `0x27` | `0x000000000b2bce3c70bc649a02749e8687721b09ed2e15997f466536b20bb127` |
| Solo | Local development node | `0xf6` | `0x00000000c05a20fbca2bf6ae3affba6af4a74b800b585bf7a4988aba7aea69f6` |

> **Simple example:** the mainnet genesis ID ends in `...4a` and its chain tag is `0x4a`. The testnet genesis ID ends in `...27`, and its chain tag is `0x27`.

- Public testnet node: `https://testnet.vechain.org`.
- Explorers: `explore.vechain.org` (mainnet) and `explore-testnet.vechain.org` (testnet).
- Nodes and the REST API are covered in depth in Module 03.

## 6. VeChain Renaissance Timeline

| Milestone | Mainnet Date | What It Introduced |
|---|---|---|
| StarGate launch | July 1, 2025 | NFT-based staking platform (bootstrapping period) |
| Galactica | July 1, 2025 (block 22,084,200) | Dynamic fee market (VIP-251), typed transactions (VIP-252), Shanghai EVM alignment (VIP-242), `/fees/history` and `/fees/priority` endpoints |
| Hayabusa | December 2, 2025 (block 23,414,400) | dPoS consensus, new VTHO tokenomics |

## 7. What This Means for vechain_flutter_sdk

| Concept | SDK Impact | Phase |
|---|---|---|
| Chain tags and genesis IDs | `VeChainNetwork` with mainnet, testnet, and solo | 1 |
| Coin type 818 | HD derivation at `m/44'/818'/0'/0` | 1 |
| VET and VTHO | Balance queries for both tokens | 2 |
| Dynamic fees | Support for type `0x51` transactions | 3 |
| VTHO only from staking | Fee delegation as a core feature | 4 |

## Summary

- VeChainThor is an EVM-compatible layer 1 with its own native features: multi-clause transactions, fee delegation, and a dual-token model.
- Since Hayabusa, it runs on **Delegated Proof of Stake**; validators need 25 million VET.
- **VET** is the value token and **VTHO** the gas token; VTHO is generated only for staked VET.
- The base fee is burned and the priority fee goes to the validator.
- Each network is identified by its chain tag: `0x4a` (mainnet), `0x27` (testnet), and `0xf6` (solo).

## Sources

- [VeChain Docs: VeChain Blockchain Specifications](https://docs.vechain.org/developer-resources/evm-compatibility-for-developers/vechain-blockchain-specifications)
- [VeChain Docs: Blocks](https://docs.vechain.org/core-concepts/blocks)
- [VeChain Docs: Transaction Calculation](https://docs.vechain.org/thor/learn/transaction-calculation)
- [StarGate Documentation](https://docs.stargate.vechain.org/)
- [StarGate Documentation: Validators](https://docs.stargate.vechain.org/overview/validators)
- [VeChain: The Economics of Staking](https://vechain.org/the-economics-of-staking-how-hayabusa-rewards-real-activity)
- [VeChain SDK Core: Network Constants](https://app.unpkg.com/@vechain/sdk-core@2.0.4/files/src/utils/const/network.ts)
- [VeChainThor v2.3.0 Release (Galactica)](https://newreleases.io/project/github/vechain/thor/release/v2.3.0)

---

*Last updated: October 2026*  
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixgroup)*
