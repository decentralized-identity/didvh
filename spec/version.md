## `did:vh` Version Changelog

The following lists the substantive changes in each version of the
specification.

- Version 0.1 (Editors Draft)
  - Initial draft, derived from the `did:webvh` Editors Draft (beyond v1.0)
    with the domain and path removed from the DID: a `did:vh` DID is
    `did:vh:<SCID>`.
  - Defines resolution as the DID Resolution `resolve(did,
    resolutionOptions)` function, with `did:vh` resolution options for the two
    ways of passing the log and witness proofs file: directly (`didLog`,
    `didWitness`) or by reference (`srcLog`, `srcWitness`, each naming a
    file at an `https` web location, a location using another supported URL
    scheme such as `ipns`, or, for a local [[ref: Resolver]], a loopback
    `http` location or a local file), plus the version options
    (`versionId`, `versionTime`, `versionNumber`).
  - Defines the `src` DID URL query parameter, aligned with `did:scid`,
    naming the location of `did.jsonl` and `did-witness.json`, from which a
    dereferencer derives `srcLog` and `srcWitness` by appending the file
    names. No `/.well-known/` location is used.
  - Defines the result of processing a log in which an entry fails
    verification, and uses the `featureNotSupported` error for a `method`
    version the [[ref: Resolver]] does not support.
  - States that no entry can follow one that sets `updateKeys` to `[]`, and
    that [[ref: pre-rotation]] is active for an entry only if the
    `nextKeyHashes` in effect before it is non-empty.
  - Returns `ttl` and the `witness` `threshold` as integers in the DID
    Document Metadata, as now permitted by DID Resolution.
  - Makes retaining a `versionId` from an earlier resolution optional for a
    [[ref: Resolver]], with a warning (not an error) for a copy shorter than
    the retained version.
  - Defines the `did:vh` [[ref: watcher]] HTTP API in its own OpenAPI
    definition, derived from that of `did:webvh`.
  - Adds the `includeDidLog` option, returning the verified log and witness
    proofs file in the resolution metadata (`didLog`, `didWitness`).
  - Adds optional comparison of the log with copies held by
    [[ref: watchers]] (`checkWatchers`, `extraWatchers`, `minCopies`), with
    the `logForked` and `insufficientCopies` errors and `copies` and
    `warnings` resolution metadata.
  - Describes, non-normatively, how a client sources the [[ref: DID Log]]
    (the log itself or a reference to it) before resolution — known in
    context, from [[ref: watchers]], peer-to-peer exchange (with updates sent
    as the complete log or just the new entries), and from a web location —
    and how to manage freshness and duplicity.
  - Removes the `did:webvh` features that depend on a web location: the
    DID-to-HTTPS transformation and domain-name rules, the `portable`
    parameter, and parallel `did:web` publishing.
  - Requires `versionTime` values in whole seconds; makes an entry that sets
    `deactivated` to `true` the last entry of the DID; requires
    [[ref: DID Controllers]] to distribute only logs that extend the log they
    previously distributed, and [[ref: witnesses]] to approve at most one
    entry with the same predecessor.
  - Makes the Security and Privacy Considerations sections non-normative,
    moving the transport requirements into [Publishing and Retrieving DID Log
    Resources](#publishing-and-retrieving-did-log-resources), adds a [Forked
    Logs](#forked-logs) section, and drops the Resolver Validation
    Checklist.
