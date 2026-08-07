<div class="remove">

# App Connect for Wallet Attached Storage v0.1

**Abstract:** App Connect defines a CHAPI/VPR profile by which a web
application connects to a Wallet Attached Storage (WAS) backed wallet in a
single exchange, receiving a per-user application identity and a set of
delegated storage capabilities in one signed response.

**Status:** Experimental W3C CCG draft, undergoing regular revisions.
Companion document to the Wallet Attached Storage specification.

</div>

## Introduction {#introduction}

<div class="note">
This section is non-normative.
</div>

An application that wants a place to keep a user's data has traditionally had
to run a backend to keep it in. App Connect exists so that it does not have
to: the user already has storage (a WAS [=Space=]) and already has a wallet
that controls it, so the application asks the wallet for the two things it
needs -- an identity of its own, and permission to write into a corner of the
user's storage -- and then talks to the storage server directly.

The whole negotiation is **one exchange**. The application opens a single
[=CHAPI=] `get` carrying a verifiable presentation request [[VCALM]] with one
[=AppConnectQuery=] in it. The wallet answers, in the same round, with one
signed Verifiable Presentation carrying:

* the [=app-key credential=] -- a self-issued Verifiable Credential holding a
  32-byte [=seed=] from which the application derives its own key material,
  matched from wallet storage for a returning user or minted on the spot for a
  new one; and
* the [=grants=] -- capabilities [[ZCAP]] delegated to the subject DID of that
  very credential, targeting collections in the user's Space.

There is no second popup, no store step, and no prior registration. The
application never names a controller DID in its request, because on first run
it does not have one yet: the wallet mints the identity and delegates to it in
the same breath. That is what collapses the flow to a single round.

### Scope {#scope}

This document defines:

