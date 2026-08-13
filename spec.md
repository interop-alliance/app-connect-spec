<div class="remove">

# App Connect for Wallet Attached Storage v0.1

**Abstract:** App Connect defines a CHAPI/VCALM profile by which a web
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
[=CHAPI=] `get` -- the in-browser transport; non-browser applications use
another [[VCALM]]-capable protocol ([[[#request-transport]]]) -- carrying a
verifiable presentation request [[VCALM]] with one
[=AppConnectQuery=] in it. The wallet answers, in the same round, with one
signed Verifiable Presentation carrying:

* the [=app-key credential=] -- a self-issued Verifiable Credential holding a
  32-byte [=seed=] from which the application derives its own key material,
  matched from wallet storage for a returning user or minted on the spot for a
  new one; and
* the [=grants=] -- capabilities [[ZCAP]] delegated to the subject DID of that
  very credential, targeting collections in the user's Space.

There is no second popup, no store step, and no prior registration. The
application never names a controller DID in its request, because a [=public
client=] on first run does not have one yet: the wallet mints the identity and
delegates to it in the same breath. That is what collapses the flow to a single round.

### Scope {#scope}

This document defines:

* the [=AppConnectQuery=] request query type and its processing rules
  ([[[#request]]]);
* the [=invocation target descriptor=] vocabulary and the per-class allowed
  actions that bound what a delegation may carry ([[[#descriptors]]]);
* the [=app-key credential=]: its `AppKeyCredential` type, claim vocabulary, seed
  encoding, and the seed-to-subject binding rule ([[[#app-key-credential]]]);
* the response presentation's `zcap` and `appConnect` members and the
  application-side verification order ([[[#response]]]);
* the grant resolution, capping, and provisioning rules
  ([[[#grants]]]);
* the [=resource log=] profile: the hash-linked log format for key resources
  co-managed between a wallet's clients and the storage server
  ([[[#resource-log-profile]]]).

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
| [[WAS-EC]] | Companion profile (draft). Defines the encrypted-collection construction: key epochs, roster recipients, envelope format, and the derivation of a recipient key from a controller DID. This profile defers to it for the recipient-key derivation ([[[#recipient-derivation]]]), the key-agreement half of the application's key material ([[[#key-derivation]]]), the epoch rotation behind forward-only re-grants ([[[#descriptor-shared-wallet-collection]]]), and the definition of [=epoch-roster recipient=] itself. |
| [[VCALM]] | Defines verifiable presentation requests and their query types: the request body that carries an [=AppConnectQuery=], the `DIDAuthentication` and `QueryByExample` query types this profile interacts with, and `AuthorizationCapabilityQuery`, the standalone capability-request query type whose entry shape [=AppConnectQuery=] reuses. Some wallets also honor `ZcapQuery` as a legacy spelling of `AuthorizationCapabilityQuery`, on the standalone capability-query channel only; this profile refers to the type by its canonical name throughout. |
| [[CHAPI]] | The transport for in-browser applications, and the one this profile's normative prose is written against: it supplies the browser-attested requesting origin the [=app-key credential=] is bound to. Non-browser applications carry the same [[VCALM]] request over other transports ([[[#request-transport]]]). |
| [[DID-WEBVH]] | The log format the [=resource log=] profile ([[[#resource-log-profile]]]) is extracted from, and one method a Space controller's [=controller document=] may be verified under. Nothing outside that profile depends on it. |

### Conformance Classes {#conformance-classes}

This specification defines two conformance classes.

Both are defined over one [=App Connect exchange=].

A conformant **[=wallet=]** MUST:

* recognize the [=AppConnectQuery=] type ([[[#appconnectquery]]]) and process it
  per [[[#request]]];
* match or mint the [=app-key credential=] per
  [[[#app-key-credential]]];
* refuse to store an `AppKeyCredential`-typed credential arriving from outside an
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
capability. This is a design goal; it keeps the exchange a
matter between the user's wallet and the application, and it keeps the storage
provider substitutable. The one exception is the [=resource log=] profile,
which requires the backend holding the log to support [[WAS]]'s optional
conditional writes ([[[#log-append]]]).
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
from the origin `https://app.example` under the application URL
`https://app.example/notes/`. The
user's wallet Space is `81246131-69a4-45ab-9bff-9c946b59cf2e` on the host
`wallet-storage.example`, so its Space URL is
`https://wallet-storage.example/space/81246131-69a4-45ab-9bff-9c946b59cf2e`.
DID and key values in examples are illustrative and are not real derivations.

**Proofs are elided.** Example credentials and presentations show
`"proof": { ... }` rather than a real signature, except where the proof's
members are themselves under discussion.

**"Byte-significant" means what it says.** Several strings in this document --
type names, term URLs, the key-derivation label, the collection-name grammar
-- are inputs to a derivation or to a canonicalized signature. Changing one
does not produce a variant profile; it produces an incompatible one, and for
the derivation inputs it orphans every identity already derived under the old
value.
</div>

## Terminology {#terminology}

<dl class="termlist definitions" data-sort="ascending">
  <dt><dfn data-lt="action|actions|allowedAction">action (allowedAction)</dfn></dt>
  <dd>See <a href="https://w3c-ccg.github.io/wallet-attached-storage-spec/#authorization-actions-and-the-root-capability">Authorization
    Actions and the Root Capability</a> in [[WAS]]. The kind of operation a
    request performs on a [=target=],
    named by a [=capability=] so it can be authorized. This profile restricts
    [=grants=] to a closed vocabulary of uppercase HTTP
    method names; see [[[#action-vocabulary]]].</dd>

  <dt><dfn data-lt="applications|requesting party">application</dfn></dt>
  <dd>An application -- typically a web application, though non-browser
    applications participate over other transports ([[[#request-transport]]]) --
    that connects to a user's [=wallet=] through an App
    Connect exchange to receive delegated access to the user's storage,
    along with the per-user [=app-key credential=] that the delegation
    targets. One of the two conformance classes; see
    [[[#conformance-classes]]]. An application is identified to the wallet by
    the identifier its transport attests -- for an in-browser application, its
    [=origin=] -- and by the <code>app</code> block of its
    [=AppConnectQuery=]. Every application this document profiles is a
    [=public client=].</dd>

  <dt><dfn data-lt="AppConnectQuery|app connect query">App Connect query
    (AppConnectQuery)</dfn></dt>
  <dd>A Verifiable Presentation Request query type this profile defines,
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
    application's [=origin=] and application URL
    (<code>appUrl</code>). It is held in the user's wallet, and is what
    makes an application's identity portable across the user's clients even
    though the application, as a [=public client=], holds no durable secret
    of its own. See
    [[[#app-key-credential]]].</dd>

  <dt><dfn data-lt="capability|capabilities|zcap|zCap">capability (zCap)</dfn></dt>
  <dd>An authorization capability authorizing a set of [=actions=] on a [=target=].
    See the <a
    href="https://interop-alliance.github.io/zcap-developer-guide/">zCap
    Developer Guide</a> and [[ZCAP]] for the model, and [[WAS]] for the storage
    profile of it that this document produces delegations under.</dd>

  <dt><dfn data-lt="CHAPI">Credential Handler API (CHAPI)</dfn></dt>
  <dd>The browser-mediated transport over which an App Connect exchange is
    carried when the [=application=] runs in a browser [[CHAPI]]. Its relevant
    property here is that it attests the requesting [=origin=] to the wallet:
    the wallet learns which origin opened the request from the mediator, not
    from the request body. Non-browser applications carry the same exchange
    over other transports; see [[[#request-transport]]].</dd>

  <dt><dfn data-lt="collections|Collection">collection</dfn></dt>
  <dd>See <a href="https://w3c-ccg.github.io/wallet-attached-storage-spec/#collections">Collections</a>
    in [[WAS]]. A namespace and configuration container for Resources
    within a [=Space=]. Every capability an App Connect response carries is
    scoped to a collection or to the Space itself.</dd>

  <dt><dfn data-lt="confidential clients">confidential client</dfn></dt>
  <dd>An [=application=] able to hold credentials of its own and to
    authenticate itself by mechanisms attested outside the request body -- a
    server-backed application presenting a TLS client certificate, or an
    application whose DID and keys are published in a trusted registry -- in
    the sense of the confidential client class of [[RFC6749]]. This document
    does not profile confidential clients; see [[[#request-transport]]].
    Contrast [=public client=].</dd>

  <dt><dfn data-lt="connecting DID|subject DID|controller DID">connecting DID
    (subject DID)</dfn></dt>
  <dd>The <code>did:key</code> DID derived from an [=app-key credential=]'s
    [=seed=] ([[[#key-derivation]]]), which is simultaneously that credential's
    <code>issuer</code>, its <code>credentialSubject.id</code>, and the
    <code>controller</code> of every [=grant=] the wallet delegates in the same
    exchange. It is per (application, user), not per browser or per
    machine.</dd>

  <dt><dfn data-lt="enrolled clients">enrolled client</dfn></dt>
  <dd>A client of the [=wallet=] account whose verification key is listed in
    the account's controller document (the [=Space=] controller's DID
    document), making it able to act with the account's own authority -- in
    this profile, to authorize appends to a [=resource log=]
    ([[[#log-authorization]]]). Enrolled clients are typically the user's
    other wallet installations (a browser wallet and a native wallet on the
    same account, say), and they are peers: each carries the account's full
    authority, unlike an [=application=], which only ever holds capabilities
    delegated to it. How a wallet enrolls a client, revokes one, or recovers
    from a lost secret is wallet-internal and out of scope
    ([[[#scope]]]).</dd>

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
    [[[#allowed-actions]]].</dd>

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

  <dt><dfn data-lt="public clients">public client</dfn></dt>
  <dd>An [=application=] that cannot hold a durable secret or credential of
    its own, in the sense of the public client class of [[RFC6749]]: an
    in-browser application, a native mobile application, a command-line tool.
    Its only trustworthy identification toward the [=wallet=] is what the
    transport or platform attests -- for the in-browser case, the [=origin=]
    -- and its identity is made stable across the user's clients by wallet
    custody of the [=seed=], not by anything the client itself holds. Every
    application this document's normative prose addresses is a public client.
    Contrast [=confidential client=].</dd>

  <dt><dfn data-lt="Resource|resources">resource</dfn></dt>
  <dd>See <a href="https://w3c-ccg.github.io/wallet-attached-storage-spec/#resources-and-blobs">Resources
    and Blobs</a> in [[WAS]]. The addressable unit of storage inside a
    [=collection=].</dd>

  <dt><dfn data-lt="seeds">seed</dfn></dt>
  <dd>32 bytes of randomness minted by the [=wallet=], carried in the
    [=app-key credential=]'s <code>credentialSubject.seed</code> claim, from
    which the [=application=] deterministically derives its signing key, its
    [=connecting DID=], and its key-agreement key. It plays the role a client
    secret plays for a [=confidential client=], except that the [=wallet=]
    custodies it; nothing else in the exchange substitutes for it.</dd>

  <dt><dfn data-lt="Spaces|Space">space</dfn></dt>
  <dd>See <a href="https://w3c-ccg.github.io/wallet-attached-storage-spec/#spaces">Spaces</a>
    in [[WAS]]. The user-owned top-level container that holds
    [=collections=]. Every App Connect [=grant=] resolves to a target inside
    exactly one Space.</dd>

  <dt><dfn data-lt="target|invocationTarget|targets">target
    (invocationTarget)</dfn></dt>
  <dd>See the <a href="https://w3c-ccg.github.io/wallet-attached-storage-spec/#dfn-target">target
    definition</a> in [[WAS]]. The resource a capability authorizes action on,
    as an absolute URL.</dd>

  <dt><dfn>unsatisfiable</dfn></dt>
  <dd>The verdict on a capability request entry that cannot be fulfilled: its
    [=invocation target descriptor=] does not resolve to a [=target=] the
    [=wallet=] can delegate under this profile's rules, or the wallet does not
    recognize what the entry asks for. An unsatisfiable entry produces no
    [=grant=]: the wallet skips it at delegation time and shows it on the
    consent surface as something it cannot fulfill. See
    [[[#descriptor-model]]].</dd>

  <dt><dfn data-lt="wallets">wallet</dfn></dt>
  <dd>The software that holds the user's credentials and controls the user's
    [=Space=], and that processes an [=AppConnectQuery=]. One of the two
    conformance classes; see [[[#conformance-classes]]]. A wallet is able to
    delegate capabilities because it can invoke the Space's root
    capability.</dd>
</dl>

## The App Connect Request {#request}

### Transport {#request-transport}

An App Connect request is a verifiable presentation request [[VCALM]] whose
`query` member contains exactly one [=AppConnectQuery=].

For an [=application=] running in a web browser, the request is delivered to
the wallet over [=CHAPI=]. CHAPI is a browser API, so it applies only to
in-browser applications; it is also the deployment this document's normative
prose is written against.

<div class="note">
**CHAPI is the in-browser transport mechanism.** The request and
response are plain [[VCALM]] bodies -- a verifiable presentation request in,
a signed verifiable presentation out -- and carry nothing CHAPI-specific. An
application that does not run in a browser (a native mobile application, or a
command-line tool) delivers the same request over another protocol capable of
carrying a [[VCALM]] exchange, such as VCALM's own exchange endpoints, and
receives the same response presentation back.

What any such transport must replicate is CHAPI's one load-bearing property:
attesting the identity of the requesting party to the wallet
([[[#security-origin]]]). Wherever this document relies on the
browser-attested [=origin=], a non-browser transport must supply an
equivalently attested application identifier, attested by the transport or
platform, for the wallet to bind the
[=app-key credential=] to. Such an application is still a [=public client=]:
the platform attestation stands in for the browser's origin attestation, the
attested identifier occupies the credential's `origin` claim
([[[#app-key-binding]]]), and it is the value meant wherever the normative
prose (written against the in-browser deployment) says "origin"
or "live browser origin". How a given non-browser transport attests its
caller is left for future work.
</div>

The request body MUST carry `challenge` and `domain` members
([[[#challenge-and-domain]]]).

Every application this profile addresses is a [=public client=]: it holds no
durable secret of its own, and its only trustworthy identification is what
the transport or platform attests, which for an in-browser application means the
[=origin=]. An [=application=] MUST NOT rely on any identification of itself
beyond that attested identifier. In particular, the `app` block described
below is display and matching metadata; it is not evidence of who is asking.

<div class="note">
**Confidential clients are left for future work.** A [=confidential
client=] -- a server-backed application that can authenticate itself by
means the transport or an external registry attests, such as a TLS client
certificate or a DID published in a trusted registry -- does not need this
profile's custodial identity model: it can hold its own keys, so the seed
custody and origin matching this document builds on do not apply to it
as-is. A profile connecting confidential clients to WAS-backed storage is
left for a future document.
</div>

### The AppConnectQuery {#appconnectquery}

An [=AppConnectQuery=] is a JSON object with the following members.

| Member            | Required | Value                                                                         |
|-------------------|----------|-------------------------------------------------------------------------------|
| `type`            | yes      | The string `AppConnectQuery`.                                                 |
| `app`             | yes      | An object naming the requesting application; see below.                       |
| `capabilityQuery` | no       | A capability request entry, or an array of them; see [[[#capability-query]]]. |

The `app` object has exactly two members, and a [=wallet=] MUST treat the
query as malformed unless both are present and are strings:

| Member   | Value                                                                                                                                                                  |
|----------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `name`   | A human-readable application name, for the wallet's consent surface. Display only; it is attacker-controlled free text and MUST NOT be treated as evidence of identity. |
| `appUrl` | The application's canonical URL, identifying it among the applications on its origin. Used to match an existing [=app-key credential=] or to mint a new one ([[[#app-key-matching]]]). |

The `appUrl` value MUST parse as an absolute URL [[URL]], MUST NOT carry a
fragment, and its origin MUST equal the attested requesting [=origin=]. A
[=wallet=] MUST treat a query violating any of these as malformed. Wherever
this document stores or compares an `appUrl`, the value used is the parsed
URL's serialization, so spellings that differ only in a default port, in
percent-encoding case, or in dot-segments do not name distinct applications.

<div class="note">
`appUrl` is what distinguishes multiple applications served from one origin:
an application identity is scoped to the pair ([=origin=], `appUrl`), and
applications sharing an origin use distinct URLs -- typically distinct paths.
The same scoping means an application cannot hold more than one identity at
one URL; a deployment that wants two identities is two applications, at two
URLs. This mirrors how the Web App Manifest identifies an installable
application: its `id` member is likewise a URL within the application's origin
[[APPMANIFEST]]. An application that has a manifest is well served by using
its processed manifest `id` as its `appUrl` -- once processed, both are
absolute same-origin URLs, and the application then carries one identity on
the platform and in this profile. Because the browser attests nothing finer
than the origin,
everything below the origin is cooperative namespacing, not isolation
([[[#security-origin]]]).
</div>

A [=wallet=] that does not recognize the `AppConnectQuery` type MUST NOT
attempt to satisfy it partially. Its response will therefore carry no [=app-key
credential=], which the application detects per [[[#wallet-unsupported]]].

### Exclusivity {#request-exclusivity}

An App Connect request asks for exactly one thing: a connection, so that
the user consents to a single, clearly described interaction rather than a
bundle of unrelated requests. A [=wallet=] MUST enforce the rules below, and
MUST treat a violation as a malformed request rather than resolving it in the
application's favour:

1. A request MUST NOT carry more than one `AppConnectQuery`.
2. A request carrying an `AppConnectQuery` MUST NOT carry any query other than
   that `AppConnectQuery` and a `DIDAuthentication` query [[VCALM]]. That
   permitted set is exhaustive: a query of any other type, whether or not the
   wallet recognizes the type, makes the request malformed.
3. A conformant [=application=] MUST also include a `DIDAuthentication`
   query [[VCALM]] alongside the `AppConnectQuery`. A [=wallet=] MUST NOT
   treat its absence as malformed; it MUST accept such a request and answer
   it with an unsigned presentation.

Rule 2 is written as a closed permitted set rather than as a list of refused
types so that query types defined after this document inherit the refusal
without a revision here. It is this profile's fail-closed processing rule
([[[#security-fail-closed]]]) applied to the query list.

<div class="note">
In particular, an `AppConnectQuery` cannot be combined with a `QueryByExample`
query, with a standalone `AuthorizationCapabilityQuery` (or its legacy alias
`ZcapQuery`), or with a `WalletOnboardingQuery` -- the wallet-to-wallet
onboarding rendezvous query some of the same wallets exchange, which no
specification defines at the time of writing. These are illustrations of rule
2 rather than the rule itself: an implementation that enumerates refused types
instead of checking the permitted set accepts requests this profile treats as
malformed.
</div>

The exclusivity rules exist because an App Connect exchange has one consent
surface describing one relationship. Mixing in a credential-sharing query, a
standalone capability query, or any other request type would put two unrelated
decisions behind one approval, with the second one described by the requesting
party's own free-text `reason` strings or by whatever surface that type would
otherwise carry.

### Capability requests {#capability-query}

The `capabilityQuery` member holds the capability requests to be delegated to
the [=connecting DID=]. A [=wallet=] MUST normalize it as follows:

* an **absent** `capabilityQuery` normalizes to the empty list. A request that
  asks for no capabilities at all is legal and is satisfied by returning the
  [=app-key credential=] alone: an application may connect purely to recover
  its own key material.
* a **single object** normalizes to a one-element list.
* an **array** normalizes to itself.
* an entry that is not an object is malformed, and the
  wallet MUST treat the whole query as malformed rather than skipping the entry.

Each entry has the shape of a capability request detail [[VCALM]], **minus two
members**:

| Member             | Required | Value                                                                                                              |
|--------------------|----------|--------------------------------------------------------------------------------------------------------------------|
| `invocationTarget` | yes      | An [=invocation target descriptor=]; see [[[#descriptors]]].                                                       |
| `allowedAction`    | yes      | A non-empty string or array of strings naming the requested [=actions=]; see [[[#action-vocabulary]]].             |
| `referenceId`      | no       | An application-chosen label. Opaque to the wallet; see [[[#reference-id]]].                                        |

A [=wallet=] MUST treat an entry with no `allowedAction` member, or one whose
value is an empty array, as making the whole query malformed. There is no
default: an application states the actions it wants, even for a target class
that can only ever yield reads. Absence has no safe meaning to fall back on --
in [[ZCAP]] an absent `allowedAction` means the capability does not restrict
actions, the opposite of "did not ask" ([[[#allowed-actions]]]).

* An entry MUST NOT carry a `controller`. The [=wallet=] fills it with the
  subject DID of the [=app-key credential=] it matched or minted
  ([[[#delegation-target]]]). This is the member a public client cannot
  supply on a first run, and dropping it is what collapses the flow to one
  round. A wallet MUST ignore any `controller` an entry does carry.
* An entry MUST NOT carry a `reason`. The App Connect consent surface describes
  the relationship as a whole and supersedes per-grant reason strings
  ([[[#consent]]]). A wallet MUST NOT display a `reason` an entry does carry.
* An entry MUST NOT carry any member not named in this section. A [=wallet=]
  MUST treat an entry carrying an unrecognized member as making the whole
  query malformed, exactly as for a non-object entry -- not skip the member,
  and not skip the entry. Ignoring the member would leave a misspelling --
  `allowedActions`, say -- indistinguishable from an entry that omitted the
  member, and the omission rule above only fails visibly if nothing
  unrecognized can stand in for the real member.

An [=application=] SHOULD NOT name the same collection in more than one request
entry. If it does, which of the resulting [=grants=] governs the application's
routing for that collection is unspecified by this profile.

#### Correlating entries with grants {#reference-id}

`referenceId` is **opaque to the wallet** and is **not echoed** in the response.
A wallet MUST NOT be required to carry it onto a delegated grant, and an
[=application=] MUST NOT rely on it appearing there.

An application MUST correlate a returned [=grant=] with the request entry that
produced it by the grant's `invocationTarget` URL -- specifically by the
collection segment of that URL ([[[#url-template]]]). For example, if an
application asks for a collection named `notes`, the grant answering that entry
is the one whose `invocationTarget` ends in `/notes` (say,
`https://wallet-storage.example/space/81246131-69a4-45ab-9bff-9c946b59cf2e/notes`).
The match is on the full path segment, bounded by the `/` delimiter -- not a
substring or bare suffix of the URL. Requests for collections named `notes`
and `published-notes` are the case that distinguishes the two readings: a
grant ending in `/published-notes` does not match the `notes` entry, though a
naive suffix match on the bare name would accept it.
The rest of the URL cannot serve as a match key: the host and Space id are
chosen by the wallet, and the application learns them only by parsing the
grant.

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
requesting [=origin=]. (The comparison is on the url host). A wallet MUST apply
this check to any request carrying a `domain`, regardless of whether it also
carries a `DIDAuthentication` query, and MUST surface a domain mismatch as a
distinct refusal rather than as a generic processing error.

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

### Example request {#request-example}

A complete App Connect request: DID authentication plus one App Connect query
asking for one private application collection, one public collection, and
read-and-decrypt access to one collection the wallet already owns.

Each descriptor's `name` becomes the collection segment of the resolved
target URL ([[[#descriptor-registry]]]): `"name": "notes"` resolves to a
full URL ending in `/notes` (for example,
`https://wallet-storage.example/space/81246131-69a4-45ab-9bff-9c946b59cf2e/notes`),
and that segment is what the application later correlates the returned
grants by ([[[#reference-id]]]).

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
        "appUrl": "https://app.example/notes/"
      },
      "capabilityQuery": [
        {
          "referenceId": "notes",
          "allowedAction": ["GET", "HEAD", "PUT", "POST", "DELETE"],
          "invocationTarget": {
            "type": "https://w3id.org/byoe#private-collection",
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
            "type": "https://w3id.org/byoe#shared-wallet-collection",
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
[=Space=]. It does not know the storage host, and it does not know the Space
id, so it cannot name a concrete target. It names an *abstract descriptor*
instead, and the [=wallet=] resolves the descriptor against the Space it
controls.

Resolution either yields a concrete target inside the user's Space, together
with the target class that bounds it ([[[#allowed-actions]]]), or determines
that the entry cannot be fulfilled, in which case the entry is [=unsatisfiable=].

An `invocationTarget` MUST be either:

* an **object** with a `type` member holding a descriptor type IRI, and an
  optional `name` member; or
* a **string** holding an absolute URL ([[[#string-targets]]]).

A [=wallet=] MUST determine the target class from the resolved target itself,
so that a descriptor object and the equivalent URL string resolve to the same
class and are bounded identically. A string target MUST NOT be able to reach a
broader allowed-action set than the descriptor form of the same target.

Honoring that parity requires the wallet to know the classification of its own
collections, independently of how a request named it. In particular, it should
know which of them are public ([[[#descriptor-public-collection]]]) and which
are protected ([[[#protected-collections]]]), since a string target carries no
descriptor type to classify it by.

An [=unsatisfiable=] request entry MUST NOT produce a [=grant=]. The wallet
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

| Type IRI                                         | `name`   | Resolves to         | Allowed actions                        |
|--------------------------------------------------|----------|---------------------|----------------------------------------|
| `https://w3id.org/byoe#private-collection`       | required | `<spaceUrl>/<name>` | `GET`, `HEAD`, `POST`, `PUT`, `DELETE` |
| `https://w3id.org/byoe#public-collection`        | required | `<spaceUrl>/<name>` | `GET`, `HEAD`, `POST`, `PUT`, `DELETE` |
| `https://w3id.org/byoe#shared-wallet-collection` | required | `<spaceUrl>/<name>` | `GET`, `HEAD`                          |

Where `name` is required, a [=wallet=] MUST validate it against the collection
name grammar ([[[#collection-name-grammar]]]) and MUST resolve the descriptor
[=unsatisfiable=] when it does not match.

#### `#private-collection` {#descriptor-private-collection}

`https://w3id.org/byoe#private-collection` requests an application-scoped
[=collection=] in the user's Space, named by `name`.

* It resolves to `<spaceUrl>/<name>`.
* If the named collection does not already exist, the wallet MUST provision it
  before delegating, per [[[#provisioning]]].
* A collection provisioned under this descriptor MUST be encrypted from
  creation ([[[#encrypted-by-default]]]).
* Its allowed actions are the full action vocabulary: this is the
  application's own data, and the consent surface plus the shorter write
  lifetime ([[[#ttl]]]) are what bound it.
* If `name` happens to be one of the wallet's own protected collections
  ([[[#protected-collections]]]), the resolved class is
  *protected collection* and the read-only bound applies instead. A wallet
  MUST NOT provision over a protected collection.

#### `#public-collection` {#descriptor-public-collection}

`https://w3id.org/byoe#public-collection` requests a [=collection=] whose
contents are readable by anyone on the web without authorization.

* It resolves to `<spaceUrl>/<name>`.
* The wallet MUST provision it as
  **[plaintext](https://w3c-ccg.github.io/wallet-attached-storage-spec/#plaintext-collection)**
  ([[WAS]]). A public collection is never
  encrypted, and MUST set a collection-level world-readable access policy on
  it. The policy is set by the wallet, which controls the Space; a delegated
  capability could not set it.
* Public covers unauthenticated **reads only**. Writes remain
  capability-authorized, so the wallet still delegates an ordinary
  collection-scoped capability alongside setting the policy.
* Its allowed actions are the full action vocabulary, the same as a private
  collection's: published content is still the application's own data, and
  un-publishing is as much data management as publishing. An application may
  rewrite (`PUT`) or retract (`DELETE`) what it published. Retraction removes
  the stored copy, not copies already fetched -- that is the nature of
  publication, not a reason to forbid it.
* **A public collection is only ever created public, never converted.** A
  `#public-collection` descriptor naming a collection that *already exists and
  is not already public* MUST resolve [=unsatisfiable=]. A wallet MUST NOT
  make an existing non-public collection world-readable in response to a
  request.
* An idempotent re-grant is unaffected: a `#public-collection` descriptor
  naming a collection that already exists *and is already public* stays
  satisfiable, and reconnecting an application to the collection it previously
  published into behaves exactly as the first connect did
  ([[[#provisioning]]]).
* A `#public-collection` descriptor naming a protected collection
  ([[[#protected-collections]]]) MUST resolve [=unsatisfiable=],
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

#### `#shared-wallet-collection` {#descriptor-shared-wallet-collection}

`https://w3id.org/byoe#shared-wallet-collection` requests read-and-decrypt access
to an encrypted collection the wallet already owns.

* It resolves to `<spaceUrl>/<name>`.
* `name` MUST name a collection the wallet itself owns and encrypts -- one
  of the wallet's own standard encrypted collections. A plaintext collection, a
  collection provisioned for an application, the [=Space=] itself, and a name
  the wallet does not recognize MUST all resolve [=unsatisfiable=].
* Because satisfiability is decided by looking the name up in the set of
  collections the wallet owns and encrypts, that lookup subsumes the collection
  name grammar check ([[[#collection-name-grammar]]]) for this descriptor type:
  a name that fails the grammar cannot be in the set.
* Its granted actions are exactly the class's allowed actions, `GET` and `HEAD`
  ([[[#allowed-actions]]]) -- not an intersection with what was requested. A
  share is never a write grant, and `HEAD` rides every read grant
  ([[[#action-vocabulary]]]), so a share that asked for a narrower subset still
  receives both.
* **The two axes are fused.** Satisfying a share means granting *both* the
  read-only [=capability=] (the fetch axis) *and* [=epoch-roster recipient=]
  status for the collection's current epoch (the decrypt axis). A [=wallet=]
  MUST NOT grant either axis alone, and MUST NOT allow a *durable partial
  state* in which one axis stands without the other. Where the two cannot be
  established in a single atomic step, the wallet MUST complete or repair the
  missing half -- at the latest on a re-run of the exchange
  ([[[#resumability]]]) -- and MUST NOT report the share as granted until both
  halves stand.
* The recipient key is never carried in the request; see
  [[[#recipient-derivation]]].

Granting only the capability would hand the application ciphertext it cannot
open, which surfaces to a user as corrupt data rather than as a failed share.
Granting only recipiency would leave a reader in the roster with no way to
fetch.

**A share covers what is already stored.** Adding the grantee to the roster
escrows every epoch's key to it, not only the current epoch's (the escrow rule
of [[WAS-EC]]'s roster operations), so the grantee can decrypt the collection's
existing contents, not only what is written after the share. A [=wallet=] MUST
state this on the consent surface ([[[#consent]]]).

**Re-granting a previously revoked grantee restores read access to previously
written content.** Adding a grantee back as an [=epoch-roster recipient=] gives
it the collection as it stands, including whatever was written while it was
revoked. A wallet MUST NOT present a re-grant as forward-only access. Restoring
access from this point forward only, rather than retroactively, requires
rotating the collection to a new key epoch and admitting the grantee to that
epoch alone; the epoch construction that makes this possible is defined in
[[WAS-EC]].

**Revocation of a share is an explicit act.** A [=wallet=]
MUST NOT rely on the delegated capability's lifetime as the removal mechanism
for a share: expiry ends the fetch axis while leaving the grantee an
epoch-roster recipient, which is precisely the unfused state the fusion rule
exists to prevent. Removing a share MUST re-epoch the collection off the
removed recipient and revoke the capability as one operation. See also
[[[#ttl]]].

### Reserved descriptor types {#descriptor-reserved}

The following type IRIs are reserved. This document defines no behavior for
them; they are recorded here so that no other specification, profile, or
implementation assigns them different semantics.

| Reserved type IRI                          | Reserved for                                                                                            |
|--------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `https://w3id.org/byoe#managed-collection` | A future write-bearing grant class for applications acting as the authoritative writer of a collection. |
| `https://w3id.org/byoe#publish-collection` | A future standing publication grant.                                                                    |
| `https://w3id.org/byoe#collection-policy`  | A future runtime access-policy control descriptor.                                                      |
| `https://w3id.org/byoe#space`              | A future Space-scoped read grant; this profile grants collection-scoped access only ([[[#string-targets]]]). |

Until this document (or a successor) defines them, a reserved type is an
unrecognized type and MUST be processed per [[[#unknown-descriptor-type]]]:
a wallet that predates such a feature resolves it [=unsatisfiable=] and refuses
visibly.

### Unrecognized descriptor types {#unknown-descriptor-type}

A [=wallet=] MUST resolve **any** `invocationTarget` descriptor whose `type` it
does not recognize as [=unsatisfiable=].

A wallet MUST NOT fall back to a related recognized type, MUST NOT strip the
type and treat the descriptor as a plain collection request, and MUST NOT
silently narrow the request into something it does understand.

<div class="note">
This is the extensibility safety rule of the whole profile, and it is what
makes new descriptor types deployable at all. The application-side reasoning
runs the other way around: an application that asks for a
`#public-collection` and is answered by a wallet that predates the type sees a
visible refusal, and knows its collection was not created. If the wallet had
instead degraded the request to `#private-collection`, the application would have been
handed a *private* collection it believes is public, and would publish into it.
The same argument covers `#shared-wallet-collection`, whose fused axes cannot be
partially honored, and the `AppConnectQuery` type itself
([[[#wallet-unsupported]]]).
</div>

### String targets {#string-targets}

A capability request MAY carry an absolute URL string as its
`invocationTarget`. A [=wallet=] MUST resolve it as follows.

The wallet MUST parse both the target and its own Space URL as URLs, and MUST
resolve the target [=unsatisfiable=] if any of the following holds:

1. either value does not parse as an absolute URL;
2. the target carries a **query** component;
3. the target carries a **fragment** component;
4. the target's origin differs from the Space URL's origin;
5. after resolving dot segments and trimming trailing slashes, the target's
   path is not a strict descendant of the Space path. In particular, a target
   naming the Space itself is [=unsatisfiable=]: this profile grants
   collection-scoped access only ([[[#descriptor-reserved]]]).

Otherwise the first path segment after the Space path is the collection id.
It MUST match the collection name grammar ([[[#collection-name-grammar]]]) or
the target resolves [=unsatisfiable=]. The resolved class is *protected
collection* if that id names a protected collection
([[[#protected-collections]]]), and *collection* otherwise.

A string target MUST NOT cause provisioning: it names something the wallet
either already has or does not.

<div class="note">
Rules 2 and 3 refuse rather than rewrite. A [[WAS]] server authorizes a
request by matching the capability's target against the request URL's scheme,
host, port, and path alone: a query string is a request modifier (pagination,
for instance), not part of the target's identity, and a fragment never
reaches the server at all. A target carrying either could therefore never
participate in authorization -- the grant would be dead on arrival. And
silently dropping the offending part of a target the user is about to approve
would show the user something other than what gets delegated. Rules 4 and 5 together are why string matching alone is not enough:
`<spaceUrl>/private-credentials?x=1` starts with the Space URL but its first
routed segment is not the collection id it appears to name, and
`<spaceUrl>/../other-space/x` starts with the Space URL while pointing outside
the Space entirely. Rule 5 also refuses the Space URL itself. Space-wide
grants are a hazard class of their own -- a Space-wide write capability would
authorize rewriting the Space Description, and therefore controller takeover
-- and no application this profile addresses can route one
([[[#grant-validation]]]), so the profile does not define them; the type IRI a
future Space-scoped read grant would use is reserved
([[[#descriptor-reserved]]]).
</div>

### Collection name grammar {#collection-name-grammar}

A collection name appearing in a descriptor's `name` member, or as the first
path segment of a string target, MUST match:

```
/^[a-z0-9][a-z0-9-]{0,63}$/
```

That is: lowercase ASCII letters and digits and the hyphen, not starting with a
hyphen, between 1 and 64 characters. A name that does not match resolves the
descriptor [=unsatisfiable=].

### Protected collections {#protected-collections}

A [=wallet=] MUST maintain a set of **protected collections**: the collections
in the user's Space that hold the user's own wallet data and the account's
system state. This MAY include:

* the wallet's own standard content collections -- the collections the wallet
  itself writes the user's credentials, activity, and application-domain data
  into;
* the account's system collections -- those holding the account's published
  identity artifacts and its key material.

Requests resolving onto a protected collection are bounded as follows:

* the resolved class is *protected collection*, whose allowed actions are
  read-only ([[[#allowed-actions]]]). A requesting party may read the user's
  own data
  when the user consents, and may never rewrite or delete it;
* a `#public-collection` descriptor whose `name` resolves to a protected
  collection MUST resolve [=unsatisfiable=]
  ([[[#descriptor-public-collection]]]);
* a wallet MUST NOT provision over one.

A `#shared-wallet-collection` descriptor naming an encrypted protected collection is
the intended way for an application to be given read access to the user's own
data, and resolves in the *share* class rather than the *protected collection*
class. Both classes are read-only, so the distinction affects only the
decrypt axis and the consent copy.

## The App Key Credential {#app-key-credential}

### Purpose {#app-key-purpose}

An [=application=] under this profile is a [=public client=] (an
application running entirely in the browser being the canonical case) and
has nowhere durable and trustworthy to keep a secret of its own. The
[=app-key credential=] answers
this by moving custody: the wallet holds the application's [=seed=] as an
ordinary credential in the user's wallet, and returns it to the same
application, same [=origin=], same `appUrl`, on every connect. This extends
what a [=public client=] can ordinarily do: the application gains a stable,
long-lived identity even though it retains no secret of its own between
visits.

The credential is self-issued and self-authenticating: its issuer, its subject,
and the DID derived from the seed it carries are all the same value
([[[#seed-binds-subject]]]). That is what lets a wallet hand it back after any
interval, and what lets an application check it without consulting anyone.

### Type array {#app-key-type-array}

The credential's `type` member MUST be an array of exactly two entries, in
this order:

```json
["VerifiableCredential", "AppKeyCredential"]
```

The `AppKeyCredential` type makes "presents as an app key" a
term check rather than a shape heuristic, which is what the store-time refusal
([[[#store-time-refusal]]]) and the match predicate
([[[#app-key-matching]]]) key off.

The type array is identical for every application: nothing in the credential's
vocabulary is application-scoped.

### Term URLs {#app-key-urls}

| Term               | URL                                      |
|--------------------|------------------------------------------|
| `AppKeyCredential` | `https://w3id.org/byoe#AppKeyCredential` |
| `appUrl`           | `https://w3id.org/byoe#appUrl`           |
| `seed`             | `https://w3id.org/byoe#seed`             |
| `origin`           | `https://w3id.org/byoe#origin`           |
| `name`             | `https://schema.org/name`                |
| `description`      | `https://schema.org/description`         |

Every term is shared by every application and means the same thing for each.

<div class="note">
The `AppKeyCredential` type is a self-declaration, not evidence: see
[[[#security-planted-credential]]].
</div>

### The context {#app-key-context}

The credential's `@context` MUST be an array whose first two entries are, in
this order:

```json
[
  "https://www.w3.org/2018/credentials/v1",
  "https://w3id.org/byoe/app-connect/v1"
]
```

The second entry is the context document that defines the BYOE terms of this
profile as a whole: the credential's terms, mapped to the URLs of
[[[#app-key-urls]]], and the response presentation's `zcap` and `appConnect`
members ([[[#response-members]]]).
Verifiers resolve it through their document loader like any other static
context; loaders that bundle contexts for offline use pin it by URL, so
verification requires no fetch at verification time.

The signature suite appends its own context entry when the credential is
signed; that entry is the suite's, not this profile's.

The context URL is also the profile's **version handle**: the trailing path
segment (`v1`) names the profile version an implementation speaks, and a
breaking change to the profile -- to the wire shapes this document defines, or
to the terms the context maps -- MUST be published under a new context URL
(`.../app-connect/v2`) rather than by editing the `v1` context in place.
Implementations advertise which version they speak through the context URL
they emit and accept; an implementing package SHOULD name the profile version
in its changelog when the version it speaks changes.

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

<div class="note">
The sibling `appUrl` claim travels with `origin` through the same moments: it
is set at mint time ([[[#app-key-minting]]]) and compared on both sides
([[[#app-key-matching]]], [[[#app-key-parsing]]]). It is not listed here as a
binding because it is not attested: the browser attests nothing finer than the
[=origin=], so `appUrl` namespaces identities within an origin without
isolating same-origin applications from one another ([[[#security-origin]]]).
</div>

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

| Moment     | Party       | Consequence of failure                                                   |
|------------|-------------|--------------------------------------------------------------------------|
| Match time | Wallet      | The credential is not a candidate; the wallet mints a fresh one instead. |
| Store time | Wallet      | The credential is refused at ingest ([[[#store-time-refusal]]]).         |
| Parse time | Application | The response is rejected ([[[#app-key-parsing]]]).                       |

<div class="note">
Self-issuance alone is a weak signal, because anyone can self-issue. The
binding is stronger and is entirely local: the credential carries the seed, so
the check is a re-derivation and a comparison, with no network and no trusted
third party.

**What the binding does and does not establish.** It establishes internal
consistency: that this credential's subject is the DID of the seed this
credential carries. It therefore rejects any credential in which the subject or
the seed has been substituted (a genuine credential with the seed swapped
out, or an attacker's subject DID pasted over a genuine seed) because either
substitution breaks the derivation.

It does not establish provenance, and it is important not to read it as
doing so. An attacker who generates their own seed, derives its DID correctly,
and self-issues a credential naming the victim's [=origin=] and the victim
application's `appUrl` produces a credential that binds perfectly well.
Nothing local can tell that credential from a legitimate one, because it *is* a
legitimate credential -- for the attacker's identity. Keeping such a credential
away from the user's wallet is the job of [[[#store-time-refusal]]], not of this
rule.
</div>

### Matching {#app-key-matching}

To find the [=app-key credential=] for a request, a [=wallet=] MUST select from
its stored credentials those satisfying *all* of:

1. the `type` array includes the `AppKeyCredential` type;
2. `credentialSubject.appUrl` equals the request's `app.appUrl`, both in
   serialized form ([[[#appconnectquery]]]);
3. `issuer` is present and equals `credentialSubject.id`;
4. `credentialSubject.origin` equals the attested requesting [=origin=];
5. the credential binds per [[[#seed-binds-subject]]].

Criterion 1 is what couples this rule to the store-time refusal: carrying that
type is the only way a credential reaches the delegation path, and
type-bearing credentials are exactly the ones [[[#store-time-refusal]]]
keeps out of the wallet's store -- so a credential that matches here was
minted by this wallet.

When more than one credential qualifies, the wallet MUST rank the qualifying
credentials latest-first by the instant their `issuanceDate` denotes and
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

### App Key Credential Minting {#app-key-minting}

To mint an [=app-key credential=], a [=wallet=] MUST:

1. generate 32 bytes from a cryptographically secure random source;
2. derive the [=connecting DID=] from them per [[[#key-derivation]]];
3. assemble the credential with the type array of
   [[[#app-key-type-array]]], the context array of [[[#app-key-context]]],
   `issuer` and `credentialSubject.id` both set to the derived DID,
   `credentialSubject.seed` encoded per [[[#seed-encoding]]],
   `credentialSubject.appUrl` set to the request's `app.appUrl` in serialized
   form ([[[#appconnectquery]]]), and
   `credentialSubject.origin` set to the attested requesting origin;
4. sign it with a signature suite the application can verify, using a signer
   for the seed-derived key, so that the credential is genuinely self-issued;
5. store it in the user's wallet before delegating.

Storing before delegating makes the operation resumable: if delegation fails,
the next attempt matches the stored credential as a returning connection rather
than minting a second identity for the same application and origin.

The credential's `id` SHOULD be a fresh `urn:uuid:` value.

The `name` and `description` members are display fields for the wallet's own
credential list. They are informative; this profile places no requirement on
their content beyond their term URLs ([[[#app-key-urls]]]).

<div class="note">
The wallet mints the seed, rather than the application minting one and asking
the wallet to store it. This is what removes the second round trip (and in cases
of in-browser CHAPI transport, a second popup): a
store-then-request flow needs one exchange to deposit the key and another to
ask for grants, and it also means the seed exists application-side before the
user has consented to anything.
</div>

### Store-time refusal {#store-time-refusal}

**App-key credentials are wallet-minted, never imported.** The only way an
[=app-key credential=] legitimately enters a wallet's store is by that wallet
minting it during an App Connect exchange ([[[#app-key-minting]]]).

Accordingly, a [=wallet=] MUST refuse to store any credential that carries the
`AppKeyCredential` type and did not originate in an [=App Connect exchange=]
this wallet performed. The refusal MUST NOT depend on whether the credential binds
per [[[#seed-binds-subject]]]: a credential carrying the `AppKeyCredential`
type arriving from outside is refused regardless of whether it is internally
consistent.

The refusal MUST be applied at every path by which a credential enters the
wallet's store from outside: a credential offered over the transport, or
imported from a URL, a QR code, a file, or a paste.

A [=wallet=] MUST additionally refuse to store an `AppKeyCredential`-typed
credential that does not bind per [[[#seed-binds-subject]]], on any path
whatsoever. The two refusals are independent, and either one alone is enough
reason to refuse.

A credential that does *not* carry that type MUST NOT be caught by these
rules, even if it happens to carry a `seed` or `origin` claim.

<div class="note">
This provenance rule is stricter than checking
the credential's internal consistency because internal consistency is not the
property under attack. An attacker can mint a perfectly well-formed app-key
credential for their own seed, name the victim application's `appUrl`
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

The binding refusal is retained alongside it as defense in depth, covering
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
    "https://w3id.org/byoe/app-connect/v1",
    "https://w3id.org/security/suites/ed25519-2020/v1"
  ],
  "id": "urn:uuid:3f2a8c40-9d51-4e0b-9f1c-5c6d0a2b7e34",
  "type": ["VerifiableCredential", "AppKeyCredential"],
  "name": "Example Notes app key",
  "description": "The Example Notes app keeps this key in your wallet so it can open your encrypted data on this and other devices.",
  "issuer": "did:key:z6MkExampleNotesAppKeySubjectDidPlaceholder",
  "issuanceDate": "2026-08-06T14:22:11Z",
  "credentialSubject": {
    "id": "did:key:z6MkExampleNotesAppKeySubjectDidPlaceholder",
    "seed": "b0dRQXBwS2V5U2VlZFBsYWNlaG9sZGVyVmFsdWUzMkI",
    "appUrl": "https://app.example/notes/",
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

### Response Members {#response-members}

A [=wallet=] answering an App Connect request MUST return a single Verifiable
Presentation carrying:

| Member                 | Value                                                                 |
|------------------------|-----------------------------------------------------------------------|
| `verifiableCredential` | An array containing the matched or minted [=app-key credential=].     |
| `zcap`                 | An array of the delegated [=grants=]. Omitted when no grant was made. |
| `appConnect`           | `{ "firstRun": <boolean> }`. See [[[#first-run]]].                    |

The `zcap` array holds one entry per satisfiable request entry that was
approved. [=Unsatisfiable=] and declined entries produce no response entry, so
the array is in general shorter than the request's `capabilityQuery` and is not
index-aligned with it; see [[[#reference-id]]] for how an application correlates
the two.

Whichever of these members the response carries MUST be added to the
presentation *before* it is signed, so that the authentication proof covers
them. A wallet MUST NOT append grants or the `appConnect` member to an
already-signed presentation. A member that is empty is omitted; omission is
not a way to add it later.

When the response carries either member, the wallet MUST append the profile's
context, `https://w3id.org/byoe/app-connect/v1` ([[[#app-key-context]]]), to
the presentation's `@context` before signing. The context defines the `zcap`
and `appConnect` terms, which is what allows JSON-LD safe-mode
canonicalization [[JSON-LD11]] to include the members in the signed payload
rather than reject them; `appConnect` is typed `@json` there, so its contents
canonicalize as one opaque literal. Because the context is appended before
signing, the signature suite's own entry, appended by the suite when the
presentation is signed, follows it in the resulting `@context` array.

Only the top-level terms are defined. The embedded capabilities' own
sub-vocabularies MUST NOT be hoisted into the presentation context; each
embedded capability self-describes through its own `@context`.

The presentation is signed by the wallet account's own DID, the value of the
presentation's `holder` property, not by the [=connecting DID=]. This is the
identity the wallet holds for the user, typically the same identity that
controls the user's [=Space=]; per [[[#identity-model-agnosticism]]], this
profile never inspects it, and the application checks only that the
authentication proof answers its challenge and domain. The wallet is the
party answering the challenge. It is the one attesting that this response
left this wallet for this domain, so it is the holder. The two attestations are separate and complementary: the
authentication proof attests that this wallet answered this challenge from this
domain, and the [=app-key credential=]'s self-issued proof plus the
seed-to-subject binding attest the identity being handed over.

Each entry in `zcap` is a self-contained delegated capability carrying its own
`@context` and its own delegation proof; the entries self-authenticate
independently of the presentation's proof.

### The firstRun member {#first-run}

A [=wallet=] MUST set `appConnect.firstRun` to `true` if and only if it minted
a new [=app-key credential=] during this exchange, and to `false` when it
matched an existing one.

An [=application=] MUST treat *only* the boolean value `true` as first run.
An absent `appConnect` member, an absent `firstRun` member, and any non-boolean
or `false` value all mean "not first run".

`firstRun` is advisory: it tells an application whether to run whatever
one-time setup it does for a brand-new user. It is not a security signal, and
an application MUST NOT use it to decide whether to trust the credential.

### Application-side verification {#response-verification}

An [=application=] MUST perform the following steps, in this order, and MUST
abort on the first failure.

1. **Verify the presentation cryptographically.** Verify the presentation
   proof and every embedded credential proof. For the [=public clients=] this
   document profiles, issuer-registry lookup MUST NOT be required: the
   [=app-key credential=] is self-issued by design.

2. **Check the proof's purpose, challenge, and domain.** The presentation MUST
   carry at least one presentation-level proof. Every presentation-level proof
   MUST have `proofPurpose` of `authentication`. Every such proof's `challenge`
   MUST equal the fresh challenge this request sent, and its `domain` MUST
   equal the origin this request sent.

   <div class="note">
   Requiring *every* presentation-level proof to be an authentication proof,
   rather than just finding one, is deliberate. A verifier selects one proof
   purpose for the whole set and skips the rest. Consider a presentation
   ordered `[assertionMethod, authentication]`: the verifier could verify it
   under the assertion purpose and not signature-check the authentication
   proof (the only freshness bind) while still reading the challenge and
   domain off it.
   </div>

3. **Locate the app-key credential.** Search the embedded credentials for one
   whose `credentialSubject.appUrl` equals the `app.appUrl` this request sent.
   The `AppKeyCredential` type MUST NOT be required at this step.

   If no such credential is present, the outcome is
   [[[#wallet-unsupported]]].

   <div class="note">
   Matching on the `appUrl` claim alone, and requiring the `AppKeyCredential`
   type only at the next step, is what keeps "the wallet returned a credential
   that is wrong" from being indistinguishable from "the wallet returned
   nothing". If the type were required here, a returned credential missing it
   would look
   like an absent credential, and an application that reads absence as first
   run would answer by minting a second key.
   </div>

4. **Parse the credential.** Apply the parse checks of
   [[[#app-key-parsing]]], obtaining the [=seed=] and the [=connecting DID=].

5. **Validate the grants.** Apply the grant checks of
   [[[#grant-validation]]] against the [=connecting DID=] obtained in step 4
   -- not against any DID taken from the response.

### App-key parse checks {#app-key-parsing}

At step 4 above, an [=application=] MUST enforce *all* of the following, and
MUST reject the response if any fails:

1. the `type` array includes `AppKeyCredential`;
2. `credentialSubject.appUrl` equals the `app.appUrl` this application sent in
   the request, compared as an **exact string**;
3. `issuer` is present, `credentialSubject.id` is present, and they are equal;
4. `credentialSubject.origin` equals this application's own origin, compared as
   an **exact string**;
5. `credentialSubject.seed` is a non-empty string that decodes as base64url
   without padding to exactly 32 bytes;
6. the [=connecting DID=] derived from those bytes per [[[#key-derivation]]]
   equals `credentialSubject.id`.

The origin value used in check 4 MUST be the same value the application sent
as the request's `domain` ([[[#challenge-and-domain]]]) (its own live browser
origin). An application MUST NOT check the credential against one origin while
having requested under another, and MUST NOT take either value from a
configuration that can drift from the origin it is actually running on.

These checks duplicate checks the wallet already made. That is the point: they
are defense in depth, and an application that omits them is trusting the wallet
about an origin binding and an identity binding that it is fully able to check
itself.

### Grant validation {#grant-validation}

At step 5 of [[[#response-verification]]], an [=application=] MUST check that:

1. every [=grant=]'s `controller` equals the [=connecting DID=] parsed from the
   credential;
2. every grant carries an `expires` value that is a string and is not already
   past;
3. every grant's `invocationTarget` is a single non-empty string. A grant
   whose `invocationTarget` is absent, is not a string, or is an array of
   targets MUST be rejected;
4. every grant's target parses under the [[WAS]] URL template
   ([[[#url-template]]]) and is collection-scoped -- that is, it names a
   [=collection=] or a [=resource=] within one. A Space-scoped target, or a
   target naming a reserved sub-endpoint rather than a collection, MUST be
   rejected;
5. all grants resolve to a single storage host and a single [=Space=];
   a grant set spanning two hosts or two Spaces MUST be rejected;
6. every collection the application requested as its own (that is, via
   `#private-collection` or `#public-collection`) is covered by a grant whose
   `allowedAction` includes every action the application requires of it, per
   [[[#required-actions]]];
7. at least one grant is present whenever the application requested at least
   one of its own collections.

Delegation-chain validity is *not* the application's to check: it is enforced
by the storage server at invocation time, per [[WAS]]. What the application
checks is structure.

<div class="note">
Check 4 mirrors the wallet side: no descriptor resolves to a Space-scoped
target, and a string target naming the Space itself is [=unsatisfiable=]
([[[#string-targets]]]), so a conformant wallet never emits a Space-scoped
grant. The application rejects one anyway, because its routing is
per-collection and it has nothing to do with a target it cannot route. An
application that wants Space-scoped access is outside the scope of this
version ([[[#descriptor-reserved]]]).
</div>

#### Required actions and the class's allowed actions {#required-actions}

The actions an [=application=] requires of a collection MUST be within the
allowed actions of the descriptor class it used to request that collection
([[[#allowed-actions]]]):

| Requested via         | Actions the application may require                  |
|-----------------------|------------------------------------------------------|
| `#private-collection` | any subset of `GET`, `HEAD`, `POST`, `PUT`, `DELETE` |
| `#public-collection`  | any subset of `GET`, `HEAD`, `POST`, `PUT`, `DELETE` |

An application MUST NOT require an action outside the allowed actions of the
class it requested, and MUST NOT fail a connection because a grant lacks such an
action.

<div class="note">
Both collection classes currently allow the full action vocabulary, so today
this rule bounds nothing. It exists so the two conformance classes cannot
contradict each other when a class *does* bound actions -- as future
descriptor classes may ([[[#descriptor-registry]]]). An application that
required an action its class never allows would reject the capped grant that a
**correct** wallet is obliged to return, so no conformant wallet could ever
satisfy it.
</div>

### Declined shares {#declined-shares}

A `#shared-wallet-collection` [=grant=] SHOULD NOT be login-gating. If the user
declines a share, or the wallet resolves it [=unsatisfiable=], the connection
SHOULD still succeed: the application simply does not open a reader for that
collection.

An application SHOULD NOT include shared collections in the coverage check of
[[[#grant-validation]]] step 6, and SHOULD tolerate a response in which it
requested only shared collections and received no grants at all.

A share is the user granting access to their *own* data. Declining it is a
legitimate answer to that question, and it says nothing about whether the user
wants to use the application.

### The wallet-unsupported outcome {#wallet-unsupported}

If the presentation verifies but carries no [=app-key credential=], an
[=application=] MUST treat this as a *distinct fail-closed outcome*: the
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
    "https://w3id.org/byoe/app-connect/v1",
    "https://w3id.org/security/suites/ed25519-2020/v1"
  ],
  "type": ["VerifiablePresentation"],
  "holder": "did:webvh:QmExampleWalletAccountScidPlaceholder:wallet.example",
  "verifiableCredential": [
    {
      "type": ["VerifiableCredential", "AppKeyCredential"],
      "issuer": "did:key:z6MkExampleNotesAppKeySubjectDidPlaceholder",
      "credentialSubject": {
        "id": "did:key:z6MkExampleNotesAppKeySubjectDidPlaceholder",
        "seed": "b0dRQXBwS2V5U2VlZFBsYWNlaG9sZGVyVmFsdWUzMkI",
        "appUrl": "https://app.example/notes/",
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

1. **Resolution** has no side effects. Each request entry is resolved to a
   target class and a capped action set ([[[#descriptors]]],
   [[[#allowed-actions]]]) with no provisioning and no delegation. Resolution
   is what the consent surface renders.
2. **Delegation** runs only after the user approves. It provisions what needs
   provisioning ([[[#provisioning]]]) and delegates each satisfiable grant.

Every delegation MUST be rooted at the user's Space root capability, and MUST
target a URL inside that Space. Targets outside the Space are [=unsatisfiable=]
by construction ([[[#string-targets]]]), so a delegation rooted elsewhere cannot
arise.

### Resumability {#resumability}

The delegation phase is not atomic: it may provision several collections, admit
the grantee to several rosters, and mint several delegations, and it can fail
partway through any of that.

A [=wallet=] MUST ensure that such a failure leaves the exchange *repairable by
re-running it*. Concretely:

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
[[[#descriptor-shared-wallet-collection]]] requires the missing half to be completed on
a later run rather than requiring the pair to be atomic.
</div>

### Grant recording {#grant-recording}

A [=wallet=] MUST retain, for each grant it delegates, a record sufficient to
revoke that grant later and to present it to the user as standing access.

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

The action vocabulary is closed: the five tokens below are the only actions
this profile recognizes, and the set is not extensible by request. No
mechanism in this profile lets an application, a wallet, or a future
descriptor class introduce an action outside it.

```
GET, HEAD, POST, PUT, DELETE
```

A [=wallet=] MUST normalize each requested action by trimming it, upper-casing
it, and intersecting it with this set. A token outside the set, such as an unknown
verb, a non-string, or an action a future server version might support, MUST
be *dropped*. It MUST NOT be passed through into an `allowedAction` array the
user's root key signs.

`HEAD` is a tolerated read alias, not an action of its own. [[WAS]] authorizes
a `HEAD` request as a `GET`; this profile includes `HEAD` in every read grant
and caps it exactly as `GET` is capped, so it never appears in an
allowed-action set that does not already permit reads.

<div class="note">
Dropping unknown action tokens is the same fail-closed treatment an
unrecognized descriptor type gets ([[[#unknown-descriptor-type]]]), applied one
level down.
</div>

### Allowed actions {#allowed-actions}

Attenuation in this profile is a **table, not a switch**. Every satisfiable
target resolves into exactly one target class, and each class has a set of
allowed actions that the normalized requested actions are intersected against.
Nothing outside a class's allowed actions is ever granted, no matter what the
request asks for.

This table is normative.

| Target class         | Allowed actions                        | Rationale                                                                                                  |
|----------------------|----------------------------------------|------------------------------------------------------------------------------------------------------------|
| protected collection | `GET`, `HEAD`                          | A requesting party may read the user's own data on consent; it may never rewrite or delete it.             |
| share                | `GET`, `HEAD`                          | A share hands over decryption as well as fetch; it is never a write grant.                                 |
| public collection    | `GET`, `HEAD`, `POST`, `PUT`, `DELETE` | The application's own published data; un-publishing and revision are data management like any other write. |
| collection           | `GET`, `HEAD`, `POST`, `PUT`, `DELETE` | The application's own data, bounded by consent and by the shorter write lifetime.                          |

The resulting `allowedAction` array MUST be ordered as in the table above, not
by the order the request asked in, so that equivalent requests yield
byte-identical grants.

If the intersection is *empty* (the request asked only for actions the
class forbids, or only for tokens outside the vocabulary) the entry MUST
resolve [=unsatisfiable=]. A wallet MUST NOT delegate a capability with an
absent or empty `allowedAction`. In [[ZCAP]], an absent member does not
restrict actions at all. A present-but-empty array is malformed in the zcap
data model, and a verifier that skipped the shape check would likewise read
it as unrestricted.

### Delegation target {#delegation-target}

A [=wallet=] MUST set every delegated grant's `controller` to the subject DID
of the [=app-key credential=] it matched or minted in this same exchange.

The request never names a controller ([[[#capability-query]]]), and a wallet
MUST NOT accept one from the request. This is the rule that makes the exchange
single-round: for the [=public clients=] this document profiles, on first run
the identity being delegated to does not exist until the wallet creates it, so
only the wallet can name it.

It is also what makes grantee DIDs per-user by construction: the [=seed=] is
minted per (user, application, origin), so the [=connecting DID=] derived from
it is distinct for every user ([[[#privacy-per-user]]]).

<div class="note">
A channel where the requesting party names its own `controller` -- such as a
standalone `AuthorizationCapabilityQuery` [[VCALM]] -- cannot make that
guarantee: nothing in such an exchange reveals whether the same controller was
named for another user. That gap is one more reason this profile never accepts
a requester-named controller.
</div>

### Recipient-key derivation {#recipient-derivation}

Where an App Connect grant makes the grantee an [=epoch-roster recipient=],
that is, a `#shared-wallet-collection` grant ([[[#descriptor-shared-wallet-collection]]]) or
the provisioning of an encrypted application collection
([[[#encrypted-by-default]]]), the recipient's key-agreement key MUST be
*derived from the grantee's controller DID*.

A [=wallet=] MUST NOT accept a recipient key supplied in the request, in any
form. An explicit recipient key would let a request pair one entity's
controller DID with another entity's decryption key; deriving makes that
substitution impossible by construction and keeps both axes of a share pointing
at the same entity.

If the controller DID has no derivable key-agreement key, the entry MUST
resolve [=unsatisfiable=]. A wallet MUST reach that verdict by attempting the
real derivation, not by inspecting the DID's shape: a well-formed-looking but
malformed DID would otherwise pass a shape check and fail only when the
derivation is actually performed.

For a `#shared-wallet-collection` entry, a wallet MUST perform that derivation during
resolution, while the consent surface is being built, whenever the
controller is known at that point, so a share the wallet cannot honor is shown
as [=unsatisfiable=] rather than failing mid-delegation.

<div class="note">
On a first-run exchange the [=connecting DID=] does not exist during resolution
(the credential is not minted until after consent), so an *absent* controller
at resolution time is not yet a failure. The wallet re-checks with the real
subject DID before delegating, and the same requirement applies then.
</div>

For every other entry class, the derivation may not be reachable before consent
because the collection may not exist until provisioning runs. A wallet is
therefore not required to reach the [=unsatisfiable=] verdict before delegation
begins; it is instead required to leave any resulting partial failure
repairable by re-running the exchange, per [[[#resumability]]]. A wallet
MUST NOT leave the application holding an inconsistent set of grants that only
manual intervention could correct.

The byte-level derivation, the resulting recipient identifier, and the roster
format are defined in [[WAS-EC]]. This profile states only the invariant and
the refusal.

### Provisioning {#provisioning}

When a satisfiable grant names a collection that does not yet exist, the
[=wallet=] MUST provision it before delegating.

Provisioning MUST be idempotent. Re-running an App Connect exchange, in
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
`#private-collection` MUST be encrypted from creation. There is no unencrypted
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
revoke, together ([[[#descriptor-shared-wallet-collection]]]) -- not an expiry.

### Consent {#consent}

A [=wallet=] MUST obtain the user's consent before minting or returning an
[=app-key credential=] and before delegating any grant.

This profile states a normative minimum for the consent surface:

1. The surface MUST present **exactly what resolution produced**. For each
   requested capability it MUST show the resolved target and the actions that
   would actually be granted after capping ([[[#allowed-actions]]]) -- not the
   actions that were requested.
2. An entry that resolved [=unsatisfiable=] MUST be shown as such. It MUST NOT
   be omitted, and MUST NOT be shown as if it would be granted.
3. A `#shared-wallet-collection` row MUST state that the grant is **read and
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

## Resource Log Profile {#resource-log-profile}

This section defines the [=resource log=], a hash-linked log format, derived
from the `did:webvh` log format [[DID-WEBVH]], and used for key resources
co-managed between a wallet's clients and the storage server (mainly collection
encryption descriptors and the key rosters associated with them, whose concrete
document schemas are defined in [[WAS-EC]]). It covers the entry format, entry
hashing, chain verification, the external-authorization rule, client-side head
pinning, and the terminal handover entry. It deliberately does not contain any
key management of its own: a resource log carries no keys, and every entry is
authorized against the log's [=controller document=] ([[[#log-authorization]]]).

### Log Model {#log-model}

A <dfn data-lt="resource logs">resource log</dfn> is an append-only sequence
of <dfn data-lt="log entry|log entries">log entries</dfn>, each carrying the
full state of one co-managed [=resource=] at that point in its history. Each
entry is hash-chained to its predecessor, and signed by the client that
appended it. The current state of the resource is the state of the verified head
entry; earlier entries exist so that any reader can verify how the resource got
there, entirely client-side, against a host that (for threat modeling purposes)
can be assumed adversarial.

Three parties appear in this profile:

* The <dfn data-lt="log controller document|controller documents">controller
  document</dfn> -- the DID document of the [=Space=]'s controller, resolved
  and verified by the reader independently of the storage server (for a
  `did:webvh` controller, by fetching and verifying its own hash-chained log).
  It is the log's root of authority; the set of keys that may authorize appends
  is defined there and only there.
* The **writers** -- [=enrolled clients=] (typically, other wallets), each
  holding a key listed in the controller document. Any number of writers may
  append; the profile assumes no coordination between them beyond the storage
  server's [Conditional Request](https://w3c-ccg.github.io/wallet-attached-storage-spec/#conditional-requests)
  compare-and-swap primitive (see [[[#log-append]]]).
* The **host** -- the storage server holding the log resource. Under [[WAS]]
  the host is trusted to enforce authorization on writes; this profile
  deliberately does not lean on that trust. For log verification the host is
  treated as a minimal store that linearizes concurrent appends and nothing
  more, and every guarantee below must hold even against a host that serves
  stale, forged, truncated, or forked logs.

The log's identifier is self-certifying (the [=SCID=] in its genesis
entry commits to the genesis content, so a host cannot substitute one log for
another under the same identity, see [[[#log-hashing]]]), while its
authority is externalized (no entry is valid on the log's own say-so;
every entry's signer must be found in the externally verified controller
document, see [[[#log-authorization]]]). The rationale for this split is given
in [[[#log-rationale]]].

### Relationship to `did:webvh` {#log-webvh}

<div class="note">
This subsection is non-normative.

The profile is an extraction from `did:webvh` [[DID-WEBVH]], and the
extraction is almost entirely subtractive:

* **Kept verbatim:** the five-member entry shape (`versionId`, `versionTime`,
  `parameters`, `state`, `proof`); JCS canonicalization; the
  SHA-256-multihash, `base58btc` entry hash; the SCID-style self-certifying
  genesis; the `eddsa-jcs-2022`-only proof rule; JSON Lines serialization.
* **Deleted:** the in-log key management (`updateKeys`, `nextKeyHashes`
  prerotation), witnesses, watchers, portability, deactivation, and the DID
  document typing of `state`.
* **Replaced:** the one verification step that consulted `updateKeys` -- the
  authorization predicate -- is replaced by the external rule of
  [[[#log-authorization]]].
* **Added:** the [=entry anchor=] ([[[#log-proof]]]), the [=terminal entry=]
  ([[[#log-handover]]]), and the [=chain-head pin=] semantics
  ([[[#log-pin]]]).

A did:webvh log answers to nobody -- it is a root of trust, so it must carry
its own key state. A resource log answers to the account that owns it, so
carrying its own key state would create a second root with no coherent
precedence rule; see [[[#log-rationale]]].
</div>

### The log resource {#log-resource}

A resource log is stored as a single [=resource=], serialized as JSON Lines
[[JSON-LINES]]: one [=log entry=] as one JSON object per line, in order, first
line first. Normatively, each line is a single JSON text [[RFC8259]]
serialized without embedded newlines, and lines are separated by U+000A LINE
FEED.

The log resource is the only serving of the resource it governs: this
profile defines no companion point-state document, and the governed
resource's current state exists only as the `state` of the verified head
entry ([[[#log-verification]]]).

The member name `history` is reserved in entry state: a [=log entry=]'s
`state` MUST NOT contain a `history` member.

### Entry format {#log-entry}

A [=log entry=] is a JSON object with exactly five members, all REQUIRED:

| Member        | Value                                                                                                                                                                                                                                             |
|---------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `versionId`   | `<n>-<entryHash>`: the entry's ordinal position `n` (decimal, starting at `1`), a single `-`, and the entry hash ([[[#log-hashing]]]).                                                                                                            |
| `versionTime` | The append time as an [[RFC3339]] UTC datetime (`Z` suffix). **Advisory only**; see below.                                                                                                                                                        |
| `parameters`  | A JSON object; see the per-entry rules below.                                                                                                                                                                                                     |
| `state`       | The full resource state at this version: a JSON object that MUST carry a `type` member (a string) identifying its schema. State schemas are defined by the referencing profile ([[WAS-EC]] for encryption descriptors and key rosters), not here. |
| `proof`       | A non-empty array of Data Integrity proofs ([[[#log-proof]]]).                                                                                                                                                                                    |

A verifier MUST reject an entry carrying members other than these five.

**`parameters` rules.** Unlike `did:webvh`, `parameters` do not evolve: format
changes travel through the handover mechanism ([[[#log-handover]]]), and not
through parameter mutation.

* The genesis entry's `parameters` MUST carry `method` (the format
  identifier, [[[#log-format-ids]]]) and `scid` (the [=SCID=]), and MAY carry
  `previousLog` (successor logs only, [[[#log-handover]]]). No other member
  is permitted.
* Every other entry's `parameters` MUST be `{}`, with one exception: a
  [=terminal entry=]'s `parameters` is exactly
  `{ "nextLog": { "method": ..., "scid": ... } }` ([[[#log-handover]]]).
* A verifier MUST reject an entry whose `parameters` carry any member this
  profile does not define for that entry position. This is deliberate
  fail-closed extensibility: the deleted `did:webvh` key-management
  parameters, in particular, MUST NOT be accepted, so a served log can never
  smuggle authorization semantics this profile removed.

**`versionTime` is advisory.** A writer MUST set it to its best knowledge of
the current time, and a verifier MUST NOT refuse an entry on temporal grounds
-- not for being out of order with respect to its neighbors, and not for being
in the future. Ordering authority rests entirely with the hash chain.

<div class="note">
This is a deliberate departure from `did:webvh`, which enforces strict
monotonicity plus a future-skew bound. Under that rule, one appender with a
fast clock produces an entry that later, correctly clocked appenders can never
legally follow -- a permanently unresolvable log. did:webvh needs enforceable
times for `versionTime`-based resolution; this profile offers no time-based
resolution, so it keeps the member for display and audit and moves all
ordering to the chain.
</div>

Example: a two-entry log (shown pretty-printed; on the wire each entry is
one line; hash, DID, and key values are illustrative):

```json
{
  "versionId": "1-QmUv7Yx3mvL2mLpp9pRgTvNqEcYtT7yiV1sfMx6xEBcCyD",
  "versionTime": "2026-08-06T17:00:00Z",
  "parameters": {
    "method": "resource-log:0.1",
    "scid": "QmXbWpS3fY9uNnQhJcM4kR8vT2eDgAqLx5oPiZsE6wHtUa"
  },
  "state": { "type": "WasEpochConfiguration", "...": "..." },
  "proof": [{ "...": "..." }]
}
{
  "versionId": "2-Qma8fN4kQjTr2vXwYhPzUdE9cLmB5oGsKiR7tD3eVpMnHx",
  "versionTime": "2026-08-07T09:30:00Z",
  "parameters": {},
  "state": { "type": "WasEpochConfiguration", "...": "..." },
  "proof": [{
    "type": "DataIntegrityProof",
    "cryptosuite": "eddsa-jcs-2022",
    "proofPurpose": "assertionMethod",
    "verificationMethod": "did:webvh:QmScid...:wallet-storage.example:space:81246131-69a4-45ab-9bff-9c946b59cf2e:id?versionId=6-QmDocVersion...#z6MkclientKey...",
    "proofValue": "z..."
  }]
}
```

### Entry hashing and the SCID {#log-hashing}

Canonicalization throughout this profile is JCS [[RFC8785]], over strict JSON
values (no `undefined`, no non-finite numbers).

**The hash serialization format.** Every hash this profile mints, including the
entry hash, the [=SCID=], and the `scid` values carried in `nextLog` and
`previousLog`, is serialized the same way: the SHA-256 digest of the
canonicalized input, wrapped as a multihash (the bytes `0x12`, `0x20`, then the
32 digest bytes), encoded with the `base58btc` alphabet, with **no** multibase
prefix. This is byte-for-byte the `did:webvh` entry-hash format. It is
deliberately NOT the `z`-prefixed multibase serialization, and NOT the
`base64url`serialization used for content addressing elsewhere in [[WAS]];
one format, chosen once, so a verifier never has to guess.

**The entry hash.** The `entryHash` of entry `n` is the hash, as above, of the
entry with its `proof` member removed and its `versionId` member replaced by
the **predecessor's** `versionId` -- for the genesis entry, by the [=SCID=].
The entry's own `versionId` is then `<n>-<entryHash>`. This is what makes the
chain a chain: each entry's identifier is a commitment to its predecessor's
identifier and transitively to the entire log back to genesis.

**The genesis entry and the <dfn>SCID</dfn>** (self-certifying identifier).
The genesis entry is built in two passes:

1. Build the preliminary entry with the literal placeholder string `{SCID}`
   as the value of both `versionId` and `parameters.scid` (and anywhere else
   the SCID will appear). The SCID is the hash, as above, of this preliminary
   entry.
2. Replace every `{SCID}` placeholder with the SCID, then compute the entry
   hash normally (`versionId` input value: the SCID). The genesis
   `versionId` is `1-<entryHash>`.

A verifier MUST recompute the SCID by the inverse procedure (substitute the
log's `parameters.scid` value back to `{SCID}`, hash, compare) and MUST
reject a log whose SCID does not verify.

Because `parameters.method` is inside the hashed genesis content, the SCID
commits to the format identifier: a host cannot downgrade a log's format
without changing its identity. And because every later entry hash chains to
the genesis through the `versionId` substitution rule, every entry hash in the
system is transitively bound to that identifier. Together with the fixed
five-member input shape, this is this profile's domain separation: no entry hash
can collide with a hash minted over any other structure.

### The entry proof and its anchor {#log-proof}

Each element of an entry's `proof` array MUST be a Data Integrity proof
[[VC-DATA-INTEGRITY]] with:

* `type`: `DataIntegrityProof`
* `cryptosuite`: `eddsa-jcs-2022` [[VC-DI-EDDSA]] -- no other cryptosuite is
  permitted
* `proofPurpose`: `assertionMethod`
* `verificationMethod`: a DID URL identifying the signing key in the
  [=controller document=], carrying the [=entry anchor=] described below
* `proofValue`: the proof value per [[VC-DI-EDDSA]]

The proof input is the complete entry, including its final `versionId`,
with the `proof` member absent. Because the `versionId` is a commitment to
the whole chain ([[[#log-hashing]]]), the signature covers the chain link;
an entry cannot be re-parented without breaking its proof.

A verifier MUST verify every proof in the array. One failing proof rejects
the entry.

**The <dfn data-lt="entry anchors|anchored version">entry anchor</dfn>.**
Where the controller's DID method provides versioned document resolution (as
`did:webvh` does), the proof's `verificationMethod` MUST carry a `versionId`
DID parameter [[DID-CORE]] naming the controller-document version the entry
was authorized under, e.g.
`did:webvh:...:id?versionId=6-Qm...#z6Mk...`. The anchor is inside the
signed proof options, so it cannot be altered without breaking the proof.
Where the controller's document is unversioned (a static DID such as
`did:key`), the anchor is omitted and every anchor rule below reads "the
controller document".

### Authorization {#log-authorization}

An entry is authorized by exactly one rule:

> Every proof's signing key MUST be listed as a verification method under the
> `assertionMethod` relation of the [=controller document=] **at the
> entry's anchored version**, where the controller document is resolved and
> verified by the reader independently of the host serving the log.

The rule's parts, unpacked, each normative:

1. **The document comes from the reader's own verification, not from the
   channel.** The verifier MUST resolve the controller document through its
   own verified resolution pipeline, cryptographically verifying every step:
   for a `did:webvh` controller, it fetches the controller log itself,
   verifies it end-to-end (SCID, entry hash chain, proofs, and its own head
   pin, where it keeps one), and answers the anchored-version lookup from
   that verified history. A verifier MUST NOT obtain the signing key by
   dereferencing the proof's `verificationMethod` URL as an ordinary
   resource fetch, and MUST NOT accept controller-document material supplied
   by the host outside that independently verified resolution -- the host
   serving the resource log is the same party a doctored controller document
   would come from. (In implementation terms: the document loader handed to
   the proof-verification library answers DID URL lookups from the verified
   controller log, not from the wire.)
2. **The anchored version must be present.** The anchor MUST identify a version
   present in the reader's verified controller log, at or before its head. An
   anchor naming an unknown version rejects the entry.
3. **Anchors are monotone along the log.** Each entry's anchored version MUST
   be the same as, or a descendant of (for a linear controller log: at or
   after), the previous entry's anchored version. An entry anchored behind
   its predecessor rejects the log.
4. **The relation is `assertionMethod`.** The log format's fixed proof shape
   ([[DID-WEBVH]]) sets `proofPurpose: assertionMethod`, and Data Integrity
   verification [[VC-DATA-INTEGRITY]] already requires a proof's verification
   method to be authorized under the controller-document relation matching
   the proof's purpose. The authorization rule uses that same relation, so
   proof verification and authorization are one membership check, enforceable
   with standard proof-verification tooling -- no second relation a signing
   key must be dual-listed under, and no custom verifier overriding the
   purpose-relation match. The flip side is that listing a key under
   `assertionMethod` is precisely what entitles it to append: a controller
   whose logs are governed by this profile MUST NOT list a key under
   `assertionMethod` unless that key is entitled to co-manage the account's
   governed resources. Keys listed under other relations only (a
   `keyAgreement`-only recovery key, an `authentication`-only login key)
   remain structurally excluded.

**Writers anchor at their verified head.** A writer appending an entry MUST
anchor it at the head of the controller document as the writer last verified
it. Together with anchor monotonicity this yields the profile's historical
verification: entries verify **as-of-append**. A client revoked from the
controller document keeps its past entries verifiable (they anchor at
versions where its key was present) while losing the ability to have new
entries accepted under post-revocation anchors.

**The sealing append.** After a controller-document change that removes a
verification method, the controller's clients MUST ensure that each governed
resource log receives at least one subsequent entry anchored at or after the
new document version. Anchor monotonicity then makes the removal effective
for that log: no later entry can anchor behind the seal, so the removed key
can never validly append again. Until a log's sealing append operation lands,
the removed key can still extend that log under a pre-removal anchor -- a
residual window with the same shape, detection, and repair story as any other
torn multi-resource ceremony. The staleness is visible in durable state (a
log whose head anchor predates the membership change), and re-running the
sealing pass will converge on the same state.

<div class="note">
In the wallet ceremonies this profile serves, the sealing append is not an
extra write: revoking a client already rotates every encrypted collection to
a new key epoch, and each rotation is itself a state change appended to the
governed log, anchored at the post-revocation document version.
</div>

### Chain verification {#log-verification}

A verifier MUST run the following checks over the full log, in order, and
MUST treat any failure as rejecting the log (not just the failing entry).
A verifier MUST NOT accept any stated or served head, digest, or count in
place of recomputing from the entries themselves.

1. **Parse.** Parse the resource as JSON Lines ([[[#log-resource]]]).
   Reject on any non-object line, on any entry violating [[[#log-entry]]]
   (member set, `parameters` rules, `state.type`), and on any `versionId` whose
   ordinal is not the entry's 1-based position.
2. **Genesis.** Recompute and check the [=SCID=] ([[[#log-hashing]]]).
   Check that `parameters.method` names a format this verifier implements
   ([[[#log-format-ids]]]); where a [=chain-head pin=] or the referencing
   profile supplied an expected method, check that it matches.
3. **Chain.** For each entry, recompute the entry hash from the
   predecessor-substituted input and check it against the `versionId`.
4. **Proofs.** For each entry, verify every proof per [[[#log-proof]]].
5. **Authorization.** For each entry, apply [[[#log-authorization]]]:
   resolve the anchor against the independently verified controller
   document, check `assertionMethod` membership of every signing key,
   and check anchor monotonicity.
6. **Termination.** If any entry is a [=terminal entry=], check that it is
   the last entry; reject a log with entries after a terminal entry, and
   refuse to append past one ([[[#log-handover]]]).
7. **Continuity.** Compare the verified head against the [=chain-head pin=],
   where one is held ([[[#log-pin]]]).

The resource's current state is the verified head entry's `state`.

<div class="note">
Full verification is a wallet-to-wallet concern: it is run by [=enrolled
clients=] co-managing governed resources against a host they do not trust for
them. An [=application=] connecting under this profile never runs it -- the
application verifies the response presentation ([[[#response-verification]]])
and invokes its capabilities; it does not read these logs. In wallet
workflows, verification runs:

* before any append -- a writer extends only a head it has verified
  ([[[#log-append]]]), so every state-changing ceremony (a key-epoch
  rotation, the sealing append after a client revocation, a handover) begins
  with a verification pass;
* before acting on current state -- a client coming back online, or a second
  enrolled wallet syncing, verifies before trusting a served roster or epoch,
  since current state is defined only as the verified head's `state`;
* at first contact -- a newly enrolled client bootstrapping onto the
  account's governed resources verifies to establish its [=chain-head pin=]
  ([[[#log-pin]]]);
* on return visits -- re-verification against the held pin is where host
  rollback, truncation, and forks are detected (step 7).
</div>

### Appending {#log-append}

Appending is linearized by the host's compare-and-swap (CAS) primitive
(conditional writes on the log resource's entity tag, per [[WAS]]); the chain
itself carries no consensus mechanism, and none is needed: concurrent writers
produce a CAS conflict as opposed to a fork.

Conditional writes are an optional feature in [[WAS]]. A host serving
resource logs MUST support them: the backend holding the log MUST advertise
the `conditional-writes` feature, and a writer MUST NOT fall back to an
unconditional write against a backend that does not. Without the
precondition, concurrent appends silently overwrite one another instead of
failing into the retry loop below.

A writer MUST:

1. Read the full log and verify it per [[[#log-verification]]] -- an entry
   is never built on an unverified head.
2. Build the new entry against the verified head (full state, hash, anchor,
   proof) and write the extended log conditionally on the entity tag of the
   read from step 1.
3. On a conditional-write conflict: re-read, re-verify, rebase the change on
   the new head, and retry.
4. **Confirm by reading back.** A write acknowledgement is a promise, not a
   fact: the writer MUST read the log back and verify that the extended,
   verified log contains its entry before treating the append -- or any
   ceremony step gated on it -- as durable.

### Head pinning {#log-pin}

A reader that returns to a log across sessions MUST keep a <dfn
data-lt="chain-head pins">chain-head pin</dfn> per log: the log's format
identifier (`parameters.method`), its [=SCID=], and the `versionId` of the
latest verified head. The pin is client-local durable state; where it is
stored and how it is protected are left up to client implementers.

* The pin is established at **first contact**: the first full verification of
  a log the reader has no pin for. First contact is trust-on-first-use, and
  what it establishes is the log's identity (the SCID); see
  [[[#security-resource-log]]] for the limits and implications.
* The pin is updated only after a full verification ([[[#log-verification]]])
  of a log whose head is the pinned head or a descendant of it, or after a
  verified handover ([[[#log-handover]]]), which replaces the pin with the
  successor's (method, SCID, head).
* A served log whose SCID or method differs from the pin, outside a verified
  handover, MUST be refused as a continuity break.
* A served log whose head is behind the pin (the pinned head's ordinal
  exceeds the served head's, with the served log an ancestor-prefix of what
  was pinned) is reconcilable divergence -- possibly replication lag. The
  reader MUST NOT adopt it and MUST NOT regress its pin; it MAY retry.
* A served log that has **forked** -- it shares a prefix with the pinned
  history but diverges at some version, so neither is an ancestor of the
  other -- MUST be refused, and the reader SHOULD retain both the served log
  and its pinned evidence: every entry is signed, so a conflicting pair of
  logs under one SCID is transferable, independently verifiable evidence of
  equivocation.

A pin record MAY carry an extensible set of attached proofs (for example, a
host-signed checkpoint, or witness co-signatures adopted later as policy),
under the rule that a reader ignores proofs it cannot attribute. This lets
stronger continuity evidence be layered on without a format change. A reader
MAY additionally render the pinned head as a short human-comparable
fingerprint for out-of-band comparison between clients.

### The terminal handover entry {#log-handover}

One mechanism serves both log compaction (truncating a long history) and
format migration (moving to a successor format): a signed, in-chain
<dfn data-lt="terminal entries">terminal entry</dfn> that closes the log and
names its successor.

* A terminal entry is an ordinary entry ([[[#log-entry]]], [[[#log-proof]]],
  [[[#log-authorization]]] all apply) whose `parameters` is exactly
  `{ "nextLog": { "method": ..., "scid": ... } }`: the successor log's format
  identifier and its [=SCID=]. Its `state` MUST equal its predecessor's
  `state` -- a handover changes no resource state.
* The successor log's genesis `parameters` additionally carries
  `previousLog: { "scid": ..., "head": ... }`: the prior log's SCID and the
  `versionId` of the prior log's last regular entry -- the terminal entry's
  predecessor. (The reference cannot name the terminal entry itself: the
  terminal entry commits to the successor's SCID, so the successor's genesis
  must exist first.)
* For a compaction, the successor's genesis `state` is the full current
  state, so the successor log stands alone; the prior log may then be
  retained or discarded as evidence policy dictates.

A verifier crossing a handover MUST check the link from both sides: the
terminal entry's `nextLog.scid` equals the successor's SCID and the methods
are consistent with what each log's own genesis declares; the successor's
`previousLog.scid` equals the prior log's SCID; and the successor's
`previousLog.head` equals the `versionId` of the terminal entry's immediate
predecessor -- that is, the terminal entry chains directly off the head the
successor references. A successor served without this verifiable link MUST be
refused as a continuity break.

Every verifier of this profile MUST recognize terminal entries and MUST
refuse to append past one -- even though nothing currently emits them. This
is what makes the mechanism a safe migration path: a v1 verifier meeting a
handed-over log refuses to extend the frozen log, rather than continuing a
history its author has closed.

### Format identifiers {#log-format-ids}

The format identifier for this profile is the byte-significant string
`resource-log:0.1`. The identifier is compared only for byte equality
and never parsed: the `0.1` is part of the opaque identifier, not an
orderable version number. A future revision of this profile is a different
identifier reached only through the handover mechanism
([[[#log-handover]]]), not a "greater version" of this one.

The identifier appears at two seams plus the pin, with a strict
authority ordering:

1. **Authoritative:** `parameters.method` in the genesis entry. It is
   SCID-committed and proof-covered ([[[#log-hashing]]]), which makes it the
   only downgrade-safe location: a host cannot alter it without changing the
   log's identity and breaking its genesis proof.
2. **Payload schema:** the `state` document's own `type` member
   ([[[#log-entry]]]) -- the resource schema, versioned by the referencing
   profile independently of the log format.
3. **The pin:** the [=chain-head pin=] stores the method beside the SCID and
   head, so a served format switch outside the handover mechanism
   ([[[#log-handover]]]) is refused as a continuity break rather than
   dispatched to a different verifier.

### Design rationale: externalized log authorization {#log-rationale}

<div class="note">
This subsection is non-normative.

The obvious challenge to this design: self-sovereignty is the prestige
property of hash-linked logs (it is what makes `did:webvh` and its
relatives valuable), so why is a resource log deliberately NOT its own
root of trust, carrying its own key set the way an identity log does?

Because self-sovereignty is the right design exactly when a log's writers
share no pre-existing root of authority. A resource log is the opposite case:
its writer set is *defined as* the account's enrolled clients, which already
have a self-sovereign home in the controller document. Making each log its
own root would not add sovereignty; it would copy the client roster into N
places and then have to keep the copies consistent.

**Revocation atomicity is what in-log key management would break.** Under the
external rule, revoking a client is one controller-document edit, and that
single edit is the revoked client's pull axis everywhere: every delegation,
every invocation, and every log-append right dies against the same document.
If each log carried its own `updateKeys`, revocation would become a rotation
ceremony per log, and a crash mid-ceremony would leave the revoked client
durably authorized on the remainder -- a drift-detection problem created
purely by the duplication. (The sealing append of [[[#log-authorization]]]
narrows per-log windows too, but its failure mode is detectable staleness
that a re-run repairs -- not standing authorization that must be hunted
down.)

**Two roots of trust have no coherent precedence rule.** If a log's in-log
key set and the controller document could disagree, one of them wins. If the
document wins, the in-log keys are dead weight -- maintenance surface with no
authority. If the in-log keys win, the document's revocation didn't revoke,
and a compromised client can entrench itself in whichever logs it can still
rotate. Every answer makes in-log key management either vestigial or a hole.
A KEL never faces this because it answers to nobody; a log subordinate to an
account cannot be half-sovereign.

**The genuine benefit of full self-certification has no audience here.** A
standalone-verifiable log matters when third parties consume it without
account context -- which is why the identity log is self-certifying. An
encryption descriptor or key roster is meaningless outside its account, and
its only readers already fetch and verify the controller document for other
reasons, so "verify the controller first" adds zero marginal cost. The cheap
half of self-certification is kept anyway: the SCID genesis makes each log's
*identity* self-certifying, so a host cannot swap logs under an id. That
split -- self-certifying identity, externalized authority -- is the design.

The rule generalizes before it breaks: if a future resource's writers span
several accounts, the authorization predicate becomes "the signer is in any
of the named controllers' verified documents" -- still external. Only a log
whose membership must evolve with no controlling document anywhere truly
needs to be its own root, and no such log exists in this system.
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

**Below the origin, `appUrl` is namespacing, not isolation.** The browser
attests nothing finer than the origin, so the `appUrl` that scopes an identity
within an origin ([[[#appconnectquery]]]) is self-asserted: applications
served from one origin can claim one another's `appUrl` and are not protected
from each other by this profile. That is the browser's own boundary, not a new
one -- same-origin applications already share storage and can already script
each other -- and deploying mutually untrusting applications on one origin is
outside this profile's threat model as it is outside the browser's.

### Seed confidentiality {#security-seed}

The [=seed=] plays the role of the application's client secret, custodied by
the wallet because a [=public client=] cannot hold one of its own: everything
the application can sign and everything it can decrypt derives from it.

In transit it exists only inside the response presentation -- for an
in-browser application, the [=CHAPI=] response, a browser-mediated channel
between the wallet and the requesting origin; over another transport, that
transport's response ([[[#request-transport]]]). It is never sent to the
storage server, never appears in a capability, and never appears in a URL.

At rest it lives in two places: in the user's wallet, as an ordinary stored
credential subject to whatever protection the wallet applies to the user's
credentials; and application-side, wherever the application persists it. An
application SHOULD treat the seed as it would any long-lived client secret, and
SHOULD scope its storage to its own origin.

Nothing downstream of the wallet's match consumes the seed: the wallet reads it
only to re-derive the subject DID for the binding check, so the seed never
reaches the grant path.

### The planted credential {#security-planted-credential}

The `AppKeyCredential` type asserts nothing that an attacker cannot also
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
the victim application's `appUrl` and the victim's [=origin=] produces a
credential that binds. There is nothing wrong with it: it is a genuine app-key
credential for the attacker's identity, and no amount of inspection
distinguishes it from the user's own, because the two differ only in which seed
was rolled.

The attack that follows is worth stating in full, because it is what the
provenance rule exists for. The attacker gets the user to import such a
credential -- from a link, a QR code, a file, or a restored backup -- and gives
it an `issuanceDate` in the future. The wallet stores it (every check passes).
On the next connect the wallet's match predicate ([[[#app-key-matching]]])
accepts it: correct type, correct `appUrl`, self-issued, correct origin,
binds.
The latest-first ranking picks it over the user's real credential on the strength
of its future date. The wallet then delegates the user's storage to the
attacker's [=connecting DID=], and hands the attacker's seed back to the
application as if it were the user's own.

**The refusal that closes it is a channel rule, not a content rule.** An
app-key credential has no legitimate reason to arrive from outside: it is
wallet-minted, in an [=App Connect exchange=], and never imported. A wallet that
refuses every `AppKeyCredential`-typed credential arriving from outside
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

Three rules in this profile refuse rather than degrade: an unrecognized
descriptor type ([[[#unknown-descriptor-type]]]), an unrecognized query type
([[[#wallet-unsupported]]]), and a query of any type other than
`DIDAuthentication` co-occurring with an `AppConnectQuery`
([[[#request-exclusivity]]]). A fourth narrows rather than refuses: an action
token outside the closed vocabulary is dropped ([[[#action-vocabulary]]]) --
which can never widen a grant, and becomes a visible refusal at the point where
it would matter, since an entry left with no actions after capping is
[=unsatisfiable=] rather than unrestricted ([[[#allowed-actions]]]).

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

### Resource log trust bounds {#security-resource-log}

The [=resource log=] profile ([[[#resource-log-profile]]]) is designed against
a host that serves stale, forged, truncated, or forked logs, and its
guarantees come entirely from client-side recomputation: the chain from the
[=SCID=] forward, every proof, and every authorization check against the
independently verified [=controller document=]. Nothing served -- a stated
head, a digest, a count -- is accepted without the log confirming it.

Two attacks remain outside the model, and stating them is part of the design:

**First-contact substitution.** A reader with no [=chain-head pin=] for a log
accepts whichever verifying log the host serves first. The SCID prevents
substitution *under a known identity*, and the authorization rule means any
substitute must still be signed entirely by the account's own enrolled keys --
so what first contact actually risks is being shown a stale or truncated
history that genuine writers produced, not a forged one. From the pin onward,
rollback and forks are refused.

**Per-client equivocation.** A host can serve different (individually valid)
extensions of one log to different clients that have not compared pins. Each
client's own pin keeps its own view consistent; the gap is cross-client. The
mitigations are layered rather than structural: any two clients that ever
compare heads (or exchange logs) hold transferable, independently verifiable
evidence of the equivocation, since every entry is signed and both logs claim
one SCID ([[[#log-pin]]]); and the pin's extensible proof set leaves room for
witness cosignatures as a policy upgrade without a format change. Writers are
better protected than pure readers: the append procedure's read-back
confirmation ([[[#log-append]]]) means a host that hides one writer's entry
from another forces a visible CAS conflict or a visible missing entry, not a
silent split.

**The revocation window.** Between a membership-removing controller-document
edit and a given log's sealing append ([[[#log-authorization]]]), the removed
key can still extend that log under a pre-removal anchor. The window is the
same one any multi-resource revocation cascade has; what the profile adds is
that the window's state is durably visible (a head anchor predating the
membership change) and closes idempotently by re-running the seal.

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

This property holds by construction rather than by normative requirement: the
wallet names the controller itself, deriving it from a per-user seed
([[[#delegation-target]]]).

Some applications connect through a channel outside this profile and name
their own `controller` (for example, via a standalone
`AuthorizationCapabilityQuery` [[VCALM]]). Such an application SHOULD NOT use
the same grantee DID across users. Reusing one DID recreates both exposures
above: the storage provider can link every user who granted to it, and one
compromised key opens every user's read axis at once. A wallet only ever sees
one user's view, so it cannot detect the reuse. This is an expectation of
application authors, not an enforceable rule. It is a SHOULD because a static
grantee DID remains a defensible choice for some
application shapes (a single-tenant backend service, for example).

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
