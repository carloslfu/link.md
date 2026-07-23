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
| §2.3 Agent keys | **Roadmap (in build)** | Delegation grants to agent multikeys + client-side keygen; grants to hub accounts run today. |
| §2.4 Custody — custodied | **Implemented** | Per-brain Ed25519, encrypted at rest, hub signs on push. |
| §2.4 Custody — self-held | **Roadmap (in build)** | Client-signed entries + hub verify-and-store. |
| §5 Feed wire profile v1 | **Implemented, both sides** | Producer + verifier in TS; independent verifier in Rust (`dbmd subscribe`); chain, identity, and signature checks per §5.4. |
| §5.5 Scope-limited reads | **Implemented** | Path-scoped readers receive head movement only. |
| §6 Grants — prefix scope, read/write, expiry, revocation | **Implemented** | Server-side enforced; grantees are hub accounts in v1 (multikey grantees ride the agent-keys build). |
| §6 Delegated sub-grants (chains deeper than owner→grantee) | **Roadmap** | Shape specified; attenuation semantics fixed. |
| §7.1 resolve | **Implemented** | Card + record resolution (id and path) for granted callers. Public unauthenticated resolve: roadmap. |
| §7.2 sync pull/push | **Implemented** | Pack export with per-file hash verify; push small/large (presign+commit). |
| §7.3 grant verb | **Implemented** | Issue / list / revoke via `dbmd grant`. |
| §7.4 propose | **Implemented (site-inbox form)** | The open door with rate limits; brain-addressed propose = extension E4. |
| §7.5 subscribe | **Implemented, both sides** | `?after/limit` paging; local §5.4 verification in the open client. |
| §8 Bearer account keys | **Implemented** | Hashed at rest server-side. |
| §8 `LinkMD-Sig` proof-of-possession | **Roadmap (in build)** | Envelope frozen in the spec; ships with agent keys so auth is built once. |
| §9 Rotation / recovery / pinning | **Specified; custodial rotation only** | Rotation statements + pinning defined; self-held rotation tooling roadmap. |
| §10 Conformance vectors | **Implemented, both directions** | `vectors/` produced by one implementation, verified by the other. |
| E1–E6 extensions | **Reserved** | None implemented; numbered slots only. |

Known open edges, stated plainly:

- **No second independent hub implementation yet.** A minimal reference node
  (`dbmd serve` — serve a local store's card/feed/packs; mirror a remote brain
  by subscribe-verify-fetch) is in build; until it ships, "any hub" is a
  design property with one production instance.
- **No public registry.** Handles resolve against the hub that hosts the
  brain; cross-hub resolution is E5.
- **Propose merge semantics (spec §7.4) have not been exercised under real
  concurrent contention.**
