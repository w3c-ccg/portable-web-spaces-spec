# Portable Web Spaces specification

> Portable Web Spaces is a general-purpose permissioned cloud storage API:
> CRUD over an HTTP hierarchy of Spaces, Collections, and Resources, secured by
> object-capability (zCap) authorization. Storage is private by default -- only
> a Space's controller may act on it unless a policy grants otherwise -- and
> requests that are not authorized return "not found" rather than "not authorized",
> so the existence of a resource is never leaked. The specification is layered:
> a minimal conformant server implements only Resource CRUD plus the authorization
> profile, while listing, collection and space management, multi-tenancy, and
> extensions (linksets, policies, metadata, export, backends, query, quotas) are
> each optional.

This repository contains the Portable Web Spaces specification (in [ReSpec 
Markdown](https://respec.org/docs/#markdown) format), and its companion
[Portable Web Spaces Authorization Profile](https://w3c-ccg.github.io/wallet-attached-storage-spec/authz-profile/)
(source in [`authz-profile/`](./authz-profile/)), which defines the baseline
zCap authorization profile and the `PublicCanRead` policy type, and
[Encrypted Collections for Portable Web Spaces](https://w3c-ccg.github.io/wallet-attached-storage-spec/ec/)
(source in [`ec/`](./ec/)), the client-side profile for end-to-end encrypted
Collections.

For LLM consumption, [`llms.txt`](llms.txt) summarizes the spec and links to its
full text, following the [llmstxt.org](https://llmstxt.org/) convention.

## Table of Contents

- [Background](#background)
- [Code of Conduct](#code-of-conduct)
- [Contributing](#contributing)
- [Status](#status)
- [Usage](#usage)
  - [Editing](#editing)
  - [Testing](#testing)
  - [Publishing](#publishing)

## Background

You can access the latest version of this specification at:

https://w3c-ccg.github.io/wallet-attached-storage-spec/

The W3C Community Group working on this specification can be found here:

https://w3c-ccg.github.io/

## Code of Conduct

W3C functions under a [code of conduct](https://www.w3.org/Consortium/cepc/).

## Contributing

We encourage contributions meeting the
[Contribution Guidelines](https://github.com/w3c-ccg/community/blob/main/CONTRIBUTING.md). 
While we prefer the creation of issues and Pull Requests in the GitHub 
repository, discussions often occur on the
[public-credentials](http://lists.w3.org/Archives/Public/public-credentials/)
mailing list as well.

## Status

This document is a draft technical specification produced by the W3C Credentials
Community Group, with the intent of progressing toward a W3C standard. Feedback
is welcome via the [issue tracker](https://github.com/w3c-ccg/wallet-attached-storage-spec/issues)
or the [public-credentials mailing list](https://lists.w3.org/Archives/Public/public-credentials/).

## Usage

### Editing

The specification source is in [`spec.md`](./spec.md), in
[ReSpec Markdown](https://respec.org/docs/#markdown) format. The authorization
profile's source is [`authz-profile/authz-profile.md`](./authz-profile/authz-profile.md),
with its own ReSpec shell at [`authz-profile/index.html`](./authz-profile/index.html).
The Encrypted Collections profile's source is [`ec/spec.md`](./ec/spec.md),
with its ReSpec shell at [`ec/index.html`](./ec/index.html).

### Testing

For testing locally, you can `npm i -g http-server`, and then:

```
http-server ./
```

### Publishing

The published site is built by [`.github/workflows/publish.yml`](.github/workflows/publish.yml)
on every push to `main`, and deployed to GitHub Pages. The workflow runs the
[ReSpec CLI](https://respec.org/docs/#respec-cli) over `index.html`,
`authz-profile/index.html`, and `ec/index.html` and publishes:

| Path                              | Contents                                                                                                                             |
|-----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| `/`                               | A static, pre-rendered snapshot of the spec. Readable without JavaScript, so search engines, `curl`, and agents see the full text.   |
| `/live/`                          | The original ReSpec page, rendered client-side from `spec.md` at view time.                                                          |
| `/spec.md`                        | The Markdown source.                                                                                                                 |
| `/llms.txt`                       | The [llmstxt.org](https://llmstxt.org/) index.                                                                                       |
| `/authz-profile/`                 | A static snapshot of the Authorization Profile.                                                                                      |
| `/authz-profile/live/`            | The Authorization Profile's ReSpec page, rendered client-side.                                                                       |
| `/authz-profile/authz-profile.md` | The Authorization Profile's Markdown source.                                                                                         |
| `/ec/`                            | A static snapshot of the Encrypted Collections profile.                                                                              |
| `/ec/live/`                       | The Encrypted Collections profile's ReSpec page, rendered client-side.                                                               |
| `/ec/spec.md`                     | The Encrypted Collections profile's Markdown source.                                                                                 |

Pull requests build the snapshot as a check but do not deploy it. To render a
snapshot yourself:

```
npx respec@latest --src index.html --out snapshot.html --localhost
(cd authz-profile && npx respec@latest --src index.html --out ../authz-snapshot.html --localhost)
(cd ec && npx respec@latest --src index.html --out ../ec-snapshot.html --localhost)
```

The repository's Pages source must be set to **GitHub Actions** (rather than
"Deploy from a branch") for the workflow to publish.
