# 0003: The ladder VM's authority clauses

- Status: accepted
- Date: 2026-08-19
- Amendments: 2026-08-19: clause A predicates tightened after
  adversarial review of the reference-server inspector (own-log
  bridge target, bookkeeping-Space typing plus its Create Space rule,
  subtree-only Space target made explicit, sole-controller and
  relationship-id normalization). 2026-08-22: shape 2 of clause B's
  license gains a per-entry refinement to the one-shot check.
  2026-09-01: clause A gains a third predicate (target-exact,
  single-verb ladder delegations on a bare Space URL) and the locked
  property is restated to cover it. 2026-09-01: clause A's predicate 1
  is narrowed by a target bound (the account Space's items subtree)
  and an action bound (the closed WAS verb vocabulary). 2026-09-03:
  clause B gains a third enumerated shape (a rotation carrying a version
  whose enrolled-client set changed, whose entry a rung of the appending
  ladder signed). 2026-09-07: clause B's scope is stated: it binds the
  user key roster log alone, and a per-collection encryption descriptor
  log admits a ladder-signed append on `assertionMethod` membership
  alone. 2026-09-27: clause A is transcribed from the reference server's
  shipped inspector. Predicates 1 to 3 are restated for the WAS v0.5
  route table and two invocation-time bounds are added; predicate 2's
  bookkeeping branch gains POST; predicate 4 (a target-exact GET of one
  Resource, from WAS-109 and freewallet FW-525) and predicate 5 (a
  target-exact POST over a delegated management capability, from
  WAS-151) are added. The locked property is restated to cover them.
- Driving work: the public-computer posture redesign for the browser
  wallet -- an account with zero enrolled durable clients is anchored
  by a ladder-derived verification method in its document, and that
  VM's authority had to be bounded so a phished unlock credential (or
  a storage host grinding the unlock record offline) cannot exercise
  silent authority during the ladder-anchored window.
- Affects: this spec's companion profile (clause A's normative text
  and its fail-open note) and Resource Log Profile (clause B, a new
  subsection beside "The sealing append"); was-teaching-server (the
  second `inspectCapabilityChain` inspector); wallet-core (the
  resource-log verification and the `ResourceLogLicenseError` class);
  both wallets.

## Context

The ladder VM is the stable document-visible verification method
derived from a standing unlock credential's random ladder seed. Its
life is keyed to its credential rather than to the account's client
census: it is installed when the credential becomes standing, it stands
for as long as that credential does, and it is struck only at that
credential's retirement -- on accounts with enrolled clients and on
accounts without them alike. It is recognized by relation asymmetry: a
`capabilityDelegation` member absent from `capabilityInvocation`
(relationship entries compared as absolute method ids -- DID Core also
permits embedded objects and relative references in a log-verified
document). It must be able to sign the generation delegation (under `capabilityDelegation`) and roster
appends (under `assertionMethod`), or a ladder-anchored account is
inoperable. Unbounded, those same relations grant three silent powers:
delegating a Space-scoped zcap directly to an attacker-held key with
no log entry anywhere; appending a roster rotation that rekeys the
account to recipients of the attacker's choosing; and turning a
successful offline grind of the unlock record from
"yields a loud self-enroll" into "yields standing silent authority".
The system's security stance is detect-and-remediate, which requires
that every exercise of credential-derived authority first extend an
auditable log.

## Decision

Two normative clauses, one per authority axis.

Clause A, the delegation axis (server-side, a second
`inspectCapabilityChain` inspector beside the existing revocation
one; no verification-library changes). A delegation whose proof VM
resolves to the ladder VM is admitted iff one of five predicates
holds. Every target must be a URL on the server's own origin with no
query or fragment. Predicates 2 to 5 compare it by exact string
equality against the canonical URL of their shape, so an alternate
encoding of the same path never passes; predicate 1 checks path
segments instead. Under the WAS
v0.5 route table a Space is addressed by its trailing-slash container
URL. The no-slash form only redirects, and no predicate matches it.

