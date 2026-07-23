# link.md

**The interconnect standard for db.md stores: how brains address, identify,
sign, exchange, and grant.**

Version: **v0 (DRAFT)** · Wire profile: **v1** · License: Apache-2.0

---

## 0. Status, scope, and versioning

link.md is the second spec in the db.md family. [db.md](https://github.com/carloslfu/db.md)
defines the store — a folder of plain text records. link.md defines everything
that happens **across a trust boundary or a wire**: identity, signing, the
verifiable feed, capability grants, and the five remote verbs.

This document is a **v0 draft published from a running implementation**, not a
proposal. Every normative wire shape in §5–§7 is deployed and cross-verified by
two independent implementations (a TypeScript server, a Rust client — see
`vectors/`). The draft label means the spec may still gain things; the
discipline is the same as db.md's: **evolution is additive**. Anything that
would break a conforming implementation gets a new wire profile version (the
`v` field), never a mutation of this one.

The words MUST, SHOULD, and MAY are used in the RFC 2119 sense.

**The boundary rule** (shared with db.md, decided once): *cross-party only →
link.md; folder-as-database → db.md; retrofit-expensive-if-skipped → one line
in db.md.* db.md carries exactly two link.md-adjacent conventions — the
recommended record `id` and the reserved `@brain/id` address shape — and
nothing else. A store never needs link.md to be valid db.md.

## 1. Model

A **brain** is a db.md store with an identity. A **hub** is any server that
hosts brains and speaks this protocol. A **client** is any tool that operates a
store and speaks the verbs (the reference client is `dbmd`, the db.md toolkit
binary — one binary, two specs, exactly as one `git` binary implements both the
object format and the wire protocol).

Four ideas carry the whole design:

1. **A brain is a keypair.** The public key is the root identity; names point
   at keys, never the reverse.
2. **The signature is travel wear.** Record files stay pure db.md plain text;
   signatures live in the feed beside the store, and appear only where records
   cross a trust boundary. A store exported as bare files is valid db.md; a
   store exported with its feed is *verifiable* db.md.
3. **The feed is history, replication, subscription, and export — one
   mechanism.** An append-only, hash-chained, signed sequence of entries.
   Follower + feed + files = a provable full copy.
4. **Access is a grant: an owner-rooted capability, not a role.** Attenuable,
   expiring, revocable, delegable to agents. Enforcement is server-side at the
   hub — a folder cannot enforce anything, so the store never pretends to.

## 2. Identity

### 2.1 Keys and the multikey encoding

A brain identity is an asymmetric keypair. Wire profile v1 uses **Ed25519**.

The canonical key identifier is a **multikey** string:

```
<alg> ":" <fingerprint>
```

- `alg` — the lowercase algorithm tag. v1 defines exactly one: `ed25519`.
- `fingerprint` — `base64url(sha256(SPKI))`, unpadded, where SPKI is the DER
  encoding of the public key in SubjectPublicKeyInfo form (RFC 5280).

Example: `ed25519:plXvdIhBGCFUevYYhNO3LX-IEElGNZhgdUnaOIucWFQ`

The prefix makes key identifiers **self-describing**: a future algorithm is a
new tag (for example a secp256k1 Schnorr key could carry a `secp256k1:` tag,
making Nostr-style keys expressible as facets), never a re-interpretation of an
existing one. Implementations MUST reject multikeys whose tag they do not
recognize rather than guessing.

Where a full key travels on the wire, it travels as `base64url(SPKI DER)`,
unpadded, in a field named `publicKeySpki` (JSON APIs) or `public_key` (feed
entries). A verifier MUST recompute the fingerprint from the delivered key and
refuse any object whose stated identity does not match.

### 2.2 The brain-card

The brain-card is the public identity object a hub serves for a brain:

```json
{ "fingerprint": "<base64url sha256(SPKI)>", "publicKeySpki": "<base64url SPKI DER>" }
```

Hubs MAY extend the card's envelope with metadata (display name, handle,
endpoints, head state — see §7.1); the two identity fields are the normative
core. Cards are exchanged, not trusted blindly: a client that has pinned a
brain's fingerprint MUST refuse a card that contradicts the pin (§9.3).

### 2.3 Agents are identities too

