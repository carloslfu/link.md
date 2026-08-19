# link.md

**The interconnect standard for db.md stores: how brains address, identify,
sign, exchange, and grant.**

Version: **v0 (DRAFT)** · Wire profiles: **v1 + v2** · License: Apache-2.0

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
  manifest) or `"edit"` (`files` is a change/invalidation disclosure). An
  `edit` MUST list every created or byte-changed path and MAY additionally
  reassert any unchanged path from the exact resulting pack; each listed entry
  MUST match that pack. This deliberately accepts both minimal-delta entries
  and historical hub entries that disclosed the full resulting manifest.
  `removed` is the exact set of paths present in the previous pack and absent
  from the resulting pack for either kind. Both kinds commit the complete
  resulting store through `pack_sha256`; no v1 signed bytes are reinterpreted
  as a partial store.
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

A verifier at the head MUST check the served head state (§7.1's
`headSeq`/`feedHash`) against the last entry's sequence and hash. A full-store
sync or mirror MUST fetch the exact snapshot addressed by that tuple, verify
its pack hash against the signed head's `pack_sha256`, and verify the complete
resulting path/hash/byte manifest before making any destination mutation
visible. It MUST persist the accepted `(headSeq, feedHash)` checkpoint outside
the store and reject a lower sequence or a different hash at the same
sequence. Export must never silently advance to a newer head.

The HTTP binding for an immutable full-store export is:

```
GET /api/hub/brains/<brain>/export?format=pack&atSeq=<headSeq>&feedHash=<feedHash>
```

The response MUST echo the same brain, sequence, feed hash, and signed pack
hash or fail with a conflict. The sole empty-head address is
`atSeq=0&feedHash=none`; it carries no pack URL.

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

### 5.7 Wire profile v2: permissioned incremental commits

Profile v2 keeps the signed, append-only feed but replaces whole-store packs
as the write primitive. A commit still names one complete logical brain state;
the physical objects written for it are only the changed blobs, changed tree
nodes, changeset, actor claim, commit, receipt, and recovery delta. This is the
same separation Git makes between a commit's complete tree and its reused
content-addressed objects, with server-enforced company permissions added.

#### 5.7.1 Canonical bytes and addresses

Every v2 JSON object is UTF-8, has exactly one trailing LF, sorts object keys by
their UTF-8 bytes, preserves array order, adds no whitespace, and permits only
`null`, booleans, strings, arrays, objects, and safe integers. A domain address
is:

```
sha256("link.md\0" || <domain> || "\0" || <canonical-bytes>)
```

Blobs alone use SHA-256 over their exact raw bytes. Implementations MUST reject
non-canonical encodings rather than normalize signed or addressed input.

The v2 content root is a persistent 16-way hash-array mapped trie. Each path
component is a leaf with `{name, kind, child_hash, bytes, nonce}`; its route is
SHA-256 over the component. The independently random 128-bit leaf nonce changes
whenever that entry state changes, so a disclosed sibling/subtree hash is not a
dictionary oracle for a guessed name or content. Branches expose slot and child
hash only; the remaining route-prefix shape is documented structural leakage.
This gives compact inclusion/non-inclusion proofs without revealing sibling
names to a scoped reader. A commit reuses every unchanged
blob and tree node. `null` is the sole empty content root. Portable paths MUST
be NFC, relative, slash-separated, at most 1,024 UTF-8 bytes and 255 bytes per
component, and free of dot components, control characters, Windows device
names, trailing dots/spaces, file/directory collisions, and Unicode/case-folded
portable aliases.

#### 5.7.2 Mutations

A changeset has `v:2`, a stable caller-generated `mutation_id`, an optional
reason, and a deterministically sorted, non-overlapping operation list:

- `put(path, expected, blob, bytes)` creates or replaces a file;
- `delete(path, expected-blob)` removes a file;
- `rename(from, expected_from, to, expected_to, blob, bytes)` moves one exact
  byte version without inferring identity from a later path;