1. Companion-DID controller inside the account Space's items subtree
   (narrowed 2026-09-01, restated 2026-09-27 for the v0.5 route
   table). Three bounds hold together.

   Grantee, by pointer equality: the delegation's sole `controller`
   equals the companion DID named by the
   `https://w3id.org/byoe#DelegatedClients` service entry of the
   account document the chain already resolved as delegator (zero
   extra I/O), behind the syntactic gate that the string parses as a
   self-hosted did:webvh. `controller` is read after normalizing the
   zcap array form; two or more entries are refused, since a second
   controller could invoke past the pointer. A GC pointer swap
   thereby instantly kills the prior generation's delegations.

   Target: the `invocationTarget` is the trailing-slash URL of the
   Space carrying the delegator DID's own history log -- the account
   Space -- or a path under that URL. The Space Metadata URL
   `/space/<S>/meta` and every path under it are refused, since a
   `PUT` there rewrites the Space's controller. Keystore targets
   (`/kms/...`) are refused as well; they lie outside the subtree by
   path shape. Under v0.5 the subtree itself contains Update Space
   Metadata and Delete Space (`DELETE /space/<S>/`), so a
   whole-subtree grant reaches both by target attenuation. The
   invocation-time bounds below hold them out of ladder reach.

   Action: `allowedAction` is present, non-empty, and a subset of the
   full closed WAS verb vocabulary {GET, HEAD, POST, PUT, DELETE}.
   The whole vocabulary is admitted rather than a chosen subset,
   because the generation delegation (decision 0002) carries exactly
   it and a child capability may not exceed its parent -- so any verb
   left out here would hold every transient-visit grant below the
   durable client's own shape. The action bound is there to refuse an
   absent or open `allowedAction` and any verb outside the protocol.
   The target bound does the narrowing.

2. Bridge-shaped target, two-branch, capability-side: the
   delegation's `invocationTarget` equals the delegator account's own
   history log resource URL -- derived from the account DID, which
   carries its log's Space and Collection; no other log-shaped target
   qualifies -- with `allowedAction` a subset of {PUT}; or equals the
   trailing-slash URL of a Space whose Space Metadata object declares
   it delegated-clients bookkeeping (`type` names both
   `AuxiliarySpace` and `DelegatedClientsSpace`; Create Space refuses
   the latter without the former, so a listed data Space cannot carry
   the subtype), with `allowedAction` a subset of {GET, PUT, POST}.
   POST (added 2026-09-27) reaches the Space's export and import
   endpoints, which the wallet's backup export invokes. By
   attenuation it also reaches Create Collection, Create Resource on
   each Collection container, backend registration, and revocation
   submission under that Space. The creates rewrite nothing standing,
   and Update Space Metadata stays controller-only. What the
   revocation reach adds is open (freewallet FW-564). Only the trailing-slash
   form is admitted, so wallets pass it explicitly when granting. The
   second branch costs one memoized Space Metadata read.

3. Target-exact single-verb Space delete or Space Metadata read, both
   chain lengths (added 2026-09-01, restated 2026-09-27 for the v0.5
   route table), split by verb. DELETE branch: the delegation's
   `invocationTarget` is the trailing-slash Space URL, equal to the
   parent's `invocationTarget` unchanged, and its `allowedAction` is
   exactly `['DELETE']`. The parent is a delegated capability or the
   Space's synthesized root, whose own target is that same URL. GET
   branch: the `invocationTarget` is the Space Metadata URL
   `/space/<S>/meta`, its `allowedAction` is exactly `['GET']`, and
   the parent's `invocationTarget` is either that same Metadata URL or
   the Space's trailing-slash URL. The child narrows down to the one
   read and cannot widen past its parent. A two-verb set qualifies on
   neither branch. This is the account-deletion ceremony's admission
   path. The ceremony's delegatee and invoker is the ladder VM's own
   bare did:key, which predicate 1 never admitted; the predicate
   itself puts no condition on the child's `controller`. Two bounds keep it
   narrow: on the management-capability arm the parent already
   carries DELETE on exactly that Space URL, so the predicate widens
   who signs the last link rather than what the account may do; and
   the target rule means the ladder VM cannot aim the child anywhere
   new.