* the [=AppConnectQuery=] request query type and its processing rules
  ([[[#request]]]);
* the [=invocation target descriptor=] vocabulary and the per-class action
  ceilings that bound what a delegation may carry ([[[#descriptors]]]);
* the [=app-key credential=]: its marker type, claim vocabulary, seed
  encoding, and the seed-to-subject binding rule ([[[#app-key-credential]]]);
* the response presentation's `zcap` and `appConnect` members and the
  application-side verification order ([[[#response]]]);
* the grant resolution, capping, and provisioning rules
  ([[[#grants]]]).

This document does **not** define:

* **the storage protocol.** Spaces, Collections, Resources, the HTTP API, and
  the capability authorization profile are defined in [[WAS]]. App Connect
  produces capabilities; invoking them is [[WAS]]'s business.
* **envelope cryptography.** How a collection's contents are encrypted, how a
  key epoch is minted, and how a recipient key is derived byte-for-byte from a
  controller DID are defined in the forthcoming WAS Encrypted Collections
  profile [[WAS-EC]]. This document states the *invariants* that the App
  Connect exchange depends on and references [[WAS-EC]] for the constructions.
* **the wallet account's own identity ceremonies.** How a wallet enrolls a new
  client of the same account, revokes one, or recovers from a lost secret is
  wallet-internal and outside this profile.

### Identity-model agnosticism {#identity-model-agnosticism}

This profile is deliberately agnostic about how the wallet account itself is
identified. A [=Space=]'s controller may be a `did:key`, a `did:webvh`, or any
other DID the storage server is willing to resolve. Nothing in App Connect
inspects it.

The profile rests on two invariants instead:

1. **Delegation targets the connecting credential's subject DID.** The wallet
   delegates to the subject of the [=app-key credential=] it matched or
   minted, and to nothing else.
2. **Grants verify under the server's controller resolution.** Whether a
   delegated capability is valid at invocation time is decided by the storage
   server resolving the Space controller, per [[WAS]]. App Connect neither
   duplicates nor constrains that decision.

A wallet may therefore change its account identity model without changing
anything an application sees.

### Relationship to other documents {#relationship}

| Document | Relationship |
|----------|--------------|
| [[WAS]] | Defines the Space / Collection / Resource model, the HTTP API, and the capability authorization profile whose delegations this profile produces. |
| [[WAS-EC]] | Forthcoming companion profile. Defines the encrypted-collection construction: key epochs, roster recipients, envelope format, and the derivation of a recipient key from a controller DID. This profile defers to it for the recipient-key derivation ([[[#recipient-derivation]]]), the key-agreement half of the application's key material ([[[#key-derivation]]]), the epoch rotation behind forward-only re-grants ([[[#descriptor-shared-collection]]]), and the definition of [=epoch-roster recipient=] itself. |
| [[VCALM]] | Defines verifiable presentation requests and their query types: the request body that carries an [=AppConnectQuery=], the `DIDAuthentication` and `QueryByExample` query types this profile interacts with, and `AuthorizationCapabilityQuery`, the standalone capability-request query type whose entry shape [=AppConnectQuery=] reuses. |
| [[CHAPI]] | The transport this profile is defined over: it supplies the browser-attested requesting origin the [=app-key credential=] is bound to. |

### Conformance Classes {#conformance-classes}

This specification defines two conformance classes.

Both are defined over one [=App Connect exchange=].

A conformant **[=wallet=]** MUST:

* recognize the [=AppConnectQuery=] type ([[[#appconnectquery]]]) and process it
  per [[[#request]]];
* match or mint the [=app-key credential=] per
  [[[#app-key-credential]]];
* refuse to store a marker-carrying credential arriving from outside an
  exchange it performed ([[[#store-time-refusal]]]);
* resolve, cap, and delegate the requested capabilities per [[[#grants]]],
  recording each grant it delegates so that it can later be revoked
  ([[[#grant-recording]]]);
* compose the response presentation per [[[#response]]], signing it when DID
  Authentication was requested;
* leave any partial failure repairable by re-running the exchange
  ([[[#resumability]]]);
* apply the fail-closed processing rules throughout: an unrecognized
  [=invocation target descriptor=] type, or a request entry it cannot satisfy,
  MUST surface as a visible refusal rather than as a quietly narrowed grant.

A conformant **[=application=]** MUST:

* build the request per [[[#request]]], with a fresh challenge, its own live
  browser origin as `domain`, and a `DIDAuthentication` query alongside the
  [=AppConnectQuery=] ([[[#request-exclusivity]]]);
* verify the response presentation in the order given in
  [[[#response-verification]]];
* enforce the parse-time checks on the [=app-key credential=] in
  [[[#app-key-parsing]]], independently of the wallet;
* treat a verifying presentation that carries no app-key credential as a
  distinct fail-closed outcome, not as a first run.

<div class="note">
**The storage server is deliberately not a party to App Connect.** It does not
see the [=CHAPI=] exchange, the request, the response presentation, the
[=seed=], or the consent. It only ever sees the resulting capability
invocations arriving on its HTTP API, which it authorizes exactly as [[WAS]]
already specifies. No server-side support is required to deploy App Connect,
and a server cannot tell an App Connect grant from any other delegated
capability. This is a design goal, not an accident: it keeps the exchange a
matter between the user's wallet and the application, and it keeps the storage
provider substitutable.
</div>

<div class="note">
**Implementation status.** Two independent implementations exist and are the
source of every normative statement below: Freewallet (wallet-side) and the
`was-react` library (application-side). Values marked informative in this
document (notably the capability lifetimes in [[[#ttl]]]) are the values those
implementations ship; the rules around them are normative.
</div>

### Reading This Document {#reading-this-document}

<div class="note">
This subsection is non-normative. It collects conventions the rest of the
document relies on.

**All examples share one setting.** The application is "Example Notes", served
from the origin `https://app.example`, and its [=app-key credential=] type is
`ExampleNotesUser` under the vocabulary base `https://app.example/vocab#`. The
user's wallet Space is `81246131-69a4-45ab-9bff-9c946b59cf2e` on the host
`wallet-storage.example`, so its Space URL is
`https://wallet-storage.example/space/81246131-69a4-45ab-9bff-9c946b59cf2e`.
DID and key values in examples are illustrative and are not real derivations.

**Proofs are elided.** Example credentials and presentations show
`"proof": { ... }` rather than a real signature, except where the proof's
members are themselves under discussion.

**"Byte-significant" means what it says.** Several strings in this document --
type names, term IRIs, the key-derivation label, the collection-name grammar
-- are inputs to a derivation or to a canonicalized signature. Changing one
does not produce a variant profile; it produces an incompatible one, and for
the derivation inputs it orphans every identity already derived under the old
value.
</div>

## Terminology {#terminology}

<dl class="termlist definitions" data-sort="ascending">
  <dt><dfn data-lt="action|actions|allowedAction">action (allowedAction)</dfn></dt>
  <dd>See [[WAS]]. The kind of operation a request performs on a [=target=],
    named by a [=capability=] so it can be authorized. This profile bounds the
    actions a [=grant=] may carry to a closed vocabulary of uppercase HTTP
    method names; see [[[#action-vocabulary]]].</dd>

  <dt><dfn data-lt="applications|requesting party">application</dfn></dt>
  <dd>A web application that connects to a user's [=wallet=] through an App
    Connect exchange in order to obtain its own identity and delegated access
    to the user's storage. One of the two conformance classes; see
    [[[#conformance-classes]]]. An application is identified to the wallet by
    its browser-attested [=origin=] and by the <code>app</code> block of its
    [=AppConnectQuery=].</dd>

  <dt><dfn data-lt="AppConnectQuery|app connect query">App Connect query
    (AppConnectQuery)</dfn></dt>
  <dd>The verifiable presentation request query type this profile defines,
    extending the query types of [[VCALM]]: it names the requesting
    [=application=] and carries the capability requests to be delegated to that
    application's [=app-key credential=] subject. See [[[#request]]].</dd>

  <dt><dfn data-lt="App Connect exchanges|exchange">App Connect exchange</dfn></dt>
  <dd>One request/response round in which an [=application=] sends a Verifiable
    Presentation Request containing an [=AppConnectQuery=] and the [=wallet=]
    returns one signed Verifiable Presentation carrying the [=app-key
    credential=] and the delegated [=grants=]. The exchange is atomic from the
    application's point of view: it either yields both halves or it fails.</dd>

  <dt><dfn data-lt="app-key credentials|app key credential">app-key
    credential</dfn></dt>
  <dd>A self-issued Verifiable Credential [[VC-DATA-MODEL]] carrying an
    application's 32-byte [=seed=], whose issuer and subject are both the
    [=connecting DID=] derived from that seed, and which is bound to the
    application's [=origin=]. It is held in the user's wallet, and is what
    makes an application's identity portable across the user's browsers without
    the application holding any durable secret of its own. See
    [[[#app-key-credential]]].</dd>

  <dt><dfn data-lt="capability|capabilities|zcap|zCap">capability (zCap)</dfn></dt>
  <dd>An object capability authorizing a set of [=actions=] on a [=target=].
    See the <a
    href="https://interop-alliance.github.io/zcap-developer-guide/">zCap
    Developer Guide</a> and [[ZCAP]] for the model, and [[WAS]] for the storage
    profile of it that this document produces delegations under.</dd>

  <dt><dfn data-lt="CHAPI">Credential Handler API (CHAPI)</dfn></dt>
  <dd>The browser-mediated transport over which an App Connect exchange is
    carried [[CHAPI]]. Its relevant property here is that it attests the
    requesting [=origin=] to the wallet: the wallet learns which origin opened
    the request from the mediator, not from the request body.</dd>

  <dt><dfn data-lt="collections|Collection">collection</dfn></dt>
  <dd>See [[WAS]]. A namespace and configuration container for Resources
    within a [=Space=]. Every capability an App Connect response carries is
    scoped to a collection or to the Space itself.</dd>

  <dt><dfn data-lt="connecting DID|subject DID|controller DID">connecting DID
    (subject DID)</dfn></dt>
  <dd>The <code>did:key</code> DID derived from an [=app-key credential=]'s
    [=seed=] ([[[#key-derivation]]]), which is simultaneously that credential's
    <code>issuer</code>, its <code>credentialSubject.id</code>, and the
    <code>controller</code> of every [=grant=] the wallet delegates in the same
    exchange. It is per (application, user), not per browser or per
    machine.</dd>

  <dt><dfn data-lt="epoch roster recipient|roster recipient">epoch-roster
    recipient</dfn></dt>
  <dd>An entity able to decrypt an encrypted [=collection=]'s current contents
    because its key-agreement key is a recipient of that collection's current
    key epoch. Recipiency is the read axis of an encrypted collection, distinct
    from the fetch axis a [=capability=] grants. Defined normatively in
    [[WAS-EC]]; referenced here only where the App Connect exchange establishes
    or requires it.</dd>

  <dt><dfn data-lt="grants|granted capability">grant</dfn></dt>
  <dd>One delegated [=capability=] carried in an App Connect response
    presentation's <code>zcap</code> array: a capability whose
    <code>controller</code> is the [=connecting DID=], whose
    <code>invocationTarget</code> is a URL inside the user's [=Space=], and
    whose <code>allowedAction</code> has been capped per
    [[[#action-ceilings]]].</dd>

  <dt><dfn data-lt="invocation target descriptors|descriptor">invocation
    target descriptor</dfn></dt>
  <dd>The value of a capability request's <code>invocationTarget</code>: either
    an absolute URL, or an abstract object of the form
    <code>{ type, name }</code> that the [=wallet=] resolves against the user's
    own [=Space=]. The descriptor form exists because the requesting
    [=application=] does not know where the user's Space lives. See
    [[[#descriptors]]].</dd>

  <dt><dfn data-lt="origins|requesting origin">origin</dfn></dt>
  <dd>The web origin of the requesting [=application=], as attested by the
    [=CHAPI=] mediator to the wallet (not as asserted in the request body). It
    is the anti-phishing bind on the [=app-key credential=]: a credential
    minted for one origin is never returned to another.</dd>

  <dt><dfn data-lt="Resource|resources">resource</dfn></dt>
  <dd>See [[WAS]]. The addressable unit of storage inside a
    [=collection=].</dd>

  <dt><dfn data-lt="seeds">seed</dfn></dt>
  <dd>32 bytes of randomness minted by the [=wallet=], carried in the
    [=app-key credential=]'s <code>credentialSubject.seed</code> claim, from
    which the [=application=] deterministically derives its signing key, its
    [=connecting DID=], and its key-agreement key. It is the application's
    client secret; nothing else in the exchange substitutes for it.</dd>

  <dt><dfn data-lt="Spaces|Space">space</dfn></dt>
  <dd>See [[WAS]]. The user-owned top-level container that holds
    [=collections=]. Every App Connect [=grant=] resolves to a target inside
    exactly one Space.</dd>

  <dt><dfn data-lt="target|invocationTarget|targets">target
    (invocationTarget)</dfn></dt>
  <dd>See [[WAS]]. The resource a capability authorizes action on, as an
    absolute URL.</dd>

  <dt><dfn data-lt="wallets">wallet</dfn></dt>
  <dd>The software that holds the user's credentials and controls the user's
    [=Space=], and that processes an [=AppConnectQuery=]. One of the two
    conformance classes; see [[[#conformance-classes]]]. A wallet is able to
    delegate capabilities because it can invoke the Space's root
    capability.</dd>
</dl>

## The App Connect Request {#request}

### Carriage {#request-carriage}

An App Connect request is a verifiable presentation request [[VCALM]] whose
`query` member contains exactly one [=AppConnectQuery=]. The request is
delivered to the wallet over [=CHAPI=].

The request body MUST carry `challenge` and `domain` members
([[[#challenge-and-domain]]]).

An [=application=] MUST NOT rely on any transport-level identification of
itself other than the [=origin=] the transport attests. In particular, the
`app` block described below is display and matching metadata; it is not
evidence of who is asking.

### The AppConnectQuery {#appconnectquery}

An [=AppConnectQuery=] is a JSON object with the following members.

| Member | Required | Value |
|--------|----------|-------|
| `type` | yes | The string `AppConnectQuery`. |
| `app` | yes | An object naming the requesting application; see below. |
| `capabilityQuery` | no | A capability request entry, or an array of them; see [[[#capability-query]]]. |

The `app` object has exactly three members, and a [=wallet=] MUST treat the
query as malformed unless all three are present and are strings:

| Member | Value |
|--------|-------|
| `name` | A human-readable application name, for the wallet's consent surface. Display only; it is attacker-controlled free text and MUST NOT be treated as evidence of identity. |
| `credentialType` | The application's own [=app-key credential=] type name, used to match an existing credential or to mint a new one. See [[[#app-key-type-array]]]. |
| `vocabBase` | The vocabulary base IRI under which the application's own type term is minted. See [[[#app-key-context]]]. |

An [=application=] MUST NOT use the empty string, `VerifiableCredential`, or
`AppKeyCredential` as its `credentialType`. A [=wallet=] SHOULD treat a query
naming one of these as malformed.

<div class="note">
The application's own type term is the one term the inline credential context
mints under `vocabBase` ([[[#app-key-context]]]). A `credentialType` of
`AppKeyCredential` would therefore redefine the marker term to
`<vocabBase>AppKeyCredential`, defeating the rule that the marker IRI is one
stable IRI for every application and is never interpolated
([[[#app-key-iris]]]). `VerifiableCredential` and the empty string collide with
the base vocabulary in the same way.
</div>

A [=wallet=] that does not recognize the `AppConnectQuery` type MUST NOT
attempt to satisfy it partially. Its response will therefore carry no [=app-key
credential=], which the application detects per [[[#wallet-unsupported]]].

### Exclusivity {#request-exclusivity}

An App Connect request is one mental model per exchange, and the rules below
keep it that way. A [=wallet=] MUST enforce all of them, and MUST treat a
violation as a malformed request rather than resolving it in the application's
favour:

1. A request MUST NOT carry more than one `AppConnectQuery`.
2. A request carrying an `AppConnectQuery` MUST NOT also carry a
   `QueryByExample` query.
3. A request carrying an `AppConnectQuery` MUST NOT also carry an
   `AuthorizationCapabilityQuery` query or its legacy `ZcapQuery` alias
   ([[[#legacy-alias]]]).
4. A request carrying an `AppConnectQuery` MAY also carry a
   `DIDAuthentication` query [[VCALM]], and a conformant
   [=application=] MUST carry one. A [=wallet=] MUST accept a request that
   carries none, and answers it with an unsigned presentation.

<div class="note">
Rule 4 is what makes rules 1 to 3 safe to state so bluntly. The
`DIDAuthentication` pairing is what causes the wallet to sign the response
presentation, and the signature is what binds the embedded [=grants=] and the
`appConnect` marker to this request's challenge and domain
([[[#response-members]]]). An App Connect request without a
`DIDAuthentication` query receives an unsigned presentation, whose grants still
self-authenticate through their own delegation proofs but whose freshness is
unattested -- which is why the requirement is asymmetric: the wallet tolerates
the unsigned case, and the application conformance class, whose verification
procedure ([[[#response-verification]]]) is defined over a signed response, does
not use it.
</div>

The exclusivity rules exist because an App Connect exchange has one consent
surface describing one relationship. Mixing in a credential-sharing query or a
standalone capability query would put two unrelated decisions behind one
approval, with the second one described by the requesting party's own free-text
`reason` strings.

### Capability requests {#capability-query}

The `capabilityQuery` member holds the capability requests to be delegated to
the [=connecting DID=]. A [=wallet=] MUST normalize it as follows:

* an **absent** `capabilityQuery` normalizes to the empty list. A request that
  asks for no capabilities at all is legal and is satisfied by returning the
  [=app-key credential=] alone: an application may connect purely to recover
  its own key material.
* a **single object** normalizes to a one-element list.
* an **array** normalizes to itself.
* an entry that is not an object is malformed, and the wallet MUST treat the
  whole query as malformed rather than skipping the entry.

Each entry has the shape of a capability request detail [[VCALM]], **minus two
members**:

| Member | Required | Value |
|--------|----------|-------|
| `invocationTarget` | yes | An [=invocation target descriptor=]; see [[[#descriptors]]]. |
| `allowedAction` | no | A string or array of strings naming the requested [=actions=]. Absent means read-only; see [[[#default-actions]]]. |
| `referenceId` | no | An application-chosen label. Opaque to the wallet; see [[[#reference-id]]]. |

* An entry MUST NOT carry a `controller`. The [=wallet=] fills it with the
  subject DID of the [=app-key credential=] it matched or minted
  ([[[#delegation-target]]]). This is the member an application could not
  supply on first run, and dropping it is what collapses the flow to one
  round. A wallet MUST ignore any `controller` an entry does carry.
* An entry MUST NOT carry a `reason`. The App Connect consent surface describes
  the relationship as a whole and supersedes per-grant reason strings
  ([[[#consent]]]). A wallet MUST NOT display a `reason` an entry does carry.

An [=application=] SHOULD NOT name the same collection in more than one request
entry. If it does, which of the resulting [=grants=] governs the application's
routing for that collection is unspecified by this profile.

#### Correlating entries with grants {#reference-id}

`referenceId` is **opaque to the wallet** and is **not echoed** in the response.
A wallet MUST NOT be required to carry it onto a delegated grant, and an
[=application=] MUST NOT rely on it appearing there.

An application MUST correlate a returned [=grant=] with the request entry that
produced it by the grant's `invocationTarget` URL -- specifically by the
collection segment of that URL ([[[#url-template]]]).

An application MUST NOT correlate by position. Unsatisfiable entries produce no
grant and are skipped, so the response's `zcap` array is not index-aligned with
the request's `capabilityQuery` array and is in general shorter than it.

### Challenge and domain {#challenge-and-domain}

An [=application=] MUST generate a fresh, unpredictable `challenge` for every
App Connect request. It MUST NOT reuse a challenge across requests, and MUST
retain the value it sent in order to check the response against it
([[[#response-verification]]]).

An application MUST set `domain` to its own live browser origin -- the origin
the [=CHAPI=] mediator will attest to the wallet -- and not to any other value.

A [=wallet=] MUST refuse a request whose `domain` does not match the attested
requesting [=origin=]. The comparison is on host. A wallet MUST apply this
check to any request carrying a `domain`, whether or not it also carries a
`DIDAuthentication` query, and MUST surface a domain mismatch as a distinct
refusal rather than as a generic processing error.

<div class="note">
A `domain` that names an origin other than the channel the request arrived on
is the signature of a relay: some other party's verifier is borrowing this
user's wallet to answer a challenge issued elsewhere. Refusing it is a
replay-protection measure, which is why the check does not depend on
`DIDAuthentication` being present -- a request can pin a foreign domain without
asking for a proof at all.
</div>

A [=wallet=] that signs the response MUST echo both the request's `challenge`
and its `domain` into the authentication proof.

### The legacy ZcapQuery alias {#legacy-alias}

On the standalone capability-request channel -- that is, a request that carries
no `AppConnectQuery` -- a [=wallet=] MUST accept the query type string
`ZcapQuery` as an alias of `AuthorizationCapabilityQuery` [[VCALM]], with
identical processing.

The alias exists for deployed requesters that predate the canonical spelling.
It is **not** part of the App Connect exchange: per [[[#request-exclusivity]]],
an `AppConnectQuery` may not co-occur with either spelling. New implementations
MUST emit `AuthorizationCapabilityQuery`.

### Example request {#request-example}

A complete App Connect request: DID authentication plus one App Connect query
asking for one private application collection, one public collection, and
read-and-decrypt access to one collection the wallet already owns.

```json
{
  "query": [
    {
      "type": "DIDAuthentication",
      "acceptedMethods": [{ "method": "key" }]
    },
    {
      "type": "AppConnectQuery",
      "app": {
        "name": "Example Notes",
        "credentialType": "ExampleNotesUser",
        "vocabBase": "https://app.example/vocab#"
      },
      "capabilityQuery": [
        {
          "referenceId": "notes",
          "allowedAction": ["GET", "HEAD", "PUT", "POST", "DELETE"],
          "invocationTarget": {
            "type": "https://w3id.org/byoe#collection",
            "name": "notes"
          }
        },
        {
          "referenceId": "published-notes",
          "allowedAction": ["GET", "HEAD", "POST"],
          "invocationTarget": {
            "type": "https://w3id.org/byoe#public-collection",
            "name": "published-notes"
          }
        },
        {
          "referenceId": "contacts",
          "allowedAction": ["GET", "HEAD"],
          "invocationTarget": {
            "type": "https://w3id.org/byoe#shared-collection",
            "name": "contacts"
          }
        }
      ]
    }
  ],
  "challenge": "0b1c9a1e-6a5f-4a2e-9c8a-2f7f4d3b6c11",
  "domain": "https://app.example"
}
```

## Invocation Target Descriptors {#descriptors}

### The descriptor model {#descriptor-model}

A capability request names what it wants access to in its `invocationTarget`.
An [=application=] performing App Connect does not know the URL of the user's
[=Space=] -- it does not know the storage host, and it does not know the Space
id -- so it cannot name a concrete target. It names an **abstract descriptor**
instead, and the [=wallet=] resolves the descriptor against the Space it
controls.

Resolution yields either a concrete target inside the user's Space, together
with the target class that bounds it ([[[#action-ceilings]]]), or the verdict
**unsatisfiable**.

An `invocationTarget` MUST be either:

* an **object** with a `type` member holding a descriptor type IRI, and an
  optional `name` member; or
* a **string** holding an absolute URL ([[[#string-targets]]]).

A [=wallet=] MUST determine the target class from the resolved target itself,
so that a descriptor object and the equivalent URL string resolve to the same
class and are bounded identically. A string target MUST NOT be able to reach a
looser ceiling than the descriptor form of the same target.

Honoring that parity requires the wallet to know the standing of each of its own
collections independently of how a request named it -- in particular which of
them are public and which are protected -- since a string target carries no
descriptor type to classify it by.

An **unsatisfiable** request entry MUST NOT produce a [=grant=]. The wallet
MUST skip it at delegation time and MUST show it on the consent surface as
something it cannot fulfill ([[[#consent]]]).

### The target URL template {#url-template}

Every [=target=] this profile produces is a URL under the [[WAS]] URL template:

```
<host>/space/{space_id}
<host>/space/{space_id}/{collection_id}
<host>/space/{space_id}/{collection_id}/...
```

A [=wallet=] MUST resolve descriptors to URLs of this shape, where
`<spaceUrl>` in [[[#descriptor-registry]]] denotes
`<host>/space/{space_id}` for the [=Space=] it controls.

An [=application=] MUST derive the storage host, the [=Space=] id, and its
per-collection routing by parsing each grant's `invocationTarget` under this
template. It MUST NOT be told these values by any other means, and MUST NOT
derive the Space id itself.

<div class="note">
Spaces mounted below a path prefix -- a deployment serving
`https://wallet-storage.example/tenant-a/space/{space_id}` -- are out of scope
for this version of the profile. Both parsing rules above, and the string-target
rules in [[[#string-targets]]], assume `/space/` sits directly under the host.
</div>

### Descriptor type registry {#descriptor-registry}

This registry is normative. The following descriptor types are defined.

| Type IRI | `name` | Resolves to | Action ceiling |
|----------|--------|-------------|----------------|
| `https://w3id.org/byoe#collection` | required | `<spaceUrl>/<name>` | `GET`, `HEAD`, `POST`, `PUT`, `DELETE` |
| `https://w3id.org/byoe#public-collection` | required | `<spaceUrl>/<name>` | `GET`, `HEAD`, `POST` |
| `https://w3id.org/byoe#shared-collection` | required | `<spaceUrl>/<name>` | `GET`, `HEAD` |
| `https://w3id.org/byoe#space` | ignored | `<spaceUrl>` | `GET`, `HEAD` |

Where `name` is required, a [=wallet=] MUST validate it against the collection
name grammar ([[[#collection-name-grammar]]]) and MUST resolve the descriptor
unsatisfiable when it does not match.

#### `#collection` {#descriptor-collection}

`https://w3id.org/byoe#collection` requests an application-scoped
[=collection=] in the user's Space, named by `name`.

* It resolves to `<spaceUrl>/<name>`.
* If the named collection does not already exist, the wallet MUST provision it
  before delegating, per [[[#provisioning]]].
* A collection provisioned under this descriptor MUST be encrypted from
  creation ([[[#encrypted-by-default]]]).
* Its ceiling is the full action vocabulary: this is the application's own
  data, and the consent surface plus the shorter write lifetime
  ([[[#ttl]]]) are what bound it.
* If `name` happens to be one of the wallet's own protected collections
  ([[[#protected-collections]]]), the resolved class is
  *protected collection* and the read-only ceiling applies instead. A wallet
  MUST NOT provision over a protected collection.

#### `#public-collection` {#descriptor-public-collection}

`https://w3id.org/byoe#public-collection` requests a [=collection=] whose
contents are readable by anyone on the web without authorization.

* It resolves to `<spaceUrl>/<name>`.
* The wallet MUST provision it as **plaintext** -- a public collection is never
  encrypted -- and MUST set a collection-level world-readable access policy on
  it. The policy is set by the wallet, which controls the Space; a delegated
  capability could not set it.
* Public covers unauthenticated **reads only**. Writes remain
  capability-authorized, so the wallet still delegates an ordinary
  collection-scoped capability alongside setting the policy.
* Its ceiling is **add-only**: `GET`, `HEAD`, and `POST`, never `PUT` or
  `DELETE`. A write to a plaintext world-readable target is not data management
  but publication under the user's identity, and is irreversible in practice:
  retracting removes the link, not the copies already fetched. An application
  may add to what it published; it may never rewrite or retract it.
* **A public collection is only ever created public, never converted.** A
  `#public-collection` descriptor naming a collection that **already exists and
  is not already public** MUST resolve **unsatisfiable**. A wallet MUST NOT
  make an existing non-public collection world-readable in response to a
  request.
* An idempotent re-grant is unaffected: a `#public-collection` descriptor
  naming a collection that already exists **and is already public** stays
  satisfiable, and reconnecting an application to the collection it previously
  published into behaves exactly as the first connect did
  ([[[#provisioning]]]).
* A `#public-collection` descriptor naming a protected collection
  ([[[#protected-collections]]]) MUST resolve **unsatisfiable**,
  unconditionally. This is the special case of the conversion rule that holds
  even where a wallet's own bookkeeping is uncertain: a requesting party can
  never flip a user's own credentials, activity, or published identity public.

<div class="note">
The conversion rule is broader than the protected-collection refusal on
purpose. Protected collections are the data a user would obviously not want
published, but they are not the only such data: a collection an application
provisioned privately on an earlier connect, and has been writing private data
into ever since, is equally not something a later request should be able to
turn world-readable. Making publicness a property fixed at creation removes the
whole class of conversion requests rather than enumerating which conversions are
unacceptable.
</div>

#### `#shared-collection` {#descriptor-shared-collection}

`https://w3id.org/byoe#shared-collection` requests read-and-**decrypt** access
to an encrypted collection the wallet already owns -- not merely the ability to
fetch its ciphertext.

* It resolves to `<spaceUrl>/<name>`.
* `name` MUST name a collection **the wallet itself owns and encrypts** -- one
  of the wallet's own standard encrypted collections. A plaintext collection, a
  collection provisioned for an application, the [=Space=] itself, and a name
  the wallet does not recognize MUST all resolve **unsatisfiable**.
* Because satisfiability is decided by looking the name up in the set of
  collections the wallet owns and encrypts, that lookup subsumes the collection
  name grammar check ([[[#collection-name-grammar]]]) for this descriptor type:
  a name that fails the grammar cannot be in the set.
* Its granted actions are exactly the class ceiling, `GET` and `HEAD`
  ([[[#action-ceilings]]]) -- not an intersection with what was requested. A
  share is never a write grant, and `HEAD` rides every read grant
  ([[[#action-vocabulary]]]), so a share that asked for a narrower subset still
  receives both.
* **The two axes are fused.** Satisfying a share means granting *both* the
  read-only [=capability=] (the fetch axis) *and* [=epoch-roster recipient=]
  status for the collection's current epoch (the decrypt axis). A [=wallet=]
  MUST NOT grant either axis alone, and MUST NOT allow a **durable partial
  state** in which one axis stands without the other. Where the two cannot be
  established in a single atomic step, the wallet MUST complete or repair the
  missing half -- at the latest on a re-run of the exchange
  ([[[#resumability]]]) -- and MUST NOT report the share as granted until both
  halves stand.
* The recipient key is never carried in the request; see
  [[[#recipient-derivation]]].

Granting only the capability would hand the application ciphertext it cannot
open, which surfaces to a user as corrupt data rather than as a failed share.
Granting only recipiency would leave a reader in the roster with no way to
fetch. Both halves are load-bearing.

**A share covers what is already stored.** Recipient status on the current
epoch lets the grantee decrypt the collection's existing contents, not only what
is written after the share. A [=wallet=] MUST state this on the consent surface
([[[#consent]]]).

**Re-granting a previously revoked grantee restores read access to previously
written content.** Adding a grantee back as an [=epoch-roster recipient=] gives
it the collection as it stands, including whatever was written while it was
revoked. A wallet MUST NOT present a re-grant as forward-only access. Restoring
access from this point forward only, rather than retroactively, requires
rotating the collection to a new key epoch and admitting the grantee to that
epoch alone; the epoch construction that makes this possible is defined in
[[WAS-EC]].

**Revocation of a share is an explicit act, never an expiry.** A [=wallet=]
MUST NOT rely on the delegated capability's lifetime as the removal mechanism
for a share: expiry ends the fetch axis while leaving the grantee an
epoch-roster recipient, which is precisely the unfused state the fusion rule
exists to prevent. Removing a share MUST re-epoch the collection off the
removed recipient and revoke the capability as one operation. See also
[[[#ttl]]].

#### `#space` {#descriptor-space}

`https://w3id.org/byoe#space` requests the whole [=Space=] as a target. It
resolves to the Space URL, and its ceiling is read-only.

The read-only ceiling is unconditional: a Space-wide write capability would
authorize rewriting the Space Description, and therefore controller takeover.

<div class="note">
The shipped application-side implementation never emits this descriptor, and
its grant parser rejects a capability that is not collection-scoped. It is
specified because a wallet must bound it correctly if some other requester does
emit it.
</div>

### Reserved descriptor types {#descriptor-reserved}

The following type IRIs are **reserved**. This document defines no behavior for
them; they are recorded here so that no other specification, profile, or
implementation assigns them different semantics.

| Reserved type IRI | Reserved for |
|-------------------|--------------|
| `https://w3id.org/byoe#managed-collection` | A future write-bearing grant class for applications acting as the authoritative writer of a collection. |
| `https://w3id.org/byoe#publish-collection` | A future standing publication grant. |
| `https://w3id.org/byoe#collection-policy` | A future runtime access-policy control descriptor. |

Until this document (or a successor) defines them, a reserved type is an
unrecognized type and MUST be processed per [[[#unknown-descriptor-type]]]:
a wallet that predates such a feature resolves it unsatisfiable and refuses
visibly.

### Unrecognized descriptor types {#unknown-descriptor-type}

A [=wallet=] MUST resolve **any** `invocationTarget` descriptor whose `type` it
does not recognize as **unsatisfiable**.

A wallet MUST NOT fall back to a related recognized type, MUST NOT strip the
type and treat the descriptor as a plain collection request, and MUST NOT
silently narrow the request into something it does understand.

<div class="note">
This is the extensibility safety rule of the whole profile, and it is what
makes new descriptor types deployable at all. The application-side reasoning
runs the other way around: an application that asks for a
`#public-collection` and is answered by a wallet that predates the type sees a
visible refusal, and knows its collection was not created. If the wallet had
instead degraded the request to `#collection`, the application would have been
handed a *private* collection it believes is public, and would publish into it.
The same argument covers `#shared-collection`, whose fused axes cannot be
partially honored, and the `AppConnectQuery` type itself
([[[#wallet-unsupported]]]).
</div>

### String targets {#string-targets}

A capability request MAY carry an absolute URL string as its
`invocationTarget`. A [=wallet=] MUST resolve it as follows.

The wallet MUST parse both the target and its own Space URL as URLs, and MUST
resolve the target **unsatisfiable** if any of the following holds:

1. either value does not parse as an absolute URL;
2. the target carries a **query** component;
3. the target carries a **fragment** component;
4. the target's origin differs from the Space URL's origin;
5. after resolving dot segments and trimming trailing slashes, the target's
   path is neither equal to the Space path nor a descendant of it.

Otherwise:

* if the target's path equals the Space path (with or without a trailing
  slash), the target is the Space itself, resolved with the *space* class and
  its read-only ceiling;
* otherwise the first path segment after the Space path is the collection id.
  It MUST match the collection name grammar ([[[#collection-name-grammar]]]) or
  the target resolves unsatisfiable. The resolved class is *protected
  collection* if that id names a protected collection
  ([[[#protected-collections]]]), and *collection* otherwise.

A string target MUST NOT cause provisioning: it names something the wallet
either already has or does not.

<div class="note">
Rules 2 and 3 refuse rather than rewrite. A [[WAS]] resource URL carries
neither a query nor a fragment, so a target that has one is not a target the
server would route as written; and silently dropping part of a target the user
is about to approve would show the user something other than what gets
delegated. Rules 4 and 5 together are why string matching alone is not enough:
`<spaceUrl>/private-credentials?x=1` starts with the Space URL but its first
routed segment is not the collection id it appears to name, and
`<spaceUrl>/../other-space/x` starts with the Space URL while pointing outside
the Space entirely.
</div>

### Collection name grammar {#collection-name-grammar}

A collection name appearing in a descriptor's `name` member, or as the first
path segment of a string target, MUST match:

```
/^[a-z0-9][a-z0-9-]{0,63}$/
```

That is: lowercase ASCII letters and digits and the hyphen, not starting with a
hyphen, between 1 and 64 characters. A name that does not match resolves the
descriptor unsatisfiable.

### Protected collections {#protected-collections}

A [=wallet=] MUST maintain a set of **protected collections**: the collections
in the user's Space that hold the user's own wallet data and the account's
system state. At minimum this set MUST include:

* the wallet's own standard content collections -- the collections the wallet
  itself writes the user's credentials, activity, and application-domain data
  into; and
* the account's system collections -- those holding the account's published
  identity artifacts and its key material.

Requests resolving onto a protected collection are bounded as follows:

* the resolved class is *protected collection*, whose ceiling is read-only
  ([[[#action-ceilings]]]). A requesting party may read the user's own data
  when the user consents, and may never rewrite or delete it;
* a `#public-collection` descriptor naming one MUST resolve unsatisfiable
  ([[[#descriptor-public-collection]]]);
* a wallet MUST NOT provision over one.

A `#shared-collection` descriptor naming an encrypted protected collection is
the intended way for an application to be given read access to the user's own
data, and resolves in the *share* class rather than the *protected collection*
class. Both ceilings are read-only, so the distinction affects only the
decrypt axis and the consent copy.

## The App Key Credential {#app-key-credential}

### Purpose {#app-key-purpose}

An [=application=] running entirely in the browser has nowhere durable and
trustworthy to keep a secret of its own. The [=app-key credential=] answers
this by moving custody: the wallet holds the application's [=seed=] as an
ordinary credential in the user's wallet, and returns it to the same
application, at the same [=origin=], on every connect. The application's
identity is therefore stable **by custody**, not because the application
managed to hold onto anything.

The credential is self-issued and self-authenticating: its issuer, its subject,
and the DID derived from the seed it carries are all the same value
([[[#seed-binds-subject]]]). That is what lets a wallet hand it back after any
interval, and what lets an application check it without consulting anyone.

### Type array {#app-key-type-array}

The credential's `type` member MUST be an array of exactly three entries, in
this order:

```json
["VerifiableCredential", "AppKeyCredential", "<credentialType>"]
```

where `<credentialType>` is the `app.credentialType` value from the request.

`AppKeyCredential` is the **marker type**. It makes "presents as an app key" a
term check rather than a shape heuristic, which is what the store-time refusal
([[[#store-time-refusal]]]) and the match predicate
([[[#app-key-matching]]]) key off.

### Term IRIs {#app-key-iris}

| Term | IRI | Scope |
|------|-----|-------|
| `AppKeyCredential` | `https://w3id.org/byoe#AppKeyCredential` | Shared by every application |
| `seed` | `https://w3id.org/byoe#seed` | Shared by every application |
| `origin` | `https://w3id.org/byoe#origin` | Shared by every application |
| `name` | `https://schema.org/name` | Shared |
| `description` | `https://schema.org/description` | Shared |
| `<credentialType>` | `<vocabBase><credentialType>` | The application's own |

The marker term IRI is **one stable IRI for every application**. It MUST NOT be
interpolated from the request's `vocabBase`. An application-scoped marker IRI
would mean the marker asserted a different thing for each application, and the
one rule that makes a store-time refusal possible ("this credential claims to
be an app key") would no longer be expressible.

The `seed` and `origin` claim IRIs are likewise shared: they mean the same
thing for every application, so they do not belong under a per-application
namespace. `vocabBase` namespaces exactly one term, the application's own type.

<div class="note">
**The marker is a self-declaration, not evidence.** The `type` array of a
planted credential is attacker-controlled like the rest of it. The marker makes
the profile's rules precise -- it is what a refusal and a match can be stated
over -- but the only thing that *authenticates* an app-key credential is the
seed-to-subject binding in [[[#seed-binds-subject]]].
</div>

### The inline context {#app-key-context}

The credential's `@context` MUST be an array whose first entry is the VC 1.1
context URL and whose second entry is an inline term-definition object of
exactly this shape:

```json
{
  "@protected": true,
  "AppKeyCredential": "https://w3id.org/byoe#AppKeyCredential",
  "<credentialType>": "<vocabBase><credentialType>",
  "seed": "https://w3id.org/byoe#seed",
  "origin": "https://w3id.org/byoe#origin",
  "name": "https://schema.org/name",
  "description": "https://schema.org/description"
}
```

Only the second term varies: it is interpolated from the request's `vocabBase`
and `credentialType`.

Carrying the terms inline, rather than by reference to a hosted context, is
deliberate: the credential stays verifiable with no remote vocabulary fetch and
no document-loader configuration on either side, which matters because both
parties verify it offline of each other.

The signature suite appends its own context entry when the credential is
signed; that entry is the suite's, not this profile's.

<div class="note">
The shared term IRIs used above are published as part of a context document at
`https://w3id.org/byoe/app-connect/v1`, which is a superset of what any one
credential's inline context carries: it defines the shared BYOE terms of this
profile as a whole, and necessarily cannot define an application's own type
term, which is minted per application under `vocabBase`. Implementations may
source the shared IRIs from it, but the credential as it appears on the wire
carries the object above inline.
</div>

### Seed encoding {#seed-encoding}

The `credentialSubject.seed` claim MUST be a string holding **32 random bytes**
encoded as base64url **without padding** [[RFC4648]].

Both parties MUST decode it and MUST require the result to be exactly 32 bytes.
A seed that is absent, is not a string, does not decode, or decodes to any
other length MUST cause the credential to be treated as not binding
([[[#seed-binds-subject]]]) -- fail closed, never truncate or pad.

### Key derivation {#key-derivation}

The [=connecting DID=] is derived from the [=seed=] as follows. This derivation
is a pinned input to the profile: every existing app-key credential's identity
depends on it, and changing any part of it orphans them all.

1. Compute a 32-byte Ed25519 key seed:

   ```
   keySeed = HMAC-SHA-256(key = seed, message = UTF-8("app-key"))
   ```

   The message string `app-key` is byte-significant and MUST be used exactly.

2. Generate the Ed25519 key pair [[RFC8032]] from `keySeed`.

3. Let `fp` be the `did:key` fingerprint of the resulting public key
   [[DID-KEY]] -- the multibase base58btc encoding of the multicodec-tagged
   Ed25519 public key. Then:

   * the [=connecting DID=] is `did:key:<fp>` (no fragment);
   * the corresponding verification method id is `did:key:<fp>#<fp>`.

The application's key-agreement key is the X25519 (Montgomery) twin of this
same key pair; its byte-level derivation and the recipient identifier built
from it are defined in [[WAS-EC]]. One key identity answers for both signing
and key agreement.

<div class="note">
Some implementations of this derivation take an additional "handle" or label
parameter alongside the key name. Such a parameter is cosmetic: it does not
enter the HMAC and does not affect the derived key or DID. Only the seed bytes
and the literal string `app-key` select the key.
</div>

### Self-issuance and origin binding {#app-key-binding}

An [=app-key credential=] MUST satisfy all of:

* **Self-issuance.** `issuer` and `credentialSubject.id` MUST be present and
  MUST be equal.
* **Seed-binds-subject.** That common value MUST be the [=connecting DID=]
  derived from `credentialSubject.seed` per [[[#key-derivation]]]. See
  [[[#seed-binds-subject]]].
* **Origin binding.** `credentialSubject.origin` MUST be the requesting
  [=origin=] the credential was minted for.

A [=wallet=] MUST set `credentialSubject.origin` to the origin the transport
attested, never to a value taken from the request body.

Origin binding is enforced **twice**, on purpose:

* the [=wallet=] enforces it at match time and at mint time, so a phishing
  origin can neither recover a credential minted for another origin nor be
  handed one bound to a different one;
* the [=application=] enforces it at parse time
  ([[[#app-key-parsing]]]), so an application does not depend on the
  wallet having done so.

### The seed-binds-subject rule {#seed-binds-subject}

To decide whether a credential **is** an app-key credential rather than merely
claiming to be one, an implementation MUST:

1. read `credentialSubject.seed` and decode it per [[[#seed-encoding]]];
2. re-derive the [=connecting DID=] from those bytes per
   [[[#key-derivation]]];
3. require the derived DID to equal `credentialSubject.id`.

The check MUST **fail closed**: an absent subject id, an absent seed, a seed
that does not decode, a seed of the wrong length, or a derivation error all
mean "does not bind". An implementation MUST NOT let such a credential proceed
on the grounds that some other check passed.

This rule is checked at three moments, and a conformant implementation MUST
apply it at each:

| Moment | Party | Consequence of failure |
|--------|-------|------------------------|
| Match time | Wallet | The credential is not a candidate; the wallet mints a fresh one instead. |
| Store time | Wallet | The credential is refused at ingest ([[[#store-time-refusal]]]). |
| Parse time | Application | The response is rejected ([[[#app-key-parsing]]]). |

<div class="note">
Self-issuance alone is a weak signal, because anyone can self-issue. The
binding is stronger and is entirely local: the credential carries the seed, so
the check is a re-derivation and a comparison, with no network and no trusted
third party.

**What the binding does and does not establish.** It establishes internal
consistency: that this credential's subject is the DID of the seed this
credential carries. It therefore rejects any credential in which the subject or
the seed has been substituted -- a genuine credential with the seed swapped
out, or an attacker's subject DID pasted over a genuine seed -- because either
substitution breaks the derivation.

It does **not** establish provenance, and it is important not to read it as
doing so. An attacker who generates their own seed, derives its DID honestly,
and self-issues a credential naming the victim's [=origin=] and the victim
application's `credentialType` produces a credential that binds perfectly well.
Nothing local can tell that credential from a legitimate one, because it *is* a
legitimate credential -- for the attacker's identity. Keeping such a credential
away from the user's wallet is the job of [[[#store-time-refusal]]], not of this
rule.
</div>

### Matching {#app-key-matching}

To find the [=app-key credential=] for a request, a [=wallet=] MUST select from
its stored credentials those satisfying **all** of:

1. the `type` array includes the `AppKeyCredential` marker;
2. the `type` array includes the request's `app.credentialType`;
3. `issuer` is present and equals `credentialSubject.id`;
4. `credentialSubject.origin` equals the attested requesting [=origin=];
5. the credential binds per [[[#seed-binds-subject]]].

The marker is **required** at match time, not merely tolerated. Requiring it
means a credential can only reach the delegation path by carrying the marker,
which is exactly what the store-time refusal screens.

When more than one credential qualifies, the wallet MUST rank the qualifying
credentials **latest-first by the instant their `issuanceDate` denotes** and
use the first. Ranking MUST be over instants, not over the literal strings: a
wallet MUST parse each `issuanceDate` as a date-time and compare the resulting
instants. A credential whose `issuanceDate` is absent or does not parse MUST
sort last.

A [=wallet=] MUST mint `issuanceDate` in the canonical UTC form -- a date-time
with a two-digit-per-component date and time, no fractional seconds beyond
those it means to express, and the `Z` designator rather than a numeric offset
(for example `2026-08-06T14:22:11Z`).

<div class="note">
Lexicographic comparison of two canonical-UTC forms gives the same order as
comparing the instants, which is why an implementation that both mints and
compares canonically will not observe a difference. The requirement is stated
over instants anyway, because the credentials being ranked are not necessarily
all minted by the same wallet, and a value carrying a numeric offset or
differing fractional-second precision sorts incorrectly under string
comparison. Ranking is the input to a security decision -- which DID the wallet
delegates to -- so it is specified over the values' meaning rather than their
spelling.
</div>

When none qualifies, the wallet MUST mint a fresh credential
([[[#app-key-minting]]]) and MUST report first run ([[[#first-run]]]).

### Minting {#app-key-minting}

To mint an [=app-key credential=], a [=wallet=] MUST:

1. generate 32 bytes from a cryptographically secure random source;
2. derive the [=connecting DID=] from them per [[[#key-derivation]]];
3. assemble the credential with the type array of
   [[[#app-key-type-array]]], the inline context of [[[#app-key-context]]],
   `issuer` and `credentialSubject.id` both set to the derived DID,
   `credentialSubject.seed` encoded per [[[#seed-encoding]]], and
   `credentialSubject.origin` set to the attested requesting origin;
4. sign it with a signature suite the application can verify, using a signer
   for the seed-derived key -- so that the credential is genuinely self-issued;
5. store it in the user's wallet before delegating.

Storing before delegating makes the operation resumable: if delegation fails,
the next attempt matches the stored credential as a returning connection rather
than minting a second identity for the same application and origin.

The credential's `id` SHOULD be a fresh `urn:uuid:` value.

The `name` and `description` members are display fields for the wallet's own
credential list. They are informative; this profile places no requirement on
their content beyond their term IRIs ([[[#app-key-iris]]]).

<div class="note">
The wallet mints the seed, rather than the application minting one and asking
the wallet to store it. This is what removes the second popup: a
store-then-request flow needs one exchange to deposit the key and another to
ask for grants, and it also means the seed exists application-side before the
user has consented to anything.
</div>

### Store-time refusal {#store-time-refusal}

**App-key credentials are wallet-minted, never imported.** The only way an
[=app-key credential=] legitimately enters a wallet's store is by that wallet
minting it during an App Connect exchange ([[[#app-key-minting]]]).

Accordingly, a [=wallet=] MUST refuse to store any credential that carries the
`AppKeyCredential` marker and did not originate in an [=App Connect exchange=]
this wallet performed. The refusal MUST NOT depend on whether the credential binds
per [[[#seed-binds-subject]]]: a marker-carrying credential arriving from
outside is refused **whether or not** it is internally consistent.

The refusal MUST be applied at every path by which a credential enters the
wallet's store from outside -- a credential offered over the transport, or
imported from a URL, a QR code, a file, or a paste.

A [=wallet=] MUST additionally refuse to store a marker-carrying credential that
does not bind per [[[#seed-binds-subject]]], on any path whatsoever. The two
refusals are independent, and either one alone is sufficient grounds to refuse.

A credential that does **not** carry the marker MUST NOT be caught by these
rules, even if it happens to carry a `seed` or `origin` claim.

<div class="note">
The provenance rule is the load-bearing one, and it is stricter than checking
the credential's internal consistency because internal consistency is not the
property under attack. An attacker can mint a perfectly well-formed app-key
credential for their own seed, name the victim application's `credentialType`
and the victim's [=origin=] in it, give it a future `issuanceDate`, and get the
user to import it. Every local check passes, because the credential is
genuine -- for the attacker's identity. It would then be stored, would satisfy
the match predicate ([[[#app-key-matching]]]), would win the latest-first
ranking on its future date, and the wallet would delegate the user's storage to
the attacker's DID on the next connect.

No inspection of the credential closes that path, because there is nothing
wrong with the credential. What closes it is refusing the *channel*: an app-key
credential has no legitimate reason to arrive from outside, so a wallet that
never accepts one from outside is never in a position to have to tell the two
apart.

The binding refusal is retained alongside it as defence in depth, covering
credentials already stored under an earlier, weaker rule, and any path a wallet
might later add.
</div>

<div class="note">
Refusing at ingest beats storing-and-ignoring. Both end with the planted
credential not being used, but only the refusal keeps it out of the user's
credential list, out of the user's Space, and out of any future code path that
might apply a weaker check than the match predicate does.
</div>

### Example credential {#app-key-example}

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    {
      "@protected": true,
      "AppKeyCredential": "https://w3id.org/byoe#AppKeyCredential",
      "ExampleNotesUser": "https://app.example/vocab#ExampleNotesUser",
      "seed": "https://w3id.org/byoe#seed",
      "origin": "https://w3id.org/byoe#origin",
      "name": "https://schema.org/name",
      "description": "https://schema.org/description"
    },
    "https://w3id.org/security/suites/ed25519-2020/v1"
  ],
  "id": "urn:uuid:3f2a8c40-9d51-4e0b-9f1c-5c6d0a2b7e34",
  "type": ["VerifiableCredential", "AppKeyCredential", "ExampleNotesUser"],
  "name": "Example Notes app key",
  "description": "The Example Notes app keeps this key in your wallet so it can open your encrypted data on this and other devices.",
  "issuer": "did:key:z6MkExampleNotesAppKeySubjectDidPlaceholder",
  "issuanceDate": "2026-08-06T14:22:11Z",
  "credentialSubject": {
    "id": "did:key:z6MkExampleNotesAppKeySubjectDidPlaceholder",
    "seed": "b0dRQXBwS2V5U2VlZFBsYWNlaG9sZGVyVmFsdWUzMkI",
    "origin": "https://app.example"
  },
  "proof": { "...": "..." }
}
```

<div class="note">
The third `@context` entry is contributed by the signature suite, not by this
profile.

The `seed` value above is a well-formed 43-character base64url-no-pad encoding
of 32 bytes, but those bytes are readable filler rather than random, and the
DID shown is a legible placeholder rather than the DID that derivation over
this seed would actually produce. A real credential carries a
cryptographically random seed and the DID genuinely derived from it
([[[#key-derivation]]]) -- and would fail
[[[#seed-binds-subject]]] if it did not.
</div>

## The Response Presentation {#response}

### Members {#response-members}

A [=wallet=] answering an App Connect request MUST return a single Verifiable
Presentation carrying:

| Member | Value |
|--------|-------|
| `verifiableCredential` | An array containing the matched or minted [=app-key credential=]. |
| `zcap` | An array of the delegated [=grants=]. Omitted when no grant was made. |
| `appConnect` | `{ "firstRun": <boolean> }`. See [[[#first-run]]]. |

The `zcap` array holds one entry per **satisfiable** request entry that was
approved. Unsatisfiable and declined entries produce no entry, so the array is
in general shorter than the request's `capabilityQuery` and is not index-aligned
with it; see [[[#reference-id]]] for how an application correlates the two.

Whichever of these members the response carries MUST be added to the
presentation **before** it is signed, so that the authentication proof covers
them. A wallet MUST NOT append grants or the `appConnect` marker to an
already-signed presentation. A member that is empty is omitted, together with
its `@context` term definition ([[[#response-context]]]); omission is not a way
to add it later.

The presentation is signed by the **wallet's** holder DID, not by the
[=connecting DID=]. The application's DID never leaves the application, so it
cannot be the holder. The two attestations are separate and complementary: the
authentication proof attests that this wallet answered this challenge from this
domain, and the [=app-key credential=]'s self-issued proof plus the
seed-to-subject binding attest the identity being handed over.

Each entry in `zcap` is a self-contained delegated capability carrying its own
`@context` and its own delegation proof; the entries self-authenticate
independently of the presentation's proof.

### Context terms {#response-context}

When embedding grants, a wallet MUST append to the presentation's `@context` a
term-definition object defining `zcap`:

```json
{
  "@protected": true,
  "zcap": { "@id": "https://w3id.org/byoe#zcap", "@container": "@set" }
}
```

When embedding the App Connect marker, a wallet MUST append a term-definition
object defining `appConnect`:

```json
{
  "@protected": true,
  "appConnect": { "@id": "https://w3id.org/byoe#appConnect", "@type": "@json" }
}
```

Both term definitions are appended **before** signing ([[[#response-members]]]),
so the signature suite's own context entry -- appended by the suite when the
presentation is signed -- follows them in the resulting `@context` array. A
member that is omitted takes its term definition with it.

Only the top-level terms are defined. The embedded capabilities' own
sub-vocabularies MUST NOT be hoisted into the presentation context; each
embedded capability self-describes through its own `@context`.

Defining the terms is what allows JSON-LD safe-mode canonicalization
[[JSON-LD11]] to include the grants and the marker in the signed payload rather
than reject them, which is what makes "the proof covers them" true rather than
aspirational. The `appConnect` value is typed `@json` so that its contents
canonicalize as one opaque literal.

These IRIs are byte-significant: they are canonicalized into the authentication
proof.

### The firstRun marker {#first-run}

A [=wallet=] MUST set `appConnect.firstRun` to `true` if and only if it minted
a new [=app-key credential=] during this exchange, and to `false` when it
matched an existing one.

An [=application=] MUST treat **only** the boolean value `true` as first run.
An absent `appConnect` member, an absent `firstRun` member, and any non-boolean
or `false` value all mean "not first run".

`firstRun` is advisory: it tells an application whether to run whatever
one-time setup it does for a brand-new user. It is not a security signal, and
an application MUST NOT use it to decide whether to trust the credential.

### Application-side verification {#response-verification}

An [=application=] MUST perform the following steps, in this order, and MUST
abort on the first failure.

1. **Verify the presentation cryptographically.** Verify the presentation
   proof and every embedded credential proof. Issuer-registry lookup MUST NOT
   be required: the [=app-key credential=] is self-issued by design.

2. **Check the proof's purpose, challenge, and domain.** The presentation MUST
   carry at least one presentation-level proof. Every presentation-level proof
   MUST have `proofPurpose` of `authentication`. Every such proof's `challenge`
   MUST equal the fresh challenge this request sent, and its `domain` MUST
   equal the origin this request sent.

   <div class="note">
   Requiring *every* presentation-level proof to be an authentication proof,
   rather than just finding one, is deliberate. A verifier selects one proof
   purpose for the whole set and skips the rest; a presentation ordered
   `[assertionMethod, authentication]` could therefore verify under the
   assertion purpose with the authentication proof -- the only freshness bind
   -- never signature-checked, while the challenge and domain read off it
   anyway.
   </div>

3. **Locate the app-key credential.** Search the embedded credentials for one
   whose `type` array includes the application's own `credentialType`. The
   `AppKeyCredential` marker MUST NOT be required at this step.

   If no such credential is present, the outcome is
   [[[#wallet-unsupported]]].

   <div class="note">
   Matching on the application's own type alone, and requiring the marker only
   at the next step, is what keeps "the wallet returned a credential that is
   wrong" from being indistinguishable from "the wallet returned nothing". If
   the marker were required here, a returned credential missing it would look
   like an absent credential, and an application that reads absence as first
   run would answer by minting a second key.
   </div>

4. **Parse the credential.** Apply the parse checks of
   [[[#app-key-parsing]]], obtaining the [=seed=] and the [=connecting DID=].

5. **Validate the grants.** Apply the grant checks of
   [[[#grant-validation]]] against the [=connecting DID=] obtained in step 4
   -- not against any DID taken from the response.

### App-key parse checks {#app-key-parsing}

At step 4 above, an [=application=] MUST enforce **all** of the following, and
MUST reject the response if any fails:

1. the `type` array includes the application's own `credentialType`;
2. the `type` array includes the `AppKeyCredential` marker;
3. `issuer` is present, `credentialSubject.id` is present, and they are equal;
4. `credentialSubject.origin` equals this application's own origin, compared as
   an **exact string**;
5. `credentialSubject.seed` is a non-empty string that decodes as base64url
   without padding to exactly 32 bytes;
6. the [=connecting DID=] derived from those bytes per [[[#key-derivation]]]
   equals `credentialSubject.id`.

The origin value used in check 4 MUST be the **same value** the application sent
as the request's `domain` ([[[#challenge-and-domain]]]): its own live browser
origin. An application MUST NOT check the credential against one origin while
having requested under another, and MUST NOT take either value from
configuration that can drift from the origin it is actually running on.

These checks duplicate checks the wallet already made. That is the point: they
are defence in depth, and an application that omits them is trusting the wallet
about an origin binding and an identity binding that it is fully able to check
itself.

### Grant validation {#grant-validation}

At step 5 of [[[#response-verification]]], an [=application=] MUST check that:

1. every [=grant=]'s `controller` equals the [=connecting DID=] parsed from the
   credential;
2. every grant carries an `expires` value that is a string and is not already
   past;
3. every grant's `invocationTarget` is a **single non-empty string**. A grant
   whose `invocationTarget` is absent, is not a string, or is an array of
   targets MUST be rejected;
4. every grant's target parses under the [[WAS]] URL template
   ([[[#url-template]]]) and is **collection-scoped** -- that is, it names a
   [=collection=] or a [=resource=] within one. A Space-scoped target, or a
   target naming a reserved sub-endpoint rather than a collection, MUST be
   rejected;
5. all grants resolve to a **single** storage host and a **single** [=Space=];
   a grant set spanning two hosts or two Spaces MUST be rejected;
6. every collection the application requested as its **own** (that is, via
   `#collection` or `#public-collection`) is covered by a grant whose
   `allowedAction` includes every action the application requires of it, per
   [[[#required-actions]]];
7. at least one grant is present, whenever the application requested at least
   one of its own collections.

Delegation-chain validity is **not** the application's to check: it is enforced
by the storage server at invocation time, per [[WAS]]. What the application
checks is structure.

<div class="note">
Check 4 is stricter than this profile requires a wallet to be: a wallet may in
principle satisfy a `#space` descriptor ([[[#descriptor-space]]]) and emit a
Space-scoped grant. An application rejects it anyway, because its routing is
per-collection and it has nothing to do with a target it cannot route. An
application that wants Space-scoped access is outside the scope of this
version.
</div>

#### Required actions and the class ceiling {#required-actions}

The actions an [=application=] requires of a collection MUST be within the
action ceiling of the descriptor class it used to request that collection
([[[#action-ceilings]]]):

| Requested via | Actions the application may require |
|---------------|--------------------------------------|
| `#collection` | any subset of `GET`, `HEAD`, `POST`, `PUT`, `DELETE` |
| `#public-collection` | any subset of `GET`, `HEAD`, `POST` |

An application MUST NOT require an action above the ceiling of the class it
requested, and MUST NOT fail a connection because a grant lacks such an action.

<div class="note">
Without this rule the two conformance classes contradict each other. An
application that asks for a public collection with a default read/write action
set, and then requires every action it asked for, rejects the `GET, HEAD, POST`
grant that a **correct** wallet is obliged to return -- the public-collection
ceiling forbids `PUT` and `DELETE` ([[[#descriptor-public-collection]]]), so no
conformant wallet can ever satisfy the requirement. An application requesting a
public collection is expected to require at most add-only access, which is what
a public collection offers.
</div>

### Declined shares {#declined-shares}

A `#shared-collection` [=grant=] MUST NOT be login-gating. If the user declines
a share, or the wallet resolves it unsatisfiable, the connection MUST still
succeed: the application simply does not open a reader for that collection.

An application MUST NOT include shared collections in the coverage check of
[[[#grant-validation]]] step 6, and MUST tolerate a response in which it
requested only shared collections and received no grants at all.

A share is the user granting access to their *own* data. Declining it is a
legitimate answer to that question, and it says nothing about whether the user
wants to use the application.

### The wallet-unsupported outcome {#wallet-unsupported}

If the presentation verifies but carries no [=app-key credential=], an
[=application=] MUST treat this as a **distinct fail-closed outcome**: the
wallet could not satisfy the `AppConnectQuery`.

This outcome MUST be distinguished from:

* **user cancellation**, which the transport reports as no response at all; and
* **first run**, which is a wallet that *did* return a credential and set
  `appConnect.firstRun` to `true`.

Conflating "the wallet cannot do App Connect" with "this is a new user" is the
failure this rule exists to prevent: the application would treat a wallet that
answered nothing as a brand-new user, and would then be operating with no
identity and no grants while believing itself connected.

### Example response {#response-example}

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    {
      "@protected": true,
      "zcap": { "@id": "https://w3id.org/byoe#zcap", "@container": "@set" }
    },
    {
      "@protected": true,
      "appConnect": {
        "@id": "https://w3id.org/byoe#appConnect",
        "@type": "@json"
      }
    },
    "https://w3id.org/security/suites/ed25519-2020/v1"
  ],
  "type": ["VerifiablePresentation"],
  "holder": "did:web:wallet.example",
  "verifiableCredential": [
    {
      "type": ["VerifiableCredential", "AppKeyCredential", "ExampleNotesUser"],
      "issuer": "did:key:z6MkExampleNotesAppKeySubjectDidPlaceholder",
      "credentialSubject": {
        "id": "did:key:z6MkExampleNotesAppKeySubjectDidPlaceholder",
        "seed": "b0dRQXBwS2V5U2VlZFBsYWNlaG9sZGVyVmFsdWUzMkI",
        "origin": "https://app.example"
      },
      "...": "...",
      "proof": { "...": "..." }
    }
  ],
  "zcap": [
    {
      "@context": "https://w3id.org/zcap/v1",
      "id": "urn:zcap:delegated:z19examplenotescollectiongrant",
      "parentCapability": "urn:zcap:root:https%3A%2F%2Fwallet-storage.example%2Fspace%2F81246131-69a4-45ab-9bff-9c946b59cf2e",
      "invocationTarget": "https://wallet-storage.example/space/81246131-69a4-45ab-9bff-9c946b59cf2e/notes",
      "controller": "did:key:z6MkExampleNotesAppKeySubjectDidPlaceholder",
      "allowedAction": ["GET", "HEAD", "POST", "PUT", "DELETE"],
      "expires": "2026-08-13T14:22:11Z",
      "proof": { "...": "..." }
    },
    {
      "@context": "https://w3id.org/zcap/v1",
      "id": "urn:zcap:delegated:z19examplenotespublishedgrant",
      "parentCapability": "urn:zcap:root:https%3A%2F%2Fwallet-storage.example%2Fspace%2F81246131-69a4-45ab-9bff-9c946b59cf2e",
      "invocationTarget": "https://wallet-storage.example/space/81246131-69a4-45ab-9bff-9c946b59cf2e/published-notes",
      "controller": "did:key:z6MkExampleNotesAppKeySubjectDidPlaceholder",
      "allowedAction": ["GET", "HEAD", "POST"],
      "expires": "2026-08-13T14:22:11Z",
      "proof": { "...": "..." }
    }
  ],
  "appConnect": { "firstRun": true },
  "proof": {
    "type": "Ed25519Signature2020",
    "proofPurpose": "authentication",
    "challenge": "0b1c9a1e-6a5f-4a2e-9c8a-2f7f4d3b6c11",
    "domain": "https://app.example",
    "...": "..."
  }
}
```

<div class="note">
In this example the user approved the two application-owned collections and
declined the `contacts` share from [[[#request-example]]], so no third grant
appears. Per [[[#declined-shares]]] the connection still succeeds.
</div>

## Grant Processing {#grants}

### Overview {#grant-overview}

Grant processing has two phases, and a [=wallet=] MUST keep them separate:

1. **Resolution** is pure. Each request entry is resolved to a target class and
   a capped action set ([[[#descriptors]]], [[[#action-ceilings]]]) with no
   provisioning, no delegation, and no side effects. Resolution is what the
   consent surface renders.
2. **Delegation** runs only after the user approves. It provisions what needs
   provisioning ([[[#provisioning]]]) and delegates each satisfiable grant.

A wallet MUST NOT show the user one resolution and then delegate a different
one.

Every delegation MUST be rooted at the user's Space root capability, and MUST
target a URL inside that Space. Targets outside the Space are unsatisfiable by
construction ([[[#string-targets]]]), so a delegation rooted elsewhere cannot
arise.

### Resumability {#resumability}

The delegation phase is not atomic: it may provision several collections, admit
the grantee to several rosters, and mint several delegations, and it can fail
partway through any of that.

A [=wallet=] MUST ensure that such a failure leaves the exchange **repairable by
re-running it**. Concretely:

* provisioning MUST be idempotent ([[[#provisioning]]]);
* delegation MUST be idempotent in effect: re-running an approved exchange MUST
  converge on the same access, and MUST NOT accumulate duplicate or broader
  standing grants;
* minting or storing the [=app-key credential=] MUST happen before delegation,
  so that a failure after it leaves a returning connection rather than a second
  identity ([[[#app-key-minting]]]).

A wallet MUST NOT leave the exchange in a state the [=application=] or the user
must manually undo, and MUST NOT require a partial failure to be repaired by any
step other than re-running the exchange or the wallet's own maintenance.

<div class="note">
Re-running is the only repair mechanism this profile defines, which is why every
side effect it specifies is expressible as a converging operation rather than an
incrementing one. The share case is the sharpest instance: its two axes cannot
always be established in one atomic step, so
[[[#descriptor-shared-collection]]] requires the missing half to be completed on
a later run rather than requiring the pair to be atomic.
</div>

### Grant recording {#grant-recording}

A [=wallet=] MUST retain, for each grant it delegates, a record sufficient to
**revoke that grant later** and to present it to the user as standing access.

A wallet MUST NOT deliver a response containing a grant it was unable to record.
If recording fails, the wallet MUST fail the exchange rather than return the
grant.

<div class="note">
Failing closed here is the conservative direction: an exchange that fails is
re-runnable ([[[#resumability]]]), whereas a delegation the wallet cannot revoke
is a standing authorization over the user's storage with no mechanism to
withdraw it -- and the user has no visibility into it either, since a wallet can
only list what it recorded. Grant lifetimes ([[[#ttl]]]) bound the damage but
are explicitly not the removal mechanism, least of all for shares.
</div>

### The action vocabulary {#action-vocabulary}

The action vocabulary is **closed**:

```
GET, HEAD, POST, PUT, DELETE
```

A [=wallet=] MUST normalize each requested action by trimming it, upper-casing
it, and intersecting it with this set. A token outside the set -- an unknown
verb, a non-string, or an action a future server version might support -- MUST
be **dropped**. It MUST NOT be passed through into an `allowedAction` array the
user's root key signs.

`HEAD` is a tolerated read alias, not an action of its own. [[WAS]] authorizes
a `HEAD` request as a `GET`; this profile includes `HEAD` in every read grant
and caps it exactly as `GET` is capped, so it never appears in a ceiling that
does not already permit reads.

<div class="note">
Dropping unknown action tokens is the same fail-closed treatment an
unrecognized descriptor type gets ([[[#unknown-descriptor-type]]]), applied one
level down.
</div>

### Default actions {#default-actions}

A request entry with **no** `allowedAction` member MUST be treated as
requesting `["GET", "HEAD"]`.

A wallet MUST NOT treat an absent `allowedAction` as inherit-all. In the
capability model an empty `allowedAction` array means *every* action, so
defaulting to the empty array would turn "did not say" into "asked for
everything".

### Action ceilings {#action-ceilings}

Attenuation in this profile is a **table, not a switch**. Every satisfiable
target resolves into exactly one target class, and each class has a ceiling
that the normalized requested actions are intersected against. Nothing above a
class's ceiling is ever granted, no matter what the request asks for.

This table is normative.

| Target class | Ceiling | Rationale |
|--------------|---------|-----------|
| space | `GET`, `HEAD` | A Space-wide write would authorize rewriting the Space Description, i.e. controller takeover. |
| protected collection | `GET`, `HEAD` | A requesting party may read the user's own data on consent; it may never rewrite or delete it. |
| share | `GET`, `HEAD` | A share hands over decryption as well as fetch; it is never a write grant. |
| public collection | `GET`, `HEAD`, `POST` | Add-only: a write to a plaintext world-readable target is publication under the user's identity and irreversible in practice. |
| collection | `GET`, `HEAD`, `POST`, `PUT`, `DELETE` | The application's own data, bounded by consent and by the shorter write lifetime. |

The resulting `allowedAction` array MUST be ordered by the ceiling, not by the
order the request asked in, so that equivalent requests yield byte-identical
grants.

If the intersection is **empty** -- the request asked only for actions the
class forbids, or only for tokens outside the vocabulary -- the entry MUST
resolve **unsatisfiable**. A wallet MUST NOT delegate a capability with an
empty `allowedAction` array, which would mean unrestricted.

### Delegation target {#delegation-target}

A [=wallet=] MUST set every delegated grant's `controller` to the subject DID
of the [=app-key credential=] it matched or minted in this same exchange.

The request never names a controller ([[[#capability-query]]]), and a wallet
MUST NOT accept one from the request. This is the rule that makes the exchange
single-round: on first run the identity being delegated to does not exist until
the wallet creates it, so only the wallet can name it.

### Per-user grantee DIDs {#per-user-grantee}

A grantee DID SHOULD NOT be shared across users. An [=application=] SHOULD hold
a distinct [=connecting DID=] per user, and a [=wallet=] SHOULD NOT delegate to
a controller it has reason to believe is shared.

App Connect satisfies this by construction: the [=seed=] is minted per (user,
application, origin), so the DID derived from it is per-user by definition. The
requirement is stated as a SHOULD because the standalone capability channel can
violate it undetectably -- a requester there names its own `controller`, and
nothing in the exchange reveals whether the same one was named for another
user.

### Recipient-key derivation {#recipient-derivation}

Where an App Connect grant makes the grantee an [=epoch-roster recipient=] --
that is, a `#shared-collection` grant ([[[#descriptor-shared-collection]]]) or
the provisioning of an encrypted application collection
([[[#encrypted-by-default]]]) -- the recipient's key-agreement key MUST be
**derived from the grantee's controller DID**.

A [=wallet=] MUST NOT accept a recipient key supplied in the request, in any
form. An explicit recipient key would let a request pair one entity's
controller DID with another entity's decryption key; deriving makes that
substitution impossible by construction and keeps both axes of a share pointing
at the same entity.

If the controller DID has no derivable key-agreement key, the entry MUST
resolve **unsatisfiable**. A wallet MUST reach that verdict by attempting the
real derivation, not by inspecting the DID's shape: a well-formed-looking but
malformed DID would otherwise pass a shape check and fail only when the
derivation is actually performed.

For a `#shared-collection` entry, a wallet MUST perform that derivation during
**resolution**, while the consent surface is being built, whenever the
controller is known at that point -- so a share the wallet cannot honor is shown
as unsatisfiable rather than failing mid-delegation.

<div class="note">
On a first-run exchange the [=connecting DID=] does not exist during resolution
(the credential is not minted until after consent), so an **absent** controller
at resolution time is not yet a failure. The wallet re-checks with the real
subject DID before delegating, and the same requirement applies then.
</div>

For every other entry class, the derivation may not be reachable before consent,
because the collection may not exist until provisioning runs. A wallet is
therefore not required to reach the unsatisfiable verdict before delegation
begins; it is instead required to leave any resulting partial failure
**repairable by re-running the exchange**, per [[[#resumability]]]. A wallet
MUST NOT leave the application holding an inconsistent set of grants that only
manual intervention could correct.

The byte-level derivation, the resulting recipient identifier, and the roster
format are defined in [[WAS-EC]]. This profile states only the invariant and
the refusal.

### Provisioning {#provisioning}

When a satisfiable grant names a collection that does not yet exist, the
[=wallet=] MUST provision it before delegating.

Provisioning MUST be **idempotent**. Re-running an App Connect exchange, in
whole or in part, MUST NOT change the outcome. In particular:

* a wallet MUST NOT clobber an existing collection's encryption descriptor;
* a wallet MUST NOT re-initialize a key epoch for a collection that already has
  one;
* if the grantee is already a recipient of the collection's current epoch, the
  wallet MUST do nothing;
* if the collection already has epochs but the grantee is not among the current
  epoch's recipients -- the state left by a previous revocation -- the wallet
  MUST add the grantee as a recipient rather than starting a new roster.

A wallet MUST NOT provision over a protected collection
([[[#protected-collections]]]).

### Encrypted by default {#encrypted-by-default}

A private collection provisioned for an [=application=] under
`#collection` MUST be encrypted from creation. There is no unencrypted
intermediate state, and no later migration step in which existing contents are
converted.

Two invariants hold for such a collection:

* **The collection owner is always a recipient.** The user, as the owner of the
  Space, MUST be an [=epoch-roster recipient=] of every encrypted collection in
  their own Space. Any departure from this MUST be an explicit consent surface,
  never a silent default.
* **The grantee is a recipient by derivation.** The application is added as a
  recipient using the key derived from its controller DID
  ([[[#recipient-derivation]]]); the wallet never sees the application's
  [=seed=], and the application never needs the user's own key.

A collection provisioned under `#public-collection` MUST be plaintext and MUST
NOT be given an encryption descriptor ([[[#descriptor-public-collection]]]).

### Grant lifetimes {#ttl}

Every delegated grant MUST carry an `expires` value.

A [=wallet=] SHOULD differentiate lifetimes by grant kind:

* a **read-only** grant on an application collection SHOULD be the longest of
  the ordinary lifetimes;
* a **write-bearing** grant -- one whose capped actions include anything beyond
  `GET` and `HEAD` -- SHOULD be shorter, since a leaked write grant can mutate
  data;
* a **share** grant SHOULD be long, for the reason given below.

<div class="note">
The values the shipped implementations use are informative: 30 days for
read-only grants, 7 days for write-bearing grants, and 365 days for shares.
</div>

**A share grant's lifetime MUST NOT be used as the removal mechanism for the
share.** This is normative and is the reason a share's lifetime is long rather
than short. The two axes of a share come apart at expiry: the delegated
capability dies while epoch-roster recipiency does not, leaving a party who can
still decrypt what it has, is still listed as a reader, and can no longer
fetch. Revocation of a shared collection is an explicit act -- re-epoch plus
revoke, together ([[[#descriptor-shared-collection]]]) -- not an expiry.

### Consent {#consent}

A [=wallet=] MUST obtain the user's consent before minting or returning an
[=app-key credential=] and before delegating any grant.

This profile states a normative minimum for the consent surface:

1. The surface MUST present **exactly what resolution produced**. For each
   requested capability it MUST show the resolved target and the actions that
   would actually be granted after capping ([[[#action-ceilings]]]) -- not the
   actions that were requested.
2. An entry that resolved **unsatisfiable** MUST be shown as such. It MUST NOT
   be omitted, and MUST NOT be shown as if it would be granted.
3. A `#shared-collection` row MUST state that the grant is **read and
   decrypt**, not merely fetch; that it covers what is **already stored** in
   the collection, not only what is written afterwards; and that removing
   access later stops future reads but cannot retract what has already been
   read.
4. A `#public-collection` row MUST state that anyone on the web will be able to
   read the collection.
5. A whole-Space row and a write-bearing row MUST each be distinguishable from
   an ordinary read row.
6. The App Connect consent surface supersedes per-grant `reason` strings; per
   [[[#capability-query]]] request entries carry none, and a wallet MUST NOT
   display one.

Where the surface shows requesting-party-supplied free text (the `app.name`),
it MUST be rendered so that the text cannot be mistaken for wallet-authored
copy or for an identifier the wallet verified.

## Resource Log Profile (Reserved) {#resource-log-profile}

This section is **reserved**. It defines no normative requirements, and an
implementation cannot conform to it.

It is expected to define a hash-linked log profile, derived from the
`did:webvh` log format, for key resources co-managed between a wallet's clients
and the storage server -- principally collection encryption descriptors and the
key rosters associated with them. The intended scope is: the entry format;
entry hashing; chain verification; the external-authorization rule (entries are
authorized against the wallet account's controller document, not against keys
the log itself declares); client-side head pinning; and a terminal handover
entry supporting log compaction and format migration.

<div class="issue">
This section is a placeholder recording intended scope. Nothing here is
implemented, and nothing here should be relied upon. The section will be
replaced with normative text, or removed, once the construction has running
code on both sides.
</div>

## Security Considerations {#security}

<div class="note">
This section is non-normative in its rationale, but the rules it refers to are
normative where stated above.
</div>

### Origin binding and what the browser attests {#security-origin}

The [=app-key credential=]'s `origin` claim is what prevents one origin from
recovering another's application identity. Its value comes from the [=CHAPI=]
mediator, which is part of the browser-mediated transport and reports the
origin that actually opened the request -- not from the request body, which the
requesting party controls entirely.

Binding is enforced on both sides ([[[#app-key-binding]]]). The wallet-side
check is what makes phishing fail: an attacker who convinces a user to approve
a connect at `https://app.exarnple` cannot be handed the credential bound to
`https://app.example`, and the credential minted for the attacker's origin
derives a different DID and therefore opens none of the victim's data. The
application-side check is what makes the application independent of the
wallet's diligence.

The `domain` check ([[[#challenge-and-domain]]]) closes the adjacent hole: a
request may not pin a `domain` other than the channel it arrived on, so a
verifier cannot relay another party's challenge through this user's wallet.

**The two checks are deliberately not the same strength, and the difference
matters.** The wallet's `domain` check compares **hosts**, so it is insensitive
to scheme, port, and any other origin component: it is a
relay-and-replay guard, sized to the question "did this request arrive on the
channel it claims". The credential's `origin` binding is an **exact string**
comparison of full origins, on both sides ([[[#app-key-binding]]],
[[[#app-key-parsing]]]), and it is the authoritative anti-phishing check -- the
one that decides whether a stored identity is handed over.

An implementation MUST NOT read the host-level `domain` check as sufficient for
that decision. Two origins sharing a host but differing in scheme are the same
`domain` and different `origin`s, and it is the origin that governs which
credential is returned.

### Seed confidentiality {#security-seed}

The [=seed=] is the application's client secret: everything the application can
sign and everything it can decrypt derives from it.

In transit it exists only inside the [=CHAPI=] response, which is a
browser-mediated channel between the wallet and the requesting origin. It is
never sent to the storage server, never appears in a capability, and never
appears in a URL.

At rest it lives in two places: in the user's wallet, as an ordinary stored
credential subject to whatever protection the wallet applies to the user's
credentials; and application-side, wherever the application persists it. An
application SHOULD treat the seed as it would any long-lived client secret, and
SHOULD scope its storage to its own origin.

Nothing downstream of the wallet's match consumes the seed: the wallet reads it
only to re-derive the subject DID for the binding check, so the seed never
reaches the grant path.

### The planted credential {#security-planted-credential}

An `AppKeyCredential` marker asserts nothing that an attacker cannot also
assert, and neither does self-issuance. Two distinct mechanisms carry the
weight, and it matters which one carries which.

**Seed-binds-subject ([[[#seed-binds-subject]]]) establishes internal
consistency.** The credential carries the seed, the seed derives a DID, and
that DID must be the credential's own subject. This rejects every credential in
which the subject or the seed has been substituted: a genuine credential whose
seed was swapped for the attacker's, or a genuine seed relabelled with the
attacker's subject DID. Both are the natural first attempts, and both fail.

**It does not establish provenance, and cannot.** An attacker who generates
their own seed, derives its DID correctly, and self-issues a credential naming
the victim application's `credentialType` and the victim's [=origin=] produces a
credential that binds. There is nothing wrong with it: it is a genuine app-key
credential for the attacker's identity, and no amount of inspection
distinguishes it from the user's own, because the two differ only in which seed
was rolled.

The attack that follows is worth stating in full, because it is what the
provenance rule exists for. The attacker gets the user to import such a
credential -- from a link, a QR code, a file, or a restored backup -- and gives
it an `issuanceDate` in the future. The wallet stores it (every check passes).
On the next connect the wallet's match predicate ([[[#app-key-matching]]])
accepts it: correct marker, correct type, self-issued, correct origin, binds.
The latest-first ranking picks it over the user's real credential on the strength
of its future date. The wallet then delegates the user's storage to the
attacker's [=connecting DID=], and hands the attacker's seed back to the
application as if it were the user's own.

**The refusal that closes it is a channel rule, not a content rule.** An
app-key credential has no legitimate reason to arrive from outside: it is
wallet-minted, in an [=App Connect exchange=], and never imported. A wallet that
refuses every marker-carrying credential arriving from outside
([[[#store-time-refusal]]]) is never in the position of having to tell a planted
credential from a real one, because the only credentials in its store are ones
it minted itself.

The binding refusal remains in place alongside it, covering credentials stored
under an earlier rule and any ingest path a wallet later grows. The two are
independent; neither substitutes for the other.

Ranking over instants rather than raw strings ([[[#app-key-matching]]]) is part
of the same picture: the ranking decides which DID the wallet delegates to, so a
comparison that can be manipulated by the *spelling* of a date rather than its
value would reopen a narrower version of the same path.

### Fail-closed processing as the extensibility rule {#security-fail-closed}

Two rules in this profile refuse rather than degrade: an unrecognized descriptor
type ([[[#unknown-descriptor-type]]]) and an unrecognized query type
([[[#wallet-unsupported]]]). A third narrows rather than refuses: an action
token outside the closed vocabulary is dropped ([[[#action-vocabulary]]]) --
which can never widen a grant, and becomes a visible refusal at the point where
it would matter, since an entry left with no actions after capping is
unsatisfiable rather than unrestricted ([[[#action-ceilings]]]).

Together they are what makes the vocabulary extensible without a version
negotiation. A new descriptor type can be deployed knowing that every wallet
which does not implement it will say so visibly, and that no wallet will
approximate it. The concrete hazard being avoided is an application that
believes it has a public collection, or a decryptable share, and does not.

### Replay protection {#security-replay}

Freshness rests on the challenge and the domain. The application generates a
fresh challenge per request and checks that the returned proof echoes it; the
wallet checks that the request's domain matches the channel it arrived on and
echoes it into the proof; the application checks the echoed domain equals its
own origin.

The grants are inside the signed presentation, which is what keeps them within
that freshness envelope. Embedding after signing would leave a response whose
authenticated portion said nothing about which capabilities it carried.

A conformant application requires every presentation-level proof to be an
authentication proof ([[[#response-verification]]]), so that the proof carrying
the challenge is provably one that was signature-checked.

## Privacy Considerations {#privacy}

<div class="note">
This section is non-normative.
</div>

### Per-user application identity {#privacy-per-user}

Because the [=connecting DID=] derives from a [=seed=] minted per user, per
application, and per origin, an application holds a different DID for every
user. This removes a correlation channel that a shared grantee DID would
create: with one DID across all users, every capability an application holds
would name the same controller, so a storage provider seeing two Spaces
delegating to that DID would learn the two accounts use the same application
and could link activity across them.

It also removes a single point of failure: compromise of one user's application
identity does not touch another's.

The corresponding expectation is stated normatively as a SHOULD in
[[[#per-user-grantee]]].

### The origin claim {#privacy-origin}

The `credentialSubject.origin` claim states which application the credential
belongs to, and it is visible to anyone who sees the credential. Inside the
wallet this is the point -- the user should be able to see which applications
hold keys in their wallet, and the wallet needs the value to match on. But an
app-key credential is not a credential to present anywhere else: it is not
selectively disclosable, it carries a client secret, and presenting it
elsewhere would disclose both the seed and the application relationship.

Applications and wallets SHOULD treat the app-key credential as
non-presentable outside the App Connect exchange that produced it.

### What the storage server learns {#privacy-server}

The storage server is not a party to the exchange ([[[#conformance-classes]]]).
It does not see the request, the response, the seed, the consent, the
application's name, or its origin.

What it does see is the traffic that follows: capability invocations naming a
controller DID, targeting collections in a Space. From these it can infer that
*some* delegated party is reading and writing particular collections, and it
can observe collection names, request timing, and volumes. It cannot tell an
App Connect grant from any other delegated capability, and the controller DID
it sees is per-user, so it does not identify the application.

Collection **names** are the notable exception to that last point: an
application requests collections by name, and the names it chooses are visible
to the storage provider. An application SHOULD choose collection names that do
not disclose more than necessary.
