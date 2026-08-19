# STATUS — what runs, what's partial, what's roadmap

Honesty ledger for the v0 draft. The spec documents deployed reality; this
file says exactly how much of it is deployed, so nobody has to guess. Updated
with the spec, never behind it.

**Reference implementations:** a TypeScript hub (production, closed source
today) and the Rust `dbmd` client (open, [carloslfu/db.md](https://github.com/carloslfu/db.md),
shipped on crates.io/Homebrew). "Both sides" below means those two.

| Spec area | State | Reality |
| --- | --- | --- |
| §2.1 Multikey identity (`ed25519:<fingerprint>`) | **Implemented, both sides** | Deployed wire form; verifiers recompute fingerprints and refuse mismatches. |
| §2.2 Brain-card | **Implemented, both sides** | Served on the brain envelope; verified locally by `dbmd`. |
| §2.3 Agent keys | **Implemented** | Client-side keygen (`dbmd key generate` — the secret never leaves the machine) + hub registration (public half only) + per-key revocation and last-use audit + a dashboard panel to see/cut every keyed actor. |
| §2.4 Custody — custodied | **Implemented** | Per-brain Ed25519, encrypted at rest, hub signs on push. |
| §2.4 Custody — self-held | **Implemented, both sides** | Create with `publicKeySpki` → the hub registers verify-only identity (it cannot lose a key it never held); `dbmd sync --push` with a local brain key signs each entry and ships it through the pack flow; the hub pins signature, chain position, pack hash, manifest set-equality, and the normative serialization, then stores the client's exact bytes. Hub-signed paths refuse with machine codes. |
| §5 Feed wire profile v1 | **Implemented, both sides** | Producer + independent Rust verifier; card/feed exact-head agreement, immutable snapshot-addressed exports, signed pack/manifest verification, and persistent rollback/equivocation checkpoints per §5.4. |
| §5.7 Permissioned incremental wire profile v2 | **Implemented, both sides; production-proven** | The TypeScript hub commits changed content-addressed objects behind one signed atomic head; the independent Rust client verifies head/commit/feed/proofs/blobs, performs private three-way sync, and advances trust only after its final barrier. A synthetic production brain proved a 573-file genesis retry, exact five-file delta, two simultaneous 516-file cold clones, direct large-blob upload, and stable no-op receipts. Cross-language HAMT vector is in `vectors/v2-content-tree.json`. |
| §5.1 v1 edit disclosure compatibility | **Frozen** | `push.files` is complete; `edit.files` must cover every actual create/change and may reassert unchanged resulting paths. Exact removals and pack equality remain mandatory. This preserves both minimal-delta vectors and already-signed complete-manifest edits. |
| §5.7 v2 deletion/conflict/local-policy semantics | **Implemented, both sides; production-proven** | Typed delete/restore, exact preconditions, disjoint rebase, same-path stop, baseline-safe pull deletion, `.sevralocal` remote-copy preservation/adoption, scoped create/delete propagation, atomic mixed-scope denial, generated-projection refusal, scope-change checkout refusal, and final revocation were exercised against production. Typed rename exists in the engine/API; released-client UX remains in the open cutover work. |
| §5.7 v2 self-custody writes | **Implemented, both sides** | The hub stages one exact strict-parent candidate and returns canonical Ed25519 bytes plus a paged proof-carried full manifest. Rust `dbmd` verifies exact manifest equality and the changeset/actor/parent/root/materializer bindings before signing, then retries the same stable mutation. Scoped writers are refused into proposals rather than shown hidden state. |
| §5.7 v2 asset root, bulk stream, purge/GC | **In progress** | Current/history byte denial and the encrypted selective-purge barrier are implemented at the hub. Signed asset inventories, physical purge/GC completion, and the exact-commit bulk stream remain cutover gates in the product plan. |
| §6 v2 action/path/history grants | **Implemented; production-proven for scoped content** | Exact/prefix action grants, org control/content split, history issuance boundary, semantic frontmatter gates, source lifecycle, final authority/hosting recheck. A foreign key was narrowed from a four-file prefix to one exact file, the old view failed closed, and revocation made the pinned brain uniformly unavailable. Full generated role/action matrices and control-transition recovery remain open in the product plan. |
| §5.5 Scope-limited reads | **Implemented** | Path-scoped readers receive head movement only. |
| §6 Grants — prefix scope, read/write, expiry, revocation | **Implemented** | Server-side enforced; grantee is a hub account OR a bare multikey (cross-party keys, XOR-enforced). |
| §6 Cross-party key grantees | **Implemented, both sides** | `dbmd grant issue @brain <publicKeySpki>` grants to a foreign keyholder with no hub account; it authenticates purely by `LinkMD-Sig` and its authority is exactly its grants (same policy path as user grants). Proven live: a foreign key reads a private brain, and loses it the instant the grant is revoked. |
| §6 Delegated sub-grants (chains deeper than owner→grantee) | **Implemented, both sides** | A key holder delegates an attenuated sub-grant of a grant it holds; authority resolves by walking parent links to an owner root (most-restrictive capability/scope, coherent issuer chain, live ancestors). Revoking any ancestor cascades with no child write. Proven live: a foreign key sub-delegates to a third key that reads the brain, over-delegation (read→write) is refused, and revoking the parent removes the sub-delegate's access. |
| §7.1 resolve | **Implemented** | Card + record resolution (id and path) for granted callers; a publishing handle resolves for ANY caller — anonymous included — while its brain is public (cross-party resolution v0). |
| §7.2 sync pull/push | **Implemented, both sides** | Pull/mirror fetch the immutable `(headSeq, feedHash)` snapshot and atomically publish only after signed pack/manifest verification; push small/large uses presign+commit. |
| §7.3 grant verb | **Implemented** | Issue / list / revoke via `dbmd grant`. |
| §7.4 propose | **Implemented (site inbox + brain-addressed + pull-drained)** | Bare `@brain` propose is live: anonymous on public brains, actor-class rate tiers (stranger / granted-user / granted-key / owner), same evidence landing and daily cap. A self-custodied brain QUEUES submissions outside the store (202) — the owner's agent drains them (read → write locally → push signed → DELETE-ack) — because only the key holder writes the store. Structured record-change proposals remain E4. |
| §7.5 subscribe | **Implemented, both sides** | `?after/limit` paging; local §5.4 verification in the open client. |
| §8 Bearer account keys | **Implemented** | Hashed at rest server-side. |
| §8 `LinkMD-Sig` proof-of-possession | **Implemented, both sides** | v2 binds the externally authoritative normalized origin as well as method/path/body, so a captured proof cannot cross hubs; v1 is rejected. The hub checks the window before key lookup and revocation immediately; mutating envelopes are durably one-shot through the shared replay store and fail closed when it is unavailable; safe reads may repeat. `dbmd` signs every authenticated verb when `DBMD_AGENT_KEY_FILE` is set, outranking the bearer. |
| §9 Rotation / recovery / pinning | **Implemented, both sides** | Every previous identity is proved by an adjacent old-key-signed statement bound to its exact feed boundary; signer epochs prevent retired-key append. Self-custody rotation durably creates the new 0600 key before an idempotent remote commit. One user-owned pin/checkpoint policy governs every authenticated verb and advances only after full verification. Hub-custodied rotation is immediately recoverable from independently encrypted state. |
| §10 Conformance vectors | **Implemented, both directions** | `vectors/` produced by one implementation, verified by the other. |
| E5 public registry (cross-hub handle resolution) | **Implemented (v0)** | `GET /api/hub/registry/<handle>` resolves any handle to { identity, home, brain } for any client; external homes are claimed by PROOF OF KEY CONTROL (the registration is signed by the key it names) so a squatter can grab a free name but only point it at a key they hold, and every resolver pins the key. `dbmd resolve @handle` follows direct-first, then the registry, fetching the card from the home and refusing an identity mismatch. Proven live (proof-of-control registration + follow + pin + revoke) and hermetically across two nodes. Name-dispute governance (beyond reserved names + report/takedown) is policy, not code. |
| E6 secp256k1 / npub key facet | **Implemented (hub verification)** | The multikey layer adds a self-describing `secp256k1` tag; agent keys, grantees, and delegation accept an npub / secp256k1 x-only key; the hub verifies BIP340 Schnorr LinkMD-Sig. Proven live: a Nostr npub holds a grant and authenticates. A `dbmd` secp256k1 keygen (so `dbmd` itself can BE an npub) is the client-side follow-on; today an npub holder signs with their own Nostr tooling. |
| E1–E4 extensions | **Reserved** | Frontmatter-query scopes, record-grade feed ops, push transports, structured propose — numbered slots only. |

Known open edges, stated plainly:

- ~~No second independent hub implementation~~ **RESOLVED**: `dbmd mirror`
  replicates a brain with full §5.4 chain verification and pins its identity
  (trust-on-first-use); `dbmd serve` re-serves the mirror over the hub HTTP
  binding from any machine, and a downstream `dbmd` re-verifies the ORIGINAL
  signatures with no hub in the loop. Two independent server implementations
  of the protocol exist, and the export is provable because signatures
  survive re-hosting.
- **Propose merge semantics (spec §7.4) have not been exercised under real
  concurrent contention.**