4. Target-exact single-verb read of one Resource (added 2026-09-27;
   shipped by the reference server 2026-09-14). The delegation's
   `invocationTarget` is a Resource URL `/space/<S>/<C>/<R>`: three
   URL-safe segments, with `<C>` and `<R>` outside the reserved
   path-segment registry, so a Collection Metadata object, a policy,
   or a query endpoint is not a Resource here. Its `allowedAction` is
   exactly `['GET']`, and the parent's `invocationTarget` is either
   that same Resource URL or the trailing-slash URL of the Resource's
   Space. The parent may be a delegated capability or the Space's
   synthesized root. This is the read a transient wallet session
   mints of one record, the keyring record of a sibling unlock Space,
   under the management capability that Space's controller delegated
   to the account. Predicate 3's GET branch covered that read on the
   v0.4 route table, and its v0.5 narrowing to the Space Metadata
   object removed it. No Space kind is recognized, since the parent
   already bounds which Space the read can target. By attenuation the
   grant also reaches the reads beneath that Resource URL (its
   `/meta`, its `/policy`, its chunks), all of them reads. The
   delegator's own history log is a Resource like any other here:
   predicate 2's {PUT} bound says what the bridge shape may write, not
   that the log is closed to reads.

5. Target-exact single-verb POST over a delegated management
   capability (added 2026-09-27; decided by the reference server
   2026-09-20 and shipped 2026-09-27). The delegation's `invocationTarget` is the
   trailing-slash Space URL, equal to the parent's `invocationTarget`
   unchanged, and its `allowedAction` is exactly `['POST']`. The
   parent must be a delegated capability; the Space's synthesized root
   does not qualify. Two further bounds make the parent the
   management capability the Space's controller granted the account.
   Its sole `controller` is the delegator account, the DID whose
   document publishes the ladder VM. And the DID of the verification
   method that verified the parent's own delegation proof is the
   Space's stored controller (one memoized Space Metadata
   read), so the parent's chain roots at that Space's root. This is
   the shape a transient wallet session mints from a sibling unlock
   Space's management capability to invoke Export Space on it. By
   attenuation it also reaches that Space's import endpoint, Create
   Collection, Create Resource, backend registration, and revocation
   submission. None of the creates rewrites a record already standing
   in the Space: import skips every id the Space already holds, and
   the others refuse an existing id. The predicate widens who signs
   the last link of a grant the account already holds. It reaches no
   Space the account held no management capability on.

Invocation-time bounds (added 2026-09-27; shipped by the reference
server 2026-09-12 and 2026-09-14). Under v0.5, Update Space Metadata
(`PUT /space/<S>/meta`) and Delete Space (`DELETE /space/<S>/`) lie
inside the subtree predicates 1 and 2 grant, so a delegation's own
target no longer holds them out of reach by path alone. Two bounds
apply at invocation, read against the operation being verified. A
verification with no invocation to read gets the delegation
predicates alone. The reference server's revocation route is one,
and its target is never a Space URL or a Space Metadata URL.

- Ladder bound. A chain carrying any ladder-signed link is refused
  when invoked as `PUT` on a Space Metadata URL. Invoked as `DELETE`
  on a trailing-slash Space URL, it is refused unless every
  ladder-signed link in the chain is itself predicate 3's DELETE
  shape for that Space: target exactly that URL, `allowedAction`
  exactly `['DELETE']`. The bound reads every ladder-signed link
  rather than the chain's tail. A holder below a ladder-signed
  subtree grant, such as the companion VM, can narrow its own child
  into the DELETE-only shape by ordinary attenuation, and a tail-only
  check would read that child as a predicate 3 grant.
- Transient-signer bound. A `DELETE` on a trailing-slash Space URL is
  refused when any link in the chain is signed by a transient
  companion VM, one published under `capabilityInvocation` and
  `capabilityDelegation` and under no other relation, whoever signed
  the links above it. It keys on a different signer than the rest of
  the clause. A generation delegation is signed by an enrolled
  client, so its chain carries no ladder link, and without this bound
  a transient VM holding one could narrow it into a DELETE-only child
  and delete the Space. The signer is recognized from its own
  document alone, so the bound also holds for a retired generation
  whose grant is still live. A DELETE-only child an enrolled client
  signs to the companion DID stays admitted.

The reference server also refuses every delegated `PUT` on an
existing Space's Metadata URL at the route, before the clause runs. A
create-by-id `PUT` on an absent Space rewrites no controller. The ladder bound's
PUT branch stays normative all the same, so the locked property does
not rest on a route rule outside this clause.

Failure semantics: the clause binds the capability decision only. A
delegation failing it MUST NOT be treated as authorizing the request;
the refusal falls through to the server's access-control policy (a
world-readable read still serves; writes and private reads have no
policy fallback, so loudness is unaffected). The clause is normative
in the companion profile with an explicit fail-open note: a server
running unmodified verification accepts exactly what the clause
refuses. The answer is conformance discovery plus a wallet-side rule:
a ladder VM is published only on a host advertising the profile.

