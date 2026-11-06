## Definitions

[[def: base58btc]]

~ Applies [[spec:draft-msporny-base58-03]] to convert data to a `base58`
encoding. Used in `did:vh` for encoding hashes for [[ref: SCIDs]] and
[[ref: entry hashes]]. `did:vh` uses the plain base58btc encoding, without a
[[ref: multibase]] prefix, for [[ref: SCIDs]], [[ref: entry hashes]] and
`nextKeyHashes` values: for example, a SHA-256 [[ref: multihash]] encodes to a
string beginning `Qm`, not `zQm`.

[[def: Data Integrity]]

~ [W3C Data
Integrity](https://www.w3.org/community/reports/credentials/CG-FINAL-data-integrity-20220722/)
is a specification of mechanisms for ensuring the authenticity and integrity of
structured digital documents using cryptography, such as digital signatures and
other digital mathematical proofs.

[[def: Decentralized Identifier, Decentralized Identifiers, DID, DIDs]]

~ Decentralized Identifiers (DIDs) [[spec:did-core]] are a type of identifier
that enable verifiable, decentralized digital identities. A DID refers to any
subject (e.g., a person, organization, thing, data model, abstract entity,
etc.) as determined by the controller of the DID.

[[def: DID Controller, DID Controller's, DID Controllers]]

~ The entity that controls (creates, updates, deactivates) a given DID, as
defined in [[spec:DID-CORE]].

[[def: DIDDoc]]

~ A DID Document as defined by [[spec:DID-CORE]] — the document returned when
a DID is resolved.

[[def: DID Log, DID Logs]]

~ A DID Log is a list of [[ref: Entries]], with an entry added for each update
of the DID, including new versions of the [[ref: DIDDoc]] or changed
information necessary to generate or validate the DID. It is conventionally
stored as the file `did.jsonl`.

[[def: DID Log Entry, DID Log Entries, Entries, Log Entries, Log Entry]]

~ A DID Log Entry is a JSON object that defines the authorized transformation
of a [[ref: DIDDoc]] from one version to the next. The initial entry
establishes the DID and version 1 of the [[ref: DIDDoc]]. All entries are
stored in the [[ref: DID Log]].

[[def: DID Log source, DID Log sources]]

~ A place from which a client sources the [[ref: DID Log]] of a `did:vh` DID
— that is, gets either the log itself or a reference to it — before passing
it to a [[ref: Resolver]]: the context in which the DID was received, a
[[ref: watcher]], a peer, or a web location. See [Sourcing the DID
Log](#sourcing-the-did-log).

[[def: DID Method, DID Methods]]

~ DID methods are the mechanism by which a particular type of DID and its
associated DID document are created, resolved, updated, and deactivated. DID
methods are defined using separate DID method specifications. This document is
the DID Method Specification for `did:vh`.

[[def: DID Resources, DID Resource]]

~ A DID Resource is an object (often a file) that is referenced by a DID URL,
with a path to the resource. The DID URL allows DID URL dereferencers to
locate and retrieve specific content associated with a DID. Examples include
configuration files, schemas, credential definitions, logos, images or other
data linked to the DID.

[[def: did:key]]

~ `did:key` as described in the [W3C CCG
specification](https://w3c-ccg.github.io/did-key-spec/) is a DID method that
derives a DID document directly from a single, encoded cryptographic public
key, requiring no registry, ledger, or network interaction to resolve.
`did:vh` uses `did:key` DIDs to identify [[ref: witnesses]], where its
cryptographic self-sufficiency means a witness proof can be verified without
resolving anything.

[[def: Entry Hash, entryHash, entry hashes]]

~ A `did:vh` entry hash is a hash generated using a formally defined process
over the input data to a [[ref: log entry]], excluding the
[[ref: Data Integrity]] proof. The input data includes content from the
predecessor to the version of the DID, ensuring that all the versions are
"chained" together in a sort of microledger. The generated
[[ref: entry hash]] is subsequently included in the `versionId` of the
[[ref: log entry]] and **MUST** be verified by a [[ref: Resolver]].

[[def: ISO8601, ISO8601 String]]

~ A date/time expressed using the [ISO8601
Standard](https://en.wikipedia.org/wiki/ISO_8601).

[[def: JSON Canonicalization Scheme]]

~ [[spec:rfc8785]] defines a method for canonicalizing a JSON structure such
that it is suitable for verifiable hashing or signing.

[[def: JSON Lines, JSON Line]]

~ A file of JSON Lines, as described on the site
[https://jsonlines.org/](https://jsonlines.org/). In short, `JSONL` is lines
of JSON with whitespace removed and separated by a newline that is convenient
for handling streaming JSON data or log files.

[[def: multibase]]

~ A specification for encoding binary data as a string using a prefix that
indicates the encoding.

[[def: multikey]]

~ A verification method that encodes key types into a single binary stream
that is then encoded as a [[ref: multibase]] value.

[[def: multihash]]

~ Per the [[spec:multiformats]], [[ref: multihash]] is a specification for
differentiating instances of hashes. Software creating a hash prefixes
(according to the specification) data to the hash indicating the algorithm
used and the length of the hash, so that software receiving the hash knows
how to verify it. Although [[ref: multihash]] supports many hash algorithms,
for interoperability, [[ref: DID Controllers]] **MUST** only use the hash
algorithms defined in this specification as permitted.

[[def: parameters, parameter]]

~ `did:vh` parameters are a defined set of configurations that control how the
[[ref: DID Controller]] has generated the DID, and how the [[ref: Resolver]]
processes the DID [[ref: Log entries]]. The use of parameters allows for
the controlled evolution of `did:vh` log handling, such as evolving the set of
permitted hash algorithms or cryptosuites. This enables support for very long
lasting identifiers -- decades.

[[def: Pre-Rotation, Key Pre-Rotation, Pre-Rotation Key]]

~ A technique for a controller of a cryptographic key to commit to the public
key it will rotate to next, without exposing that actual public key. It
protects from an attacker that gains knowledge of the current private key from
being able to rotate to a new key known only to the attacker.

[[def: Resolver, Resolvers]]

~ A DID resolver, as defined in [[spec:DID-RESOLUTION]], that implements this
specification: given a `did:vh` DID and a [[ref: DID Log]] passed in its
resolution options, it verifies the log and returns the current or a
historical [[ref: DIDDoc]].

[[def: self-certifying identifier, self-certifying identifiers, SCID, SCIDs]]

~ An object identifier derived from initial data such that an attacker could
not create a new object with the same identifier. The input for a `did:vh`
SCID is the initial [[ref: DID log entry]] with the placeholder `{SCID}`
wherever the SCID is to be placed. A `did:vh` DID is `did:vh:` followed by its
SCID.

[[def: Verifiable Credential, Verifiable Credentials]]

~ A verifiable credential can represent all of the same information that a
physical credential represents, adding technologies such as digital
signatures, to make the credentials more tamper-evident and so more
trustworthy than their physical counterparts. The [Verifiable Credential Data
Model](https://www.w3.org/TR/vc-data-model/) is a W3C Standard.

[[def: Verifiable Presentation, Verifiable Presentations]]

~ A [[ref: verifiable presentation]] data model is part of W3C's [Verifiable
Credential Data Model](https://www.w3.org/TR/vc-data-model/) that contains a
set of [[ref: verifiable credentials]] about a `credentialSubject`, and a
signature across the verifiable credentials generated by that subject. In this
specification, the use case of primary interest is where the DID is the
`credentialSubject` and the DID signs the [[ref: verifiable presentation]].

[[def: watcher, watchers]]

~ Watchers are entities within a decentralized trust ecosystem that retrieve,
verify, archive and re-serve [[ref: DID Logs]], indexed by [[ref: SCID]], on
behalf of their clients. Watchers maintain a historical cache of DID versions
and verify that the [[ref: DID Controller]] is consistently following the
prescribed evolution process, including detecting conflicting copies of a
log. For `did:vh`, watchers are also a primary [[ref: DID Log source]].
Watchers provide endpoints for retrieving DID information and to receive
[[ref: webhooks]] notifying the watcher about updates to the DID and deletion
requests.

[[def: webhook, webhooks]]

~ A webhook is a mechanism that enables real-time communication between
systems by sending HTTP callbacks (typically POST requests) to a specified URL
when an event occurs. Webhooks are commonly used for event-driven integrations
and automation. Although webhooks are an implementation pattern rather than a
formal standard, best practices are documented in [[spec:rfc8030]].

[[def: witness, witnesses, witnessed]]

~ Witnesses are participants in the process of creating and verifying a
version of a `did:vh` [[ref: DIDDoc]]. Notably, a witness receives from the
[[ref: DID Controller]] a [[ref: DID Log]] entry ready for distribution,
verifies it according to this specification, and approves it according to its
ecosystem governance (whatever that might be). If the verification and
approval process results are positive, the witness returns to the
[[ref: DID Controller]] a [[ref: Data Integrity]] proof attesting to that
positive result. `did:vh` witnesses are identified by [[ref: did:key]] DIDs.

[[def: threshold, witness threshold]]

~ An algorithm that defines when a sufficient number of [[ref: witnesses]]
have submitted valid [[ref: Data Integrity]] proofs for a
[[ref: DID Log entry]] such that it is approved and can be distributed. The
algorithm details are in the [Witness Threshold
Algorithm](#witness-threshold-algorithm) section of this specification.