- `restore(path, source_seq, source_commit, source_path, expected, blob,
  bytes)` restores exact authorized historical bytes;
- `withdraw_from_hosting(path, expected-blob, reason)` removes the hosted
  coordinate while recording that stronger lifecycle intent.

Every touched coordinate carries an exact precondition: `{kind:"absent"}` or
`{kind:"blob",hash:<sha256>}`. Omitting preconditions is invalid. A hub applies
the whole changeset or none of it. If the base is stale, `rebase:"disjoint"`
MAY apply it only when every touched coordinate still satisfies its
precondition; `strict` requires the exact head. A request whose authorized
desired bytes are already current converges as a no-op. Any other same-path or
rename-destination mismatch returns one atomic conflict set. An LLM may propose
new bytes after a conflict, but it is never the concurrency mechanism and the
hub never silently chooses a winner.

`mutation_id` is idempotent per stable principal and brain. Repeating the same
request returns its durable receipt; reusing the ID for different request bytes
MUST fail. Inline changed blobs are bounded; larger changed blobs use expiring,
principal-bound reservations and direct conditional upload. Before commit, the
hub MUST re-read and re-hash every reserved object.

#### 5.7.3 Signed commit and head

The brain key signs the canonical commit object. Its normative fields are:
`v`, `seq`, `ts`, `brain`, `public_key`, `signer_epoch`, `control_revision`,
`op:"changeset"`, `parent_commit`, `parent_root`, `state_root`,
`parent_asset_root`, `asset_root`, `materializer`, `changes_hash`, `actor_ref`,
and `prev_entry_hash`. The Ed25519 signature covers the canonical bytes of
those fields without `sig`; the stored canonical object then adds `sig`. The
commit hash uses domain `v2/commit`; the feed hash is plain SHA-256 of the exact
signed commit bytes. `prev_entry_hash` chains those feed hashes.

`actor_ref` addresses a separately signed actor claim bound to the principal,
credential class, organization/role, grants used, mutation/request IDs,
parent/result roots, counts, rebase result, control revision, and authorization
time. The claim is audit evidence, not authority by itself; the hub MUST resolve
live authority again at the final commit point.

The mutable head is a hub-signed pointer to `{brain, seq, commit_hash,
feed_hash, content_root, asset_root, materializer, signer_epoch,
control_revision, backup_preparation, prior_pointer_hash, signed_at}`. The hub
MUST compare-and-set this pointer against the exact prior pointer. It MUST make
the required recovery delta durable before pointer advancement. A client pins
the brain identity and hub pointer signer, verifies commit signatures and feed
continuity, verifies every manifest proof and fetched blob, checks the head
again after local installation or after its write, and only then advances its
private accepted checkpoint. A lower sequence or different hash at an accepted
sequence is rollback/equivocation.

`materializer` names the deterministic working-copy projection. The reference
`dbmd-projection-v1` transform rebuilds `index.md`/`index.jsonl` from the signed
authoritative files and materializes `assets.jsonl` from the signed asset root.
Those derived files MUST NOT enter the content root or recovery delta. The hub
rebuilds them ephemerally for complete-store validation, and each full/scoped
checkout or export rebuilds them from exactly its readable view. Activating a
different materializer is an explicit administrator migration commit; it never
silently changes how an old commit materializes.

For a self-custodied brain, `POST commits` returns `202
brain_signature_required` after it has staged and validated one exact
candidate. The full-read key holder pages the principal-bound
`GET signing-challenges/<id>` manifest, verifies every proof, checks exact set
equality against the requested post-mutation state, verifies the canonical
changeset/actor/parent/root/materializer bindings, and only then signs the
returned canonical commit bytes. The same mutation is retried with the
challenge id and signature. A scoped writer cannot be given a brain-wide
candidate manifest and therefore uses the encrypted proposal workflow; only a
full-read key holder accepts and signs that proposal.

