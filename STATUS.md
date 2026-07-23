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
| §5 Feed wire profile v1 | **Implemented, both sides** | Producer + verifier in TS; independent verifier in Rust (`dbmd subscribe`); chain, identity, and signature checks per §5.4. |
| §5.5 Scope-limited reads | **Implemented** | Path-scoped readers receive head movement only. |
| §6 Grants — prefix scope, read/write, expiry, revocation | **Implemented** | Server-side enforced; grantee is a hub account OR a bare multikey (cross-party keys, XOR-enforced). |
| §6 Cross-party key grantees | **Implemented, both sides** | `dbmd grant issue @brain <publicKeySpki>` grants to a foreign keyholder with no hub account; it authenticates purely by `LinkMD-Sig` and its authority is exactly its grants (same policy path as user grants). Proven live: a foreign key reads a private brain, and loses it the instant the grant is revoked. |
| §6 Delegated sub-grants (chains deeper than owner→grantee) | **Implemented, both sides** | A key holder delegates an attenuated sub-grant of a grant it holds; authority resolves by walking parent links to an owner root (most-restrictive capability/scope, coherent issuer chain, live ancestors). Revoking any ancestor cascades with no child write. Proven live: a foreign key sub-delegates to a third key that reads the brain, over-delegation (read→write) is refused, and revoking the parent removes the sub-delegate's access. |
| §7.1 resolve | **Implemented** | Card + record resolution (id and path) for granted callers; a publishing handle resolves for ANY caller — anonymous included — while its brain is public (cross-party resolution v0). |
| §7.2 sync pull/push | **Implemented** | Pack export with per-file hash verify; push small/large (presign+commit). |
| §7.3 grant verb | **Implemented** | Issue / list / revoke via `dbmd grant`. |
| §7.4 propose | **Implemented (site inbox + brain-addressed + pull-drained)** | Bare `@brain` propose is live: anonymous on public brains, actor-class rate tiers (stranger / granted-user / granted-key / owner), same evidence landing and daily cap. A self-custodied brain QUEUES submissions outside the store (202) — the owner's agent drains them (read → write locally → push signed → DELETE-ack) — because only the key holder writes the store. Structured record-change proposals remain E4. |
| §7.5 subscribe | **Implemented, both sides** | `?after/limit` paging; local §5.4 verification in the open client. |
| §8 Bearer account keys | **Implemented** | Hashed at rest server-side. |
| §8 `LinkMD-Sig` proof-of-possession | **Implemented, both sides** | Hub verifies (window before key lookup; path/body/method binding; immediate revocation); `dbmd` signs every authenticated verb when `DBMD_AGENT_KEY_FILE` is set, outranking the bearer. Proven end to end against the production reference hub. |
| §9 Rotation / recovery / pinning | **Implemented, both sides** | `dbmd key rotate` on a self-custody brain signs the §9.1 statement with the old key; the hub also self-rotates a custodied key; the old identity moves into `previous` and every pre-rotation feed entry keeps verifying (proven live). `dbmd mirror`/`serve` carry `previous`. Recovery beyond rotation, and pinning enforcement in the hub client, stay per §9.2/§9.3. |
| §10 Conformance vectors | **Implemented, both directions** | `vectors/` produced by one implementation, verified by the other. |
| E1–E6 extensions | **Reserved** | None implemented; numbered slots only. |

Known open edges, stated plainly:

- ~~No second independent hub implementation~~ **RESOLVED**: `dbmd mirror`
  replicates a brain with full §5.4 chain verification and pins its identity
  (trust-on-first-use); `dbmd serve` re-serves the mirror over the hub HTTP
  binding from any machine, and a downstream `dbmd` re-verifies the ORIGINAL
  signatures with no hub in the loop. Two independent server implementations
  of the protocol exist, and the export is provable because signatures
  survive re-hosting.
- **No public registry.** Handles resolve against the hub that hosts the
  brain; cross-hub resolution is E5.
- **Propose merge semantics (spec §7.4) have not been exercised under real
  concurrent contention.**
