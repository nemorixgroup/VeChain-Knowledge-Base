# Keypair, Address and Checksum

**Phase:** 1 - Foundations  
**Status:** ✅ Done ([`0.0.4-dev`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/CHANGELOG.md#004-dev))  
**Files:** [`vechain_keypair.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/crypto/vechain_keypair.dart) | [`vechain_address.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/address/vechain_address.dart) | [`vechain_secp256k1.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/crypto/vechain_secp256k1.dart) | [`vechain_crypto_exception.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/exceptions/vechain_crypto_exception.dart) | [`vechain_address_exception.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/exceptions/vechain_address_exception.dart)

## Summary

Every VeChainThor account is controlled by a secp256k1 private key.
From it come the public key, used to verify signatures, and the
address, where the account receives funds. This release adds:

- `VeChainKeyPair`: generates or restores a key pair, validated once at
  creation.
- `VeChainAddress`: derives, parses, and prints addresses with the
  EIP-55 checksum.
- `VeChainSecp256k1`: the internal curve operations both share (not
  exported).

```dart
final keyPair = VeChainKeyPair.generate();
print(keyPair.address); // 0x... (EIP-55 checksum)
```

## How an Address Is Derived

VeChainThor derives addresses exactly like Ethereum. Using the official
test vector:

1. **Private key:** 32 bytes, valid only as an integer in [1, n - 1],
   where n is the secp256k1 order.
   `0x5434c159b817c377a55f6be66369622976014e78bce2adfd3e44e5de88ce502f`
2. **Public key:** the curve point `privateKey × G`, uncompressed as
   `0x04` + X + Y (65 bytes).
   `0x04a6711e14234b1d...e6eef65607`
3. **Hash:** Keccak-256 of the public key **without** its `0x04` prefix
   (64 bytes).
4. **Address:** the last 20 bytes of that hash, shown with the EIP-55
   checksum.
   `0x769E8AA372c8309c834EA6749B88861FF73581FF`

**Source:** [vechain-sdk-js: Address.ts](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/src/vcdm/Address.ts)

## Design Decision: Enforce the EIP-55 Checksum

EIP-55 encodes a checksum in the letter case of the address: the
Keccak-256 hash of the lowercase hex digits (as ASCII text) decides
which letters are uppercase. `VeChainAddress.parse` applies it like
this:

- All-lowercase or all-uppercase: no checksum is carried, accepted.
- Mixed case: must match the checksum exactly, or
  `VeChainAddressException` is thrown.

**Why:** the official SDK accepts any mixed case without checking it.
Its own tests accept `0xCfb79a9c950b78E14c43efa621ebcf9660dbe01F` as a
"valid checksum format", but the real EIP-55 checksum of that address
is `0xcfB79A9C950B78E14c43Efa621EBCf9660dbe01F`. Enforcing EIP-55 is the
point of the checksum: in a payments SDK, an address with one mistyped
character is caught before any funds are sent to it.

The error message deliberately does not show the "expected" checksum:
if a character was mistyped, it would be the checksum of the wrong
address, and showing it would invite copying it.

**Sources:**
[EIP-55: Mixed-case checksum address encoding](https://eips.ethereum.org/EIPS/eip-55) |
[vechain-sdk-js: Address tests](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/tests/vcdm/Address.unit.test.ts)

## Design Decision: Reject Short Addresses

`VeChainAddress.parse` requires exactly 40 hex digits.

**Why:** the official `Address.of` pads short input with leading zeros
(`0xcaffee` becomes a 20-byte address), because its address type also
behaves as a number. For an address, a missing digit is a copy error,
and padding turns it into a different, valid-looking address that
could receive funds. This follows the same rule as the strict hex codec
of `0.0.3-dev`.

## Design Decision: Validate Public Keys Explicitly

`VeChainSecp256k1.decodePublicKey` accepts only the two encodings
VeChain uses, checks the curve equation `y^2 = x^3 + 7 (mod p)`, and
decompresses points itself:

- `0x04` + X + Y (65 bytes): the point must satisfy the equation.
- `0x02` or `0x03` + X (33 bytes): Y is recovered as
  `(x^3 + 7)^((p + 1) / 4) mod p`, which works because
  `p = 3 (mod 4)`. If Y squared does not give back `x^3 + 7`, that X
  has no point on the curve. The prefix picks the root: `0x02` for an
  even Y, `0x03` for an odd Y.

**Why:** pointycastle's `decodePoint` does not check that an
uncompressed point is on the curve, accepts encodings VeChain never
uses (`0x00`, `0x06`, `0x07`), and signals an invalid compressed point
by throwing an `Error`, which the SDK's lint rules forbid catching.
Decoding explicitly removes all three problems.

**Sources:**
[vechain-sdk-js: Secp256k1.ts](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/src/secp256k1/Secp256k1.ts) |
[bcgit/pc-dart: ecc_base.dart](https://github.com/bcgit/pc-dart/blob/master/lib/ecc/ecc_base.dart)

## Design Decision: Secure Key Generation by Rejection

`VeChainKeyPair.generate` draws 32 bytes from `Random.secure()` and
draws again whenever the value falls outside [1, n - 1].

**Why:** `Random.secure()` is Dart's cryptographically secure source and
works the same on every platform. Rejecting out-of-range values, instead
of reducing them modulo n, keeps every valid key equally likely. The
official SDK applies the same range rule through `isValidPrivateKey`.

**Source:** [vechain-sdk-js: Secp256k1.ts](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/src/secp256k1/Secp256k1.ts)

## Design Decision: One Curve Multiplication per Key Pair

`VeChainKeyPair.fromPrivateKey` derives the uncompressed public key once
and builds the compressed form from it: the prefix `0x02` or `0x03`
from the parity of Y, followed by X.

**Why:** the point multiplication is the expensive step. Deriving both
formats separately would do it twice. This is the same method the
official SDK uses in `compressPublicKey`, and a test compares the result
against pointycastle's own compression on a fresh random key in every
run.

## Design Decision: Limit Private Key Exposure

- Getters return copies, so callers cannot modify the stored keys.
- The input is copied, so changing the caller's list later has no
  effect.
- `toString` shows only the address.
- Error messages never include key material.

**Why:** the official SDK tries to wipe private data from memory after
use. Dart cannot do that reliably, because the garbage collector may
copy or move objects in memory without notice. Instead of promising an
erasure it cannot guarantee, the SDK limits where the private key can
leak.

## Design Decision: An Internal Curve Class

Curve operations live in `VeChainSecp256k1`, shared by
`VeChainKeyPair` and `VeChainAddress.fromPublicKey`, and not exported
in the public API.

**Why:** both need the same operations. One internal class keeps a
single strict implementation, and keeps the public API limited to the
types an app actually needs.

## Test Vectors

| Source | Data |
|---|---|
| vechain-sdk-js (Address tests) | Private key `0x5434c1...502f`, its public key, and address `0x769E8AA372c8309c834EA6749B88861FF73581FF` |
| vechain-sdk-js (Secp256k1 tests) | Private key `0x7582be...e26a` with its compressed and uncompressed public keys |
| vechain-sdk-js | Lowercase, uppercase, and wrong-checksum forms of `0xcfb79a...e01f`; invalid inputs `0xcaffee`, `-0xcaffee` and `[1, 2, 3, 4, 5]` |
| EIP-55 specification | Four checksummed addresses |
| Independently computed | Compressed form of the first key, the address of the second key, the generator point G (private key 1) and its address, and X = 5 (no point on the curve) |

The independently computed values were obtained with a separate
implementation of the secp256k1 arithmetic written from scratch, and
every official vector was recomputed the same way.

## What 0.0.4-dev Tests Actually Check

**Curve operations (`VeChainSecp256k1`)**

1. The official private keys are valid.
2. The range limits 1 and n - 1 are valid.
3. Zero is rejected.
4. The curve order n is rejected.
5. The official invalid key (all `0xff`) is rejected.
6. Keys that are not 32 bytes are rejected.
7. The official uncompressed public keys are derived exactly.
8. The official compressed public key is derived exactly.
9. Private key 1 derives the generator point G.
10. Deriving from an invalid private key throws.
11. A valid uncompressed key is returned unchanged.
12. Compressed keys with an odd Y (`0x03`) inflate to the official uncompressed keys.
13. A compressed key with an even Y (`0x02`) inflates to G.
14. An empty key throws.
15. The official invalid example `[1, 2, 3, 4, 5]` throws.
16. A prefix that does not match the length throws.
17. A hybrid encoding (`0x06`) throws.
18. An uncompressed point off the curve throws.
19. A compressed X with no point (x = 5) throws.
20. A compressed X outside the field throws.
21. `bigIntToBytes` pads with leading zeros.
22. `bytesToBigInt` and `bigIntToBytes` round-trip.

**Addresses (`VeChainAddress`)**

23. The official address is derived from an uncompressed public key.
24. A compressed public key gives the same address.
25. The address of the second key is derived correctly.
26. The address of the generator point is derived correctly.
27. The official invalid public key `[1, 2, 3, 4, 5]` throws `VeChainCryptoException`.
28. The official all-lowercase address is accepted.
29. The official all-uppercase address is accepted.
30. A correct EIP-55 checksum is accepted.
31. Every EIP-55 specification vector is accepted and printed back identically.
32. An uppercase `0X` prefix is accepted.
33. The official mixed-case string with a wrong checksum is rejected.
34. A single mistyped character in a checksummed address is rejected.
35. A short address (official `0xcaffee`) is rejected, not padded.
36. A negative sign (official `-0xcaffee`) is rejected.
37. An address without the `0x` prefix is rejected.
38. 41 hex digits are rejected.
39. A non-hex character is rejected.
40. `fromBytes` creates an address from 20 bytes.
41. `fromBytes` rejects 19 and 21 bytes.
42. `fromBytes` copies its input.
43. `bytes` returns a copy.
44. The same address in any accepted form is equal, with the same hash code.
45. Different addresses are not equal.

**Exceptions**

46. `VeChainCryptoException` exposes its message.
47. `VeChainCryptoException.toString` includes the type name and message.
48. `VeChainCryptoException` is an `Exception`.
49. `VeChainAddressException` exposes its message.
50. `VeChainAddressException.toString` includes the type name and message.
51. `VeChainAddressException` is an `Exception`.

**Key pairs (`VeChainKeyPair`)**

52. The official private key derives the official public key and address.
53. The official uncompressed and compressed public keys are derived.
54. An odd Y is compressed with the `0x03` prefix.
55. An even Y is compressed with the `0x02` prefix (private key 1).
56. The compressed public key derives the same address.
57. A zero private key throws.
58. A private key equal to the curve order throws.
59. A private key that is not 32 bytes throws.
60. The error message never contains the private key.
61. The input is copied, so later changes do not affect the key pair.
62. Getters return copies.
63. `toString` shows the address and never the private key.
64. `generate` produces a valid key pair in both public key formats.
65. Two generated key pairs are different.
66. A generated key pair can be restored from its private key.
67. The fast compression matches pointycastle's on a fresh random key.

See the tests under [`test/src/crypto/`](https://github.com/nemorixgroup/vechain-flutter-sdk/tree/main/test/src/crypto),
[`test/src/address/`](https://github.com/nemorixgroup/vechain-flutter-sdk/tree/main/test/src/address)
and [`test/src/exceptions/`](https://github.com/nemorixgroup/vechain-flutter-sdk/tree/main/test/src/exceptions).

## Out of Scope for 0.0.4-dev

- **Signing and public key recovery:** arrive in `0.0.5-dev`, built on
  `VeChainSecp256k1`.
- **Mnemonics and HD derivation:** arrive in `0.0.6-dev` and
  `0.0.7-dev`.
- **Encrypted keystores:** arrive in `0.0.8-dev`.
- **Web platform support:** to be evaluated at the Phase 1 closing
  audit (`0.1.0-dev`).

## Related

- Previous: [Hashing and Hex Encoding](https://github.com/nemorixgroup/VeChain-Knowledge-Base/blob/main/docs-sdk/phase-1/hashing/README.md) (`0.0.3-dev`)
- Next: Signatures (`0.0.5-dev`)
- [CHANGELOG entry for 0.0.4-dev](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/CHANGELOG.md#004-dev)
- Module 02: Accounts and Keys

## Sources

- [vechain-sdk-js: Address.ts](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/src/vcdm/Address.ts)
- [vechain-sdk-js: Secp256k1.ts](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/src/secp256k1/Secp256k1.ts)
- [vechain-sdk-js: Address tests](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/tests/vcdm/Address.unit.test.ts)
- [vechain-sdk-js: Secp256k1 tests](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/tests/secp256k1/Secp256k1.unit.test.ts)
- [EIP-55: Mixed-case checksum address encoding](https://eips.ethereum.org/EIPS/eip-55)
- [bcgit/pc-dart: ecc_base.dart](https://github.com/bcgit/pc-dart/blob/master/lib/ecc/ecc_base.dart)
- [bcgit/pc-dart: secp256k1 curve](https://github.com/bcgit/pc-dart/blob/master/lib/ecc/curves/secp256k1.dart)
