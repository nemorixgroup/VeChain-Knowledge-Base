# Hashing and Hex Encoding

**Phase:** 1 - Foundations  
**Status:** ✅ Done ([`0.0.3-dev`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/CHANGELOG.md#003-dev))  
**Files:** [`vechain_hash.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/crypto/vechain_hash.dart) | [`vechain_hex.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/codec/vechain_hex.dart) | [`vechain_codec_exception.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/exceptions/vechain_codec_exception.dart)

## Summary

VeChainThor uses exactly two hash functions, both with a 32-byte
output: BLAKE2b-256 and Keccak-256. `VeChainHash` implements both,
each in a single-input form and a multi-part form that hashes several
inputs as if they were concatenated.

`VeChainHex` converts bytes to and from the `0x`-prefixed hex strings
that VeChainThor uses for hashes, addresses, signatures, and raw
transactions. Every later feature of the SDK (addresses, signatures,
transaction IDs, fee delegation) is built on these two pieces.

```dart
final data = Uint8List.fromList(utf8.encode('VeChainThor'));
final id = VeChainHash.blake2b256(data); // 32 bytes
print(VeChainHex.encode(id)); // 0x... (64 lowercase hex digits)
```

## Design Decision: Two Hash Functions, Each with Its Own Role

The VeChainThor node defines exactly two hash functions in
`thor/hash.go`, and uses each for different purposes:

| Function | Node implementation | Used for |
|---|---|---|
| BLAKE2b-256 | `blake2b.New256` / `blake2b.Sum256` | Transaction signing hash, transaction ID, block ID, fee delegator hash |
| Keccak-256 | `keccak.NewLegacyKeccak256()` | Address derivation from a public key, ABI encoding |

**Why:** the SDK has to produce exactly the same bytes as the node,
so it implements exactly the functions the node defines, with the same
output length.

**Source:** [vechain/thor: thor/hash.go](https://github.com/vechain/thor/blob/master/thor/hash.go)

## Design Decision: Legacy Keccak-256, Not SHA3-256

`VeChainHash.keccak256` uses pointycastle's `KeccakDigest(256)`, never
`SHA3Digest`.

**Why:** SHA3-256 is the standardized version of Keccak, but with a
different padding byte, so it produces completely different hashes.
The node uses the original ("legacy") Keccak, as Ethereum does. For an
empty input:

- Keccak-256: `0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470`
- SHA3-256: `0xa7ffc6f8bf1ed76651c14756a061d662f580ff4de43b49fa82d80a4b80f8434a`

Using SHA3-256 by mistake would derive wrong addresses for every key.
A dedicated test guards against this.

**Sources:**
[vechain/thor: thor/hash.go](https://github.com/vechain/thor/blob/master/thor/hash.go) |
[bcgit/pc-dart: KeccakDigest](https://github.com/bcgit/pc-dart/blob/master/lib/digests/keccak.dart)

## Design Decision: An Explicit 32-Byte BLAKE2b Digest

`VeChainHash.blake2b256` creates `Blake2bDigest(digestSize: 32)`.

**Why:** pointycastle's `Blake2bDigest` defaults to a 64-byte digest
(BLAKE2b-512). Without the explicit size, the output would not match
the node's `blake2b.New256`.

**Source:** [bcgit/pc-dart: Blake2bDigest](https://github.com/bcgit/pc-dart/blob/master/lib/digests/blake2b.dart)

## Design Decision: Multi-Part Hashing

Both functions also come in a `Concat` form that takes a list of parts:

```dart
VeChainHash.blake2b256Concat([signingHash, origin]);
```

**Why:** the node's hash functions accept several inputs
(`Blake2b(data ...[]byte)`), and it relies on that to build values the
SDK will need in later phases. For example, the transaction ID, which
is also the hash a fee delegator signs under VIP-191, is
`blake2b256(signingHash + origin)`:

```go
// According to VIP-191, it's identical to tx id.
func (t *Transaction) DelegatorSigningHash(origin thor.Address) thor.Bytes32 {
    return thor.Blake2b(t.SigningHash().Bytes(), origin[:])
}
```

The parts are fed to the digest one by one, so no intermediate
concatenated copy is created.

**Sources:**
[vechain/thor: thor/hash.go](https://github.com/vechain/thor/blob/master/thor/hash.go) |
[vechain/thor: tx/transaction.go](https://github.com/vechain/thor/blob/master/tx/transaction.go)

## Design Decision: A Strict Hex Codec

`VeChainHex` follows the official SDK's `Hex` type for the common
rules: `encode` returns `0x` plus lowercase digits, and `decode`
accepts an optional `0x` prefix and digits in any case, rejecting any
non-hex character (the official tests use `0xfoe` as the invalid
example).

It deliberately differs in one rule: **an odd number of digits is
rejected** with `VeChainCodecException`, where the official `Hex` pads
with a leading `0`.

**Why:** the official `Hex` type also represents numbers, where `0xabc`
and `0x0abc` are the same value. `VeChainHex` represents bytes (hashes,
addresses, signatures), where a missing digit is almost always a copy
error. Padding it silently would turn, for example, a mistyped address
into a different valid-looking address. Numeric values (amounts, gas)
will get their own type with numeric rules in later phases.

**Source:** [vechain-sdk-js: Hex.ts](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/src/vcdm/Hex.ts)

## Test Vectors

The official vectors come from the VeChain SDK for JavaScript, which
hashes the UTF-8 text `'Hello world - Здравствуйте - こんにちは!'` and
an empty input:

| Input | BLAKE2b-256 | Keccak-256 |
|---|---|---|
| UTF-8 text | `0x6a908bb80109908919c0bf5d0594c890700dd46acc097f9f28bfc85a0a2e6c0c` | `0x1e86a83a4fcab1b47b8c961f7ab6c5d32927eefa8af20af81f6eab0bc3be582a` |
| Empty | `0x0e5751c026e543b2e8ab2eb06099daa1d1e5df47778f7787faab45cdf12fe3a8` | `0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470` |

All four values were also recomputed with independent implementations
(Python `hashlib` and `pycryptodome`) before being used as test
vectors.

**Source:** [vechain-sdk-js: hash tests](https://github.com/vechain/vechain-sdk-js/tree/main/packages/core/tests/vcdm/hash)

## What 0.0.3-dev Tests Actually Check

1. BLAKE2b-256 matches the official vector for the UTF-8 text.
2. BLAKE2b-256 matches the official vector for an empty input.
3. BLAKE2b-256 always returns 32 bytes, including at the 128-byte block size boundary.
4. BLAKE2b-256 is deterministic and sensitive to a one-character change.
5. `blake2b256Concat` over several parts equals `blake2b256` over the joined bytes.
6. `blake2b256Concat` with a single part equals `blake2b256`.
7. `blake2b256Concat` with an empty list equals the hash of an empty input.
8. Keccak-256 matches the official vector for the UTF-8 text.
9. Keccak-256 matches the official vector for an empty input.
10. Keccak-256 is legacy Keccak, not SHA3-256.
11. Keccak-256 always returns 32 bytes, including at the 136-byte block size boundary.
12. Keccak-256 is deterministic and sensitive to a one-character change.
13. `keccak256Concat` over several parts equals `keccak256` over the joined bytes.
14. `keccak256Concat` with a single part equals `keccak256`.
15. `keccak256Concat` with an empty list equals the hash of an empty input.
16. BLAKE2b-256 and Keccak-256 give different hashes for the same input.
17. `encode` of an empty list returns `0x`.
18. `encode` pads every byte to two digits.
19. `encode` always uses lowercase digits.
20. `decode` accepts a `0x`-prefixed string.
21. `decode` accepts a string without prefix.
22. `decode` accepts an uppercase `0X` prefix.
23. `decode` accepts digits in any case.
24. `decode` returns an empty list for `0x` and for the empty string.
25. `decode` and `encode` round-trip an official hash vector.
26. `decode` rejects a non-hex character (official example `0xfoe`).
27. `decode` rejects an odd number of digits instead of padding.
28. `decode` rejects surrounding whitespace.
29. `decode` rejects a doubled prefix.
30. `decode` rejects a negative sign.
31. `VeChainCodecException` exposes its message.
32. `VeChainCodecException.toString` includes the type name and message.
33. `VeChainCodecException` is an `Exception`.

See [`test/src/crypto/vechain_hash_test.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/test/src/crypto/vechain_hash_test.dart),
[`test/src/codec/vechain_hex_test.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/test/src/codec/vechain_hex_test.dart)
and [`test/src/exceptions/vechain_codec_exception_test.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/test/src/exceptions/vechain_codec_exception_test.dart).

## Out of Scope for 0.0.3-dev

- **Address derivation and checksum:** uses `keccak256`, arrives in
  `0.0.4-dev`.
- **Transaction IDs and the VIP-191 delegator hash:** use
  `blake2b256Concat`, arrive with transactions (Phase 3) and fee
  delegation (Phase 4).
- **Hex for numeric values:** amounts and gas will get their own
  numeric type in later phases.
- **SHA-256:** the official SDK exposes it, but the protocol features
  covered by this SDK do not use it so far; it will be added only if
  a later feature needs it.
- **Web platform support:** to be evaluated at the Phase 1 closing
  audit (`0.1.0-dev`).

## Related

- Previous: [Network Configuration](https://github.com/nemorixgroup/VeChain-Knowledge-Base/blob/main/docs-sdk/phase-1/network-configuration/README.md) (`0.0.2-dev`)
- Next: Keypair and Address (`0.0.4-dev`)
- [CHANGELOG entry for 0.0.3-dev](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/CHANGELOG.md#003-dev)

## Sources

- [vechain/thor: thor/hash.go](https://github.com/vechain/thor/blob/master/thor/hash.go)
- [vechain/thor: tx/transaction.go](https://github.com/vechain/thor/blob/master/tx/transaction.go)
- [vechain-sdk-js: Blake2b256.ts](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/src/vcdm/hash/Blake2b256.ts)
- [vechain-sdk-js: Keccak256.ts](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/src/vcdm/hash/Keccak256.ts)
- [vechain-sdk-js: Hex.ts](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/src/vcdm/Hex.ts)
- [vechain-sdk-js: hash tests](https://github.com/vechain/vechain-sdk-js/tree/main/packages/core/tests/vcdm/hash)
- [bcgit/pc-dart: Blake2bDigest](https://github.com/bcgit/pc-dart/blob/master/lib/digests/blake2b.dart)
- [bcgit/pc-dart: KeccakDigest](https://github.com/bcgit/pc-dart/blob/master/lib/digests/keccak.dart)
