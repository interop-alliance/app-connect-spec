# 0002: The generation delegation

- Status: accepted
- Date: 2026-08-19
- Driving work: the public-computer posture redesign for the browser
  wallet -- transient visit clients, enrolled per visit into a
  disposable companion did:webvh, need WAS write authority on the
  account Space with nobody left to act when the visit ends.
- Affects: this spec's companion profile (the delegation's normative
  shape) and the App Connect grant-chain section (depth-3 chains,
  visit-scoped TTLs); wallet-core (the mint, renewal, and GC
  ceremonies); was-teaching-server (chain verification);
  was-react (chain fixtures and expiry handling); depends on
  `@interop/zcap` >= 11.1.0 (the trailing-slash-base `isValidTarget`
  fix).

## Context

A transient client holds an in-memory key published only in the
companion document, not in the account document, so the server's
current-key-set rule gives it no authority of its own. It needs the
account Space's ordinary read/write surface (roster reads, credential
and activity writes, collection provisioning, App Connect grant
minting) for the duration of a visit, bounded in time, without
per-visit server-side state. The mechanism is one standing Space-scoped
zcap per companion generation, delegated to the companion DID;
transient keys invoke under it as `<companionDid>#<vm>`, and App
Connect grants chain through it. Its scope, chain shape, lifetime, and
renewal policy are permanent wire artifacts.

## Decision

- `invocationTarget` is the account Space items subtree: the Space URL
  with a trailing slash (`https://<host>/space/<spaceId>/`).
  `allowedAction` is the full closed vocabulary
  `['GET', 'HEAD', 'POST', 'PUT', 'DELETE']`.
- Attenuation is structural, not enumerated. Downward, App-Connect-time
  sub-delegations keep the per-target-class action caps unchanged
  (child-within-parent is enforced on both actions and targets, so any
  verb missing here would cap every transient-visit app grant below
  its durable-client shape). The zcap revocation endpoint stays out by
  construction: its chain must root exactly in the Space and is
  invoked under the root capability, so no verb set here reaches it.
  The subtree target excludes the bare Space URL itself, keeping the
  Space Description PUT (a controller rewrite -- the permanent-takeover
  escalation a window-bounded key must not hold) and the Space DELETE
  outside the delegation in the capability bytes.
- Chain shape is depth 3: the account Space's root zcap, the
  generation delegation (`controller` = the bare companion DID string;
  `proof.verificationMethod` = the account document's ladder VM or a
  durable client's VM), then the App Connect grant
  (`capabilityChain` = `[root id string, the full embedded
  generation-delegation object]`, per zcap's
  all-strings-except-last-embeds rule). The library's per-hop
  `expires` monotonicity IS the TTL clamp: an app grant can never
  outlive the generation delegation.
- `expires` is 365 days. GC's explicit
  revoke is the intended end-of-life; expiry is the backstop.
- GC cadence is fixed quarterly (90 days), run at the first durable
  login after the period elapses. A fixed period keeps the permanent
  account-log pointer-entry rhythm coarse: at most about four entries
  per year, revealing little about transient-use frequency.
- Renew precedes mint. Inside the 30-day renewal window, a transient
  App Connect approval runs the licensed generation-delegation renewal
  as a blocking pre-mint stage (ladder-signed, published over the
  two-branch bridge, so even a hard-expired delegation is
  recoverable). A renewal failure fails the approval with the standard
  retryable-ceremony posture. By construction the bounded grants (30d
  read, 7d write) always receive their full TTL; only 365-day-class
  grants ever meet the monotonicity clamp, at 30 or more days
  remaining.
- Grants approved in a transient session are visit-scoped: TTL clamped
  to the signing authority's lifetime, with consent copy stating the
  grant lasts until it expires (logout as the early end).

## Rejected Alternatives

- A narrowed verb/target enumeration: fails the provisioning floor
  structurally -- a Collection Description PUT targets a URL that does
  not exist at mint time, so only a subtree prefix can cover it -- and
  would track the wallet's write set forever on a standing embedded
  artifact, with every miss failing after consent.
- An unexcluded Space-wide delegation: converts a window-bounded key
  capture into permanent, log-free takeover via the Space Description
  controller rewrite.
- The server-side companion-chain guard clause as the exclusion
  mechanism: a second normative fail-open clause where the capability
  bytes can carry a fail-closed one. It stays the recorded fallback
  had the zcap `isValidTarget` fix been declined; an unfixed verifier
  refuses the subtree's descendants (fails closed), the right failure
  direction.
- 120-day or 180-day TTLs: novel constants, with renewal churn
  beginning exactly when GC is merely due.
- A monthly GC cadence: a fine-grained public usage signal in the
  account log.
- Usage-driven or size-triggered GC: the interval between permanent
  pointer entries becomes a transient-use-frequency histogram, and a
  size-based generation has unbounded duration, unmatchable to any
  `expires`.
- A numeric near-expiry mint floor (refuse approvals close to expiry):
  a come-back-from-a-trusted-browser state contradicts the posture's
  premise.
- Clamp-on-renewal-failure fallback: delivers exactly the silently
  short grant the renew-precedes-mint rule exists to prevent.

## Consequences

- Transient-visit app grants have the same action shape as
  durable-client grants; nothing downstream needs a posture branch.
- The consequence residues are bounded by the transient VM's window
  (amended 2026-08-19, the public-computer posture design's delta
  review: no per-VM expiry exists -- the sidecar carries no
  `expires`, per the VM-expiry closure -- so the bound is the
  generation delegation's revoke or its 365-day `expires`, together
  with GC replacement; what defines a "live"
  visit for the GC guard is the 24-hour max-visit quiet bound,
  resolved 2026-08-19 and recorded in wallet-core's GC-observables
  decision) and no
  stronger than the read/write authority the visit necessarily holds:
  export/import POSTs (equivalent to the GET and PUT authority already
  granted), collection policy flips (expose ciphertext only),
  overwrite/delete breadth (unconditional PUT already destroys by
  overwrite), and a roster-log clobber (DoS-grade -- forged entries
  cannot verify and continuity pins surface the refusal).
- Visit-scoped grants die with the visit; the app reconnects through
  the ordinary flow with fresh consent, same identity and recipient
  key.
- Ordinary revocation of the durable client that minted the current
  generation kills the delegation mid-generation; the revocation
  cascade gains a re-mint stage, and mid-generation grant death is a
  stated consequence of ordinary disconnects.
- Verifiers older than `@interop/zcap` 11.1.0 refuse the subtree's
  descendants outright.

## Revisit Criteria

Reopen this decision when one or more of the following holds:

1. The WAS action vocabulary grows or a new server route class appears
   inside the Space subtree whose reachability under this delegation
   was not weighed (re-run the write-set inventory before widening or
   narrowing).
2. Measured GC behavior shows quarterly cadence plus the 365-day TTL
   producing renewal churn or delegation lapses on real accounts.
3. A deployment requires per-visit server-side revocability finer than
   the generation, which the standing-delegation shape cannot express.

If revisited, change the scope as a new delegation profile version;
never reinterpret the shipped subtree target in place.
