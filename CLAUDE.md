# CLAUDE.md — did:vh DID Method (DIF)

## About This Repository

This repo holds the `did:vh` DID Method specification, hosted by the Decentralized
Identity Foundation (`decentralized-identity/didvh`).

A `did:vh` DID is just `did:vh:<SCID>`: no location component, so the DID Log can be
sourced from anywhere (the context the DID was received in, a watcher, a peer, a web
location) and verified the same way.

Local, machine-specific notes (clone paths, remotes) belong in `CLAUDE.local.md`, which
is git-ignored. Keep this file relevant to anyone working on the DIF repo.

**Current state:** the Spec-Up template has been replaced. `spec/` (standalone did:vh,
v0.1 Editors Draft) is written, reviewed, and renders to `next/`; the watcher API is in
`watcherOpenAPI/`. `spec-vhlog/` is not started.

## Two Specs in One Repo

| Folder | What it is | Based on |
| ------ | ---------- | -------- |
| `spec/` | **Standalone did:vh** — a self-contained DID method spec. | The did:webvh Editor's Draft ([`decentralized-identity/didwebvh`](https://github.com/decentralized-identity/didwebvh) `spec/`, branch `main`) with the "web" aspect removed. |
| `spec-vhlog/` | **did:vh as a VH Log specialisation** — the same method, defining only the DID-specific parts and referencing VH Log for the rest. | The VH Log spec ([`decentralized-identity/vhlog`](https://github.com/decentralized-identity/vhlog), currently being moved there from [`swcurran/VH-Log`](https://github.com/swcurran/VH-Log)), starting from the pre-DIF did:vh draft. |

The two specs define the **same** DID method and must stay behaviourally equivalent: a
log valid under one is valid under the other. They should differ only in whether
material is stated inline (`spec/`) or by reference to VH Log (`spec-vhlog/`). When
changing one, check whether the other needs the matching change.

### `spec/` — did:webvh minus "web"

Derived from did:webvh's `spec/` (all eight files). Removed or replaced everything that
exists because the DID contains a web location:

- Method-specific identifier: `did:vh:<SCID>` only — drop the domain/port/path parts,
  "Case Sensitivity and Normalization", and "International Domain Names".
- "The DID to HTTPS Transformation" as the *way to locate* the log — replaced by
  resolution options that pass the log directly or by reference (`srcLog` /
  `srcWitness`; `src` is a DID URL parameter only), and a non-normative "Sourcing the
  DID Log" section (see Design Decisions). A web location remains one way to obtain the
  log.
- "DID Portability" and the `portable` parameter (nothing to port — no location).
- "Publishing a Parallel `did:web` DID" and "Parallel `did:web` DID URL Path Handling".
- DID URL path handling, the `#files` / `#whois` services and the `/whois` use case
  (left to the DID Controller; not mentioned in the spec).
- `method` parameter values become `did:vh:<version>`; file names stay `did.jsonl` /
  `did-witness.json`.
- Security/privacy considerations tied to DNS, domains, or HTTPS hosting (rewrite for
  location-independent sourcing; keep transport rules that apply to web locations).

Everything else from did:webvh (log entry structure, SCID, entry hashing, parameters,
pre-rotation, witnesses, watchers, cryptographic agility) is kept inline, adjusted for
`did:vh`. The DID-specific sections (resolution options, `src`, sourcing, peer-to-peer
updates, error names) come from the pre-DIF did:vh draft.

To keep `spec/` equivalent to `spec-vhlog/`, VH Log behaviour that did:webvh lacks is
written out inline in `spec/`: whole-second `versionTime`; the deactivating entry is the
last entry; controllers distribute only extending logs; witnesses approve at most one
child of an entry; no conflicting proofs in the witness file; "Comparing Copies of a DID
Log" (`checkWatchers`/`extraWatchers`/`minCopies`, `copies`/`warnings`); "Selecting a
Version"; and "Publishing and Retrieving DID Log Resources" (transport/SSRF rules, moved
out of the now non-normative Security Considerations). When VH Log changes any of these,
update `spec/` to match.

Neither spec defines DID URL path handling or `#files`/`#whois` services (the pre-DIF
draft has none either). The watcher notification uses did:webvh's `POST /log?did=`
(matching the did:webvh watcher OpenAPI), where VH Log uses `?id=` — to be reconciled.

The watcher API is defined in `watcherOpenAPI/watcher-v1.0.0.yml`, derived from
did:webvh's file of the same name and adapted for did:vh (`POST /log` with a `src` DID URL
or the log as the body; `POST /witness`; the witness file as an array). Keep it in step
with "Watcher HTTP API Operations" in `spec/specification.md`.