Clause B, the roster axis (client-side, enforced in wallet-core's
resource-log verification): the ceremony-tail license on
ladder-signed roster appends. Define S(V), the credential-posture key
set at document version V: the `keyAgreement` verification methods
whose `controller` equals the account DID (the deliberately unmarked
credential entries, `Multikey` and `MultikeyCommitment` alike), union
the ladder VMs. An entry is posture-changing iff S(V) differs from
S(V-1), in either direction; ordinary client enrollment and
revocation are excluded structurally, because a client's
`keyAgreement` twin carries the `did:key` controller marker. A
ladder-signed roster append is accepted in exactly three shapes:

1. A roster's first entry -- creation, never extension.
2. A rotation carrying a posture-changing document version, and
   one-shot: refused when the verified roster head already contains
   an entry carrying V or later. Comparison is by position in the
   controller's verified version history
   (`headControllerVersionIndex >= indexOf(V)`, the structural twin of
   the shipped sealing check). Refinement: the one-shot is evaluated
   per entry, over the entry's set of signing keys. At most one of an
   entry's proofs may be by a ladder key; a rotation co-signed by a
   member stays licensed.
3. A rotation carrying a document version V whose enrolled-client set
   differs from version V-1's, in either direction, AND whose entry was
   signed by a rung of the same ladder that signs the append (added
   2026-09-03). The enrolled-client set is the `capabilityInvocation`
   verification methods of version V, equivalently the `keyAgreement`
   twins carrying the `did:key` controller marker. One-shot per version
   under the same head comparison as shape 2, and under the same
   single-ladder-proof refinement.

   The signer conjunct is what keeps this shape out of the rejected
   any-`keyAgreement`-change predicate. A client's own enrollment or
   revocation entry is client-signed, so it mints no ladder shot, and
   the class the license exists to exclude stays excluded. A shot is
   minted only by the ladder holder's own world-readable act of
   enrolling or removing a client, and only for that ladder. Without the
   conjunct the owner's ordinary enrollment of a phone from a remembered
   session would mint a shot. A phished credential's ladder could then
   spend that shot on a silent rekey to recipients of its own.

   Shape 3 is a verifier-side rule on an append-only log. A reader
   without it refuses the whole roster log. The license throws from the
   admission hook, and the verifier propagates that throw as a whole-log
   refusal. Rollout is therefore verifier-first. Every roster-log reader
   ships shape 3 before any writer emits one. The readers are
   wallet-core's license module and the controller-inventory member it
   reads, both wallets, and any consumer of the
   `@interop/vh-resource-log` library. The encrypted-collections-spec
   parties table is walked beside this spec's, that profile being the
   Resource Log Profile's home.

Everything else -- above all a rotation against an unchanged
document, the silent-rekey shape -- is refused by every verifier.
The refusal is a write-time admission error, a new named class
`ResourceLogLicenseError` beside `ResourceLogIntegrityError` and
`ResourceLogContinuityError`: retryable after a posture-changing
entry, not log corruption, and the profile's reject-the-whole-log
severity does not apply.

Scope (stated 2026-09-07). Clause B binds the user key roster log, the
log that governs the account's root key, and no other resource log. A
per-collection encryption descriptor log (encrypted-collections-spec's
log form, one log per governed collection) admits a ladder-signed
append on `assertionMethod` membership at the anchored version alone,
with no shape check and no one-shot. The silent-rekey shape the license
exists to refuse is a rotation of the root key to recipients of a
credential thief's choosing, invisible in every log. A descriptor
append escrows one recipient into one collection, and lands as a
hash-chained entry signed by that credential's ladder VM, auditable by
the account's clients and attributable to the credential. Before the
log form the same act was an unsigned Collection Description write
under the generation delegation, so the governed form is louder than
what it replaces, and a credential holder wanting more reach has the
louder self-enrollment path already. The bound this leaves is
detect-and-remediate: a recipient escrowed this way shows on the
wallet's shared-collection listing, and credential rotation retires the
signer, whose struck ladder VM then seals every collection log through
the rotation's own full-state appends. Implementations parameterize the
admission hook by log class rather than exempting per call site, so a
new log class states its rule when it is introduced. Driven by the
browser wallet's FW-134 (log-governed descriptors) and the rule that a
transient session runs every ceremony a remembered one does.

