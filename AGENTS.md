# App Connect for Wallet Attached Storage -- Specification

This repo is the **App Connect** specification, a W3C CCG-style document
authored in **ReSpec + Markdown**. App Connect is a CHAPI/VCALM profile by
which a web application connects to a Wallet Attached Storage (WAS)-backed
wallet in a **single exchange**, obtaining a per-user application identity
(the app-key credential) and delegated storage capabilities -- so the app can
use user-owned storage without running a backend.

This is a **companion spec layered on top of [WAS]**
(the Wallet Attached Storage spec): WAS defines the Space / Collection /
Resource storage model, the HTTP API, and the zCap authorization profile; App
Connect defines how an application *obtains* capabilities into that model. It
also references the **WAS Encrypted Collections** profile ([WAS-EC], sibling
checkout `../encrypted-collections-spec`) for envelope cryptography and
recipient-key derivation.

The profile is implemented by the **`@interop/wallet-core`** library and used
by the **DCW** and **Freewallet** wallets (see reference checkouts below).

This is a **spec repo, not a code repo**. The deliverable is the rendered HTML
document. There is no build/test/lint pipeline -- "correct" means the prose is
accurate, internally consistent, and renders cleanly in ReSpec.

## Files

- `spec.md` -- the entire normative specification (single file, ~2100 lines).
  This is what you edit 99% of the time.
- `index.html` -- ReSpec shell. Sets config (`specStatus: "unofficial"`, GitHub
  repo, xref to `did-core`, and a `localBiblio` for `WAS`, `WAS-EC`, `VCALM`,
  `CHAPI`, `ZCAP`, `DID-KEY`, `DID-WEBVH`) and includes `spec.md` via
  `<div data-include="spec.md" data-include-format="markdown">`. The `<h1>`,
  abstract, SotD, and conformance sections live here, not in `spec.md`.
- `README.md` -- short; overview and local preview instructions.

## Preview locally

```
npx serve .
```
Then open the served `index.html`; ReSpec renders client-side. There is no
static build step -- the published site (GitHub Pages at
`interop-alliance.github.io/app-connect-spec/`) renders ReSpec the same way.

## ReSpec / Markdown authoring conventions

`spec.md` mixes Markdown with ReSpec's inline HTML markup. Match the existing
style:

- **Cross-references to sections:** `[[[#section-id]]]` renders as a live link
  with the section's title. Headings carry explicit ids
  (`### Matching {#app-key-matching}`) -- use those.
- **Term references:** `[=term=]` links to a `<dfn>` defined in the Terminology
  list (e.g. `[=app-key credential=]`, `[=AppConnectQuery=]`, `[=grant=]`,
  `[=seed=]`, `[=resource log=]`, `[=Space=]`). Define new terms in the
  `## Terminology` `<dl>` block with `<dfn data-lt="alias|aliases">`.
- **Bibliography refs:** `[[WAS]]`, `[[WAS-EC]]`, `[[VCALM]]`, `[[CHAPI]]`,
  `[[ZCAP]]`, `[[RFC...]]` -- ReSpec auto-resolves these (custom ones via the
  `localBiblio` in `index.html`). Uppercase keys.
- **Headings map to numbered sections.** `##` = top-level section, `###`/`####`
  nest.
- Honor the global rules: use `--` not an em dash, and `to` not `→`.

## The App Connect model (so edits stay consistent)

The whole negotiation is **one exchange**: one CHAPI `get` carrying a VCALM
presentation request with one `AppConnectQuery`, answered by one signed
Verifiable Presentation. No second popup, no store step, no prior
registration.

The spec's main moving parts, in document order:

- **The AppConnectQuery request** -- transport over CHAPI/VCALM, exclusivity
  rules, capability requests, challenge/domain.
- **Invocation target descriptors** -- the typed vocabulary describing what
  storage a grant targets, with a descriptor type registry and **per-class
  allowed actions** bounding what a delegation may carry.
- **The app-key credential** -- a self-issued VC carrying a 32-byte seed from
  which the application derives its key material. Key invariants: the
  **seed-binds-subject rule** (subject DID is derived from the seed) and
  **origin binding** (the wallet scopes the credential to the browser-attested
  requesting origin, minting on first run and matching on return visits).
- **The response presentation** -- the `zcap` and `appConnect` members, the
  `firstRun` member, and the application-side verification order.
- **Grant processing** -- resolution, capping, recording, provisioning,
  encrypted-by-default collections, lifetimes, consent.
