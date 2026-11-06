## Overview

`did:webvh` added a verifiable history to `did:web`: a
[[ref: self-certifying identifier]] (SCID), a hash-chained log of every version
of the [[ref: DIDDoc]], signed updates, pre-rotation, [[ref: witnesses]] and
[[ref: watchers]]. It kept the domain name from `did:web` in the DID, so that
a [[ref: Resolver]] can turn the DID into an HTTPS URL and fetch the
[[ref: DID Log]].

The domain name does two jobs in `did:webvh`: it tells a [[ref: Resolver]]
where to find the log, and it makes the DID a web identifier. It is not needed
for security — the [[ref: SCID]] and the verifiable history provide that on
their own. But it does tie the DID to one location. When that location
changes, the DID changes, which `did:webvh` handles with its optional
portability mechanism: the [[ref: SCID]] stays the same and the
[[ref: DID Log]] carries the DID's history across the move, but the DID is no
longer the same string.

Many uses of DIDs need an identifier that never changes: as a permanent
account identifier, embedded in content-addressed data, or held in long-lived
credentials. Others have no web location at all: DIDs exchanged between two
parties, DIDs kept in an organisation's database, or DIDs anchored on a
ledger. `did:vh` serves those uses by removing the location from the DID:

```text
did:webvh:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ:example.com:dids:issuer
did:vh:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ
```

Everything else is kept as close to `did:webvh` as makes sense. The
[[ref: DID Log]] format, [[ref: parameters]], [[ref: SCID]] and entry-hash
algorithms, key authorization and pre-rotation, [[ref: witnesses]],
[[ref: watchers]], and the version-selection resolution options are those of
`did:webvh`. Implementations of `did:webvh` can support `did:vh`
by replacing the DID-to-HTTPS transformation with the [resolution
options](#didvh-resolution-options) defined here and dropping the
web-location-specific features.

The following is a `tl;dr` summary of how `did:vh` works:

1. A [[ref: DID Controller]] creates the first [[ref: DID log entry]] exactly
   as in `did:webvh`, using the DID string `did:vh:{SCID}` with the `{SCID}`
   placeholder, calculates the [[ref: SCID]] and replaces the placeholders.
2. The [[ref: DID Log]] (`did.jsonl`) is a [[ref: JSON Lines]] file of
   [[ref: DID log entries]], one per line. Each entry is a JSON object
   containing:
    1. `versionId` -- a value that combines the version number (starting at
       `1` and incremented by one per version), a literal dash `-`, and a hash
       of the entry. The [[ref: entry hash]] calculation links each entry to
       its predecessor in a ledger-like chain.
    2. `versionTime` -- as asserted by the [[ref: DID Controller]].
    3. `parameters` -- a set of [[ref: parameters]] that impact the processing
       of the current and future [[ref: log entries]], such as the version of
       the `did:vh` specification being used, the [[ref: SCID]] and the update
       key(s).
    4. `state` -- the new version of the [[ref: DIDDoc]].
    5. `proof` -- a [[ref: Data Integrity]] proof across the entry, signed by a
       [[ref: DID Controller]]-authorized key.

   If the [[ref: DID Controller]] uses [[ref: witnesses]], a separate file
   (`did-witness.json`) holds the witnesses' [[ref: Data Integrity]] proofs
   for [[ref: DID Log entries]].
3. The [[ref: DID Controller]] makes the [[ref: DID Log]] available wherever it
   is needed: to the other party in a relationship, to one or more
   [[ref: watchers]], at one or more web locations, or in any other store.
   There is no single authoritative location.
4. A client that needs to resolve `did:vh:<SCID>` first
   [sources](#sourcing-the-did-log) the [[ref: DID Log]]: it gets the log
   itself from context, a [[ref: watcher]] or a peer, or a reference to it,
   such as the web location given in
   `did:vh:<SCID>?src=https://example.com/dids/issuer`.
5. The client calls the [[ref: Resolver]], [passing](#didvh-resolution-options)
   the [[ref: DID Log]] (and witness proofs) directly, or by reference in the
   `srcLog` (and `srcWitness`) options. The [[ref: Resolver]] verifies every entry and returns the
   [[ref: DIDDoc]]. The [[ref: SCID]] check ensures the log is the history of
   that DID, regardless of where it came from. Optionally, the client asks the
   [[ref: Resolver]] to compare the log with copies held by
   [[ref: watchers]].
6. Updates append entries to the log, which the [[ref: DID Controller]]
   re-distributes to the places the log is kept.

Because a `did:vh` [[ref: DID Log]] has no single authoritative location, a
[[ref: Resolver]] cannot always be sure it holds the *latest* version of the
log, and a [[ref: DID Controller]] could give different parties different
extensions of the same log. [[ref: Watchers]] and [[ref: witnesses]] are the
tools `did:vh` uses to manage those risks; see [Log Freshness and
Duplicity](#log-freshness-and-duplicity).

### Relationship to `did:scid`

The proposed
[`did:scid`](https://lf-toip.atlassian.net/wiki/spaces/HOME/pages/88572360/DID+SCID+Method+Specification)
metamethod defines a family of location-independent DIDs, each using the
verification data format of an existing SCID-based DID method. Its
`did:scid:vh` format uses the `did:webvh` format, and so is comparable to
`did:vh`:

```text
did:scid:vh:1:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ
did:vh:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ
```

`did:vh` shares the design goals of `did:scid` — a permanent, portable,
self-certifying DID whose history can be kept in any number of locations — and
adopts its `src` DID URL query parameter for naming a location that holds
the log. The differences are:

- `did:vh` is a standalone DID method rather than a format of a metamethod.
- The version of the specification in use is carried in the `method`
  [[ref: parameter]] of the [[ref: DID Log]], as in `did:webvh`, rather than in
  the DID string.
- In this version of `did:vh`, the `src` parameter carries only a URL: an
  `https` URL or a URL with another scheme the [[ref: Resolver]] supports
  (such as `ipns`). `did:scid` also allows it to name a DID method that
  stores the log.
- `src` is not a `did:vh` resolution option. A client passing the log to a
  [[ref: Resolver]] by reference names each file explicitly, in the `srcLog`
  and `srcWitness` options, which a dereferencer derives from a `src` DID URL
  parameter. These options can also name loopback `http` URLs and local
  `file:` URLs for a local [[ref: Resolver]].

The `did:scid:vh` format is currently defined on `did:webvh`, whose DIDs carry
a web location that `did:scid` does not need. As `did:vh` is the
location-independent form of the same verifiable history, we expect the
definition of `did:scid:vh` to evolve to use `did:vh` instead of `did:webvh`.
See the [note on the initial DIDDoc](#note-did-scid-vh-diddoc-id) for how
`did:vh` leaves room for that.

Because the DID string is part of the first [[ref: DID log entry]], and so of
the input to the [[ref: SCID]], a `did:vh`, a `did:scid:vh` and a `did:webvh`
DID can never share a [[ref: DID Log]] or an [[ref: SCID]]. A
[[ref: DID Controller]] can link such DIDs using `alsoKnownAs`.