The locked property across both clauses, restated 2026-09-01,
2026-09-03, and 2026-09-27: every ladder delegation either needs a
loud companion entry to resolve and stays inside the account Space's
items subtree; can only write a log or reach a Space declared
delegated-clients bookkeeping; is a target-exact single-verb DELETE of
one Space or GET of one Space Metadata object, under a parent the
account already holds on that Space; is a target-exact GET of one
Resource; or is a target-exact
POST under a management capability the Space's own controller granted
the account. That is a read, a destruction whose account-Space case
removes the log any record would live in and leaves no reader to
remediate, or a create that rewrites nothing standing, with
revocation submission open as noted under predicate 2. No ladder-descended chain rewrites a Space's
controller, and none deletes a Space unless every ladder-signed link
in it is the target-exact DELETE shape for that Space. Every ladder
roster append carries the version of a loud document event, either an
inventory-changing one (the "posture-changing" of clause B above) or a
change to the enrolled-client set that the same ladder signed. A
ladder-signed append to a per-collection descriptor log is outside
this property and stands on `assertionMethod` membership alone, per
the scope paragraph above. Predicate 1's target bound closes one gap
the grantee-keyed form left open: a companion VM that also holds
`capabilityDelegation` can mint onward grants no companion entry
records, and every such grant is a child of the admitted delegation.
None of them reaches a keystore, and the ladder bound holds Update
Space Metadata and Delete Space out of reach of every one of them. The
prior absolute form ("no ladder authority whose exercise leaves no
record") is superseded: a DELETE admitted under predicate 3 leaves no
record, and that carve-out is the account-deletion design's stated
trade. A POST admitted under predicate 5 writes no log either. Its
record is the Space controller's grant of the parent management
capability, which the wallet's own unlock record carries.

## Rejected Alternatives

- Unrestricted `capabilityDelegation` on the ladder VM: silent grant
  authority -- a phished credential, or a host that grinds the record
  offline, could delegate directly to an attacker key with no record
  anywhere.
- A conformance-required wallet-written delegation log: voluntary
  loudness binds only conformant parties; the adversary is not one.
- Widening clause A for rotation support: rotation's blocker was the
  roster axis, so a wider delegation clause is authority without a
  consumer.
- An unconstrained ladder-delegation predicate (no target or verb
  bound), and a server rule admitting `capabilityDelegation` members
  as direct root invokers of a Space DELETE: both rejected in the
  transient account-deletion design (its wallet-side record holds the
  do-not-reopen); predicate 3 is the bounded form that stands.
- A narrower verb set on predicate 1 (for instance dropping DELETE, or
  admitting reads alone), considered and rejected 2026-09-01: the
  generation delegation carries the full vocabulary, and a child
  capability may not exceed its parent, so every omitted verb would
  hold each transient-visit grant below the durable client's own
  shape while adding no bound the target test does not already give.
- Syntactic-only controller matching (any self-hosted did:webvh
  qualifies): rests loudness solely on invocation-time companion-log
  membership.
- Tightening the controller predicate to the generation-collection
  spelling: bakes a naming convention into the server clause.
- The Space-type check as the controller test: one extra read for
  less than pointer equality gives.
- Path-shape-only target matching in the bridge branch: any
  whole-Space subtree target would pass, so the ladder VM could be
  handed the account Space wholesale.
- A fixed `id` Collection for the log-target branch: refuses valid
  accounts anchored in another Collection and admits other Spaces'
  log-shaped paths; the delegator DID already names the only log
  that matters.
- Admitting the no-slash Space URL in the bookkeeping branch (the
  client-library default grant target): under target attenuation it
  also covers `PUT <base>/space/<S>`, which can rewrite the Space's
  controller.
- Anchoring the bookkeeping branch to the Space hosting the companion
  DID's log instead of the type check: assumes the bookkeeping Space
  and the companion log's Space coincide, which the profile does not
  require.
- The request-side variant of the target test: bounds the invocation
  rather than the delegation, and needs request context closed into
  the inspector.
- A PUT branch beside predicate 5's POST, for a restore creating a
  sibling unlock Space by id, drafted and withdrawn 2026-09-20 before
  it shipped. Under the prefix rule of target attenuation, a PUT
  child of the trailing-slash Space URL reaches every Resource beneath
  the Space, a sibling unlock Space's keyring record included. A
  transient session holding one credential could overwrite another
  credential's record, and neither the account log nor the companion
  log would carry a trace. That fails loudness, where the POST shape
  only widens who signs a link. A restore's create-by-id stage waits
  on a bounded create-only shape instead.
- Hard invocation rejection on clause failure: a behavioral change to
  the policy fallback with no security gain on world-readable
  targets.
- An any-`keyAgreement`-change posture predicate: admits ordinary
  enroll/revoke, widening the license by exactly the excluded class.
  Shape 3 was checked against this rejection when it was added
  2026-09-03 and is not that predicate. Its signer conjunct means an
  ordinary client-signed enrollment or revocation mints no ladder shot,
  so the excluded class stays excluded.
- The wider form of shape 3 weighed during drafting, in which any
  version changing the enrolled-client set mints a shot for ANY standing
  ladder, rejected 2026-09-03. There the shot is minted by someone
  else's entry. The owner enrolls a phone from a remembered session, and
  a phished credential's ladder spends that version on a rotation onto
  recipients of its own, with the world-readable log showing only the
  owner's enrollment.
- Striking and reinstalling the ladder VM to mint a licensed version,
  taken as the route for enrollment approval and client disconnect run
  from a credential-only session, rejected 2026-09-03. It is
  self-lockout. The struck VM signed the generation delegation and the
  bridge the visit rides, so the visit that just lost both cannot
  publish the reinstall entry. It also costs two permanent entries per
  ceremony. The last-client transition keeps the pair, since there a
  still-standing enrolled client publishes the reinstall.
- A VM-type-driven posture set: equivalent in effect but fragile for
  the high-entropy passkey's plain `Multikey` entry.
- An ordinal-prefix numeric controller-version comparison: diverges
  from the implementation and from the profile's descendant-of hedge.
- Folding the refusal into the integrity class: callers could not
  distinguish an unlicensed append (retryable) from a corrupt log
  (not retryable).

Two widenings of clause B were proposed by the FW-356 credential-keyed
ladder VM work and rejected 2026-08-28 as do-not-reopen. The first
would have admitted a ladder-signed roster append against a version
that did not change the credential inventory, provided the signing
ladder key already stood in that version, so the last-client
transition could rotate the roster with its ladder VM already
standing. That is a loosening of the one-shot refinement rather than a
new enumerated shape, which Revisit Criteria 2 rules out in advance.
wallet-core `decisions/0008-atomic-forget-removal-entry.md` had
already turned the same widening down for a sibling ceremony, on the
ground that it admits exactly the ordinary enroll/revoke class the
license exists to exclude. The transition instead strikes and
reinstalls its own ladder VM, and the reinstall entry supplies the
inventory-changing version the clause already admits.

The second was co-signature: admit the append when a still-standing
enrolled client's key co-signs the entry, tested over `proofKeys`. It
is rejected because it would build an admission on a value
`@interop/vh-resource-log`'s port documents as host-mutable in order
and multiplicity (`controller.d.ts:86-95`). A host that strips the
client's proof from a served entry makes a legitimately written append
read as unlicensed on read-back, and the verifier then rejects the
whole roster log. That is a permanent user-key denial primitive handed
to the one party the threat model assumes hostile.

A narrowing of clause B was considered and rejected 2026-08-29
(wallet-core WC-156). It would have admitted only a version that ADDS
an inventory member, so a removal-only version would license nothing:
the last-client transition's strike entry, and a self-enrollment's
ladder-VM strike. The narrowing closes no sibling-ladder capability.
The transition's reinstall version stays licensed in the same window,
and that is the shot the transition's own rotation needs, so a thief
holding a sibling credential loses nothing while the normative
predicate moves and its downstream transcriptions are re-owed. The
strike version's shot is therefore accepted. Its exposure is bounded:
a sibling ladder stands only on an account with two or more standing
credentials, such an account has an enrolled client by construction,
so credential rotation is reachable as the remedy, the sibling's
append is attributable in the roster log, and the stolen credential's
standing wrap already opens every epoch.

One premise of that bound has since been replaced, and the bound is
re-derived without it 2026-09-03. The premise read that a sibling ladder
stands only on an account with two or more standing credentials, and
that such an account has an enrolled client by construction. A
credential-only session can now add a passphrase or a passkey to a
client-less account, so a second standing credential no longer implies
an enrolled client. What remains carries the bound. The strike version's
shot is still attributable in the roster log. The stolen credential's
standing wrap already opens every epoch, so the append buys its holder
no reading they lacked. A rotation wraps only to recipients the verified
document lists. And credential rotation is now reachable from a
credential-only session itself, the work this amendment comes from, so
the remedy no longer depends on an enrolled client.

## Consequences

- The loudness invariant becomes uniform across every axis: ladder
  authority acts only through, or carries the version of, a log
  entry. A successful offline grind of the unlock record yields a
  loud record, not standing silent authority.
- Clause A is fail-open on unaware servers; until conformance
  discovery ships, the wallet-side publish-only-on-conforming-hosts
  rule is the only guard.
- The ladder VM's bridge surface is exactly its own log resource plus
  the subtrees of Spaces declared bookkeeping at creation; requiring
  `AuxiliarySpace` means no Space can be both listed user data and
  ladder-delegable. Wallet grant code passes the subtree target
  explicitly instead of a client library's whole-Space default.
- The attacker symmetry of clause B is accepted as stated: both
  parties hold the credential, the refusal adds an equal loud step
  for both, and the loser's remedy is the recovery code, which the
  attacker does not hold.
- wallet-core's log controller seam must become posture-aware:
  evaluating S(V) and recognizing the relation asymmetry are both
  invisible through an `assertionMethod`-only accessor.
- Torn-ceremony tails still complete: a late-arriving licensed
  rotation passes because no roster entry carrying its
  posture-changing version exists yet.

## Revisit Criteria

Reopen this decision when one or more of the following holds:

1. Server-level conformance discovery ships and deployment data shows
   the fail-open window closed, allowing the wallet-side publish rule
   to relax or the clause to harden into a hard requirement.
2. A new ceremony legitimately needs a ladder-signed roster append
   outside the licensed shapes; extend the license as a new
   enumerated shape with its own controller-version rule, never by
   loosening the one-shot refinement. Considered 2026-08-28 by the
   credential-keyed ladder VM work and NOT exercised: the last-client
   transition strikes and reinstalls its own ladder VM instead, and the
   reinstall supplies an inventory-changing version the license already
   admits. Clause B is unchanged. Exercised 2026-09-03 by the browser
   wallet's credential-anchored ceremony branches. Enrollment approval
   and client disconnect run from a credential-only session, where the
   ladder signs the document entry and the roster append both, and
   neither entry changes the credential inventory. Clause B gained shape
   3, a new enumerated shape with its own controller-version rule, which
   is the route this criterion prescribes. The one-shot refinement was
   not loosened.
3. The ladder VM's authority breadth gets a principled scoping story
   inside the capability bytes themselves (caveat-level restriction),
   making the server-side inspector redundant.
4. A profile change lets an account's history log move after
   creation; the own-log derivation then needs a history-aware rule.

## Changelog

- 2026-08-28: the Context's lifecycle sentence was rewritten in place.
  It previously read that the ladder VM exists only while the account has
  no enrolled client; a VM's life is now keyed to its credential, so one
  stands for the life of every standing credential, on accounts with
  enrolled clients too. That is a reversal of the lifecycle rule, not a
  refinement -- read the prior version for the superseded wording. The
  clauses themselves are untouched: neither clause A's predicates nor
  clause B's license depends on the client census. What changes is the
  scale of what they bound, from one VM during a transitional window to
  one per standing credential for the account's life.
- 2026-08-29: two rejected widenings of clause B were added to Rejected
  Alternatives, both decided 2026-08-28 in the FW-356 design pass. No
  normative clause text changed.
- 2026-08-29: a rejected narrowing of clause B was added to Rejected
  Alternatives, decided in wallet-core WC-156. The last-client
  transition's strike version keeps its license shot. No normative
  clause text changed.
- 2026-09-01: clause A gains predicate 3, admitting a target-exact,
  single-verb (`['DELETE']` or `['GET']`) ladder-signed delegation on a
  bare Space URL over either parent kind, and the locked property is
  restated to cover it. That is a widening, not a refinement -- the
  prior property was absolute; read the prior version for it. Driven by
  the browser wallet's transient account-deletion design.
- 2026-09-01: clause A's predicate 1 gains a target bound (the
  `invocationTarget` must be the account Space's items subtree, in the
  trailing-slash form or a path under it; the bare Space URL and
  keystore targets are refused) and an action bound (`allowedAction`
  present, non-empty, and within the full closed WAS verb vocabulary).
  That is a narrowing rather than a reversal, so it amends the record
  in place: what the predicate admits now is a subset of what it
  admitted before. The locked property's restatement now says
  predicate 1 delegations stay inside that subtree, so Update Space
  Description is out of reach through every admitted shape. Driven by
  the browser wallet's FW-356 / FW-359 work and the reference server's
  roadmap; shipped in was-teaching-server 0.25.0.
