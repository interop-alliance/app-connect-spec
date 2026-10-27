# 0001: byoe service entries dispatch on type, not fragment

- Status: accepted
- Date: 2026-08-19
- Driving work: the public-computer posture redesign for the browser
  wallet -- per-visit transient clients recorded in a disposable
  companion did:webvh, referenced from the account document, with the
  generation's standing delegation carried inside the companion
  document itself. Amended by the design that lets every session kind
  register a replica server, which adds a replica pointer to the
  companion document.
- Affects: this spec's companion profile (the service-entry
  vocabulary), the byoe-context term definitions
  (`https://w3id.org/byoe#DelegatedClients`,
  `https://w3id.org/byoe#GenerationDelegation`), the pws context term
  definition (`https://w3id.org/pws#ReplicaServer`), wallet-core (the
  entry builders and readers, and the restate in every path that mints
  a client-annex generation), was-teaching-server (the
  authorization-side pointer read, and the replica pointer read of
  decision 0003's predicate 6), and both wallets.

## Context

The companion (delegated clients) mechanism needs two pieces of state carried in
did:webvh documents: the account document must point at its current companion
DID (the transient-members extension), and the companion document must
carry the generation's standing delegation where an enrolling
transient client can reach it before it holds any other authority.
did:webvh constrains the options: `updateDID` clones prior state under
a closed overlay allowlist, so a custom top-level document property is
genesis-only and immutable, while the pointer must be re-pointed at
every garbage-collection swap. DID `service` entries are first-class,
typed, and updatable, so both pieces ride as service entries. That
forces a convention for how readers identify them.

Replication adds a third piece of state. A ladder VM may sign a pull
grant to a replica server only while a record names that server
(decision 0003, predicate 6). The record has to be readable by the
source server when it verifies a pull, updatable as replica servers
come and go, and kept from the world at large. The companion document
(the client-annex document) meets all three. The source server already
reaches it through the account document's pointer. Its log takes
ordinary update entries. Its auxiliary Space is capability-gated. It is
mortal, though. A garbage-collection swap starts a fresh generation
from a bare genesis, so whatever the annex carries must be restated
into each fresh generation.

## Decision

byoe service entries are identified by their `type` IRI, never by
their fragment id. Fragment ids are non-semantic: any stable
identifier (random or content-derived) is valid, readers MUST dispatch
on the `type` IRI, and the profile states this rule. The principle is
uniform across all byoe service entries, present and future.

The rule covers every service entry this profile defines, whatever
namespace its `type` IRI sits in. Three entries are defined under it:

- The transient-members pointer, in the account document. `type` is
  `https://w3id.org/byoe#DelegatedClients`; `serviceEndpoint` is the
  companion DID string. The GC ceremony re-points it per generation
  with an ordinary document update entry.
- The generation delegation, in the companion document. `type` is
  `https://w3id.org/byoe#GenerationDelegation`; `serviceEndpoint` is
  the full delegated-zcap JSON as a single map (DID-spec-conformant),
  byte-identical to what `zcapClient.delegate` produces. It is
  installed by the entry that publishes the generation's first
  transient verification method, never by genesis: the zcap's
  `controller` embeds the SCID, SCID verification recomputes the
  genesis hash with every SCID occurrence swapped back to the
  placeholder, and the zcap's `proofValue` signs the real-SCID form,
  so a genesis-embedded signed zcap can never verify.
- The replica pointer, in the client-annex document, not the account
  document. `type` is `https://w3id.org/pws#ReplicaServer`. The term is
  defined in the pws context, since replication is the PWS companion
  spec's domain. There is one entry per replica server.
  `serviceEndpoint` is the replica server's bare did:webvh string,
  whose path is `space:server:id`. The id's fragment is
  `#replica-<multihash>`, where `<multihash>` is the SHA-256 multihash
  of the `serviceEndpoint` string (the replica server's DID, as UTF-8
  bytes), base64url with no padding. The id is therefore the same on
  every restate. It carries nothing beyond `serviceEndpoint`. A reader
  still dispatches on `type`, and among the replica pointers it may
  match one server's entry by id or by endpoint. The wallet writes the
  entry for every replica server the account registers, whichever
  signer mints the pull grant.
  The source server gates only a ladder-signed pull on it (decision
  0003, predicate 6). For a pull grant an enrolled client signs, the
  entry is the record and the basis of the wallet's anomaly check.

The install rule, extended to the replica pointers. Every path that
mints a fresh client-annex generation installs the generation
delegation in one entry: the swap's install stage, or the entry that
publishes the generation's first transient verification method. That
installing entry also carries every replica pointer the restate
predicate selects. Genesis stays bare, carrying neither the delegation
nor any pointer.

The restate predicate selects a replica Space that the wallet's
connections directory records as not stopped, on an entry not
declined, with a recorded pull grant whose proof verifies as the
account's. A registration torn before its pull grant is recorded is
therefore never restated, and neither is a planted entry carrying no
wallet-signed grant. The directory is the source of record. Where each
path reads the pointers from:

- A swap run by an enrolled client restates from the directory. The
  superseded generation's pointers serve only as a cross-check, so a
  pointer with no directory entry is not carried forward.
- A fresh mint signed by a ladder runs before the session holds the
  user key, so it cannot read the directory. On the arm where the
  superseded generation's log is still readable (the uncommitted-rung
  arm, which already reads that log for the rung restate), the
  installing entry carries that document's pointers forward. The
  wallet's replica pass then reconciles them against the directory and
  drops any pointer with no verified entry.
