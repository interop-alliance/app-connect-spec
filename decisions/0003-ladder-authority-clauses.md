# 0003: The ladder VM's authority clauses

- Status: accepted
- Date: 2026-08-19
- Driving work: the public-computer posture redesign for the browser
  wallet -- an account with zero enrolled durable clients is anchored
  by a ladder-derived verification method in its document, and that
  VM's authority had to be bounded so a phished unlock credential (or
  a storage host grinding the unlock record offline) cannot exercise
  silent authority during the client-less window.
- Affects: this spec's companion profile (clause A's normative text
  and its fail-open note) and Resource Log Profile (clause B, a new
  subsection beside "The sealing append"); was-teaching-server (the
  second `inspectCapabilityChain` inspector); wallet-core (the
  resource-log verification and the `ResourceLogLicenseError` class);
  both wallets.

## Context

The ladder VM is the stable document-visible verification method
derived from a standing unlock credential's random ladder seed. It
exists only while the account has no enrolled durable client, and it
is recognized by relation asymmetry: a `capabilityDelegation` member
absent from `capabilityInvocation`. It must be able to sign the
generation delegation (under `capabilityDelegation`) and anchor roster
appends (under `assertionMethod`), or a client-less account is
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
resolves to the ladder VM is admitted iff one of two predicates
holds:

1. Companion-DID controller, by pointer equality: the delegation's
   `controller` string equals the companion DID named by the
   `https://w3id.org/byoe#DelegatedClients` service entry of the
   account document the chain already resolved as delegator (zero
   extra I/O), behind the syntactic gate that the string parses as a
   self-hosted did:webvh. A GC pointer swap thereby instantly kills
   the prior generation's delegations.
2. Bridge-shaped target, two-branch, capability-side: the
   delegation's `invocationTarget` equals the account log resource
   URL (`<base>/space/<S>/id/did.jsonl` for some Space S) with
   `allowedAction` a subset of {PUT}; or equals the trailing-slash
   Space URL of a Space whose Description `type` includes
   `DelegatedClientsSpace`, with `allowedAction` a subset of
   {GET, PUT}. The second branch costs one memoized Space Description
   read.

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
2. A rotation anchored at a posture-changing document version, and
   one-shot: refused when the verified roster head already contains
   an entry anchored at V or later. Comparison is by position in the
   controller's verified version history
   (`headAnchorIndex >= indexOf(V)`, the structural twin of the
   shipped sealing check).

Everything else -- above all a rotation against an unchanged
document, the silent-rekey shape -- is refused by every verifier.
The refusal is a write-time admission error, a new named class
`ResourceLogLicenseError` beside `ResourceLogIntegrityError` and
`ResourceLogContinuityError`: retryable after a posture-changing
entry, not log corruption, and the profile's reject-the-whole-log
severity does not apply.

The locked property across both clauses: no ladder authority whose
exercise leaves no record. Every ladder delegation either needs a
loud companion entry to resolve or can only write a log; every
ladder roster append is anchored at a loud document event.

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
- An ordinal-prefix numeric anchor comparison: diverges from the
  implementation and from the profile's descendant-of hedge.
- Folding the refusal into the integrity class: callers could not
  distinguish an unlicensed append (retryable) from a corrupt log
  (not retryable).

## Consequences

- The loudness invariant becomes uniform across every axis: ladder
  authority acts only through, or anchored at, a log entry. A
  successful offline grind of the unlock record yields a loud record,
  not standing silent authority.
- Clause A is fail-open on unaware servers; until conformance
  discovery ships, the wallet-side publish-only-on-conforming-hosts
  rule is the only guard.
- The attacker symmetry of clause B is accepted as stated: both
  parties hold the credential, the refusal adds an equal loud step
  for both, and the loser's remedy is the recovery code, which the
  attacker does not hold.
- wallet-core's log controller seam must become posture-aware:
  evaluating S(V) and recognizing the relation asymmetry are both
  invisible through an `assertionMethod`-only accessor.
- Torn-ceremony tails still complete: a late-arriving licensed
  rotation passes because no roster entry anchored at its
  posture-changing version exists yet.

## Revisit Criteria

Reopen this decision when one or more of the following holds:

1. Server-level conformance discovery ships and deployment data shows
   the fail-open window closed, allowing the wallet-side publish rule
   to relax or the clause to harden into a hard requirement.
2. A new ceremony legitimately needs a ladder-signed roster append
   outside the two licensed shapes; extend the license as a new
   enumerated shape with its own anchor rule, never by loosening the
   one-shot refinement.
3. The ladder VM's authority breadth gets a principled scoping story
   inside the capability bytes themselves (caveat-level restriction),
   making the server-side inspector redundant.
