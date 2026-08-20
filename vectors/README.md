# Conformance vectors

Cross-implementation test vectors for **wire profiles v1 and v2** (SPEC §5,
§10).

## v2-content-tree.json, v2-portable-paths.json, v2-commit-bridge.json, and v2-changeset-withheld.json

The v2 vectors pin the domain-separated content-tree root, hiding-proof privacy
expectations, and the portable path decisions shared by the TypeScript hub and
Rust `dbmd`. The path corpus includes NFC, case aliases, Windows reserved names,
separator/ADS forms, control bytes, and accepted Unicode/device-name edges. Both
implementations consume byte-identical copies and independently run randomized
map/tree order-equivalence tests.

`v2-commit-bridge.json` pins the exact canonical commit field set, the
mandatory null-or-exact `v1_bridge` shape, deterministic Ed25519 signatures,
domain-separated commit hashes, plain feed hashes, and semantic refusals for a
missing/extra field, malformed bridge, or bridge after genesis. Its published
private key is test material only.

`v2-changeset-withheld.json` pins the exact additive canonical changeset bytes
and domain hash for one checkout-local kept-home observation. It proves that
old changesets remain byte-compatible while both implementations agree on the
new sorted `withheld_links` plus `checkout_id` binding.

## feed-v1.json

Signed feed entries: one valid three-entry chain (push → edit →
edit-with-removal) served as a full head response, and five tampered sets,
each with exactly one defect and a machine-checkable `reason`:

| reason | defect |
| --- | --- |
| `signature` | entry content altered after signing |
| `chain` | `prev_entry_hash` altered |
| `identity` | `brain` multikey does not match `public_key` |
| `sequence` | an entry missing from the middle |
| `head` | advertised `feedHash` is not the last entry's hash |

A conforming **verifier** (SPEC §5.4) accepts the valid chain and rejects
every invalid set. The `identity` and `signature` cases are entangled by
construction (tampering either breaks both); a verifier may report either
category for those two.

The `identity.privateKeyPkcs8` in the file is a **test key, published
deliberately** so any implementation can re-derive and re-sign every byte
from scratch. Never reuse it for anything.

## Provenance — both directions

- **Produced by** the TypeScript reference implementation's production signer
  (the same code path that signs real feeds), deterministically regenerable
  from the frozen test key.
- **Independently verified by** the Rust `dbmd` client
  ([carloslfu/db.md](https://github.com/carloslfu/db.md)): its integration
  suite serves these vectors through a mock hub to the real `dbmd subscribe`
  verb — accepts the valid head, refuses the tampered one.
- **The reverse direction** — a Rust-side independently generated fixture
  verified by the TypeScript implementation — runs in the TypeScript suite.

An implementation claiming conformance should wire these files into its own
tests the same way rather than re-deriving expectations by hand.
