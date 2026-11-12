## `did:vh` DID Method Specification

### Conformance

As well as sections marked as non-normative, all examples and notes in this
specification are non-normative. Everything
else in this specification is normative.

The key words **MAY**, **MUST**, **MUST NOT**, **NOT REQUIRED**,
**RECOMMENDED**, **REQUIRED**, **SHOULD**, and **SHOULD NOT** in this
specification are to be interpreted as described in BCP 14 [[spec:RFC2119]]
[[spec:RFC8174]] when, and only when, they appear in all capitals, as shown
here.

### Relationship to `did:webvh`

`did:vh` is derived from the
[`did:webvh`](https://identity.foundation/didwebvh/) DID method, and is
deliberately as close to it as its lack of a location allows. The log format,
[[ref: SCID]] and [[ref: entry hash]] algorithms, key authorization,
pre-rotation, [[ref: witnesses]], witness identifiers (`did:key`), the witness
proofs file format, [[ref: watchers]], and the `versionId`, `versionTime` and
`versionNumber` resolution options are the same in both methods. The
differences are:

| | `did:webvh` | `did:vh` |
|---|---|---|
| DID | `did:webvh:<SCID>:<domain>[:<path>]` | `did:vh:<SCID>` |
| Locating the [[ref: DID Log]] | DID-to-HTTPS transformation | Passed to the [[ref: Resolver]] in [resolution options](#didvh-resolution-options), after the client [sources it](#sourcing-the-did-log) |
| `method` values | `did:webvh:1.0` | `did:vh:1.0` |
| `portable` parameter | Optional | Not used — the DID never changes |
| Parallel `did:web` | Optional | Not applicable |

Implementations of `did:webvh` can support `did:vh` by replacing the
DID-to-HTTPS transformation with the [resolution
options](#didvh-resolution-options) defined here and dropping the features
that depend on a web location.

### Target System

`did:vh` does not have a single target system. A `did:vh` [[ref: DID Log]] can
be held by any system that can store and return a file: a web server, a
[[ref: watcher]], a database or registry, a distributed ledger, or the wallets
of the parties that use the DID. The [[ref: SCID]] and the verifiable history
make the [[ref: DID Log]] verifiable no matter which of those systems it was
obtained from.

### Method Name

The namestring that identifies this DID method is: `vh`. A DID that uses this
method **MUST** begin with the prefix `did:vh:`. Per the DID specification, the
prefix **MUST** be in lowercase. The remainder of the DID, after the prefix, is
the [method-specific identifier](#method-specific-identifier).

### Method-Specific Identifier

Every `did:vh` DID **MUST** conform to the DID Syntax ABNF Rules in
[[spec:DID-CORE]] Section 3.1. The `did:vh` method-specific identifier is the
[[ref: SCID]] of the DID, and nothing else. The [[ref: SCID]] **MUST** be
[generated](#scid-generation-and-verification) during the creation of the DID
from its initial content.

A `did:vh` DID **MUST** conform to the following ABNF [[spec:RFC5234]]:

```abnf
vh-did      = "did:vh:" scid
scid        = 46base58-char
base58-char = %x31-39 / %x41-48 / %x4A-4E / %x50-5A / %x61-6B / %x6D-7A
              ; the base58btc alphabet: 1-9, A-Z except I and O,
              ; a-z except l
```

ABNF string literals are case-insensitive, but [[spec:DID-CORE]] requires the
method name, and so the `did:vh:` prefix, to be lowercase. The `scid`
production describes the base58btc-encoded [[ref: multihash]] generated and
verified as specified in [SCID Generation and
Verification](#scid-generation-and-verification). It is case-sensitive: two
`did:vh` DIDs are the same DID if and only if they are identical character for
character. The `{SCID}` value used temporarily during DID creation is a
placeholder and is not a conforming `scid`; it **MUST** be replaced before the
DID is distributed or resolved.

A `did:vh` DID URL **MUST** conform to the DID URL Syntax ABNF Rules in
[[spec:DID-CORE]] Section 3.2, and its contained DID **MUST** conform to the
rule above. `did:vh` defines the DID URL query parameters described in
[Reading `did:vh` DID URLs](#reading-didvh-did-urls), including the optional
[`src`](#the-src-did-url-parameter) parameter.

The `id` of a resolved `did:vh` [[ref: DIDDoc]] identifies the DID subject and
therefore **MUST** be a bare `did:vh` DID. It **MUST NOT** contain a DID URL
path, query, or fragment component.

::: example
A `did:vh` DID, and DID URLs using the `src` and `versionNumber` query
parameters, and a fragment:

```text
did:vh:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ
did:vh:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ?src=https://example.com/dids/issuer
did:vh:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ?versionNumber=3
did:vh:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ#key-1
```
:::

::: warning
Verification of a `did:vh` DID using this specification ensures cryptographic
validity, but that does not imbue "trust" in the identifier itself. Trust in a
`did:vh` DID comes from external sources, such as verifiable
credentials issued by trusted parties or via trust registries that
maintain authoritative records of trusted DIDs in a given context.
Implementers are advised not to conflate technical verification with
trustworthiness.
:::

### The DID Log File

The [[ref: DID Log]] contains a list of [[ref: entries]], one for each version
of the DID. A version of the DID is an update to the contents of the resolved
[[ref: DIDDoc]] for the DID, and/or a change to the [[ref: parameters]] that
control the generation and verification of the DID.

Each entry is a JSON object consisting of the following properties.

`{ "versionId": "", "versionTime": "", "parameters": {}, "state": {}, "proof" : [] }`

1. The value of `versionId` **MUST** be a string consisting of the DID version
   number (starting at `1` and incrementing by one per DID version), a literal
   dash `-`, and the `entryHash`, a hash calculated across the
   [[ref: log entry]] content. The input to the hash is chosen so as to link
   each entry to its predecessor in a ledger-like chain. The input to the hash
   is specified in the [Entry Hash Generation and
   Verification](#entry-hash-generation-and-verification) section of this
   specification.
2. The value of `versionTime` **MUST** be a timestamp in UTC of the entry in
   [[ref: ISO8601]] format, as asserted by the [[ref: DID Controller]]. The
   timestamp **MUST** be the time the entry will be retrieved by a
   [[ref: witness]] or [[ref: Resolver]], or before. The timestamp **MUST** be
   expressed in whole seconds, with no fractional seconds (for example,
   `2025-04-01T17:39:50Z`). Because each entry's `versionTime` is later
   than the previous entry's, consecutive entries are at least one second
   apart.
3. The JSON object `parameters` contains the configurations/options set by the
   [[ref: DID Controller]] to be used in the processing of current and future
   [[ref: log entries]]. Permitted `parameters` are defined in the [`did:vh`
   DID Method Parameters](#didvh-did-method-parameters) section of this
   specification.
4. The JSON object `state` contains the [[ref: DIDDoc]] for this version of the
   DID.
5. The JSON array `proof` contains a [[ref: Data Integrity]] proof created for
   the entry and signed by a key authorized to update the [[ref: DIDDoc]].

After creation, each entry has (per the [[ref: JSON Lines]] specification) all
extra whitespace removed, a `\n` character appended, and the result added to
the [[ref: DID Log]].

When the [[ref: DID Log]] is stored as a file, the file **SHOULD** be named
`did.jsonl`, and **MUST** be named `did.jsonl` when stored at a location named
by a [`src` DID URL parameter](#the-src-did-url-parameter). Its media type
**SHOULD** be `text/jsonl`. When [[ref: witnesses]] are used, the [witness
proofs file](#the-witness-proofs-file) **SHOULD** be named `did-witness.json`,
and **MUST** be named `did-witness.json` when stored at a location named by a
`src` DID URL parameter. Whenever the files are transferred over a network, they
**MUST** be published and retrieved as described in [Publishing and Retrieving
DID Log Resources](#publishing-and-retrieving-did-log-resources).

A more comprehensive description of how to create and update a
[[ref: DID log entry]] is given in steps 4 - 6 of the [create
DID](#create-register) section.

::: example

**Examples of log entries that fail verification.** Illustrative and
non-exhaustive; the normative verification criteria are defined in the
[Resolution Algorithm](#resolution-algorithm).

1. **Duplicate witness IDs.** `{"threshold": 1, "witnesses": [{"id":
   "did:key:X"}, {"id": "did:key:X"}]}` — the same `did:key` appears more than
   once in the witnesses list; each witness has to be unique.

2. **Pre-rotation active but `updateKeys` omitted.** If entry N-1 commits
   `nextKeyHashes: ["H1","H2"]`, then entry N has to contain an explicit
   `updateKeys` whose every member hashes to a value in `["H1","H2"]`. An
   entry N omitting `updateKeys` (intending to inherit) fails verification.

3. **Wrong cryptosuite on log-entry proof.**
   `{"type":"DataIntegrityProof","cryptosuite":"ecdsa-jcs-2019","proofPurpose":"assertionMethod"}`
   — fails verification even if the signature is structurally valid, because
   `parameters.method` with value `did:vh:1.0` requires `eddsa-jcs-2022`.

4. **`state.id` is not the DID.** The first entry has `parameters.scid:
   "Qm111..."`, and an entry has `state.id:
   "did:vh:Qm222..."`, `"did:vh:Qm111...?src=https://example.com"` or
   `"did:webvh:Qm111...:example.com"`. Every entry's `state.id` has to be
   exactly `did:vh:Qm111...`.

5. **`portable` parameter present.** An entry includes `"portable": true`.
   `portable` is not a `did:vh` [[ref: parameter]].

6. **Malformed `method` value.** The first entry has `parameters.method:
   "did:vh:1.0-rc1"`, `"did:vh:01.0"`, `"did:webvh:1.0"` or `"didvh:1.0"`.
   (A well-formed value for a version the [[ref: Resolver]] does not support,
   such as `"did:vh:99.0"`, is not a verification failure: the
   [[ref: Resolver]] returns `featureNotSupported`, and never silently
   downgrades.)

7. **Fractional seconds.** An entry has `versionTime:
   "2025-04-01T17:39:50.123Z"`. `versionTime` values are in whole seconds.

8. **Entry after `updateKeys: []`.** Entry N sets `updateKeys: []` (with
   [[ref: pre-rotation]] not active). Entry N+1 sets `nextKeyHashes: ["H1"]`
   and `updateKeys: ["K1"]`, where K1 hashes to H1, and is signed by K1. It
   fails verification: [[ref: pre-rotation]] is not active for entry N+1, so
   it has to be signed by a key from entry N's `updateKeys`, which is empty.

:::

### DID Method Operations

#### Create (Register)

Creating a `did:vh` DID is done by carrying out the following steps.

1. **Define the DID string**
   The DID string **MUST** be the literal string `did:vh:{SCID}`, where
   `{SCID}` is a placeholder that will be replaced by the calculated
   [[ref: SCID]] later in the process (see step 5). Unlike `did:webvh`, no
   domain name or path follows the placeholder: the location where the
   [[ref: DID Log]] will be kept is not part of the DID.

2. **Generate the authorization key pair(s)**
   [Authorized keys](#authorized-keys) are authorized to control (create,
   update, deactivate) the DID. At the same time, generate any other key pairs
   that will be placed into the initial [[ref: DIDDoc]] for the DID.

   1. If the DID is to use [[ref: pre-rotation]], additional key generation
      will be necessary to generate the required "next" authorization keys and
      their corresponding [[ref: pre-rotation]] hashes.
   2. For each authorization key pair, generate a [[ref: multikey]] based on
      the key pair's public key. The [[ref: multikey]] representations of the
      public keys are placed in the `updateKeys` property in
      [[ref: parameters]].
   3. The public key(s) of the authorization key pair(s) **MAY** be used in
      the [[ref: DIDDoc]] as well, but that is not required.

3. **Create the initial [[ref: DIDDoc]] for the DID**
   The [[ref: DIDDoc]] **MUST** contain the top level `id` property, which
   **MUST** be the DID string from step 1, `did:vh:{SCID}`. Other
   [[ref: DIDDoc]] verifications **SHOULD** be performed.

   All other absolute references to the DID in the [[ref: DIDDoc]] **MUST**
   use the form defined in step 1, with the placeholder for the
   [[ref: SCID]] (e.g., `did:vh:{SCID}#key-1`).

   The [[ref: DIDDoc]] can contain any other content as deemed necessary by
   the [[ref: DID Controller]].

   1. Note: The placeholder (the string `{SCID}`) **MUST** be in every place
      in the [[ref: DIDDoc]] where the [[ref: SCID]] is to be placed.

<a id="note-did-scid-vh-diddoc-id"></a>

::: note
**Future use by `did:scid:vh`.** In this version of `did:vh`, the `id` of
every [[ref: DIDDoc]] in the [[ref: DID Log]], starting with the initial one,
is the `did:vh` DID (see step 6 of [Read (Resolve)](#read-resolve)). If
`did:scid:vh` evolves to use the `did:vh` format (see [Relationship to
`did:scid`](#relationship-to-didscid)), the `id` of each [[ref: DIDDoc]] in a
`did:scid:vh` log — and so the DID string in the initial entry that is input
to the [[ref: SCID]] — would be the `did:scid:vh` DID, since DID Resolution
requires a resolved DIDDoc's `id` to be the DID being resolved. The same
question has been raised for `did:webvh` in [did:webvh issue
#305](https://github.com/decentralized-identity/didwebvh/issues/305). Because
`did:scid` is not yet stable, this specification does not define that case. A
later version of `did:vh` is expected to, for example by allowing the
[[ref: DIDDoc]] `id` to be a `did:scid:vh` DID with the same [[ref: SCID]],
while the rest of the [[ref: DID Log]] is created and verified as defined
here.
:::

4. **Generate a preliminary DID Log Entry** JSON object containing the same
   JSON properties that will be in the published [[ref: DID log entry]], but
   with some values preset, pending calculation of the [[ref: SCID]] and
   [[ref: entryHash]] and without the `proof`.

   1. The value of `versionId` string **MUST** be the placeholder literal
      `"{SCID}"`.
   2. The value of `versionTime` string **MUST** be a valid UTC
      [[ref: ISO8601]] date/time string in whole seconds, and the represented
      time **MUST** be before or equal to the current time.
   3. The value of the `parameters` property **MUST** be a JSON object defined
      at the discretion of the [[ref: DID Controller]]. The properties in this
      nested JSON object **MUST** be as permitted in the [`did:vh` DID Method
      Parameters](#didvh-did-method-parameters) section of this specification,
      and all required values in the first version of the DID **MUST** be
      present. In addition, where the [[ref: SCID]] of the DID is referenced
      in the parameters, the placeholder literal string `{SCID}` **MUST** be
      used in place of the to-be-calculated [[ref: SCID]].
   4. The value of the `state` property **MUST** be the initial
      [[ref: DIDDoc]] as defined in the previous step 3 of this process.

5. **Update the preliminary DID Log Entry to the initial DID Log Entry**
   Use the preliminary [[ref: DID log entry]] to perform the consecutive
   steps:

   1. **Calculate the [[ref: SCID]]**
      The preliminary JSON object **MUST** be used to calculate the
      [[ref: SCID]] for the DID as defined in the [SCID Generation and
      Verification](#scid-generation-and-verification) section of this
      specification.
   2. **Replace the placeholder `{SCID}`** Treating the preliminary JSON
      object entry as a string, perform a literal text replacement of every
      occurrence of the placeholder `{SCID}` with the calculated
      [[ref: SCID]] from the previous step.
      - NOTE: As a consequence, the literal string `{SCID}` cannot appear as a
        value anywhere in the first [[ref: DID Log]] entry — it will always be
        replaced. If that string is required in the [[ref: DIDDoc]], it can be
        introduced via an update in a subsequent [[ref: log entry]].
   3. **Calculate the [[ref: Entry Hash]]**
      The preliminary JSON object updated in the previous step **MUST** be
      used to calculate the [[ref: Entry Hash]] (`entryHash`) for the
      [[ref: log entry]], as defined in the [Entry Hash Generation and
      Verification](#entry-hash-generation-and-verification) section of this
      specification.
   4. **Replace the preliminary `versionId` value** The value of the
      `versionId` property **MUST** be updated with the literal string `1`
      (for version number 1), a literal `-`, followed by the `entryHash` value
      calculated in the previous step.
   5. **Generate the [[ref: Data Integrity]] proof** A [[ref: Data Integrity]]
      proof on the preliminary JSON object as updated in the previous step
      **MUST** be generated using an authorized key in the required
      `updateKeys` property in the [[ref: parameters]] object and the
      `proofPurpose` set to `assertionMethod`.
   6. **Add the [[ref: Data Integrity]] proof** The [[ref: Data Integrity]]
      proof is added to the preliminary JSON object. The resultant JSON object
      is the initial [[ref: DID log entry]] for the DID.

6. **Generate the first [[ref: JSON Line]]**
   The [[ref: DID log entry]] **MUST** be updated to be a [[ref: JSON Lines]]
   entry by removing extraneous white space and appending a newline
   character, and the result stored as the contents of the [[ref: DID Log]]
   (`did.jsonl`).

   If the [[ref: DID Controller]] has opted to use [[ref: witnesses]] for the
   DID, the required proofs from the DID's [[ref: witnesses]] **MUST** be
   collected into the witness proofs file (`did-witness.json`) before the
   [[ref: DID Log]] is distributed. See the [DID Witnesses](#did-witnesses)
   section of this specification.

7. **Distribute the [[ref: DID Log]]**
   Instead of publishing the [[ref: DID Log]] at a location derived from the
   DID, the [[ref: DID Controller]] makes the [[ref: DID Log]] (and witness
   proofs file) available through the [[ref: DID Log sources]] (see [Sourcing
   the DID Log](#sourcing-the-did-log)) that the parties who will resolve the
   DID will use: giving it to a peer, sending it to [[ref: watchers]],
   publishing it at one or more web locations, or storing it where the
   ecosystem expects to find it. A [[ref: DID Controller]] **MUST** make the
   log available through at least one source that those parties can use.

   If there are [[ref: watchers]] configured for the DID, the
   [[ref: DID Controller]] notifies them that a new DID is available, as
   described in [Watcher HTTP API Operations](#watcher-http-api-operations).

#### Read (Resolve)

A `did:vh` [[ref: Resolver]] implements the `resolve` function defined in
[[spec:DID-RESOLUTION]]:

```text
resolve(did, resolutionOptions) →
   « didResolutionMetadata, didDocument, didDocumentMetadata »
```

The `did` input is the DID itself, `did:vh:<SCID>`, which says nothing about
where the [[ref: DID Log]] is. The client of the [[ref: Resolver]] therefore
passes the [[ref: DID Log]] — and, if [[ref: witnesses]] are in use, the
witness proofs — in `resolutionOptions`, in one of the [two
ways](#didvh-resolution-options) defined below: directly, or by reference, in
which case the [[ref: Resolver]] first retrieves the files. The
[[ref: Resolver]] then verifies the [[ref: DID Log]] and returns the requested
version of the [[ref: DIDDoc]].

The [[ref: Resolver]] does not discover the [[ref: DID Log]]. How a client
sources the [[ref: DID Log]] — gets the log itself, or a reference to it —
before calling `resolve` is described in the non-normative [Sourcing the DID
Log](#sourcing-the-did-log) section. Once it has the [[ref: DID Log]] the
client passed, the [[ref: Resolver]] retrieves further copies only when the
client asks it to, with the `checkWatchers` and `extraWatchers` options, to
check that the client's copy is current and has not been forked.

##### `did:vh` Resolution Options

A `did:vh` [[ref: Resolver]] recognises the following options in
`resolutionOptions`. `versionId` and `versionTime` are defined in
[[spec:DID-RESOLUTION]]. The others are `did:vh`-specific, and are to be
registered in [[spec:DID-EXTENSION-RESOLUTION]].

| Option | Value | Purpose |
|---|---|---|
| `didLog` | String | The complete [[ref: DID Log]], in [[ref: JSON Lines]] format. |
| `didWitness` | JSON array | The content of the [witness proofs file](#the-witness-proofs-file). |
| `srcLog` | String | A reference to the [[ref: DID Log]] file. See [The `srcLog` and `srcWitness` Options](#the-srclog-and-srcwitness-options). |
| `srcWitness` | String | A reference to the [witness proofs file](#the-witness-proofs-file). See [The `srcLog` and `srcWitness` Options](#the-srclog-and-srcwitness-options). |
| `versionId` | String | Select a version by `versionId`, as defined in [Selecting a Version](#selecting-a-version). |
| `versionTime` | [[ref: ISO8601]] string | Select the version active at a time, as defined in [Selecting a Version](#selecting-a-version). |
| `versionNumber` | Positive integer | Select a version by version number, as defined in [Selecting a Version](#selecting-a-version). |
| `checkWatchers` | Non-negative integer, or the string `all` | The number of the DID's listed [[ref: watchers]] from which the [[ref: Resolver]] **MUST** retrieve a copy of the [[ref: DID Log]]. Defaults to `0`. See [Comparing Copies of a DID Log](#comparing-copies-of-a-did-log). |
| `extraWatchers` | List of `https` URLs | [[ref: Watchers]], which need not be listed in the [[ref: DID Log]], from each of which the [[ref: Resolver]] **MUST** retrieve a copy of the [[ref: DID Log]]. Defaults to an empty list. |
| `minCopies` | Positive integer | The minimum number of retrieved copies of the [[ref: DID Log]] that **MUST** match for resolution to succeed. Defaults to `1`. |
| `includeDidLog` | Boolean | If `true`, the [[ref: Resolver]] **MUST** return the verified [[ref: DID Log]] and witness proofs file in the `didLog` and `didWitness` properties of the [DID Resolution Metadata](#did-resolution-metadata). Defaults to `false`. |

The `didLog`, `didWitness`, `srcLog` and `srcWitness` options support two ways
of giving the [[ref: Resolver]] the [[ref: DID Log]] and the witness proofs
file:

- **Passing directly.** The client passes the content of the
  [[ref: DID Log]] in `didLog`, and of the witness proofs file (if any) in
  `didWitness`.
- **Passing by reference.** The client passes, in `srcLog`, a reference to
  the file from which the [[ref: Resolver]] retrieves the [[ref: DID Log]],
  and, in `srcWitness`, a reference to the file from which it retrieves the
  witness proofs file (if any), as defined in [The `srcLog` and `srcWitness`
  Options](#the-srclog-and-srcwitness-options). In this version of `did:vh`,
  a reference is a web location, a location using another URL scheme the
  [[ref: Resolver]] supports, or, for a [[ref: Resolver]] local to its
  client, a local file or loopback web location.

The two files are given independently, so the [[ref: DID Log]] can be passed
directly and the witness proofs file by reference, or the reverse.

The following rules apply to the options:

- Exactly one of `didLog` and `srcLog` **MUST** be given.
- At most one of `didWitness` and `srcWitness` **MAY** be given.
- At most one of `versionId`, `versionTime` and `versionNumber` **MAY** be
  given.
- A malformed option, a value that does not conform to the rules for
  `srcLog` or `srcWitness`, a violation of any of the rules above, or an
  `extraWatchers` value that is not an `https` URL, **MUST** cause the
  `invalidOptions` error.
- A [[ref: Resolver]] **MUST** support files passed directly. It **MAY**
  decline to retrieve from `srcLog` and `srcWitness` references, based on its
  policy (for example, a [[ref: Resolver]] that never retrieves from
  locations chosen by its callers), and then **MUST** return the
  `featureNotSupported` error, except as defined in [Web
  Locations](#web-locations) and [Local Files](#local-files).
- A [[ref: Resolver]] **MUST** support `versionId` and `versionTime`, and
  **SHOULD** support `versionNumber`, `checkWatchers`, `extraWatchers`,
  `minCopies` and `includeDidLog`. A [[ref: Resolver]] given an option it does not support
  **MUST** return the `featureNotSupported` error rather than ignore the
  option.

::: example
Passing the files directly, resolving version 2:

```json
{
  "did": "did:vh:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ",
  "resolutionOptions": {
    "didLog": "{\"versionId\":\"1-Qm...\",...}\n{\"versionId\":\"2-Qm...\",...}\n",
    "didWitness": [ { "versionId": "2-Qm...", "proof": [ ... ] } ],
    "versionNumber": 2
  }
}
```

Passing the files by reference to a web location, and checking the log
against two of the DID's listed [[ref: watchers]]:

```json
{
  "did": "did:vh:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ",
  "resolutionOptions": {
    "srcLog": "https://example.com/dids/issuer/did.jsonl",
    "srcWitness": "https://example.com/dids/issuer/did-witness.json",
    "checkWatchers": 2,
    "minCopies": 2
  }
}
```

Passing the files by reference to local files, using a local
[[ref: Resolver]]:

```json
{
  "did": "did:vh:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ",
  "resolutionOptions": {
    "srcLog": "file:///var/lib/wallet/dids/QmfGEU.../did.jsonl",
    "srcWitness": "file:///var/lib/wallet/dids/QmfGEU.../did-witness.json"
  }
}
```
:::

###### The `srcLog` and `srcWitness` Options

The `srcLog` ("source of the log") option is a reference to the file holding
the [[ref: DID Log]], and the `srcWitness` ("source of the witness proofs")
option is a reference to the file holding the [witness proofs
file](#the-witness-proofs-file). Each value names a file, not a directory: the
[[ref: Resolver]] retrieves exactly the resource named, and does not add to
or change the path. The file names `did.jsonl` and `did-witness.json` are
therefore not required, and the two files need not be in the same place.

Each value **MUST** be an absolute URL [[spec:RFC3986]], and its scheme
determines the kind of reference:

- `https`, or `http` with a loopback host: a [web
  location](#web-locations), such as
  `https://example.com/dids/issuer/did.jsonl` or
  `http://localhost:8000/dids/issuer/did.jsonl`.
- `file`: a [local file](#local-files) [[spec:RFC8089]], such as
  `file:///var/lib/wallet/dids/QmfGEU.../did.jsonl`.
- Any other scheme: an [other scheme](#other-schemes) reference, such as
  `ipns://k51qzi5uqu5d.../dids/issuer/did.jsonl`.

A value that is not an absolute URL, or that is not valid for its kind,
**MUST** cause the `invalidOptions` error.

The [[ref: Resolver]] **MUST** retrieve the [[ref: DID Log]] from the
`srcLog` reference when one is given. If it is not found, the
[[ref: Resolver]] **MUST** return the `notFound` error.

The [[ref: Resolver]] **MUST** retrieve the witness proofs file from the
`srcWitness` reference only if the active [[ref: parameters]] of the
[[ref: DID Log]] require [[ref: witnesses]], and **MUST NOT** treat a
`srcWitness` reference that it did not need as an error. A client can
therefore pass `srcWitness` whenever a witness proofs file may exist. When
[[ref: witnesses]] are required but no witness proofs file was passed, or the
`srcWitness` reference was not found, the [[ref: Resolver]] **MUST** proceed
as if the witness proofs file contained no proofs, as defined in [Verifying
Witness Proofs During Resolution](#verifying-witness-proofs-during-resolution).

A `srcLog` or `srcWitness` reference is not authoritative: a
[[ref: DID Controller]] **MAY** stop publishing at a web location at any
time, and the [[ref: DID Log]] found through any reference may not be the
latest. Because the reference is not part of the DID, the DID does not change
when the log is stored elsewhere.

A reference also says nothing about who controls the DID: a
[[ref: DID Controller]] can publish a log at any location it can write to,
and any client can pass any reference. [[ref: Resolvers]] and their clients
**MUST NOT** treat a `srcLog` or `srcWitness` reference, or a [`src` DID URL
parameter](#the-src-did-url-parameter), as evidence of an association between
the DID and the owner of the location.

::: note
The `did:scid` `src` parameter can also name a DID method that stores the
verification data (for example, a DID-Linked Resource on a ledger). This
version of `did:vh` supports only URL references. Other kinds of reference,
such as DID values or references to other ways of storing a
[[ref: DID Log]], may be added in a later version.
:::

###### Web Locations

A web location reference is the URL of a file published on a web server,
such as `https://example.com/dids/issuer/did.jsonl`.

The following requirements apply to a web location reference:

- The value **MUST** be an absolute URL with the `https` or `http` scheme and
  a non-empty path, and **MUST NOT** include a fragment component.
- The value **MAY** include a port, such as
  `https://example.com:3000/dids/did.jsonl`, and **MAY** include a query
  component, such as a signed URL from a storage service or a
  [[ref: watcher]] operation (`https://watcher.example.org/log?scid=<SCID>`).
- For `https`, the host **MUST** be a domain name, not an IP address. A
  host containing non-ASCII characters (an internationalised domain name)
  **MUST** be converted to its ASCII form (A-labels) as defined in IDNA2008
  [[spec:RFC5891]] before retrieval; a host that cannot be converted **MUST**
  cause the `invalidOptions` error.
- For `http`, the host **MUST** be a loopback host: `localhost`, an IPv4
  address in `127.0.0.0/8`, or `[::1]`. An `http` reference with any other
  host **MUST** cause the `invalidOptions` error. An `http` reference is a
  *loopback reference*, intended for development and testing.
- Each path segment, once percent-decoded, **MUST NOT** be `.` or `..`, and
  **MUST NOT** contain `/`, `\` or NUL. The checks apply to the decoded form
  because percent-encoded variants such as `%2E%2E`, `%2F`, `%5C` and `%00`
  would otherwise pass unnoticed. A segment that fails a check **MUST** cause
  the `invalidOptions` error.
- A loopback reference is subject to the same restrictions as a [local
  file](#local-files) reference: a [[ref: Resolver]] that accepts requests
  from other systems **MUST NOT** retrieve from it and **MUST** return the
  `invalidOptions` error. A loopback reference would otherwise let a caller
  make a [[ref: Resolver]] send requests to services on its own host.

To retrieve a file from a web location, the [[ref: Resolver]] **MUST** make
an HTTP `GET` request (HTTPS for an `https` reference) for the URL as given,
following the requirements in [Retrieving DID Log
Resources](#retrieving-did-log-resources). When retrieving from a loopback
reference, the requirements in that section to use HTTPS and to reject
loopback hosts do not apply.

###### Local Files

A local file reference identifies a file on the system where the
[[ref: Resolver]] runs. It is for a [[ref: Resolver]] used by a client on the
same system — for example, a [[ref: Resolver]] used as a library or
command-line tool — such as a wallet that keeps the [[ref: DID Logs]] it holds
in local storage.

- A local file reference **MUST** be a `file:` URL [[spec:RFC8089]] with
  an absolute path, an empty host or the host `localhost`, and no query or
  fragment component. Each path segment, once percent-decoded, **MUST NOT**
  be `.` or `..`, and **MUST NOT** contain `/`, `\` or NUL; a segment that
  fails a check **MUST** cause the `invalidOptions` error.
- The [[ref: Resolver]] **MUST** read the file named by the path. A file that
  does not exist or cannot be read is treated as not found.
- A [[ref: Resolver]] **MUST** treat the content of the file exactly as if it
  had been passed directly.
- A [[ref: Resolver]] that accepts requests from other systems — for example,
  over HTTP — **MUST NOT** read local files, and **MUST** return the
  `invalidOptions` error for a local file reference.

A local file reference is never derived from a DID URL (see [The `src` DID
URL Parameter](#the-src-did-url-parameter)). A DID URL can come from an
untrusted party; allowing it to name local files would let such a party make
a [[ref: Resolver]] read files on its host.

::: example

Local file references and the corresponding files:

`file:///var/lib/wallet/dids/QmfGEU.../did.jsonl` --> `/var/lib/wallet/dids/QmfGEU.../did.jsonl`

`file://localhost/home/alice/dids/QmfGEU.../did-witness.json` --> `/home/alice/dids/QmfGEU.../did-witness.json`

:::

###### Other Schemes

A [[ref: Resolver]] **MAY** support `srcLog` and `srcWitness` references using
other URL schemes that name a file — for example, an `ipns` URL naming a file
in an IPFS directory, or an `ipfs` URL naming a file by the hash of its
content. For such a reference:

- The value **MUST NOT** include a fragment component.
- The [[ref: Resolver]] **MUST** retrieve the resource the URL names, without
  changing its path.
- A [[ref: Resolver]] that does not support the scheme **MUST** return the
  `featureNotSupported` error.
- A [[ref: Resolver]] **MUST** apply to the retrieval protections equivalent
  to those required in [Retrieving DID Log
  Resources](#retrieving-did-log-resources), including to any gateway it uses
  to reach the scheme's network.

::: example

Other scheme references:

`ipns://k51qzi5uqu5d.../dids/issuer/did.jsonl`

`ipfs://bafybei.../did.jsonl`

`ipfs://bafkrei...` (the [[ref: DID Log]] file itself, by content hash)

:::

::: note
Some schemes name immutable content. An `ipfs` URL names a file, or the
directory holding it, by the hash of its content, so any change to the
[[ref: DID Log]] or witness proofs file — including witness proofs added
after an entry — gives it a new identifier. An `ipfs` reference is therefore
a snapshot, useful for archiving but never updated; an `ipns` name, which the
[[ref: DID Controller]] repoints to each new version, can serve the latest
log. See [Log Freshness and Duplicity](#log-freshness-and-duplicity).
:::

##### Resolution Algorithm

The following steps **MUST** be executed to resolve the [[ref: DIDDoc]] for a
`did:vh` DID:

1. **The DID.** The `did` input **MUST** conform to the ABNF in
   [Method-Specific Identifier](#method-specific-identifier); if it does not,
   return the `invalidDid` error.
2. **The options.** Validate the `resolutionOptions` as defined in [`did:vh`
   Resolution Options](#didvh-resolution-options). If they are invalid,
   return the `invalidOptions` error.
3. **Getting the log.** Get the [[ref: DID Log]] according to how it was
   passed:
   - **Directly:** the `didLog` value.
   - **By reference:** retrieve it from the `srcLog` reference as defined in
     [The `srcLog` and `srcWitness`
     Options](#the-srclog-and-srcwitness-options). If the [[ref: DID Log]] is
     not found, return the `notFound` error.

   The witness proofs file is needed only if the active [[ref: parameters]]
   require [[ref: witnesses]]. When it is, use the `didWitness` value if one
   was given, or retrieve the file from the `srcWitness` reference if one was
   given, as defined in [The `srcLog` and `srcWitness`
   Options](#the-srclog-and-srcwitness-options).
4. **Processing the log.** Process the [[ref: DID Log]] as described below.
5. **Other copies.** If `checkWatchers`, `extraWatchers` or a `minCopies`
   greater than `1` was given, or the [[ref: Resolver]] has retained a
   `versionId` from an earlier resolution of the DID, process the copies as
   described in [Comparing Copies of a DID Log](#comparing-copies-of-a-did-log).
6. **Result.** Return the requested [[ref: DIDDoc]] (unless the DID is
   deactivated, as defined in [Deactivate (Revoke)](#deactivate-revoke)) with
   the metadata defined in [DID Resolution
   Metadata](#did-resolution-metadata), or the `error` listed there.

To process the [[ref: DID Log]], the [[ref: Resolver]] **MUST** carry out the
following steps on each of the [[ref: log entries]] in the order they appear,
applying the [[ref: parameters]] from the current and previous entries. Every
step **MUST** be performed for **every** entry; in particular,
[[ref: Data Integrity]] proof verification (step 2) and `entryHash`
verification (step 3) **MUST NOT** be skipped for intermediate entries on the
grounds that the [[ref: Resolver]] only needs the latest [[ref: DIDDoc]].

An entry that does not meet a requirement of these steps, or of the sections
they reference, *fails verification*. The [[ref: Resolver]] **MUST** stop
processing at the first entry that fails verification, and **MUST** then
produce the result as defined in [Results of Processing the
Log](#results-of-processing-the-log).

::: note Basic Note

NOTE: A [[ref: Resolver]] implementation that caches previously verified
state could process only subsequent entries and resume processing after the
last verified entry rather than reprocessing the full log, provided the
cached state was itself produced by full verification and the cache has not
been modified since.

Even without caching state, a [[ref: Resolver]] **MAY** retain the latest
`versionId` it observes for each DID it resolves. Comparing a later copy of
the [[ref: DID Log]] against that `versionId`, as described in [Comparing
Copies of a DID Log](#comparing-copies-of-a-did-log), reveals a fork,
truncation or rollback that has happened since the previous resolution.

:::

As noted in the [DID Log File](#the-did-log-file) section, [[ref: log entries]] are each a JSON object with the following properties:

1. `versionId`
2. `versionTime`
3. `parameters`
4. `state` -- the version's [[ref: DIDDoc]].
5. `proof` -- a [[ref: Data Integrity]] proof for the [[ref: log entry]].

For each entry:

1. Update the currently active [[ref: parameters]] with the
   [[ref: parameters]] from the entry (if any). The `parameters` **MUST**
   adhere to the [`did:vh` DID Method Parameters](#didvh-did-method-parameters)
   section of this specification. Continue processing using the now active
   set of [[ref: parameters]].
   - While all [[ref: parameters]] in the first [[ref: Log Entry]] take effect
     immediately, some kinds of [[ref: parameters]] defined in later
     [[ref: entries]] only take effect *after* that entry has been published.
     For example, updates to the `witness` parameter take effect only *after*
     the entry in which they are defined has been published.
2. The [[ref: Data Integrity]] proof in the entry **MUST** be valid and signed
   by an authorized key as defined in the [Authorized Keys](#authorized-keys)
   section of this specification, and with a `proofPurpose` set to
   `assertionMethod`.
   1. If the active `updateKeys` is empty — for example, after an entry that
      set `updateKeys` to `[]` — no key is authorized and the entry fails
      verification.
   2. If the [[ref: DID Controller]] has opted to use [[ref: witnesses]],
      [[ref: Resolvers]] **MUST** verify the DID's witness proofs file. For
      details, see the [DID Witnesses](#did-witnesses) section of this
      specification.
3. Verify the `versionId` for the entry.
   1. The version number **MUST** be `1` for the first entry and **MUST**
      equal the previous entry's version number + 1 for each subsequent entry.
      Gaps (e.g., entry 3 after entry 1) **MUST** cause the entry to fail
      verification.
   2. Exactly one dash `-` **MUST** follow the version number; missing or
      multiple dashes **MUST** cause the entry to fail verification.
   3. The `entryHash` **MUST** follow the dash, **MUST** be a valid
      [[ref: multihash]] in the algorithm permitted by the active `method`,
      and **MUST** be verified per [Entry Hash Generation and
      Verification](#entry-hash-generation-and-verification). Verification
      **MUST** be performed for every entry.
4. The `versionTime` **MUST** be a valid UTC [[ref: ISO8601]] string with
   explicit `Z` (or `+00:00`); values without a zone designator, or expressing
   a non-UTC zone, **MUST** cause the entry to fail verification.
   - The `versionTime` **MUST** be in whole seconds. Values with fractional
     seconds **MUST** cause the entry to fail verification.
   - The `versionTime` of **every** entry **MUST** be strictly greater than
     the immediately preceding entry's. Equal timestamps **MUST** cause the
     entry to fail verification.
   - The `versionTime` of every entry **MUST NOT** be more than a small,
     implementation-defined tolerance in the future relative to the
     [[ref: Resolver]]'s current time. [[ref: Resolvers]] **SHOULD** use a
     tolerance of no more than 5 minutes. Entries that exceed this tolerance
     **MUST** cause the entry to fail verification.
5. When processing the first [[ref: DID log]] entry, verify the
   [[ref: SCID]] (defined in the [[ref: parameters]]) according to the [SCID
   Generation and Verification](#scid-generation-and-verification) section of
   this specification. The [[ref: SCID]] **MUST** equal the [[ref: SCID]] of
   the `did` input.
6. Get the value of the [[ref: log entry]] property `state`, which is the
   [[ref: DIDDoc]] for the version. Its top-level `id` **MUST** be exactly
   `did:vh:` followed by the `scid` [[ref: parameter]] of the first entry.
   This check **MUST** apply to **every** entry's `state.id`, not just the
   first, and to every entry of any further copy retrieved with
   `checkWatchers` or `extraWatchers`. An entry with a mismatch fails
   verification.
7. If [[ref: Key Pre-Rotation]] is active (the previously active
   `nextKeyHashes` is non-empty), the entry being processed **MUST** include
   an explicit `parameters.updateKeys` and **MUST NOT** rely on inheritance.
   The hash of **every** [[ref: multikey]] in `parameters.updateKeys`
   (computed per [Pre-Rotation Key Hash Generation and
   Verification](#pre-rotation-key-hash-generation-and-verification)) **MUST**
   appear in the previous entry's `nextKeyHashes`. The check applies to the
   **full** `updateKeys` set, not only keys that are new relative to the
   previous entry. A key not in the previous entry's `nextKeyHashes` **MUST**
   cause the entry to fail verification, even if it appeared in an earlier
   entry's `updateKeys`.
8. If a previous entry set `deactivated` to `true`, the entry **MUST** be
   treated as failing verification, as defined in [Deactivate
   (Revoke)](#deactivate-revoke).
9. As each [[ref: log entry]] is processed and verified, collect the
   following information about each version:
   1. The [[ref: DIDDoc]].
   2. The `versionId` of the [[ref: DIDDoc]].
   3. The UTC `versionTime` of the [[ref: DIDDoc]].
   4. The latest list of active [[ref: multikey]] formatted public keys
      authorized to update the DID, from the `updateKeys` lists in the
      [[ref: parameters]].
   5. If [[ref: pre-rotation]] is being used, the hashes of authorized keys
      that are to be used in the `updateKeys` list of the next [[ref: DID log]]
      entry. The [[ref: pre-rotation]] hashes are in the `nextKeyHashes` list
      in the [[ref: parameters]].
   6. All other `did:vh` processing configuration settings as defined in the
      `parameters` object.
10. If the `parameters` for any of the versions define that some or all of
    the [[ref: DID Log entries]] require witnessing, the [[ref: Resolver]]
    **MUST** also verify the [[ref: witness]] proofs, as defined in the [DID
    Witnesses](#did-witnesses) section of this specification.

###### Results of Processing the Log

Once the [[ref: log entries]] have been processed as above, the
[[ref: Resolver]] **MUST** produce the result as follows. The *valid entries*
are the entries before the first entry that failed verification, or all of
the entries if none failed.

If processing stopped at an [unsupported](#didvh-did-method-parameters)
entry — one whose `method` value names a version of this specification the
[[ref: Resolver]] does not support — the entries before it are the valid
entries, and the rules below apply with `featureNotSupported` in place of
`invalidDid`. In step 3, the [[ref: Resolver]] **MUST** return the
`featureNotSupported` error rather than the [[ref: DIDDoc]] of the last valid
entry, because the unsupported entries may change the [[ref: DIDDoc]] or its
keys.

1. If the first entry failed verification, the DID is invalid: return the
   `invalidDid` error.
2. If a target version was requested, as defined in [Selecting a
   Version](#selecting-a-version):
   - if it is one of the valid entries, return its [[ref: DIDDoc]] with a
     successful status, even if later entries failed verification;
   - if it is an entry that failed verification, or comes after one, return
     the `invalidDid` error;
   - if no entry matches, return the `notFound` error.
3. Otherwise, return the [[ref: DIDDoc]] of the last valid entry. If any
   entry failed verification, include the `invalid-entries` warning defined
   in [DID Resolution Metadata](#did-resolution-metadata).

The `problemDetails` of an `invalidDid` or `featureNotSupported` error, and
the `message` of an
`invalid-entries` warning, **SHOULD** identify the entry that failed
verification (by version number, and `versionId` if it can be parsed) and the
requirement it did not meet.

While caching policies are an implementation matter and largely outside the
scope of this specification, [[ref: Resolvers]] **SHOULD NOT** cache a DID
that fails verification. This ensures that the [[ref: DID Controller]] has
the opportunity to recover a DID that may have been erroneously or
maliciously invalidated.

##### Comparing Copies of a DID Log

A [[ref: Resolver]] retrieves further copies of the [[ref: DID Log]] when the
client asks it to, using the `checkWatchers`, `extraWatchers` and `minCopies`
[resolution options](#didvh-resolution-options). These options let a client
check the [[ref: DID Log]] it passed against other copies, to detect a copy
that is out of date, has been tampered with, or has been forked. They apply as
follows:

1. The copy passed directly or by reference is processed as defined in the
   [Resolution Algorithm](#resolution-algorithm).
2. If `checkWatchers` is greater than `0`, the [[ref: Resolver]] reads the
   `watchers` [[ref: parameter]] active at the last entry of the verified
   copy. If `checkWatchers` is `all`, or is not less than the number of listed
   [[ref: watchers]], the [[ref: Resolver]] **MUST** retrieve a copy from
   every listed [[ref: watcher]]. Otherwise, it **MUST** retrieve a copy from
   that many listed [[ref: watchers]], and **SHOULD** choose them at random,
   so that which [[ref: watchers]] are checked cannot be predicted.
3. The [[ref: Resolver]] **MUST** retrieve a copy from each [[ref: watcher]]
   in `extraWatchers`.
4. Copies are retrieved from a [[ref: watcher]] using the [Watcher HTTP API
   Operations](#watcher-http-api-operations) and the [[ref: SCID]] of the DID:
   the [[ref: DID Log]] from `<WATCHER URL>/log?scid=<SCID>` and, if
   [[ref: witnesses]] are active, the witness proofs file from
   `<WATCHER URL>/witness?scid=<SCID>`. All retrievals **MUST** follow
   [Retrieving DID Log Resources](#retrieving-did-log-resources).
5. All copies are compared as described below, and each is reported in
   `copies` in the [DID Resolution Metadata](#did-resolution-metadata).
6. If fewer than `minCopies` copies have a `status` of `current` or `behind`,
   the [[ref: Resolver]] **MUST** fail resolution with the
   `insufficientCopies` error. The copy passed directly or by reference counts
   towards `minCopies`.

A copy that cannot be retrieved, because the [[ref: watcher]] is unreachable,
does not hold the [[ref: DID Log]], or returns an error, does not by itself
fail resolution. It is reported with `status` `unavailable` and a
`copy-unavailable` warning, and does not count towards `minCopies`.

`extraWatchers` causes the [[ref: Resolver]] to retrieve from URLs supplied by
its caller. A [[ref: Resolver]] offered as a service to other parties **MAY**
refuse, or limit the number of, `extraWatchers` URLs. A URL that is not
retrieved for this reason **MUST** be reported with `status` `unavailable` and
a `copy-unavailable` warning stating that it was refused.

These options apply even when a target version is selected, because a forked
log fails resolution for every version, as described below.

A [[ref: Resolver]] **MAY** retain the latest `versionId` it has verified for
each DID it resolves — the `versionId` of the last valid entry of the
reference copy — and use it when it next resolves the DID. A
[[ref: Resolver]] that does so **MUST** compare it with the reference copy as
defined in step 3 below. Retention is optional: a [[ref: Resolver]] that
retains no state conforms to this specification, but detects a fork,
truncation or rollback only among the copies it is given in a single
resolution. A retained `versionId` records which DIDs the
[[ref: Resolver]] has resolved; see [Surveillance](#surveillance).

A [[ref: Resolver]] that has more than one copy of a [[ref: DID Log]], or that
has retained a `versionId` from a previous resolution of the DID, need not
verify every copy in full. Instead, it **MUST**:

1. Use the copy passed directly or by reference as the reference copy, once it
   has been fully verified.
2. Compare each remaining copy to the reference copy by checking a single
   `versionId` in each:
   - **If the copy has the same number of entries or fewer:** compare the
     `versionId` of the copy's last entry with the reference copy's
     `versionId` for the same version number. If they match, the copy is the
     same log, possibly out of date, and needs no further verification.
   - **If the copy has more entries:** compare the `versionId` of the
     reference copy's last entry with the copy's `versionId` for the same
     version number. If they match, the copy extends the reference copy.
     Verify only the copy's additional entries, starting from the state
     reached at the reference copy's last entry. If they pass verification,
     the copy becomes the reference copy. If they fail, discard the copy.
   - **If the `versionId` values do not match:** the copy diverges from the
     reference copy. Verify it in full. If it fails verification, discard it.
     If it passes verification, the log has been forked, and the
     [[ref: Resolver]] **MUST** fail resolution as described below.
3. If the [[ref: Resolver]] has retained a `versionId` from a previous
   resolution of the DID, compare it with the reference copy's `versionId`
   for the same version number. The retained `versionId` was verified when it
   was first observed, so it needs no further verification.
   - **If the reference copy has fewer entries than the retained version
     number:** there is no entry to compare. Every copy is older than the log
     previously observed, so the log has been truncated or rolled back, or
     diverges at an entry the [[ref: Resolver]] cannot compare. The copy may
     simply be stale — for example, an older copy held by the client or a
     peer — so the [[ref: Resolver]] **MUST NOT** fail resolution for this
     reason, and **MUST** include the `retained-version-not-found` warning
     described in [DID Resolution
     Metadata](#did-resolution-metadata).
   - **If they match:** the reference copy is the log previously observed, or
     an extension of it.
   - **If they do not match:** the log has been forked, and the
     [[ref: Resolver]] **MUST** fail resolution as described below. In place
     of the first differing entry, the report **MUST** include the retained
     `versionId` and the reference copy's `versionId` for the same version
     number.

A copy already found to match the reference copy still matches when a longer
copy replaces it, because the longer copy extends the earlier reference copy;
earlier comparisons need not be repeated. When every copy and any retained
`versionId` have been compared, the [[ref: Resolver]] resolves from the
reference copy.

If two or more copies of a [[ref: DID Log]] pass verification but diverge,
the log has been forked by a party holding authorized keys, and no copy can be
relied on. The [[ref: Resolver]] **MUST** fail resolution with the
`logForked` error, including for a requested version that precedes the point
of divergence. The `problemDetails` **MUST** identify each diverging copy (for
example, by the location it was retrieved from) and **MUST** include the
version number of the first [[ref: log entry]] at which the copies differ and
each copy's `versionId` for that entry. The [[ref: Resolver]] does not choose
between the copies; see [Forked Logs](#forked-logs).

These checks rely on two properties of the [[ref: DID Log]]. Each
[[ref: entry hash]] covers the previous entry's `versionId`, so equal
`versionId` values at entry `k` mean the two copies commit to the same history
up to and including entry `k`. And because a `versionId` begins with its
version number, entry `k` is always on line `k` of the log, so each
comparison reads a single known line of each copy rather than searching for
the `versionId`.

::: note
[[ref: Watchers]] listed in the [[ref: DID Log]] are chosen by the
[[ref: DID Controller]]. Checking them detects a source that is out of date or
has been altered, but may not detect a [[ref: DID Controller]] that
deliberately gives different parties different logs, since such a
[[ref: DID Controller]] could list [[ref: watchers]] it controls.
`extraWatchers` lets a client use [[ref: watchers]] it chooses itself, such as
its own or those run by its ecosystem, which is a stronger check.
:::

##### Selecting a Version

A request to resolve a `did:vh` DID **MAY** specify a target version using one
of the `versionId`, `versionTime` or `versionNumber` [resolution
options](#didvh-resolution-options).

- **By `versionId`:** the [[ref: log entry]] whose `versionId` is exactly
  equal to the given value.
- **By `versionTime`:** the [[ref: log entry]] that was active at the given
  [[ref: ISO8601]] time — the entry with the latest `versionTime` that is not
  later than the given time.
- **By `versionNumber`:** the [[ref: log entry]] whose version number — the
  integer that precedes the `-` in its `versionId` — equals the given positive
  integer.

If no [[ref: log entry]] matches, the [[ref: Resolver]] **MUST** return the
`notFound` error.

::: note
Selecting by version number is less safe than selecting by `versionId`. A
`versionId` includes the [[ref: entry hash]], so it identifies the content of
exactly one [[ref: log entry]]. A version number identifies an entry only
within the copy of the log being resolved: if two diverging copies of a
[[ref: DID Log]] exist, `did:vh:<SCID>?versionNumber=3` can resolve to a
different [[ref: DIDDoc]] for different parties, while
`did:vh:<SCID>?versionId=3-Qm...` resolves to one [[ref: DIDDoc]] or fails.
Where a reference needs to identify exactly one version — for example, to record
which version of the [[ref: DIDDoc]] was relied on — `versionId` is the safer
choice.
:::

##### DID Resolution Metadata

As defined in [[spec:DID-RESOLUTION]], a `did:vh` [[ref: Resolver]]
**SHOULD** return the following DID Document Metadata:

```json
{
  "versionId": "1-QmRRaLXwc6BjBuBPosSupJwEQ8w9f3znP7yfbpGfwcnLr6",
  "versionTime": "2025-01-23T04:12:36Z",
  "created": "2025-01-23T04:12:36Z",
  "updated": "2025-01-23T04:12:36Z",
  "scid": "QmPEQVM1JPTyrvEgBcDXwjK4TeyLGSX1PxjgyeAisdWM1p",
  "deactivated": false,
  "ttl": 3600,
  "witness": { ... },
  "watchers": [ ... ]
}
```

where the items in the metadata are:

- `versionId` — The `versionId` of the [[ref: DID log entry]] of the resolved
  [[ref: DIDDoc]] version.
- `versionTime` — The `versionTime` of the [[ref: DID log entry]] of the
  resolved [[ref: DIDDoc]] version, as an [[ref: ISO8601]] timestamp.
- `created` — The `versionTime` of the first [[ref: DID log entry]], as an
  [[ref: ISO8601]] timestamp: when, according to the
  [[ref: DID Controller]], the DID was created.
- `updated` — The `versionTime` of the last valid [[ref: DID log entry]], as
  an [[ref: ISO8601]] timestamp.
- `scid` — The [[ref: SCID]] of the DID.
- `deactivated` — A boolean that is `true` if the DID has been deactivated,
  as defined in [Deactivate (Revoke)](#deactivate-revoke), even when an
  earlier version was resolved; otherwise `false`.
- `ttl` — The active `ttl` [[ref: parameter]], an unsigned integer number of
  seconds: guidance from the [[ref: DID Controller]] on how long to cache the
  resolution result.
- `witness` — The active `witness` [[ref: parameter]] object, as defined in
  [The `witness` Parameter](#the-witness-parameter), if [[ref: witnesses]]
  are active, with its `threshold` as a positive integer.
- `watchers` — The active list of [[ref: watcher]] URLs from the `watchers`
  [[ref: parameter]].

"Active" means as set by the last valid [[ref: DID log entry]]. That is the
last entry in the [[ref: DID Log]] unless later entries fail verification, in
which case it is the last entry before the first failure.

Metadata values have the same data types as the [[ref: parameters]] and
[[ref: log entry]] properties they come from. `updated`, and the values marked
active, reflect the [[ref: DID Log]] passed to the [[ref: Resolver]], which
may not be the latest (see [Log Freshness and
Duplicity](#log-freshness-and-duplicity)).

The following items describe the resolution rather than the DID, so they are
returned in the `didResolutionMetadata`, not the DID Document Metadata:

- `didLog` — If `includeDidLog` was `true` and resolution succeeded, the
  [[ref: DID Log]] the [[ref: Resolver]] resolved from, as a
  [[ref: JSON Lines]] string in the same form as the `didLog` resolution
  option. It is the
  reference copy (see [Comparing Copies of a DID
  Log](#comparing-copies-of-a-did-log)), which may be longer than the copy the
  client passed, ending at its last valid entry. It **MUST NOT** be shortened
  to a requested target version, and **MUST NOT** include entries that failed
  verification.
- `didWitness` — If `includeDidLog` was `true`, resolution succeeded and the
  [[ref: DID Log]] requires [[ref: witnesses]], the witness proofs file the
  [[ref: Resolver]] used, as a JSON array in the same form as the
  `didWitness` resolution option. The [[ref: Resolver]] **MAY** omit items
  whose `versionId` is not in the returned `didLog`, and **MAY** omit proofs
  that failed verification.

The returned `didLog` and `didWitness` can be passed unchanged to a later
resolution of the DID, so a client can keep the verified files — for example,
to resolve the DID again without retrieving them, to pass them to another
party, or to send them to a [[ref: watcher]] — without retrieving them
separately. When resolution fails, neither is returned.

- `copies` — If the [[ref: Resolver]] retrieved, or attempted to retrieve,
  more than one copy of the [[ref: DID Log]], a list with one item for each
  copy. A [[ref: Resolver]] given `checkWatchers`, `extraWatchers` or a
  `minCopies` greater than `1` **MUST** include `copies`. Each item has:
  - `source` — Where the copy came from: the `srcLog` value, `didLog` for a
    log passed directly, or the URL of the [[ref: watcher]] that served it.
  - `versionId` — The `versionId` of the copy's last [[ref: log entry]].
    Absent if `status` is `unavailable`.
  - `status` — One of `current` (the copy's last [[ref: log entry]] is the
    last entry of the reference copy), `behind` (the copy matches the
    reference copy but has fewer entries), `invalid` (the copy failed
    verification and was discarded) or `unavailable` (the copy could not be
    retrieved, or the [[ref: Resolver]] declined to retrieve it).
- `warnings` — A list of conditions found during resolution that did not
  prevent it. A [[ref: Resolver]] **SHOULD** include an item for each of the
  following conditions, with a `code` from the list below and a
  human-readable `message`, identifying the copy it concerns by its `source`,
  where there is one:
  - `copy-behind` — A copy has `status` `behind`. The message includes how
    many entries the copy is behind the reference copy.
  - `copy-invalid` — A copy has `status` `invalid`. The message includes the
    version number of the first [[ref: log entry]] that failed verification.
  - `copy-unavailable` — A copy has `status` `unavailable`. The message says
    why, such as the [[ref: watcher]] being unreachable, not holding the
    [[ref: DID Log]], or the [[ref: Resolver]] refusing the URL.
  - `invalid-entries` — Some [[ref: log entries]] at the end of the reference
    copy failed verification, so resolution used the last valid entry.
  - `retained-version-not-found` — The reference copy has fewer entries than a
    `versionId` retained from a previous resolution of the DID, as described
    in [Comparing Copies of a DID Log](#comparing-copies-of-a-did-log).

```json
{
  "contentType": "application/did+json",
  "copies": [
    {
      "source": "https://example.com/dids/issuer",
      "versionId": "3-QmRRaLXwc6BjBuBPosSupJwEQ8w9f3znP7yfbpGfwcnLr6",
      "status": "current"
    },
    {
      "source": "https://watcher.example.org",
      "versionId": "2-QmPEQVM1JPTyrvEgBcDXwjK4TeyLGSX1PxjgyeAisdWM1p",
      "status": "behind"
    },
    {
      "source": "https://watcher.example.net",
      "status": "unavailable"
    }
  ],
  "warnings": [
    {
      "code": "copy-behind",
      "source": "https://watcher.example.org",
      "message": "Copy is 1 entry behind the reference copy."
    },
    {
      "code": "copy-unavailable",
      "source": "https://watcher.example.net",
      "message": "Watcher did not respond."
    }
  ]
}
```

When a DID resolution error occurs, the `error` field **MUST** be included in
the `didResolutionMetadata`, as defined in [[spec:DID-RESOLUTION]], and
[[ref: Resolvers]] **SHOULD** include supplemental "Problem Details" metadata
following [[spec:rfc9457]], using the following structure:

```json
"didResolutionMetadata": {
  "error": "invalidDid",
  "problemDetails": {
    "type": "https://w3id.org/security#INVALID_CONTROLLED_IDENTIFIER_DOCUMENT_ID",
    "title": "The resolved DID is invalid.",
    "detail": "The SCID in state.id of entry 3 does not match the SCID of the DID."
  }
}
```

The following `error` values **MUST** be used under the corresponding
conditions. The first four are from [[spec:DID-EXTENSION-RESOLUTION]];
`logForked` and `insufficientCopies` are `did:vh`-specific, and are to be
registered there.

- `invalidDid` — The DID is malformed, or the [[ref: DID Log]] passed to the
  [[ref: Resolver]] (or the requested version) fails verification.
- `invalidOptions` — A resolution option is malformed or not permitted,
  including a local file or loopback reference passed to a
  [[ref: Resolver]] that accepts requests from other systems.
- `notFound` — The [[ref: DID Log]] was not found through the `srcLog`
  reference, the requested version does not exist, or the resource referenced
  by a DID URL was not found.
- `featureNotSupported` — The [[ref: Resolver]] declined to retrieve from a
  `srcLog` or `srcWitness` reference, does not support the scheme of such a
  reference, does not support `versionNumber`, `checkWatchers`,
  `extraWatchers`, `minCopies` or `includeDidLog`, or does not support the version of this
  specification named in a `method` [[ref: parameter]] of the
  [[ref: DID Log]].
- `logForked` — Two copies of the [[ref: DID Log]] both pass verification but
  diverge, as defined in [Comparing Copies of a DID
  Log](#comparing-copies-of-a-did-log). The `problemDetails` **MUST** include
  the fork report that section requires.
- `insufficientCopies` — Fewer than `minCopies` copies of the
  [[ref: DID Log]] were retrieved and matched.

[[ref: Resolvers]] **SHOULD** populate the `problemDetails` field to aid in
diagnosing and understanding resolution failures, without including internal
stack traces, file paths, or library versions.

##### Reading `did:vh` DID URLs

The `src`, `versionId`, `versionTime` and `versionNumber` DID URL query
parameters can be used with a `did:vh` DID, such as
`did:vh:<SCID>?src=https://example.com/dids/issuer&versionNumber=2`. `src` is
a DID URL parameter only: it names the location holding both files, from
which the dereferencer derives the `srcLog` and `srcWitness` resolution
options, as defined in [The `src` DID URL
Parameter](#the-src-did-url-parameter). `didLog`, `didWitness`, `srcLog`,
`srcWitness`, `checkWatchers`, `extraWatchers`, `minCopies` and
`includeDidLog` are resolution options only, and **MUST NOT** be used as DID URL query parameters.

When a `did:vh` DID URL is dereferenced, the DID URL dereferencer (the client
of the [[ref: Resolver]]) decides which of these parameters to use when
calling `resolve`:

- It **SHOULD** pass a `versionId`, `versionTime` or `versionNumber` DID URL
  parameter as the resolution option of the same name. These select a version
  as defined in [Selecting a Version](#selecting-a-version).
- It **MAY** apply its own policy to a valid `src` value — for example, using
  it only if it names a location the dereferencer trusts, or ignoring it in
  favour of a copy of the [[ref: DID Log]] it already has. If it uses the
  `src` value, it derives the file URLs as defined in [The `src` DID URL
  Parameter](#the-src-did-url-parameter) and passes them by reference in
  `srcLog` and `srcWitness`, or retrieves the files itself and passes them
  directly in `didLog` and `didWitness`.

The `src` parameter never changes which version is returned.

###### The `src` DID URL Parameter

The `src` ("source") DID URL parameter names the location where `did.jsonl`
and, if [[ref: witnesses]] are active, `did-witness.json` are published. It
is equivalent to the `src` parameter of
[`did:scid`](https://lf-toip.atlassian.net/wiki/spaces/HOME/pages/88572360/DID+SCID+Method+Specification)
when its value is an `https` URL.

The following requirements apply to a `src` DID URL parameter:

- It **MUST** be percent-encoded as required for a query parameter value by
  [[spec:RFC3986]] ([Section
  3.4](https://www.rfc-editor.org/rfc/rfc3986#section-3.4) and, for
  characters that conflict with delimiters such as `&` and `=`, [Section
  2.2](https://www.rfc-editor.org/rfc/rfc3986#section-2.2)), and a DID URL
  **MUST NOT** include more than one. The dereferencer **MUST** percent-decode
  it once before validating it.
- Its value **MUST** be an absolute URL [[spec:RFC3986]] with no query or
  fragment component, using either the `https` scheme or another scheme with
  hierarchical paths (such as `ipns`).
- An `https` value **MUST** meet the requirements for an `https` [web
  location](#web-locations), except that its path **MAY** be empty. After
  any trailing `/` is removed, each path segment, once percent-decoded,
  **MUST** be non-empty and **MUST NOT** begin or end with whitespace.
- It **MUST NOT** be a `file:` URL or an `http` URL. A DID URL can come from
  an untrusted party; allowing it to name local files or loopback hosts would
  let such a party make a [[ref: Resolver]] read files or send requests on
  its own host.
- The dereferencer **MUST** return the `invalidDidUrl` error for a `src`
  value that does not meet these requirements, and **MUST NOT** pass any
  reference derived from it to the [[ref: Resolver]].

To derive the `srcLog` and `srcWitness` values from a `src` value, the
dereferencer **MUST**:

1. Take the `src` value, removing any trailing `/`.
2. Append `/did.jsonl`. The result is the `srcLog` value.
3. Append `/did-witness.json` to the value from step 1. The result is the
   `srcWitness` value.

The same rule applies to every scheme and to a `src` value with no path; no
`/.well-known/` location is used.

The dereferencer passes both values; as defined in [The `srcLog` and
`srcWitness` Options](#the-srclog-and-srcwitness-options), the
[[ref: Resolver]] retrieves the witness proofs file only if it needs it.

For an `https` value with a path, a `src` value names the same location as
the domain and path of a `did:webvh` DID.

::: example

`src` values and the corresponding `srcLog` values:

`https://example.com` --> `https://example.com/did.jsonl`

`https://example.com/.well-known` --> `https://example.com/.well-known/did.jsonl` (a `/.well-known/` location is used only when it is in the `src` value)

`https://example.com/dids/issuer` --> `https://example.com/dids/issuer/did.jsonl`

`https://example.com:3000/dids/issuer` --> `https://example.com:3000/dids/issuer/did.jsonl`

`https://example.com/users/{SCID}` --> `https://example.com/users/{SCID}/did.jsonl`

`ipns://k51qzi5uqu5d.../dids/issuer` --> `ipns://k51qzi5uqu5d.../dids/issuer/did.jsonl`

`ipfs://bafybei.../` --> `ipfs://bafybei.../did.jsonl`

:::

Because a location can hold only one `did.jsonl`, a [[ref: DID Controller]]
publishing several `did:vh` [[ref: DID Logs]] in one place **SHOULD** give
each its own location (a URL path). Using the [[ref: SCID]] as the last path
segment is a simple way to do so.

#### Update (Rotate)

To update a DID, the [[ref: DID Controller]] generates a new, verifiable
[[ref: DID Log Entry]], has it witnessed (if necessary), appends it to the
existing [[ref: DID Log]] (`did.jsonl`), and distributes the result. The process to generate a verifiable
[[ref: DID Log Entry]] follows a similar process to the
[Create](#create-register) process, as follows:

1. Make the desired changes to the [[ref: DIDDoc]]. The top-level `id` in the
   [[ref: DIDDoc]] **MUST** remain `did:vh:<SCID>`. See the [note on future
   use by `did:scid:vh`](#note-did-scid-vh-diddoc-id) for how this may change
   in a later version.
2. Define the [[ref: parameters]] JSON object to include the properties that
   affect the evolution of the DID. The `parameters` **MUST** be from those
   listed in the [`did:vh` DID Method Parameters](#didvh-did-method-parameters)
   section of this specification. Any [[ref: parameters]] defined in the JSON
   object override the previously active value, while any
   [[ref: parameters]] not included imply the existing values remain in
   effect. If no changes to the [[ref: parameters]] are needed, an empty JSON
   object `{}` **MUST** be used.
   - While all [[ref: parameters]] in the first [[ref: Log Entry]] take effect
     immediately, some types of [[ref: parameters]] defined in later
     [[ref: entries]] only take effect after the entry has been published.
     For example, rotating the keys authorized to update a DID or changing
     the [[ref: witnesses]] for a DID take effect only *after* the entry in
     which they are defined has been published.
3. Generate a preliminary [[ref: DID log entry]] JSON object containing the
   following properties:
   1. The value of `versionId` **MUST** be the value of `versionId` from the
      *previous* [[ref: DID log entry]].
   2. The `versionTime` value **MUST** be a string that is an
      [[ref: ISO8601]] format UTC timestamp in whole seconds. The time
      **MUST** be greater than the time of the previous [[ref: log entry]],
      and **MUST** be the time the entry will be retrieved by a
      [[ref: witness]] or [[ref: Resolver]], or before.
   3. The [[ref: parameters]] passed in as a JSON object.
   4. Set the `state` JSON object to be the new version of the
      [[ref: DIDDoc]].
4. Calculate the new `versionId` of the new [[ref: DID Log Entry]], including
   incrementing the version number integer and using the process described in
   the [Entry Hash Generation and
   Verification](#entry-hash-generation-and-verification) section of this
   specification.
5. Replace the value of the `versionId` property in the preliminary
   [[ref: DID Log Entry]] with the value produced in the previous step.
6. Generate a [[ref: Data Integrity]] proof on the [[ref: DID log entry]]
   using an authorized key, as defined in the [Authorized
   Keys](#authorized-keys) section of this specification, and the
   `proofPurpose` set to `assertionMethod`.
7. If [[ref: Key Pre-Rotation]] is being used, the hash of all `updateKeys`
   entries in the `parameters` property **MUST** match a hash in the array of
   `nextKeyHashes` [[ref: parameter]] from the previous [[ref: DID log]]
   entry, as defined in the [Pre-Rotation Key Hash Generation and
   Verification](#pre-rotation-key-hash-generation-and-verification) section
   of this specification.
8. The proof JSON object **MUST** be added as the value of the `proof`
   property in the [[ref: log entry]].
9. The entry **MUST** be made a [[ref: JSON Line]] by removing extra
   whitespace, adding a `\n` to the entry.
10. If the [[ref: DID Controller]] opts to use [[ref: witnesses]] for the
    DID, the [[ref: DID Controller]] **MUST** collect the [[ref: threshold]]
    of proofs from the DID's [[ref: witnesses]], and update the DID's
    witness proofs file. The updated witness proofs file **MUST** be made
    available **BEFORE** the updated [[ref: DID Log]] is distributed. See
    the [DID Witnesses](#did-witnesses) section of this specification.
11. The new [[ref: log entry]] **MUST** be appended to the existing contents
    of the [[ref: DID Log]].
12. The [[ref: DID Controller]] **SHOULD** make the updated
    [[ref: DID Log]] available through every [[ref: DID Log source]] where it
    previously made the log available: updating each web location, notifying
    [[ref: watchers]] (see [DID Watchers](#did-watchers)), and sending it to
    peers.

A [[ref: DID Controller]] **MUST** only distribute a [[ref: DID Log]] that
extends the log it most recently distributed: every distributed
[[ref: log entry]] is retained, and the new [[ref: log entry]]'s predecessor
is the last distributed [[ref: log entry]]. A [[ref: DID Controller]]
**MUST NOT** distribute, or give to any party, two different
[[ref: log entries]] with the same predecessor.

##### Applying Peer-to-Peer Updates

When [[ref: DID Logs]] are exchanged [peer-to-peer](#peer-to-peer-exchange),
each party holds the only copies of the other's log, and an update can carry
only the new entries. A party that receives new [[ref: DID log entries]] for a
[[ref: DID Log]] it holds **MUST** apply them as follows:

1. Skip any received entry whose `versionId` is identical to that of an entry
   it already holds. If a received entry has the same version number as a held
   entry but a different `versionId`, the update is duplicitous and **MUST**
   be rejected.
2. Append the remaining entries, in order, to a copy of the held
   [[ref: DID Log]].
3. Resolve the DID with that copy as `didLog`, the received witness proofs
   file (or, if none was received, the held one) as `didWitness`, and the
   `versionId` of the last appended entry as `versionId`.
4. If resolution succeeds, replace the held [[ref: DID Log]] and witness proofs
   file with the ones used in step 3. Otherwise, reject the update and keep the
   held files unchanged.

A party **SHOULD** keep a rejected update as evidence of duplicity.

No second copy of the log is needed to detect an update that does not extend
the held log. The [[ref: entry hash]] of each entry is calculated with the
`versionId` of the previous entry as input, so an appended entry verifies only
if it continues from the last held entry. Requesting the `versionId` of the
last appended entry in step 3 ensures that resolution fails, rather than
returning an earlier version, if any appended entry is invalid.

#### Deactivate (Revoke)

Deactivating a DID allows a [[ref: DID Controller]] to signal that the DID is
no longer being maintained or updated. There is an explicit approach to
deactivation that aligns with the [[spec: DID-CORE]] specification, and
another approach that a `did:vh` [[ref: DID Controller]] can use if the
[[spec: DID-CORE]] approach does not achieve their desired outcome.

To deactivate a `did:vh` DID per the [[spec:DID-CORE]] specification, the
[[ref: DID Controller]] **MUST** add to the [[ref: DID log entry]]
[[ref: parameters]] the property name and value `"deactivated": true`. That
entry is the last [[ref: log entry]] of the DID: a [[ref: Resolver]] **MUST**
treat any [[ref: log entry]] after it as failing verification. Once a DID is
deactivated, a [[ref: Resolver]] **MUST NOT** return the [[ref: DIDDoc]] and
**MUST** include `"deactivated": true` in the DID Document Metadata. Because
no entry can follow the deactivating entry, there is no need to also remove
the authorized keys when deactivating.

To resolve a prior version of a deactivated `did:vh` DID, a client can use the
`versionId`, `versionTime` or `versionNumber` [resolution
options](#didvh-resolution-options). When a prior version is resolved in this
way, the DID Document Metadata **MUST** include `"deactivated": true`.

A concern with using the [[spec: DID-CORE]] approach to deactivation is the
resolver requirement that the [[ref: DIDDoc]] not be returned for a
deactivated DID. A [[ref: DID Controller]] might want the final
[[ref: DIDDoc]] to continue to be resolved, while also signaling that the DID
is no longer being updated. An alternative is to set the `updateKeys` array
to empty (`[]`) without setting the `deactivated` [[ref: parameter]] to
`true`. The final [[ref: DIDDoc]] then continues to be returned by default,
and no further entry can be signed: a [[ref: Resolver]] treats any entry after
it as failing verification, as defined in [Authorized
Keys](#authorized-keys), just as it does any entry after a deactivating
entry. If the DID is using [[ref: pre-rotation]], this takes two
[[ref: DID log entries]]: the first to stop the use of pre-rotation by setting
`nextKeyHashes` to `[]`, and the second to set `updateKeys` to `[]`. One
entry is not enough, because an entry verified with [[ref: pre-rotation]]
active is signed by a key in its own `updateKeys`, so an entry with empty
`updateKeys` could not be signed.

Unlike `did:webvh`, removing a published [[ref: DID Log]] does not effectively
deactivate a `did:vh` DID, because copies may be held by [[ref: watchers]],
peers, and other sources. A [[ref: DID Controller]] that wants to retire a DID
**SHOULD** deactivate it explicitly and distribute the final log, so that all
sources report the deactivation. [[ref: Watchers]] holding a DID whose
published log has been removed **SHOULD** continue to cache its last known
valid state. See [Log Freshness and Duplicity](#log-freshness-and-duplicity)
for why a [[ref: Resolver]] may not see the deactivation immediately.

### Sourcing the DID Log

*This section is non-normative.*

A `did:vh` [[ref: Resolver]] resolves the [[ref: DID Log]] it is given. Before
calling `resolve`, the client therefore needs to *source* the [[ref: DID Log]]: it
either obtains the [[ref: DID Log]] and witness proofs themselves, which it
passes directly in `didLog` and `didWitness`, or learns references to where
they are stored, which it passes in `srcLog` and `srcWitness`. In this specification, "source"
covers both outcomes, while "obtain" and "retrieve" mean getting the files
themselves. How the client sources the log depends on how it received the
DID. Each place it can source the [[ref: DID Log]] from is a
[[ref: DID Log source]]; this section describes the common ones. They can be
combined, and a [[ref: DID Log]] sourced from any of them is verified by the
[[ref: Resolver]] in exactly the same way.

#### Known in Context

In many uses, a client already has the [[ref: DID Log]], or knows where to get
it, because of the context in which it received the DID. For example:

- **A log sent with the DID.** A protocol that carries a DID — such as a
  credential or presentation exchange — also carries the [[ref: DID Log]] and
  witness proofs, or web locations for them. The client passes them
  directly, or passes the web locations by reference in `srcLog` and
  `srcWitness`.
- **An ecosystem DID host.** An ecosystem requires its participants to publish
  their `did:vh` [[ref: DID Logs]] to a registry or database it operates. The
  client retrieves the files from the host and passes them directly, or
  passes the files' web locations on the host by reference in `srcLog` and
  `srcWitness`.
- **A previously resolved log.** The client kept the files from an earlier
  resolution — for example, as returned with the `includeDidLog` [resolution
  option](#didvh-resolution-options) — and passes them directly, or by reference to the local
  files where it keeps them — perhaps after checking another source for
  newer entries.
- **A DID URL with a `src` parameter.** The client received a DID URL such as
  `did:vh:<SCID>?src=https://example.com/dids/issuer`, and uses the location as
  described in [Reading `did:vh` DID URLs](#reading-didvh-did-urls).

#### From Watchers

A [[ref: watcher]] retrieves, verifies, archives and re-serves
[[ref: DID Logs]], indexed by [[ref: SCID]]. Because a `did:vh` DID is its
[[ref: SCID]], a client that knows of a [[ref: watcher]] holding the DID —
for example, one its ecosystem uses — can retrieve:

- the [[ref: DID Log]] with **GET `<WATCHER URL>/log?scid=<SCID>`**, and
- if [[ref: witnesses]] are active, the witness proofs file with
  **GET `<WATCHER URL>/witness?scid=<SCID>`**.

These are operations defined in [Watcher HTTP API
Operations](#watcher-http-api-operations). The client then passes the files
directly, or passes these operation URLs by reference in `srcLog` and
`srcWitness`, for example `"srcLog":
"https://watcher.example.org/log?scid=<SCID>"`.

Instead of retrieving from [[ref: watchers]] itself, a client can pass the
copy it has and ask the [[ref: Resolver]] to check it against
[[ref: watchers]], using the `checkWatchers` option for [[ref: watchers]]
listed in the log and the `extraWatchers` option for others. See [Comparing
Copies of a DID Log](#comparing-copies-of-a-did-log).

Once a client has resolved the DID, the `watchers` metadata lists the
[[ref: watchers]] the [[ref: DID Controller]] has chosen. A client can use them
to check for newer entries, for example when the copy it holds may be out of
date. A client is not limited to those [[ref: watchers]]: any party can run
its own [[ref: watcher]], or use one its ecosystem operates, and send it the
[[ref: DID Logs]] it relies on. See [Log Freshness and
Duplicity](#log-freshness-and-duplicity).

#### Peer-to-Peer Exchange

Two or more parties can use `did:vh` DIDs with one another by exchanging
[[ref: DID Logs]] directly, in the manner of `did:peer` [[spec:DID-PEER]],
without publishing them anywhere. Unlike `did:peer`, the DIDs keep all the
`did:vh` features, including key rotation and pre-rotation. Each party keeps
the other's [[ref: DID Log]] (and witness proofs file, if [[ref: witnesses]]
are active), and its client passes them directly, or by reference to the
[local files](#local-files) where it keeps them.

**Initial exchange.** The [[ref: DID Controller]] gives the other parties the
complete [[ref: DID Log]] from the first entry, and the witness proofs file if
[[ref: witnesses]] are active.

**Updates.** When the DID is updated, the [[ref: DID Controller]] sends each
party either the complete [[ref: DID Log]] or just the new
[[ref: DID log entry]]. Sending just the new entry keeps updates small, and
works when each party receives every update in turn; a party that has missed
an update cannot apply it, and needs the complete [[ref: DID Log]]. If the new
entry requires witnessing, the [[ref: DID Controller]] also sends the
complete current witness proofs file; otherwise that is optional. The
protocol used to send updates is outside the scope of this specification.

**Applying an update.** A party receiving new entries applies them as defined
in [Applying Peer-to-Peer Updates](#applying-peer-to-peer-updates). Because
each [[ref: entry hash]] chains to the previous entry, this rejects any entry
that does not continue from the log the party holds, without needing a second
copy of the log.

A `did:vh` DID used only peer-to-peer can be created for each relationship, so
that no identifier is shared between relationships.

#### From a Web Location

A client that learns a web location where the [[ref: DID Controller]]
publishes the [[ref: DID Log]] can pass the files there by reference in the
`srcLog` and `srcWitness` options, or retrieve them itself and pass them
directly. A client can learn such a location from a `src` DID URL query
parameter (deriving the file URLs as defined in [The `src` DID URL
Parameter](#the-src-did-url-parameter)), from
an `alsoKnownAs` entry in a previously resolved [[ref: DIDDoc]] (see
[Advertising DID Log Sources](#advertising-did-log-sources)), or from the
context in which it received the DID.

#### Advertising DID Log Sources

A [[ref: DID Controller]] can tell others where current copies of the
[[ref: DID Log]] are kept:

- The `watchers` [[ref: parameter]] lists [[ref: watchers]] that archive the
  log.
- The [[ref: DIDDoc]] `alsoKnownAs` property can include `did:vh` DID URLs for
  the same DID with a `src` parameter, one for each location where the
  [[ref: DID Log]] is published (a web location, or another scheme such as
  `ipns`), as in `did:scid`. Each location listed this way is expected to hold
  the same [[ref: DID Log]], although some may briefly lag others while an
  update is distributed.

```json
"alsoKnownAs": [
  "did:vh:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ?src=https://example.com/dids/issuer",
  "did:vh:QmfGEUAcMpzo25kF2Rhn8L5FAXysfGnkzjwdKoNPi615XQ?src=https://backup.example.org/{SCID}"
]
```

Both are available only after the DID has been resolved at least once, so
they serve to check for newer entries rather than to source the log in the
first place.

#### Log Freshness and Duplicity

A `did:vh` [[ref: DID Log]] has no single authoritative location, so a client
faces two risks that `did:webvh` handles by trusting the DID's web location:

- **Staleness.** A source may hold an older, shorter copy of the log, and so
  miss updates such as a key rotation or deactivation. A [[ref: Resolver]]
  verifies the log it is given, but cannot know whether newer entries exist
  elsewhere.
- **Duplicity.** A [[ref: DID Controller]], or an attacker holding its keys,
  could create two different extensions of the same log and show different
  ones to different parties.

For two verified copies of a [[ref: DID Log]] for the same DID, log B
**extends** log A if B contains every entry of A, in the same positions, with
the same `versionId`s. Since each `versionId` includes a hash chained to the
previous entry, it is enough to check that B contains an entry with the
`versionId` of A's last entry, at the same position.

A client that needs the current version of a DID can reduce the risk of
staleness by sourcing the log from a source expected to be current — such as
a [[ref: watcher]] listed in the log — rather than relying only on a cached
copy or a copy supplied by the party presenting the DID. The `ttl` metadata
indicates how long the [[ref: DID Controller]] expects a copy to stay current.

A client that obtains copies from more than one source can compare them. If
one extends the other, the longer is the more current. If neither extends the
other, the log is **duplicitous**, and no version of it can be relied on —
the copies agree on their earlier entries, but whoever created the fork could
have signed those too. The client can keep the conflicting entries as
evidence and report them, for example to [[ref: watchers]]. A client can also
have the [[ref: Resolver]] make these comparisons, with the `checkWatchers`,
`extraWatchers` and `minCopies` [resolution
options](#didvh-resolution-options); the [[ref: Resolver]] then fails with
`logForked` for every version if the log has been forked (see [Forked
Logs](#forked-logs)).

[[ref: Witnesses]] and [[ref: watchers]] address duplicity in different ways:
[[ref: witnesses]] are expected to approve only an entry that extends their
own copy of the log, but no one else can confirm that they do, while
[[ref: watchers]] detect conflicting copies directly, but only the copies they
see. For `did:vh` these matter more than for `did:webvh`, because there is no
web location to act as the reference copy. A relying party that runs its own
[[ref: watcher]], or uses one its ecosystem operates, gets a check that is
independent of the [[ref: DID Controller]]; `extraWatchers` lets it name such
[[ref: watchers]] when resolving. [[ref: DID Controllers]] of `did:vh` DIDs
that are relied on by many parties are encouraged to use [[ref: witnesses]]
and [[ref: watchers]], and parties relying on such DIDs are encouraged to use
[[ref: watchers]] they trust.

### DID Method Processes

The [DID Method Operations](#did-method-operations) reference several
processes that are executed during [[ref: DIDDoc]] generation and DID
resolution verification. Each of those processes is specified in the following
sections.

#### `did:vh` DID Method Parameters

All `did:vh` [[ref: Log entries]] contain the JSON object `parameters`. This
object defines the DID processing [[ref: parameters]] used by the
[[ref: DID Controller]] when creating the current and subsequent
[[ref: DID log entries]]. [[ref: Resolvers]] **MUST** use the same
[[ref: parameters]] to process the [[ref: DID Log]] to resolve the DID. The
`parameters` object **MUST** only include properties defined in the version of
the `did:vh` DID Method specification being used.

**General Rules for Parameters:**

- **Default Values**: When the `method` parameter (see below) sets the
  [[spec: semver]] version of this specification to be used for a DID, any
  parameter introduced by that version but not explicitly set in the same
  [[ref: log entry]] **MUST** assume the default value defined in this
  section. The `method` parameter appears in the first [[ref: log entry]],
  and can appear in later entries to upgrade the DID to a newer
  [[spec: semver]] version of the `did:vh` specification, as defined below.

- **Allowed Values**: Each parameter is constrained by the data type,
  structure and allowed values specified in this section. If a value does not
  conform, the parameter is invalid and **MUST** cause the
  [[ref: log entry]] to fail verification.

- **Deactivation**: Parameters that support deactivation (such as `witness`
  or `nextKeyHashes`) are set to defined values, described below, to indicate
  they are no longer active.

- The JSON `null` value **MUST NOT** be used to indicate default or
  deactivated values, as it removes the typing information required for
  proper interpretation.

::: example
An example of the `parameters` property in the first [[ref: DID Log]] entry:

```json
{
  "method": "did:vh:1.0",
  "scid": "{SCID}",
  "updateKeys": [
    "z82LkqR25TU88tztBEiFydNf4fUPn8oWBANckcmuqgonz9TAbK9a7WGQ5dm7jyqyRMpaRAe"
  ],
  "nextKeyHashes": [
    "enkkrohe5ccxyc7zghic6qux5inyzthg2tqka4b57kvtorysc3aa"
  ]
}
```

:::

The following lists the [[ref: parameters]], their data types, and enumerated
values.

- `method`: Specifies the `did:vh` [[spec: semver]] specification version to
  be used for processing the DID's log. Each acceptable value in turn defines
  what cryptographic algorithms are permitted for the current and subsequent
  [[ref: DID log entries]]. An update to the specification version in the
  middle of a [[ref: DID Log]] could introduce new [[ref: parameters]].
  - **MUST** appear in the first [[ref: log entry]] and **MUST** be one of the
    enumerated acceptable values below.
  - A `method` value **MUST** be `did:vh:` followed by a major and minor
    version number, each a non-negative integer with no leading zeros,
    separated by `.` (for example, `did:vh:1.0`). An entry with a `method`
    value not of this form **MUST** fail verification.
  - A [[ref: Resolver]] **MUST NOT** process an entry whose `method` value is
    of this form but is not one of the acceptable values for the version(s)
    of this specification the [[ref: Resolver]] supports, or any later entry.
    Such an entry is *unsupported*, and the result is as defined in [Results
    of Processing the Log](#results-of-processing-the-log). Unknown values
    **MUST NOT** be silently downgraded, defaulted, or ignored.
  - If not present in later entries, the previous value continues to be
    active.
  - **MAY** appear in later entries to upgrade the spec version. A change to a
    *lower* version than currently active **MUST** cause the entry to fail
    verification.
  - Acceptable values:
    - `did:vh:1.0`
      - Permitted hash algorithms: `SHA-256` [[spec:rfc6234]] (multihash code
        `0x12`) **only**. Any other algorithm **MUST** cause the entry to
        fail verification.
      - Permitted [[ref: Data Integrity]] cryptosuites for **both** log-entry
        proofs **and** witness proofs: exactly `eddsa-jcs-2022`
        [[spec:di-eddsa-v1.0]]. [[ref: Resolvers]] **MUST** verify the proof's
        `cryptosuite` property; an absent, mismatched, or non-conformant
        `cryptosuite` **MUST** cause the entry to fail verification. Verifying only
        `proofPurpose` is **insufficient**.
        - [[ref: witness]] [[ref: did:key]] identifiers **MUST** use a key
          compliant with the `eddsa-jcs-2022` cryptosuite defined in
          [[spec:di-eddsa-v1.0]].
- `scid`: The [[ref: SCID]] value for the DID.
  - **MUST** appear in the first [[ref: log entry]].
  - **MUST NOT** appear in later [[ref: log entries]].
- `updateKeys`: A JSON array of [[ref: multikey]] formatted public keys
  associated with the private keys that are authorized to sign the log entries
  that update the DID. See the [Authorized Keys](#authorized-keys) section of
  this specification for additional details.
  - This property **MUST** appear in the first [[ref: log entry]] and **MAY**
    appear in subsequent entries.
  - If not present in later [[ref: DID log entries]], the previous value
    continues to apply.
  - A key from the active `updateKeys` array **MUST** be used to authorize
    each [[ref: log entry]], where active is defined as follows.
    - In the first [[ref: log entry]], the active `updateKeys` is the one
      defined in that entry.
    - In all other [[ref: log entries]] *without* [[ref: Key Pre-Rotation]]
      active, the active `updateKeys` is that of the most recent **prior**
      [[ref: log entry]].
    - In all other [[ref: log entries]] *with* [[ref: Key Pre-Rotation]]
      active, the active `updateKeys` is that of the most current
      [[ref: log entry]].
    - Whether [[ref: Key Pre-Rotation]] is active for an entry is defined in
      [Authorized Keys](#authorized-keys).
  - `updateKeys` **MAY** be set to an empty array `[]` to stop further
    updates without deactivating the DID. Any [[ref: log entry]] after one
    whose `updateKeys` is in effect as `[]` **MUST** fail verification, as
    defined in [Authorized Keys](#authorized-keys). See the
    [deactivate](#deactivate-revoke) section of this specification for more
    details.
- `nextKeyHashes`: A JSON array of strings that are hashes of
  [[ref: multikey]] formatted public keys that **MAY** be added to the
  `updateKeys` list in the next [[ref: log entry]]. At least one entry of
  `nextKeyHashes` **MUST** be added to the next `updateKeys` list.
  - The process for generating the hashes and additional details for using
    [[ref: pre-rotation]] are defined in the [Pre-Rotation Key Hash Generation
    and Verification](#pre-rotation-key-hash-generation-and-verification)
    section of this specification.
  - If not set in the first [[ref: log entry]], its value defaults to an empty
    array (`[]`).
  - It can be omitted from a later [[ref: log entry]] only while
    [[ref: Key Pre-Rotation]] is not active, and its value then remains
    `[]`. Its value is never inherited while pre-rotation is active, because
    it **MUST** then be present in every entry (see below).
  - Once an entry sets `nextKeyHashes` to a non-empty array,
    [[ref: Key Pre-Rotation]] is active for every subsequent entry, as defined
    in [Authorized Keys](#authorized-keys). While active, **both**
    `nextKeyHashes` **AND** `updateKeys` **MUST** be present as explicit
    properties in every subsequent [[ref: log entry]] until pre-rotation is
    deactivated (by setting `nextKeyHashes` to `[]`). A subsequent entry that
    omits either **MUST** fail verification, even if the omission would
    otherwise inherit the previous value. Inheritance of
    `updateKeys` is **never** permitted while pre-rotation is active — that
    bypass would defeat the pre-rotation commitment.
  - While [[ref: Key Pre-Rotation]] is active, **every** [[ref: multikey]] in
    the current entry's `updateKeys` (not only those that appear new)
    **MUST** have its hash in the previous entry's `nextKeyHashes`.
  - A [[ref: DID Controller]] **MAY** include extra hashes in the
    `nextKeyHashes` array that are not subsequently used in an `updateKeys`
    entry. Any unused hashes in `nextKeyHashes` arrays are ignored.
  - The value of `nextKeyHashes` **MAY** be set to an empty array (`[]`) to
    deactivate [[ref: pre-rotation]]. For additional details about turning
    off [[ref: pre-rotation]], see the [Pre-Rotation Key Hash Generation and
    Verification](#pre-rotation-key-hash-generation-and-verification) section
    of this specification.
- `witness`: A JSON object declaring the set of witnesses and threshold number
  of witness proofs required to update the DID. For details of this data and
  its usage in the DID update approval process, see the [DID
  Witnesses](#did-witnesses) section of this specification.
  - Defaults to `{}` if not set in the first [[ref: log entry]].
  - If not set in other [[ref: log entries]], its value is retained from the
    most recent prior value.
  - If the `witness` property is updated from `{}`, the change is immediately
    active, and the corresponding [[ref: log entry]] **MUST** be
    [[ref: witnessed]].
  - The `witness` [[ref: parameter]] **MAY** be set to `{}` to indicate that
    witnesses are not (or no longer) being used. If witnesses are active when
    the `witness` [[ref: parameter]] is set to `{}`, that [[ref: log entry]]
    **MUST** be [[ref: witnessed]].
- `watchers`: An optional entry whose value is a JSON array containing a list
  of URLs ([[spec:rfc9110]]) of [[ref: watchers]] that have notified the
  [[ref: DID Controller]] that they are willing to watch the DID. See the [DID
  Watchers](#did-watchers) section of this specification for more details.
  - Defaults to `[]` if not set in the first [[ref: log entry]].
  - If not set in other [[ref: log entries]], its value is retained from the
    most recent prior value.
  - **MAY** be set to an empty array `[]` to indicate that watchers are not
    (or no longer) being used.
- `deactivated`: A JSON boolean that indicates whether the DID has been
  deactivated. See the [deactivate (revoke)](#deactivate-revoke) section of
  this specification for more details.
  - Defaults to `false` if not set in the first [[ref: DID log entry]].
  - If set to `true`, the DID is deactivated, and the entry that sets it
    **MUST** be the last [[ref: log entry]] of the DID.
- `ttl`: An unsigned integer that indicates how long, in seconds, the
  [[ref: DID Controller]] expects a [[ref: Resolver]] or client to cache the
  resolved `did:vh` DID before refreshing. It provides guidance from the [[ref: DID Controller]] on cache
  duration, with a range of 0 to 2<sup>31</sup> - 1 (2,147,483,647). The
  parameter is analogous to the `TTL` parameter used in DNS
  [[spec: rfc2181]]. Caching a `did:vh` can be valuable in places where the
  business rules require resolving a number of DID URLs for the same DID. For example, a client might want to call the
  [[ref: Resolver]] to get the current [[ref: DIDDoc]], and then make repeated
  calls to get all of the previous versions of the [[ref: DIDDoc]]. By caching
  the [[ref: DIDDoc]] state, the [[ref: Resolver]] would not have to retrieve
  and process the [[ref: DID Log]] on each call. For `did:vh`, `ttl` is also a
  hint about how long a copy of the [[ref: DID Log]] can be expected to stay
  current (see [Log Freshness and Duplicity](#log-freshness-and-duplicity)).
  - Defaults to `3600` (1 hour) if not set in the first
    [[ref: DID log entry]].
  - If set to `0`, indicates that the [[ref: DID Controller]] expects the DID
    not to be cached.

`did:vh` does not use the `did:webvh` `portable` [[ref: parameter]]. An entry
that includes it **MUST** fail verification.

#### Cryptographic Agility

The `did:vh` DID method is designed to support cryptographic agility—the
ability to adapt to evolving cryptographic algorithms and suites over time
without breaking compatibility or requiring global coordination.

Cryptographic agility in `did:vh` is achieved through the following
mechanisms:

- **Self-describing cryptographic formats:** All cryptographically generated
  data (e.g., hashes and signatures) use formats that encode the algorithm
  used. Hashes use the [[ref: Multihash]] format, and signatures use
  [[ref: Data Integrity]] proofs. These formats allow verifiers to determine
  the algorithm from the data itself, enabling flexible and extensible
  support.

- **Specification versioning via the `method` parameter:** The `method`
  [[ref: parameter]] specifies the version of the `did:vh` specification used
  for the entry that sets it and for subsequent [[ref: log entries]]. It is
  set in the first [[ref: log entry]], and a later entry can set it again to
  adopt a newer (but not older) version of the specification, as defined in
  [`did:vh` DID Method Parameters](#didvh-did-method-parameters). This allows long-lived
  DIDs to transition to updated algorithm sets over time.

- **Version-specific algorithm policies:** Each version of the `did:vh`
  specification defines the permitted cryptographic algorithms and suites for
  that version. This constrains the algorithms [[ref: DID Controllers]] can
  use and limits the verification requirements placed on [[ref: Resolvers]].
  For example, the `did:vh:1.0` version permits only one hash algorithm and
  one [[ref: Data Integrity]] cryptosuite, while future versions can change or
  expand this set.

- **Response to cryptographic vulnerabilities:** If flaws are identified in a
  cryptographic algorithm permitted by a given version of the specification, a
  new version of the `did:vh` specification will be released that removes or
  replaces the compromised algorithms. [[ref: DID Controllers]] can then
  rotate to the newer version by updating the `method` parameter in a new log
  entry.

This design allows `did:vh` to remain interoperable and verifiable across
cryptographic eras while minimizing the burden on [[ref: Resolvers]] and
preserving backward compatibility where safe to do so.

#### SCID Generation and Verification

The [[ref: self-certifying identifier]] or `SCID` is a required
[[ref: parameter]] in the first [[ref: DID log entry]] and is the hash of the
DID's inception event. For `did:vh`, the [[ref: SCID]] is also the whole of
the [method-specific identifier](#method-specific-identifier).

##### Generate SCID

To generate the [[ref: SCID]] for a `did:vh` DID, the
[[ref: DID Controller]] **MUST** execute the following function:

 `base58btc(multihash(JCS(preliminary log entry with placeholders), <hash algorithm>))`

Where:

1. The `preliminary log entry with placeholders` consists of the following
   pre-publication JSON object of what will become the first
   [[ref: log entry]]. The placeholder is the literal string "`{SCID}`".

   - The `versionId` entry, which **MUST** be `{SCID}`.
   - The `versionTime` entry, which **MUST** be a string that is the current
     time in UTC [[ref: ISO8601]] format in whole seconds, e.g.,
     `"2024-04-05T07:32:58Z"`
   - The complete `parameters` for the initial [[ref: log entry]] as defined
     by the [[ref: DID Controller]], with the placeholder wherever the
     [[ref: SCID]] will eventually be placed.
   - The `state` JSON object with the value being the initial [[ref: DIDDoc]]
     with placeholders (the literal string "`{SCID}`") wherever the
     [[ref: SCID]] will eventually be placed in the [[ref: DIDDoc]],
     including in the DID string `did:vh:{SCID}`.

2. `JCS` is an implementation of the [[ref: JSON Canonicalization Scheme]]
   [[spec:rfc8785]]. It outputs a canonicalized representation of its JSON
   input.
3. `multihash` is an implementation of the [[ref: multihash]] specification.
   Its output is a hash of the input using the associated `<hash algorithm>`,
   prefixed with a hash algorithm identifier and the hash size.
4. `<hash algorithm>` is the hash algorithm used by the
   [[ref: DID Controller]]. The hash algorithm **MUST** be one listed in the
   [parameters](#didvh-did-method-parameters) defined by the version of the
   `did:vh` specification being used by the [[ref: DID Controller]].
5. `base58btc` is an implementation of the [[ref: base58btc]] function. Its
   output is the plain base58btc encoding of its input, without a
   [[ref: multibase]] prefix.

##### Verify SCID

To verify the [[ref: SCID]] of a `did:vh` DID being resolved, the
[[ref: Resolver]] **MUST** execute the following process:

1. Extract the first [[ref: DID log entry]] and use it for the rest of the
   steps in this process.
2. Extract the `scid` property value from the [[ref: parameters]] in the
   [[ref: DID log entry]].
3. Determine the hash algorithm used by the [[ref: DID Controller]] from the
   [[ref: multihash]] `scid` value.
   - The hash algorithm **MUST** be one listed in the
     [parameters](#didvh-did-method-parameters) defined by the version of the
     `did:vh` specification being used by the [[ref: DID Controller]] based on
     the active `method` [[ref: parameters]] property.
4. Remove the [[ref: data integrity]] proof property from the
   [[ref: DID log entry]].
5. Replace the `versionId` property value with the literal `"{SCID}"`.
6. Treat the resulting [[ref: log entry]] as a string and do a text
   replacement of the `scid` value from Step 2 with the literal string
   `{SCID}`.
7. Use the result and the hash algorithm (from Step 3) as input to the
   function defined in the [Generate SCID](#generate-scid) section (above).
8. The output string **MUST** match the `scid` extracted in Step 2. If not,
   the first entry fails verification, and resolution fails with the
   `invalidDid` error.

#### Entry Hash Generation and Verification

The `entryHash` follows the version number and dash character `-` in the
`versionId` property in each DID log entry. Each `entryHash` is calculated
across its [[ref: log entry]], excluding the [[ref: Data Integrity]] proof.
The `versionId` used in the input to the hash is a predecessor value to the
current [[ref: log entry]], ensuring that the [[ref: entries]] are
cryptographically "chained" together in a microledger. For the first
[[ref: log entry]], the predecessor `versionId` is the SCID (itself a hash),
while for all other entries it is the `versionId` property from the previous
log entry.

##### Generate Entry Hash

To generate the [[ref: entry hash]] and `versionId` for a `did:vh`
[[ref: log entry]], the [[ref: DID Controller]] **MUST** execute the following
process:

1. Start with the preliminary [[ref: log entry]], without a
   [[ref: Data Integrity]] `proof`.
2. Set its `versionId` to the predecessor value: for the first
   [[ref: log entry]], the [[ref: SCID]] of the DID; for every other entry,
   the `versionId` of the previous [[ref: log entry]].
3. Calculate the [[ref: entry hash]] as
   `base58btc(multihash(JCS(entry), <hash algorithm>))`, where:
   1. `entry` is the data from the previous step.
   2. `JCS` is an implementation of the [[ref: JSON Canonicalization Scheme]]
      ([[spec:rfc8785]]). Its output is a canonicalized representation of its
      input.
   3. `multihash` is an implementation of the [[ref: multihash]]
      specification. Its output is a hash of the input using the associated
      `<hash algorithm>`, prefixed with a hash algorithm identifier and the
      hash size.
   4. `<hash algorithm>` is the hash algorithm used by the
      [[ref: DID Controller]]. The hash algorithm **MUST** be one listed in
      the [parameters](#didvh-did-method-parameters) defined by the version of
      the `did:vh` specification being used by the [[ref: DID Controller]].
   5. `base58btc` is an implementation of the [[ref: base58btc]] function.
      Its output is the plain base58btc encoding of its input, without a
      [[ref: multibase]] prefix.
4. Set the `versionId` of the entry to `<version number>-<entry hash>`, where
   the version number is `1` for the first [[ref: log entry]] and the previous
   entry's version number plus one for every other entry.

The [[ref: Data Integrity]] proof is then added to the entry, as described in
[Create (Register)](#create-register) and [Update (Rotate)](#update-rotate).

The following is an example of a preliminary [[ref: log entry]] that is
processed to produce an [[ref: entry hash]]. As this is a first entry in a
[[ref: DID Log]], the input `versionId` is the [[ref: SCID]] of the DID.

```json
{"versionId": "QmdmPkUdYzbr9txmx8gM2rsHPgr5L6m3gHjJGAf4vUFoGE", "versionTime": "2025-04-01T17:39:50Z", "parameters": {"witness": {"threshold": 2, "witnesses": [{"id": "did:key:z6Mkkc51mg2vpQzKWAbWQZupeGYhowaBjYkmvcKMTqteqHB4"}, {"id": "did:key:z6MkuDdJdKLCgwZuQuEi9xG6LVgJJ9Tebr74CXPYPSumqgJs"}, {"id": "did:key:z6MkoSWmQyp4fTk4ZQy4KUsss9dFX51XfEUzKKKj1J1JUsrF"}]}, "updateKeys": ["z6MkgzBDcBFV3sk4ypPE5YXMZHmS213A3HpYY2LmcVKV15jr"], "nextKeyHashes": ["QmZreDcjvWEpyRFznQeExWNCsvMLk5i59AcRJJuQC8UodJ"], "method": "did:vh:1.0", "scid": "QmdmPkUdYzbr9txmx8gM2rsHPgr5L6m3gHjJGAf4vUFoGE"}, "state": {"@context": ["https://www.w3.org/ns/did/v1"], "id": "did:vh:QmdmPkUdYzbr9txmx8gM2rsHPgr5L6m3gHjJGAf4vUFoGE"}}
```

Resulting [[ref: entryHash]]: `QmNdBb5MGCjawxfCRYKY31deSYWpqp2rxmeRH6acJAsCMe`

##### Verify The Entry Hash

To verify the `entryHash` for a given `did:vh` [[ref: DID log entry]], a
[[ref: Resolver]] **MUST** execute the following process:

1. Extract the `versionId` in the [[ref: DID log entry]], and remove from it
   the version number and dash prefix, leaving the log entry `entryHash`
   value.
2. Determine the hash algorithm used by the [[ref: DID Controller]] from the
   [[ref: multihash]] `entryHash` value.
   - The hash algorithm **MUST** be one listed in the
     [parameters](#didvh-did-method-parameters) defined by the version of the
     `did:vh` specification being used by the [[ref: DID Controller]] based on
     the `method` [[ref: parameters]] property set in the current or most
     recent prior [[ref: log entry]].
3. Remove the [[ref: Data Integrity]] `proof` from the [[ref: log entry]].
4. Set the `versionId` in the entry object to be the `versionId` from the
   previous [[ref: log entry]]. If this is the first entry in the log, set the
   value to `<scid>`, the value of the [[ref: SCID]] of the DID.
5. Calculate the hash string as
   `base58btc(multihash(JCS(entry), <hash algorithm>))`, where:
   1. `entry` is the data from the previous step.
   2. `JCS` is an implementation of the [[ref: JSON Canonicalization Scheme]]
      ([[spec:rfc8785]]). Its output is a canonicalized representation of its
      input.
   3. `multihash` is an implementation of the [[ref: multihash]]
      specification. Its output is a hash of the input using the associated
      `<hash algorithm>`, prefixed with a hash algorithm identifier and the
      hash size.
   4. `<hash algorithm>` is the hash algorithm from Step 2.
   5. `base58btc` is an implementation of the [[ref: base58btc]] function. Its
      output is the plain base58btc encoding of its input, without a
      [[ref: multibase]] prefix.
6. Verify that the calculated value matches the extracted `entryHash` value
   from Step 1. If not, the entry fails verification.

#### Authorized Keys

Each entry in the [[ref: DID Log]] **MUST** include a [[ref: Data Integrity]]
`proof` where, at minimum:

1. `type` is `DataIntegrityProof`,
2. `cryptosuite` **MUST** be one listed in the
   [parameters](#didvh-did-method-parameters) defined by the version of the
   `did:vh` specification being used by the [[ref: DID Controller]] based on
   the active `method` [[ref: parameters]] property.
3. `proofPurpose` is `assertionMethod`,
4. `verificationMethod` resolves to a [[ref: multikey]] that appears verbatim
   in the **active** `updateKeys`.

An entry whose proof fails *any* check **MUST** fail verification. A
structurally-valid signature over a *different* cryptosuite than allowed by
the active `method` parameter **MUST NOT** be accepted.

The authorized verification keys for `did:vh` are the [[ref: multikey]]
formatted public keys in the **active** `updateKeys` list from the
`parameters` property of the [[ref: log entries]]. Any of the authorized
verification keys **MAY** be referenced in the [[ref: Data Integrity]] proof.

For the first [[ref: log entry]] the **active** `updateKeys` list is the one
in that first [[ref: log entry]].

A [[ref: Resolver]] of the DID **MUST** verify the signature and the key used
for signing each [[ref: DID Log]] entry **MUST** be one from the list of
active `updateKeys`. If not, the entry fails verification.

The **active** `updateKeys` for subsequent [[ref: entries]] depends on whether
[[ref: Pre-Rotation]] is active or not. [[ref: Pre-rotation]] is active for an
entry if, and only if, the `nextKeyHashes` in effect *before* that entry — as
set by an earlier entry — is non-empty. An entry that first sets a non-empty
`nextKeyHashes` is therefore verified as defined in [No Key
Pre-Rotation](#no-key-pre-rotation), not as defined in [Pre-Rotation
Active](#pre-rotation-active).

If the active `updateKeys` for an entry is empty, no key is authorized to sign
it, and the entry **MUST** fail verification. In particular, once an entry
sets `updateKeys` to `[]` while [[ref: pre-rotation]] is not active, every
later entry fails verification, including one that sets a non-empty
`nextKeyHashes` or new `updateKeys`.

A `did:vh` [[ref: DID Log]] has no location whose control an attacker would
also need in order to publish an update (see [No Authoritative
Location](#no-authoritative-location)), so [[ref: DID Controllers]]
**SHOULD** use [[ref: pre-rotation]].

##### No Key Pre-Rotation

For all subsequent [[ref: entries]], the **active** list is the most recent
`updateKeys` **before** the [[ref: log entry]] to be verified. Thus, the
general case is that each [[ref: log entry]] is signed by the keys from the
**previous** [[ref: log entry]]. Once a [[ref: log entry]] containing an
`updateKeys` list is published, that `updateKeys` becomes the active list, and
previous `updateKeys` are ignored.

##### Pre-Rotation Active

For all subsequent [[ref: entries]], the **active** list is the `updateKeys`
from the **current** [[ref: log entry]] to be verified. Thus, the general case
is that each [[ref: log entry]] is signed by the keys from the **current**
[[ref: log entry]].

#### Pre-Rotation Key Hash Generation and Verification

Pre-rotation requires a [[ref: DID Controller]] to commit to the authorization
keys that will be used ("rotated to") in the next [[ref: log entry]] for
updating the [[ref: DIDDoc]]. The purpose of committing to future keys is that
if the currently authorized keys are compromised by an attacker, the attacker
cannot take control of the DID by using the compromised keys to
rotate to new keys the attacker controls. Assuming the attacker has not also
compromised the committed key pairs, they cannot rotate the authorization keys
without detection. See the non-normative section about [Using Pre-Rotation
Keys](https://didwebvh.info/latest/implementers-guide/prerotation-keys/) in the
`did:webvh` Implementer's Guide, which applies equally to `did:vh`, for
additional guidance.

As described in the [parameters](#didvh-did-method-parameters) section of this
specification, a [[ref: DID Controller]] **MAY** include the
[[ref: parameter]] `nextKeyHashes` with a non-empty list in any
[[ref: DID log entry]] to activate the [[ref: pre-rotation]] feature. When
[[ref: pre-rotation]] is active, all [[ref: multikey]] representations of the
public keys in the `updateKeys` [[ref: parameters]] property in other than the
initial version of the [[ref: DID log entry]] **MUST** have their hash in the
`nextKeyHashes` array from the previous [[ref: DID log entry]]. If not, the
entry fails verification.

A [[ref: DID Controller]] **MAY** turn off the use of pre-rotation by setting the
[[ref: parameter]] `nextKeyHashes` to `[]` (empty array) in any
[[ref: DID log entry]]. If there is an active set of `nextKeyHashes` at the
time, the pre-rotation requirements remain in effect for the
[[ref: DID Log entry]]. The subsequent [[ref: DID Log entry]] **MUST** use the
non-pre-rotation rules.

To create a hash to be included in the `nextKeyHashes` array, the
[[ref: DID Controller]] **MUST** execute the following process for each
possible future authorization key.

1. Generate a new key pair. The key type **MUST** be one that can be used as a
   `did:vh` authorization key.
2. Generate a [[ref: multikey]] representation of the public key of the new
   key pair.
3. Calculate the hash string as `base58btc(multihash(multikey, <hash algorithm>))`, where:
   1. `multikey` is the [[ref: multikey]] representation of the public key
      from Step 2.
   2. `multihash` is an implementation of the [[ref: multihash]]
      specification. Its output is a hash of the input using the associated
      `<hash algorithm>`, prefixed with a hash algorithm identifier and the
      hash size.
   3. `<hash algorithm>` is the hash algorithm used by the
      [[ref: DID Controller]]. The hash algorithm **MUST** be one listed in the
      [parameters](#didvh-did-method-parameters) defined by the version of the
      `did:vh` specification being used by the [[ref: DID Controller]].
   4. `base58btc` is an implementation of the [[ref: base58btc]] function.
      Its output is the plain base58btc encoding of its input, without a
      [[ref: multibase]] prefix.
4. Insert the calculated hash into the `nextKeyHashes` array being built up
   within the [[ref: parameters]] property.
5. The generated key pair **SHOULD** be safely stored so that it can be used
   in the next [[ref: log entry]] to become a DID authorization key. At that
   time, the [[ref: multikey]] representation of the public key will be
   inserted into the `updateKeys` property in the [[ref: parameters]] and the
   private key can be used to sign the [[ref: log entry]]'s DID update
   authorization proofs.

A [[ref: DID Controller]] **MAY** include extra entries (for keys or just
random strings) in a `nextKeyHashes` array.

After rotating from a pre‑rotation public key, the corresponding private key
**SHOULD** be treated as **spent** and **securely destroyed**. Reusing a
revealed pre‑rotation key is strongly discouraged because it weakens the
intended containment and forward‑security properties of pre‑rotation.

When processing other than the first [[ref: DID log entry]] where the
[[ref: pre-rotation]] feature is active, a `did:vh` [[ref: Resolver]]
**MUST**:

1. For each [[ref: multikey]] in the `updateKeys` property in the
   `parameters` of the [[ref: log entry]], calculate the hash and hash
   algorithm for the [[ref: multihash]] [[ref: multikey]].
2. The hash algorithm **MUST** be one listed in the
   [parameters](#didvh-did-method-parameters) defined by the version of the
   `did:vh` specification being used by the [[ref: DID Controller]].
3. The resultant hash **MUST** be in the `nextKeyHashes` array from the
   previous [[ref: log entry]] prior to being processed. If not, the entry
   fails verification.
4. A new `nextKeyHashes` list **MUST** be in the `parameters` of the
   [[ref: log entry]] currently being processed. If not, the entry fails
   verification.

#### DID Witnesses

The [[ref: witness]] process for a `did:vh` DID provides a way for
collaborators to work with the [[ref: DID Controller]] to "witness" the
creation of new versions of the DID. This specification defines the technical
mechanism for using [[ref: witnesses]]. Governance and policy questions about
when and how to use the technical mechanism are outside the scope of this
specification.

Witnesses can prevent a [[ref: DID Controller]] from updating or removing
versions of a DID without detection by the witnesses. They are also a
mitigation against a malicious actor who has compromised a
[[ref: DID Controller]]'s authorization key(s). A `did:vh` [[ref: DID Log]]
has no single location whose control the attacker would also need, so with
the keys alone the attacker could distribute an update through any
[[ref: DID Log source]]. With [[ref: witnesses]] active, the attacker cannot
create a valid update, or rewrite the previous history, without also having
compromised a sufficient number of [[ref: witnesses]]. [[ref: DID Controllers]]
**SHOULD** use [[ref: witnesses]] for DIDs relied on by many parties.

This protection depends on [[ref: witnesses]] behaving as this specification
requires — in particular, approving only an entry that extends their own copy
of the [[ref: DID Log]] (see [Witnessing a DID Version
Update](#witnessing-a-did-version-update)). A witness proof shows only that a
[[ref: witness]] approved an entry; a [[ref: Resolver]] cannot confirm how the
[[ref: witness]] reached that decision. How far [[ref: witnesses]] are trusted
is therefore a matter for the governance of the ecosystem. [[ref: Watchers]]
provide a way to detect conflicting logs that does not depend on that trust;
see [DID Watchers](#did-watchers) and [Log Freshness and
Duplicity](#log-freshness-and-duplicity).

##### Witness Lists

The list of DIDs that witness DID updates is defined in the `witness`
parameter, as described in the [Parameters](#didvh-did-method-parameters)
section of this specification. After the first `witness` parameter has been
set to other than `{}` (empty object) in a [[ref: DID log entry]], and while
there are active witnesses, the [[ref: DID Controller]] **MUST** obtain valid
proofs from a [[ref: threshold]] of the active witnesses for each
[[ref: DID log entry]] before distributing it. If a [[ref: DID log entry]]
contains a new (replacement) list of witnesses (by including a new `witness`
[[ref: parameter]]) that new list becomes active **AFTER** the new
[[ref: DID log entry]] has been published. Such a replacement **MAY** be a
`{}` (empty object). Once the `witness` attribute set to `{}` becomes active,
updates to the DID are not [[ref: witnessed]].

##### Witness DIDs and Reputation

Since `did:vh` witness DIDs are `did:key` DIDs, there is not an explicitly
published identifier for each witness. If there is a need in an ecosystem to
identify who the witnesses are, a mechanism can be defined by the
governance of the ecosystem, such as the entry of the DID in a trust registry.
Such mechanisms are outside the scope of this specification.

When a [[ref: did:key]] DID is used in any `did:vh` context — as a witness
`id`, as a `verificationMethod` controller, or as an `assertionMethod`
reference in a [[ref: Data Integrity]] proof — the [[ref: did:key]]
specification **MUST** be followed. Notably, the multibase value in the
method-specific identifier (the **body** of the DID) **MUST** equal the
multibase value in any fragment identifier (if present) that references the
lone verification method within the DID. For `did:key:z6MkABC...#z6MkABC...`,
body and fragment **MUST** be byte-for-byte equal. Verifiers **MUST** reject
any reference where they differ, because the body authoritatively defines the
public key while the fragment is the Verification Method id; permitting
divergence would let an attacker claim a proof was made by `did:key:A` while
actually signing with `did:key:B`.

##### The `witness` Parameter

The `witness` element in a [[ref: parameters]] object of a
[[ref: DID Log entry]] has the following data structure:

```json
"witness" : {
  "threshold": n,
  "witnesses" : [
      {
         "id": "<did:key DID of witness>"
      }
   ]
}
```

where:

- `threshold`: a positive integer (JSON number, no fractional part, value ≥ 1)
  that **MUST** be attained or surpassed by the count of **distinct** verified
  [[ref: witness]] approvals for a [[ref: DID log entry]] to be considered
  approved. The `threshold` **MUST** be between 1 and the number of
  **distinct** `witnesses[].id` values, inclusive. A `witness` parameter where
  `threshold` is missing, non-integer, < 1, or > count(distinct ids) **MUST**
  cause the entry that sets it to fail verification; [[ref: Resolvers]]
  **MUST NOT** silently coerce a malformed `witness` to "no witnesses".
- `witnesses`: the array of [[ref: witnesses]] that **MUST** be non-empty,
  with each entry including the field:
  - `id`: (required) the DID of the witness. The DID **MUST** be a `did:key`
    DID and **MUST** be unique within the array (compared byte-for-byte after
    Unicode NFC normalisation). Each `id` contributes at most one approval to
    threshold counting, regardless of how many proofs are attributed to it.
  - The `did:key` used as a witness `id` **MUST** decode to a public key
    compatible with one of the cryptosuites listed in the
    [parameters](#didvh-did-method-parameters) defined by the version of the
    `did:vh` specification being used by the [[ref: DID Controller]] based on
    the active `method` [[ref: parameters]] property. A witness `id` whose
    body decodes to a key of any other type **MUST** cause the entry that
    sets it to fail verification at parameter-validation time — not at
    signature-verification time — so an invalid witness configuration cannot
    ever take effect.

##### Witness Threshold Algorithm

The use of the [[ref: threshold]] versus needing approvals from all
[[ref: witnesses]] is to prevent faulty [[ref: witnesses]] from blocking the
creation of a new version of the DID. To determine if the [[ref: threshold]]
has been met, participants **MUST**:

1. Verify each [[ref: Data Integrity]] proof in the witness proofs file for
   the relevant `versionId` independently.
2. Attribute each verified proof to a witness `id` from the **active**
   `witnesses` list. Proofs that cannot be attributed (the proof's key does
   not belong to any listed witness) **MUST** be discarded.
3. Form the set of **distinct** attributed `id` values. The threshold check
   applies to the size of this set, not to the raw proof count.
4. If `|set| ≥ threshold`, the update is "[[ref: witnessed]]"; otherwise
   the entry is not witnessed and fails verification.

The client of a [[ref: Resolver]] can evaluate the witnesses according to
the governance of the ecosystem. Such an evaluation is outside the scope of
this specification.

##### The Witness Proofs File

Proofs from [[ref: witnesses]] are placed into a separate file
(`did-witness.json`) from the [[ref: DID Log]]. It reaches the
[[ref: Resolver]] in the same ways as the [[ref: DID Log]]: passed directly in
`didWitness`, or by reference in `srcWitness`. The two files are passed
independently, so a client can take the witness proofs file from a different
[[ref: DID Log source]] than the [[ref: DID Log]] — for example, from a
[[ref: watcher]]'s `/witness` operation. When derived from a [`src` DID URL
parameter](#the-src-did-url-parameter), the witness proofs file is
`did-witness.json` in the same location as `did.jsonl`. The media type of the file
**SHOULD** be `application/json`.

The data model for the `did-witness.json` file is:

```json
[
  {
    "versionId": "1-Qmba111111...",
    "proof": [{ ... }, { ... }]
  },
  {
    "versionId": "2-Qzmb222222...",
    "proof": [{ ... }, { ... }]
  }
]
```

Where:

- `versionId` is the `versionId` of the [[ref: DID log entry]] to which the
  [[ref: witness]] proofs apply.
- `proof` is an array of [[ref: Data Integrity]] proofs that use the
  `versionId` as input data. The permitted [[ref: Data Integrity]]
  cryptosuite **MUST** be one listed in the
  [parameters](#didvh-did-method-parameters) defined by the version of the
  `did:vh` specification being used by the [[ref: DID Controller]] based on
  the active `method` [[ref: parameters]] property, and the `proofPurpose`
  set to `assertionMethod`.

Because a witness `id` is a `did:key` DID, the verification key is fully
determined by decoding the `did:key` body — no DID resolution is required and
no `verificationMethod` lookup is permitted. [[ref: Resolvers]] verifying a
witness proof **MUST**:

1. Parse the proof's `verificationMethod` as
   `did:key:<multibase>#<multibase>` and recover the public key by decoding
   the body multibase per the [[ref: did:key]] specification.
2. Verify the signature using **only** that key. [[ref: Resolvers]]
   **MUST NOT** dereference the witness DID for a key, **MUST NOT** consult
   any other key store, and **MUST NOT** accept a proof whose body multibase
   decodes to a key structurally invalid for the cryptosuites mandated by the
   active `method`.

A valid proof from a [[ref: witness]] carries the implication that **ALL**
prior [[ref: DID Log entries]] are also approved by that witness. To maintain
a manageable `did-witness.json` file size, the [[ref: DID Controller]]
**SHOULD** remove all older proofs for **distributed** [[ref: DID Log entries]], keeping only the latest proof for each witness.

To eliminate the race condition in distributing the [[ref: DID Log]] and
`did-witness.json` files, when a new [[ref: DID Log entry]] is being added,
[[ref: witness]] proofs **MUST** be added to the `did-witness.json` file and
that file made available **BEFORE** the [[ref: DID Log]] containing the new
[[ref: DID Log entry]] is distributed. As a result, [[ref: Resolvers]] may
find proofs for undistributed [[ref: DID log entries]] in the
`did-witness.json` file. [[ref: Resolvers]] **MUST** ignore proofs with
`versionId`s not in the [[ref: DID Log]]. Since [[ref: Resolvers]] cannot
verify an undistributed [[ref: DID log entry]], the [[ref: witness]] proofs on
undistributed [[ref: DID log entries]] do not carry the implication of
approval of prior [[ref: DID Log entries]]. Therefore, at times there may be
two proofs in the `did-witness.json` file for a [[ref: witness]]:

- The most recent proof for a distributed [[ref: DID Log entry]].
- An additional proof that applies to an undistributed
  [[ref: DID Log entry]].

To avoid unnecessary clutter in the `did-witness.json` file, array entries
without proofs (e.g., containing only the `versionId`) **SHOULD** be removed.

A witness proofs file **MUST NOT** contain proofs for two different
`versionId`s with the same version number. When adding proofs to the witness
proofs file, a [[ref: DID Controller]] **MUST** reject any proof for a
`versionId` that conflicts in this way with the [[ref: DID Log]] it is
distributing.

##### Witnessing a DID Version Update

The following process is used to witness a DID version update:

- The [[ref: DID Controller]] prepares the full [[ref: DID Log Entry]]
  (including the `proof` element) for the new version of the DID, and shares
  it with the active [[ref: witnesses]].
  - The specification leaves to implementers *how* the [[ref: log entry]]
    data is provided to the [[ref: witnesses]].
- Each [[ref: witness]] **MUST** hold its own copy of the [[ref: DID Log]]
  prior to witnessing, and **MUST** confirm that the controller-supplied
  candidate entry verifies as the next entry to that [[ref: DID Log]].
  - A [[ref: witness]] that does not yet hold the [[ref: DID Log]] — for
    example, one added to the `witness` [[ref: parameter]] after the DID was
    created — obtains it from a [[ref: DID Log source]], typically the
    [[ref: DID Controller]] along with the candidate entry, and **MUST**
    verify it in full before witnessing. That copy then becomes its own copy,
    which later candidate entries have to extend. A [[ref: witness]] can also
    compare the copy with those held by [[ref: watchers]], to check that the
    log it was given has not been forked before it joined.
- Each [[ref: witness]] **MUST** independently verify the candidate entry
  using every step in the [Resolution Algorithm](#resolution-algorithm). Any
  failure **MUST** cause the witness to refuse approval.
- A [[ref: witness]] **MUST NOT** approve more than one [[ref: log entry]]
  with the same predecessor.
- Each [[ref: witness]] determines (based on the governance of the ecosystem)
  if they approve of the DID version update.
  - The meaning of "approve" for any given implementation is outside the
    scope of this specification.
- If the verification is successful and approval granted, the
  [[ref: witness]] creates and sends to the [[ref: DID Controller]] a
  [[ref: Data Integrity]] proof signed using the [[ref: witness]]'s `did:key`
  DID.
  - The specification leaves to implementers how [[ref: witness]] proofs are
    sent to the [[ref: DID Controller]].
- The [[ref: DID Controller]] **MUST** add the proof to the record for the
  applicable `versionId` for the undistributed [[ref: DID log entry]] in the
  `did-witness.json` file.
  - The [[ref: DID Controller]] **MAY** make the updated `did-witness.json`
    file available as new witness proofs are added to the file.
  - The [[ref: DID Controller]] **MUST** make the updated `did-witness.json`
    file available **after** a [[ref: threshold]] of witness proofs have been
    received and **before** the witnessed [[ref: DID Log]] is distributed.

::: note
A [[ref: Resolver]] can verify that a [[ref: threshold]] of
[[ref: witnesses]] signed an entry, but not that each [[ref: witness]] held
its own copy of the log or checked the entry against it. The requirements on
[[ref: witnesses]] above define correct behaviour; whether a given
[[ref: witness]] meets them is a matter of trust in that [[ref: witness]].
:::

##### Verifying Witness Proofs During Resolution

A `did:vh` [[ref: Resolver]] **MUST** verify that all [[ref: DID Log entries]] that have active [[ref: witnesses]] have a [[ref: threshold]] of
approving witnesses. [[ref: Resolvers]] **MUST**:

1. Complete all non-witness verifications of the [[ref: DID Log]] **before**
   processing any witness proof. Witness verification **MUST NOT** substitute
   for entry-hash or signature verification.
2. Get the witness proofs file, passed directly in `didWitness` or retrieved
   from the `srcWitness` reference. If no witness proofs file was passed, or
   the `srcWitness` reference was not found, treat the file as containing no
   proofs.
3. For each entry in the witness proofs file, confirm its `versionId` matches
   a `versionId` present in **this** [[ref: DID Log]]. Non-matching entries,
   including those for future entries, **MUST** be discarded and **MUST NOT**
   be treated as evidence of witnessing for any other entry.
4. Verify enough proofs to meet the [[ref: threshold]] for all entries
   requiring witnessing.
5. For each entry requiring witnessing, confirm a threshold of verified,
   distinct-witness proofs whose `versionId` matches the current or any later
   entry in the [[ref: DID Log]]. Otherwise, that entry fails verification.

For each witness proof, the [[ref: Resolver]] **MUST** extract
`verificationMethod` and confirm that:

1. It is a valid, [[ref: did:key]] specification-compliant DID URL of the form
   `did:key:<multibase>#<multibase>` where body and fragment multibases are
   byte-equal (see [`did:key` body/fragment
   check](#witness-dids-and-reputation)).
2. No previously verified proof was found in the witnesses array for the same
   `versionId` and `id` (one count per witness per entry).

A proof failing either requirement **MUST** be discarded from threshold
counting.

A [[ref: DID Controller]] is expected to prune the `did-witness.json` file to
include only the last proof for each witness for a distributed
[[ref: DID log entry]]. However, if a [[ref: DID Controller]] does not prune
the file, a [[ref: Resolver]] **MAY** do the pruning as part of the
resolution process, verifying only the minimum number of proofs needed to meet
the [[ref: threshold]] for each [[ref: DID log entry]]. While it is expected
that a [[ref: DID Controller]] will exclude any proofs that fail verification,
a [[ref: Resolver]] **MAY** ignore any proofs that fail verification and still
resolve the DID if there are enough valid proofs to meet the
[[ref: threshold]] requirements.

If you want to learn more about the practical application of witnesses, see
the Implementer's Guide section on
[Witnesses](https://didwebvh.info/latest/implementers-guide/witnesses/) on the
`did:webvh` information site, which applies equally to `did:vh`.

#### DID Watchers

[[ref: Watchers]] are components found in some digital trust and DID
ecosystems that monitor DIDs on behalf of clients for various purposes, such
as:

- **Caching verified DIDs:** Storing verified [[ref: DID Logs]] to facilitate
  efficient resolution.
- **Ensuring persistence:** Maintaining access to DIDs even after removal by
  the [[ref: DID Controller]] from other sources.
- **Detecting inconsistencies:** Identifying malicious behavior by the
  [[ref: DID Controller]], such as distributing altered [[ref: DID Logs]], or
  giving different parties conflicting logs (duplicity). A [[ref: watcher]]
  does this by checking that each copy of a [[ref: DID Log]] it receives or
  retrieves extends the copy it holds. This is the most reliable way to detect
  duplicity, because it does not depend on trusting the
  [[ref: DID Controller]] or the [[ref: witnesses]], but it detects only the
  copies the [[ref: watcher]] sees. A network of [[ref: watchers]] can reach
  consensus independently of [[ref: witnesses]].

For `did:vh`, [[ref: watchers]] are also a primary [[ref: DID Log source]],
as described in [From Watchers](#from-watchers). Any party can set up a
[[ref: watcher]] for `did:vh` DIDs, without the involvement of the
[[ref: DID Controller]] and without being listed in the [[ref: DID Log]] —
for example, a relying party that wants a check independent of the
[[ref: DID Controller]]. A `did:vh` [[ref: DID Controller]] can also opt to
collaborate with specific [[ref: watchers]] by publishing their URIs in the
[[ref: parameters]] of [[ref: DID log entries]]. It is outside the scope of
this specification how a [[ref: DID Controller]] requests a [[ref: watcher]]
monitor a DID or how a [[ref: watcher]] requests it be included in the
[[ref: DID Log]] of a DID.

A [[ref: watcher]] **MUST** fully verify each [[ref: DID Log]] it receives or
retrieves, **SHOULD** check that it extends the copy it holds (see [Log
Freshness and Duplicity](#log-freshness-and-duplicity)), and **SHOULD** report
any divergence it detects between copies. A [[ref: watcher]] indexes the
[[ref: DID Logs]] it holds by [[ref: SCID]].

The governance of [[ref: watchers]] is out of scope for this specification,
which defines only the technical mechanisms for notifying and querying
[[ref: watcher]] services.

##### Publishing Watcher URLs

`did:vh` provides a mechanism for notifying [[ref: Resolvers]] (and their
clients via [[spec:DID-RESOLUTION]] metadata) about configured
[[ref: watchers]]. The `watchers` [[ref: parameter]] lists URIs that identify
the DID's [[ref: watchers]].

When resolving a `did:vh` DID, [[ref: Resolvers]] **MUST** provide the active
list of [[ref: watchers]] in the DID Document Metadata, as noted in [DID
Resolution Metadata](#did-resolution-metadata).

Watcher URIs **MAY** use schemes other than HTTP(S), such as a DID
([[spec:DID-CORE]]), depending on the specific implementation or network
requirements. However, this specification does not define how non-HTTP(S)
[[ref: watcher]] URIs are resolved or interacted with. If a
[[ref: watcher]] uses an HTTP(S) URL, it **MUST** support the HTTP-based
interaction model defined in the [Watcher Endpoints and
Behavior](#watcher-endpoints-and-behavior) section.

If a `watchers` entry is included in a [[ref: DID log entry]], it replaces
the active set of [[ref: watchers]].

If a new [[ref: watcher]] is added after a DID has existed for some time, the
[[ref: DID Controller]] **SHOULD** notify the new [[ref: watcher]] about
previously created [[ref: DID Resources]].

[[ref: Watchers]] do not need to be listed in the [[ref: DID log]].
[[ref: DID Controllers]] **MAY** send notifications to [[ref: watchers]] that
the [[ref: DID Controller]] is aware of but does not list in the
[[ref: DID Log]].

##### Watcher Endpoints and Behavior

A [[ref: watcher]] is a web server accessible via HTTP that **MUST** support
the following capabilities:

- **Client Requests:**
  - Retrieve the [[ref: DID log]] for a given [[ref: SCID]].
  - Retrieve the witness proofs file for a given [[ref: SCID]].
  - Retrieve a resource for a given [[ref: SCID]] and resource path.
- **Notifications (typically) from the [[ref: DID Controller]]:**
  - Notify the [[ref: watcher]] about a new log entry for a DID.
  - Notify the [[ref: watcher]] about a new or updated [[ref: DID resource]].
  - Request removal of a given [[ref: SCID]] from the [[ref: watcher]]'s
    cache.
  - Request removal of a given DID resource from the [[ref: watcher]]'s
    cache.

##### Watcher HTTP API Operations

The following HTTP API operations define the interaction between
[[ref: watchers]] and other components. They are defined in the `did:vh`
[Watcher OpenAPI YML
Definition](https://github.com/decentralized-identity/didvh/blob/main/watcherOpenAPI/watcher-v1.0.0.yml),
which includes the request and response data models and status codes for each
of the endpoints. It is derived from the `did:webvh` [Watcher OpenAPI YML
Definition](https://github.com/decentralized-identity/didwebvh/blob/main/watcherOpenAPI/watcher-v1.0.0.yml),
with the changes noted below for notifying a [[ref: watcher]] of a DID that
has no web location.

- **GET `<WATCHER URL>/log?scid=<SCID>`**: Returns the latest
  [[ref: DID Log]] for the given [[ref: SCID]].
- **POST `<WATCHER URL>/log?did=<DID URL>`**, where the DID URL includes a
  `src` query parameter: notifies the [[ref: watcher]] of a log update,
  prompting retrieval of the latest [[ref: DID Log]] (and witness proofs file)
  from the location `src` names, as defined in [The `src` DID URL
  Parameter](#the-src-did-url-parameter). As for any DID URL, the `src` value
  cannot be a local file or loopback reference.
- **POST `<WATCHER URL>/log?did=<DID>`**, where there is no `src`
  parameter: notifies the [[ref: watcher]] of a log update, and **MUST** carry
  the complete [[ref: DID Log]] as the request body, with media type
  `text/jsonl`. If [[ref: witnesses]] are active, the
  [[ref: DID Controller]] **MUST** first send the witness proofs file with
  **POST `<WATCHER URL>/witness?did=<DID>`**, with media type
  `application/json`.
- **POST `<WATCHER URL>/log/delete?scid=<SCID>`**: Notifies the
  [[ref: watcher]] that the requester asks for the given `<SCID>` to be
  deleted from the [[ref: watcher]]'s cache. If it is removed, the
  [[ref: watcher]] returns a `404 Not Found` status to subsequent requests for
  that `<SCID>`. The body of the request
  is a Data Integrity proof from the requester that may be used by the
  [[ref: Watcher]] to decide on the legitimacy of the request. The
  [[ref: Watcher]] will act (or not) on the request according to its
  governance, which is out of scope of this specification. For example, a
  [[ref: watcher]] might implement a workflow that has to be completed to
  approve the deletion of a `<SCID>` from the [[ref: watcher]]'s cache. The
  endpoint could be used to carry out a "right to be forgotten" order, such as
  might be required under Europe's [General Data Protection Regulation
  (GDPR)](https://gdpr-info.eu/).
- **GET `<WATCHER URL>/witness?scid=<SCID>`**: Returns the latest
  `did-witness.json` file for the given [[ref: SCID]].
- **GET `<WATCHER URL>/resource?scid=<SCID>&path=<resourcePath>`**: Retrieves
  the requested resource.
- **POST `<WATCHER URL>/resource?scid=<SCID>&path=<resourcePath>`**: Notifies
  the [[ref: watcher]] of a new or updated resource.
- **POST `<WATCHER URL>/resource/delete?scid=<SCID>&path=<resourcePath>`**:
  Notifies the [[ref: watcher]] that the given `<resourcePath>` associated
  with the `<SCID>` is to be deleted from the [[ref: watcher]]'s cache. If it
  is removed, the [[ref: watcher]] returns a `404 Not Found` status to
  subsequent requests for that `<SCID>` and `<resourcePath>`. The body of the request is a
  Data Integrity proof from the requester that may be used by the
  [[ref: Watcher]] to decide on the legitimacy of the request, as for
  `/log/delete`.

In both forms of the `/log` notification, the [[ref: watcher]] **MUST** fully
verify the [[ref: DID Log]], **MUST** confirm that it extends any copy the
[[ref: watcher]] already holds, and indexes it by [[ref: SCID]].

### Publishing and Retrieving DID Log Resources

This section applies whenever a [[ref: DID Log]], witness proofs file, or
other [[ref: DID Resource]] is published to, or retrieved from, a network
location over HTTP — a web location named by a `srcLog` or `srcWitness`
reference or by a `src` DID URL parameter, or a
[[ref: watcher]]. It applies equally to a [[ref: Resolver]] and to a client
that retrieves the files itself before passing them directly.

#### Publishing DID Log Resources

1. [[ref: DID Log]] resources **MUST** be served over HTTPS, with the server
   authenticated by TLS server authentication. Plain HTTP **MUST NOT** be
   used, except for testing or non-production deployments.
2. Self-signed TLS certificates **SHOULD NOT** be used in production.
3. So that a [[ref: Resolver]] running in a web browser can retrieve it, the
   HTTP response for the [[ref: DID Log]] **MUST** include the header
   `Access-Control-Allow-Origin: *`.
4. A [[ref: DID Controller]] that serves [[ref: DID Log]] resources through
   intermediaries such as CDN caches or load balancers **MUST** ensure that
   those intermediaries do not serve stale or altered resources.

#### Retrieving DID Log Resources

A [[ref: Resolver]] fetches resources from locations chosen by its caller, or
by whoever gave the caller a DID URL, which makes it a potential Server-Side
Request Forgery (SSRF) vector. When retrieving [[ref: DID Log]] resources, a
[[ref: Resolver]] **MUST**, except as defined for loopback references in [Web
Locations](#web-locations):

1. **HTTPS only.** Reject any URL scheme other than `https`, including after a
   redirect, and validate the server's TLS certificate. Certificate validation
   **MUST NOT** be disabled by default.
2. **No automatic redirects.** Not automatically follow HTTP 3xx responses
   when fetching the [[ref: DID Log]] or the witness proofs file. If following redirects is
   offered as an opt-in, every check in this list **MUST** be re-applied to
   each redirect target.
3. **Case-insensitive percent-decoding.** Normalise the case of
   percent-encoded octets, per [[spec:rfc3986]] §2.1, **before** making any
   allow or deny decision.
4. **Re-validate after decoding.** Apply all host and path checks to the
   decoded values. Percent-encoded IP literals and traversal sequences (such
   as `%2E%2E` and `%2e%2e`) **MUST** be rejected after decoding.
5. **IP-literal and private-address rejection.** Reject IPv4 and IPv6 literal
   hosts both (a) after percent-decoding the host, and (b) after DNS
   resolution. By default, deny loopback (`127.0.0.0/8`, `::1`), private
   (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `fc00::/7`) and
   link-local (`169.254.0.0/16`, `fe80::/10`) addresses. Opt-ins for
   development **MAY** be provided, but **MUST** be off by default.
6. **No localhost in production.** Not issue requests to `localhost`,
   `127.0.0.0/8`, `::1`, or names resolving to them, except through a testing
   opt-in that is off by default, or for a loopback reference.
7. **Response size cap.** Enforce a maximum body size for both the
   [[ref: DID Log]] and the witness proofs file, terminating the transfer if
   it is exceeded regardless of any declared `Content-Length`. The
   `Content-Length` header **SHOULD** be checked before the body is read, and
   its absence treated as grounds for a stricter cap or for rejection. A cap
   of 5 MiB is **RECOMMENDED** as a default.
8. **Operation timeout.** Enforce a wall-clock timeout on the complete
   fetch-and-verify operation for a single resolution. A timeout of 30
   seconds is **RECOMMENDED** as a default.

When performing the DNS resolution for a retrieval, the [[ref: Resolver]]
**SHOULD** use [[spec:rfc8484]] in order to prevent tracking of the identity
being resolved.

A [[ref: Resolver]] **MAY** also limit retrievals to locations permitted by its
configuration. A [[ref: Resolver]] that supports `srcLog` and `srcWitness`
references using [other schemes](#other-schemes) applies equivalent protections to those retrievals.