- **The resource log** -- the hash-linked log format for key resources
  co-managed between a wallet's clients and the storage server -- is defined
  in [WAS-EC], not here; this spec only cites its external-authorization
  rule for enrolled clients. Do not re-introduce the profile text.
- **Fail-closed processing** is the extensibility rule throughout: anything
  unrecognized is rejected, not ignored.

Identity-model agnosticism: nothing in App Connect inspects the wallet
account's own DID method; delegation targets the app-key credential's subject
DID, and grant validity is the storage server's decision per WAS.

Wallet terminology: follow the global `clientId` / `writerId` rules -- never
"device" for either concept.

## Parties to this contract

Every repo that implements or consumes this profile, with the specific
modules that speak it. **The maintenance rule: a normative change's checklist
is a walk of this table** -- for each row, resolve the impact as shipped
(naming what landed, including the row's ARCHITECTURE/AGENTS docs) or
explicitly waived (`unaffected: <repo> (<why>)`). When the profile version
changes (see the hosted context URL note in `spec.md`), each implementing
package's CHANGELOG names the profile version it now speaks.

| Repo | Modules speaking the contract |
| --- | --- |
| wallet-core | The wallet-side implementation both wallets consume: `src/request/classify.ts` (`appConnectRequestOf`, the `AppConnectQuery` validation), `src/request/appKey.ts` (the app-key credential: mint / match / legacy re-issue / store-time refusal, the marker type and inline context), `src/request/composeVp.ts` (the response VP with the `zcap` array and `appConnect` marker). |
| freewallet | Consent UI, credential storage, and the delegation machinery over wallet-core: `src/lib/walletRequest/` (esp. `appConnect.ts`), `WalletGetPage`. |
| dcw (private) | The mobile wallet's App Connect flow, over the same wallet-core modules. |
| was-react | The app side: `src/auth/loginRequest.ts` (`buildAppConnectVpr`), `src/identity/seedCredential.ts` (`findSeedCredential` / `parseSeedCredential`), `src/auth/verifyResponse.ts`. Counterpart-tested against wallet-core's real implementation in `src/auth/walletCoreCounterpart.test.ts`, which runs in its ordinary CI. |
| byoe-react-examples | Example apps consuming the profile through was-react; the wallet-tier e2e drives a real freewallet popup. |
| life-advisor (private) | A consuming app, through was-react. |
| was-conformance-suite | No App Connect-specific suite; the delegated grants the profile issues are exercised generically against servers (`client-delegation`). Re-check this row when the profile grows a server-visible surface. |

## Ecosystem conventions

- Cross-repo lessons (invariants, gotchas, and process recipes that span
  repos) live in the ecosystem learnings file,
  [byoe-ecosystem/LEARNINGS.md](https://github.com/interop-alliance/byoe-ecosystem/blob/main/LEARNINGS.md)
  (usually checked out beside this repo as `../byoe-ecosystem`); read it at
  the start of any cross-repo task.
- Decisions about the contract this spec owns (profile and wire-contract
  decisions) are recorded in this repo's [decisions/](decisions/) directory,
  one `NNNN-slug.md` file per decision; the convention and template are
  canonical in
  [isomorphic-lib-template's `decisions/`](https://github.com/interop-alliance/isomorphic-lib-template/tree/main/decisions).

## Reference material (read-only, outside this repo)

These are separate repositories. Use them to ground spec prose against real
behavior -- check with the user before editing anything in them.

- [wallet-attached-storage-spec](https://github.com/w3c-ccg/wallet-attached-storage-spec)
  -- the WAS spec this profile layers on; its `spec.md` is the source of
  truth.
- [wallet-core](https://github.com/interop-alliance/wallet-core) --
  `@interop/wallet-core`, the library that implements the wallet side of this
  profile; shared by both wallets below.
- dcw (private repo) and
  [freewallet](https://github.com/interop-alliance/freewallet) -- the two
  wallets that speak App Connect. The canonical "does the spec match an
  implementation?" checks.
- [was-client](https://github.com/interop-alliance/was-client) /
  [was-teaching-server](https://github.com/interop-alliance/was-teaching-server)
  -- reference WAS client and server, for confirming the storage API the
  granted capabilities are invoked against.
- [zcap-developer-guide](https://github.com/interop-alliance/zcap-developer-guide)
  -- how zCaps (delegation, invocation, verification, root-of-trust) actually
  work. Consult before writing anything about authorization.