An agent operating a brain SHOULD hold its own keypair, distinct from the
brain's root key, authorized by a grant (§6) whose grantee is the agent's
multikey. Authorship is then always attributable: who signed, what kind of
actor it was, and on whose authority (the grant chain). A client generating an
agent key MUST generate it locally and MUST NOT place the private key inside
the store.

### 2.4 Custody

Where the private key lives is an operational choice, never a format one:

- **Custodied** — a hub holds the brain key (encrypted at rest) and signs feed
  entries server-side on the owner's authenticated writes.
- **Self-held** — the client holds the brain key and ships signed entries; the
  hub verifies and stores (verify-and-store).

Both produce the identical wire artifacts. The private key MUST NOT appear
inside the store in either model; only public identity ever travels.

## 3. Addressing

Every record anywhere is addressable as:

```
@<brain-handle>/<record-id>
```

- `<record-id>` is the db.md v0.4 record `id` — a lowercase ULID.
- `<brain-handle>` is a human-legible name bound to a key at a registry, or the
  brain's opaque id where no name is bound. The key is the identity; the handle
  is a pointer.

`@brain` alone addresses the brain (resolves to its card). An unresolved
`@brain/id` in a wiki-link is legal db.md text; resolution is a link.md client
capability, never a format requirement. db.md reserves this shape and defines
nothing else about it — resolution, registries, and verification are entirely
this spec's concern.

## 4. Signing

**What is signed:** feed entries (§5) — the tuple describing a store
transition — using the brain's key. Record files are never modified by
signing; integrity of content is carried by hashes inside the signed entry.

**Where signatures live:** in the feed, not in the files. This keeps the
format untouched and means signatures appear exactly where they mean
something — between parties. Local integrity of a working store remains the
job of versioning (git or equivalent), as db.md defaults.

**Hashes:** every content hash in this spec is SHA-256. File hashes are
computed over the file's raw bytes — no newline normalization, no encoding
transformation. Decide-once rule: the bytes on disk are the bytes that hash.

## 5. The feed — wire profile v1

The feed is a per-brain, append-only sequence of **signed entries**, numbered
from `seq = 1`.

### 5.1 Entry shape

An entry is a JSON object with these fields **in exactly this order**:

```json
{
  "v": 1,
  "seq": 41,
  "ts": "2026-07-14T00:00:00.000Z",
  "brain": "ed25519:plXvdIhBGCFUevYYhNO3LX-IEElGNZhgdUnaOIucWFQ",
  "public_key": "MCowBQYDK2VwAyEAgJLl1ujKETgW6L9RU4sVvKsDOURNZpjy6KnffeIj4VU",
  "kind": "push",
  "op": "snapshot",
  "pack_sha256": "<sha256 hex of the store pack this entry commits>",
  "files": [ { "path": "DB.md", "sha256": "<sha256 hex>", "bytes": 3 } ],
  "removed": [],
  "prev_entry_hash": null,
  "sig": "<base64url Ed25519 signature>"
}
```

Field semantics:

- `v` — wire profile version. This profile is `1`.
- `seq` — 1-based position. MUST increase by exactly 1 per entry.
- `ts` — ISO 8601 UTC timestamp with milliseconds.
- `brain` — the brain's multikey (§2.1).
- `public_key` — the brain's full public key, `base64url(SPKI DER)`.
- `kind` — `"push"` (the entry's `files` list is the complete resulting
  manifest) or `"edit"` (`files` lists only changed paths; `removed` lists
  deletions). Both commit the complete resulting store.
- `op` — `"snapshot"` in profile v1: the entry addresses the full resulting
  store pack. Finer-grained ops are extension territory (§11).
- `pack_sha256` — sha256 hex of the complete store pack after this entry.
- `files` — objects with fields in the order `path`, `sha256`, `bytes`.
- `removed` — store-relative paths deleted by this entry.
- `prev_entry_hash` — the previous entry's hash (§5.3); `null` iff `seq` is 1.
- `sig` — the signature (§5.2).

### 5.2 Signing bytes

The signature is Ed25519 over the UTF-8 bytes of the JSON serialization of the
entry **without** `sig`, fields in the §5.1 order, minimal JSON (no added
whitespace, no key sorting, no float re-encoding). Serialize-then-sign;
append `sig` as the final field to form the entry.