### `spec-vhlog/` — did:vh on VH Log

Start from the pre-DIF did:vh draft, which is already written as a VH Log
specialisation. Update it to:

- link to VH Log at its DIF home (presumably `https://identity.foundation/vhlog/`)
  rather than `swcurran.github.io/VH-Log`;
- link to this repo's DIF home rather than `swcurran/didvh`;
- describe its relationship to `spec/` (the standalone version of the same method);
- bring it up to date with `spec/`, which changed a lot during review. The pre-DIF draft
  still has the old design: a `src` resolution option naming a directory, `.well-known`,
  integer-as-string metadata, "terminate resolution". Take the current behaviour from
  Design Decisions below and from `spec/`, and handle the VH Log items in "Port to VH
  Log" (Open TODOs) at the same time.

Prior art for "a DID method on VH Log": did:webvh's experimental `spec-exp/` (branch
`exp-wvh-on-vhlog`).

## Repo Setup

Tooling (Spec-Up 0.11.6, `render.mjs`/`edit.mjs`/`spec-refs.mjs`, `.npmrc`, the
`main`-branch render workflow, `LICENSE.md`) came from the pre-DIF did:vh repo.
`.gitignore` must keep `CLAUDE.local.md` and `.claude/settings.local.json` ignored.

When `spec-vhlog/` is added:

- `specs.json`: add a second entry for `./spec-vhlog` with its own `output_path`
  (proposal: `./vhlog`, following did:webvh's `./experimental` pattern — to be
  confirmed) and its own `spec_refs`.
- `index.html`: keep redirecting to the `spec/` render; add a link to the `spec-vhlog/`
  render.
- `README.md`: update the layout and the "Relationship" section for the two specs.

## Notes for Claude

- **Spec-Up:** v0.11.6 from npm. Add external references to `spec_refs` in `specs.json`
  (made available to `[[spec:name]]` by `spec-refs.mjs`). Do NOT use `external_specs`:
  in 0.11.6 it only fetches other Spec-Up pages for `[[xref:]]` terms, and fetching
  non-Spec-Up pages produces huge jsdom CSS error dumps.
- **Don't split `[[ref: ...]]` / `[[spec: ...]]` across lines** — Spec-Up resolves only
  the first line's text (e.g. `[[ref: log` + newline + `entries]]` becomes "log").
- **Rendered output** folders are git-ignored: don't edit or commit them.
- **Check after edits:** `npm run render`, confirm every `href="#..."` in each rendered
  `index.html` has a matching `id`, and that every VH Log anchor used in `spec-vhlog/`
  exists in the rendered VH Log spec. Spec-Up derives anchors from heading text, so a
  renamed heading breaks inbound links — check VH Log's and did:webvh's links into this
  repo before renaming.

## Design Decisions (open to revision)

These apply to both specs. Some come from the pre-DIF draft; the rest were decided in
the review of `spec/` (November 2026).

- DID is `did:vh:<SCID>` only; the spec version is in the `method` parameter
  (`did:vh:1.0`), not the DID.
- Log/witness file names `did.jsonl` / `did-witness.json`, as in did:webvh.
- **Resolve vs. source are separated.** Read (Resolve) is normative: `resolve(did,
  resolutionOptions)` with exactly one DID Log per call. The log and the witness file are
  each passed directly (`didLog` JSONL / `didWitness`) or by reference (`srcLog` /
  `srcWitness`: the URL of the file itself — https, or loopback http, possibly with a
  query; a local `file:` file; or another supported scheme — retrieved as given, no path
  appended), independently, so they can be mixed. `srcWitness` is fetched only when
  witnesses are required; missing = no proofs. Plus
  `versionId`/`versionTime`/`versionNumber`. The resolver does not discover logs itself;
  it compares extra copies only when asked (`checkWatchers` / `extraWatchers` /
  `minCopies`). "Sourcing the DID Log" is a separate, non-normative section.
- `includeDidLog` (did:vh only, not in VH Log for now) returns the verified reference
  copy (to its last valid entry, not cut to a target version) and witness file as
  `didLog` / `didWitness` in didResolutionMetadata, in the same form as the input
  options; nothing on error.
