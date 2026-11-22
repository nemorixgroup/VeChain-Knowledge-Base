# Signatures and Public Key Recovery

**Phase:** 1 - Foundations  
**Status:** ✅ Done ([`0.0.5-dev`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/CHANGELOG.md#005-dev))  
**Files:** [`vechain_signature.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/crypto/vechain_signature.dart) | [`vechain_secp256k1.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/crypto/vechain_secp256k1.dart) | [`vechain_keypair.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/lib/src/crypto/vechain_keypair.dart)

## Summary

A VeChainThor transaction does not carry the sender's public key. The
node recovers it from the signature and the signed hash, and from it
derives the sender's address. The same mechanism identifies the gas
payer under fee delegation (VIP-191) and the signer of a certificate
(VIP-192).

This release adds:

- `VeChainKeyPair.sign`: signs a 32-byte hash.
- `VeChainSignature`: a validated 65-byte signature that recovers its
  signer's public key and address.
- `sign` and `recoverPublicKey` in the internal `VeChainSecp256k1`
  class.

```dart
final signature = keyPair.sign(hash);
print(signature.recoverAddress(hash) == keyPair.address); // true
```

## Signature Layout

A VeChainThor signature is 65 bytes:

| Bytes | Field | Meaning |
|---|---|---|
| 0-31 | r | X coordinate of the curve point R, reduced modulo n |
| 32-63 | s | The proof that ties the hash to the private key |
| 64 | recovery | Parity of the Y coordinate of R: `0` or `1` |

The recovery byte is `0` or `1`, as in the official SDK, which rejects
any other value. It is not `27` or `28` as in Ethereum transactions.

**Source:** [vechain-sdk-js: Secp256k1.ts](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/src/secp256k1/Secp256k1.ts)

## Design Decision: Deterministic Signatures (RFC 6979)

The signer uses pointycastle's `ECDSASigner(null, HMac(SHA256Digest(), 64))`:
the nonce k is derived from the private key and the hash with
HMAC-SHA256, and the `null` digest means the 32-byte hash is signed as
given.

**Why:** every ECDSA signature needs a secret nonce k. If k is ever
reused or predictable, the private key can be computed from the
signatures. RFC 6979 derives k deterministically, which removes the
dependency on a random source at signing time and makes signatures
reproducible: the same hash and key always give the same signature.
That is what lets the SDK match the official vector byte for byte.

**Sources:**
[RFC 6979: Deterministic Usage of DSA and ECDSA](https://www.rfc-editor.org/rfc/rfc6979) |
[bcgit/pc-dart: ecdsa_signer.dart](https://github.com/bcgit/pc-dart/blob/master/lib/signers/ecdsa_signer.dart)

## Design Decision: Always a Low S

For every valid signature (r, s), the pair (r, n - s) is also valid.
This "malleability" lets anyone alter a signature without invalidating
it. The SDK always normalizes S to its low form (s ≤ n / 2) with
pointycastle's `ECSignature.normalize`.

**Why:** the VeChainThor node rejects high-S signatures in the
transaction pool, for both the sender and the gas payer:

```go
// EnforceSignatureLowS checks that the S value in the signature are <= secp256k1 N/2.
// This is not required for consensus, but a protection against signature malleability.
func (t *Transaction) EnforceSignatureLowS() error {
```

A transaction with a high S never enters the pool. The official vector
confirms the normalization: the raw RFC 6979 signature for it has a
high S, so without normalization the SDK would not match it.

**Sources:**
[vechain/thor: tx/transaction.go](https://github.com/vechain/thor/blob/master/tx/transaction.go) |
[vechain/thor: txpool/tx_pool.go](https://github.com/vechain/thor/blob/master/txpool/tx_pool.go) |
[bcgit/pc-dart: ECSignature](https://github.com/bcgit/pc-dart/blob/master/lib/ecc/api.dart)

## Design Decision: Reject High-S Signatures When Parsing

`VeChainSignature.fromBytes` rejects any signature with s > n / 2. The
curve-level `VeChainSecp256k1.recoverPublicKey` still accepts it, so
the rule is a policy of the public type, not of the math.

**Why:** the official SDK accepts a high S when recovering. But such a
signature will be rejected by the network anyway, so the SDK reports it
early with a clear message instead of letting an app reach a generic
node error. Signatures produced by the SDK never have a high S.

## Design Decision: Recovery Implemented by the SDK

pointycastle signs and normalizes, but does not compute the recovery
byte or recover public keys. The SDK implements public key recovery
following SEC 1, section 4.1.6:

1. Build the point R with X = r and the Y parity given by the recovery
   byte (reusing the point decompression from `0.0.4-dev`).
2. Compute `Q = r⁻¹ (s × R - e × G)`, where e is the hash as an integer,
   evaluated as `u1 × G + u2 × R` with `u1 = -e × r⁻¹` and
   `u2 = s × r⁻¹ (mod n)`.
3. Q is the signer's public key.

To pick the recovery byte when signing, the SDK tries `0` and `1` and
keeps the one that recovers its own public key. Values `2` and `3`
would require r ≥ n, which happens with a probability of about 2⁻¹²⁸;
the official SDK rejects them too.

**Source:** [SEC 1: Elliptic Curve Cryptography, section 4.1.6](https://www.secg.org/sec1-v2.pdf)

## Design Decision: Only 32-Byte Hashes

`sign` and `recoverPublicKey` accept only a 32-byte hash, and reject
anything else (the official tests use the 18-byte text
`some_invalid_stuff` as the invalid example).

**Why:** VeChainThor always signs a hash (the BLAKE2b-256 signing hash
of a transaction, or the Keccak-256 hash of a message), never a raw
message. Requiring 32 bytes prevents an app from signing a message by
mistake, which would produce a valid-looking signature over the wrong
data.

## Design Decision: Validated Signature Type

`VeChainSignature.fromBytes` checks, before anything else:

- exactly 65 bytes;
- r and s in [1, n - 1];
- s ≤ n / 2;
- a recovery byte of `0` or `1`.

**Why:** a signature that passes these checks is always well-formed,
so the rest of the SDK (transactions, fee delegation, certificates) can
rely on `VeChainSignature` without repeating them.

## Test Vectors

| Source | Hash | Private key | Signature |
|---|---|---|---|
| vechain-sdk-js (official) | `keccak256('hello world')` = `0x47173285...4cb01fad` | `0x7582be...e26a` | `0xf8fe82c7...6b934d72` + `00` |
| Independently computed | same hash | `0x5434c1...502f` | `0x275f93a4...7efb8a1e` + `01` |
| Independently computed (high S) | same hash | `0x7582be...e26a` | the official signature with `n - s` and the recovery byte flipped |

The official signature was regenerated with a separate implementation
(Python `ecdsa`, RFC 6979 with HMAC-SHA256, then low-S normalization)
and matched byte for byte. The independent vectors were produced the
same way and confirmed by recovering their public keys. Together, the
two signed vectors cover both recovery bytes, `0` and `1`.

**Source:** [vechain-sdk-js: Secp256k1 tests](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/tests/secp256k1/Secp256k1.unit.test.ts)

## What 0.0.5-dev Tests Actually Check

**Signing (`VeChainSecp256k1.sign`)**

1. Signing the official hash with the official key gives the official signature.
2. An independent vector with recovery byte `1` is reproduced exactly.
3. Signing is deterministic.
4. Twenty different hashes all get a low S and recover the signer.
5. A hash that is not 32 bytes (official example) throws.
6. An invalid private key (official all `0xff`) throws.

**Recovery (`VeChainSecp256k1.recoverPublicKey`)**

7. The official signature recovers the official public key.
8. A high-S signature still recovers at the curve level.
9. A hash that is not 32 bytes throws.
10. A recovery byte of `8` (official forged example) throws.
11. A signature that is not 65 bytes throws.
12. r = 0 throws.
13. s equal to the curve order throws.

**Parsing (`VeChainSignature.fromBytes`)**

14. The official signature parses, and its fields, bytes and text form round-trip.
15. A high-S signature is rejected, as the transaction pool does.
16. s = n / 2 is accepted and s = n / 2 + 1 is rejected.
17. A recovery byte of `2` is rejected.
18. A length other than 65 bytes is rejected.
19. r = 0 is rejected.

**Signer recovery (`VeChainSignature`)**

20. `recoverPublicKey` returns the official public key.
21. `recoverAddress` returns the signer's address for both vectors.
22. The same signature over a different hash recovers a different address.
23. Signatures with the same bytes are equal, with the same hash code.

**Key pairs (`VeChainKeyPair.sign`)**

24. The official key pair produces the official signature.
25. A generated key pair signs and recovers its own address.
26. A hash that is not 32 bytes throws.

See [`test/src/crypto/vechain_signature_test.dart`](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/test/src/crypto/vechain_signature_test.dart).

## Out of Scope for 0.0.5-dev

- **Transaction signing hashes:** signatures are applied to transaction
  signing hashes in Phase 3.
- **Fee delegation:** the gas payer signs
  `blake2b256(signingHash + origin)`, built in Phase 4 with this
  signer and the multi-part hashing of `0.0.3-dev`.
- **Certificates (VIP-192):** signed messages for authentication arrive
  in Phase 7.
- **Web platform support:** to be evaluated at the Phase 1 closing
  audit (`0.1.0-dev`).

## Related

- Previous: [Keypair, Address and Checksum](https://github.com/nemorixgroup/VeChain-Knowledge-Base/blob/main/docs-sdk/phase-1/keypair-and-address/README.md) (`0.0.4-dev`)
- Next: Mnemonic (`0.0.6-dev`)
- [CHANGELOG entry for 0.0.5-dev](https://github.com/nemorixgroup/vechain-flutter-sdk/blob/main/CHANGELOG.md#005-dev)

## Sources

- [vechain-sdk-js: Secp256k1.ts](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/src/secp256k1/Secp256k1.ts)
- [vechain-sdk-js: Secp256k1 tests](https://github.com/vechain/vechain-sdk-js/blob/main/packages/core/tests/secp256k1/Secp256k1.unit.test.ts)
- [vechain/thor: tx/transaction.go](https://github.com/vechain/thor/blob/master/tx/transaction.go)
- [vechain/thor: txpool/tx_pool.go](https://github.com/vechain/thor/blob/master/txpool/tx_pool.go)
- [bcgit/pc-dart: ecdsa_signer.dart](https://github.com/bcgit/pc-dart/blob/master/lib/signers/ecdsa_signer.dart)
- [bcgit/pc-dart: ECSignature](https://github.com/bcgit/pc-dart/blob/master/lib/ecc/api.dart)
- [RFC 6979: Deterministic Usage of DSA and ECDSA](https://www.rfc-editor.org/rfc/rfc6979)
- [SEC 1: Elliptic Curve Cryptography](https://www.secg.org/sec1-v2.pdf)