The authenticated head response also labels the caller's effective content
view as `full` or `scoped` and carries the current actor-specific control
revision. This view descriptor is not part of the brain commit: permissions can
change without changing content. A client MUST include it in both pre-install
and post-commit barriers. Once a checkout pins a scoped view, a different view
kind or control revision requires a new checkout; it MUST NOT silently widen,
narrow, or repurpose the existing directory.

#### 5.7.4 HTTP binding

The reference binding is rooted at
`/api/hub/brains/<brain>/v2/`:

| Method/path | Meaning |
| --- | --- |
| `GET head` | profile negotiation, signed current pointer, brain identity, effective permission view |
| `GET commit?commit=…` | exact canonical signed commit bytes |
| `GET feed?after=…&limit=…` | bounded contiguous commit replay |
| `GET files?commit=…&after=…&limit=…` | current readable manifest plus proofs |
| `GET blob?commit=…&path=…&sha256=…` | one proven, readable blob |
| `POST downloads` | verify bounded manifest proofs and mint short-lived direct blob GET capabilities |
| `POST uploads` | reserve direct transport for changed blobs |
| `POST commits` | atomically authorize, validate, back up, and commit a changeset |
| `GET signing-challenges/<id>?after=…&limit=…` | proof-carried exact candidate for self-custody verification/signing |
| `GET proposals?state=…&after=…&limit=…` | list permission-filtered proposal envelopes without changed bytes |
| `GET proposals/<id>` | return one verified proposal descriptor and signed submission claim |
| `GET proposals/<id>/blob?sha256=…` | return one exact declared proposal blob after use-time authorization |
| `DELETE proposals/<id>` | reject one pending proposal with an idempotent audited decision |
| `GET history`, `GET history/blob`, `GET diff`, `GET trash` | permission-filtered history and recovery reads |
| `GET/POST grants`, `DELETE grants/<id>`, `GET/PATCH policy` | v2 authority and company policy control plane |

A caller may send `proposal_only:true` to `POST commits`; a self-custodied
scoped writer is also routed there automatically because it cannot verify or
sign a brain-wide candidate. The hub authorizes the operations, encrypts the
canonical descriptor and each new blob under brain/proposal-specific keys,
writes independent retention backup copies, and returns a fixed-shape `202
proposal_queued` own-effect receipt. The receipt exposes no head, root, or
unrelated path.

The hub signs an immutable proposal-submission claim over the proposer actor
root, brain/proposal/mutation IDs, encrypted-payload address, clear descriptor
digest, control revision, and submission time. Review clients MUST verify the
claim's canonical bytes, domain-separated address, Ed25519 signature, pinned
hub signer, exact descriptor digest, sorted unique blob declarations, and
origin-bound blob endpoints before displaying or accepting it. Changed blobs
MUST be fetched one at a time and checked against their declared length and
SHA-256.

Exact acceptance re-submits the proposal operations to `POST commits` with
`proposal_id` and `proposal_mode:"exact"`. The hub rechecks reviewer authority,
proposal state/expiry, proposer revocation, every path precondition, protected
semantic effects, and the current head. The accepting commit's signed actor
claim names the accepter and references the signed proposal claim and proposer;
the proposal changes from pending to accepted in the same authority transaction
as pointer CAS. A post-pointer repair may complete only after independently
verifying that exact reference in the immutable committed actor claim. Modified
acceptance is a distinct newly authorized mutation with
`proposal_mode:"modified"`; an agent may construct it, but the agent never
decides concurrency or permissions. Rejection requires the current control
revision, stable decision mutation ID, reason, and independent encrypted
before/after control recovery evidence.

