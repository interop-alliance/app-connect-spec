# App Connect for Wallet Attached Storage

> App Connect is a CHAPI/VCALM profile by which a web application connects to a
> Wallet Attached Storage (WAS)-backed wallet in a single exchange -- one
> request, one consent screen, one signed response. The wallet returns a
> per-user application identity (the app-key credential) together with
> delegated storage capabilities, so an application can read and write
> user-owned storage without operating a backend of its own. The profile
> defines the `AppConnectQuery` request, the app-key credential and its
> seed-to-subject binding rule, the invocation-target descriptor vocabulary
> with per-class allowed actions, the response presentation, and the
> fail-closed processing rules.

This repository contains the App Connect specification, in
[ReSpec Markdown](https://respec.org/docs/#markdown) format: `index.html` is
the ReSpec shell (title, abstract, references), and the specification body
lives in [`spec.md`](spec.md).

App Connect is a companion to the
[Wallet Attached Storage specification](https://github.com/w3c-ccg/wallet-attached-storage-spec)
and to a forthcoming WAS Encrypted Collections profile (which defines the
envelope cryptography and recipient-key derivation this profile references).

## Status

Experimental draft, undergoing regular revisions. Intended for incubation as a
W3C Credentials Community Group work item alongside the WAS specification.

## Editing

Everything content-related goes into `spec.md`; `index.html` carries the
ReSpec configuration and the bibliography for references not in SpecRef. To
preview locally, serve the repository root (for example
`npx serve .`) and open `index.html` in a browser.

## License

See [LICENSE.md](LICENSE.md).
