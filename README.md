# VeChain Knowledge Base

A structured, beginner-friendly technical knowledge base for the VeChainThor blockchain, built by [Nemorix Group](https://github.com/nemorixgroup) as the companion documentation for [vechain_flutter_sdk](https://github.com/nemorixgroup/vechain-flutter-sdk), a native Flutter/Dart SDK for VeChain.

Every module starts from zero: what a concept is, why it matters, and only then the technical details. All technical content is sourced exclusively from official VeChain and VeBetterDAO references.

## Why This Repository

VeChainThor has features that set it apart from other EVM-compatible chains: a dual-token model (VET and VTHO), multi-clause transactions, native fee delegation, and, since the Galactica hard fork, a dynamic fee market with typed transactions. This knowledge base explains those features in depth so that developers, especially mobile developers, can build on VeChain with confidence.

## Modules

| # | Module | Topics | Status |
|---|---|---|---|
| 01 | [Foundations](module-01-foundations/README.md) | VeChainThor, networks, dual-token model (VET/VTHO) | ✅ Done |
| 02 | [Accounts and Keys](module-02-accounts-and-keys/README.md) | Keypairs, addresses, mnemonics, HD derivation, keystore | 🔄 In progress |
| 03 | [Thor REST API](module-03-thor-rest-api/README.md) | Nodes, accounts, blocks, receipts, fee endpoints | ⏳ Pending |
| 04 | [Transactions](module-04-transactions/README.md) | Transaction model, clauses, intrinsic gas, legacy and dynamic fee types | ⏳ Pending |
| 05 | [Fee Delegation](module-05-fee-delegation/README.md) | VIP-191 designated gas payer, Multi-Party Payment | ⏳ Pending |
| 06 | [Contracts and Tokens](module-06-contracts-and-tokens/README.md) | ABI, contract calls, ERC-20 tokens, events and logs | ⏳ Pending |
| 07 | [VeBetterDAO](module-07-vebetterdao/README.md) | B3TR, VOT3, X-to-Earn apps, rewards distribution | ⏳ Pending |
| 08 | [Certificates](module-08-certificates/README.md) | VIP-192 signed certificates and authentication | ⏳ Pending |

## SDK Technical Decisions

The [docs-sdk/](docs-sdk/README.md) folder documents the design decisions behind each release of vechain_flutter_sdk, one write-up per feature, each citing the official source that justifies it.

| Phase | Scope | Versions | Status |
|---|---|---|---|
| 1 | Foundations | 0.1.x | ⏳ Pending |
| 2 | Thor REST client | 0.2.x | ⏳ Pending |
| 3 | Transactions | 0.3.x | ⏳ Pending |
| 4 | Fee delegation | 0.4.x | ⏳ Pending |
| 5 | Contracts and tokens | 0.5.x | ⏳ Pending |
| 6 | VeBetter integration | 0.6.x | ⏳ Pending |
| 7 | Certificates and demo app | 0.7.x | ⏳ Pending |

## Official References

- [VeChain Documentation](https://docs.vechain.org)
- [VeChain Academy](https://academy.vechain.org)
- [VeBetterDAO Documentation](https://docs.vebetterdao.org)
- [VeChainThor Node (GitHub)](https://github.com/vechain/thor)
- [VeChain Improvement Proposals (VIPs)](https://github.com/vechain/VIPs)
- [VeChain SDK for JavaScript](https://github.com/vechain/vechain-sdk-js)

## About This Guide
 
This knowledge base is built from official VeChain and VeBetterDAO documentation, technical references published at vechain.org, and verified on-chain data. No speculative content or price predictions are included.
 
The goal is to provide a clear, honest, and technically accurate resource for understanding VeChain from the inside out.

## Language

The modules are written in English. A Spanish edition for the Latin American developer community is planned.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

## Support This Project

If this project is useful to you or your team, consider supporting its development. Every contribution helps cover infrastructure, documentation, and the time invested in building and maintaining this open source resource for the VeChain community. Thank you!

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-Support-FFDD00?logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/nemorixgroupllc)
[![Sponsor](https://img.shields.io/badge/Sponsor-GitHub-EA4AAA?logo=github-sponsors&logoColor=white)](https://github.com/sponsors/nemorixgroup)
[![Ko-fi](https://img.shields.io/badge/Ko--fi-Support-FF5E5B?logo=ko-fi&logoColor=white)](https://ko-fi.com/nemorixgroupllc) 

---

*Last updated*: **November 2026**  
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixgroup)*