- 2026-09-03: clause B gains shape 3, admitting a ladder-signed rotation
  that carries a version whose enrolled-client set changed and whose
  entry a rung of the appending ladder signed. That is a widening, a new
  admitted shape rather than a refinement; read the prior version for
  the two-shape license. The one-shot refinement and the per-entry
  single-ladder-proof rule are untouched. Rollout is verifier-first.
  Every roster-log reader ships shape 3 before any writer emits one. The
  WC-156 paragraph's bound is re-derived in place, one of its premises
  having ended. Driven by the browser wallet's design for running the
  account-management ceremonies from a credential-only session.
- 2026-09-07: clause B's scope is stated as the user key roster log, and
  a per-collection encryption descriptor log is placed outside it: a
  ladder-signed append there is admitted on `assertionMethod` membership
  alone. That is a scoping of a clause whose text always said "roster",
  not a change to any of its three shapes, but it does place a ladder
  append outside the locked property for the first time, and the
  property's restatement says so. The alternatives considered and
  rejected are recorded in the browser wallet's FW-134: an annex-VM
  signer (the transient VM stays out of `assertionMethod`, and the annex
  is mortal while descriptor logs are permanent), delegation-chained
  entry proofs (a new proof rule), a license anchored at the visit's
  annex entry (unverifiable outside the account), a user-key-derived
  assertion key (loses per-credential attribution), and refusing
  descriptor-writing ceremonies on a transient session (out by the
  full-session rule). Driven by freewallet FW-134.
