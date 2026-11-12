# The did:vh DID Method Specification

The `did:vh` (DID + Verifiable History) DID method gives a DID a
cryptographically verifiable history without tying it to any location. A
`did:vh` DID is just `did:vh:<SCID>`: it contains no domain name or path, so
its DID Log can be sourced from anywhere — the context in which the DID was
received, a watcher, a peer, or a web location — and is verified the same way.

Read the spec (Editors Draft): <https://identity.foundation/didvh/>

## Current Status of the Specification

**Editors Draft v0.1.** This is pre-standard work. Do not implement against
this specification until it reaches Draft status.

## Relationship to Other Specifications

- **[did:webvh](https://identity.foundation/didwebvh/)** — `did:vh` is derived
  from the `did:webvh` Editors Draft, with the domain and path removed from the
  DID. The log format, SCID and entry-hash algorithms, key authorization,
  pre-rotation, witnesses and watchers are the same. The features that depend
  on a web location — the DID-to-HTTPS transformation, portability, and
  parallel `did:web` publishing —
  are replaced by resolution options that pass the DID Log to the resolver
  directly (`didLog`, `didWitness`) or by reference (`srcLog`, `srcWitness`),
  and a `src` DID URL parameter. The specification's "Relationship to
  `did:webvh`" section lists the differences.
- **[VH Log](https://github.com/decentralized-identity/vhlog)** — the
  Verifiable History Log specification generalises the `did:webvh` log
  mechanism for any versioned object. A second version of this specification,
  defining `did:vh` as a VH Log specialisation, is planned for this repository
  (`spec-vhlog/`). Both versions will define the same DID method.
- **did:scid** — `did:vh:<SCID>` is comparable to the `did:scid:vh` format of
  the proposed ToIP `did:scid` metamethod, and adopts its `src` parameter.
  Unlike `did:scid:vh:1:<SCID>`, the spec version is not in the DID; it is in
  the log's `method` parameter.

## Contributing to the Specification

Pull requests (PRs) to this repository may be accepted. Each commit of a PR
must have a DCO (Developer Certificate of Origin -
[https://github.com/apps/dco](https://github.com/apps/dco)) sign-off. This can
be done from the command line by adding the `-s` (lower case) option on the
`git commit` command (e.g., `git commit -s -m "Comment about the commit"`).

The specification is written in [Spec-Up] Markdown (v0.11.6 from npm), in the
`spec/` folder. See the [Spec-Up Documentation] for its features. Rendering it
locally requires `node` and `npm`:

```sh
npm install
npm run render   # render once to next/
npm run edit     # re-render on change
```

Load the root `index.html` in a browser to be redirected to the rendered
specification. The `.npmrc` file sets `node-options=--dns-result-order=ipv4first`,
which is needed on machines where IPv6 is broken and is harmless elsewhere.

### Repository Layout

```text
spec/               # Specification source (Spec-Up Markdown)
  header.md         #   title, status, editors
  abstract.md
  overview.md
  specification.md  #   the normative body
  security_and_privacy.md
  definitions.md    #   terminology ([[def:]] terms)
  references.md
  version.md        #   changelog
watcherOpenAPI/     # Watcher HTTP API (OpenAPI), derived from did:webvh's
  watcher-v1.0.0.yml
specs.json          # Spec-Up configuration, including `spec_refs`
spec-refs.mjs       # Spec-Up plugin that adds `spec_refs` to [[spec:]] lookups
render.mjs          # Render once (npm run render)
edit.mjs            # Render on change (npm run edit)
index.html          # Redirect to the rendered spec in next/
next/               # Rendered output (git-ignored)
```

### External References

References not in Spec-Up's bundled reference data are added to the
`spec_refs` array in `specs.json`, with entries of the form
`{ "name": { "href", "title", "rawDate", "authors", "status" } }`. The
`spec-refs.mjs` plugin makes them available to `[[spec:name]]` references.

### Heading Anchors

Spec-Up derives section anchors from heading text, so renaming a heading
breaks links to it from other specifications. Avoid renaming headings, or
update the linking specifications when you do.

## Publishing

On each push to `main`, the `spec-up-render` GitHub Action
(`.github/workflows/render-specs.yml`) renders the spec and publishes the
working tree, including the rendered `next/` folder, to the `gh-pages` branch,
which GitHub Pages serves. The root `index.html` redirects to the Editors Draft
in `next/`.

[Spec-Up]: https://github.com/decentralized-identity/spec-up
[Spec-Up Documentation]: https://identity.foundation/spec-up/
