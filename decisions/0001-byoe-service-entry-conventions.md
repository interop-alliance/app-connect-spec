# 0001: byoe service entries dispatch on type, not fragment

- Status: accepted
- Date: 2026-08-19
- Driving work: the public-computer posture redesign for the browser
  wallet -- per-visit transient clients recorded in a disposable
  companion did:webvh, referenced from the account document, with the
  generation's standing delegation carried inside the companion
  document itself.
- Affects: this spec's companion profile (the service-entry
  vocabulary), the byoe-context term definitions
  (`https://w3id.org/byoe#DelegatedClients`,
  `https://w3id.org/byoe#GenerationDelegation`), wallet-core (the
  entry builders and readers), was-teaching-server (the
  authorization-side pointer read), and both wallets.

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

## Decision

byoe service entries are identified by their `type` IRI, never by
their fragment id. Fragment ids are non-semantic: any stable
identifier (random or content-derived) is valid, readers MUST dispatch
on the `type` IRI, and the profile states this rule. The principle is
uniform across all byoe service entries, present and future.

Two entries are defined under it:

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

## Consequences

- Readers hold no fragment expectations; adding a byoe service entry
  never risks a fragment collision or a reader keying on a spelling.
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

## Revisit Criteria

Reopen this decision when one or more of the following holds:

1. A DID-spec revision or a consuming resolver rejects a map-valued
   `serviceEndpoint`, forcing a different carrier for the embedded
   delegation.
2. A byoe service entry acquires a reader that structurally cannot
   dispatch on `type` (for example, a registry keyed by fragment),
   which would force fragment semantics back in as a parallel
   convention, not a retrofit of existing entries.
