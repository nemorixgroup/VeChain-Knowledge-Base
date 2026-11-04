# SDK Technical Decisions

This section documents the implementation decisions, technical
rationale, and key references behind **vechain_flutter_sdk**, a native
Flutter/Dart SDK for the VeChainThor blockchain.

Every decision documented here is grounded in official VeChain sources
(docs.vechain.org, VeChain Academy, the official VeChain SDK for
JavaScript, and the VIPs). Where relevant, implementation code and test
vectors are included to make each decision fully verifiable.

## Why This Documentation Exists

Building a production-quality SDK for VeChainThor requires making
decisions that are not always obvious: which cryptographic library to
use, how a value is derived, how an algorithm was verified against the
official specification.

This documentation answers those questions for:

- **Contributors** who want to understand the rationale before
  submitting a pull request
- **Developers** who want to verify that the SDK follows official
  VeChain standards
- **Auditors** who need a clear trail from specification to
  implementation

## Versioning

Each phase is built in small `-dev` sub-versions and closes with a
consolidation release: Phase 1 runs from `0.0.1-dev` and closes at
`0.1.0-dev`; Phase 2 starts at `0.1.1-dev` and closes at `0.2.0-dev`,
and so on.

## Structure

```
docs-sdk/
  README.md    <- You are here
    phase-1/   <- Foundations
    phase-2/   <- Thor REST Client
    phase-3/   <- Transactions
    phase-4/   <- Fee Delegation
    phase-5/   <- Contracts and Tokens
    phase-6/   <- VeBetter Integration
    phase-7/   <- Certificates and Demo App
```


## Phase 1 - Foundations

| Feature | Description | Version | Status |
|---------|-------------|---------|--------|
| [Network Configuration](phase-1/network-configuration/README.md) | Official networks, custom networks, chain tag derived from the genesis ID | `0.0.2-dev` | ✅ Done |
| Hashing | `keccak256` and `blake2b256` | `0.0.3-dev` | 🔄 Next |
| Keypair and Address | secp256k1 keys, address derivation, checksum | `0.0.4-dev` | ⏳ Pending |
| Signatures | Signing and public key recovery | `0.0.5-dev` | ⏳ Pending |
| Mnemonic | BIP-39 mnemonic generation and validation | `0.0.6-dev` | ⏳ Pending |
| HD Wallet | Derivation at `m/44'/818'/0'/0` | `0.0.7-dev` | ⏳ Pending |
| Keystore | Password-encrypted private keys | `0.0.8-dev` | ⏳ Pending |
| Phase 1 Closing Audit | Edge cases, error handling and docs review | `0.1.0-dev` | ⏳ Pending |

## Phase 2 - Thor REST Client

| Feature | Description | Status |
|---------|-------------|--------|
| HTTP Client | Base Thor client, error model, node status | ⏳ Pending |
| Accounts | VET and VTHO balances, code and storage | ⏳ Pending |
| Blocks and Transactions | Blocks, transactions and receipts | ⏳ Pending |
| Fees | `/fees/history` and `/fees/priority` | ⏳ Pending |
| Simulation | Read-only clause execution | ⏳ Pending |
| Phase 2 Closing Audit | Error handling review, test consolidation | ⏳ Pending |

## Phase 3 - Transactions

| Feature | Description | Status |
|---------|-------------|--------|
| Clauses | Clause model, VET and VTHO transfers | ⏳ Pending |
| Intrinsic Gas | Official calculation and estimation | ⏳ Pending |
| Legacy Encoding | RLP, signing hash, signature, transaction ID | ⏳ Pending |
| Dynamic Fee | Typed transactions after Galactica (`0x51`) | ⏳ Pending |
| Submission | Send, receipt polling, multi-clause, dependsOn | ⏳ Pending |
| Phase 3 Closing Audit | Error handling review, test consolidation | ⏳ Pending |

## Phase 4 - Fee Delegation

| Feature | Description | Status |
|---------|-------------|--------|
| Delegation Flag and Hash | VIP-191 features flag, gas payer signing hash | ⏳ Pending |
| Local Gas Payer | Dual signature with a sponsor key | ⏳ Pending |
| Remote Delegator | Delegation service client | ⏳ Pending |
| Phase 4 Closing Audit | Error handling review, test consolidation | ⏳ Pending |

## Phase 5 - Contracts and Tokens

| Feature | Description | Status |
|---------|-------------|--------|
| ABI | Function and event encoding and decoding | ⏳ Pending |
| Contracts | Read calls and contract clauses | ⏳ Pending |
| ERC-20 Tokens | Balance, transfer, approve, allowance; VTHO | ⏳ Pending |
| Events and Logs | Event and transfer filters | ⏳ Pending |
| Phase 5 Closing Audit | Error handling review, test consolidation | ⏳ Pending |

## Phase 6 - VeBetter Integration

| Feature | Description | Status |
|---------|-------------|--------|
| B3TR and VOT3 | Token balances and metadata | ⏳ Pending |
| X2EarnApps | Registered apps and their information | ⏳ Pending |
| Rewards Pool | App funds and `distributeRewardWithProof` | ⏳ Pending |
| VeBetterPassport | User verification | ⏳ Pending |
| Phase 6 Closing Audit | Error handling review, test consolidation | ⏳ Pending |

## Phase 7 - Certificates and Demo App

| Feature | Description | Status |
|---------|-------------|--------|
| Certificates | VIP-192 signing and verification | ⏳ Pending |
| Demo App | Flutter X2Earn app with fee delegation | ⏳ Pending |
| Phase 7 Closing Audit | Final review before `1.0.0` | ⏳ Pending |

## Related

- [vechain_flutter_sdk](https://github.com/nemorixgroup/vechain-flutter-sdk) - the SDK itself
- [vechain_flutter_sdk on pub.dev](https://pub.dev/packages/vechain_flutter_sdk)
- [VeChain Knowledge Base](https://github.com/nemorixgroup/VeChain-Knowledge-Base/blob/main/README.md) - the main knowledge base

---

*Last updated*: **November 2026**  
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixgroup)*