- On the arms where nothing is readable (a dead generation, or a
  missing auxiliary Space), the replica pass appends the pointers right
  after the session assembles.

## Rejected Alternatives

- Semantic fragments (`#generation-delegation`, the initially
  chosen spelling, retrofitted when the principle was made uniform):
  fragments drift and invite readers to key on them; the type IRI is
  the contract.
- For the pointer, `alsoKnownAs`: it asserts identifier equivalence,
  a claim readers may act on, where this is a typed reference to a
  sibling document.
- For the pointer, a dedicated top-level document property: under
  `updateDID`'s closed overlay allowlist it is genesis-only and
  immutable, while GC must re-point per generation.
- For the pointer's endpoint, a URL: it bakes in the host, which the
  account pointer already carries; the DID string is self-certifying
  and host-independent.
- For the delegation, a sixth top-level log-entry member: forks the
  closed `DIDLogEntry` shape, the entry-hash preimage, and the
  server's generalized webvh parsing.
- For the delegation, a base64url `data:` URL endpoint: opacifies the
  delegation at exactly the record whose purpose is inspectability,
  and forces an encoding collision -- a conformant `data:` URL wants
  standard padded base64, against the house base64url-nopad
  preference.
- For the replica pointer, the account document. Rejected 2026-10-24
  by the profile's editor, do-not-reopen: "Broadcasting to the world at
  large which other servers a space is replicated to is unacceptable."
  The account document is world-readable and its log is permanent.
- For the replica pointer, the annex genesis. The pointer names the
  replica server's DID and not the SCID, so unlike the delegation it
  could ride the genesis. That breaks the bare-genesis convention every
  annex reader relies on, for no gain: the installing entry lands
  before the account document re-points at the generation either way.
- For the replica pointer's disclosure, a blinded endpoint (an HMAC of
  the DID). It hides the pointers from the other replica servers alone,
  since the source server needs the key to check the pull predicate.
- For the replica pointer's disclosure, a per-server filtered copy of
  the annex. It breaks the annex log's chain hash.

## Consequences

- Readers need no fragment expectations to dispatch; adding a byoe
  service entry never risks a fragment collision or a reader keying on
  a fragment string. The replica pointer's derived fragment only lets a
  reader find one server's entry by id as well as by endpoint.
- The companion entry proof (JCS canonicalization) covers the
  embedded delegation map byte-for-byte, so host tampering with the
  stored delegation is client-visible; the server's own chain
  verification already made it authority-safe.
- webvh restates full document state per entry, so the roughly 1.2KB
  delegation repeats across the generation's few entries -- the
  priced cost of the free tamper evidence.
- A generation with no visits never carries a delegation (it is
  installed with the first transient VM), so an orphan generation is
  authorization-inert.
- The replica pointers are disclosed to every reader of the annex. The
  annex is capability-gated. Its readers are the account's own
  sessions, the source host, and each replica server once the auxiliary
  Space is copied. Each of them learns every replica server's DID. This
  is accepted. The source host learns each pulling server's DID from
  the pull invocations anyway.
- The replica pointer is mortal. Removing it, or a swap that does not
  restate it, ends that server's ladder-signed pull of the account
  Space. Its pull of the auxiliary Space still verifies through
  decision 0003's predicate 2 second branch, which admits a GET of a
  bookkeeping Space to any grantee, until that branch is narrowed.
  Revocation at the auxiliary Space is that pull's only stop. A missed
  restate stops the account-Space copy until the replica pass appends the pointer,
  which runs on both login chains, so a credential-only visit converges
  it. In a CHAPI popup the pass does not run, and the window lasts
  until the next full login.
- On the uncommitted-rung arm a pointer with no directory entry, such
  as one a credential thief appended, survives one carry-forward, until
  the replica pass drops it.
- A collected generation takes its pointer entries with it, so a
  hostile pointer the owner never saw leaves no evidence after the next
  swap. The swap does not carry it forward either.

## Revisit Criteria

Reopen this decision when one or more of the following holds:

1. A DID-spec revision or a consuming resolver rejects a map-valued
   `serviceEndpoint`, forcing a different carrier for the embedded
   delegation.
2. A byoe service entry acquires a reader that structurally cannot
   dispatch on `type` (for example, a registry keyed by fragment),
   which would force fragment semantics back in as a parallel
   convention, not a retrofit of existing entries.
3. A profile change makes the client-annex log or its auxiliary Space
   world-readable. The replica pointer's disclosure was accepted for a
   capability-gated audience alone, so its placement must then be
   reopened. The account document stays rejected.

## Changelog

- 2026-10-26: a third entry is added, the replica pointer
  (`https://w3id.org/pws#ReplicaServer`) on the client-annex document,
  with the restate rule that carries the pointers into each fresh
  generation's installing entry. This is an addition. The dispatch rule
  and the two existing entries are unchanged. The account-document
  placement, the genesis placement, and two ways of hiding the pointers
  are recorded as rejected. Driven by the design that lets every
  session kind register a replica server.
- 2026-10-26: the replica pointer's fragment is fixed as
  `#replica-<multihash>`, the base64url (no padding) SHA-256 multihash
  of the replica server's DID, so the id is stable across restates. It
  was previously non-semantic. Dispatch on `type` is unchanged. The
  pointer's mortality is restated: its removal ends the ladder-signed
  pull of the account Space only, since decision 0003's predicate 2
  second branch still admits the auxiliary-Space pull.
