# 0004: Transient sessions sign DIDAuth as bare did:key

- Status: accepted
- Date: 2026-08-19
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
