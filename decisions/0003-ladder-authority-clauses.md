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
  property is restated to cover it.
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
resolves to the ladder VM is admitted iff one of three predicates
holds:

1. Companion-DID controller, by pointer equality: the delegation's
   sole `controller` equals the companion DID named by the
   `https://w3id.org/byoe#DelegatedClients` service entry of the
   account document the chain already resolved as delegator (zero
   extra I/O), behind the syntactic gate that the string parses as a
   self-hosted did:webvh. `controller` is read after normalizing the
   zcap array form; two or more entries are refused, since a second
   controller could invoke past the pointer. A GC pointer swap
   thereby instantly kills the prior generation's delegations.
2. Bridge-shaped target, two-branch, capability-side: the
   delegation's `invocationTarget` equals the delegator account's own
   history log resource URL -- derived from the account DID, which
   carries its log's Space and Collection; no other log-shaped target
   qualifies -- with `allowedAction` a subset of {PUT}; or equals the
   trailing-slash (subtree) URL of a Space whose Description declares
   it delegated-clients bookkeeping (`type` names both
   `AuxiliarySpace` and `DelegatedClientsSpace`; Create Space refuses
   the latter without the former, so a listed data Space cannot carry
   the subtype), with `allowedAction` a subset of {GET, PUT}. Only
   the subtree form is admitted: a no-slash target would also cover
   Update Space Description under target attenuation, so wallets pass
   the subtree target explicitly when granting. The second branch
   costs one memoized Space Description read.

3. Target-exact single-verb, both chain lengths (added 2026-09-01):
   the delegation's `invocationTarget` is a bare Space URL and its
   `allowedAction` is exactly `['DELETE']` or exactly `['GET']`. When
   the parent is a delegated capability, the target must equal the
   parent's unchanged; when the parent is the synthesized Space root,
   the target must equal that root's own Space URL. This is the
   account-deletion ceremony's admission path; its delegatee and
   invoker is the ladder VM's own bare did:key, which predicate 1
   never admitted. Two bounds keep it narrow: on the
   management-capability arm the parent already carries DELETE on
   exactly that Space URL, so the predicate widens who signs the last
   link rather than what the account may do; and the target-equality
   rule means the ladder VM cannot aim the child anywhere new.

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
ladder-signed roster append is accepted in exactly two shapes:

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

Everything else -- above all a rotation against an unchanged
document, the silent-rekey shape -- is refused by every verifier.
The refusal is a write-time admission error, a new named class
`ResourceLogLicenseError` beside `ResourceLogIntegrityError` and
`ResourceLogContinuityError`: retryable after a posture-changing
entry, not log corruption, and the profile's reject-the-whole-log
severity does not apply.

The locked property across both clauses, restated 2026-09-01: every
ladder delegation either needs a loud companion entry to resolve, can
only write a log, or is a target-exact single-verb GET or DELETE on
one Space of the delegator's own account -- a read, or a destruction
whose account-Space case removes the log any record would live in and
leaves no reader to remediate; every ladder roster append carries a
loud document event's version. The prior absolute form ("no ladder
authority whose exercise leaves no record") is superseded: a DELETE
admitted under predicate 3 leaves no record, and that carve-out is the
account-deletion design's stated trade.

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
- Hard invocation rejection on clause failure: a behavioral change to
  the policy fallback with no security gain on world-readable
  targets.
- An any-`keyAgreement`-change posture predicate: admits ordinary
  enroll/revoke, widening the license by exactly the excluded class.
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
   outside the two licensed shapes; extend the license as a new
   enumerated shape with its own controller-version rule, never by
   loosening the one-shot refinement. Considered 2026-08-28 by the
   credential-keyed ladder VM work and NOT exercised: the last-client
   transition strikes and reinstalls its own ladder VM instead, and the
   reinstall supplies an inventory-changing version the license already
   admits. Clause B is unchanged.
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
