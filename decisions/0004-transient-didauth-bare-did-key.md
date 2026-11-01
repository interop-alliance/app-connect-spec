# 0004: Transient sessions sign DIDAuth as bare did:key

- Status: accepted
- Date: 2026-08-19
- Amended: 2026-08-28 -- the `<companionDid>#<vm>` form also signs the
  grants a transient visit delegates, so the "only WAS invocations"
  clause below no longer holds. The decision itself is unchanged. See
  wallet-core `decisions/0013-transient-vm-capability-delegation.md`.
- Amended: 2026-09-03 -- the browser wallet now pins the App Connect
  holder to the client did:key by an explicit override at composition
  time, so this record's consequence holds by construction rather than
  by which signer happened to be provisioned. Its generic DIDAuth
  holder, outside App Connect, may be the account did:webvh or its
  did:web projection when the request's `acceptedMethods` asks for one.
  The decision itself is unchanged. See the browser wallet's
  `decisions/0016-didauth-holder-dispatches-on-accepted-methods.md`.
- Amended: 2026-10-31 -- the holder of an App Connect response is now the
  wallet's pairwise did:key toward the app, derived from a per-connection
  seed on the wallet's connection entry, on every session kind. It is still
  a bare did:key, so the resolvability rule below holds. The consequence
  "a fresh, unlinkable holder per visit" no longer holds: the holder is
  stable per connection and changes only when the wallet rotates the seed,
  which it does in the ceremonies that withdraw a client's or a
  credential's authority. The response presentation's holder text is
  rewritten to match. See the browser wallet's
  `decisions/0052-the-app-connect-holder-is-the-pairwise-connection-did-key.md`
  and wallet-core's `decisions/0032-pairwise-seed-rotation-and-the-rotate-message.md`.
- Driving work: the public-computer posture redesign for the browser
  wallet -- a transient visit client, enrolled in the capability-gated
  companion did:webvh, still has to answer an App Connect request with
  a DIDAuth presentation the app can verify.
- Affects: this spec's response-presentation section (holder and proof
  spellings for transient sessions), was-react (the app-side loader
  and verification), wallet-core (VP composition), both wallets.

## Context

A transient session's key lives in two spellings: the companion form
`<companionDid>#<vm>` used for WAS invocations, and the key's own bare
did:key form. The companion log is capability-gated (private), so any
companion-spelled proof key is unresolvable by the app-side document
loader -- a 401 in the exact popup flow the posture exists for. The
App Connect response VP needs a holder and a
`proof.verificationMethod` that every app can resolve. was-react's
verification today checks proof purpose and challenge/domain only,
and verifier-core types the holder as optional, so the holder half of
the rule is normative future-proofing rather than a currently
enforced check. Bare did:key is also the one DID method both ends
already declare (`acceptedMethods: [{ method: 'key' }]` in the
request, honored pre-consent), and it is byte-identical to the
browser wallet's existing non-KMS signer branch.

## Decision

In a transient session, the App Connect response VP's holder and its
DIDAuth `proof.verificationMethod` both use the transient key's bare
did:key form (`did:key:z6Mk...` and `did:key:z6Mk...#z6Mk...`). Only
WAS invocations use the `<companionDid>#<vm>` spelling.

## Rejected Alternatives

- A companion-DID holder: publishes the private companion DID to
  every connected app, and asserts a holder no app can resolve.
- The account did:webvh as holder: claims an identity the transient
  key is not a document member of -- the mismatch a stricter future
  verifier would refuse.

## Consequences

- The proof key resolves in any app-side loader with no access to the
  companion log; the dominant public-terminal App Connect workflow
  does not depend on companion resolvability.
- Apps see a fresh, unlinkable holder per visit; nothing ties two
  visits' presentations together at the app.
- The rule mirrors the existing did:webvh/did:key split for durable
  clients (the VP holder stays the client did:key there too), so
  app-side loaders need no new case.

## Revisit Criteria

Reopen this decision when one or more of the following holds:

1. A verifier in the ecosystem begins enforcing holder semantics that
   require the holder to be a resolvable member of the granting
   account's document, breaking the bare-did:key holder.
2. The companion document gains a public-resolvability profile, which
   would remove the 401 constraint (the privacy argument against
   publishing the companion DID to apps would still need its own
   weighing).

## Changelog

- 2026-08-28: no change to the decision. The transient key's
  `<companionDid>#<vm>` form was found to sign delegated grants too, so
  the "only WAS invocations" clause in the Decision no longer holds.
- 2026-09-03: no change to the decision. Recorded that the browser
  wallet pins the App Connect holder to the client did:key by explicit
  override, closing a drift in which a remembered session against a KMS
  deployment presented a did:web holder, and that its holder outside
  App Connect dispatches on `acceptedMethods`.
- 2026-10-31: the holder moved from the transient key's bare did:key to the
  wallet's pairwise did:key toward the app. Still a bare did:key; the
  resolvability decision stands, the per-visit unlinkability consequence is
  reversed.
