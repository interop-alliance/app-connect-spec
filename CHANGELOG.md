# App Connect for Portable Web Spaces Changelog

## 0.2.0 - TBD

### Added
- The `https://w3id.org/byoe#plaintext-collection` descriptor type: a
  plaintext, capability-gated collection with no world-readable access policy.
  It has a registry row, its own subsection, a row in the required-actions
  table, and a coverage entry in grant validation. Naming an existing encrypted
  or public collection resolves unsatisfiable.
- A consent rule for `#plaintext-collection` rows: the storage server's
  operator can read the contents, and the public cannot.
- A Privacy Considerations paragraph on what the storage server reads in a
  plaintext collection.
- The VPR-root `agent` member (`agent: { name }`), the requester's
  self-declared display name, with its validation rules, its relation to an
  `AppConnectQuery`, and how the consent surface shows it.
- A `requester` term: an app, service, or agent that asks for grants and
  receives them.

### Changed
- "Encrypted by default" now covers `#private-collection` and states the
  plaintext exceptions.
- Provisioning states the form each collection descriptor type provisions,
  and that the epoch rules apply to encrypted collections only.
- Text about the requesting party in general says "requester". Text about the
  App Connect exchange itself (the app-key credential, the attested origin,
  `appUrl`, the CHAPI popup, the response presentation) keeps "application".
  No normative behavior changed.
