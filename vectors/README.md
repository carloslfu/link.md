# Conformance vectors

Cross-implementation test vectors for **wire profile v1** (SPEC §5, §10).

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