A brain selects exactly one write profile at a time. A v2 brain MUST refuse v1
snapshot writers with a typed negotiation response; a non-empty v1 brain MUST
not accept v2 writes until an explicit, verified bridge commits its initial v2
root. There is no automatic two-head mode.
After a client accepts a v2 checkpoint for a brain reference, a hidden/404 v2
head MUST be treated uniformly as unavailable (revoked, removed, or hidden).
It MUST NOT trigger v1 negotiation or erase/replace local checkout files.

A download window carries the exact inclusion proof returned by `files`, bound
to the requested commit, path, blob hash, and byte count. The hub MUST verify
the proof against the current content root and resolve live read authority for
every path before minting any capability. The request is atomic for authority:
one unauthorized, malformed, duplicate, or mismatched claim yields no URLs.
Capabilities are short-lived and address immutable objects; the client MUST
still rehash every response and complete the final head barrier. This batching
reduces hub authorization traffic without turning a content hash into a bearer
capability or weakening per-principal rate limits.

#### 5.7.5 Local three-way sync

The client stores its accepted remote manifest and its local transfer-policy
state outside the db.md store. For each path it compares baseline, current
remote, and current local hashes. Pull removes a local file only when that path
was baseline-known, the remote deleted it, and the local bytes are unchanged.
Push emits a delete only when the same baseline proves the local absence is an
intentional change. Files excluded by `.sevralocal` are never opened for
transport and their existing remote copies are not implicitly deleted. If a
local-policy relaxation makes a formerly excluded path eligible, ordinary sync
stops for explicit adoption before uploading it.

A permission-scoped manifest is not necessarily a complete db.md store and may
withhold canonical `DB.md`. The client therefore materializes a deterministic,
clearly non-authoritative local `DB.md` plus `.dbmd/view.json` projection
metadata. Neither file is remote brain data: they are excluded from the
baseline's riding-file set and from every mutation. Editing or removing the
generated `DB.md` fails closed. Canonical schema and semantic validation always
run at the hub against the complete brain, not against the scoped projection.
Local validation rebuilds the view's catalogs and reports links leaving the
view as unresolved-withheld information without asserting that the hidden
target is absent. If the complete-brain gate refuses a candidate, the hub MAY
return structured issues only for paths the actor can currently read; every
issue and related path outside that view MUST be omitted. The client preserves
those safe details under a stable validation-refusal machine code.
Scope widening or narrowing starts a new verified view rather than merging
newly visible or newly forbidden paths into an existing checkout.

The local operation is serialized per brain. A pull uses head H1, verifies the
manifest/blobs, installs atomically, then requires H2 = H1 before saving its
baseline. A push requires the accepted commit still be the current head and
rescans local bytes before reporting clean. Concurrent editor activity is
reported as local-dirty; it never rewrites the remote receipt.

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