- `src` is a **DID URL parameter only** (did:scid-compatible base location). The
  dereferencer derives `srcLog`/`srcWitness` by appending `/did.jsonl` and
  `/did-witness.json` — no `.well-known` (dropped: the DID isn't the domain's, and
  did:scid doesn't use it), for every scheme and an empty path; the
  `did.jsonl`/`did-witness.json` names are required only there.
- `file:` and loopback-http `srcLog`/`srcWitness` only for resolvers local to their client;
  network-facing resolvers reject them (`invalidOptions`); a `src` DID URL parameter must be
  https or another hierarchical scheme, never `file:`/`http` (`invalidDidUrl` otherwise).
  Only `src` and the version parameters can be DID URL parameters.
- Peer-to-peer updates send only new entries (plus `did-witness.json` when needed); the
  normative rule lives in Update (Rotate) → Applying Peer-to-Peer Updates.
- No `portable` parameter, no parallel did:web, and no DID URL path handling or
  `#files`/`#whois` services — those are left to the DID Controller and not mentioned.
- Error names follow did:webvh's camelCase style — still to be aligned with DID
  Resolution's current `INVALID_DID`-style names.

## Open TODOs

- Add a comparison with did:cel (<https://w3c-ccg.github.io/did-cel-spec/>), e.g. a
  "Related DID Methods" overview section covering did:scid and did:cel.
- Explore `src` naming a DID method that stores the log (as did:scid allows), and other
  storage options (DID-Linked Resources, content-addressed storage, registries).
- Pick up VH Log's planned mandatory key pre-rotation (`updateKeys` removed,
  `nextKeyHashes` → `prerotationHashes`, `eddsa-jcs-prerotation-2026` cryptosuite) —
  inline in `spec/`, by reference in `spec-vhlog/`.
- Move material shared with did:webvh (resolution metadata, witness `did:key` rules) into
  VH Log so `spec-vhlog/` references VH Log rather than did:webvh.
- Review the error names and the `copies`/`warnings` placement in didResolutionMetadata.
- **Raise on did:webvh** (fixed in `spec/` Authorized Keys, `updateKeys` parameter,
  Deactivate, and invalid-entry example 8): did:webvh only implies that no entry may
  follow `updateKeys: []`. It doesn't say so outright, and it never defines when
  pre-rotation is "active" for an entry. An implementation that treats an entry setting
  `nextKeyHashes` for the first time as pre-rotation-active (signed by its own
  `updateKeys`) lets anyone re-open a DID whose `updateKeys` is `[]`, a takeover. did:vh
  now states: pre-rotation is active for an entry only if the `nextKeyHashes` in effect
  *before* it is non-empty; an empty active `updateKeys` means the entry fails
  verification.
- **Port to VH Log** (made in `spec/` first; VH Log is not yet changed — do it when work
  on VH Log / `spec-vhlog/` starts, so the two stay equivalent):
  - "Results of Processing the Log": an entry that fails a check *fails verification*
    (no "terminate resolution"), processing stops there, and the outcomes are defined
    (first entry → error; target version before/after the failure; last valid entry +
    `invalid-entries` warning). Uniform wording: "MUST cause the entry to fail
    verification" for every entry-level check.
  - Well-formed but unsupported `method` version → `featureNotSupported` (not
    `invalidDid`), and no fallback to the last supported entry.
  - "Forked Logs" Security Considerations subsection (forks prove a signer of both
    branches, incl. a key-deriving/quantum attacker; resolver must not pick a branch;
    `logForked` is a DoS not a takeover; register new DIDs with watchers early; recovery
    = new DID), plus the cross-references from Comparing Copies, Log Freshness and Post
    Quantum Attacks, and the "no version can be relied on" wording.
  - Already ported to VH-Log's working tree (uncommitted): optional retained
    `versionId` (MAY + MUST compare), `retained-version-not-found` warning-only, and the
    Surveillance paragraph.

## Authoring Guidelines

- Normative requirements use RFC 2119 terms (MUST, SHOULD, MAY); each normative
  statement should be individually testable.
- Security and Privacy Considerations sections are **non-normative** (following DID Core
  §9/§10): put every RFC 2119 requirement in the normative body and have the
  considerations sections describe threats and link to it.
- Wording: say "pass directly" / "pass by reference" (not "conveyance"); "web location"
  only for https / loopback-http `srcLog`/`srcWitness` references and https `src` values.
- Lowercase must/should/may: use an RFC 2119 keyword if the sentence states a
  requirement; otherwise reword ("can", "needs to", "has to"). "may" meaning "might" is
  fine.
- Entry-level checks say "MUST cause the entry to fail verification" (not "reject" or
  "terminate resolution"); the outcome is defined once, in "Results of Processing the
  Log".
- In `spec-vhlog/`, cross-reference VH Log where behaviour is inherited and keep only
  DID-specific content. In `spec/`, keep the text self-contained.
