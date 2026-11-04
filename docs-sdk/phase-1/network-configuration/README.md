# Network Configuration

**Phase:** 1 - Foundations  
**Status:** ✅ Done ([`0.0.2-dev`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/CHANGELOG.md#002-dev))  
**Files:** [`vechain_network.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/network/vechain_network.dart) | [`vechain_network_exception.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/exceptions/vechain_network_exception.dart)

## Summary

`VeChainNetwork` is the first building block of the SDK: an immutable
definition of a VeChainThor network, made of its name, its genesis
block ID, the node URL used to reach it, and its chain tag. Every
transaction the SDK will sign in later phases embeds this chain tag,
so the network definition has to be correct by construction.

The three official networks (mainnet, testnet, and solo) are
predefined as `const` values. Any other network, such as a private
node, is created with `VeChainNetwork.custom`, which validates and
normalizes every input.

```dart
const network = VeChainNetwork.testnet;
print(network.chainTag); // 39 (0x27)
print(network.nodeUrl); // https://testnet.vechain.org
```

## Design Decision: Derive the Chain Tag from the Genesis ID

The official transaction model defines the chain tag as the last byte
of the genesis block ID. Instead of storing the chain tag as a separate
field and validating that it matches, the SDK computes it from the
genesis ID through a getter.

```dart
int get chainTag =>
    int.parse(genesisId.substring(genesisId.length - 2), radix: 16);
```

**Why:** a stored chain tag can disagree with its genesis ID; a derived
one cannot. This removes an entire class of configuration errors (a
network that signs transactions for the wrong chain) instead of
detecting it at runtime.

**Source:** [VeChain Docs: Transaction Model](https://docs.vechain.org/thor/learn/transaction-model)

## Design Decision: Node URL as a String

`nodeUrl` is stored as a `String`, with a `nodeUri` getter that parses
it on demand.

**Why:** Dart's `Uri` cannot be created in a `const` expression. The
official networks need to be `const` so they can be used as default
parameter values (for example, a client that defaults to testnet).
URLs passed to `VeChainNetwork.custom` are still fully validated: they
must be absolute `http` or `https` URLs with a host.

**Source:** [Dart: Final and const](https://dart.dev/language/variables#final-and-const)

## Design Decision: Official Node URLs, and 127.0.0.1 for Solo

The node URLs and genesis IDs are copied from the constants of the
official VeChain SDK. For the solo network, the SDK uses
`http://127.0.0.1:8669` instead of `http://localhost:8669`.

**Why:** the official SDK documents this choice explicitly: the
explicit IPv4 address avoids IPv6 resolution issues in CI. Following
it keeps local development and CI behavior identical to the reference
implementation.

**Sources:**
[@vechain/sdk-network: network URLs](https://app.unpkg.com/@vechain/sdk-network@2.0.7/files/src/utils/const/network/network.ts) |
[@vechain/sdk-core: network constants](https://app.unpkg.com/@vechain/sdk-core@2.0.4/files/src/utils/const/network.ts)

## Design Decision: Normalize Custom Inputs

`VeChainNetwork.custom` trims surrounding whitespace from every input,
lowercases the genesis ID, and removes trailing slashes from the node
URL.

**Why:** two definitions of the same network should be equal no matter
how they were typed. A custom network built with the official mainnet
values and a trailing slash in its URL is equal to
`VeChainNetwork.mainnet`. Removing trailing slashes also lets the Thor
REST client (Phase 2) append request paths consistently.

## Design Decision: A Dedicated VeChainNetworkException

Invalid network definitions throw `VeChainNetworkException` instead of
Dart's generic `ArgumentError`.

**Why:** calling code can catch configuration problems specifically
and tell them apart from other SDK failures. The message always quotes
the invalid input and states the expected format.

## Design Decision: Leave the EVM Chain ID Out

The official SDK also defines an EVM chain ID (`0x186a9` on mainnet,
`0x186aa` on testnet), which is different from the chain tag.

**Why:** native VeChainThor transactions use the chain tag. The EVM
chain ID is only used by Ethereum-compatible tooling, so it will be
added when the SDK reaches contracts and tokens.

**Source:** [@vechain/sdk-network: chain ID and chain tag](https://app.unpkg.com/@vechain/sdk-network@2.0.0/files/src/provider/utils/const/chain-id/chain-id.ts)

## What 0.0.2-dev Tests Actually Check

1. Mainnet matches the official name, genesis ID, chain tag (`0x4a`), and node URL.
2. Testnet matches the official name, genesis ID, chain tag (`0x27`), and node URL.
3. Solo matches the official name, genesis ID, chain tag (`0xf6`), and node URL.
4. For every official network, the chain tag equals the last byte of its genesis ID.
5. Official networks work as `const` default parameter values.
6. `nodeUri` parses the node URL into the right scheme and host.
7. A custom network is created and its chain tag is derived correctly.
8. A custom network accepts `http` URLs with a port.
9. The genesis ID is normalized to lowercase.
10. Surrounding whitespace is trimmed from every input.
11. Single and multiple trailing slashes are removed from the node URL.
12. A custom network built with official values equals the official network, including its hash code.
13. Networks with different node URLs are not equal.
14. An empty name throws.
15. A whitespace-only name throws.
16. A genesis ID without the `0x` prefix throws.
17. A genesis ID that is too short throws, and the message quotes it.
18. A genesis ID that is too long throws.
19. A genesis ID with non-hex characters throws.
20. A node URL without a scheme throws, and the message quotes it.
21. A node URL with a non-http scheme (`ws://`) throws.
22. A node URL without a host throws.
23. An empty node URL throws.
24. `toString` shows the name, the chain tag in hex, and the node URL.
25. `VeChainNetworkException` exposes its message.
26. `VeChainNetworkException.toString` includes the type name and message.
27. `VeChainNetworkException` is an `Exception`.

See [`test/src/network/vechain_network_test.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/test/src/network/vechain_network_test.dart)
and [`test/src/exceptions/vechain_network_exception_test.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/test/src/exceptions/vechain_network_exception_test.dart).

## Out of Scope for 0.0.2-dev

- **Connecting to a node:** `VeChainNetwork` only describes where a
  node is; the HTTP client that talks to it arrives in Phase 2
  (`0.1.1-dev`).
- **Detecting a network from its node:** reading the genesis block
  from a node to confirm it matches the expected network needs the
  Thor REST client (Phase 2).
- **EVM chain ID:** deferred to Phase 5 (contracts and tokens).
- **Web platform support:** to be evaluated at the Phase 1 closing
  audit (`0.1.0-dev`), once the hashing and signing vectors can be run
  in a browser.

## Related

- Next: Hashing (`0.0.3-dev`)
- [CHANGELOG entry for 0.0.2-dev](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/CHANGELOG.md#002-dev)
- [Module 01: VeChainThor Foundations](https://github.com/nemorixgroup/VeChain-Knowledge-Base/blob/main/module-01-foundations/README.md) - networks and chain tags explained from zero

## Sources

- [VeChain Docs: Transaction Model](https://docs.vechain.org/thor/learn/transaction-model)
- [VeChain Academy: Connect to the Network](https://academy.vechain.org/academy/connect-to-the-network)
- [@vechain/sdk-network: network URLs](https://app.unpkg.com/@vechain/sdk-network@2.0.7/files/src/utils/const/network/network.ts)
- [@vechain/sdk-network: chain ID and chain tag](https://app.unpkg.com/@vechain/sdk-network@2.0.0/files/src/provider/utils/const/chain-id/chain-id.ts)
- [@vechain/sdk-core: network constants](https://app.unpkg.com/@vechain/sdk-core@2.0.4/files/src/utils/const/network.ts)
- [Dart: Final and const](https://dart.dev/language/variables#final-and-const)
