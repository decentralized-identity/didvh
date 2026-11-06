## Security Considerations

*This section is non-normative.*

This section follows the guidelines in [[spec:RFC3552]] and aligns with the
[[spec:DID-CORE]] requirements in [DID Core
7.3](https://www.w3.org/TR/did-core/#security-requirements). It describes the
threats to `did:vh` and how the requirements in this specification address
them. The requirements themselves are defined in the normative sections
referenced below.

### No Authoritative Location

A `did:webvh` [[ref: Resolver]] treats the DID's web location as the
authoritative place to find the current [[ref: DID Log]]. A `did:vh` DID has
no such location. The integrity of any copy of the log is fully protected by
the [[ref: SCID]], the hash chain and the proofs, so a [[ref: Resolver]] can
safely accept a log from any source. What a [[ref: Resolver]] cannot know from
the log alone is whether it is *complete* — whether newer entries exist.

The resulting threats, and their mitigations, are:

- **Withholding and truncation.** A source, or the party presenting a DID,
  supplies an older copy of the log, for example one from before a key
  rotation or deactivation. A [[ref: Resolver]] verifies the log it is given
  but cannot detect this on its own. Mitigations: clients that need the
  current version source the log from a source expected to be current, such
  as a listed [[ref: watcher]], and never treat the absence of newer entries
  at one source as proof that none exist; a [[ref: Resolver]] that retains the
  latest `versionId` it has seen detects a copy that is shorter than one seen
  before; and a client can ask the [[ref: Resolver]] to compare the copy with
  those held by [[ref: watchers]] (see [Comparing Copies of a DID
  Log](#comparing-copies-of-a-did-log) and [Log Freshness and
  Duplicity](#log-freshness-and-duplicity)).
- **Duplicity (split view).** A [[ref: DID Controller]], or an attacker with
  its update keys, creates two different extensions of one log and gives them
  to different parties. This specification requires
  [[ref: DID Controllers]] to distribute only logs that extend the log they
  previously distributed ([Update (Rotate)](#update-rotate)),
  [[ref: witnesses]] to approve at most one child of any entry ([Witnessing a
  DID Version Update](#witnessing-a-did-version-update)), and the witness
  proofs file to contain no conflicting proofs ([The Witness Proofs
  File](#the-witness-proofs-file)). Because no other party can verify that a
  [[ref: witness]] has followed these rules, detecting split views ultimately
  relies on comparing copies of the log from multiple sources: clients that
  obtain copies from several sources compare them and do not rely on versions
  past the point of divergence; peers apply updates so that non-extending
  entries are rejected; [[ref: Resolvers]] given more than one copy fail with
  `logForked` when the copies diverge; and [[ref: watchers]] — including ones
  run by relying parties — detect and can report conflicting logs.
  [[ref: Watchers]] listed in the log are chosen by the
  [[ref: DID Controller]], so the `extraWatchers` [resolution
  option](#didvh-resolution-options), naming [[ref: watchers]] the client
  chooses, gives the stronger check.
- **Compromised keys and no location to reclaim.** In `did:webvh`, control of
  the web location is a second factor an attacker who steals the update keys
  also needs in order to publish. A `did:vh` log has no such location, so an
  attacker with the update keys can distribute a valid update through any
  source. Mitigation: [[ref: DID Controllers]] are expected to use
  [[ref: pre-rotation]] (see [Authorized Keys](#authorized-keys)), and to use
  [[ref: witnesses]] for DIDs relied on by many parties, so that an update
  also needs the approval of a threshold of [[ref: witnesses]] (see [DID
  Witnesses](#did-witnesses)).

### Forked Logs

Two copies of a [[ref: DID Log]] that both pass verification but diverge are
proof that a party able to sign as the [[ref: DID Controller]] — and as a
threshold of the [[ref: witnesses]], if any — created both. That party is the
[[ref: DID Controller]] acting duplicitously, an attacker holding its keys, or
an attacker able to derive private keys from public keys, such as one with a
cryptographically relevant quantum computer. Such an attacker can sign as the
[[ref: witnesses]] too, so the rule that a [[ref: witness]] approves at most
one child of an entry does not help, and [[ref: pre-rotation]] protects only
entries not yet published: each published entry reveals the public keys an
attacker would need to sign an alternative to it. Pre-rotation does still stop
the attacker from extending the genuine branch.

Verification cannot show which branch is genuine, and the copy passed to the
[[ref: Resolver]] has no special standing — a forked branch can be passed
directly, named in a `src` DID URL, or held by a [[ref: watcher]]. A
[[ref: Resolver]] that chose a branch would let whoever supplied the reference
copy take control of the DID. A [[ref: Resolver]] that detects a fork
therefore fails with `logForked` (see [Comparing Copies of a DID
Log](#comparing-copies-of-a-did-log)). That failure can itself be the
attacker's goal: a forked copy at a [[ref: watcher]] can make resolution fail.
But it affects only clients that compare against that [[ref: watcher]],
whether it was named in `extraWatchers` or listed in the log. That is a denial
of service rather than a takeover.

A [[ref: watcher]] keeps the first copy of a DID it receives and accepts only
copies that extend it, so a [[ref: watcher]] that first receives a forged
branch keeps it. [[ref: DID Controllers]] are advised to send each new DID to
their [[ref: watchers]] when it is created, so that their first copy is
genuine. Copies that [[ref: watchers]] archived before a fork appeared are the
best available evidence of which branch came first.

A forked DID cannot be repaired; its [[ref: DID Controller]] has to create a
new DID. Against an attacker that can derive private keys, the defence is to
move to a cryptosuite the attacker cannot break, by upgrading the `method`
version (see [Post Quantum Attacks](#post-quantum-attacks)), before such an
attack is possible.

### Threats and Attacks

The following classes of attack apply to all DID operations:

- **Eavesdropping** — [[ref: DID Logs]], witness proofs files and other
  [[ref: DID Resources]] retrieved over a network are served over HTTPS (see
  [Publishing and Retrieving DID Log
  Resources](#publishing-and-retrieving-did-log-resources)). The verifiability
  of `did:vh` means that tampering with the contents of individual log entries
  is detectable; TLS protects against passive observation and other
  network-based risks.

- **Replay attacks** — A [[ref: Resolver]] verifies the whole log, including
  monotonic progression of versions and times and the chain of entry hashes,
  and rejects any log that does not verify, as defined in the [Resolution
  Algorithm](#resolution-algorithm).

- **Message insertion, deletion, and modification** — Each
  [[ref: DID Log entry]] and witness proof is integrity-protected by a
  cryptographic signature. Any change to a log entry causes verification to
  fail, as defined in the [Resolution Algorithm](#resolution-algorithm).

- **Truncation, withholding and split views** — See [No Authoritative
  Location](#no-authoritative-location) above.

- **Denial of Service (DoS) and amplification** — A malicious or compromised
  source could attempt to exhaust resolver resources by supplying
  pathologically large logs, or by holding a connection open and streaming log
  entries indefinitely. This risk is most acute for resolvers that encounter
  previously-unseen DIDs on behalf of their clients — for example, a service
  that performs DID resolution in onboarding new users. In such cases an
  attacker need only find a service that will pass their DID and log through
  to a resolver on their behalf. A [[ref: Resolver]] that returns the
  [[ref: DID Log]] in the resolution metadata, when asked with the
  `includeDidLog` option, returns a response up to the size of the log it
  accepted, so the size cap also bounds the response.

  - The requirement that each entry's `versionTime` be strictly greater than
    the previous one, in whole seconds, and not in the future, constrains log
    length — a log with implausibly rapid `versionTime` progression is likely
    malformed. However, this does not fully prevent the attack, as a patient
    attacker can pace log entry generation to match valid `versionTime`
    increments.

  - The size cap and timeout required in [Retrieving DID Log
    Resources](#retrieving-did-log-resources) bound the cost of a single
    resolution. A declared `Content-Length` lets a resolver reject an
    oversized log before processing begins, but cannot itself be relied on;
    the presence of HTTP range request support (`Accept-Ranges: bytes`) is a
    useful signal that a server is serving a static resource rather than
    generated content. A [[ref: Resolver]] that accepts logs passed directly
    is advised to apply an equivalent size limit to `didLog` and
    `didWitness` values. Resolvers are also advised to apply rate limits to
    resolution requests, particularly for previously-unseen DIDs, and
    concurrency limits on simultaneous resolution operations.

  - What counts as "excessively large" or "repeated" is best established
    through community or ecosystem governance. These mitigations are standard
    HTTP service hardening practices and are not unique to `did:vh`.

- **Man-in-the-middle (MitM)** — HTTPS and signature verification of log
  entries and witness proofs protect against undetected modification of
  individual entries. TLS also authenticates the server and reduces
  opportunities for active interference.

- **Other attacks** — The required `did:vh` verification process mitigates
  downgrade attacks on cryptographic algorithms and prevents poisoning of log
  or witness files, since unauthorized changes fail signature verification.
  Verification does not, however, address availability. Keeping the
  [[ref: DID Log]] in several places, including with [[ref: watchers]],
  improves resilience.

### References and Other Client-Supplied Values

The `srcLog` and `srcWitness` resolution options, and a `src` DID URL query
parameter, are supplied by the client or by whoever gave it the DID URL, and
so are untrusted input.
The same is true of a `didLog` value, which is why it is verified in full, and
of `extraWatchers` URLs:

- **Server-side request forgery.** A web location reference or
  `extraWatchers` URL can name any host. [[ref: Resolvers]] apply to these
  retrievals the transport requirements in [Retrieving DID Log
  Resources](#retrieving-did-log-resources), including rejecting IP literals
  and private addresses and not following redirects automatically. These are
  standard SSRF defences; they are stated normatively because common
  implementations have been found vulnerable to at least one of: following
  redirects, accepting IP literals, case-sensitive percent-decoding, and
  checking values before rather than after decoding. A [[ref: Resolver]] can
  also decline to retrieve from `srcLog` and `srcWitness` references at all, refuse or limit
  `extraWatchers` (see [`did:vh` Resolution
  Options](#didvh-resolution-options)), or allow only locations permitted by
  its configuration. The same hardening applies to a client retrieving from a
  web location or a [[ref: watcher]] itself. The one exception is a loopback
  (`http`) reference, which is accepted only by a [[ref: Resolver]] local to
  its client and never from a DID URL; see [Web Locations](#web-locations). A
  [[ref: Resolver]] supporting [other schemes](#other-schemes) applies
  equivalent protections to those retrievals and to any gateway it uses.
- **Returning retrieved content.** With the `includeDidLog` option, a
  [[ref: Resolver]] returns the files it retrieved to its caller. It returns
  only a [[ref: DID Log]] that has verified as the history of the DID being
  resolved, and the witness proofs for it, so the option cannot be used to
  read arbitrary resources from locations the [[ref: Resolver]] can reach.
- **Stale or wrong logs.** A web location may serve an old copy of the log,
  or a log for a different DID. The first is covered under [No Authoritative
  Location](#no-authoritative-location); the second is detected by the
  [[ref: SCID]] check that the [Resolution Algorithm](#resolution-algorithm)
  applies to every entry.
- **Local file and service access.** A local file or loopback reference is
  accepted only by a [[ref: Resolver]] local to its client; a
  [[ref: Resolver]] that accepts requests from other systems never reads local
  files or retrieves from loopback hosts, and a `src` DID URL parameter is
  never a `file:` or loopback URL. A local [[ref: Resolver]] reads only the
  files its client names, and the content of each is verified in full. See
  [Local Files](#local-files) and [The `src` DID URL
  Parameter](#the-src-did-url-parameter).
- **Misleading association.** The domain in a reference or `src` value says
  nothing about who controls the DID. A [[ref: DID Controller]] can publish a
  log at any location it can write to, and anyone can put any `src` value in
  a DID URL.
  For this reason, [[ref: Resolvers]] and their clients do not treat a web
  location as evidence of an association between the DID and the owner of
  the domain (see [The `srcLog` and `srcWitness`
  Options](#the-srclog-and-srcwitness-options)).

### Peer-to-Peer Use

When [[ref: DID Logs]] are exchanged peer-to-peer, each party holds the only
copies of the other's log, and an update can carry only the new entry. With no
other copy to compare against, the hash chain is what protects a party: it
applies an update as defined in [Applying Peer-to-Peer
Updates](#applying-peer-to-peer-updates), so that entries that do not continue
from the log it holds are rejected, and keeps a rejected update as evidence of
duplicity.

### Residual Risks

Residual risks include:

- Compromise of a system holding a copy of the [[ref: DID Log]], such as a web
  host or a [[ref: watcher]].
  - While this can impact access to the [[ref: DID Log]] and associated files,
    it does not compromise the integrity of the log entries themselves, nor
    the verifiability of the DID.
- Compromise of [[ref: DID Controller]] private keys.
  - A [[ref: DID Controller]] can mitigate this risk through the use of
    [pre-rotation keys](#pre-rotation-key-hash-generation-and-verification),
    treating each revealed pre-rotation key as spent and never reusing it.
    Reuse of a revealed key is not invalid, but it reduces the containment
    that pre-rotation provides.
  - Other good security practices also help, such as using hardware security
    modules (HSMs) or secure enclaves for key storage, enforcing strong access
    controls, maintaining secure backups of critical keys, and performing
    regular key rotations.
- Weaknesses in underlying cryptographic algorithms after deployment.
- Misconfiguration of cache control or TTL values.
- Implementation errors in [[ref: Resolvers]] or [[ref: DID Controllers]].
- Resolvers operating in contexts where they may encounter large numbers of
  previously-unseen DIDs — such as open verification services — face elevated
  resource exhaustion risk. Such resolvers are advised to implement rate
  limiting and monitoring for abusive resolution patterns, with the ability to
  block offending sources.

### Integrity Protection and Update Authentication

All DID operations (create, update, deactivate) are integrity-protected by the
cryptographic verification of [[ref: DID Log entries]]. Update authentication
is provided by verifying the [[ref: DID Controller]]'s proof(s) against the
active `updateKeys` in the DID [[ref: parameters]] (see [Authorized
Keys](#authorized-keys)).

Because a `did:vh` [[ref: DID Log]] and its entries are self-certifying, they
can be verified and trusted regardless of how they are obtained — passed with
the DID, from a web location, via a cache, through a CDN, from a
[[ref: watcher]], from a peer, or via a trusted resolver service. The
verification process ensures authenticity and integrity independent of the
transport channel.

### Authentication Characteristics

The authentication of DID updates is based on possession of the private keys
associated with the update and pre-rotation keys. The security of the DID
therefore depends on the strength of these keys, their secure storage, and the
cryptographic algorithms used.

### Unique Assignment of DIDs

In `did:vh`, uniqueness of a DID is based on the
[[ref: self-certifying identifier]] (SCID) generated at the inception of the
DID. The SCID is cryptographically bound to the initial content of the DID,
including the [[ref: DID Controller]]'s keys, and ensures that no two
independently created DIDs can have the same identifier. A `did:vh` DID
contains nothing but its SCID, so there is no other component, such as a
domain name, whose assignment could be confused or reassigned.

### Endpoint Authentication

Retrievals from web locations and from [[ref: watchers]] use HTTPS with TLS
server authentication (see [Publishing and Retrieving DID Log
Resources](#publishing-and-retrieving-did-log-resources)). The only exception
is a retrieval from a loopback (`http`) reference by a [[ref: Resolver]] local
to its client. As with all sources, a [[ref: DID Log]] is verified by its
content, not by the authentication of the server that provided it; TLS
provides additional protection against active network attacks.

### Network Topology

`did:vh` does not require any particular network. A [[ref: DID Log]] may be
held on web infrastructure, by [[ref: watchers]], in other stores, or
exchanged peer-to-peer. CDN caches and load balancers in front of a web
location can serve stale data; [Publishing DID Log
Resources](#publishing-did-log-resources) requires [[ref: DID Controllers]]
that use them to prevent this.

### Cryptographic Protection

The following data is protected:

- **DID Log entries** — Signed by the [[ref: DID Controller]]'s keys.
- **Witness proofs** — Signed by witness keys.

These signatures provide integrity and update authentication but not
confidentiality; [[ref: DID Log]] entries are public. Private keys and other
secret material do not belong in the [[ref: DID Log]].

### Signature Implementation

`did:vh` uses standard [[ref: Data Integrity]] proof mechanisms for signing
[[ref: DID Log entries]] and witness proofs, as defined in the cryptographic
suite used. The cryptosuites permitted for each `method` value are listed in
[`did:vh` DID Method Parameters](#didvh-did-method-parameters).

### Cross-Origin Resource Sharing (CORS)

To support resolution by client applications running in a web browser, a
[[ref: DID Log]] published at a web location needs to be accessible from any
origin. [Publishing DID Log Resources](#publishing-did-log-resources) requires
the `Access-Control-Allow-Origin: *` header for this reason.

### Post Quantum Attacks

The `did:vh` [[ref: Key Pre-Rotation]] approach provides enough flexibility
for "post-quantum safety." For guidance on mitigating post-quantum attacks,
implementers are advised to consult the [corresponding `did:webvh`
Implementation Guide
section](https://didwebvh.info/latest/implementers-guide/prerotation-keys/#post-quantum-attacks),
which applies equally to `did:vh`. [[ref: Pre-rotation]] is especially
valuable for `did:vh`, for the reason given under [No Authoritative
Location](#no-authoritative-location). Pre-rotation protects entries not yet
published, but not entries already published, whose public keys are known;
see [Forked Logs](#forked-logs).

### Implementation Hygiene

The following practices are not unique to `did:vh` but represent recurring
failure points observed across implementations of `did:webvh`, from which
`did:vh` is derived. Implementers are advised to treat these as baseline
hygiene.

- **Dependency currency.** Keep cryptographic and HTTP dependencies on
  supported, patched versions. Run a vulnerability scanner (`cargo audit`,
  `pip-audit`, `npm audit`, OWASP Dependency-Check) on every build.
- **Filesystem permissions.** Create private keys and secret-bearing
  configuration files with owner-only permissions (e.g., `0600` on POSIX). Do
  not read secret material from the current working directory or other
  untrusted locations by default.
- **CLI tools.** Do not print private keys, mnemonics, or other long-lived
  secrets to stdout/stderr or shell history. Where display is necessary,
  prompt before printing and offer a file-output alternative with restrictive
  permissions.
- **HTTPS certificate validation.** Never disable certificate validation by
  default (this is required for resolvers in [Retrieving DID Log
  Resources](#retrieving-did-log-resources)). Certificate pinning is not
  required (and is incompatible with platform hosting), but ecosystems with
  appropriate trust models can opt in.
- **Error messages.** Avoid including internal stack traces, file paths, or
  library versions in `problemDetails` returned to clients.

## Privacy Considerations

*This section is non-normative.*

This section addresses the privacy considerations in alignment with
[[spec:RFC6973]] Section 5, and aligns with the [[spec:DID-CORE]]
requirements in [DID Core
7.4](https://www.w3.org/TR/did-core/#privacy-requirements).

### Surveillance

Retrieving a [[ref: DID Log]] reveals the [[ref: Resolver]]'s interest in the
DID to the source it retrieves from: the host of a web location, or a
[[ref: watcher]]. Because `did:vh` lets a client choose its sources, it can
reduce this by using a cached copy, a [[ref: watcher]] it trusts, or a copy
supplied by the party presenting the DID, and by not retrieving from web
locations chosen by others. A [[ref: Resolver]] or client can also retrieve
through privacy-enhancing network tools, such as Tor or a VPN, or a trusted
resolver service, so that the source does not learn the network address of
the party interested in the DID. While the contents of a [[ref: DID Log]] are
generally intended to be public, the timing, frequency, and correlation of
updates can be observed and may reveal operational patterns or associations.

A [[ref: Resolver]] that retains the latest `versionId` of each DID it
resolves, as permitted in
[Comparing Copies of a DID Log](#comparing-copies-of-a-did-log), keeps a
record of which DIDs it has resolved and when. A [[ref: Resolver]] offered as
a service to other parties is advised to protect that record as it would any
request log, to retain it no longer than it is useful, and to say in its
documentation whether it retains it; a client concerned about this can use a
local [[ref: Resolver]].

### Stored Data Compromise

[[ref: DID Logs]] may be stored on web servers, by [[ref: watchers]], and in
other stores. A compromise of any of them could allow tampering with the
stored copy. Cryptographic signatures protect integrity, so tampering is
detected, but confidentiality is not provided.

### Correlation and Identification

A `did:vh` DID contains no domain name, so it does not by itself link its
controller to a domain owner. `src` values, `alsoKnownAs` entries and service
endpoints can reintroduce such a link, so [[ref: DID Controllers]] that want
to avoid it are advised not to publish them. The use of a static DID and
public [[ref: DID Log]] entries can enable correlation of activities over
time; [[ref: DID Controllers]] are advised to avoid embedding personal
identifiers or unnecessary service endpoints in DID documents.

For peer-to-peer use, a [[ref: DID Controller]] can create a separate `did:vh`
DID for each relationship, so that the relationships cannot be correlated by
identifier.

### Unsolicited Traffic

Distributing a [[ref: DID Log]] does not inherently solicit inbound traffic
beyond normal DID resolution. However, public exposure of service endpoints in
the DID Document may increase unsolicited interactions. [[ref: DID Controllers]] are advised to avoid publishing unnecessary endpoints.

### Right to Erasure ([GDPR Art. 17](https://gdpr-info.eu/art-17-gdpr/))

A `did:vh` [[ref: DID Log]] is likely to be held in several places — by
[[ref: watchers]], at web locations, and by peers. A [[ref: DID Controller]]
cannot remove every copy itself. The [[ref: watcher]] deletion operation
defined in [Watcher HTTP API Operations](#watcher-http-api-operations) can be
used to request removal from [[ref: watchers]]; how such requests are handled
is best defined by ecosystem governance.

### Secondary Use

Information published in the [[ref: DID Log]] may be repurposed by third
parties. [[ref: DID Controllers]] are advised to minimise the publication of
data that could be used for purposes beyond the intended use.

### Disclosure

All data in the [[ref: DID Log]] may become publicly accessible, so sensitive
data does not belong in it.

### Exclusion

`did:vh` does not require a [[ref: DID Controller]] to own a domain name or
operate any particular infrastructure. A [[ref: DID Log]] can be published on
a web-hosting platform that serves static files over HTTPS, kept with
[[ref: watchers]] or in an ecosystem's registry, or exchanged directly with
peers. This reduces barriers to participation, and because the DID does not
change when the log is kept elsewhere, a [[ref: DID Controller]] who loses
access to one host can move the log to another without affecting the DID.

### No Phone Home Mitigations

A privacy concern in decentralized identity ecosystems is the possibility of
an issuer of identity information (such as Verifiable Credentials) being
notified when and where individuals present those credentials. This "phone
home" surveillance problem (such as described by
[nophonehome.com](https://nophonehome.com)) can occur if the presentation of a
credential requires contacting the issuer's infrastructure, either directly or
indirectly, in a way that can be linked to the credential holder.

Sourcing a `did:vh` [[ref: DID Log]] need not involve the
[[ref: DID Controller]] at all: the log can be passed with the DID, retrieved
from a [[ref: watcher]], or held from an earlier resolution. A `src` value or
service endpoint chosen by an issuer, however, may point at infrastructure the
issuer operates, as may related resources retrieved at presentation time —
for example, revocation registries or status lists. If these resources are
implemented in a way that enables linking access patterns to individual
credential holders, the [[ref: DID Controller]] could use that information for
surveillance.

Privacy-respecting credential issuers, credential holders, and verifiers all
have a role in preventing "phone home" surveillance. The following practices
can help:

- **Privacy-respecting Issuers (including DID Controllers hosting VC-related
  resources)**
  - Do not design or deploy credential-related resources (such as revocation
    registries) in a way that enables the identification of individual
    holders at presentation time.
  - Use privacy-preserving designs — such as compact status lists, batching,
    and/or large revocation registries that provide "lost in a crowd"
    anonymity — to prevent correlation of access patterns to specific
    credential holders.
  - Use HTTP caching headers (e.g., `Cache-Control`, `ETag`) to enable CDNs,
    browsers, and resolvers to cache DID resources efficiently.
- **Holders**
  - Pass the issuer's [[ref: DID Log]] along with the credential, or use
    [[ref: watchers]], trusted intermediaries, or privacy-enhancing network
    tools to retrieve DID and status data without revealing the holder's
    location or identity to the issuer.
  - Separate in time interactions with the issuer from credential
    presentations to verifiers.
- **Verifiers**
  - Be flexible in the timeliness of credential status checks, and consider
    omitting them entirely when risk is low.
  - Prefer [[ref: DID Logs]] passed by the holder, or held from earlier
    resolutions and checked against [[ref: watchers]], over retrieving from
    locations operated by the issuer.
  - Use privacy-enhancing network tools or trusted intermediary resolvers to
    retrieve DID resources in a way that avoids revealing verifier identity or
    network location to the issuer or hosting provider.

A related risk is that an issuer may deliberately or inadvertently create
**holder-specific identifiers** for data elements that are expected to be
common across all holders — for example, by issuing personalized revocation
list URLs, unique resource paths, or holder-specific `src` values. This
enables tracking of specific holders even if the underlying credential is
otherwise privacy-preserving. Preventing this requires shared responsibility:
issuers are expected not to generate such holder-specific identifiers;
holders and verifiers can reject credentials or status mechanisms that
contain them; and independent third parties, including [[ref: watchers]], can
monitor issuer implementations to detect and report violations of this
principle.