- 2026-09-27: clause A is restated for the WAS v0.5 route table, where
  a Space is addressed by its trailing-slash URL and Update Space
  Metadata and Delete Space both lie inside the Space's subtree.
  Predicate 1 refuses the Space Metadata URL and every path under it.
  Predicate 3 splits by verb: its DELETE branch targets the
  trailing-slash Space URL, and its GET branch targets the Space
  Metadata URL under a parent on that URL or the Space URL. Two
  invocation-time bounds are added: a chain carrying a ladder-signed
  link may not PUT a Space Metadata URL, and may DELETE a Space only
  when every ladder-signed link in it is the target-exact DELETE
  shape; a chain carrying a transient-companion-signed link may not
  DELETE a Space at all. These are narrowings and a re-targeting, so
  they amend the record in place. Transcribed from the reference
  server, which shipped them in was-teaching-server 0.32.0 and 0.35.0.
- 2026-09-27: predicate 2's bookkeeping branch admits `allowedAction`
  within {GET, PUT, POST}, widened from {GET, PUT}. That is a widening;
  read the prior version for the two-verb set. The locked property is
  restated to say the branch reaches the bookkeeping Space rather than
  a log alone, and the reach of a POST grant is listed, its revocation
  submission left open. Driven by the browser wallet's backup export;
  shipped in was-teaching-server 0.38.0.
- 2026-09-27: clause A gains predicate 4, admitting a target-exact
  `['GET']` delegation of one Resource under a parent on that Resource
  URL or its Space's URL. That is a widening, a new admitted shape;
  read the prior version for the three-predicate clause. The locked
  property is restated to cover it. Driven by WAS-109 and freewallet
  FW-525, the transient session's read of a sibling unlock Space's
  keyring record; shipped in was-teaching-server 0.33.0. WAS-109's
  close left this amendment to the record's maintainer.
- 2026-09-27: clause A gains predicate 5, admitting a target-exact
  `['POST']` delegation over a delegated management capability the
  Space's own controller granted the account. That is a widening, a
  new admitted shape; read the prior version for the four-predicate
  clause. The locked property is restated to cover it, and the
  withdrawn PUT branch is added to Rejected Alternatives. Driven by
  WAS-151, the transient session's Export Space on a sibling unlock
  Space; shipped in was-teaching-server 0.38.0.
