# link.md

**Your database can talk to other databases.** link.md is the interconnect
standard for [db.md](https://github.com/carloslfu/db.md) stores: identity,
signing, a verifiable feed, capability grants, and five remote verbs —
everything a folder of plain text needs the moment it crosses a trust
boundary, and nothing it needs before that.

A db.md store is a folder anyone can read. link.md is how that folder gets:

- **an identity** — a brain is a keypair; names point at keys ([SPEC §2](SPEC.md#2-identity))
- **an address** — every record anywhere is `@brain/id` ([§3](SPEC.md#3-addressing))
- **a verifiable history** — an append-only, hash-chained, signed feed that is
  simultaneously audit log, replication, subscription, and the export
  ([§5](SPEC.md#5-the-feed--wire-profile-v1))
- **shareable authority** — owner-rooted capability grants: attenuable,
  expiring, revocable, delegable to agents ([§6](SPEC.md#6-grants))
- **five verbs** — `resolve` / `sync` / `grant` / `propose` / `subscribe`
  ([§7](SPEC.md#7-the-five-verbs-and-the-hub-http-binding))

The signature is **travel wear**: record files stay pure plain text, and
cryptography appears only where records cross parties. A store exported as
bare files is valid db.md; exported with its feed it is *verifiable* db.md —
feed + files = a provable full copy, portable to any home. That is ownership
by openness, made mechanical.

## Status

**v0 DRAFT, published from a running implementation.** The wire shapes in the
spec are deployed and cross-verified by two independent implementations (a
TypeScript hub, the Rust [`dbmd`](https://github.com/carloslfu/db.md) client
— one binary, two specs, like `git`). [STATUS.md](STATUS.md) says exactly
what is implemented, partial, and roadmap — the spec never claims more than
what runs.

Evolution is **additive**: implemented first, specified from reality,
extended through numbered optional extensions ([§11](SPEC.md#11-extensions)).
Breaking changes get a new wire profile version, never a mutation.

## Conformance

[`vectors/`](vectors/) holds cross-implementation test vectors: valid signed
feeds and deliberately tampered ones with expected failure reasons. A
conforming verifier accepts all valid vectors and rejects every tampered one.
Both reference implementations consume the same files, in both directions.

## The family

| Spec | Job |
| --- | --- |
| **db.md** | The store: a folder of plain text records. No signing, no wire — implementable in an afternoon. |
| **link.md** | The interconnect: identity, feed, grants, verbs — everything with a trust boundary. |

The boundary rule, decided once: *cross-party only → link.md;
folder-as-database → db.md.* A store never needs link.md to be valid db.md.

## License

Apache-2.0. The standard is open, un-branded, and genuinely forkable — by
design.