There is deliberately no canonicalization algorithm: profile v1 fixes the
field order and the minimal serialization instead, which two independent
implementations reproduce today. If a future profile ever needs an abstract
canonical form (for example RFC 8785 JCS), it arrives as `v: 2` — a verifier
of v1 entries never needs one.

### 5.3 Entry bytes, entry hash, and the chain

- **Entry bytes** = the full entry JSON (including `sig`, same order rules)
  followed by exactly one `\n`.
- **Entry hash** = sha256 hex of the entry bytes.
- **Chain rule**: entry N's `prev_entry_hash` MUST equal entry N−1's entry
  hash. Entry 1's MUST be `null`.

The chain makes truncation and tampering detectable: changing any byte of any
entry changes its hash and breaks every later entry.

### 5.4 Verification

A verifier processing entries `after..head` MUST check, per entry:

1. `seq` is contiguous (previous + 1);
2. `prev_entry_hash` equals the previous entry's hash (or `null` at seq 1);
3. `brain` equals `ed25519:` + the fingerprint recomputed from `public_key`;
4. `public_key` matches the brain identity the verifier expects (the card, or
   a pinned fingerprint);
5. `sig` verifies over the §5.2 bytes.

A verifier at the head SHOULD additionally check the served head state
(§7.1's `headSeq`/`feedHash`) against the last entry's hash, and MAY check
`pack_sha256` against separately fetched content.

### 5.5 Scope-limited readers

A signed entry's `files` manifest cannot be filtered without invalidating the
signature, and returning it whole would leak out-of-scope paths. A hub serving
a reader whose grant covers only part of a store (§6) MUST NOT serve entry
bodies; it serves head movement only (`headSeq`, `feedHash`, empty `entries`,
`scopeLimited: true`). Full-feed reads require a full-store grant.

### 5.6 What the feed is

The one mechanism is simultaneously:

- **history** — the audit log of the brain;
- **replication** — a follower replays entries and fetches content by hash;
- **subscription** — deltas since seq N;
- **the verifiable export** — feed + files = a provable full copy, portable to
  any other home. Ownership by openness is this property, made mechanical.

## 6. Grants

A grant is an owner-rooted capability:

```yaml
issuer: <the brain owner (root) or a parent grantee (delegation)>
to: <grantee: an account, or an agent/person multikey>
capability: read | write
scopePrefix: <store-relative path prefix; absent = the whole store>
expiresAt: <ISO timestamp; absent = until revoked>
```

Profile v1 semantics:

- **Attenuation only.** A delegated grant MUST NOT exceed its parent in scope,
  capability, or lifetime. Chains terminate at the owner. A grant carries a
  `parent` (the grant it attenuates; absent = owner-rooted) and, when a key
  rather than the owner issued it, the issuer's multikey. A verifier resolves a
  key's authority by walking parent links to an owner root: every ancestor MUST
  be live (unrevoked, unexpired), each grant's issuer MUST be its parent's
  grantee (you may delegate only a grant you hold), and the effective
  capability/scope is the most restrictive along the whole chain. Revoking any
  ancestor therefore invalidates the entire subtree with no cascade write.
- **Revocation wins.** A revoked grant is dead for every party derived from it.
- **Scope is a path prefix** in v1 — cheap to enforce, covers the real cases
  (a records subtree, a client folder). Frontmatter-query scopes
  (`type=invoice AND meta-type=fact`) are the planned extension, layered
  additively (§11).
- **Enforcement is server-side.** The hub is the reference monitor. The store
  itself never enforces anything, and this spec never pretends otherwise —
  which is also why it contains no encryption-based sharing scheme (§12).
- Grantees in v1 deployments may be hub accounts rather than keys; the record
  shape is identical, so upgrading a grantee to a multikey is a field change,
  not a migration. Not RBAC, not blockchain: object capabilities
  (UCAN/macaroon lineage) with the owner as the root of every chain.

## 7. The five verbs and the hub HTTP binding

The verbs are the protocol's whole client surface:

| Verb | Meaning |
| --- | --- |
| `resolve` | handle/id → brain-card; `@brain/id` → a record |
| `sync` | pull or push a granted slice of the store |
| `grant` | issue, list, revoke capabilities |
| `propose` | submit evidence/changes without write trust; the owner's curator accepts or rejects |
| `subscribe` | follow and locally verify a brain's signed feed |

Profile v1 binds them to HTTP against a hub base URL. All non-public calls are
authenticated (§8). Paths are relative to the hub origin.

### 7.1 resolve

```
GET /api/hub/brains/<brain>
```

`<brain>` is the brain id, or the caller's own slug for it. Returns the brain
envelope: `id`, `name`, `visibility`, publishing `handle`, `headSeq`,
`feedHash`, and `identity` — the brain-card (§2.2). Hidden brains MUST 404,
never 403 (no existence oracle).

```
GET /api/hub/brains/<brain>/resolve?id=<ulid>
GET /api/hub/brains/<brain>/resolve?path=<store-relative path>
```

Resolves a single record to its content for a caller whose grant covers it.

### 7.2 sync

Pull:

```
GET /api/hub/brains/<brain>/export?format=pack
```

Returns the granted slice as an immutable pack; the client MUST verify each
file's sha256 against the manifest.

Push (custodied signing, v1):

```
POST /api/hub/brains/<brain>/push        { "files": [ { "path", "content" } ] }
POST /api/hub/brains/<brain>/packs/presign   { "sha256", "bytes" }   (large stores)
POST /api/hub/brains/<brain>/packs/commit    (after upload)
```

The hub appends the resulting signed entry to the feed. A self-held-key client
ships the signed entry itself and the hub verifies before storing (§2.4).

### 7.3 grant

```
GET    /api/hub/brains/<brain>/grants
POST   /api/hub/brains/<brain>/grants        { "email" | "to", "capability", "scopePrefix"?, "expiresAt"? }
DELETE /api/hub/brains/<brain>/grants/<id>
```

### 7.4 propose

```
POST /api/hub/sites/<handle>/inbox           { "app", "body" }
```

Propose is deliberately the open door: unauthenticated evidence-in, rate
limited, landing in the owner's inbox for the curator to accept or reject.
Nothing enters the store without the owner's side accepting it. Merge
semantics for concurrent proposals against one record: curator-serialized,
first-accepted-wins; later proposals are rebased by the accepting agent
against the new state.

Brain-addressed propose — `POST /api/hub/brains/<brain>/inbox` — is the
generalization for bare `@brain` addresses: anonymous callers reach PUBLIC
brains only (the open door), authenticated callers (user or key) earn larger
actor-class budgets. A self-custodied brain (§2.4) MUST NOT have proposals
signed into its store by the hub — only its key holder writes it — so the hub
QUEUES the submission outside the store and returns 202; the owner's agent
drains the queue (read → write locally → push signed → acknowledge). The hub
is a mailbox there, never an author.

### 7.5 subscribe

```
GET /api/hub/brains/<brain>/feed?after=<seq>&limit=<n>
```

Returns:

```json
{
  "brain": "<id>", "headSeq": 41, "feedHash": "<hex>",
  "identity": { "fingerprint": "…", "publicKeySpki": "…" },
  "entries": [ { "hash": "<hex>", "entry": { …§5.1… } } ],
  "nextAfter": 41, "hasMore": false, "scopeLimited": false
}
```

`after` ≥ 0 (default 0), `limit` 1–100 (default 100). The client MUST verify
per §5.4 and SHOULD treat the advertised `feedHash` as the head to converge
on. Transport is plain HTTP polling in v1; push transports are extensions.

## 8. Authentication

Two client authentication methods:

- **Account key (bearer)** — a hub-minted secret presented per request. Hubs
  MUST store only a hash of it. The simple path.
- **Signed requests (proof of possession)** — for key-holding actors (§2.3):

```
Authorization: LinkMD-Sig v1,key=<multikey>,ts=<unix-seconds>,sig=<base64url(ed25519(canonical))>
canonical = "v1" LF method LF path-and-query LF ts LF (sha256hex(body) | "-")
```

The hub verifies the signature with the registered public key for `key`,
enforces a ±60 second window on `ts`, and binds the signature to the exact
method, path, and body. Nothing reusable ever crosses the wire, appears in a
log, or lands in an agent transcript — the credential cannot leak in transit
because it never travels.

## 9. Rotation, recovery, and pinning

### 9.1 Rotation

Identity rotation is a **rotation statement**: a declaration binding the new
key, signed by the **old** key, appended to the brain's history. A verifier
that trusts fingerprint F and sees a valid rotation F→F′ SHOULD trust F′ and
treat F as historical. Chains of rotations verify transitively. A brain-card
MAY carry a `previous` list of prior identities; a feed entry verifies when it
is signed by the current identity OR any listed previous one, so rotation
never invalidates history.

### 9.2 Recovery

- Custodied brains: the custodian rotates on the owner's authenticated
  request — recovery is an account concern, not a protocol one.
- Self-held brains: recovery is the exported identity file. A lost self-held
  key without a rotation signed by it is a lost identity; parties who pinned
  it must re-verify out of band. This spec deliberately defines no social
  recovery.

### 9.3 Pinning

A client that has synced a brain SHOULD pin its fingerprint in local toolkit
state (outside the store) and MUST refuse subsequent responses presenting a
different identity absent a valid rotation chain. First contact is
trust-on-first-use unless the fingerprint arrived out of band.

## 10. Conformance

An implementation conforms as a **verifier** if it accepts every valid vector
in `vectors/` and rejects every tampered one for the stated reason; as a
**producer** if its output verifies under an independent conforming verifier.
The vectors are cross-implementation artifacts: produced by one
implementation, verified by another, in both directions. See
`vectors/README.md`.

## 11. Extensions

The core above is frozen per wire profile. Everything else arrives as
**numbered, optional, strictly additive extensions** — an implementation
advertises what it supports; the core never breaks. Reserved slots:

| Ext | Subject |
| --- | --- |
| E1 | Frontmatter-query grant scopes |
| E2 | Record-grade feed ops (per-record entries beside `snapshot`) |
| E3 | Push transports for subscribe (SSE/webhooks) |
| E4 | Brain-addressed propose + structured record-change proposals |
| E5 | Public registries + cross-hub handle resolution |
| E6 | Key facets (algorithm bindings beyond ed25519, e.g. secp256k1) |

## 12. Non-goals

Named so implementers do not wait for them:

- **No CRDTs.** Collaboration is record-granular and curator-mediated;
  same-record conflicts are rare and resolved by the owner's side.
- **No encryption-based access control.** Read access is enforced by the
  serving side; ciphertext-sharing schemes fail at revocation and attenuation,
  and pretending otherwise is worse than saying "the server enforces."
- **No blockchain, no consensus, no tokens.** Every chain here is a plain
  hash chain with one writer.
- **No numeric event kinds.** Records carry human-legible `type:` words;
  agents read words.
- **No DID methods, no key transparency logs.** A multikey and a pin do the
  v1 job; heavier identity systems can bind to E6 facets if ever warranted.
- **No vectors/embeddings anywhere.** Inherited from db.md.

## 13. Relationship to neighboring systems

- **git** — the closest relative in spirit. The feed is to a brain what the
  commit chain is to a repo: history, replication, and export in one hashed
  structure. link.md exists because brains need two things repos do not:
  per-slice access control on the serving side, and signatures that survive
  hosting (travel wear, not commit authorship).
- **Nostr** — shares the "actor = keypair, artifact = signed JSON" instinct,
  and deployments of it (e.g. Block's Buzz) validate symmetric key-based
  identity for humans and agents. The models differ underneath: Nostr events
  are standalone datagrams with relay-optional persistence — no sequence, no
  chaining, no completeness; link.md's feed is a chained, complete,
  state-replication log, because a brain's export must be provable, not
  best-effort. Grants have no Nostr counterpart (authenticated actors are
  peers; brains need owner-rooted attenuable authority). The multikey design
  (§2.1, E6) keeps key-level interop deliberately cheap.
- **AT Protocol** — bundles identity, schema, and hosting into one stack.
  db.md/link.md cut the other way on purpose: the format stays
  afternoon-implementable with zero cryptography, and every trust concern
  lives here, adopted only when data crosses parties.
- **Solid** — shares the ownership goal, not the mechanism: link.md attaches
  ownership to an exportable artifact (feed + files anywhere) rather than to
  where data lives.

---

*link.md consumes db.md v0.4 (record `id`s; the reserved `@brain/id` shape)
and adds nothing to the format. A store never needs link.md to be valid
db.md.*