Profile v2 retains attenuation, expiry, revocation, and server-side enforcement
and makes the authorization vocabulary explicit. A grant is scoped to one
exact file or path prefix and carries a set of actions such as `read_current`,
`read_history`, `create_record`, `update_record`, `delete_record`,
`append_source`, `withdraw_source`, `append_log`, `curate_conclusion`,
`write_contract`, `manage_assets`, `publish_content`, `replace_scope`,
`bulk_change`, `restore_version`, `propose_change`, `review_proposals`, and
separate administration/audit actions. `propose_change` can enqueue a scoped
candidate but cannot inspect the queue. `review_proposals` is separately
required, together with a full readable view, to list, inspect, accept, or
reject candidates. This separation prevents both a proposer and a content-blind
organization administrator from turning the queue into a read oracle.
History authority includes an issuance boundary. Organization administrators
receive control-plane authority, not implicit content access; org content is
restricted unless an explicit grant or declared all-members content preset
provides it. Protected frontmatter changes are authorized by semantic effect,
not merely by filename. Source evidence is append-only, withdrawal is explicit,
and reuse of a withdrawn source coordinate requires an exact historical
restore. Revocation and hosting denies are rechecked at the final pointer CAS.

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
Authorization: LinkMD-Sig v2,key=<multikey>,ts=<unix-seconds>,sig=<base64url(signature)>
canonical = "v2" LF origin LF method LF path-and-query LF ts LF (sha256hex(body) | "-")
```

The hub verifies the signature with the registered public key for `key`,
enforces a ±60 second window on `ts`, and binds the signature to the exact
origin, method, path, and body. `origin` is the externally authoritative
request origin, never a caller-selected forwarding header: lowercase
HTTP(S) scheme and IDNA host, the default port elided and a non-default port
retained, with no path, query, fragment, or trailing slash. `method` is
uppercase. `path-and-query` is the exact request target used on the wire. The
body hash is lowercase hexadecimal SHA-256 of the exact request body bytes, or
the literal `-` when there is no body. There is no trailing LF. A signature
captured at one origin therefore cannot authenticate at another. v1 is
invalid; implementations MUST NOT silently fall back. The private credential
never crosses the wire.

The signed envelope itself is replayable until its timestamp leaves that
window. A hub MUST durably claim each exact signed envelope at most once before
executing a mutating request and MUST fail closed when that replay store is
unavailable. Safe reads (`GET`, `HEAD`, `OPTIONS`) MAY repeat the same envelope.
Clients retrying a refused mutation MUST sign a fresh timestamp. This closes
the gap between proof of key possession and proof that one mutation was
authorized only once.

## 9. Rotation, recovery, and pinning

### 9.1 Rotation

Identity rotation is a **rotation statement**: a declaration binding the new
key and the current feed boundary, signed by the **old** key:

```json
{
  "v": 1,
  "op": "rotate",
  "brain": "<old multikey>",
  "public_key": "<old base64url SPKI>",
  "new_brain": "<new multikey>",
  "new_public_key": "<new base64url SPKI>",
  "prior_head_seq": 41,
  "prior_feed_hash": "<entry hash, or null iff sequence is zero>",
  "ts": "2026-07-30T00:00:00.000Z",
  "sig": "<base64url Ed25519 signature>"
}
```

The field order above is normative. `sig` covers the exact minimal JSON
serialization without `sig`, using that order. A hub accepts the statement
only when `(prior_head_seq, prior_feed_hash)` atomically equals its current
head, persists the exact statement, and serves the oldest-first chain with the
brain card. A verifier that trusts fingerprint F and verifies a connected
F→F′ statement SHOULD trust F′ and treat F as historical. Chains verify
transitively.

The old key may sign feed entries only through its statement's
`prior_head_seq`; the new key may sign only later entries. Rotation boundaries
MUST be monotonic, and every non-empty `prior_feed_hash` MUST match the stored
entry bytes at that sequence. A card's unproved `previous` list is never
authority by itself. These epoch rules preserve historical verification
without letting a retired key append after rotation.

### 9.2 Recovery

- Custodied brains: the custodian rotates on the owner's authenticated
  request — recovery is an account concern, not a protocol one.
- Self-held brains: recovery is the exported identity file. A lost self-held
  key without a rotation signed by it is a lost identity; parties who pinned
  it must re-verify out of band. This spec deliberately defines no social
  recovery.

### 9.3 Pinning

A client that has synced a brain SHOULD pin its fingerprint together with the
accepted `(headSeq, feedHash)` in local toolkit state (outside the store) and
MUST refuse subsequent responses presenting a different identity absent a
valid old-key-signed rotation chain from that pin. Every authenticated verb
uses the same pin and rotation policy. A successful advancement writes the
new identity and checkpoint only after the exact response has verified and
the destination transition can commit atomically. First contact is
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
| E5 | Public registries + cross-hub handle resolution *(v0 implemented — the resolver + proof-of-control registration; name-dispute governance is policy)* |
| E6 | Key facets beyond ed25519 *(secp256k1/npub implemented hub-side — the hub verifies BIP340 and grants accept npubs)* |

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
