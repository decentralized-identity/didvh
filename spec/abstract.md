## Abstract

DID + Verifiable History (`did:vh`) is a [[ref: DID Method]] that gives a
[[ref: Decentralized Identifier]] (DID) a cryptographically verifiable history
without tying it to any location. It is
the location-independent sibling of
[`did:webvh`](https://identity.foundation/didwebvh/): a `did:vh` DID is simply

```text
did:vh:<SCID>
```

where `<SCID>` is the [[ref: self-certifying identifier]] derived from the
DID's first [[ref: DID log entry]]. Unlike `did:webvh`, a `did:vh` DID
contains no domain name or path, and so is not bound to any location where its
[[ref: DID Log]] is published.

Because the DID carries no location, sourcing the [[ref: DID Log]] is
separated from resolving the DID. A `did:vh` [[ref: Resolver]] implements the
standard DID Resolution `resolve(did, resolutionOptions)` function: the client
passes the [[ref: DID Log]] and witness proofs directly, or by reference in the
[`srcLog` and `srcWitness` options](#the-srclog-and-srcwitness-options), as
[resolution options](#didvh-resolution-options), and the [[ref: Resolver]] verifies the
log and returns the [[ref: DIDDoc]]. How a client sources the
[[ref: DID Log]] beforehand — getting the log itself or a reference to it,
because it is [known in context](#known-in-context), from a [[ref: watcher]],
[exchanged peer-to-peer](#peer-to-peer-exchange), or from a web location — is
described non-normatively, and a [[ref: DID Log]] sourced in any of these ways
is verified in exactly the same way. The [[ref: SCID]] guarantees that the log
is the authentic history of the DID, wherever it came from.

`did:vh` provides the same verifiable-history features as `did:webvh`:

- A [[ref: self-certifying identifier]] (SCID), globally unique and derived
  from the initial [[ref: DID log entry]], binding the DID to its full
  history and mitigating the risk of attackers creating a new object with the
  same identifier.
- The ability to resolve the full history of the DID using a verifiable chain
  of updates to the [[ref: DIDDoc]], from creation to deactivation.
- [[ref: DIDDoc]] updates containing a proof signed by a
  [[ref: DID Controller]]-authorized key.
- An optional mechanism for publishing "pre-rotation" key hashes to prevent
  the loss of control of a DID in cases where an active private key is
  compromised.
- An optional mechanism for having collaborating [[ref: witnesses]] approve
  updates to the DID by a [[ref: DID Controller]] before they are distributed.
- Support for cryptographic agility through versioned specification upgrades,
  algorithm-identifying formats (e.g., [[ref: multihash]] and
  [[ref: Data Integrity]] proofs), and per-entry method parameters in the
  [[ref: DID Log]] that enable DIDs to evolve cryptographically over time.
- An optional mechanism for listing [[ref: watchers]] that archive and
  re-serve the DID's history, and that resolvers can use to check that a copy
  of the log is current and has not been forked.

Separating the identifier from the location of its history gives `did:vh`
properties that a web-located DID cannot have:

- **Permanence.** The DID never has to change because the place its history
  is kept changes. There is no need for a portability mechanism.
- **Location independence.** The [[ref: DID Log]] can be kept on a web server,
  with a [[ref: watcher]], in a database, on a distributed ledger, or only in
  the wallets of the parties to a relationship — and in several such places at
  once.
- **Peer-to-peer use.** A `did:vh` DID can be used between two parties who
  exchange their [[ref: DID Logs]] directly, in the manner of `did:peer`
  [[spec:DID-PEER]], while still supporting key rotation, pre-rotation and the
  other `did:vh` features.

`did:vh` is comparable to the `did:scid:vh` format of the proposed
[`did:scid`](https://lf-toip.atlassian.net/wiki/spaces/HOME/pages/88572360/DID+SCID+Method+Specification)
metamethod, and adopts its `src` parameter.
