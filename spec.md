<div class="remove">

# Portable Web Spaces v0.5

**Abstract:** Portable Web Spaces is a general purpose permissioned
storage API -- CRUD over an HTTP hierarchy with object-capability authorization.

**Status:** Experimental W3C CCG draft, undergoing regular revisions.
Rendered version: https://w3c-ccg.github.io/wallet-attached-storage-spec/

</div>

## Introduction {#introduction}

The Portable Web Spaces (PWS) specification brings together the lessons
learned from many attempts to standardize permissioned cloud storage over
the years.

This specification aims to provide:

* A set of **design constraints, goals, and requirements** for PWS, see **Appendix
  [[[#goals-and-requirements]]]**.
* A tiered composable **data model** for storage primitives
* An **HTTP API** binding for storage operations (other bindings, such as JSON-RPC
  or CBOR-based RPCs, are left for future work).
* An authorization profile for use with this storage, see
  [[[#pws-authorization-profile-v0-1]]].

<div class="note">
This document is deliberately comprehensive: it specifies both a small required
core and a set of optional extensions. A conformant minimal server implements
only Resource CRUD ([[[#resources-and-blobs]]]) and the authorization profile
([[[#pws-authorization-profile-v0-1]]]); every other endpoint group is OPTIONAL.
See [[[#scope-and-conformance-profiles]]] for the full conformance-tier map, and
[[[#quickstart-your-first-request]]] to watch a first request succeed.
</div>

### Version History and Roadmap {#version-history}

<div class="note">
This subsection is non-normative.

* **v0.1** (January 2025) -- Initial version, created for the MIT Digital
  Credentials Consortium in collaboration with Benjamin Goering, as the
  "Wallet Attached Storage" specification.
* **v0.2** (April 2026) -- Added examples and editorial fixes. Snapshot at
  <https://wallet.storage/spec>.
* **v0.3** (through June 2026) -- Initial incubation at MIT DCC, based on
  implementation experience.
* **v0.4** (mid-July 2026 - September 2026) -- Migrated to the W3C Credentials
  Community Group (CCG)'s Capability Based Storage Task Force for further
  incubation. Goal: updating spec to match deployments and learnings.
* **v0.5** (September 2026 - now) -- Breaking changes, refactoring, community
  input. Renamed from Wallet Attached Storage to Portable Web Spaces.
  Breaking changes so far:
  * The description of a container moved to its `meta` sub-resource at every
    level, and a Collection's description merged with its Metadata object.
    `GET`/`PUT` at `/space/{space_id}/meta` read and write the
    [[[#space-metadata-data-model]]]; `GET`/`PUT` at
    `/space/{space_id}/{collection_id}/meta` read and write the
    [[[#collection-metadata-data-model]]], which now carries the members the
    Collection Description used to hold. The term "Collection Description" is
    retired.
  * A container URL is canonically written with a trailing slash, and a
    trailing slash no longer distinguishes two operations (see
    [[[#reading-this-document]]]).
  * The Space is an ordinary container: `GET /space/{space_id}/` lists its
    Collections and `POST /space/{space_id}/` creates one. The
    `/space/{space_id}/collections/` endpoint is retired; its segment stays
    reserved.
  * A method a server does not implement at a reserved endpoint answers
    `405 Method Not Allowed` (see [[[#methods-at-reserved-endpoints]]]). This
    replaces the earlier choice between `405` and a [=reserved-id=] (409)
    error at the Collection `meta` path. A `PUT` at a container URL is now a
    MUST `405` rather than a SHOULD.
  * An update of a Collection's Metadata object that omits `backend` keeps the
    stored backend selection, rather than resetting it to the default.
  * Create Resource can no longer respond with [=reserved-id=], or with
    [=id-conflict=] for an existing id. The server generates the Resource id
    there, and a client that chooses one uses Update (or Create by Id)
    Resource. That operation can no longer respond with [=reserved-id=]
    either, since a reserved segment in the Resource position is a reserved
    endpoint (see [[[#methods-at-reserved-endpoints]]]).
  * A server publishes a [=service description=] listing every specification
    version it speaks, found through a `Link: <...>; rel="service"` header on
    every response (see [[[#service-description]]]). Serving it is REQUIRED,
    and a response without the link identifies a v0.4 server.
  * Delete Space, Update (or Create by Id) Space, Delete Collection, and
    Update (or Create by Id) Collection now require a root capability
    invoked directly by the Space's controller, with exactly two named
    exceptions (see [[[#the-container-rule]]]). Delegated capabilities
    previously authorized a Delete Collection carrying the `DELETE` action,
    and a Delete Space or Update Space carrying any zcap scoped to the
    Space; both are now refused outside the exceptions.
  * A Backend description no longer carries a `features` array; conditional
    writes and key-epoch stamping are now baseline server requirements rather
    than backend-advertised affordances. `changes-query` is now advertised
    server-wide in the service description's `features` array rather than per
    backend. PWS-EC's optional affordances (`blinded-index-query` and
    `governed-history-logs`) are now tokens of its own version entry rather
    than backend features, and serving the chunk endpoints is now a plain
    requirement of PWS-EC conformance rather than a token at all.

No stored data moves across the v0.4-to-v0.5 path changes. The durable
artifacts to audit are capabilities. A delegated capability whose
`invocationTarget` was a v0.4 description URL with a `PUT` action has no
meaning under the v0.5 layout and needs re-delegation against the `meta` URL,
while a capability on a container URL keeps its prefix coverage unchanged. A
server MAY serve the two route tables together during a transition, listing
both versions in its service description.
</div>

### Reading This Document {#reading-this-document}

<div class="note">
This subsection is non-normative. It collects conventions that the rest of the
document relies on, so that a section read in isolation is still intelligible.

**`Authorization: ...` is a placeholder.** Every request example that carries an
`Authorization` header abbreviates a signed [=zCap=] (capability) invocation
(as opposed to a bearer token). Reads are authorized in PWS just as writes are.
The expanded form -- with the `Digest`, `Capability-Invocation`, and `Signature`
headers -- appears once, in [[[#performing-authorized-api-calls]]].

**Every path has one canonical form, and a trailing slash marks a container.**
A container -- a Space or a Collection -- is canonically written with a
trailing slash, and everything else without one. So
`/space/{space_id}/{collection_id}/` is the Collection, and
`/space/{space_id}/{collection_id}/{resource_id}` is a Resource in it. The two
forms of a path name the same thing. A request to the non-canonical form is not
a different operation, and a [=server=] SHOULD answer it with a
`308 Permanent Redirect` to the canonical form. No two paths defined by this
specification differ only by a trailing slash. Because a signed capability
invocation covers the request target (see
[[[#performing-authorized-api-calls]]]), a client that follows such a redirect
MUST sign again for the new target rather than replay its `Authorization`
header; sending the canonical form to begin with avoids the round trip.

**A container's description lives at its `meta` sub-resource.** `GET` on a
container lists its members and `POST` adds one, at every level: a Space's
members are its Collections, a Collection's are its Resources. What the
container *is* -- its name, its configuration -- is a separate document one
segment down, at `meta`: [[[#space-metadata-data-model]]] at
`/space/{space_id}/meta` and [[[#collection-metadata-data-model]]] at
`/space/{space_id}/{collection_id}/meta`. The same split holds for a Resource,
whose content is at its own URL and whose
[[[#resource-metadata-data-model]]] is at `meta` beneath it. Reading a
description is therefore always a `GET` of `meta`. The one exception is the
[=server=] itself, which has no container URL: its description is the
[=service description=], found by following the `service` link every response
carries (see [[[#service-description]]]).

**Paths are relative to a server root, which may be a subpath.** Every path in
this document, `/space/{space_id}/...` and `/spaces/`, is written from the
[=server=]'s root. That root is not necessarily an origin: a server MAY be
mounted under a path such as `https://example.com/pws/`. A client cannot
derive the root from a URL it holds, which is why the service description is
linked rather than placed at a fixed path, and why every URL the service
description carries is absolute.

**All examples share one Space.** Every example in this document uses the Space
id `81246131-69a4-45ab-9bff-9c946b59cf2e` on the host `example.com`. Path
segments in braces -- `{space_id}`, `{collection_id}`, `{resource_id}` -- are
placeholders for those identifiers.

**Normative lists live in the appendices.** Error `type` URIs are catalogued in
[[[#error-type-registry]]], path segments this specification reserves in
[[[#reserved-path-segment-registry]]], and client-side encryption schemes in
[[[#encryption-scheme-registry]]]. Those registries are normative: they, not the
surrounding prose, are where an implementation looks such values up.
</div>

### Use Cases

Initial use cases that are motivating this work:

* Sharing of W3C Verifiable Credentials from mobile credential wallets, as well
  as document sharing in general
* Serving as standardized storage (and a point of sync and interop) for
  multiple Verifiable Credential wallets.
* Enabling data portability and service provider interoperability for
  user-controlled social networking
* Enabling the "bring your own storage" architecture pattern of web app development
* Providing data storage and authorization frameworks for Agentic AI

### Scope and Conformance Profiles {#scope-and-conformance-profiles}

This specification represents a layered and modular approach to storage,
combining core features and optional extension points.

The layers below describe **conformance tiers**.
Each tier adds optional capability on top of the one before it, so an
implementer can stop at any tier and still be conformant. The normative body,
by contrast, is organized container-first (outermost to innermost: Spaces
Repositories, Spaces, Collections, then Resources), which is convenient as a
reference but is the reverse of the tiers. If you're new to the spec, scan the
profile table below for the conformance tiers, then walk through
[[[#quickstart-your-first-request]]] to watch a request succeed. The core tier
is just [[[#resources-and-blobs]]] plus [[[#pws-authorization-profile-v0-1]]].

| Profile               | Adds                                                                                             | Defining sections                                                                                     | Advertised as (see [[[#service-description-data-model]]]) |
|-----------------------|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| **Minimal**           | Resource CRUD (KV + blob read/write) + authorization + the service description                   | [[[#resources-and-blobs]]], [[[#pws-authorization-profile-v0-1]]], [[[#service-description]]]         | the version entry itself                                  |
| **+ Listing**         | list resources / collections / spaces                                                            | [[[#list-collection-operation]]], [[[#list-all-collections-operation]]], [[[#list-spaces-operation]]] | `listing`                                                 |
| **+ Collection mgmt** | create / manage collections in a Space                                                           | [[[#collections]]]                                                                                    | `collection-management`                                   |
| **+ Space mgmt**      | manage an individual Space                                                                       | [[[#read-space-operation]]] (Space endpoints)                                                         | `space-management`                                        |
| **+ Multi-tenant**    | create / manage many Spaces on a server                                                          | [[[#spaces-repositories]]]                                                                            | the `spaces` URL member                                   |
| **+ Extensions**      | linksets, policy, metadata, export, backends, query, quotas, encryption, changes-query | [[[#linksets]]]                                                                                       | one `features` token per extension, named as listed       |

Each tier stacks on the one above it. The **Minimal** profile is a permissioned
key/value CRUD API built from simple HTTP verbs and delegatable
capability-based authorization -- enough, on its own, to read and write any
resource (text, structured document, or binary blob) without collection or
space management -- plus the [=service description=] through which a client
learns which versions and tiers the server speaks. **Listing**, **Collection** and **Space** management will feel
familiar to anyone who has used a GUI front end for a database or file system.
**Multi-tenant** support lets a provider host many Spaces on one server. The
**Extensions** tier layers on optional features -- an access-policy resource,
user-writable metadata (for example, "tags" on binary files), a Space export
endpoint, pluggable [[[#backends]]], query, quotas, and client-side encryption
(via [Encrypted Data Vaults](https://identity.foundation/edv-spec/)) --
advertised server-wide in the service description's `features` array and, per
Space, through the linkset feature-detection mechanism (from [[RFC9264]]; see
[[[#linksets]]]).

**Normative status.** Section back-placement does not imply informative status.
Everything from [[[#introduction]]] through [[[#quotas]]] is normative, as are the
appendices [[[#pagination]]], [[[#reserved-path-segment-registry]]],
[[[#encryption-scheme-registry]]], and [[[#error-type-registry]]] (each of which
also carries an inline "This appendix is normative." banner). The remaining
appendices -- [[[#goals-and-requirements]]] and [[[#iana-considerations]]] -- are
informative. Optionality is orthogonal to normativity: many normative sections
describe OPTIONAL endpoint groups, but a server that implements them MUST follow
the stated requirements.

### Quickstart: Your First Request {#quickstart-your-first-request}

<div class="note">
This walkthrough is a non-normative tutorial, not a conformance requirement. It
assumes a server that already hosts a single Space as well as a
`messages` collection, and that you are that Space's [=controller=]. The goal is
to store one JSON Resource and read it back.
</div>

**Step 1: Write.** `PUT` a JSON document to a resource path under a collection.
A `204 No Content` confirms the write:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{"message":"hi"}
```

```http
HTTP/1.1 204 No Content
```

**Step 2: Read.** `GET` the same path to retrieve what you just stored:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

```http
HTTP/1.1 200 OK
Content-type: application/json

{"message":"hi"}
```

(The `Authorization: ...` placeholder stands for a signed zCap invocation).
PWS reads are authorized too, so the header is required on the
`GET` just as on the `PUT`. How that invocation is constructed is defined in
[[[#performing-authorized-api-calls]]], and its fully expanded form -- with the
`Digest`, `Capability-Invocation`, and `Signature` headers spelled out -- is
shown once in the worked example within that section. Producing that signature
requires a conformant PWS client: it is an Ed25519 `did:key` capability
invocation, not something you can hand-write with curl.

To go beyond a single pre-existing Space -- to create your own Space, or
delegate access to others -- see the conformance-profile table in
[[[#scope-and-conformance-profiles]]] and the full endpoint map in
[[[#api-summary]]].

### API Summary {#api-summary}

API summary at a glance.

Unless otherwise specified by the [=controller=], all **operations
require authorization**. This can be overridden by the controller via the
(optional) `/policy` endpoints. Hosting "public-read" resources, such as HTML
files for websites, or media files you can link to via `<img src="">`, is a
common use case.

The endpoints below divide into a **Core** profile that every conformant server
implements, and **Optional extensions** grouped by feature. Each row links to the
section that defines its operation. Rows marked **Reserved** name a path this
specification anchors (see [[[#reserved-path-segment-registry]]]) but does not
yet define an operation for; a server MUST NOT repurpose a reserved path. A
method the server does not implement at a reserved endpoint is answered with
`405 Method Not Allowed`; see [[[#methods-at-reserved-endpoints]]].

#### Core

**Service Description:**

* `GET {service description URL}` -- [[[#read-service-description-operation]]].
  The URL has no fixed path; every response carries it in a
  `Link: <...>; rel="service"` header (see
  [[[#discovering-the-service-description]]]).

**Resource CRUD (Create, Read, Update, Delete):**

* `POST /space/{space_id}/{collection_id}/` -- [[[#create-resource-add-resource-to-collection-operation]]]
* `GET /space/{space_id}/{collection_id}/{resource_id}` -- [[[#read-resource-operation]]]
* `PUT /space/{space_id}/{collection_id}/{resource_id}` -- [[[#update-or-create-by-id-resource-operation]]]
* `DELETE /space/{space_id}/{collection_id}/{resource_id}` -- [[[#delete-resource-operation]]]
* Use a `HEAD` instead of a `GET` to check a resource's headers/metadata without
  fetching the whole resource. However, do not rely on this for atomic inserts
  (that is, to check if a resource exists before attempting to create it).
  Instead, use the backend-specific Transaction mechanisms when appropriate.

#### Optional extensions

**List resources in a Collection:**

If not implemented on a server, implies that only individual Key/Value operations
are supported.

* `GET /space/{space_id}/{collection_id}/` -- [[[#list-collection-operation]]]

**Manage Collections in a Space:**

If not implemented on a server, implies that collections are pre-configured or
implicit, controlled by the server.

* `POST /space/{space_id}/` -- [[[#create-collection-add-collection-to-a-space-operation]]]
* `GET /space/{space_id}/` -- [[[#list-all-collections-operation]]]
* `GET /space/{space_id}/{collection_id}/meta` -- [[[#read-collection-metadata-operation]]]
* `PUT /space/{space_id}/{collection_id}/meta` -- [[[#update-or-create-by-id-collection-operation]]]
* `DELETE /space/{space_id}/{collection_id}/` -- [[[#delete-collection-operation]]]
* `GET|PUT /space/{space_id}/{collection_id}/meta/log` --
  [[[#collection-governing-history-log]]] (available only on a server whose
  PWS-EC version entry advertises `governed-history-logs`; see
  [[[#service-description-data-model]]])

**Spaces Repository Endpoints -- Manage Spaces on a Server:**

If not implemented on a server, implies that any existing Spaces are
pre-configured and controlled by the server.

* `POST /spaces/` -- [[[#create-space-operation]]]
* `GET /spaces/` -- [[[#list-spaces-operation]]]

**Space Endpoints -- Manage an individual Space:**

* `GET /space/{space_id}/meta` -- [[[#read-space-operation]]]
* `PUT /space/{space_id}/meta` -- [[[#update-or-create-by-id-space-operation]]]
* `DELETE /space/{space_id}/` -- [[[#delete-space-operation]]]
* `POST /space/{space_id}/export` -- **Reserved / not yet specified.** Export
  (download) a Space's contents (all collections and resources).
* `POST /space/{space_id}/import` -- **Reserved / not yet specified.** Import
  (upload) a previously exported Space archive (the operation the registered
  [=invalid-import=] error kind anticipates).

**Advanced Resource Endpoints:**

* `GET /space/{space_id}/{collection_id}/{resource_id}/meta` -- [[[#read-resource-metadata-operation]]]
* `PUT /space/{space_id}/{collection_id}/{resource_id}/meta` -- [[[#update-resource-metadata-operation]]]

**Chunked Resource Endpoints** (see [[[#chunked-resources]]]; serving them is a
requirement of [[PWS-EC]] conformance, so a server that does not implement
[[PWS-EC]] MAY omit them entirely):

* `PUT|GET|HEAD|DELETE /space/{space_id}/{collection_id}/{resource_id}/chunks/{index}`
  -- store, read, head, delete a single chunk.
* `GET /space/{space_id}/{collection_id}/{resource_id}/chunks/` -- list a
  Resource's chunks.

**Policy Related Endpoints:**

Policy overrides are hierarchical and inherited. A policy set for the entire
Space applies to all its Collections and Resources (unless overridden by a
more specific policy, either at the Collection or Resource level).

* `GET|PUT|DELETE /space/{space_id}/policy` -- **Reserved / not yet specified.**
  CRUD on the policy object for the Space.
* `GET|PUT|DELETE /space/{space_id}/{collection_id}/policy` -- **Reserved / not
  yet specified.** CRUD on the policy object for the Collection.
* `GET|PUT|DELETE /space/{space_id}/{collection_id}/{resource_id}/policy` --
  **Reserved / not yet specified.** CRUD on the policy object for the Resource.

**Linkset / Discovery Endpoints:**

Required if Space endpoints or Collection endpoints are supported.

* `GET /space/{space_id}/linkset` -- [[[#space-linkset]]]
* `GET /space/{space_id}/{collection_id}/linkset` -- [[[#collection-linkset]]]

**Query Endpoints:**

* `POST /space/{space_id}/query` -- Cross-collection queries (backend-specific).
* `POST /space/{space_id}/{collection_id}/query` -- Queries within a Collection,
  discriminated by a request-body `profile`; catalogued in [[[#query-profile-registry]]].

**Backend Management Endpoints** (see [[[#backends]]]):

* `GET /space/{space_id}/backends` -- [[[#space-backends-available]]]
* `GET /space/{space_id}/{collection_id}/backend` -- [[[#collection-backend-selected]]]
  (the backend summary is also displayed in the Collection's Metadata object)

**Quota Endpoints** (see [[[#quotas]]]):

* `GET /space/{space_id}/quotas` -- [[[#quotas]]]: the Quota report object, grouped
  by available Backend (add `?include=collections` for a per-collection usage breakdown)
* `GET /space/{space_id}/{collection_id}/quota` -- [[[#quotas]]]: the Quota report
  object for the specific Collection (not all Backends will support per-collection
  quotas however)

## Terminology

<dl class="termlist definitions" data-sort="ascending">
  <dt><dfn data-lt="action|actions|allowedAction">action (allowedAction)</dfn></dt>
  <dd>The kind of operation a request performs on a target, named by a capability
    so it can be authorized. PWS uses the uppercase HTTP method names
    (<a>GET</a>, <a>POST</a>, <a>PUT</a>, <a>DELETE</a>) as its action
    vocabulary. See section
    [[[#authorization-actions-and-the-root-capability]]].</dd>

  <dt><dfn id="term-backend" data-lt="backends">backend</dfn></dt>
  <dd>A storage engine that a [=collection=]'s resources are physically stored
    on, registered at the Space level either by server configuration or by
    client-side "Bring Your Own Storage" registration. Backends are an OPTIONAL
    concern, orthogonal to the Space > Collection > Resource hierarchy: a
    Collection names one via its <code>backend</code> property, and is assigned
    the <code>default</code> backend when it does not. See section
    [[[#backends]]].</dd>

  <dt><dfn data-lt="chunks">chunk</dfn></dt>
  <dd>One opaque byte sequence of a chunked Resource, addressed by a
    non-negative integer index under the Resource's reserved <code>chunks</code>
    sub-path. The [=server=] stores a chunk exactly like a binary Resource
    representation and never parses it; framing and reassembly are the client's
    concern. Serving chunks is a requirement of [[PWS-EC]] conformance; a
    server that does not implement [[PWS-EC]] MAY omit the chunk endpoints.
    See section [[[#chunked-resources]]].</dd>

  <dt><dfn data-lt="collections">collection</dfn></dt>
  <dd>A namespace and configuration container for resources. Conceptually maps
    to folders (for file system like storage), buckets (for object storage), or
    database tables (for RDBMSs). See section [[[#collections]]].</dd>

  <dt><dfn data-lt="controllers">controller</dfn></dt>
  <dd>An entity that can make changes to a given object.</dd>

  <dt><dfn data-lt="did|dids">decentralized identifier (DID)</dfn></dt>
  <dd>See [[DID-CORE]].</dd>

  <dt><dfn data-lt="server|servers|instance|instances">instance, server</dfn></dt>
  <dd>A deployed instance of an application or service that implements this
    specification's API.</dd>

  <dt><dfn data-lt="policies|access control policy|access control policies">policy</dfn></dt>
  <dd>A JSON document with a required <code>type</code> property that declares what
    access a [=target=] grants to callers in general, independent of any [=zCap=]
    a caller might present. A policy is stored at the <code>/policy</code>
    auxiliary resource of a Space, Collection, or Resource. Policies are inherited
    most-specific-wins (Resource over Collection over Space), can only broaden
    access (never deny a caller holding a valid capability), and grant nothing
    when absent or of an unrecognized <code>type</code>. See section
    [[[#access-control-policies]]].</dd>

  <dt><dfn id="term-quota" data-lt="storage quota|storage quotas">quota</dfn></dt>
  <dd>A storage limit enforced per [=backend=], together with the <em>usage</em>
    measurement of the storage consumed against it. Quota reporting and
    enforcement are OPTIONAL and backend-dependent: a Space's per-backend report
    is read from its <code>/quotas</code> auxiliary resource, and a
    per-Collection breakdown from <code>/quota</code>. A write that would exceed
    a quota is rejected with [=quota-exceeded=] (507); a single upload larger
    than the backend's <code>maxUploadBytes</code> constraint is rejected with
    [=payload-too-large=] (413). See section [[[#quotas]]].</dd>

  <dt><dfn data-lt="root capabilities|root zcap">root capability</dfn></dt>
  <dd>The implied capability for a [=target=] whose [=controller=] is the Space's
    controller; it is the root of trust from which all other capabilities for
    that target are delegated. See section [[[#root-capability]]].</dd>

  <dt><dfn data-lt="service descriptions">service description</dfn></dt>
  <dd>The JSON document through which a [=server=] advertises, without
    authorization, every specification and version it implements and the
    server-wide URLs and affordances a client needs before it holds a Space.
    It has no fixed path: every response carries a <code>Link</code> header
    with the <code>service</code> relation pointing to it. See section
    [[[#service-description]]].</dd>

  <dt><dfn data-lt="target|invocationTarget|targets">target (invocationTarget)</dfn></dt>
  <dd>The resource a request acts on, including the full request URL (scheme,
    host, port, and path) -- and the scope a capability authorizes. A capability's
    `invocationTarget` MUST match the request target for the invocation to be
    valid. See section [[[#authorization-actions-and-the-root-capability]]].</dd>

  <dt><dfn data-lt="zcap|zCaps|capability|authorization capability">zCap (Authorization Capability)</dfn></dt>
  <dd>See [zCap Developer Guide](https://interop-alliance.github.io/zcap-developer-guide/) for more
    details.</dd>
</dl>

## Identifiers {#identifiers}

### Identifier Required Properties

Space, Collection, and Resource identifiers used in this specification are
required to have the following properties.

1. **URL-safety** - All characters in a given identifier MUST be URL-safe.
2. **Uniqueness** - All identifiers MUST be unique within a given container.
   That is: Space `id`s (denoted by `{space_id}` in URL templates) MUST be
   unique within a given [=server=], Collection `id`s (denoted by `{collection_id}`
   in URL templates) MUST be unique within a given Space, and Resource `id`s
   (denoted by `{resource_id}` in URL templates) MUST be unique within a given
   Collection.

### Identifier Length and Format

Identifier length limits are currently left to the implementer. However,
implementations SHOULD limit identifier length to their appropriate use case.

Identifier format constraints are currently left to the implementer. Common
identifier formats include:

* Random IDs (either UUIDv4 with hyphens or using a compact encoding).
  Not recommended for _extremely_ high throughput use cases, since
  the clients or servers that are generating them might be limited by available
  device entropy.
* Semi-random Time-sortable IDs (UUIDv7). Useful for cases involving logs,
  histories, feeds, and other situations where sorting by time is desirable.
* Content-based IDs (CIDs). Useful for data deduplication use cases.
* Human-readable IDs. Some use cases might require human-readable identifiers
  (for example, a user hosting a blog in their space might want meaningful collection
  and resource IDs such as `/posts/2020-01-01-hello-world`).

## Authorization {#authorization}

The ability to do cross-domain, operator-independent, standardized cloud storage
operations requires an authorization system that is:

* Modular and layered (for future agility / upgradability)
* Not limited to traditional domain-based "usernames and passwords"
* A hybrid, using object capability principles at its baseline, but also able
  to provide ACL or RBAC-like functionality for user convenience. In other words,
  the system needs to support both "anyone with the link can..." and "these are
  specific people and groups allowed to..." styles of access control
* Compatible with cross-domain replication
* Compatible with end-to-end client side encryption (but also not rely on
  encryption as the sole authorization method)
* "Private by default".
  That is, by default, unless otherwise specified, only the controller of a
  space (or of a collection or resource) is authorized to perform any operation
  (read, write, delete, etc)

As the state of the art in cross-domain authorization advances, we expect there
to be multiple profiles and specs that could be used to perform PWS API
calls. However, to start with, this specification will focus on a single minimal
authorization profile.

### PWS Authorization Profile v0.1 {#pws-authorization-profile-v0-1}

Like many authorization specifications, the PWS Authorization Profile tries
to address opposing tensions. On the one hand, to cover the full range of use
cases, it needs to be delegatable, revocable, secure, flexible, and thus
 capability-based. On the other hand, for ease of implementation and adoption,
and for maximum developer usability, the profile must make the most common
operations as simple and friction-free as possible.

To that end, the profile offers the following layered mechanisms.

1. **Root Access**: For basic admin CRUD operations, use the space's `controller`
   DID directly to sign API calls with HTTP Signatures.
2. **Public Read**: For the common "public read" use case (the typical web
   publishing workflow, where a site or a file is shared for anyone to access
   via an HTTP GET), use the simple `{ "type": "PublicCanRead" }` PWS
   Authorization syntax, see below.
3. **Advanced Delegatable Capabilities** ("anyone with the link..." style):
   Use zCaps [Authorization Capabilities v0.3](https://w3c-ccg.github.io/zcap-spec/)
4. **Policy Based Access Control** (including the familiar "share with this list
   of people or groups" style): Use the space's `linkset` property to point to
   a linkset that includes a URL to an access control policy document.

#### Authorization Specification Dependencies at a Glance {#authorization-specification-dependencies-at-a-glance}

The initial PWS Authorization Profile uses the following specifications.

1. Identity (for controllers or clients/agents): [DID 1.0](https://www.w3.org/TR/did-1.0/)
2. Capability data model: [Authorization Capabilities for Linked Data v0.3](https://w3c-ccg.github.io/zcap-spec/)
3. Protocol for getting authorization: Out of scope (implementers are encouraged
   to use VC-API, OpenId4VP, OAuth2, or GNAP, as appropriate)
4. Proof of Possession / authorization invocation: HTTP Signatures.
   MUST - [HTTP Signatures (Cavage draft 12)](https://datatracker.ietf.org/doc/html/draft-cavage-http-signatures),
   MAY - [[RFC9421]] HTTP Message Signatures (a future direction for this
   profile, see the note below)
5. Request body integrity: the `Digest` header, bound to the request
   signature -- see [[[#request-body-integrity-digest-header]]]
6. Access Control / Policy language data model: see
   [[[#access-control-policies]]] (`PublicCanRead` is the only normative type for
   v0.1)

<div class="ednote">
**Signature suite (transitional).** The signature suite of this profile is
the Cavage HTTP Signatures draft (draft 12). The covered-headers list uses
its pseudo-headers (`(key-id)`, `(created)`, `(expires)`,
`(request-target)`), and the worked examples carry an
`Authorization: Signature ...` header in its syntax. This matches every
current implementation. [[RFC9421]] HTTP Message Signatures (the
`Signature` and `Signature-Input` headers) is a future direction for this
profile. It will be adopted as a coordinated migration across the
implementations, together with the `Content-Digest` migration described in
[[[#request-body-integrity-digest-header]]].
</div>

#### Space `controller` and the Root of Trust {#space-controller-and-the-root-of-trust}

Conceptually, the space's controller serves as the root of trust and authorization
for any operations on the space or its collections or resources.
That is, any operation requiring an authorization MUST provide a chain of proof
all the way to the space controller, by one of the following:

1. Direct: Provide a root capability invoked directly by the controller, or
2. Delegated: Invoke a capability delegated to some other agent by the controller, or
3. Matching Policy: (if using any kind of access control policy mechanism) Match
   an authorization policy specified in the `linkset` property of the space. This
   resource is related to the space controller because initially, it can only be
   modified either by the controller or an authorized party delegated to by the
   controller.

Space `controller`s MUST be in the form of a [DID](https://www.w3.org/TR/did-1.0/).

For minimal compatibility, all PWS implementations MUST support the
[`did:key` DID Method](https://w3c-ccg.github.io/did-key-spec/), using the
Multikey encoding of `Ed25519` elliptic curve keys, as specified in the
[Multikey section of the CID spec](https://www.w3.org/TR/cid-1.0/#Multikey)
as the space `controller`.

A Space `controller` MAY use a DID method other than `did:key`, provided the
method is listed in the [[[#space-controller-did-method-registry]]]. Support for
every method other than `did:key` is OPTIONAL. A server refuses a `controller`
whose DID method it does not support with [=invalid-request-body=] (400),
whether at creation or on an update (see [[[#create-space-errors]]] and
[[[#update-or-create-by-id-space-operation]]]).

##### Current-key-set rule {#current-key-set-rule}

Whatever the DID method, the key material that verifies an invocation or a
delegation is the signer's DID document as resolved at the time of
verification. A signature verifies if and only if its verification method is
present in that document, under the verification relationship the operation
requires: `capabilityInvocation` for an invocation, `capabilityDelegation` for
a delegation. A `did:key` document never changes. A document of a method with
a mutable, verifiable history (see [[[#verified-log-did-methods]]]) changes
as that history is extended, so the set of accepted keys is the current one.
A delegation signed by a key that has since been removed from the delegator's
document stops verifying the moment that verification method leaves the
document, even though the capability itself was not revoked. Every capability
delegated onward from it stops verifying with it. Removing a key from a
controller's document is therefore an immediate, server-enforced withdrawal
of everything that key delegated, independent of any capability revocation
mechanism.

This rule applies to every DID that signs in a capability chain, not only to
the Space's `controller`. A capability rooted in a `did:key`-controlled Space
MAY be delegated to a DID of any method the server supports; the server
resolves that delegate's document the same way when it verifies the
delegate's invocation or onward delegation.

##### Setting a controller to an optional DID method {#setting-a-controller-to-optional-did-method}

A server that supports an optional DID method decides at which point it
admits a `controller` of that method. It MAY accept one at Space creation. It
MAY instead require that every Space be created with a `did:key` controller
and admit the optional method only through a later update of the existing
Space (see [[[#update-or-create-by-id-space-operation]]]); such an update
promotes the `did:key`-controlled Space to the new controller. In both cases
the proposed
`controller` MUST resolve and verify before it is stored; a proposed
controller that does not is refused with [=invalid-request-body=] (400). After
the controller is stored, later updates to the Space are authorized by the DID
document being resolved, so storing an unresolvable DID (a typo, a history
not yet published) would leave the Space with no party able to act on it.

When a space is created via an HTTP `POST` or `PUT` operation (see
[[[#http-api-post-spaces]]] and [[[#http-api-put-space-space_id]]]), the
controller for that space
is set explicitly. That is, a client specifies the `controller` as part of the
payload of the PUT or POST create space request, and the server MUST verify
that the invocation is authorized by that `controller`, by one of the first two
mechanisms above: either directly -- the signing key (key ID) used in the
headers is authorized in the `capabilityInvocation` section of the
`controller`'s DID document -- or via a capability delegated by the
`controller` to the signing DID. (The third mechanism, matching policy, does
not apply: no Space, and therefore no policy, exists yet.)

See [[[#http-api-post-spaces]]] below for examples of `controller`
determination and verification.

#### Performing Authorized API Calls {#performing-authorized-api-calls}

Unless otherwise explicitly allowed via access control policy (see below),
all PWS API calls require authorization.

This can be done in one of two ways:

1. (for admin-like root access) Use the `controller` DID directly to sign
   HTTP API requests using the HTTP Signatures specification, invoking the
   target's [=root capability=].
2. (for advanced delegatable use cases) Use HTTP Signatures in combination
   with [Authorization Capabilities v0.3](https://w3c-ccg.github.io/zcap-spec/),
   and include a capability invocation header in the API request.

Throughout this specification, the request examples abbreviate this header as a
placeholder, `Authorization: ...`, rather than reproducing a full signature or
capability invocation. In each case it stands for a credential constructed as
described in [[[#performing-authorized-api-calls]]].

#### Request Body Integrity (Digest Header) {#request-body-integrity-digest-header}

When an authorized request carries a body (a Resource write, a Space create,
and so on), this profile binds the body to the request's HTTP Signature so
that the payload cannot be substituted without invalidating the signature:

1. The client MUST include a `Digest` header whose value is the hash of the
   request body, carried as a `mh` (multihash) parameter: a multibase
   base64url-encoded (`u` prefix) multihash of the body's SHA-256 digest
   (Multihash and Multibase as defined in the
   [CID 1.0 specification](https://www.w3.org/TR/cid-1.0/)). For example:

   ```http
   Digest: mh=uEiCPO-qYr-z0GYV5F75-N1l8Rhjv4xIkKZsnbTZeZ7emSA
   ```

2. The `content-type` and `digest` headers MUST be included in the
   signature's covered (signed) headers list, alongside the Cavage draft-12
   pseudo-headers `(key-id)`, `(created)`, `(expires)`, and
   `(request-target)`, and the `host` and `capability-invocation` headers.
3. For any request that carries a `Content-Type` header, the server MUST
   require `digest` among the covered headers, and SHOULD independently
   recompute the digest of the received body and compare it to the `Digest`
   header value. A missing, malformed, or non-matching `Digest` on a request
   with a body is rejected with an [=invalid-authorization-header=] (400)
   error.

Bodyless requests (`GET`, `HEAD`, `DELETE`) carry no `Digest` header.

Example authorized write request, showing the `Digest` header and the
covered headers list (the space's [=controller=] invoking the
[=root capability=] for the target; line breaks within the `Authorization`
header are for display only):

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/photos/sunset.png HTTP/1.1
Host: example.com
Content-Type: image/png
Digest: mh=uEiCPO-qYr-z0GYV5F75-N1l8Rhjv4xIkKZsnbTZeZ7emSA
Capability-Invocation: zcap id="urn:zcap:root:https%3A%2F%2Fexample.com%2Fspace%2F81246131-69a4-45ab-9bff-9c946b59cf2e%2Fphotos%2Fsunset.png",action="PUT"
Authorization: Signature keyId="did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW#z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW",
  headers="(key-id) (created) (expires) (request-target) host capability-invocation content-type digest",
  signature="6GoRQ+rW69wBhNyERkafAXEZXZArezHvGRNUWC0HNI4Ss1xAiiMHdayS5aA2R6hLuYRNw6h9J9eCmQVMuHE1Bw==",
  created="1758150502",expires="1758151102"

...binary PNG bytes...
```

<div class="ednote">
**Digest vs Content-Digest.** The `Digest` header used by this profile
descends from [[RFC3230]] (Instance Digests in HTTP). [[RFC9530]] (Digest
Fields) obsoletes RFC 3230 and replaces `Digest` with `Content-Digest` /
`Repr-Digest`. The current PWS implementation stack uses the legacy header
with a multihash value; migration to `Content-Digest` (alongside the move to
[[RFC9421]] HTTP Message Signatures, see
[[[#authorization-specification-dependencies-at-a-glance]]]) is a future
direction for this profile.
</div>

#### Authorization Actions and the Root Capability {#authorization-actions-and-the-root-capability}

A capability invocation names an [=action=] that the invoked capability must
permit. PWS uses the uppercase HTTP method names as its action vocabulary:

* <dfn id="get-action">`GET`</dfn> -- read a Space, Collection, or Resource. A
  `HEAD` request is authorized as a `GET`.
* <dfn id="post-action">`POST`</dfn> -- create a child item in a container (add a
  Resource to a Collection, a Collection to a Space, or a Space to the Spaces
  Repository).
* <dfn id="put-action">`PUT`</dfn> -- create-by-id or replace: a Resource at
  its own URL, or a Space or a Collection through its `meta` sub-resource (see
  [[[#reading-this-document]]]).
* <dfn id="delete-action">`DELETE`</dfn> -- delete a Space, Collection, or
  Resource.

A request is authorized by a capability when all the following hold:

1. the capability's `invocationTarget` matches the request's [=target=] -- the
   full request URL (scheme, host, port, and path);
2. the capability's `allowedAction` includes the request's action (the HTTP
   method); and
3. the invocation is signed by a key the capability authorizes, carried as a
   valid HTTP Signature over the request (see
   [[[#performing-authorized-api-calls]]]).

##### Root Capability {#root-capability}

Every [=target=] has an implied **root capability** whose `controller` is the
Space's [=controller=]. It is identified by the URI `urn:zcap:root:` followed by
the percent-encoded target URL:

```json
{
  "@context": "https://w3id.org/zcap/v1",
  "id": "urn:zcap:root:https%3A%2F%2Fexample.com%2Fspace%2F81246131-69a4-45ab-9bff-9c946b59cf2e%2Fmessages%2Fhello-world",
  "invocationTarget": "https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

The Space [=controller=] MAY invoke the root capability directly -- signing the
request with a key listed in the `capabilityInvocation` section of the
controller's DID document -- to perform any operation. This is the "root access"
path. All other authorized access derives from a capability delegated, directly
or transitively, from this root.

##### Delegation {#delegation}

To grant another agent access, the [=controller=] (or any agent holding a
sufficiently broad capability) delegates a capability that names the grantee as
its new `controller`, the `invocationTarget` to scope it to, and the
`allowedAction`s to permit. A delegation MAY set an `expires` time. For example,
granting another DID read-only access to a single Collection:

```json
{
  "@context": "https://w3id.org/zcap/v1",
  "id": "urn:uuid:6c9f3a1e-2b4d-4f8a-9c1e-7d2b3a4c5e6f",
  "parentCapability": "urn:zcap:root:https%3A%2F%2Fexample.com%2Fspace%2F81246131-69a4-45ab-9bff-9c946b59cf2e%2Fmessages%2F",
  "invocationTarget": "https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/",
  "controller": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "allowedAction": ["GET"],
  "expires": "2026-12-31T23:59:59Z",
  "proof": { "...": "delegation proof signed by the parent capability's controller" }
}
```

The delegated capability is handed to the recipient out of band. The recipient
invokes it by signing a request with their own key and including the capability
in the `Capability-Invocation` header.

A [=target=] covers everything beneath it, so the choice of target is what
attenuates a grant. A capability on a Collection URL, as above, covers the
Collection's listing, every Resource in it, and its Metadata object -- which is
what "share this collection" means. To grant the Collection's metadata
alone, without its Resources, target the `meta` URL instead; that grant also
covers the governing history log beneath it (see
[[[#collection-governing-history-log]]]). The same holds one level up for a
Space.

##### The Container Rule {#the-container-rule}

Target attenuation stops working for four operations: `DELETE
/space/{space_id}/`, `PUT /space/{space_id}/meta` on a Space that already
exists, `DELETE /space/{space_id}/{collection_id}/`, and `PUT
/space/{space_id}/{collection_id}/meta`. A capability delegated at a
container's `invocationTarget` -- the Space URL or the Collection URL --
covers everything beneath it, including its data, so nothing about that
target tells apart "delete this container" from "delete a Resource inside
it". These four operations therefore MUST be authorized by a root
capability invoked directly by the Space's [=controller=]. A delegated
capability MUST NOT authorize any of them, with exactly two exceptions.

1. **Delete Space.** A delegated capability MAY authorize `DELETE
   /space/{space_id}/`, for a Space of any controller DID method, when the
   invoked capability's `invocationTarget` is exactly that Space's URL and
   its `allowedAction` is exactly `['DELETE']`. The shape is judged on the
   invoked capability itself: a parent capability further up its
   delegation chain MAY carry a broader `allowedAction` set, as long as the
   invoked capability narrows it to exactly `['DELETE']` on exactly the
   Space URL. Such a capability is not a data grant -- an `allowedAction`
   of `['DELETE']` alone cannot be used to read or write anything beneath
   the Space -- so it does not carry the hazard the rule otherwise guards
   against.

2. **Update (or Create by Id) Collection.** A delegated capability MAY
   authorize `PUT /space/{space_id}/{collection_id}/meta` when the
   invoked capability's `invocationTarget` is the Space's own canonical
   (trailing-slash) URL, rather than the Collection container URL, the
   Collection Metadata URL, or a Resource URL. A Space-level
   grant is not a Collection-level data grant, so it does not reopen the
   hazard either.

Both exceptions are judged on the shape of the capability alone -- its
`invocationTarget` and `allowedAction` -- independent of who signed it. A
server MUST NOT treat a capability as satisfying either exception because
of the verification relationship its signing key holds in a DID document
(for example, membership in a `capabilityDelegation` set); only the
capability's own shape, and the [=root capability=] its chain derives
from, decide whether an exception applies.

Creating a Collection (`POST /space/{space_id}/`) is unaffected by this
rule: it remains an ordinary attenuated operation, and a delegated
capability targeting the Space MAY authorize it, like any other create
by `POST`.

<div class="ednote">
**Revocation.** This specification does not yet define a revocation
operation, although its goals require that a grant can be withdrawn before it
expires. The current PWS implementation stack ships a Space-scoped revocation
endpoint (`POST /space/{space_id}/zcaps/revocations/{revocation_id}`; every
Space-rooted capability verification checks the presented delegation chain
against the recorded revocations). A future revision will specify the
operation and reserve its path segments.
</div>

#### Specifying Access Policy With Space Link Sets

To set access control policy for a space, use the `linkset` property.

Example (fetching a space's link set):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset HTTP/1.1
Host: example.com
Accept: application/linkset+json
Authorization: ...
```

Response:

```http
HTTP/1.1 200 OK
Content-type: application/linkset+json

{
  "linkset": [
    {
      "anchor": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/",
      "https://w3id.org/pws#policy": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/policy",
          "type": "application/json"
        }
      ]
    }
  ]
}
```

Example (fetching a specific policy document from the link set):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/policy HTTP/1.1
Host: example.com
Authorization: ...
```

```http
HTTP/1.1 200 OK
Content-type: application/json

{ "type": "PublicCanRead" }
```

#### Access Control Policies {#access-control-policies}

Capabilities answer the question "does the caller hold a credential that grants
this action?" Access control *policies* answer the complementary question
"does this target grant this action to callers in general (or to a named set of
principals)?" Policies are how a [=controller=] makes a target public-readable
(or, in future profiles, shares it with a list of people or groups) without
having to issue a capability to each caller.

A policy is a JSON document with a required `type` property, stored at the
`/policy` auxiliary resource of a Space, Collection, or Resource and
discoverable via the [=policy relation=] in the linkset (see [[[#space-linkset]]]
and [[[#collection-linkset]]]).

**Evaluation contract:**

* **Capability first, policy second.** A request is first checked against any
  capability invocation it carries. The effective policy is consulted only as a
  fallback, and it can only **broaden** access -- a policy never denies a caller
  who presents a valid capability.
* **Fail-closed.** An absent policy, or a policy whose `type` an implementation
  does not recognize, grants nothing.
* **Most-specific-wins inheritance.** The effective policy for a target is the
  one set at the most specific level that has a policy document: a Resource
  policy overrides a Collection policy, which overrides a Space policy.
* **Access kind.** For policy evaluation, the request action is reduced to a
  coarse access kind: [=GET=] (and `HEAD`) is a `read`; [=POST=], [=PUT=], and
  [=DELETE=] are a `write`.

##### `PublicCanRead`

Policy `type` values are defined by policy profiles, in the same way that
authorization is defined by an authorization profile. This specification does
not fix the set. The one type this version defines is `PublicCanRead`:

```json
{ "type": "PublicCanRead" }
```

It grants the `read` access kind to any caller (including unauthenticated ones)
and grants no write access. This is the canonical "public read" pattern -- for
example, hosting an HTML file or an image that anyone may `GET`, while writes
still require a capability. Setting it on a Space makes the whole Space
public-readable (subject to any more specific Collection or Resource policy);
setting it on a single Resource exposes only that Resource.

## Error Handling {#error-handling}

This specification uses [[RFC9457]] Problem Details for HTTP APIs for error
responses.

* Error responses SHOULD be returned using the `application/problem+json`
  content type.
* `type` and `title` properties are REQUIRED.
* `errors` array with `{ detail, pointer }` objects is encouraged where appropriate.

The `type` property is a URI identifying the _kind_ of problem (not the
operation), and the same `type` is reused across operations. See Appendix
[[[#error-type-registry]]] for the catalog of `type` URIs this specification
defines, along with their typical status codes and a canonical example response
for each kind.

When returning errors, keep in mind the principle of **maximum privacy**:
always "not found" instead of "not authorized". The _existence_ of a resource,
collection, or space, is by itself an item of sensitive information.
If a client makes an API call, and they have insufficient authorization to
perform that action, a "not found" error (such as HTTP 404) MUST be returned,
just as if that resource (or space or collection) did not exist.
To put it another way, an unauthorized client (meaning, either not carrying
any authorization in the request itself, or possessing insufficient permissions)
MUST NOT be able to discover the existence of a resource based on the error
response.

This privacy rule governs failures that would otherwise reveal whether a target
*exists*. It does not require masking failures that describe the *request
itself*:

* **Request and credential validation** failures: a malformed or missing
  request body, a missing `Content-Type`, or a capability invocation or
  signature that is present but malformed or fails to verify -- MAY be reported
  with precise status codes and `type`s (for example [=invalid-request-body=],
  [=missing-content-type=], [=invalid-authorization-header=], or
  [=controller-mismatch=]). These responses describe the request and do not
  disclose whether any particular target exists.
* **Authorization** failures against an existing target: a well-formed request
  that simply lacks sufficient privilege, including one that carries no
  authorization at all, MUST be reported as [=not-found=] (HTTP 404),
  indistinguishable from the target being absent.

"List Spaces" operations are an exception to 404 masking: rather than returning
an error, they return `200 OK` with only the subset of items the caller is
authorized to see (an empty `items` array if none) -- see [[[#list-spaces-operation]]].

## Common Behaviors

This section defines HTTP mechanics that are shared across the operations in
this specification, rather than belonging to any single endpoint: caching,
conditional requests (optimistic concurrency control), and pagination of list
responses.

### Caching

PWS relies on ordinary [[RFC9111]] HTTP caching and defines no caching layer of
its own. On the read side, a Resource's strong `ETag` validator (see
[[[#conditional-requests]]]) drives standard validation: a `GET` carrying
`If-None-Match: "<etag>"` yields `304 Not Modified` when the Resource is
unchanged. Servers SHOULD emit `ETag` (and MAY emit `Last-Modified`) on
`GET`/`HEAD` responses, and SHOULD mark responses to non-idempotent operations
as non-cacheable (for example, `Cache-Control: no-store`).

<div class="ednote">
Freshness lifetime and `Cache-Control` directive semantics beyond validation
are not yet specified.
</div>

### Conditional Requests {#conditional-requests}

A server MUST support [[RFC9110]] conditional requests to provide optimistic
concurrency control on every Resource write path. This mechanism prevents the
"lost update" problem, where two clients that both read version N of a Resource
each write version N+1 and the second silently clobbers the first.

A Resource carries a strong **`ETag`** validator that changes whenever its
stored content changes. Servers SHOULD return the `ETag` on `GET`/`HEAD`
responses. The validator is opaque to clients: how a server derives it is a
server-side concern.

A state-changing request (`PUT` or `DELETE`) MAY carry a precondition:

* **`If-Match: "<etag>"`** -- perform the write only if the Resource's current
  `ETag` matches it (an "update-if-unchanged"). If it does not match, the server
  MUST NOT perform the write and MUST respond with [=precondition-failed=]
  (`412`).
* **`If-None-Match: *`** -- perform the write only if the Resource does not yet
  exist (a "create-if-absent"). If it already exists, the server MUST NOT
  perform the write and MUST respond with [=precondition-failed=] (`412`).

A server MUST evaluate the precondition atomically with the write, so that two
concurrent writers cannot both observe the same prior version and both
succeed. A client recovers from a `412` by re-reading the current Resource,
re-applying its change on top of the new version, and retrying.

<div class="note">
The storage engine behind a [=backend=] need not offer a precondition
primitive of its own. A client never addresses the engine directly, so the
server can serialize writes itself (under a per-Resource lock, or a shared
lock or compare-and-swap when horizontally scaled) and mint its own opaque
validator, such as a content hash or an internal version counter. Emulating
over an engine that is also writable outside the server, as a
client-registered external provider often is, carries two hazards: an
out-of-band write leaves the validator describing stale content, and a
validator stored apart from the bytes can be orphaned by a crash between the
two writes. Neither has a client-side remedy; embedding the validator in the
stored object addresses the second.
</div>

As with [=id-conflict=], a server MUST verify the caller's authorization
before evaluating a precondition, so a `412` is only ever observed by a caller
already authorized to write the target; an under-authorized caller receives the
merged [=not-found=] (`404`) instead, per the maximum-privacy rule in
[[[#error-handling]]].

The mechanism extends to a container's **Metadata object**, whose update
operation replaces the whole object -- so two concurrent recipient changes to
a key-epoch roster (see [[[#key-epochs]]]) would otherwise silently clobber
one another:

* The [[[#read-collection-metadata-operation]]] response includes an `ETag`
  header -- a strong validator over the object's `metaVersion` (the same
  quoted form Resources use).
* The [[[#update-or-create-by-id-collection-operation]]] request MAY carry
  `If-Match`. A server implementing the operation MUST evaluate it atomically
  with the write and reject a stale validator with a [=precondition-failed=]
  error (412). The Update and Create responses carry the new `ETag`.
* That request MAY instead carry `If-None-Match: *`, the create-if-absent
  form, which succeeds only when the Collection does not exist. Two clients
  that both read the object, find the Collection absent, and `PUT` a create
  would otherwise let the loser's full replacement overwrite the winner's
  (replacing, for example, its `backend` selection with the loser's).
* `If-Match` is opt-in: an unconditional PUT remains valid (and remains
  last-writer-wins). Recipient-management clients MUST use `If-Match`.

The **Space Metadata object** carries the same validator, on the same terms:
the [[[#read-space-operation]]] response includes an `ETag` header over the
object's monotonic version, the [[[#update-or-create-by-id-space-operation]]]
request MAY carry `If-Match` or `If-None-Match: *`, and a server MUST evaluate
either atomically with the write and reject a failed precondition with
[=precondition-failed=] (412). The Create Space and Update Space responses
carry the new `ETag`. The
create-if-absent form is what lets two clients provisioning the same Space at
once resolve the race at the server: the loser's replacement `PUT` would
otherwise rewrite the winner's `type` array, which a server accepts at
creation only (see [[[#space-metadata-data-model]]]).

<div class="ednote">
Authorization for recipient changes is the plain Collection-update capability
in this version. A dedicated action for recipient management (a policy
operation, distinct from content writes) remains an open question.
</div>

A Collection's `metaVersion` is independent of every Resource's validator:
writing a Resource never advances it, and writing the Metadata object never
advances a Resource's. A server implementing the Collection management
operations MUST maintain it and MUST support both preconditions, on the same
baseline terms as a Resource's validator, and it covers configuration and
annotation writes alike; see [[[#collection-metadata-versioning]]].

A `412` arises only from an explicit `If-Match` / `If-None-Match`
precondition header. It is deliberately distinct from the header-less `409`
conflict kinds -- [=id-conflict=] (a `POST` create whose chosen `id` is already
taken) and [=reserved-id=] -- which describe a conflict with current state where
the client stated no precondition. Conditional requests are the versioned,
header-driven concurrency mechanism; the `409` kinds are not.

### Paginated List Responses {#paginated-list-responses}

The list operations -- [[[#list-spaces-operation]]], [[[#list-all-collections-operation]]],
and [[[#list-collection-operation]]] -- MAY paginate their responses, returning
one page of items at a time using the cursor-based profile defined in Appendix
[[[#pagination]]]. Pagination is OPTIONAL: a server that returns every item in
a single response is conformant, and a client MUST be prepared for either
behavior.

## Service Description {#service-description}

A [=server=] publishes one [=service description=]: a JSON document, readable
without authorization, that names every specification the server implements
and the versions it speaks, and carries the server-wide facts a client needs
before it can perform operations. A client choosing a host at signup, or
deciding which URL layout to speak after a breaking change, reads this document
first. Every other signal in this specification (the Space and Collection
linksets, the Backend description) is reached through a Space-scoped URL,
which is the very thing a layout change alters, so none of them can serve this
purpose.

Serving the service description is REQUIRED at every conformance tier (see
[[[#scope-and-conformance-profiles]]]). It is the one document a client of any
version must be able to read, so its own format is frozen: a future version of
this specification may add members to it but MUST NOT rename, remove, or
change the type of a member defined here.

The service description has no fixed path. A [=server=] MAY be mounted under
any path of an origin, and a client that holds a PWS URL cannot derive the
server root from it, so the document is found by following a link rather than
by convention (see [[[#discovering-the-service-description]]]). This is the
deliberate exception to the "description lives at `meta`" convention of
[[[#reading-this-document]]]: the server level has no container URL of its
own to hang a `meta` segment on.

### Discovering the Service Description {#discovering-the-service-description}

A [=server=] MUST include a `Link` header [[RFC8288]] with the `service` link
relation [[RFC5023]] on every response it sends, whose target is the service
description's URL. This includes error responses (a maximum-privacy "not found"
for an unauthorized request still carries the header), the `308` redirects to a
canonical path, and responses to `HEAD` and `OPTIONS`. The header is static and
server-wide, so it reveals nothing about the existence of any Space, Collection,
or Resource.

```http
Link: <https://example.com/pws/service>; rel="service"
```

The `service` relation is chosen over the `service-desc` and `service-doc`
pair of [[RFC8631]] because this specification defines a single machine-readable
document and no separate human-readable one; [[RFC8631]] section 3.3 recommends
`service` for exactly that case.

Because the primary consumers are browser-hosted clients on other origins, two
CORS requirements apply:

* A [=server=] MUST include `Access-Control-Expose-Headers: Link` on every
  response, so that a cross-origin client can read the `Link` header.
* A [=server=] MUST serve the service description with
  `Access-Control-Allow-Origin: *`, since it is fetched anonymously and
  cross-origin.

The discovery flow is therefore two requests from any PWS URL a client has
come across (a link to a shared Resource, a public Collection, a service
endpoint in a DID document):

1. `HEAD` the URL. The response status does not matter; the client reads the
   `Link` header's `service` target.
2. `GET` that target (see [[[#read-service-description-operation]]]).

A DID document's service endpoint MAY point at the service description itself,
in which case a client skips the first step. The service `type` for that entry
is not yet specified.

The service description is also linked from the [[[#space-linkset]]] and the
[[[#collection-linkset]]] under the same `service` relation, as a convenience
for a client that already holds a Space. Those links are secondary: a client
that can read a linkset is already speaking a layout.

A response with no `service` link comes from a [=server=] that predates this
document, which was introduced in v0.5. A client MUST treat such a server as
speaking v0.4.

### Service Description Data Model {#service-description-data-model}

The service description is a JSON object with the following members:

* `url` (required) - The service description's own canonical URL, so that a
  copy of the document (from a cache, or a DID document) can be refetched.
* `specs` (required) - An object whose keys are specification identifiers and
  whose values are arrays of [=version entries=], one per version of that
  specification the server implements. A specification is listed at every
  version the server speaks, so a server can serve two route tables during a
  transition and a client can pick the newest it understands.
* `instance` (optional) - An object identifying the deployed software, for
  operators who choose to disclose it. See [[[#instance-disclosure]]] for its
  members and the trade-off.

Every URL in the service description MUST be absolute. The document is
self-contained: nothing in it is resolved against the URL it was fetched from.

**Specification identifiers.** The key of a `specs` entry is the persistent
identifier that the specification declares for itself in its own text, which
is not the same as its current document location. A specification that
declares no identifier cannot be advertised. Clients compare keys as opaque
strings, with no URL normalization: a trailing slash or a scheme difference is
a different key. The version is never part of the key; the key names the
specification family, and the [=version entries=] under it carry the versions.

This specification's identifier is `https://w3id.org/pws`.

<div class="ednote">
The identifier above is provisional: it is registered by the rename this
document is pending (see [[[#version-history]]]) and may change with it before
v0.5 is final.
</div>

**Version entries.** A <dfn data-lt="version entry|version entries">version
entry</dfn> is a JSON object describing one version of one specification. Its
members are defined by the specification the key names. Two members are common
to every entry:

* `version` (required) - The specification version, as a bare `major.minor`
  string such as `"0.5"`. The patch level never changes the wire contract and
  is not carried.
* `url` (optional) - The URL of the specification document at exactly this
  version, for a reader who wants the text. A version-frozen location (a dated
  snapshot or a tag) is preferred over a "latest" URL, which stops describing
  the version once the specification moves on.

An entry means the server conforms to that version's required profile. What
else the entry advertises is the owning specification's business: endpoint
URLs, optional sections it also implements, supported algorithms. An optional
section that has an endpoint of its own is advertised by a URL member, and its
absence means the section is not supported; a feature token is used only for
an affordance with no URL of its own. This keeps the service description the
root of the URL graph: with no fixed path for anything at the server level, a
client learns every server-level URL from here. A companion specification
defines the tokens (if any) of its own version entry's `features` array; this
specification does not enumerate them.

**This specification's version entry.** A version entry under
`https://w3id.org/pws` carries, in addition to `version` and `url`:

* `spaces` (optional) - The absolute URL of the Spaces Repository (see
  [[[#spaces-repositories]]]). Absent when the server does not implement it.
* `features` (optional) - An array of feature tokens naming the OPTIONAL
  sections of this specification the server implements, beyond the Minimal
  profile: the vocabulary is open and additive, a client MUST ignore tokens
  it does not recognize, and MUST treat an absent token (or an absent array)
  as "not supported". The tokens are those of the profile table in
  [[[#scope-and-conformance-profiles]]].
* `signatureAlgorithms` (optional) - An array of the signature algorithms the
  server accepts on capability invocations (see
  [[[#performing-authorized-api-calls]]]), named by their JSON Web Algorithms
  [[RFC7518]] identifiers, so `EdDSA` [[RFC8037]] for Ed25519.
* `zcapCryptosuites` (optional) - An array of the Data Integrity cryptosuite
  names the server accepts on capability delegation proofs, such as
  `eddsa-jcs-2022`.

<div class="ednote">
The authorization profile ([[[#pws-authorization-profile-v0-1]]]) does not yet
catalogue which values of these two members it requires; the examples below
use the values the reference implementations exchange today.
</div>

### Instance Disclosure {#instance-disclosure}

The `instance` member identifies the software serving the document. It is
OPTIONAL, its presence is the operator's choice, and a client MUST NOT gate any
behavior on it: what a client acts on is `specs`. Its members:

* `name` (optional) - The software's name, such as the project name of a
  reference implementation.
* `version` (optional) - The software's release version. Disclosing an exact
  patch level is NOT RECOMMENDED; see below.
* `source` (optional) - The absolute URL of the software's source code. A
  deployment under a license that requires offering source to network users
  can satisfy that requirement here.
* `homepage` (optional) - The absolute URL of the software's project page.

The members follow the `software` object of NodeInfo, the most widely
deployed vocabulary for this purpose, with `source` in place of its
`repository`.

Two practices pull in opposite directions here, and this specification takes
no side beyond the default above. Server-hardening guidance (for example, the
OWASP Application Security Verification Standard's requirement that responses
not expose detailed version information of system components) says to
withhold software versions, because mass scanners and exploit kits select
targets by matching an exact version string against a known-vulnerable
release. Federated ecosystems (Fediverse NodeInfo, Matrix)
routinely advertise software and version, because interop debugging across
many independent implementations and an ecosystem-wide vulnerability response
both depend on knowing who runs what.

Two facts specific to this specification narrow the gap. The `specs` member
already fingerprints a server by the versions and features it speaks, so
withholding the software name buys less than it does for a web server behind a
generic page. And a server's conformance behavior identifies its
implementation to any client that cares to probe. What `instance` adds is the
cheap path: a scanner keying on a version string. Omitting `version`, or
carrying only its major component, removes that path while keeping `name`,
`source`, and `homepage` for debugging and license compliance.

### Read Service Description operation {#read-service-description-operation}

`GET` on the URL named by the `service` link returns the service description as
`application/json`. The operation requires no authorization. The response
SHOULD be cacheable (`Cache-Control` and an `ETag`), so that a client
re-checking a host pays a revalidation rather than a fetch.

Example discovery from a Resource URL the client holds:

```http
HEAD /pws/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
```

Response (the Resource is private, so the unauthorized request is "not found";
the `Link` header is present regardless):

```http
HTTP/1.1 404 Not Found
Link: <https://example.com/pws/service>; rel="service"
Access-Control-Expose-Headers: Link
```

Example service description request and response:

```http
GET /pws/service HTTP/1.1
Host: example.com
Accept: application/json
```

Response:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Link
Cache-Control: public, max-age=3600
ETag: "a1b2c3"
Link: <https://example.com/pws/service>; rel="service"

{
  "url": "https://example.com/pws/service",
  "specs": {
    "https://w3id.org/pws": [
      {
        "version": "0.5",
        "url": "https://w3c-ccg.github.io/wallet-attached-storage-spec/v0.5/",
        "spaces": "https://example.com/pws/spaces/",
        "features": ["listing", "collection-management", "space-management",
                     "linksets", "policy", "backends", "quotas"],
        "signatureAlgorithms": ["EdDSA"],
        "zcapCryptosuites": ["eddsa-jcs-2022"]
      },
      {
        "version": "0.4",
        "url": "https://w3c-ccg.github.io/wallet-attached-storage-spec/v0.4/",
        "spaces": "https://example.com/pws/spaces/",
        "features": ["listing", "collection-management", "space-management",
                     "linksets", "policy", "backends", "quotas"]
      }
    ],
    "https://w3id.org/encrypted-collections": [
      { "version": "0.2" }
    ]
  },
  "instance": {
    "name": "was-teaching-server",
    "source": "https://github.com/interop-alliance/was-teaching-server"
  }
}
```

The server above serves the v0.4 and v0.5 route tables together during the
transition described in [[[#version-history]]].

### Selecting a Version {#selecting-a-version}

A client reads the service description before its first structural request to
a [=server=] and applies these rules:

* A key it does not recognize is ignored.
* A [=version entry=] without a `version` member is ignored.
* For each key, the client uses the highest version it understands.
* If no entry under `https://w3id.org/pws` names a version the client
  understands, the server is incompatible and the client MUST stop rather than
  guess a layout. The same applies to a document that is not valid JSON, or
  that lacks `url` or `specs`.
* A response with no `service` link means v0.4 (see
  [[[#discovering-the-service-description]]]).

How often a client re-reads the document is a client policy. A client that
gates a security-relevant behavior on an advertised affordance (for example,
publishing only to a host that enforces a verification rule) SHOULD re-check
before relying on it again, since a host can stop advertising an affordance
it once did.

### Protocol Evolution {#protocol-evolution}

This specification changes in two ways, and the service description
distinguishes them.

Additive change (a new optional endpoint, a new feature token, a new error
kind, a new query profile, a new reserved segment) is advertised through the
registries in the appendices and through `features`. A client that does not
know the addition keeps working, because unknown tokens are ignored and absent
means not supported.

Incompatible change (an existing URL changing meaning, a member changing type,
a required behavior being removed) takes a new specification version, and the
new version is advertised as a new [=version entry=]. A server MAY list the old
and the new version together during a transition, and a client picks the
newest it understands.

Versions never appear in URL paths. A PWS URL is a capability target and a
long-lived identity: it is what a delegation chain is rooted on and what a
client stores to find a Resource again. A version segment in the path would
bind every delegation chain to a protocol version and would turn every version
bump into a re-delegation of every capability. The version lives in the service
description, and a layout change keeps the old URLs meaningful for as long as
the server lists the old version.

## Spaces Repositories {#spaces-repositories}

A Spaces Repository is a set of API endpoints that supports the creation and
management of multiple spaces on a given [=server=].
This `/spaces/` set of API endpoints is optional. If a server does not support
this feature (for example, if it is a single-tenant server with an existing
hardcoded Space), then it can implement only the `/space/{space_id}/` endpoints
and get most of the functionality of this specification.

<div class="ednote">
The Spaces Repository endpoints are a candidate for extraction into a
standalone companion specification (multi-tenant space provisioning) in a
future version of this document.
</div>

### Create Space operation {#create-space-operation}

To create a Space:

* Perform an authorized Create Space operation that includes a Proof of
  (cryptographic material) Possession via a mechanism such as HTTP Signatures.

* If no `id` is provided, it will be generated by the storage server.

* A `controller` DID MUST be provided in the request body, and the request MUST
  be authorized by that DID.
  * If no `controller` is provided, the server MUST return an HTTP 400 error
    response
  * The capability invocation MUST be *authorized by* the body's `controller`:
    either signed directly by the `controller` DID (the common case), or signed
    by another DID presenting a valid, unexpired delegation chain rooted in the
    `controller` (see [[[#delegation]]]). This is how the root of trust is
    initially set up (see [[[#space-controller-and-the-root-of-trust]]] for
    more details).
  * The delegated form supports creating a Space *on behalf of* its eventual
    controller: for example, a provisioning service that handles payment or
    onboarding can be delegated a `POST` capability for `/spaces/` by the user,
    and create a Space that the user's DID controls from the start. What is
    never acceptable is installing a `controller` that has authorized nothing
    at all: that would let anyone bind arbitrary DIDs to Spaces without their
    consent -- squatting meaningful ids "in their name", or burning
    per-controller onboarding allowances such as "one free Space per
    controller".

* (Optional, out of scope) A given storage provider MAY impose additional
  requirements to create a Space for a given controller, such as:
  - a Verifiable Credential representing a pre-arranged onboarding coupon
  - a proof of payment
  - a proof of membership in an organization

#### (HTTP API) POST `/spaces/` {#http-api-post-spaces}

To create a space via HTTP API using the Spaces Repository POST API:

```http
POST /spaces/ HTTP/1.1
Host: example.com
Accept: application/json
Content-type: application/json
Authorization: ...

{
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

Example success response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

Note that in the example above:

* the `id` was not specified in the body of the request, and so was generated by
  the server and returned in the response

#### Create Space Errors {#create-space-errors}

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=invalid-request-body=] (400) -- the required `controller` property is
  missing from the request body, or is a DID of a method the server does not
  accept as a Space controller (see
  [[[#space-controller-did-method-registry]]]: `did:key` is mandatory to
  support, other methods are optional, and a server may admit them only on
  update).
* [=missing-authorization=] (401) -- the request lacks a valid proof of
  possession of the `controller` DID.
* [=controller-mismatch=] (400) -- the invocation is not currently
  authorized by the `controller` DID in the request body: it is neither signed
  by that DID nor accompanied by a valid, unexpired delegation chain rooted in
  it. A chain rooted in a different DID, an expired delegation, and a chain
  whose proof fails verification all fall under this one type. Servers are not
  required to distinguish these (delegation-chain verifiers often report
  failure opaquely), but SHOULD differentiate the cause in the non-normative
  `detail` string where they can. In a delegated provisioning flow, the
  cause determines who must act (the user re-delegates an expired capability;
  the service corrects a request whose chain is rooted in the wrong DID).
* [=invalid-id=] (400) -- the supplied Space `id` is not URL-safe (see
  [[[#identifiers]]]).
* [=id-conflict=] (409) -- a Space with the supplied `id` already exists.

Onboarding requirements, if any, are provider-specific and out of scope for this
specification (see the optional onboarding material above). Their error `type`,
`title`, and status code are likewise provider-defined.

A `POST /spaces/` that supplies an `id` already in use returns the
[=id-conflict=] error. To create or replace a Space at a client-chosen
`id` without conflict, use the idempotent
[[[#update-or-create-by-id-space-operation]]] instead.

The existence check behind [=id-conflict=] is security-critical here, not
merely a conformance detail. Create Space is the one operation whose capability
invocation is verified against the `controller` supplied *in the request
body* -- the Space does not exist yet, so there is no stored controller to
verify against. A server that instead treats `POST /spaces/` as
create-or-replace turns this operation into a takeover: any caller able to
construct a valid invocation for their *own* `controller` could overwrite an
existing Space's Metadata object (controller included) simply by POSTing its
`id`. Servers MUST check for an existing Space with the supplied `id`, and
reject with [=id-conflict=], before writing anything.

<div class="note">
The [=id-conflict=] response on this operation necessarily reveals whether a
Space id is taken: any caller permitted to attempt creation can distinguish
`409` from `201` -- just as they could by observing whether creation succeeds.
This disclosure is inherent to client-chosen ids on a create endpoint, and is
not a violation of the maximum-privacy principle of [[[#error-handling]]],
which governs errors about *existing* targets the caller is not authorized
for. Unguessable (for example, UUID) Space ids keep the signal worthless to an
attacker, and providers that gate Space creation behind onboarding requirements
also bound who can observe it.
</div>

<div class="note">
Differentiated `detail` strings on [=controller-mismatch=] are likewise
privacy-safe: Create Space has no existing target to protect, and everything
its verification examines (the body's `controller`, the capability chain,
its signatures) is supplied by the caller, so failure granularity reveals
nothing the caller does not already hold. This reasoning does *not* extend to
failure causes that depend on server-side state (for example, capability
revocation status or per-controller onboarding allowances); whether and how to
disclose those remains provider-defined.
</div>

### List Spaces Operation {#list-spaces-operation}

* Requires appropriate authorization (root zcap invoked by the controller of one
  or more spaces, or a zcap granting permission to read a one or more spaces)

* Lists only the spaces the requester is authorized to see

* MAY be paginated (see [[[#pagination]]])

#### (HTTP API) GET `/spaces/`

Example request:

```http
GET /spaces/ HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example success response (requester has read access to at least one space):

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "url": "/spaces/",
  "totalItems": 1,
  "items": [
    {
      "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/"
    }
  ]
}
```

Example success response (requester does NOT have access to any spaces):

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "url": "/spaces/",
  "totalItems": 0,
  "items": []
}
```

A request that carries no authorization (or authorization for no spaces) is not
an error: like any list operation it returns the `200 OK` empty-list response
shown above, revealing nothing about which spaces exist (see
[[[#error-handling]]]).

## Spaces {#spaces}

A space is a namespace for collections and a unit of general configuration,
a volume of storage that contains one or more collections.
Conceptually, is maps to a disk partition (for file systems), or a database
(for relational databases).

### Space Metadata Data Model {#space-metadata-data-model}

A Space's description is its **Metadata object**, addressable at the reserved
`meta` path segment under the Space URL (one of the
[[[#space-level-reserved-endpoints]]]). It is what
[[[#read-space-operation]]] returns and what
[[[#update-or-create-by-id-space-operation]]] writes; the Space URL itself is
the container of the Space's Collections (see
[[[#list-all-collections-operation]]]).

Because the `meta` segment occupies the `{collection_id}` position, `meta` is a
reserved Collection id: creating a Collection with the id `meta` is a
[=reserved-id=] conflict (see [[[#reserved-path-segment-registry]]]).

`PUT` is not defined at the Space URL itself; the replacement write is a `PUT`
of this object. A server MUST answer a `PUT` at the Space URL with
`405 Method Not Allowed`.

`Space` properties:

* `id` - A unique identifier for a space at a given [=server=]. Created by the
  server if not provided. Note: the `{space_id}` template parameter used in URL
  templates in this spec MUST match that space's `id` property. See
  [[[#identifiers]]] for additional constraints.
* `type` - An array of strings, MUST include the type `Space`. The array SHOULD
  be lexically sorted so that the object has a canonical, stable serialization
  (useful for hashing or signing).
* `name` (optional) - An arbitrary human-readable name for the space. Does not
  have to be unique.
* `controller` - A cryptographic identifier (a [=did=])
  of the entity that is authorized to perform operations on the space (or to
  delegate authorization to other entities). See
  [[[#space-controller-did-method-registry]]] for the DID methods a
  controller may use.

Space properties automatically added by the server:

* `createdBy` (optional) - The [=did=] of the party whose capability invocation
  created the space. It is recorded on the first write and preserved unchanged
  by later writes, so it names the creator rather than the last writer. It is
  read-only: a server MUST ignore a `createdBy` supplied in a request body.
  `createdBy` is distinct from `controller` -- under delegated provisioning
  (see [[[#space-controller-and-the-root-of-trust]]]) the party that creates a
  space need not be the party that controls it. Recording it is OPTIONAL, so a
  client MUST treat an absent `createdBy` as "not recorded" rather than as an
  assertion that the space has no creator.
* `url` - A relative URL to the space itself, in canonical (trailing-slash)
  form. Added by the server, used in the Space Metadata object as well as
  the [[[#list-spaces-operation]]] result. The Metadata object's own URL is
  this value followed by `meta`.
* `linkset` - A relative URL to a resource which contains
  a set of links to auxiliary resources (such as to access control policy
  documents). See section [[[#space-linkset]]].
  Note that this is one of the [[[#space-level-reserved-endpoints]]].

A server that supports conditional writes (see [[[#conditional-requests]]])
also keeps a server-managed version validator for each Space Metadata object:
an opaque strong validator that changes on every write of the object and
is never reused after the Space is deleted. It is not a member of the Space
Metadata object; it is surfaced only as the `ETag` header of the Read Space
response and of the Create Space and Update Space responses, and consumed only
through the `If-Match` and `If-None-Match: *` preconditions of the Update (or
Create by Id) Space operation.

### Read Space operation {#read-space-operation}

* Returns the [[[#space-metadata-data-model]]] for the specified space `id`
* Requires appropriate authorization (root zcap invoked by the space's controller,
  or a zcap granting permission to read a particular space)
* On a server supporting conditional writes, the response includes an `ETag`
  header over the Space Metadata object's version, for use with `If-Match` on a
  subsequent update (see [[[#conditional-requests]]]); a request carrying an
  `If-None-Match` that covers it is answered `304 Not Modified` with the
  `ETag` and no body (see [[[#caching]]])

The format of the response is determined based on content negotiation;
`application/json` is the REQUIRED baseline (see
[[[#content-types-and-representations]]]).

#### (HTTP API) GET `/space/{space_id}/meta`

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/meta HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json
ETag: "z3fkq.2"

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/",
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW",
  "createdBy": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW",
  "linkset": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset"
}
```

#### Read Space Errors

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Space does not exist, or the caller has missing
  or insufficient authorization. A server MUST return the same error response in
  both cases, per [[[#error-handling]]].

### Update (or Create by Id) Space operation {#update-or-create-by-id-space-operation}

A `PUT` of a Space's Metadata object is a full replacement of that object, and
creates the Space when none exists under that `id`. The client specifies the
`id` of the Space in the request URL.

* Requires appropriate authorization. Updating an existing Space's Metadata
  object MUST be authorized by a root capability invoked directly by the
  Space's [=controller=]; a delegated capability never authorizes this
  write, whatever its `allowedAction` (see [[[#the-container-rule]]]).
  Creating a Space at this URL follows the create-by-id rule below instead.
* Allows the client to update the following fields:
  - `name`
  - `controller`

The `controller` property in the request body is the *proposed* value, not the
root of trust for the request. When the Space already exists, the capability
invocation MUST be verified against the Space's currently *stored*
`controller`: only the current controller (or its delegate) may update the
Space, including transferring it by writing a new `controller`. The body's
`controller` participates in verification only when the `PUT` creates the
Space -- there is no stored controller yet, so, as with
[[[#create-space-operation]]], the invocation MUST be authorized by the body's
`controller`: signed by it directly, or presented with a valid, unexpired
delegation chain rooted in it (violations are [=controller-mismatch=], as for
POST). A server that verifies an *update* against the
body's `controller` reopens the takeover described under
[[[#create-space-errors]]]: any caller could seize an existing Space by
PUTting its `id` with themselves as the `controller`.

A server that records `createdBy` (see [[[#space-metadata-data-model]]]) sets
it only on the `PUT` that creates the Space, to the [=did=] of the party that
invoked the capability -- which is not necessarily the body's `controller`. On
a `PUT` that updates an existing Space, the stored `createdBy` is preserved, so
transferring the Space by writing a new `controller` does not rewrite its
creator.

A new `controller` written by an update is subject to the same method
acceptance as one set at creation (see
[[[#space-controller-did-method-registry]]]), plus the promotion rule of
[[[#setting-a-controller-to-optional-did-method]]]: a proposed controller of
an optional method MUST resolve and verify before it is stored, and a server
MAY admit such controllers only through this operation.

The request MAY carry a precondition (see [[[#conditional-requests]]]):
`If-Match: "<etag>"` performs the update only if the Space Metadata object's
current `ETag` matches it, and `If-None-Match: *` performs the write only if
no Space exists under that id yet. A server that supports
conditional writes MUST evaluate the precondition atomically with the write,
after authorization, and answer a failed one with [=precondition-failed=]
(412). Both are opt-in: an unconditional `PUT` remains valid and remains
last-writer-wins. A client that creates a Space it has just read as absent
SHOULD send `If-None-Match: *` and, on a 412, re-read the Space instead of
treating the response as a failure. The success response carries the Metadata
object's new `ETag`.

#### (HTTP API) PUT `/space/{space_id}/meta`

Note that this is a _full_ update (partial updates via http `PATCH` verb might
be supported later). However, some fields cannot be updated (like `id`) and so
may be omitted from the request payload.

Note that this operation is idempotent.

* When creating a space via PUT, a `controller` property is required in the PUT
  request body.

Example request (creating a new space via PUT):

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/meta HTTP/1.1
Host: example.com
Accept: application/json
Content-type: application/json
Authorization: ...

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

Example success response. The `Location` header names the Space that was
created, not the Metadata object that was written:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/
ETag: "z3fkq.1"
```

Example request (updating the `name` property of a space). Note that
server-managed properties such as `url` and `linkset` are not client-writable
and are omitted from the request body:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/meta HTTP/1.1
Host: example.com
Content-type: application/json
Accept: application/json
Authorization: ...

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Newly renamed space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

Example success response:

```http
HTTP/1.1 204 No Content
ETag: "z3fkq.2"
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Space does not exist, or the caller has missing
  or insufficient authorization; per [[[#error-handling]]] the two are
  indistinguishable.
* [=invalid-request-body=] (400) -- the client is attempting to change an
  immutable field, such as setting a body `id` that does not match the
  `{space_id}` in the request URL; or the proposed `controller` is a DID of a
  method the server does not accept, or one that does not resolve and verify
  (see [[[#setting-a-controller-to-optional-did-method]]]).
* [=precondition-failed=] (412) -- the request carried an `If-Match` that
  does not match the Metadata object's current `ETag`, or an
  `If-None-Match: *` against a Space that already exists (see
  [[[#conditional-requests]]]).

### Delete Space operation {#delete-space-operation}

* Requires appropriate authorization: a root capability invoked directly by
  the space's [=controller=], or a delegated capability whose
  `invocationTarget` is exactly this Space's URL and whose `allowedAction`
  is exactly `['DELETE']` -- the Space-delete exception to the container
  rule, see [[[#the-container-rule]]]
* Deletes the space and all the data (collections and resources) contained
  in it
* This operation is idempotent

#### (HTTP API) DELETE `/space/{space_id}/`

Example request (no request body):

```http
DELETE /space/81246131-69a4-45ab-9bff-9c946b59cf2e/ HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the caller has missing or insufficient authorization.
  Because DELETE is idempotent, an authorized request for an already-absent
  Space returns `204`; an under-authorized request returns `404` instead, per
  [[[#error-handling]]], so that an unauthorized caller cannot probe for
  existence.
* [=invalid-id=] (400) -- the supplied Space `id` is not URL-safe.

### List All Collections operation {#list-all-collections-operation}

* Returns the list of all Collections in a Space (that the requester has
  permission to access)
* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request must be
    signed by the space's [=controller=], or invoke a delegated capability that
    allows the [=GET=] action
* Since Collection's `name` property is optional, default it to be the same
  value as `id` when `name` is missing. (The name is intended to drive UIs, so
  defaulting to `id` simplifies consuming client logic.)
* Each listed item carries the Collection's `id`, `url`, and `name` -- a
  projection of the Collection's Metadata object (see
  [[[#collection-metadata-data-model]]]). A server MAY additionally surface a
  `public` member on each item: a boolean indicating
  whether a `PublicCanRead` access control policy (see
  [[[#access-control-policies]]]) is attached to that Collection. It is surfaced
  inline so that a listing consumer need not issue one policy probe per item to
  learn whether the Collection is publicly readable. A server that surfaces
  `public` MUST include it on *every* item in the listing, expressing `false`
  explicitly rather than omitting it, so that a client can distinguish "not
  public" from "the server does not surface the flag". Consistent with the
  fail-closed rule of the policy evaluation contract, a policy whose `type` the
  server does not recognize MUST be reported as `"public": false`. When `public` is absent
  from a listing, a client falls back to probing each Collection's policy (see
  [[[#access-control-policies]]]).
* MAY be paginated (see [[[#pagination]]])

#### (HTTP API) GET `/space/{space_id}/`

Example request (a container URL, so it carries the trailing slash):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/ HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/",
  "totalItems": 2,
  "items": [
    {
      "id": "example",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/example/",
      "name": "Example Collection",
      "public": false
    },
    {
      "id": "73WakrfVbNJBaAmhQtEeDv",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/",
      "name": "73WakrfVbNJBaAmhQtEeDv",
      "public": true
    }
  ]
}
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Space does not exist, or the caller has missing or
  insufficient authorization; per [[[#error-handling]]] a Space the caller is
  not authorized to read is indistinguishable from one that does not exist.

## Collections {#collections}

A collection is a namespace for Resources, and a unit of configuration, within
a space.

In other storage systems, the concept of collections has many different names.
For example, *Directory, Folder, RDBMS Table, Document Collection, Graph, WebAPI
FileList, Bucket, LDP Basic Container, EDV Vault*, and so on.

Collections do not contain other collections (this specification adopts a
flat collection structure).

<div class="note">
Rationale: Nested collections (such as those used by Solid/LDP/LWS) encode query
relationships into URL hierarchy, which creates significant complexity
(recursive permissions, recursive deletes, path traversal, depth-limited listing)
and maps poorly to flat backends like RDBMS tables or S3 buckets. The use cases
that motivate nesting (e.g., "all comments on a post") are better served by the
`query` endpoint with field-based filtering.
</div>

### Collection Metadata Data Model {#collection-metadata-data-model}

A Collection's description is its **Metadata object**, addressable at the
reserved `meta` path segment under the Collection URL (one of the
[[[#collection-level-reserved-endpoints]]]). It holds everything recorded about
the Collection: what it is called, how its Resources are stored and whether
they are client-side encrypted, who created it, and whatever annotations the
user keeps on it. The Collection URL itself is the container of the
Collection's Resources (see [[[#list-collection-operation]]]).

Because the `meta` segment occupies the `{resource_id}` position, `meta` is a
reserved Resource id (see [[[#reserved-path-segment-registry]]]). A `GET` or
`PUT` at this path is always a Collection metadata operation, so no Resource
is ever addressed at the id `meta`. Methods this specification does not define
at this path (`DELETE` in particular) are not Resource operations either; see
[[[#methods-at-reserved-endpoints]]] for how a server answers them. The same
`meta` segment also roots the Collection's governing history log, a separate
sub-resource at `meta/log` with operations and a validator of its own (see
[[[#collection-governing-history-log]]]).

`PUT` is not defined at the Collection URL itself; the replacement write is a
`PUT` of this object. A server MUST answer a `PUT` at the Collection URL with
`405 Method Not Allowed`.

Reading and writing this object are part of the OPTIONAL Collection management
group (see [[[#api-summary]]]): on a server where Collections are
pre-configured or implicit, they are not implemented, and such a server SHOULD
answer requests to this path with an [=unsupported-operation=] (501) error.

Writable properties:

* `id` - A unique collection identifier (within a given space). Created by the
  server if not provided. Note: the `{collection_id}` template parameter used in
  URL templates in this spec MUST match that collection's `id` property.
  See [[[#identifiers]]] for additional constraints.
* `type` - An array of strings, MUST include the type `Collection`. As with a
  Space's `type`, the array SHOULD be lexically sorted for a canonical, stable
  serialization.
* `name` (optional) - An arbitrary human-readable name for the collection. Does not
  have to be unique.
* `generator` (optional) - The [=did=] of the application this collection was
  provisioned for, named for the ActivityStreams 2.0 `generator` term (the
  application that generated an object). It is the client-asserted identity
  axis of [[[#writer-attribution]]]: client-supplied, persisted, and writable
  by the Space controller on create and on update -- updatable so that a
  controller can backfill the property onto collections that predate it --
  in contrast to the server-observed, read-only `createdBy`. The contrast is
  the reason the property exists: under delegated provisioning the party
  whose capability invocation creates the collection is the user's agent (a
  wallet), so `createdBy` records the user's [=did=], never the
  application's, and only the controller is in a position to name the
  application it provisioned the collection for. `generator` is an assertion
  by the controller, not a server-verified fact: a server MUST NOT verify,
  compute, or default it, and MUST NOT use it as an input to authorization.
  A server MAY separately record that the `generator` [=did=] has been
  observed invoking a delegated capability on the collection -- corroboration
  on the same evidentiary footing as `createdBy` (server-observed use, not
  verified identity). Either way, any reader other than the controller (a
  delegated consumer of a shared collection, a reader of a world-readable
  one) MUST treat `generator` as exactly a controller assertion. A present
  but empty or non-[=did=]-shaped value is an [=invalid-request-body=]
  error.
* `generatorOrigin` (optional) - The Web origin (its ASCII serialization,
  e.g. `https://app.example.com`) the `generator` [=did=] was bound to when
  the collection was provisioned. A provisioning exchange in which the
  user's agent authenticates the requesting application by origin (for
  example a browser-attested credential-handler exchange) establishes an
  origin binding the storage server is never a party to; stamping the origin
  beside `generator` preserves that fact and gives a reader a human-readable
  attribution label without further lookups. Same footing as `generator` in
  every other respect: controller-asserted, writable by the Space controller
  on create and update, never verified or defaulted by the server. A present
  value that is not the ASCII serialization of a Web origin (a URL carrying
  a path, query, or fragment; the empty string) is an
  [=invalid-request-body=] error. `generatorOrigin` SHOULD NOT be present
  without `generator`.
* `backend` (optional) - An object describing the storage backend selected for
  this collection. If not specified when the Collection is created, defaults to
  the value `{ "id": "default" }`. An update that omits the member keeps the
  stored selection (see [[[#update-or-create-by-id-collection-operation]]]).
  The backend object's `id` property MUST be from the list of [[[#space-backends-available]]]
  for the given space. If an unavailable (unsupported) backend is specified,
  the server MUST throw an error.
  See section [[[#backends]]] for more details.
* `encryption` (optional) - A non-secret descriptor declaring that this collection's
  Resources are client-side encrypted, and naming the scheme. The value is an
  object with a required string `scheme` and an optional positive-integer
  `version` (e.g. `{ "scheme": "edv", "version": 1 }` for the EDV-over-PWS
  scheme); an absent `version` means `1`, and an absent descriptor means the
  collection is a <dfn id="plaintext-collection"
  data-lt="plaintext|plaintext collections">plaintext collection</dfn> -- its
  Resources are stored as the client sent them, with no client-side encryption
  envelope. The `version` names the version of the
  scheme's wire format as registered in the [[[#encryption-scheme-registry]]],
  so a scheme's envelope profile can evolve without ambiguity about which shape
  a given collection stores; it is an opaque registry key with a total order,
  not a semantic version. The server
  MUST NOT interpret the descriptor beyond validating its shape -- it never holds key
  material and stores the descriptor opaquely. Its purpose is discovery: any
  authorized reader (including a delegated consumer that did not create the
  collection) learns from the Collection's Metadata object that the collection
  is encrypted, and decrypts with its own keys. The descriptor is **set-once,
  version-monotonic**: a server
  MUST allow declaring it on a collection that lacks one (e.g. migrating a
  pre-existing collection), provided no `plaintext` member is present (the
  two are mutually exclusive; see `plaintext` below), and MUST accept a
  re-declaration of the standing
  values (including an explicit `"version": 1` on a descriptor that had omitted
  it) as an idempotent no-op, but MUST reject (with an `encryption-immutable` error)
  any attempt to change the `scheme`, decrease or remove the `version`, or
  clear the descriptor, on an existing
  collection, since that would corrupt the already-stored encrypted Resources.
  On a collection whose stored descriptor
  carries an explicit `"version": 1`, an update whose descriptor omits
  `version` counts as an attempt to remove it and is rejected under this rule,
  though because an absent `version` means `1`, a server MAY instead accept
  such an update as an idempotent re-declaration of the standing value. Under
  either reading, the standing version survives unchanged.
  Raising the `version` is permitted (subject to the recognition rule in
  [[[#encryption-scheme-registry]]]): subsequent writes are validated against
  the new profile, while already-stored Resources are unaffected.
  See section [[[#backends]]] (client-side encryption note) for the rationale.
  A server that recognizes the declared `scheme` and `version` enforces them
  structurally on write -- rejecting any non-envelope body so plaintext can never
  be stored in an encrypted Collection; see [[[#encryption-scheme-registry]]].
  On the `edv` scheme, the descriptor MAY additionally carry the key-epoch
  public references, `currentEpoch` and `epochs` (see [[[#key-epochs]]]),
  which let several readers hold per-reader keys and let reader removal rotate
  the collection key, and the blinding-key reference `hmac` (see
  [[[#blinding-key-member]]]), which distributes the key under which a client
  blinds the attributes it indexes for the `blinded-index` query profile.
  These members contain only public key identifiers and wrapped-key
  ciphertext. As with the rest of the descriptor, the server stores them
  opaquely. A server that recognizes the scheme validates their shape,
  enforces a small set of server-side invariants on the epoch members
  (`epochs` is append-only, `currentEpoch` never moves backwards), holds the
  `hmac` member permanent once present, and interprets nothing else.
  Client-side profiles built on epochs (such as [[PWS-EC]]) require the epoch
  members from the descriptor's creation onward; the server-side optionality
  here exists because key management is out of this specification's scope, and
  is not a license for such a profile's clients to operate an epoch-less
  descriptor.
  On a server whose PWS-EC version entry advertises `governed-history-logs`
  (see [[[#service-description-data-model]]]), the descriptor MAY instead
  be governed by the Collection's history log: the server then derives it from
  the log's head entry, stamps a `history` member on the served value, and
  refuses a direct write of it with an [=encryption-history-log-governed=]
  error; see [[[#collection-governing-history-log]]].

* `plaintext` (optional) - An object declaring the server-side processing the
  server may apply to this collection's Resources. It is the counterpart of
  `encryption`: the two members describe the two ways a server may treat
  what it stores, and a Collection's Metadata object MUST NOT carry both. The
  exclusion is by presence. An empty `plaintext` object still excludes
  `encryption`, and a server MUST reject a create or update whose resulting
  object would carry both with an [=invalid-request-body=] error. A
  Collection carrying neither member is a [=plaintext collection=] with no
  server-side processing declared. Unlike the set-once `encryption`,
  `plaintext` is updatable for the collection's life: a supplied object
  replaces the stored one, an update that omits the member leaves the stored
  one untouched, and `{}` is the empty state (present, declaring nothing).
  A `plaintext` value that is not an object is an [=invalid-request-body=]
  error. Its one member in this version is `indexes`:
  * `indexes` (optional) - An array naming the attributes the server extracts
    from stored Resources and indexes for query profiles that read plaintext
    (see [[[#query-profile-registry]]]). This is
    <dfn id="plaintext-indexes" data-lt="plaintext index|plaintext indexes">server-side plaintext indexing</dfn>:
    the server reads the stored values. It is distinct from the client-blinded indexes of an
    encrypted collection, which the client computes under the descriptor's
    `hmac` key (see [[[#blinding-key-member]]]), carries in each envelope's
    `indexed` member, and queries through the `blinded-index` profile
    (see [[[#query-profile-blinded-index]]]). Each entry is an object
    `{ "name": ..., "source": ..., "unique": ... }`, or a bare string as
    shorthand for `{ "name": "<string>", "source": "content" }`:
    * `name` - The attribute name. Required, a non-empty string, and unique
      across the array regardless of `source`, since queries refer to an
      attribute by name alone.
    * `source` (optional) - Where the attribute is read from: `content`
      (the default) reads the top-level member of that name from a JSON
      Resource's stored content; `custom` reads it from the `custom` object
      of the Resource's Metadata (see [[[#resource-metadata-data-model]]]),
      which is how a Blob Resource becomes queryable. Any other value is an
      [=invalid-request-body=] error.
    * `unique` (optional) - A boolean, `false` when absent. When `true`, the
      attribute's `(name, value)` pair is a uniqueness claim within the
      collection: a server MUST reject a Resource write whose extracted
      value is already held by a *different* Resource in the same
      collection with an [=id-conflict=] error, under the same
      authorization-first and atomicity rules as unique blinded attributes
      (see [[[#query-profile-blinded-index]]], *Unique blinded
      attributes*). A Resource re-asserting its own value never
      self-conflicts. A server MUST also reject an update that adds
      `unique: true` for an attribute whose already-stored Resources
      violate it, with an [=id-conflict=] error, leaving the stored
      declaration unchanged.

    Declaring a `content`-sourced entry is the collection's opt-in to the
    server parsing its JSON Resource content on write. A malformed
    `indexes` array (not an array, an entry that is neither a non-empty
    string nor an object with a non-empty string `name`, a non-boolean
    `unique`, or a duplicate `name`) is an [=invalid-request-body=] error.
    Adding or removing an entry is a declaration change only: a server
    MUST NOT require the already-stored Resources to be rewritten for the
    change to take effect.

* `custom` (optional) - A JSON object holding the user's annotations, with the
  same shape as the Resource-level `custom` (see
  [[[#resource-metadata-data-model]]]): an optional human-readable `name` and
  an optional `tags` object of string-valued annotations. A Collection with no
  annotations set MAY omit `custom` or report it as `{}`. A Collection
  therefore has two name slots. The top-level `name` above is server-visible
  plaintext and is what listings report; `custom.name` is supplementary. On an
  encrypted Collection the two trade places: a client SHOULD leave the
  plaintext `name` unpopulated, so that a listing defaulting a missing `name`
  to the `id` reveals nothing (see [[[#list-all-collections-operation]]]), and
  `custom.name` inside the encrypted envelope is then the only name the
  Collection has.
* `epoch` (optional) - The key-epoch `id` this object's `custom` envelope was
  encrypted under (see [[[#key-epochs]]]), declared by the writer and stored
  opaquely; the server never computes or verifies it. Its omission rule
  deliberately inverts the Resource-level one: an omitted `epoch` on a write of
  this object clears the stored value, where at Resource level it is preserved.
  The Resource-level `epoch` describes the content write, which a write of this
  object does not touch, so preserving it there is correct; here the only thing
  `epoch` can describe is the `custom` envelope itself, which every write
  replaces wholesale, so a preserved value would misdescribe the new envelope.
  A client writing to an encrypted Collection therefore re-declares `epoch`
  each time.

Properties automatically added by the server, and read-only: a server MUST
ignore any of them supplied in a request body.

* `createdAt` (optional) - The [[RFC3339]] `date-time` at which the Collection
  was created.
* `updatedAt` (optional) - The [[RFC3339]] `date-time` at which this Metadata
  object was last modified.
* `createdBy` (optional) - The [=did=] of the party whose capability invocation
  created the collection, on the same terms as a Space's `createdBy` (see
  [[[#space-metadata-data-model]]]): recorded on the first write, preserved by
  later writes, and OPTIONAL.
* `url` - A relative URL to the collection itself, in canonical
  (trailing-slash) form. Added by the server, used in this object as well as
  the List Collections operations result. This object's own URL is that value
  followed by `meta`.
* `linkset` - A relative URL to a resource which contains
  a set of links to auxiliary resources (such as to access control policy
  documents). See section [[[#collection-linkset]]].
  Note that this is one of the [[[#collection-level-reserved-endpoints]]].

There is no `writerId` member. A Collection's Metadata object is configuration
maintained by the Space [=controller=], not replicated per-revision content, so
the attribution machinery of [[[#writer-attribution]]] does not apply at this
level.

On a Collection that declares an `encryption` descriptor (see
[[[#encryption-scheme-registry]]]), `custom` is stored **encrypted**, exactly
as at Resource level: its value is an envelope of the Collection's declared
scheme, validated structurally on write (rejecting a plaintext `custom` with an
[=encryption-scheme-mismatch=]) and never decrypted by the server, while every
other member of this object stays plaintext. This is what gives an encrypted
Collection a client-encrypted display name and tags, and a discoverable,
conditionally-writable home for profile-level configuration such as the
[[PWS-EC]] blinded-index schema. On an encrypted Collection a present `custom`
MUST be either a conforming envelope or an empty object; an empty object clears
the stored annotations. Omitting `custom` clears them too, exactly as on a
plaintext Collection, because a `PUT` of this object is a full replacement.
That is the same omission-clears rule `epoch` follows.

Example Metadata object of a plaintext Collection (JSON representation):

```json
{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/",
  "type": ["Collection"],
  "name": "Verifiable Credentials Collection",
  "createdAt": "2026-06-10T09:12:00Z",
  "updatedAt": "2026-06-12T13:25:00Z",
  "createdBy": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW",
  "generator": "did:key:z6MkfriqYRX3JBqzsbVbKuBUxDR2nsjLTu6AbrxJZAmFmXWb",
  "generatorOrigin": "https://app.example.com",
  "linkset": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/linkset",
  "custom": {
    "tags": { "project": "demo" }
  }
}
```

#### Versioning {#collection-metadata-versioning}

The Metadata object carries a monotonic version, `metaVersion`, surfaced as a
strong `ETag` validator on read. It is the Collection's one validator: it
covers configuration and annotation writes alike, so a change to `backend` and
a change to `custom` advance the same counter. (The name is the same one the
[`changes` query profile](#query-profile-changes) reports per Resource; this is
its Collection-scoped counterpart.) It is independent of every Resource's
validator -- writing a Resource never advances it, and writing this object
never advances a Resource's -- and of the governing history log's, which
versions its own sub-resource (see [[[#governing-log-versioning]]]).

A server implementing these operations MUST maintain the version and MUST honor
`If-Match` and `If-None-Match: *` preconditions on the update, evaluated
atomically with the write; see [[[#conditional-requests]]]. The version is
surfaced only as the `ETag`: it is not a member of the object's JSON
representation. Because the object exists exactly as long as its Collection
does, `If-None-Match: *` here means "write only if the Collection does not
exist" -- the create-if-absent precondition of
[[[#update-or-create-by-id-collection-operation]]].

Carrying configuration and annotation on one validator means a configuration
change invalidates an in-flight annotation write. That is intended. A change to
`encryption` requires the `custom` envelope to be re-sealed in any case, and
for anything else the client re-reads and retries. A client that wants to
update `custom` alone still resends the configuration members it just read,
because `PUT` is a full replacement.

Writes of this object are invisible to replication: the
[`changes` query profile](#query-profile-changes) is per-Resource and emits no
entry for them, so a client mirroring the object re-reads it and compares the
returned `ETag`.

#### Lifecycle {#collection-metadata-lifecycle}

The Metadata object and its Collection share one lifecycle. The object comes
into existence with the Collection, written by the `POST` or `PUT` that creates
it, and there is no state in which a Collection exists without one: a `GET` of
`meta` on an existing Collection always succeeds and always carries an `ETag`.

There is no `DELETE` at `meta`. Deleting the Collection removes its Metadata
object with it. To clear the user's annotations alone, send a `PUT` carrying
the configuration members unchanged and an empty `custom` object, or none at
all; both forms clear on an encrypted Collection too (as above). Clearing is
itself a write and advances `metaVersion`. A Collection later re-created under
the same id starts over: its
`metaVersion` restarts, so a client MUST NOT compare validators across a delete
and a re-create.

### Create Collection (Add Collection to a Space) operation {#create-collection-add-collection-to-a-space-operation}

A Collection is added to a Space by `POST`ing its Metadata object (see
[[[#collection-metadata-data-model]]]) to the Space. The client can specify the
`id` of the Collection in the body. If the `id` is not specified, one is
auto-generated by the server and returned as part of the `Location` response
header.

#### (HTTP API) POST `/space/{space_id}/`

Example request (`id` not specified, auto-generated by the server and returned
in the response `Location` header):

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/ HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"]
}
```
In this example, the `id` in the body is not specified, and will be auto-generated
by the server. The `backend` is not specified, and will be assigned the default
value.


Example response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/21f81693-f4a3-4caa-b81c-b663d6e1e3ae/
```

Example request, a valid `id` specified in the body:

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/ HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "id": "credentials",
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"],
  "backend": { "id": "default" }
}
```

Example response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/credentials/
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=reserved-id=] (409) -- the supplied Collection `id` collides with one of the
  [[[#space-level-reserved-endpoints]]] (for example, `query` or `collections`).
* [=id-conflict=] (409) -- a Collection with the supplied `id` already exists.
  To create or replace a Collection at a client-chosen `id` without conflict,
  use the idempotent [[[#update-or-create-by-id-collection-operation]]] instead.
  Servers MUST perform this existence check only *after* the caller's
  authorization has been verified: an under-authorized caller receives the
  privacy-merged [=not-found=] (404) per [[[#error-handling]]], so that it
  cannot use the `409` to probe a Space for existing Collection ids.
* [=unsupported-backend=] (409) -- the supplied `backend` id is not in that
  space's [[[#space-backends-available]]] list.
* [=invalid-request-body=] (400) -- the request body is not a JSON object,
  or it breaks a validation rule of the Collection Metadata object (see
  [[[#collection-metadata-data-model]]]) or of its key-epoch and `hmac`
  members (see [[[#key-epoch-server-validation]]]).

### Read Collection Metadata operation {#read-collection-metadata-operation}

* Returns the [[[#collection-metadata-data-model]]] for the specified Collection
* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request must be
    signed by the space's [=controller=], or invoke a delegated capability that
    allows the [=GET=] action
* The response includes an `ETag` header over the object's `metaVersion`, for
  use with `If-Match` on a subsequent update (see [[[#conditional-requests]]]);
  a request carrying an `If-None-Match` that covers it is answered
  `304 Not Modified` with the `ETag` and no body (see [[[#caching]]])
* On a Collection governed by its history log, the `encryption` member is
  derived from the log's head entry and carries a `history` member naming the
  log (see [[[#collection-governing-history-log]]])

#### (HTTP API) GET `/space/{space_id}/{collection_id}/meta`

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/meta HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example response:

```http
HTTP/1.1 200 OK
Content-type: application/json
ETag: "3"

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/",
  "type": ["Collection"],
  "name": "Verifiable Credentials Collection",
  "createdAt": "2026-06-10T09:12:00Z",
  "updatedAt": "2026-06-12T13:25:00Z",
  "createdBy": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW",
  "linkset": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/linkset",
  "backend": { "id": "default" },
  "custom": {
    "tags": { "project": "demo" }
  }
}
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Collection does not exist, or the caller has
  missing or insufficient authorization; per [[[#error-handling]]] a Collection
  the caller is not authorized to read is indistinguishable from one that does
  not exist.
* [=unsupported-operation=] (501) -- the server does not implement the optional
  Collection management operations.

### Update (or Create by Id) Collection operation {#update-or-create-by-id-collection-operation}

A `PUT` of a Collection's Metadata object is a full replacement of that object,
and creates the Collection when none exists under that `id`. The client
specifies the `id` of the Collection in the request URL; it MUST NOT collide
with the list of [[[#space-level-reserved-endpoints]]].

* Requires appropriate authorization: a root capability invoked directly by
  the space's [=controller=]. Exception: a delegated capability MAY
  authorize this write when the invoked capability's `invocationTarget` is
  the Space's own canonical (trailing-slash) URL, rather than the
  Collection container URL, the Collection Metadata URL, or a Resource URL
  -- the Collection-metadata exception to the container rule, see
  [[[#the-container-rule]]].

Full replacement means that a writable member the request omits is cleared, so
a client that wants to change one member reads the object, edits it, and writes
it back. Some members qualify that rule. The server-managed members are
read-only: a server MUST ignore `createdAt`, `updatedAt`, `createdBy`, `url`,
and `linkset` in a request body, so that a read-modify-write roundtrip needs no
stripping. On a Collection whose `encryption` descriptor is governed by its
history log, the server derives that member from the log's head and refuses a
direct write of it (see [[[#collection-governing-history-log]]]). An update
that omits `plaintext` leaves the stored member untouched (see its member
definition). And an update that omits `backend` MUST keep the stored backend
selection. Clearing it would repoint the Collection at the default backend. The
Resources already stored in the selected backend would become unreachable, and
later writes would land elsewhere. Only a create that omits `backend` is
assigned the default.

The request MAY carry a precondition (see [[[#conditional-requests]]]):
`If-Match: "<etag>"` performs the write only if the object's current `ETag`
matches it, and `If-None-Match: *` performs the write only if the Collection
does not exist yet. A failed precondition is [=precondition-failed=] (412); an
unconditional `PUT` remains last-writer-wins. The success response carries the
object's new `ETag`.

#### (HTTP API) PUT `/space/{space_id}/{collection_id}/meta`

Example successful "create" request. A `PUT` that creates the Collection and a
`PUT` that reconfigures an existing one are the same operation:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/meta HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"],
  "backend": { "id": "default" }
}
```

The `Location` header names the Collection that was created, not the Metadata
object that was written:

```http
HTTP/1.1 201 Created
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/
ETag: "1"
```

Example request annotating an existing Collection. The configuration members
are resent unchanged, because this is a full replacement, and the `If-Match`
precondition makes the write conditional on nothing else having changed in the
meantime:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/meta HTTP/1.1
Host: example.com
Content-Type: application/json
If-Match: "1"
Authorization: ...

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"],
  "backend": { "id": "default" },
  "custom": {
    "tags": { "project": "demo" }
  }
}
```

```http
HTTP/1.1 204 No Content
ETag: "2"
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the caller has missing or insufficient authorization,
  per [[[#error-handling]]]. A Collection that does not exist is created by
  this operation rather than being an error.
* [=reserved-id=] (409) -- the supplied Collection `id` collides with one of the
  [[[#space-level-reserved-endpoints]]] (for example, `collections` or
  `linkset`).
* [=encryption-immutable=] (409) -- the update tried to change the `scheme`,
  decrease or remove the `version`, or clear an existing `encryption`
  descriptor (the descriptor is set-once, version-monotonic; see
  [[[#collection-metadata-data-model]]]). Also raised when the update changes the `id`
  or `type` of the descriptor's `hmac` member, or removes that member; the
  blinding key is permanent once present (see [[[#blinding-key-member]]]).
* [=invalid-request-body=] (400) -- the request body is not a JSON object,
  or it breaks a validation rule of the Collection Metadata object (see
  [[[#collection-metadata-data-model]]]), its key-epoch and `hmac` members
  (see [[[#key-epoch-server-validation]]]), or its top-level `epoch` member
  (see [[[#epoch-stamping-on-resources]]]).
* [=encryption-history-log-governed=] (409) -- the update carries an
  `encryption` member on a Collection whose descriptor is governed by its
  history log; the member is read-only on this path and changes by an append
  to the log (see [[[#collection-governing-history-log]]]).
* [=id-conflict=] (409) -- the update adds `unique: true` to a
  `plaintext.indexes` entry whose already-stored Resources violate the
  claim; the stored declaration is left unchanged (see
  [[[#collection-metadata-data-model]]]).
* [=encryption-scheme-mismatch=] (422) -- on an encrypted Collection, the
  request carries a `custom` that is neither a conforming envelope of the
  Collection's declared scheme nor an empty object. An empty or omitted
  `custom` is not an error: it clears the stored annotations, as on a
  plaintext Collection (see [[[#collection-metadata-data-model]]]).
* [=precondition-failed=] (412) -- the request carried an `If-Match`
  precondition and the object's current `ETag` does not match it, or an
  `If-None-Match: *` precondition against a Collection that already exists
  (see [[[#conditional-requests]]]).
* [=unsupported-operation=] (501) -- the server does not implement the optional
  Collection management operations.

### List Collection operation {#list-collection-operation}

* Returns the list of Collection resources
* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request must be
    signed by the space's [=controller=], or invoke a delegated capability that
    allows the [=GET=] action
* MAY be paginated (see [[[#pagination]]]); the example below shows a single
  unpaginated page, and the paginated example that follows shows the `next` link
  and cursor continuation.

#### (HTTP API) GET `/space/{space_id}/{collection_id}/`

Example request (a container URL, so it carries the trailing slash):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/ HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/",
  "name": "Example JSON Documents Collection",
  "type": ["Collection"],
  "totalItems": 2,
  "items": [
    {
      "id": "321efd4e-23cb-497c-aaee-7bd26e66d39e",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/321efd4e-23cb-497c-aaee-7bd26e66d39e",
      "contentType": "application/json"
    },
    {
      "id": "3943c87f-b617-44bc-ba75-8de2b16c3640",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/3943c87f-b617-44bc-ba75-8de2b16c3640",
      "contentType": "application/json"
    }
  ]
}
```

##### Paginated example

A client requests a bounded page with `limit` (here, two items from a Collection
that holds more):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/?limit=2 HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

The response carries the page of items plus a `next` link, signalling that more
items follow. `totalItems` is omitted here -- the server does not return a total
for this (potentially large) Collection:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/",
  "name": "Example JSON Documents Collection",
  "type": ["Collection"],
  "items": [
    {
      "id": "321efd4e-23cb-497c-aaee-7bd26e66d39e",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/321efd4e-23cb-497c-aaee-7bd26e66d39e",
      "contentType": "application/json"
    },
    {
      "id": "3943c87f-b617-44bc-ba75-8de2b16c3640",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/3943c87f-b617-44bc-ba75-8de2b16c3640",
      "contentType": "application/json"
    }
  ],
  "next": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/?limit=2&cursor=eyJhZnRlciI6IjM5NDNjODdmLWI2MTctNDRiYy1iYTc1LThkZTJiMTZjMzY0MCJ9"
}
```

The client follows `next` verbatim to fetch the subsequent page (the `cursor` is
opaque and supplied by the server -- the client does not build it):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/?limit=2&cursor=eyJhZnRlciI6IjM5NDNjODdmLWI2MTctNDRiYy1iYTc1LThkZTJiMTZjMzY0MCJ9 HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

The final page omits `next`, marking the end of the list:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/",
  "name": "Example JSON Documents Collection",
  "type": ["Collection"],
  "items": [
    {
      "id": "9c7b2e51-0d4a-4c2f-8b3e-1f6a5d8e7c90",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/9c7b2e51-0d4a-4c2f-8b3e-1f6a5d8e7c90",
      "contentType": "application/json"
    }
  ]
}
```

Where a Resource carries a key-epoch stamp (see [[[#key-epochs]]]), its item
summary additionally carries it as `epoch` (optional), mirroring the Resource
Metadata property -- so a reader walking a listing can select its epoch key
without a per-Resource metadata fetch. A writer-attribution label (see
[[[#writer-attribution]]]) is likewise carried as `writerId` (optional),
mirroring the Resource Metadata property.

On a Collection that declares an `encryption` descriptor (see
[[[#encryption-scheme-registry]]]), each item's user-writable metadata is stored
encrypted (see [[[#resource-metadata-data-model]]]). Item summaries for an
encrypted Collection therefore carry only server-visible fields (`id`, `url`,
`contentType`, and, where declared, `epoch` and `writerId`) and omit `name`; a
client that needs names decrypts each Resource's Metadata itself.

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Collection does not exist, or the caller has
  missing or insufficient authorization; per [[[#error-handling]]] a Collection
  the caller is not authorized to read is indistinguishable from one that does
  not exist.

### Delete Collection operation {#delete-collection-operation}

#### (HTTP API) DELETE `/space/{space_id}/{collection_id}/`

* Requires appropriate authorization: a root capability invoked directly by
  the space's [=controller=]. A delegated capability never authorizes this
  operation, whatever its `allowedAction` (see [[[#the-container-rule]]]).

* This operation is idempotent

* (Assuming the request carries appropriate authorization) Sending a DELETE
  request to a collection that does not exist (or has already been deleted)
  results in a 204 success response

Example request:

```http
DELETE /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/ HTTP/1.1
Host: example.com
Authorization: ...
```

Example success response:

```http
HTTP/1.1 204 No Content
```

### Collection Governing History Log {#collection-governing-history-log}

A Collection MAY carry one **governing history log**: an append-only,
client-written log that the server stores at the sub-path
`/space/{space_id}/{collection_id}/meta/log` and from whose head entry the
server derives a member of the Collection's Metadata object. The clients that
share the Collection write the log and read it. The server stores it, checks
a minimal line contract on every write, and serves the derived member to
every reader. Which members a log governs, what an entry is beyond its
`state`, and how a reader verifies the log are defined by a **governing
profile**, not by this specification. The `encryption` descriptor of the
[[[#collection-metadata-data-model]]] is the first governed member, under the
resource log profile of [[PWS-EC]]; see
[[[#governed-encryption-descriptor]]].

Support is OPTIONAL and discoverable. A server that implements the
sub-resource advertises `governed-history-logs` in its PWS-EC version entry's
`features` array (see [[[#service-description-data-model]]]). A server without
the affordance serves a client-written member instead, so a producer MUST
consult the token before choosing which form to write. A [=server=] that does
not implement the
sub-resource SHOULD return an [=unsupported-operation=] (501) error for
requests to its path, and MUST NOT treat them as Resource operations (the
path lies under the reserved `meta` segment; see
[[[#collection-level-reserved-endpoints]]]).

#### The sub-resource {#governing-log-sub-resource}

The log is a sub-resource of the Collection, addressed beside its Metadata
object under the reserved `meta` segment. It is neither a Resource of the
Collection nor a member of the Metadata object:

* It does not appear in the Collection listing
  ([[[#list-collection-operation]]]) or in the `changes` query profile's feed
  ([[[#query-profile-registry]]]).
* It is exempt from the encrypted-Collection envelope rule
  ([[[#encryption-scheme-registry]]]): the log is plaintext JSON Lines on an
  encrypted Collection too.
* It is not part of the `meta` body. A [[[#read-collection-metadata-operation]]]
  response does not include it, and an
  [[[#update-or-create-by-id-collection-operation]]] write does not touch it.
* It carries a validator of its own (see [[[#governing-log-versioning]]]),
  independent of the Metadata object's and every Resource's.
* It is created and deleted with the Collection. Deleting the Collection
  removes the log, and a Collection later re-created under the same id
  starts with no log.

Authorization follows the Collection. The log is readable under any
capability whose `invocationTarget` covers the Collection URL, so a share
grantee or an app reads it with the capability it already holds; a read is
also served where the Collection's access-control [=policy=] grants reads. A
write requires a capability that allows the [=PUT=] action at the log's URL;
a policy grants no writes to it. As everywhere in this specification, a
caller that is not authorized receives the merged [=not-found=] (404), per
[[[#error-handling]]].

#### Line contract {#governing-log-line-contract}

The log body is JSON Lines [[JSON-LINES]]: one JSON object per line, lines
separated by U+000A LINE FEED, with an optional trailing line feed. The whole
of what a server requires of the content is:

* each line is a JSON object carrying a `state` member whose value is a JSON
  object;
* the last line is the **head** entry, whose `state` is the current state of
  what the log governs;
* the first (genesis) line MAY carry a string `parameters.method`, the log
  format identifier the derived member's `history.method` echoes.

A server MUST refuse a write whose body breaks this contract with an
[=invalid-request-body=] (400) error: an empty body, a line that is not a
JSON object, a line without an object `state`, or a blank line other than the
trailing line feed. A server MUST NOT require anything else of a line.
Proofs, hash chaining, the `type` of a `state`, and any member name reserved
inside `state` are defined by the governing profile (for the resource log
format, [[PWS-EC]]) and are not checked here. The server stores the log; it
does not verify it.

#### Declaration and the derived member {#governing-log-declaration}

The guarded create of the log is the declaration. A `PUT` to the sub-resource
carrying `If-None-Match: *` (see [[[#conditional-requests]]]) on a Collection
that has no log makes the Collection **log-governed**. Nothing is added to
the Collection's Metadata object, and a Collection stays governed until it is
deleted.

From that write on, the server derives the governed member from the head
entry: the served value is the head's `state` with one member stamped on,

```json
"history": {
  "method": "<the genesis entry's parameters.method>",
  "resource": "<the absolute URL of the log>"
}
```

Where the genesis entry carries no `parameters.method`, the server serves
the head `state` without `history`. The derived member appears wherever the
Metadata object is served; the stored object holds no copy of
it. Derivation is last-line parsing, so the derived member is exactly what a
verifying reader computes from the head after stripping `history`.

A direct write of a governed member through the
[[[#update-or-create-by-id-collection-operation]]] MUST be refused; the
member is read-only on that path, and the object's other members update
normally. For the `encryption` descriptor the refusal is
[=encryption-history-log-governed=] (409).

Declaring governance on a Collection whose Metadata object already holds a
client-written value of the governed member MUST be refused. There is no
conversion between the two forms in this version: a Collection is
provisioned in one form or the other. For the `encryption` descriptor the
refusal is [=encryption-immutable=] (409).

#### Trust {#governing-log-trust}

The log is the only authoritative serving of what it governs. The derived
member is a projection the server computes by parsing the head line, for its
own checks and for readers that do not verify; the server verifies neither
entry proofs nor the chain. A verifying reader reads the log, verifies it
under the governing profile, and compares the derived member to the verified
head's `state` after stripping `history`. A derived member that differs is a
stale projection, and the reader acts on the verified head instead (for the
resource log format, the equality check of [[PWS-EC]]).

#### Versioning {#governing-log-versioning}

The log carries its own monotonic version, exposed as a strong `ETag`
validator on read, on the same terms as the Collection Metadata object (see
[[[#collection-metadata-versioning]]]). A server implementing the
sub-resource MUST maintain it and MUST honor `If-Match` and `If-None-Match: *`
on write, evaluated atomically with the write. A log that does not exist has no
validator, and `If-None-Match: *` succeeds exactly then. Because a log write
changes the derived member, it also advances the Collection's `metaVersion`: a
client holding that `ETag` sees the change on its next conditional read. A
Collection re-created under the same id restarts the log's version, so a client
MUST NOT compare validators across a delete and re-create.

#### The first governed member: `encryption` {#governed-encryption-descriptor}

The `encryption` descriptor (see [[[#collection-metadata-data-model]]]) is the
first, and in this version the only, member a profile governs. On a log-governed
Collection the head entry's `state` is an encryption descriptor, and the
Collection is an encrypted Collection from the declaration onward: the
envelope rule applies to its Resources and its Metadata object, and the
`plaintext` member is excluded, exactly as when the descriptor is written on
the Metadata object. The profile's `state` carries a `type` member naming its
schema ([[PWS-EC]]); the server stores it with the rest of the state and
does not interpret it.

On every log write the server runs, on the head `state`, the same checks the
[[[#update-or-create-by-id-collection-operation]]] runs on a supplied
descriptor: the shape validation of [[[#key-epoch-server-validation]]] and
the scheme recognition rule of [[[#encryption-scheme-registry]]]. On an
append it also runs the transition invariants against the prior head's
`state`: `epochs` is append-only, `currentEpoch` never moves backwards,
`hmac` is permanent once present, and the descriptor's `scheme` and
`version` are set-once, version-monotonic. A violating write MUST be refused
with the error the Update Collection operation raises for the same change,
and MUST leave the log unchanged. This is the one clause of this mechanism
that is specific to encryption; a later governed member adds its own.

### Read Governing History Log Operation {#read-governing-history-log-operation}

#### (HTTP API) GET `/space/{space_id}/{collection_id}/meta/log`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request must
    either: be signed by the space's [=controller=], or invoke a delegated
    capability that allows the [=GET=] action whose target covers the
    Collection URL
* Returns the log body verbatim, as `text/jsonl`, with an `ETag` header over
  the log's version (see [[[#governing-log-versioning]]])
* A request carrying `If-None-Match` with the current validator is answered
  `304 Not Modified` with the `ETag` and no body (see [[[#caching]]])

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/meta/log HTTP/1.1
Host: example.com
Accept: text/jsonl
Authorization: ...
```

Example success response (one entry per line; the entry members beyond
`state` are the governing profile's):

```http
HTTP/1.1 200 OK
Content-type: text/jsonl
ETag: "z3fkq.2"

{"versionId":"1-...","versionTime":"2026-09-07T10:00:00Z","parameters":{"method":"resource-log:0.1","scid":"z..."},"state":{"type":"WasEpochConfiguration","scheme":"edv","currentEpoch":"urn:uuid:8d3f...","epochs":[...]},"proof":[...]}
{"versionId":"2-...","versionTime":"2026-09-08T09:30:00Z","parameters":{},"state":{"type":"WasEpochConfiguration","scheme":"edv","currentEpoch":"urn:uuid:c17a...","epochs":[...]},"proof":[...]}
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Collection does not exist, it carries no log, or
  the caller has missing or insufficient authorization; per
  [[[#error-handling]]] a log the caller is not authorized to read is
  indistinguishable from one that does not exist.
* [=unsupported-operation=] (501) -- the server does not implement the
  optional governing history log.

### Write Governing History Log Operation {#write-governing-history-log-operation}

The body of a write is the whole log. A client appends by sending the stored
bytes verbatim followed by the new line, under an `If-Match` precondition
carrying the validator of the log it read. A client creates the log, and so
declares the Collection governed, with `If-None-Match: *`. A client MUST
carry one of the two preconditions on every write. A server evaluates the
precondition it is given atomically with the write. On a Collection that
already carries a log, a server MUST also check, atomically with the write,
that the body fast-forwards the stored log: the stored bytes verbatim
followed by exactly one new line. A body the stored log is not a prefix of
(a stale read, or a rewritten prefix) MUST be refused with a
[=precondition-failed=] error (412), whether or not the write carries
`If-Match`; a body that extends the stored bytes by no line or by several
MUST be refused with an [=invalid-request-body=] error (400). A write
capability can therefore add to a log but not erase or rewrite it. The
server also checks the line contract and, for a governed `encryption`
descriptor, the head-state transition
([[[#governed-encryption-descriptor]]]). It verifies nothing inside an
appended entry: a bad proof or a broken hash link is a chain break that a
verifying reader detects under the governing profile.

#### (HTTP API) PUT `/space/{space_id}/{collection_id}/meta/log`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request must
    either: be signed by the space's [=controller=], or invoke a delegated
    capability that allows the [=PUT=] action whose target covers the log's
    URL
* The request body is the whole log as `text/jsonl`
* Carries `If-None-Match: *` (guarded create, the declaration) or
  `If-Match: "<etag>"` (compare-and-swap append); see
  [[[#conditional-requests]]]
* Does not create a Collection: a write to the log path of a nonexistent
  Collection returns a [=not-found=] (404) error
* Returns a `204` success response, with an `ETag` header carrying the log's
  new validator

Example request (the guarded create, with a genesis entry):

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/meta/log HTTP/1.1
Host: example.com
Content-Type: text/jsonl
If-None-Match: *
Authorization: ...

{"versionId":"1-...","versionTime":"2026-09-07T10:00:00Z","parameters":{"method":"resource-log:0.1","scid":"z..."},"state":{"type":"WasEpochConfiguration","scheme":"edv","currentEpoch":"urn:uuid:8d3f...","epochs":[...]},"proof":[...]}
```

Example success response:

```http
HTTP/1.1 204 No Content
ETag: "z3fkq.1"
```

From this point the Collection Metadata object's `encryption` member reads as
the entry's `state` plus `history`:

```json
"encryption": {
  "type": "WasEpochConfiguration",
  "scheme": "edv",
  "currentEpoch": "urn:uuid:8d3f...",
  "epochs": [...],
  "history": {
    "method": "resource-log:0.1",
    "resource": "https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/meta/log"
  }
}
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Collection does not exist (this operation does
  not create one), or the caller has missing or insufficient authorization,
  per [[[#error-handling]]].
* [=invalid-request-body=] (400) -- the body breaks the line contract
  ([[[#governing-log-line-contract]]]); or, for a governed `encryption`
  descriptor, the head `state` is malformed or the append violates an epoch
  invariant, as the Update Collection operation would report it.
* [=unsupported-encryption-scheme=] (400) -- the head `state` declares a
  `scheme` (or a `version` of one) the server does not recognize.
* [=encryption-immutable=] (409) -- the guarded create targets a Collection
  whose Description already holds a client-written `encryption` descriptor;
  or the append changes the `scheme`, moves the `version` backward, or
  changes or drops the `hmac` member.
* [=precondition-failed=] (412) -- an `If-Match` / `If-None-Match: *`
  precondition evaluated false: another writer appended first, or the log
  already exists; see [[[#conditional-requests]]].
* [=unsupported-operation=] (501) -- the server does not implement the
  optional governing history log.

## Resources and Blobs {#resources-and-blobs}

### Blob Data Model {#blob-data-model}

A unit of data, in transit or at rest.
(As described in the [W3C FileAPI: Blob Interface](https://w3c.github.io/FileAPI/#blob-section)).

Blob properties:

* byte stream
* `size` - length of the byte stream in bytes
  - Note: Although size can be derived from bytes, it's useful to be able to
    have it up front (for the receiving system to decide to reject an upload
    based on quota / exceeding max size, etc).
* `type` (from the IANA mime type registry) - If not specified, defaults to
  `application/octet-stream`

When a Blob is stored as a Resource, its `type` and `size` are reported as
the `contentType` and `size` properties of the Resource's Metadata object
(see [[[#resource-metadata-data-model]]]), and the stored bytes are returned
verbatim on reads (see [[[#content-types-and-representations]]]).

### Resource Data Model {#resource-data-model}

A resource is a named (addressable) Blob stored in a given [=collection=],
with metadata.
The data model is derived from [W3C FileAPI: File
Interface](https://w3c.github.io/FileAPI/#file-section), but with the addition
of a few crucial properties.

In similar storage systems, a resource is called "File", "Object", "Document",
"Row", "Graph", and so on.

Resource properties:

* `id` - A unique identifier for a resource for a given collection. Created by the
  server if not provided. Note: the `{resource_id}` template parameter used in URL
  templates in this spec MUST match that resource's `id` property (as returned by
  the [[[#list-collection-operation]]]).
* `name` - (optional) - Human-readable name for the resource, useful for building
  user interfaces for browsing collection contents.
* `type` (optional) - An array of strings describing the resource. Unlike Space
  and Collection, no specific `type` value is required of a Resource.
* `url` (optional) - A relative URL (provided by the server when listing resources
  via the [[[#list-collection-operation]]])
* `contentType` (optional) - The MIME type of the default representation of the
  resource (provided by the server when listing resources
  via the [[[#list-collection-operation]]])
* Each Resource also has an associated Metadata object -- server-managed
  properties (`contentType`, `size`, optional timestamps) plus user-writable
  ones (`name` and `tags`, nested under `custom`) -- addressable at the
  reserved `/meta` path segment under the Resource URL. See
  [[[#resource-metadata-data-model]]].

### Content Types and Representations {#content-types-and-representations}

A Resource has exactly one current representation: the stored bytes plus
the content type they are stored under. Every successful write (POST or PUT)
replaces that representation entirely -- including replacing a representation
previously stored under a different content type. Servers are not required to
store, convert between, or negotiate multiple representations of a Resource.

**Writing.** A Resource write request MUST carry a `Content-Type` header; a
server MUST reject a write without one with a [=missing-content-type=] (400)
error. The request body is interpreted according to its content type:

* **JSON** -- a content type of `application/json`, or any type with a
  `+json` structured-syntax suffix [[RFC6839]] (for example,
  `application/ld+json`), is stored as a JSON document.
* **Raw binary** (support REQUIRED) -- any other content type is stored as an
  opaque byte stream (see [[[#blob-data-model]]]), verbatim, under the
  supplied content type. Clients with no more specific type SHOULD use
  `application/octet-stream`.
* **Multipart upload** (support OPTIONAL) -- a server MAY additionally accept
  `multipart/form-data` [[RFC7578]] uploads (the HTML form workflow). The
  request MUST contain exactly one file part; that part supplies the stored
  bytes, and its own content type (not `multipart/form-data`) becomes the
  stored content type. Because a write targets a single Resource, there is no
  batch form: a server MUST reject a multipart request with no file part, or
  with more than one, with an [=invalid-request-body=] (400) error.

**Reading.** A `GET` returns the current representation, verbatim, with the
stored content type as the response `Content-Type`. Because Resources are
single-representation, the request's `Accept` header is advisory: a server
MAY ignore it, and MUST NOT reject a Resource read for lack of an acceptable
representation (no `406 Not Acceptable`) -- the stored representation is
always the answer. A `HEAD` request returns the same headers without the
body; the response `Content-Type` and `Content-Length` correspond to the
`contentType` and `size` properties of the Resource's Metadata object (see
[[[#resource-metadata-data-model]]]).

**Server-generated documents.** API documents generated by the server --
Metadata objects, listings, quota reports -- MUST be available as
`application/json`. Linkset documents use
`application/linkset+json` [[RFC9264]], and error responses use
`application/problem+json` [[RFC9457]] (see [[[#error-handling]]]). Other
representations of these documents (negotiated via `Accept`) MAY be offered
in addition to, never instead of, the JSON baseline.

### Create Resource (Add Resource to Collection) Operation {#create-resource-add-resource-to-collection-operation}

#### (HTTP API) POST `/space/{space_id}/{collection_id}/`

The request MAY include a `Key-Epoch` header declaring the key epoch the
body was encrypted under (see [[[#key-epochs]]]); the value is stored as the
Resource Metadata `epoch` property. When absent, any stored `epoch` stamp is
cleared. The request MAY likewise include a `Writer-Id` header declaring the
writing agent's attribution label (see [[[#writer-attribution]]]); the value
is stored as the Resource Metadata `writerId` property, and when absent any
stored `writerId` is cleared.

Example request (adds a JSON object to the `messages` collection).
The server generates the Resource id and returns it in the `Location` response
header. A client that chooses the id uses
[[[#update-or-create-by-id-resource-operation]]] instead.

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/ HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{"message":"hi"}
```

Example success response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/6b5be748-5f39-4936-a895-409e393c399c
```

Example request (multipart upload of a photo to the `photos` collection, on
a server that supports the OPTIONAL multipart form; see
[[[#content-types-and-representations]]]). The stored content type is the
file part's own type, `image/png`, not `multipart/form-data`:

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/photos/ HTTP/1.1
Host: example.com
Content-Type: multipart/form-data; boundary=----boundary314
Authorization: ...

------boundary314
Content-Disposition: form-data; name="file"; filename="sunset.png"
Content-Type: image/png

...binary PNG bytes...
------boundary314--
```

Example success response:

```http
HTTP/1.1 201 Created
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/photos/d2887cf0-186f-4e2e-a575-c9ce1b9d8a45
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=missing-content-type=] (400) -- the write request carries no
  `Content-Type` header (see [[[#content-types-and-representations]]]).
* [=invalid-request-body=] (400) -- the body does not match its declared
  content type (for example, a multipart upload with no file part, or with
  more than one).
* [=id-conflict=] (409) -- the write's extracted value for a `unique: true`
  [=plaintext index=] is already held by a different Resource in the
  Collection (see [[[#collection-metadata-data-model]]]). Checked only after
  the caller's authorization has been verified, and atomically with the write.
* [=quota-exceeded=] (507) -- the Collection's backend has no storage quota
  remaining (see [[[#quotas]]]).
* [=payload-too-large=] (413) -- the upload exceeds the backend's
  `maxUploadBytes` constraint (see [[[#quotas]]]).

### Read Resource Operation {#read-resource-operation}

A read returns the Resource's single stored representation, with the stored
content type; see [[[#content-types-and-representations]]] for content type
and `Accept` header handling (including the `HEAD` variant).

#### (HTTP API) GET `/space/{space_id}/{collection_id}/{resource_id}`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for
    authorization, the request must either: be signed by the resource's or the
    space's [=controller=], or invoke a delegated capability that allows the
    [=GET=] action

Example request to retrieve a resource:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Accept: application/json
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{"message":"hi"}
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Resource does not exist, or the caller has
  missing or insufficient authorization; per [[[#error-handling]]] a resource
  the caller is not authorized to read is indistinguishable from one that does
  not exist.

### Update (or Create By Id) Resource Operation {#update-or-create-by-id-resource-operation}

When creating or modifying a Resource via PUT, the client specifies the `id`
of the Resource. This Resource `id` MUST NOT collide with the list of
[[[#collection-level-reserved-endpoints]]].

#### (HTTP API) PUT `/space/{space_id}/{collection_id}/{resource_id}`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for
    authorization, the request must either: be signed by the resource's or the
    space's [=controller=], or invoke a delegated capability that allows the
    [=PUT=] action
* This operation is idempotent
* Returns a `204` success response

The request MAY include a `Key-Epoch` header declaring the key epoch the
body was encrypted under (see [[[#key-epochs]]]); the value is stored as the
Resource Metadata `epoch` property. When absent, any stored `epoch` stamp is
cleared. The request MAY likewise include a `Writer-Id` header declaring the
writing agent's attribution label (see [[[#writer-attribution]]]); the value
is stored as the Resource Metadata `writerId` property, and when absent any
stored `writerId` is cleared.

Example request to create a resource via PUT:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{"message":"hi"}
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Example request to update the created resource via PUT:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{"message":"no I've changed my mind"}
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Example request (uploading a binary Blob via PUT; the bytes are stored
verbatim under the supplied content type, see
[[[#content-types-and-representations]]]). The `Digest` header binds the
body to the request signature when using the PWS Authorization Profile (see
[[[#request-body-integrity-digest-header]]]):

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/photos/sunset.png HTTP/1.1
Host: example.com
Content-Type: image/png
Digest: mh=uEiCPO-qYr-z0GYV5F75-N1l8Rhjv4xIkKZsnbTZeZ7emSA
Authorization: ...

...binary PNG bytes...
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=missing-content-type=] (400) -- the write request carries no
  `Content-Type` header (see [[[#content-types-and-representations]]]).
* [=invalid-request-body=] (400) -- the body does not match its declared
  content type (for example, a multipart upload with no file part, or with
  more than one).
* [=not-found=] (404) -- the enclosing Space or Collection is missing or
  invalid, or the caller has missing or insufficient authorization; per
  [[[#error-handling]]] an under-authorized request is indistinguishable from a
  missing target.
* [=id-conflict=] (409) -- the write's extracted value for a `unique: true`
  [=plaintext index=] is already held by a different Resource in the
  Collection (see [[[#collection-metadata-data-model]]]). Checked only after
  the caller's authorization has been verified, and atomically with the write.
* [=quota-exceeded=] (507) -- the Collection's backend has no storage quota
  remaining (see [[[#quotas]]]).
* [=payload-too-large=] (413) -- the upload exceeds the backend's
  `maxUploadBytes` constraint (see [[[#quotas]]]).
* [=precondition-failed=] (412) -- a conditional write's `If-Match` /
  `If-None-Match` precondition evaluated false (see
  [[[#conditional-requests]]]).

This operation accepts the `If-Match` / `If-None-Match` write preconditions
described in [[[#conditional-requests]]].

### Delete Resource Operation {#delete-resource-operation}

#### (HTTP API) DELETE `/space/{space_id}/{collection_id}/{resource_id}`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request
    must either: be signed by the resource's or the space's [=controller=],
    or invoke a delegated capability that allows the
    [=DELETE=] action

* This operation is idempotent

* (Assuming the request carries appropriate authorization) Sending a DELETE
  request to a resource that does not exist (or has already been deleted)
  results in a 204 success response

The request MAY include a `Writer-Id` header declaring the deleting agent's
attribution label (see [[[#writer-attribution]]]): a deletion is a revision
like any other, and where the server keeps a tombstone the label it carries
attributes the deletion (see the tombstone notes of the
[`changes` profile](#query-profile-changes)), on the same declare-or-clear
terms as a content write.

Example request to delete a resource via DELETE:

```http
DELETE /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Authorization: ...
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the caller has missing or insufficient authorization.
  Because DELETE is idempotent, an authorized request for an already-absent
  resource returns `204`; a request that lacks sufficient authorization instead
  returns `404`, per [[[#error-handling]]], so that an unauthorized caller
  cannot probe for existence.

### Resource Metadata Data Model {#resource-metadata-data-model}

Each Resource has an associated **Metadata object**, addressable at the
reserved `/meta` path segment under the Resource URL (one of the
[[[#resource-level-reserved-endpoints]]]). The metadata endpoints are
OPTIONAL; a [=server=] that does not implement them SHOULD return an
[=unsupported-operation=] (501) error for requests to `/meta` paths.
(A Collection has a Metadata object of its own, at the reserved `meta`
segment under the Collection URL; see [[[#collection-metadata-data-model]]].)

The Metadata object is a JSON document containing two groups of
properties:

* **Server-managed properties** appear at the top level of the Metadata
  object. They are maintained by the server as a side effect of Resource
  write operations and are read-only: the
  [[[#update-resource-metadata-operation]]] cannot change them.
* **User-writable properties** are nested under the `custom` property and
  are set by the client via the [[[#update-resource-metadata-operation]]].
  Keeping them in their own object makes the writable surface explicit and
  leaves the top level free for future server-managed properties.

Server-managed properties (`contentType` and `size` are REQUIRED in every
Metadata object; the timestamps and `createdBy` are OPTIONAL):

* `contentType` - The MIME type of the stored representation of the Resource,
  recorded from the `Content-Type` header of the request that last wrote the
  Resource's content (`application/octet-stream` if none was provided). The
  same value is reported in [[[#list-collection-operation]]] results and as
  the `Content-Type` header of `GET`/`HEAD` responses for the Resource itself.
* `size` - The length in bytes of the stored representation of the Resource
  (see the `size` property of the [[[#blob-data-model]]]).
* `createdAt` (optional) - The [[RFC3339]] `date-time` at which the Resource
  was created.
* `updatedAt` (optional) - The [[RFC3339]] `date-time` at which the Resource's
  content or user-writable metadata was last modified.
* `createdBy` (optional) - The [=did=] of the party whose capability invocation
  created the Resource. It is recorded on the first write and preserved
  unchanged by later writes, so it names the creator rather than the last
  writer -- in a Collection written by several delegated agents, `createdBy`
  and the party responsible for the current content need not be the same. Like
  the other server-managed properties it is read-only: it cannot be set through
  the [[[#update-resource-metadata-operation]]]. Recording it is OPTIONAL, so a
  client MUST treat an absent `createdBy` as "not recorded" rather than as an
  assertion that the Resource has no creator.

This metadata model does not mandate a stored `etag` / version property as
such; a Resource's version is always exposed as an HTTP `ETag` validator, per
[[[#conditional-requests]]].

<div class="ednote">
The broader Transaction (multi-write atomic) mechanism remains deferred.
</div>

User-writable properties:

* `custom` (optional) - A JSON object holding the user-writable properties
  below. A Metadata object with no user-writable properties set MAY omit
  `custom` or report it as `{}`.
  * `name` (optional) - A human-readable name for the Resource. This is the
    same `name` property defined in the [[[#resource-data-model]]].
  * `tags` (optional) - A JSON object of application-defined annotations.
    Keys are strings chosen by the application; values SHOULD be strings, so
    that tags stay cheap to index and filter on. Applications that need
    richer structured metadata SHOULD store it as a Resource in its own
    right rather than in `tags`.

  On a plaintext Collection, a `custom` value that is not a JSON object, or a
  `name` or `tags` member that does not have the shape described above, is an
  [=invalid-request-body=] error. On an encrypted Collection, `custom` is
  validated as an envelope instead (see below).
* `epoch` (optional) - The key-epoch `id` this Resource's content was
  encrypted under (see [[[#key-epochs]]]). Declared by the writer via the
  `Key-Epoch` header on a content write or a top-level `epoch` member on
  an Update Resource Metadata request, and stored opaquely. The server
  never computes or verifies it. `epoch` is a sibling field alongside `custom`,
  since on an encrypted Collection `custom` is the opaque envelope and is replaced
  wholesale by every metadata write. An omitted `epoch` on a metadata write
  preserves the stored value; a content write without the header clears it.
* `writerId` (optional) - An opaque label naming the writing agent that
  produced the Resource's current revision (see [[[#writer-attribution]]]
  for its semantics, its deliberate separation from `createdBy`, and its
  privacy considerations). The writer declares it, via the `Writer-Id`
  header on a content write or a delete, or a top-level `writerId` member
  on an Update Resource Metadata request, and the server stores it
  verbatim. The value is not verified, and it MUST NOT be an input to any
  authorization decision. Like `epoch`, it is a sibling of `custom` rather
  than part of it. Unlike `epoch`, whose stamp describes the content
  write, `writerId` describes the latest revision whichever operation
  produced it: every write either declares the writer's own label or
  clears the stored one. A write that declares no `writerId` clears the
  stored value, since attribution to a bygone writer is worse than no
  attribution at all.

On a Collection that declares an `encryption` descriptor (see
[[[#encryption-scheme-registry]]]), the user-writable `custom` object is stored
**encrypted**: its value is an envelope of the Collection's declared scheme
(the same envelope profile used for a Resource's content). The server-managed
top-level properties (`contentType`, `size`, the timestamps, and `createdBy`)
remain plaintext -- the server needs them for listings, `GET`/`HEAD` headers,
quotas, and, in the case of `createdBy`, because it is the very identity the
server authenticated on the creating write. The server
validates the `custom` envelope structurally on write (rejecting a plaintext
`custom` with an [=encryption-scheme-mismatch=]) and never decrypts it; a client
holding the keys decrypts `custom` back to `{ name, tags, ... }` after reading.
An empty `custom` object is accepted as well and clears the stored value, and
so does an omitted `custom`, here as on a plaintext Collection.
Because the server cannot read an encrypted `name`, [[[#list-collection-operation]]]
summaries for an encrypted Collection carry no `name`.

A Resource's Metadata object is created and deleted together with the
Resource itself: it comes into existence (with only server-managed
properties) when the Resource is created and is removed when the Resource is
deleted. There is no `DELETE /meta` operation; to clear the user-writable
properties, send a `PUT` with an empty `custom` object (`{ "custom": {} }`)
or an empty body object (`{}`); both forms clear on an encrypted Collection
too.

#### Writer attribution: `writerId` and `createdBy` {#writer-attribution}

`createdBy` and `writerId` (both optional) answer different questions, and
neither can substitute for the other:

* `createdBy` is a **keyed, server-verified identity**: the [=did=] the
  server itself authenticated on the creating capability invocation. It is
  server-managed, read-only, and names the creator only; later writes
  never change it.
* `writerId` is an **unkeyed, client-declared attribution label**: an opaque
  string the writing agent volunteers about itself, naming which writing
  agent produced the current revision. The server stores and serves it
  verbatim and MUST NOT verify it, MUST NOT compute or default it, and MUST
  NOT use it as an input to authorization or any other server decision. It
  is advisory replication metadata, nothing more.

These two leave room for a third, distinct axis: a client-asserted identity
-- a property whose value is a [=did=] the controller writes and maintains
about another party. The Collection-level `generator` property (see
[[[#collection-metadata-data-model]]]), naming the application a Collection
was provisioned for, is this axis. Unlike `writerId` it is an
identity claim and a stable join key; unlike `createdBy` it is a controller
assertion rather than a server-verified fact, and a reader treats it as
exactly that.

In every usage, including the `Writer-Id` request header and the top-level
`writerId` member of an Update Resource Metadata request, the label is an opaque
non-empty string. A present but empty or malformed value is an
[=invalid-request-body=] error.

The reason `writerId` is necessary: several distinct writing agents may share
one invoking [=did=] (for example, the same delegated capability held by a
user's several installs of one app), so `createdBy` (or any other
server-verified identity) structurally cannot distinguish exactly the
writers that most often race each other. A per-revision label lets
replication clients break same-timestamp last-writer-wins ties with a
shared, deterministic `(updatedAt, writerId)` sort key. It also helps them
recognize their own writes echoed back on the
[`changes`feed](#query-profile-changes) without a fetch (on an encrypted
Collection, without a decrypt), and attribute revision history for display.

Requirements on the writing client:

* A `writerId` MUST NOT be derived from any secret, key material, or other
  identity: it is an attribution label, never an identity, and it MUST NOT
  be treated as one (that is the `createdBy` axis). A random value minted
  per writing agent is RECOMMENDED.
* A client performing a read-modify-write of a Metadata object SHOULD
  replace the `writerId` it read with its own label (or omit the member)
  rather than echoing the previous writer's back.

**Privacy considerations:** unlike the encrypted `custom` object, `writerId` is
plaintext to the server, and a stable per-agent label reveals to the server
(and to any reader of the Collection) how many writing agents stand behind
one invoking [=did=], durably per revision -- information the server could
otherwise only approximate from traffic analysis. This is accepted as the
explicit price of pre-fetch echo suppression, which only a server-visible
field can provide, and it is bounded: the label reveals the multiplicity of
one principal's own writing agents, not any cross-party relationship (the
axis the [[[#key-epochs]]] recipients/grantees separation protects). And
because it is random, unkeyed, and clearable, it cannot be linked across
users or upgraded into a tracking identity. The field is optional
end-to-end: a client that does not want the server to hold even this simply
never declares one.

### Read Resource Metadata Operation {#read-resource-metadata-operation}

#### (HTTP API) GET `/space/{space_id}/{collection_id}/{resource_id}/meta`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for
    authorization, the request must either: be signed by the resource's or the
    space's [=controller=], or invoke a delegated capability that allows the
    [=GET=] action

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world/meta HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "contentType": "application/json",
  "size": 16,
  "createdAt": "2026-06-10T09:12:00Z",
  "updatedAt": "2026-06-12T13:25:00Z",
  "createdBy": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "writerId": "z6fVXHKn8PdQm2Rt",
  "custom": {
    "name": "Hello World greeting",
    "tags": { "project": "demo", "status": "draft" }
  }
}
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Resource does not exist, or the caller has
  missing or insufficient authorization; per [[[#error-handling]]] a Metadata
  object the caller is not authorized to read is indistinguishable from one
  that does not exist.
* [=unsupported-operation=] (501) -- the server does not implement the
  optional metadata endpoints.

### Update Resource Metadata Operation {#update-resource-metadata-operation}

A `PUT` to the `/meta` endpoint is a _full_ replacement of the Metadata
object's `custom` object: the stored `custom` object is replaced by the one
in the request body, so any user-writable property omitted from it is
cleared (and a request body with no `custom` property clears them all).

Two other top-level members are also writable, each with its own omission
behavior. A request MAY carry `epoch` to set the Resource's key-epoch
stamp; a request that omits it preserves the stored stamp, because the
stamp describes the content write, not the metadata write (see
[[[#key-epochs]]]). A request MAY carry `writerId` to declare the writing
agent's attribution label; a request that omits it clears the stored label
(see [[[#writer-attribution]]]). The label is cleared rather than
preserved because a metadata write is itself a revision, and on an
encrypted Collection it replaces the `custom` envelope wholesale, so
keeping a previous writer's label would misattribute it.

Server-managed properties are not affected by this operation; a server MUST
ignore any top-level properties other than `custom`, `epoch`, and `writerId`
present in the request body (so that a client may read the Metadata object,
modify it, and `PUT` it back without first stripping the server-managed
properties).

Unlike the [[[#update-or-create-by-id-resource-operation]]], a `PUT` to
`/meta` does not create anything: a Metadata object cannot exist apart
from its Resource, so a `PUT` to the `/meta` path of a nonexistent Resource
returns a [=not-found=] (404) error.

#### (HTTP API) PUT `/space/{space_id}/{collection_id}/{resource_id}/meta`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for
    authorization, the request must either: be signed by the resource's or the
    space's [=controller=], or invoke a delegated capability that allows the
    [=PUT=] action
* This operation is idempotent
* Returns a `204` success response

Example request:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world/meta HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "custom": {
    "name": "Hello World greeting",
    "tags": { "project": "demo", "status": "draft" }
  }
}
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Resource does not exist (this operation does not
  create one), or the caller has missing or insufficient authorization,
  per [[[#error-handling]]].
* [=invalid-request-body=] (400) -- the request body is not a JSON object,
  or it breaks a validation rule of the Resource Metadata object (see
  [[[#resource-metadata-data-model]]]), its top-level `epoch` member (see
  [[[#epoch-stamping-on-resources]]]), or its top-level `writerId` member
  (see [[[#writer-attribution]]]).
* [=unsupported-operation=] (501) -- the server does not implement the
  optional metadata endpoints.

## Chunked Resources {#chunked-resources}

<div class="note">
Chunked Resources are an OPTIONAL feature. Serving the chunk endpoints is a
requirement of [[PWS-EC]] conformance; a server that does not implement
[[PWS-EC]] MAY omit these endpoints entirely and remain conformant; see
[[[#scope-and-conformance-profiles]]].
</div>

A Resource MAY carry an ordered set of **chunks**: opaque byte sequences, each
addressed by a non-negative integer index under the Resource's reserved `chunks`
sub-path. Chunks let a client store a representation larger than a single request
(or a single encryption envelope) can carry, without the server ever parsing or
reassembling anything. The server treats each chunk exactly like a binary
Resource representation -- stored bytes plus a content type (see
[[[#content-types-and-representations]]]) -- and framing and reassembly
(including any client-side encryption) are entirely the client's concern. The
server never concatenates a Resource's chunks, and reading the parent Resource's
own content (see [[[#read-resource-operation]]]) returns only that content, not
its chunks; the chunk set is discovered and read through the endpoints below.

Chunks are the substrate the [[[#edv-over-pws-profile-v0-1]]] uses to store a
large or streamed encrypted document, but the mechanism itself is
scheme-agnostic: the bytes of a chunk are opaque to the server whether they are
plaintext, ciphertext, or anything else.

### The chunk address {#the-chunk-address}

A single chunk is addressed in **member form** (no trailing slash), and a
Resource's chunk set is listed in **container form** (trailing slash), following
the trailing-slash convention in [[[#reading-this-document]]]:

* `PUT` / `GET` / `HEAD` / `DELETE`
  `/space/{space_id}/{collection_id}/{resource_id}/chunks/{index}` -- store,
  read, head, and delete a single chunk.
* `GET` `/space/{space_id}/{collection_id}/{resource_id}/chunks/` -- list the
  Resource's chunks.

As elsewhere, a request to the non-canonical variant of either form is
redirected to the canonical one with `308 Permanent Redirect` (which, unlike a
`302`, requires the client to replay the same method and body): a `GET` of the
member-form `chunks` container without its trailing slash redirects to the
trailing-slash form, and a member `PUT` (or other member method) carrying a
trailing slash redirects to the no-slash form.

The `{index}` path segment MUST be a canonical non-negative decimal integer: a
single `0`, or a digit run with no leading zero, no sign, and no non-digit
characters (so `0`, `1`, `42` are valid; `01`, `+1`, `-1`, `1e3`, `1.0` are
not). A server MUST reject a non-canonical index with an [=invalid-id=] (`400`)
error. Requiring a canonical spelling keeps each chunk addressable at exactly
one URL.

### Store Chunk Operation {#store-chunk-operation}

#### (HTTP API) PUT `/space/{space_id}/{collection_id}/{resource_id}/chunks/{index}`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request must
    either: be signed by the space's [=controller=], or invoke a delegated
    capability that allows the [=PUT=] action, whose `invocationTarget` is the
    chunk's own full URL (see [[[#chunk-authorization]]]).
* Upserts the chunk at `{index}`: a write replaces any chunk already stored
  there. Indexes need not be written contiguously or in order.
* Returns a `204` success response carrying the chunk's `ETag`.

The request body is raw bytes under any `Content-Type`. The server MUST NOT
parse or validate a chunk body. This holds even for a Collection that declares
an `encryption` descriptor (see [[[#encryption-scheme-registry]]]): the
scheme's envelope validation applies to a Resource's own content, **not** to its
chunks, because the chunks of an encrypted stream are ciphertext fragments, not
envelope documents. The parent Resource MUST already exist; a `PUT` to a chunk
of a Resource that does not exist is rejected with [=not-found=] (`404`), so a
chunk can never be orphaned. The `Digest` request-body-integrity requirement
(see [[[#request-body-integrity-digest-header]]]) applies per request -- that
is, per chunk. The backend's `maxUploadBytes` cap and quota accounting apply to
a chunk write exactly as they do to a Resource write (see [[[#quotas]]]).

Each chunk carries its own strong `ETag` validator, independent of the parent
Resource's and of the other chunks'. The `If-Match` / `If-None-Match` write
preconditions of [[[#conditional-requests]]] apply per chunk, against that
validator.

Example request (storing chunk `0` as raw bytes):

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/backups/bigfile/chunks/0 HTTP/1.1
Host: example.com
Content-Type: application/octet-stream
Digest: mh=uEiCPO-qYr-z0GYV5F75-N1l8Rhjv4xIkKZsnbTZeZ7emSA
Authorization: ...

...raw chunk bytes...
```

Example success response:

```http
HTTP/1.1 204 No Content
ETag: "1"
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=invalid-id=] (400) -- the `{index}` segment is not a canonical non-negative
  decimal integer (see [[[#the-chunk-address]]]).
* [=not-found=] (404) -- the parent Resource (or its enclosing Space or
  Collection) does not exist, or the caller has missing or insufficient
  authorization; per [[[#error-handling]]] an under-authorized request is
  indistinguishable from a missing target.
* [=payload-too-large=] (413) -- the chunk exceeds the backend's `maxUploadBytes`
  constraint (see [[[#quotas]]]).
* [=quota-exceeded=] (507) -- the Collection's backend has no storage quota
  remaining (see [[[#quotas]]]).
* [=precondition-failed=] (412) -- a conditional write's `If-Match` /
  `If-None-Match` precondition evaluated false against the chunk's own `ETag`
  (see [[[#conditional-requests]]]).

### Read Chunk Operation {#read-chunk-operation}

#### (HTTP API) GET `/space/{space_id}/{collection_id}/{resource_id}/chunks/{index}`

A read returns the chunk's stored bytes, verbatim, with the content type they
were stored under, and the chunk's `ETag`. A `HEAD` on the same address returns
those same headers -- the response `Content-Type` and `Content-Length` are read
from the chunk's stored metadata, so the byte stream is never opened -- with no
body, mirroring the Resource `HEAD` variant in
[[[#content-types-and-representations]]]. Authorization for a chunk read (and
`HEAD`) is capability-or-policy and resolves at the parent Resource's access
level (see [[[#chunk-authorization]]]).

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/backups/bigfile/chunks/0 HTTP/1.1
Host: example.com
Authorization: ...
```

Example success response:

```http
HTTP/1.1 200 OK
Content-Type: application/octet-stream
ETag: "1"

...raw chunk bytes...
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=invalid-id=] (400) -- the `{index}` segment is not canonical (see
  [[[#the-chunk-address]]]).
* [=not-found=] (404) -- no chunk is stored at `{index}` (the parent Resource may
  exist but have no chunk there), or the caller has missing or insufficient
  authorization; per [[[#error-handling]]] the two are indistinguishable.

### Delete Chunk Operation {#delete-chunk-operation}

#### (HTTP API) DELETE `/space/{space_id}/{collection_id}/{resource_id}/chunks/{index}`

* Requires appropriate authorization on the same terms as the Store Chunk
  operation (capability-only against the chunk's own URL; see
  [[[#chunk-authorization]]]).
* Removes the chunk at `{index}` and returns a `204` success response.
* Accepts the `If-Match` precondition of [[[#conditional-requests]]] against the
  chunk's own `ETag`.

Unlike the [[[#delete-resource-operation]]], deleting a chunk is **not**
idempotent: a `DELETE` of an absent chunk is rejected with [=not-found=]
(`404`), not a `204`. This is deliberate -- a client reassembling a chunked
representation must be able to distinguish a chunk that is *gone* from one that
was *never written*, which an idempotent delete would erase.

Example request:

```http
DELETE /space/81246131-69a4-45ab-9bff-9c946b59cf2e/backups/bigfile/chunks/0 HTTP/1.1
Host: example.com
Authorization: ...
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=invalid-id=] (400) -- the `{index}` segment is not canonical.
* [=not-found=] (404) -- no chunk is stored at `{index}` (see above), or the
  caller has missing or insufficient authorization; per [[[#error-handling]]] the
  two are indistinguishable.
* [=precondition-failed=] (412) -- an `If-Match` precondition evaluated false
  against the chunk's `ETag`.

### List Chunks Operation {#list-chunks-operation}

#### (HTTP API) GET `/space/{space_id}/{collection_id}/{resource_id}/chunks/`

The container form lists a Resource's stored chunks. Because the server never
reassembles a chunked Resource, this listing is the discovery mechanism: a
reader learns the chunk set here -- how many chunks exist and each one's index,
size, and content type -- and then reads indexes `0` through `count - 1` itself.
Authorization is capability-or-policy against the `chunks/` container URL,
resolving at the parent Resource's access level (see [[[#chunk-authorization]]]).

The response is an `application/json` object:

* `resourceId` -- the parent Resource's id.
* `count` -- the number of stored chunks.
* `chunks` -- an array, in ascending `index` order, of one entry per stored
  chunk:
  * `index` -- the chunk's non-negative integer index.
  * `size` -- the length in bytes of the stored chunk.
  * `contentType` -- the content type the chunk was stored under.
  * `version` (optional) -- the chunk's monotonic version, the integer from
    which its strong `ETag` is derived (the `ETag` is this integer, quoted).
    Present when the server derives the chunk's `ETag` from an internal
    version counter; absent when it derives the `ETag` some other way, such
    as a content hash (see [[[#conditional-requests]]]).

The parent Resource MUST exist for its chunk container to: a listing under an
absent Resource is a [=not-found=] (`404`). A Resource that exists but has no
chunks lists as `count` `0` with an empty `chunks` array.

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/backups/bigfile/chunks/ HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "resourceId": "bigfile",
  "count": 3,
  "chunks": [
    { "index": 0, "size": 1048576, "contentType": "application/octet-stream", "version": 1 },
    { "index": 1, "size": 1048576, "contentType": "application/octet-stream", "version": 1 },
    { "index": 2, "size": 524288, "contentType": "application/octet-stream", "version": 1 }
  ]
}
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the parent Resource does not exist, or the caller has
  missing or insufficient authorization; per [[[#error-handling]]] the two are
  indistinguishable.

### Chunk lifecycle {#chunk-lifecycle}

A Resource's chunks are bound to the Resource. Deleting the parent Resource (see
[[[#delete-resource-operation]]]) MUST cascade-delete all of its chunks; there is
no way to leave chunks behind a deleted Resource. A Resource's chunks are carried
alongside its content by a Space export and restored by the matching import (the
`export` reserved segment; see [[[#reserved-path-segment-registry]]]).

Chunk writes and deletes are invisible to the `changes` query profile (see
[[[#query-profile-changes]]]): storing or deleting a chunk affects only the
chunk's own `ETag` validator and MUST NOT advance the parent Resource's
position in the feed, and the feed enumerates Resources only, never chunks. A
client replicating a chunked Resource MUST therefore finish the write by
updating the parent Resource's own content (its manifest; see
[[[#update-or-create-by-id-resource-operation]]]) after its chunks are stored:
that final Resource write is what surfaces the change to replicating readers.
A client that mutates a Resource purely through its chunks never appears on
the feed.

### Chunk authorization {#chunk-authorization}

Chunk operations use the same authorization model as every other operation in
this specification (see [[[#authorization]]]): writes (`PUT`, `DELETE`) are
capability-only, while reads (`GET`, `HEAD`, and the container listing) are
capability-or-policy. A chunk write's capability `invocationTarget` MUST be the
chunk's own full URL (member form), and the listing's the `chunks/` container
URL -- the same exact-match target rule that governs every PWS URL (see
[=target=]). For a read, the governing access-control [=policy=] is the parent
Resource's: a chunk exposes a fragment of the same content the Resource holds, so
whoever may read the Resource may read its chunks, and the maximum-privacy
[=not-found=] rule (see [[[#error-handling]]]) applies to a chunk exactly as to
the Resource.

## Linksets {#linksets}

<div class="note">
Linksets are an OPTIONAL extension. A server MAY omit this feature-detection
mechanism entirely and remain conformant; see [[[#scope-and-conformance-profiles]]].
</div>

Linksets (from [[RFC9264]]) are the feature detection and extension mechanism
for Space- and Collection-scoped facts; server-wide facts, and the versions the
server speaks, are in the [=service description=] (see
[[[#service-description]]]). Linksets can be discovered, via the `linkset`
property, from the following:

* The Space Metadata object, see [[[#space-metadata-data-model]]].
* The Collection Metadata object, see [[[#collection-metadata-data-model]]].

### Space Linkset {#space-linkset}

The space `linkset` resource (one of the [[[#space-level-reserved-endpoints]]]),
located at `/space/{space_id}/linkset` contains a set of links to auxiliary
resources and extension points:

* <dfn data-lt="policy relation" id="policy-rel"><code>policy</code> relation</dfn>
  (`https://w3id.org/pws#policy`) - A link to the
  `/space/{space_id}/policy` resource, which contains a set of links to access
  control policy documents.
* <dfn id="backends-available">`backends-available`</dfn>
  (`https://w3id.org/pws#backends-available`) - A link to the
  `/space/{space_id}/backends` "Backends Available" resource.
* <dfn data-lt="quotas relation" id="quotas-rel"><code>quotas</code> relation</dfn>
  (`https://w3id.org/pws#quotas`) -
  A link to the `/space/{space_id}/quotas` per-backend storage [=quota=] report
  (see [[[#quotas]]]).
* `service` relation ([[RFC5023]]) - A link to the [=service description=]
  (see [[[#discovering-the-service-description]]]).
* `/space/{space_id}/query` - Reserved for cross-space query operations.

Example space linkset resource request and response:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset HTTP/1.1
Accept: application/linkset+json
Authorization: ...
```

Response:

```http
HTTP/1.1 200 OK
Content-type: application/linkset+json

{
  "linkset": [
    {
      "anchor": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/",
      "https://w3id.org/pws#policy": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/policy",
          "type": "application/json"
        }
      ],
      "https://w3id.org/pws#backends-available": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/backends",
          "type": "application/json"
        }
      ],
      "https://w3id.org/pws#quotas": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/quotas",
          "type": "application/json"
        }
      ],
      "service": [
        {
          "href": "https://example.com/service",
          "type": "application/json"
        }
      ]
    }
  ]
}
```

### Collection Linkset {#collection-linkset}

The collection `linkset` resource (one of the [[[#collection-level-reserved-endpoints]]]),
located at `/space/{space_id}/{collection_id}/linkset` contains a set of links
to auxiliary resources and extension points:

* The [=policy relation=] (`https://w3id.org/pws#policy`) - A link to the
  `/space/{space_id}/{collection_id}/policy` resource, which contains a set of links
  to access control policy documents.
* <dfn data-lt="backend relation" id="backend-rel"><code>backend</code> relation</dfn>
  (`https://w3id.org/pws#backend`) - A link to the
  detailed `/space/{space_id}/{collection_id}/backend` "Backend Selected for this
  collection" resource.
* <dfn data-lt="quota relation" id="quota-rel"><code>quota</code> relation</dfn>
  (`https://w3id.org/pws#quota`) - A
  link to the `/space/{space_id}/{collection_id}/quota` storage [=quota=] report
  for this collection (see [[[#quotas]]]).
* `service` relation ([[RFC5023]]) - A link to the [=service description=]
  (see [[[#discovering-the-service-description]]]).
* `/space/{space_id}/{collection_id}/query` - (Optional) Query operations within
  a collection, discriminated by a request-body `profile` (see
  [[[#query-profile-registry]]]).

Example collection linkset resource request and response:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/linkset HTTP/1.1
Accept: application/linkset+json
Authorization: ...
```

Response:

```http
HTTP/1.1 200 OK
Content-type: application/linkset+json

{
  "linkset": [
    {
      "anchor": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/",
      "https://w3id.org/pws#policy": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/policy",
          "type": "application/json"
        }
      ],
      "https://w3id.org/pws#backend": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/backend",
          "type": "application/json"
        }
      ],
      "https://w3id.org/pws#quota": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/quota",
          "type": "application/json"
        }
      ],
      "service": [
        {
          "href": "https://example.com/service",
          "type": "application/json"
        }
      ]
    }
  ]
}
```

## Backends {#backends}

<div class="note">
This section defines an OPTIONAL extension. A server MAY skip pluggable backends
entirely and remain conformant; see [[[#scope-and-conformance-profiles]]].
</div>

Backends are an optional infrastructure concern that is orthogonal to the
hierarchical Spaces Repository > Space > Collection > Resource storage model.
They exist to serve advanced use cases that need fine-grained control over
storage configurations.

Available backends are registered on the Space level, as a combination of
server-side configuration and client-side "Bring Your Own Storage" registration.

For example, on the server configuration side, a given server might support several
backends -- a file system default backend, an EDV encrypted backend, and a
PostgreSQL database backend. And on the client side, a user might register an
external storage provider by connecting to their Dropbox account.

When a Collection is created, the client can optionally specify the preferred
backend for that Collection. If no preferred backend is specified, one is
assigned by the server (usually the `default` backend).

An implementer or client of a given server can omit the `backend` property when
creating a Collection. By default, if not specified, all Collections are
assigned the `default` backend. Omitting it on a later update of the
Collection's Metadata object keeps the backend already selected.

### Backend Data Model {#backend-data-model}

A Backend description object advertises a backend's identity, who operates it,
and its persistence, so that clients can select a suitable backend for each
Collection (and so that storage management UIs can present meaningful choices
to users). Quota constraints on a backend are reported separately, in the
Quota report (see [[[#quotas]]]).

Backend description properties:

* `id` (required) - A unique backend identifier (within a given space). The id
  `default` is conventionally used for the server-assigned default backend.
  See [[[#identifiers]]] for additional constraints.
* `name` (optional) - An arbitrary human-readable name for the backend. Does
  not have to be unique.
* `managedBy` (optional) - Who operates the backend. One of:
  - `server` - configured and operated server-side (for example, the server's
    filesystem or database backends). This is the default if unspecified.
  - `external` - a "Bring Your Own Storage" provider registered by the client
    (for example, a connected Dropbox account).
* `persistence` (optional) - Either `durable` (the engine stores data on
  persistent media, e.g. disk, and it survives a restart) or `volatile` (the
  engine stores data in memory, e.g. a RAM-backed cache tier, and data may not
  survive a restart). Defaults to `durable`. Note that this is a _technical_
  property of the storage engine, deliberately distinct from any _administrative_
  data-retention rules (see the editor's note on lifecycle configuration below).

Note that client-side encryption is deliberately not a backend property: an
encrypted document is opaque client-encrypted JSON that any document-capable
backend stores faithfully with no server cooperation, and whether a given
Collection is encrypted varies _per Collection_ on the very same backend.
Encryption is therefore a property of a Collection's data (signaled at the
Collection level and held in the client's keys), not a property of the
backend. Concretely, this signal is the Collection's optional `encryption`
descriptor (see [[[#collection-metadata-data-model]]]): a non-secret
declaration any authorized reader discovers from the Collection's Metadata
object, while the keys stay in the client.

Every backend on which a Collection may be created honors the conditional
writes of [[[#conditional-requests]]]; this holds regardless of what
persistence primitives the underlying storage engine offers, since the server
mediates every write.

<div class="ednote">
The schema of a backend's connection configuration (server-internal
connections vs OAuth-style setup flows for external providers) is not yet
specified.
</div>

### Space Backends Available {#space-backends-available}

The list of [=backends=] registered on a Space is discoverable via the
[=backends-available=] relation in the linkset (see [[[#space-linkset]]]).

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/backends HTTP/1.1
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json

[
  {
    "id": "default",
    "name": "Server Filesystem",
    "managedBy": "server",
    "persistence": "durable"
  },
  {
    "id": "dropbox",
    "name": "Dropbox",
    "managedBy": "external",
    "persistence": "durable"
  },
  {
    "id": "edv",
    "name": "Encrypted Data Vault",
    "managedBy": "server",
    "persistence": "durable"
  }
]
```

### Collection Backend Selected {#collection-backend-selected}

Each collection has an optional `backend` property that is set during its creation
(see [[[#collection-metadata-data-model]]]). If not specified, it is assumed to
have the `id` of `default`. The selected backend is discoverable via the
[=backend relation=] in the collection's linkset (see
[[[#collection-linkset]]]).

<div class="ednote">
**Replicas (planned generalization).** The single `backend` property is
expected to generalize to one or more _replicas_ per Collection, each replica
hosted on a backend, with the current single-backend configuration as the
degenerate one-replica case. (Collections do not "live" on a single backend;
backends are infrastructure endpoints, not a fourth tier of the storage
hierarchy.) That generalization will bring with it:

* Client-evaluated replica roles (`active` vs `available`) -- this
  specification will define the vocabulary for declaring replica topology;
  the logic for _selecting_ the active replica (based on latency, battery,
  network status) belongs to the client SDK.
* A sync-status vocabulary per replica (e.g. `synced`, `syncing`, `stale`);
  the exact state machine, including how "cold boots" of `volatile` backends
  are surfaced, is to be determined.
* Logical vs physical byte accounting in quota reports: the deduplicated size
  of the user's data vs the total bytes consumed across all replicas. These
  only diverge once replication exists, so the [[[#quotas]]] report currently
  carries plain per-backend usage.
* Concurrency control (`If-Match` ETags) for updates to replica topology,
  building on the per-Resource conditional-writes mechanism (see
  [[[#conditional-requests]]]) generalized to multi-replica writes.
</div>

<div class="ednote">
**Lifecycle configuration (administrative retention).** Retention rules such
as "delete guest data after one hour" (a time-to-live, a grace period, and an
expiry action) are expected to be configurable at the Space or Collection
level. Lifecycle configuration is deliberately distinct from both the
backend's `persistence` property (a technical attribute of the storage engine)
and from access-control [=policy=] (who may act on the data). The property
naming is to be determined, precisely to avoid overloading the term "policy".
</div>

## Quotas {#quotas}

<div class="note">
This section defines an OPTIONAL extension. A server MAY omit quota reporting and
enforcement entirely and remain conformant; see [[[#scope-and-conformance-profiles]]].
</div>

[=Quota=] reporting and enforcement are optional, and support is
backend-dependent. The quota API serves two distinct consumers:

* The **hot path**: applications checking whether they have room to write,
  served by a compact per-backend report.
* The **admin path**: storage-manager applications that need a per-collection
  usage breakdown, served by the same endpoint with an `include` query
  parameter.

Quota endpoints follow the same maximum-privacy invariant as the rest of this
specification (see [[[#error-handling]]]): a caller not authorized to read a
Space's quota report MUST receive a [=not-found=] (404) error, never a 403.

### (HTTP API) GET `/space/{space_id}/quotas`

Returns the storage report for a Space, grouped by backend, and is discoverable
via the [=quotas relation=] in the Space's linkset (see [[[#space-linkset]]]).
Each entry in the `backends` array carries:

* `id`, `name`, `managedBy` - identifying properties from the corresponding
  Backend description object (see [[[#backend-data-model]]]).
* `state` - the backend's current condition, one of: `ok`, `near-limit`,
  `over-quota`, or `unreachable`.
* `usageBytes` - total bytes consumed on this backend by this Space.
* `limit` - an object with `capacityBytes` and an `isUnlimited` boolean (when
  `isUnlimited` is `true`, `capacityBytes` MAY be omitted).
* `constraints` (optional) - operational constraints such as
  `maxUploadBytes`, the largest single upload the backend accepts.
* `restrictedActions` - an array of [=actions=] (uppercase HTTP verbs, the
  same vocabulary as the PWS Authorization Profile, see
  [[[#authorization-actions-and-the-root-capability]]]) currently unavailable
  on this backend. For example, a full backend reports `["POST", "PUT"]`
  while still permitting reads and deletes.
* `measuredAt` - when the usage numbers were measured. For `external`
  ("Bring Your Own Storage") backends, the server proxies the provider's
  reported numbers, which may be cached or stale; `measuredAt` lets clients
  judge freshness independently of the report's top-level `respondedAt`.

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/quotas HTTP/1.1
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "respondedAt": "2026-06-12T13:25:00Z",
  "backends": [
    {
      "id": "default",
      "name": "Server Filesystem",
      "managedBy": "server",
      "state": "ok",
      "usageBytes": 524288000,
      "limit": { "capacityBytes": 10737418240, "isUnlimited": false },
      "restrictedActions": [],
      "measuredAt": "2026-06-12T13:25:00Z"
    },
    {
      "id": "dropbox",
      "name": "Dropbox",
      "managedBy": "external",
      "state": "near-limit",
      "usageBytes": 314572800,
      "limit": { "capacityBytes": 367001600, "isUnlimited": false },
      "constraints": { "maxUploadBytes": 157286400 },
      "restrictedActions": [],
      "measuredAt": "2026-06-12T13:24:45Z"
    }
  ]
}
```

### (HTTP API) GET `/space/{space_id}/quotas?include=collections`

With the `include=collections` query parameter, each backend entry
additionally carries a `usageByCollection` array, giving storage-manager
applications a full breakdown in a single request while keeping the hot-path
payload lean:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "respondedAt": "2026-06-12T13:25:00Z",
  "backends": [
    {
      "id": "default",
      "name": "Server Filesystem",
      "managedBy": "server",
      "state": "ok",
      "usageBytes": 524288000,
      "limit": { "capacityBytes": 10737418240, "isUnlimited": false },
      "restrictedActions": [],
      "measuredAt": "2026-06-12T13:25:00Z",
      "usageByCollection": [
        { "id": "credentials", "usageBytes": 419430400 },
        { "id": "inbox", "usageBytes": 104857600 }
      ]
    }
  ]
}
```

### (HTTP API) GET `/space/{space_id}/{collection_id}/quota`

Returns the storage report for a single Collection, scoped to that
Collection's backend (the entry has the same shape as a `backends` array
entry, with `usageBytes` reflecting only this Collection's consumption). It is
discoverable via the [=quota relation=] in the Collection's linkset (see
[[[#collection-linkset]]]).

Not all backends support per-collection accounting. Where unsupported, the
server returns an [=unsupported-operation=] (501) error.

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Space or Collection does not exist, or the caller
  is not authorized to read the quota report.
* [=unsupported-operation=] (501) -- the Collection's backend does not support
  per-collection quota accounting.

<section class="appendix">

## Pagination {#pagination}

This appendix is normative. It defines the full pagination profile summarized
in [[[#paginated-list-responses]]].

The list operations -- [[[#list-spaces-operation]]], [[[#list-all-collections-operation]]],
and [[[#list-collection-operation]]] -- return a collection of items in the
common envelope (`url`, `totalItems`, `items`). A Space or Collection may hold
more items than is practical to return in a single response, so servers MAY
paginate these responses, returning one page of items at a time. Pagination
is OPTIONAL: a server that returns every item in a single response (as the
examples in those sections show) is conformant, and a client MUST be prepared
for either behavior.

This profile uses cursor-based (also called "keyset") pagination rather than
numeric offsets. A cursor identifies a position in a stably ordered result set,
so paging stays correct and cheap even as items are concurrently added or
removed, and at any depth into a large collection.

### Ordering

A paginated list operation MUST return items in a stable total order: an
ordering that is deterministic and in which no two items compare equal. The
default order is ascending by item `id` (which is unique within its parent, see
[[[#identifiers]]]). A server MAY offer additional orderings, but every order it
supports for pagination MUST be stable and total -- if the primary sort key is
not unique (for example, a timestamp), the server MUST break ties on a unique key
(such as `id`) so that the combined order is total. This stable order is what a
cursor seeks within; an unstable order cannot be paginated correctly.

### Requesting a page

A client requests pagination with two OPTIONAL query parameters:

* `limit` - the maximum number of items the client wants in the page, a
  positive integer. A server applies its own default when `limit` is absent, and
  MAY clamp an oversized `limit` down to a server maximum rather than rejecting
  the request. A server MAY return fewer items than `limit` (including zero) and
  still indicate more pages follow; clients MUST NOT treat a short page as the
  end of the list (see `next`, below).
* `cursor` - an opaque token, obtained from a prior page's `next` value (see
  below), naming the position to continue from. A client MUST treat the cursor as
  opaque: it MUST NOT construct, parse, or modify it, and MUST NOT carry a cursor
  from one collection, ordering, or server to another. A server encodes whatever
  it needs into the cursor to resume the ordered scan (typically the sort-key
  value of the last item already returned).

The first page is requested with no `cursor` (a bare `limit`, or neither
parameter).

### The paginated response

A paginated response carries the usual envelope, plus a `next` member when more
items may follow:

* `next` - a URL the client dereferences (with the same authorization) to
  retrieve the following page. The server bakes the appropriate `cursor` (and
  any `limit`) into this URL, so the client follows it without constructing query
  parameters itself. `next` is present if and only if more items may follow;
  its absence marks the last page. This presence/absence is the authoritative
  end-of-list signal.
* `totalItems` - when present, the total number of items in the entire
  collection, not the number in the current page (the page count is simply the
  length of `items`). Computing an exact total can be expensive at scale, so a
  paginating server MAY omit `totalItems`. A client MUST NOT infer the number of
  pages, or whether more pages exist, from `totalItems`; only `next` is
  authoritative.

A client consumes a multi-page list by following `next` from each response until
a response omits it. A server SHOULD ensure that an item present throughout the
traversal appears exactly once across the pages; an item added or removed
concurrently with the traversal MAY or MAY NOT appear. Snapshot consistency
(a paginated traversal observing a single point in time) is permitted but not
required. A server MAY encode a snapshot identifier into the cursor to provide
such consistency.

A `cursor` that is malformed, or that a server can no longer honor (for example,
an expired snapshot), is rejected with an [=invalid-cursor=] (`400`) error. As
with other request-validation failures, a server MUST verify the caller's
authorization for the target before validating the cursor, so the error is
only ever observed by a caller already authorized to list that target; an
under-authorized caller receives the merged [=not-found=] (`404`) instead, per
[[[#error-handling]]].

### Pagination parameters and authorization

A capability authorizes a list [=target=] independent of which page is being
read: a capability that authorizes listing a Space, Collection, or Spaces
Repository authorizes retrieval of *every* page of that list. The `limit` and
`cursor` parameters select a page within an already-authorized target; they do
not narrow, widen, or otherwise change the [=target=] a capability must match
(see [[[#root-capability]]]). A server MUST NOT require a distinct capability per
page.

</section>

<section class="appendix normative">

## Space Controller DID Method Registry {#space-controller-did-method-registry}

<div class="note">
Every method in this registry other than `did:key` is OPTIONAL to support. A
server that supports only `did:key` Space controllers is conformant; a server
that does support a registered method MUST follow that method's profile here.
</div>

This registry lists the DID methods a Space `controller` (and, by the
[[[#current-key-set-rule]]], any DID signing in a capability chain) may use,
and the verification a [=server=] performs for each. See
[[[#space-controller-and-the-root-of-trust]]] for the method-neutral rules:
the current-key-set rule, refusal of unsupported methods, and the
resolve-before-store rule for a proposed controller.

| Method                                                   | Support  | Kind         | Self-hosted history layout                                  | Profile                                   |
|----------------------------------------------------------|----------|--------------|-------------------------------------------------------------|-------------------------------------------|
| [`did:key`](https://w3c-ccg.github.io/did-key-spec/)     | REQUIRED | static       | n/a                                                         | [[[#did-key-controller-profile]]]         |
| [`did:webvh`](https://identity.foundation/didwebvh/)     | OPTIONAL | verified log | `/space/{space_id}/{collection_id}/did.jsonl`               | [[[#did-webvh-controller-profile]]]       |

### Verified-log DID methods {#verified-log-did-methods}

A verified-log DID method is one whose DID document is derived from a
self-certifying history (a log of entries, each bound to the one before it
and to the DID's own identifier) rather than from the identifier alone. This
subsection states the rules common to every such method in the registry; a
method's profile adds the log layout and the verification its own
specification requires.

A server that supports a verified-log method MUST resolve a DID of that
method by reading its history and fully verifying it, as the method's
specification defines, before using the resulting DID document. The history
is verified rather than trusted. Once a Space names such a DID as controller,
the writes that extend that history may themselves be authorized by the very
document being resolved, so an unverified history would let whoever can write
it rewrite the Space's root of trust.

#### Self-hosted histories {#self-hosted-histories}

The history of a verified-log controller MAY be stored on the PWS server
itself, as one or more Resources in a Collection of a Space on that server.
The method's standard DID-to-URL transformation then lands on
`/space/{space_id}/{collection_id}/` followed by the method's history file
name(s); each profile gives the exact layout. Such a controller is
self-hosted. The Space hosting the history need not be the Space the
controller controls, since the DID names the history's own Space and
Collection.

A server that supports a verified-log method MUST resolve self-hosted DIDs of
that method by reading the history out of its own storage. A network fetch of
its own URL is not a substitute: it would pass through the history
Collection's access-control policy and through any intermediary in front of
the server. Resolution reads the history regardless of the Collection's
policy, so a capability-gated history still resolves for authorization while
remaining unreadable to clients that hold no capability for it. A server MAY
accept only self-hosted controllers of a method and refuse every other DID of
that method (one whose history lives on another host) with
[=invalid-request-body=] (400), the same way it refuses an unsupported
method.

Hosting a history under a Space's path is not an endorsement by that Space's
controller: any party holding a write grant there can store a resolvable
history. A verified-log DID is self-certifying through its own identifier and
history. It acquires authority in PWS only by being referenced, as a Space's
stored `controller` or as the `controller` of a delegated capability, and not
by where its history is stored.

A server MAY cache a resolved document. It MUST drop the cached entry
whenever the history the document was resolved from may have changed, so
that a write to a self-hosted history (such as one adding or removing a key)
is reflected by the next verification that resolves that DID.

### `did:key` controller profile {#did-key-controller-profile}

The mandatory floor. A conformant server MUST accept a
[`did:key`](https://w3c-ccg.github.io/did-key-spec/) using the Multikey
encoding of `Ed25519` elliptic curve keys, as specified in the
[Multikey section of the CID spec](https://www.w3.org/TR/cid-1.0/#Multikey),
as a Space `controller`. The DID document is derived from the identifier
alone; there is nothing to resolve or verify beyond the key encoding, and the
document never changes.

### `did:webvh` controller profile {#did-webvh-controller-profile}

A verified-log method; the rules of [[[#verified-log-did-methods]]] apply.

History layout. The history is the method's `did.jsonl` log. Self-hosted,
the DID `did:webvh:{scid}:{host}:space:{space_id}:{collection_id}` resolves
from the Resource at `https://{host}/space/{space_id}/{collection_id}/did.jsonl`
under the method's standard DID-to-HTTPS transformation. The `{collection_id}`
is any Collection id; PWS Collection ids are restricted to characters the
method's path encoding leaves untouched, so the mapping is direct.

Verification. Before using the resolved document the server MUST verify the
log as the method's specification defines: the SCID is checked against the
log's first entry, the hash chain of entries is verified, pre-rotation
commitments (`nextKeyHashes`) are enforced, and each entry's proof is
verified against the update keys authorized by the entry before it.

</section>

<section class="appendix">

## Reserved Path Segment Registry {#reserved-path-segment-registry}

This appendix is normative.

### Space-level reserved endpoints {#space-level-reserved-endpoints}

The following path segments represent reserved API endpoints for [[[#spaces]]]
level operations. Usually, the path segment following the `/space/{space_id}/` prefix
is the id of a Collection. The list of reserved endpoints below means that
collections `id`s MUST NOT collide with the corresponding reserved segments.

| Reserved API Endpoint           | Reserved segment | Purpose                                |
|---------------------------------|------------------|----------------------------------------|
| `/space/{space_id}/meta`        | `meta`           | Space metadata; see [[[#space-metadata-data-model]]] |
| `/space/{space_id}/policy`      | `policy`         | Access control policy                  |
| `/space/{space_id}/backends`    | `backends`       | Storage backends available             |
| `/space/{space_id}/collections` | `collections`    | Retired in v0.5. Listing and creating collections moved to the Space URL (see [[[#list-all-collections-operation]]]); a server MAY answer this path with a `308 Permanent Redirect` to it. The segment stays reserved |
| `/space/{space_id}/export`      | `export`         | Export (download) space contents       |
| `/space/{space_id}/import`      | `import`         | Import (upload) a space export archive |
| `/space/{space_id}/linkset`     | `linkset`        | Links to auxiliary resources           |
| `/space/{space_id}/query`       | `query`          | Reserved for cross-collection queries  |
| `/space/{space_id}/quotas`      | `quotas`         | Reserved for per-backend quota report  |

If a client attempts to create a collection with an `id` that collides with a
reserved segment list above, the server MUST return a 409 Conflict error.

### Collection-level reserved endpoints {#collection-level-reserved-endpoints}

The following path segments represent reserved API endpoints for [[[#collections]]]
level operations. Usually, the path segment following the
`/space/{space_id}/{collection_id}/` prefix is the id of a Resource. The list of
reserved endpoints below means that resource `id`s MUST NOT collide with the
corresponding reserved segments.

| Reserved API Endpoint                        | Reserved segment | Purpose                             |
|----------------------------------------------|------------------|-------------------------------------|
| `/space/{space_id}/{collection_id}/policy`   | `policy`         | Access control policy               |
| `/space/{space_id}/{collection_id}/backend`  | `backend`        | Storage backend selected            |
| `/space/{space_id}/{collection_id}/linkset`  | `linkset`        | Links to auxiliary resources        |
| `/space/{space_id}/{collection_id}/meta`     | `meta`           | Collection metadata: its description together with the user's annotations; see [[[#collection-metadata-data-model]]] |
| `/space/{space_id}/{collection_id}/meta/log` | `meta`           | Governing history log, a sub-resource under the `meta` segment; see [[[#collection-governing-history-log]]] |
| `/space/{space_id}/{collection_id}/query`    | `query`          | Query resources within a collection (see [[[#query-profile-registry]]]) |
| `/space/{space_id}/{collection_id}/quota`    | `quota`          | Storage quota report for collection |

If a client attempts to create a resource with an `id` that collides with a
reserved segment list above, the server MUST return a 409 Conflict error.

### Resource-level reserved endpoints {#resource-level-reserved-endpoints}

The following path segments represent reserved API endpoints for Resource
level operations.

| Reserved API Endpoint                                  | Reserved segment | Purpose                                              |
|--------------------------------------------------------|------------------|------------------------------------------------------|
| `/space/{space_id}/{collection_id}/{resource_id}/meta`   | `meta`           | Resource metadata (server-managed and user-writable) |
| `/space/{space_id}/{collection_id}/{resource_id}/chunks` | `chunks`         | Chunk addressing for a chunked Resource (see [[[#chunked-resources]]]) |

Unlike the Space- and Collection-level reserved segments, which occupy the id
position one level down and so constrain Collection and Resource id choice,
these sit *below* the Resource level -- they qualify a `{resource_id}` -- and
impose no constraint of their own on id choice at any level. `meta` appears in
all three tables, holding the description of the thing it hangs under (see
[[[#reading-this-document]]]), but it constrains id choice only where it
occupies an id position: the Space-level row makes `meta` a reserved Collection
id and the Collection-level row makes it a reserved Resource id, while the
Resource-level `/meta` and `/chunks` segments sit one level lower and shadow
nothing -- a Resource whose own id is `chunks` is unaffected.

### Methods at Reserved Endpoints {#methods-at-reserved-endpoints}

This subsection governs requests to the reserved endpoints listed in the
three tables above.

A request to a reserved endpoint with a method the server does not implement
at that endpoint MUST be answered with `405 Method Not Allowed`. The response
carries an `Allow` header naming the methods the server does implement there,
as [[RFC9110]] requires of a 405.

An endpoint can belong to an OPTIONAL group the server does not implement at
all. Where that is so, the group's own rule governs the request instead,
typically an [=unsupported-operation=] (501) error (see for example
[[[#collection-metadata-data-model]]] and [[[#resource-metadata-data-model]]]).

`HEAD` counts as implemented wherever `GET` is. This subsection does not
govern `OPTIONS`.

The `308` redirect this registry permits for the retired `collections`
segment (see [[[#space-level-reserved-endpoints]]]) is the server's
implementation of the redirected methods. Those methods are not refused.

The server MUST answer the 405 without regard to whether the Space,
Collection, or Resource named in the path exists. So the answer reveals
nothing the [=not-found=] rule protects (see [[[#error-handling]]]). A server
MUST NOT treat such a request as an operation on a Collection or Resource
whose id is the reserved segment.

This refusal has no entry in the Error Type Registry. Its meaning is the HTTP
status alone. A problem body, where a server sends one, uses [[RFC9457]]'s
`about:blank` type.

</section>

<section class="appendix">

## Encryption Scheme Registry {#encryption-scheme-registry}

This appendix is normative.

<div class="note">
This registry defines values used by an OPTIONAL feature (client-side encryption):
a server MAY support no encryption schemes and remain conformant; see
[[[#scope-and-conformance-profiles]]]. Optionality is orthogonal to normativity --
a server that does implement encrypted Collections MUST follow the wire formats
catalogued here.
</div>

A Collection's optional `encryption` descriptor (see
[[[#collection-metadata-data-model]]]) names a client-side encryption `scheme`
and a `version` of that scheme's wire format -- a positive integer, starting at
`1` for each scheme and incremented
per registered revision (an absent `version` means `1`). This registry maps
each `scheme`/`version` pair to the wire format the
server can expect for Resources in such a Collection, so
that a [=server=] can hold the [=collection=]'s fail-closed guarantee
*structurally*, by validating the shape of what is written, rather than
relying on every client to encrypt correctly. The server never holds key
material and never decrypts; it validates only the non-secret envelope
structure, so this enforcement neither requires nor weakens confidentiality.

| `scheme` | `version` | Media type         | Envelope profile                                                                                                                                                                                                                                                                                                                          | Reference                                                      |
|----------|-----------|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| `edv`    | `1`       | `application/json` | An [Encrypted Data Vault](https://identity.foundation/edv-spec/) **Encrypted Document**: a JSON object whose `jwe` member is a JWE in JSON Serialization ([[RFC7516]] §7.2) -- a JSON object carrying at least a `ciphertext` member and either a `recipients` array (general serialization) or a top-level `encrypted_key`/`protected` (flattened serialization). The document MAY also carry EDV bookkeeping members (`id`, `sequence`, `indexed`); these are opaque to the server. | [Encrypted Data Vaults](https://identity.foundation/edv-spec/) |

### Server-side write validation

A [=server=] that recognizes the `scheme`/`version` pair declared by a
Collection's `encryption` descriptor MUST validate the body of every Resource
content write ([=POST=] or [=PUT=]) into that Collection against that pair's
envelope profile, and MUST reject a non-conforming body -- or a body sent under a
`Content-Type` other than the scheme's registered media type -- with an
[=encryption-scheme-mismatch=] error.

This is the structural fail-closed guarantee: a client that has not encrypted
the body, whether through a bug or an omission, cannot store server-visible
plaintext in an encrypted Collection. The check is purely structural; a server
MUST NOT attempt decryption and MUST NOT inspect the envelope's contents.

The rule applies to a **Resource's stored representation** and, on an encrypted
Collection, to the user-writable `custom` object of its
[[[#resource-metadata-data-model]]] (see the encrypted-Collection note there).
It does not apply to the rest of the server-managed API documents -- Collection
Descriptions, the server-managed top-level Metadata properties, [=policy=]
documents, linksets -- which remain `application/json` regardless of the
Collection's encryption status. For the Metadata object specifically, the
document itself stays a plaintext `application/json` object (so the server keeps
its top-level `contentType`, `size`, and timestamps); only the `custom`
sub-value MUST be a conforming envelope on an encrypted Collection, validated the
same fail-closed way (an [=encryption-scheme-mismatch=] on non-conformance). An
empty or omitted `custom` is a clearing write and is not validated.

As with [=id-conflict=], a server MUST verify the caller's authorization
before validating the envelope, so that [=encryption-scheme-mismatch=] is
observable only to callers already authorized to write at that target; an
under-authorized caller receives the merged [=not-found=] instead, and learns
nothing about the Collection.

### Accepting a descriptor only when it can be enforced

When a Collection create or update operation declares an `encryption` descriptor (see
[[[#update-or-create-by-id-collection-operation]]]), a server SHOULD reject a
`scheme` -- or a `version` of a recognized scheme -- that it does not recognize
with an [=unsupported-encryption-scheme=] error, rather than storing a descriptor it
cannot enforce. This ensures that every descriptor
a server accepts is one it validates on write: "this Collection is declared
encrypted" then structurally implies "plaintext writes to it are rejected here,"
closing the gap that a silently unenforced descriptor would reopen.

A server MAY instead choose to store descriptors for schemes it does not enforce
(treating the descriptor as fully opaque, per
[[[#collection-metadata-data-model]]]), but such a server MUST document that it
provides no server-side fail-closed guarantee for those Collections, leaving the
guarantee entirely to clients.

### Validation profile

<div class="informative">

A server MAY implement the `edv` `1` envelope profile with a JSON Schema equivalent
to the following. The outer object MUST carry a `jwe` member; only the
`jwe`'s structural members are checked; their values are opaque ciphertext and
are not interpreted. A plaintext object under `application/json` (one with no
valid `jwe`) fails this profile and is rejected with an
[=encryption-scheme-mismatch=], preserving the fail-closed guarantee.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["jwe"],
  "properties": {
    "jwe": {
      "type": "object",
      "required": ["ciphertext"],
      "properties": {
        "protected": { "type": "string" },
        "iv": { "type": "string" },
        "ciphertext": { "type": "string" },
        "tag": { "type": "string" },
        "encrypted_key": { "type": "string" },
        "recipients": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "header": { "type": "object" },
              "encrypted_key": { "type": "string" }
            }
          }
        }
      },
      "anyOf": [
        { "required": ["recipients"] },
        { "required": ["encrypted_key"] },
        { "required": ["protected"] }
      ]
    }
  }
}
```

</div>

### Key Epochs {#key-epochs}

A key epoch is one generation of a Collection's encryption key. Rather than
encrypting every Resource under a single long-lived key, writers encrypt each
Resource under whichever epoch was current when it was written; the
Collection's `encryption` descriptor carries the full, append-only roster of
epochs, so the roster as a whole reads as the Collection's access history. The
full client-side construction is specified in [[PWS-EC]]; this section defines
only what the [=server=] stores and validates.

An encrypted Collection with a single shared key set has no cryptographic
notion of removing a reader: revoking the reader's capability stops the server
serving it ciphertext, but the reader still holds the key. Key epochs make
removal meaningful by separating two axes:

* **pull** -- fetching ciphertext. Governed by capabilities, enforced by the
  server at request time; revocation is immediate and total.
* **read** -- decrypting it. Governed by possession of an epoch key, enforced
  by mathematics; revocation is prospective only.

Removing a reader means doing **both**: revoke its capability AND rotate the
epoch. Neither alone suffices.

#### Epoch data model {#epoch-data-model}

A Collection's `encryption` descriptor MAY carry the two key-epoch members
below. It MAY also carry the `hmac` member of [[[#blinding-key-member]]],
where its clients use the `blinded-index` query profile.

* `epochs` - An array of epoch objects, each with:
  * `id` - A unique, opaque epoch identifier (a non-empty string). The
    reference client uses the `did:key` identifier of the epoch's public
    key-agreement key, so that a stored envelope's JWE recipient `kid` names
    its epoch directly, but servers MUST treat the value as opaque.
  * `recipients` - A non-empty array of wrapped-key entries, one per reader.
    Each entry reuses the JWE General Serialization `recipients` entry shape
    verbatim ([[RFC7516]] section 7.2): a `header` object with at least a
    non-empty string `kid` (the reader's key-agreement key identifier) and
    `alg` (e.g. `ECDH-ES+A256KW`), plus the wrapped epoch key as a string
    `encrypted_key`. This is the same shape the `edv` envelope profile already
    validates -- one wire vocabulary for "a key wrapped to a recipient".
* `currentEpoch` - The `id` of the epoch new writes encrypt under. MUST name
  an entry in `epochs`.

Nothing secret appears in these fields: recipient entries hold public key
identifiers and wrapped-key ciphertext only. The server never holds key
material, never unwraps, and never verifies that a wrapped key is correct --
that is unverifiable without the keys, by construction.

Example descriptor:

```json
{
  "scheme": "edv",
  "currentEpoch": "did:key:z6LSepoch2...#z6LSepoch2...",
  "epochs": [
    {
      "id": "did:key:z6LSepoch1...#z6LSepoch1...",
      "recipients": [
        {
          "header": {
            "kid": "did:key:z6MkappA...#z6LSkakA...",
            "alg": "ECDH-ES+A256KW",
            "epk": { "kty": "OKP", "crv": "X25519", "x": "..." },
            "apu": "...",
            "apv": "..."
          },
          "encrypted_key": "base64url..."
        }
      ]
    },
    {
      "id": "did:key:z6LSepoch2...#z6LSepoch2...",
      "recipients": [ "..." ]
    }
  ]
}
```

#### The blinding-key member {#blinding-key-member}

A Collection's `encryption` descriptor MAY also carry an `hmac` member: the
public reference to the Collection's blinding key, the HMAC key under which a
client blinds the attribute names and values it stores in an envelope's
`indexed` member and sends in a `blinded-index` query (see
[[[#query-profile-blinded-index]]]). The member is a JSON object:

* `id` - A non-empty string identifying the blinding key, opaque to the
  server. It is the value an envelope's `indexed[].hmac.id` carries and the
  value a `blinded-index` query names as its `index`.
* `type` - A non-empty string naming the key type (the [[PWS-EC]] profile uses
  `Sha256HmacKey2019`). Servers MUST treat the value as opaque.
* `recipients` - A non-empty array of wrapped-key entries, one per reader, in
  exactly the `recipients` entry shape of an epoch (see
  [[[#epoch-data-model]]]). Each entry carries the blinding secret wrapped to
  one reader's key-agreement key, so one wrap construction serves epoch keys
  and the blinding key alike.

As with the epoch members, nothing secret appears here: the blinding secret
travels only as per-reader wrapped ciphertext, and the server never unwraps
it. How the key is minted, which readers hold it, and how its `recipients` are
maintained alongside the epochs' are client-side matters defined by
[[PWS-EC]]. One property of the member does matter to the server: the
blinding key is permanent. Every blinded token in the Collection is computed
under it, so replacing or dropping it would orphan every blinded index at
once. The server therefore treats a stored `hmac` member as immutable in its
`id` and `type` and never removable (see the server validation below). As
with the epoch members, this validation is part of recognizing the `edv`
scheme: a server that does not support encrypted Collections, or that stores
descriptors it does not enforce (see [[[#encryption-scheme-registry]]]),
treats `hmac` as an opaque member. The
[[PWS-EC]] profile goes further and installs the key when the Collection is
provisioned, since envelopes written before it carry no tokens. This
specification leaves that timing rule to clients and does not require the
server to enforce it.

Example descriptor carrying a blinding key (epoch members elided):

```json
{
  "scheme": "edv",
  "hmac": {
    "id": "urn:uuid:6d4a2c3e-9b1f-4c8e-a0d2-7f5e1b3c9a44",
    "type": "Sha256HmacKey2019",
    "recipients": [
      {
        "header": {
          "kid": "did:key:z6MkappA...#z6LSkakA...",
          "alg": "ECDH-ES+A256KW",
          "epk": { "kty": "OKP", "crv": "X25519", "x": "..." },
          "apu": "...",
          "apv": "..."
        },
        "encrypted_key": "base64url..."
      }
    ]
  },
  "currentEpoch": "did:key:z6LSepoch1...#z6LSepoch1...",
  "epochs": [ "..." ]
}
```

#### Server validation {#key-epoch-server-validation}

A server that recognizes the `edv` scheme MUST validate the key-epoch members
when a Collection create or update supplies them, rejecting a violation with an
[=invalid-request-body=] error (and a JSON pointer to the offending member):

* `epochs` and `currentEpoch` MUST appear together or not at all.
* `epochs` MUST be a non-empty array; each entry MUST have a non-empty string
  `id`, unique across the array, and a non-empty `recipients` array.
* Each `recipients` entry MUST have a `header` object with non-empty string
  `kid` and `alg` members, plus a string `encrypted_key`.
* `currentEpoch` MUST name an `id` present in `epochs`.

The same server MUST validate the `hmac` member (see
[[[#blinding-key-member]]]) when a create or update supplies it, on the same
terms:

* `hmac` MUST be an object with non-empty string `id` and `type` members and a
  non-empty `recipients` array; each `recipients` entry MUST have the shape
  required of an epoch's entries above (a `header` object with non-empty
  string `kid` and `alg` members, plus a string `encrypted_key`).

A server that does not recognize the scheme applies none of these checks; see
[[[#encryption-scheme-registry]]] for what such a server does with the
descriptor.

On an [update](#update-or-create-by-id-collection-operation) of a descriptor
that already carries epochs, two server-side invariants also apply. These
protect the Collection's own readers from client bugs -- a dropped epoch
strands every Resource stamped with it -- and are enforceable without the
server reading a single byte of what the epochs protect:

* **`epochs` is append-only.** Every previously stored epoch `id` MUST still be
  present. (Entries within an existing epoch's `recipients` MAY change: adding
  a reader wraps existing epoch keys to it.)
* **`currentEpoch` never moves backwards.** The updated `currentEpoch` MUST
  either equal the stored one or name an epoch `id` that was not previously
  present (a newly appended epoch). This formulation is independent of array
  order.

A third invariant covers the `hmac` member, on an update of a descriptor that
already carries one:

* **`hmac` is permanent.** The member MUST remain present, with its `id` and
  `type` unchanged. Its `recipients` entries MAY change (a reader being added
  or removed). Changing either value, or removing the member, would orphan
  every blinded index in the Collection, just as a `scheme` change would
  corrupt its Resources. A violation is therefore rejected with the same
  [=encryption-immutable=] error (see [[[#collection-metadata-data-model]]]).
  Introducing `hmac` on a stored descriptor that lacks it is not a server-side
  violation; whether a client may do so is a matter for the client-side profile
  (see
  [[[#blinding-key-member]]]).

On a Collection whose descriptor is governed by its history log (see
[[[#collection-governing-history-log]]]), the descriptor is not supplied on
an update; it is the `state` of the log's head entry. The shape validation
above then runs on the head `state` of every log write, and the invariants
run on each append against the prior head's `state`, with the same errors.

Beyond this shape validation and these invariants the server MUST NOT
interpret the members. In particular it MUST NOT attempt to check that the
`recipients` of an epoch or of `hmac` match any set of capability grantees --
the two axes are deliberately independent, and keeping them in sync is the
client's job.

#### Epoch stamping on Resources {#epoch-stamping-on-resources}

A reader must know which epoch key to unwrap before attempting decryption.
The writer therefore declares the epoch a Resource was encrypted under, and the
server stores and serves that declaration:

* A Resource content write ([=POST=] or [=PUT=]) MAY carry a `Key-Epoch`
  request header whose value is the epoch `id` (a non-empty string; a present
  but empty or malformed value is an [=invalid-request-body=] error). The
  server MUST store the value verbatim as the Resource Metadata `epoch`
  property. A content write without the header MUST clear any stored
  stamp: the new representation's epoch is unknown, and a stale stamp is worse
  than none. The value is client-declared, advisory metadata: the server MUST
  NOT validate it against the descriptor's `epochs` (a write may race a rotation)
  and MUST NOT compute it.
* An Update Resource Metadata request MAY carry a top-level `epoch` member (a
  sibling of `custom`, and like the header a non-empty string). Unlike
  `custom`, which is a full replacement, an omitted `epoch` PRESERVES the
  stored value -- the stamp describes the content write, not the metadata
  write. It MUST NOT live inside `custom`: on an encrypted Collection `custom`
  is the opaque envelope and is replaced wholesale by every metadata write.
* An [[[#update-or-create-by-id-collection-operation]]] request MAY likewise
  carry a top-level `epoch` member, stamping the epoch the Collection Metadata
  object's own `custom` envelope was encrypted under. Its omission rule is
  inverted: an omitted
  `epoch` CLEARS the stored value, because at that level the stamp describes
  the `custom` envelope itself, which every metadata write replaces
  wholesale. See [[[#collection-metadata-data-model]]].
* The server MUST return the stored stamp as the `epoch` property of the
  Resource Metadata object, on each item of a List Collection response, and on
  each document of the [`changes` query profile](#query-profile-changes) -- so a
reader walking a listing
  or replicating from the feed can select its epoch key without a per-Resource
  metadata fetch.

#### Client procedures

<div class="informative">

The reference construction (implemented by `@interop/was-client` and
normatively specified in [[PWS-EC]]); other constructions are conformant at
this specification's level so long as stored envelopes keep satisfying the
[`edv` envelope profile](#encryption-scheme-registry):

* **A collection's metadata carries its epoch roster from creation.** The
  first epoch is installed at provision time by a
  [create-if-absent](#conditional-requests) write
  (an existing roster, including a concurrent provisioner's is adopted,
  not overwritten), before the collection's first content write. There is
  no epoch-less era: a client of this construction refuses fail-closed to
  build a cipher from an `edv` descriptor without epochs, and never seals
  an envelope directly to a reader's own key-agreement key.
* An **epoch key** is a fresh 32-byte secret used as the seed of an X25519
  key-agreement key pair, freshly generated per epoch. The epoch `id` is the
  `did:key` identifier of the epoch public key, so any standard `did:key`
  resolver that supports X25519 type `did:key` DIDs can resolve the JWE
  recipient `kid` of a stored envelope, and the `kid` itself names the epoch.
* **Resources are encrypted with a fresh per-resource content encryption key
  wrapped to the epoch key** (`ECDH-ES+A256KW`, the epoch key pair as the JWE's
  sole recipient). The stored envelope is the ordinary EDV Encrypted Document
  shape; only the key resolution process differs from a native EDV. Each
  envelope additionally binds its epoch into the JWE protected header
  ([[PWS-EC]]'s `was` binding), checked unconditionally on read.
* The **descriptor's `recipients` entries wrap the 32-byte epoch secret to each
  reader's own key-agreement key** with `ECDH-ES+A256KW` (ephemeral-static
  ECDH, the RFC 7518 Concat KDF, AES key wrap). A reader finds its `kid` in an
  epoch's `recipients`, unwraps the epoch secret with its own key, and
  reconstructs the epoch key pair. A failed unwrap is a failure.
  In particular, key servers whose unwrap operation resolves a null key on
  mismatch must be handled by trying the next candidate or failing with a
  typed error.
* **Reads** use the Resource's [`epoch` stamp](#epoch-stamping-on-resources)
  (from metadata, the listing
  item, or the feed document) as advisory pre-fetch routing: it lets a
  replica select and unwrap the epoch key before fetching the envelope. The
  authoritative epoch is the envelope's own -- the JWE recipient `kid`
  names it, and the AEAD-bound epoch binding is verified against the
  decrypting key ([[PWS-EC]]). An absent stamp therefore just means
  route-after-fetch; it is never treated as "assume `currentEpoch`", and
  there are no unstamped pre-epoch resources to tolerate.
* **Writes** always encrypt under `currentEpoch` and stamp it via
  `Key-Epoch`.
* **Adding a reader** wraps EVERY epoch's key to it (adding a reader means it
  can read the Collection, history included) and writes the updated Metadata
  object with `If-Match`. No rotation: adds are inexpensive, removals rotate.
* **Removing a reader** is one indivisible procedure: (1) revoke the reader's
  capabilities; (2) mint a fresh epoch key, wrap it to each REMAINING
  recipient, append the epoch, repoint `currentEpoch`, write with `If-Match`;
  (3) subsequent writes use the new epoch. Client libraries should not expose
  a rotate-without-revoke or revoke-without-rotate implementation of "remove".
* On a [=precondition-failed=] response, re-read the Metadata object, re-apply
  the recipient change to the fresh descriptor, and retry (bounded).

</div>

#### Security considerations

* **Limitations.** Rotation protects Resources written after the
  rotation, and nothing else. It cannot somehow delete the data a removed reader
  already downloaded; Resources still stored under an earlier epoch remain
  readable to a removed reader that obtains their ciphertext (a backup, a
  colluding reader, a feed pull made before revocation); and it provides no
  post-compromise security for the removed reader's past traffic. Closing
  those gaps requires re-encrypting the Collection under the new epoch, which is
  a client-side bulk rewrite, out of scope here. Specifications and libraries
  documenting removal MUST state this limitation rather than implying stronger
  guarantees.
* **Pull and read stay separate.** The capability governs pull
  (server-enforced, immediate); the epoch key governs read (mathematics,
  prospective). Documentation and error messages should never conflate them.
* **The [blinding key](#blinding-key-member) does not rotate with the
  epoch.** Rotating the `hmac` reference on removal would invalidate every
  blinded index in the Collection, which is why the server treats the member
  as permanent (see [[[#blinding-key-member]]] and
  [[[#key-epoch-server-validation]]]). A removed reader retaining the ability
  to compute blinded index terms is harmless: the server gates the
  `blinded-index` query profile (see [[[#query-profile-blinded-index]]])
  behind a capability the reader no longer holds.
* **A rotation emits no [`changes` feed](#query-profile-changes) entry.** A rekey is a Collection
  Description change, not a Resource change. A replicating reader that
  encounters an `epoch` it does not know MUST re-read the Collection
  Description.

</section>

<section class="appendix">

## Query Profile Registry {#query-profile-registry}

This appendix is normative.

<div class="note">
This registry defines values used by an OPTIONAL feature (the query endpoint): a
server MAY expose no query endpoint and remain conformant; see
[[[#scope-and-conformance-profiles]]]. Optionality is orthogonal to normativity:
a server that does serve queries MUST follow the request and response shapes
catalogued here.
</div>

A [=collection=] MAY expose an OPTIONAL query endpoint at
`POST /space/{space_id}/{collection_id}/query`. The endpoint is a single URL that
serves several distinct query dialects; a request selects one by carrying a
`profile` discriminator in its body. This registry maps each `profile` token to
the request and response wire shape of that dialect.

The request body is an `application/json` object with a REQUIRED `profile` string
member. All other members are profile-specific. A [=server=] that recognizes the
`profile` answers with `200 OK` on success and a profile-specific response body.
A server that does not implement a given `profile` MUST respond with an
[=unsupported-operation=] (`501`) error.

The `profile` vocabulary is open and additive: this specification defines
the tokens below, and future profiles register additional tokens. A server MUST
treat a `profile` it does not recognize the same as one it does not serve, with
[=unsupported-operation=].

**Authorization.** The query endpoint requires read authorization (a capability
or a [=policy=] grant, as for any read); the invoked [=action=] is [=POST=]. The
capability's [=invocationTarget=] is the bare `/query` URL. Every query parameter
travels in the request body, which is signed and covered by the request `Digest`
(see [[[#request-body-integrity-digest-header]]]), so no query-string
attenuation is involved, and a capability that authorizes the `/query` target
authorizes any query body it signs. As with every other operation, a server MUST
verify authorization before validating the request body. An under-authorized
caller therefore receives the privacy-preserving [=not-found=] (`404`, see
[[[#error-handling]]]) and never learns whether its query body was well-formed,
nor whether the target Collection exists.

| `profile`        | Summary                                                     | Defining subsection                 |
|------------------|-------------------------------------------------------------|-------------------------------------|
| `changes`        | Ordered replication change feed (a resumable pull loop)     | [[[#query-profile-changes]]]        |
| `blinded-index`  | Matching over client-computed blinded (HMAC'd) attributes   | [[[#query-profile-blinded-index]]]  |

### `changes` profile {#query-profile-changes}

The `changes` profile serves an ordered, resumable feed of the content and
metadata changes in a [=collection=], so that an offline-first replication client
can pull a Collection's state incrementally and keep a local replica in sync. A
server that serves this profile advertises `changes-query` in its service
description `features` array (see [[[#service-description-data-model]]]),
rather than per Collection or per backend.

Request body:

```json
{
  "profile": "changes",
  "checkpoint": { "id": "<resourceId>", "updatedAt": "<ISO-8601 timestamp>" },
  "limit": 100
}
```

* `checkpoint` (OPTIONAL) - the position to resume from, echoed from a prior
  response (see below). When present it MUST be an object with a string `id` and a
  string `updatedAt`; a malformed `checkpoint` is rejected with an
  [=invalid-request-body=] (`400`) error. When absent, the feed starts at the
  beginning.
* `limit` (OPTIONAL) - the requested batch size. A non-finite value, or a value
  less than `1`, falls back to a server default (a server default of `100` is
  RECOMMENDED); a server MAY clamp `limit` down to an implementation maximum.

Response body:

```json
{
  "documents": [
    {
      "id": "hello-world",
      "_deleted": false,
      "updatedAt": "2026-01-15T12:00:00.000Z",
      "version": 1,
      "etag": "3mJr7AoUXx2.1",
      "metaVersion": 3,
      "metaEtag": "3mJr7AoUXx2.3",
      "createdBy": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
      "writerId": "z6fVXHKn8PdQm2Rt",
      "data": { "message": "hi" },
      "custom": {
        "name": "Hello World greeting",
        "tags": { "project": "demo", "status": "draft" }
      }
    }
  ],
  "checkpoint": { "id": "hello-world", "updatedAt": "2026-01-15T12:00:00.000Z" }
}
```

Each entry in `documents` describes one changed Resource:

* `id` - the Resource id.
* `_deleted` - a boolean; `true` on a tombstone (a soft-deleted Resource), `false`
  otherwise. The underscore-prefixed member name is deliberate: it matches the
  wire convention of offline-first replication clients.
* `updatedAt` - the ISO-8601 timestamp of the change; the feed's ordering key.
* `version` - a monotonic content version, for ordering and comparison. It is
  always present, and a content write bumps it. It is not a validator: a client
  MUST NOT construct an `If-Match` value from it.
* `etag` (optional) - the Resource's current content `ETag`, quoted exactly as
  the server returns it in the response header (see
  [[[#conditional-requests]]]). A server that supports conditional writes MUST
  include it, so that a replica can send `If-Match` from the feed entry without
  a `GET` per Resource. A client echoes it verbatim.
* `metaVersion` - a monotonic metadata version, present only once metadata has
  been written for the Resource, and bumped by a metadata-only edit.
* `metaEtag` (optional) - the current `ETag` of the Resource's Metadata object,
  quoted exactly as the server returns it on the `meta` sub-resource. Present
  under the same conditions as `metaVersion` on a server that supports
  conditional writes; echoed verbatim as `If-Match` on a metadata write.
* `createdBy` - the [=did=] of the Resource's creator, as defined in
  [[[#resource-metadata-data-model]]]. A server that records `createdBy` SHOULD
  surface it here, so that a replica learns each Resource's creator from the feed
  rather than dereferencing the `meta` sub-resource of every Resource it pulls.
  Present only when recorded; a client MUST treat an absent `createdBy` as "not
  recorded". It is server-managed and read-only, as it is everywhere else.
* `data` - the stored JSON body verbatim. The body is nested under `data` (rather
  than spread onto the entry) so that a non-object JSON body round-trips
  faithfully. Omitted on a tombstone.
* `custom` - the user-writable metadata object (see
  [[[#resource-metadata-data-model]]]), present only when set. On an encrypted
  Collection this is the opaque client-encrypted envelope, passed through
  untouched.
* `epoch` (optional) - the Resource's key-epoch stamp, mirroring the Resource
  Metadata property (see [[[#key-epochs]]]), carried so that a replica can
  select its epoch key without a metadata fetch per document. Note that a
  rekey (a Collection Metadata change) emits no feed entry of its own: a
  replica that encounters an `epoch` it does not know re-reads the Collection
  Description.
* `writerId` (optional) - the Resource's writer-attribution label, mirroring
  the Resource Metadata property (see [[[#writer-attribution]]]). A server
  that stores `writerId` MUST include it here, as this is the member that lets a
  replica recognize its own writes echoed back (entries carrying its own
  label) without fetching (and on an encrypted Collection, without decrypting)
  each document, and lets replicas break same-`updatedAt` last-writer-wins
  ties deterministically on a shared `(updatedAt, writerId)` key. Like
  everywhere else it is advisory and never server-verified.

**Tombstones.** A soft-deleted Resource surfaces as
`{ "id", "_deleted": true, "updatedAt", "version", "etag" }` with no `data`
member; the deletion bumps `version` and `etag` is the tombstone's validator.
A tombstone retains its `createdBy`, where one was
recorded, so that a deletion replicates together with the attribution of the
Resource it removes. It also carries as `writerId` the label the deleting
request declared (see [[[#writer-attribution]]]), if any. Since a deletion is a
revision, its attribution and tie-breaking work like any other write's.

**Ordering and resumption.** Entries are ordered by an ascending `(updatedAt, id)`
keyset. The top-level `checkpoint` echoes the position of the last entry in
`documents`, or is `null` when nothing changed past the supplied `checkpoint`
(the end of the feed). A client resumes by sending the returned `checkpoint` as
the `checkpoint` of its next request; the resulting sequence of calls is the pull
loop of a replication protocol. A metadata-only edit re-surfaces the Resource
with a bumped `updatedAt` and `metaVersion` but an unchanged `version` and `data`.

**Scope.** Binary (non-JSON) Resources are excluded from the `changes` feed; blob
replication is out of scope for this profile.

Example -- request the first batch and receive one changed document plus the
checkpoint to resume from:

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/query HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{ "profile": "changes", "limit": 100 }
```

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "documents": [
    {
      "id": "hello-world",
      "_deleted": false,
      "updatedAt": "2026-01-15T12:00:00.000Z",
      "version": 1,
      "createdBy": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
      "data": { "message": "hi" }
    }
  ],
  "checkpoint": { "id": "hello-world", "updatedAt": "2026-01-15T12:00:00.000Z" }
}
```

### `blinded-index` profile {#query-profile-blinded-index}

The `blinded-index` profile serves queries over client-computed blinded (HMAC'd)
attributes, so that a [=server=] can match encrypted documents without seeing
plaintext attribute names or values. Its semantics follow the query operation of
the [Encrypted Data Vaults](https://identity.foundation/edv-spec/) specification
(the same specification the [[[#encryption-scheme-registry]]] references for the
`edv` envelope). This profile is a PWS-EC affordance: a server that serves it
advertises `blinded-index-query` in its PWS-EC version entry's `features`
array (see [[[#service-description-data-model]]]).

Documents opt in by carrying the EDV `indexed` member: an array of entries of the
shape `{ "hmac": { "id", "type" }, "sequence", "attributes": [ { "name",
"value", "unique"? } ] }`, in which each `name` and `value` is a blinded (HMAC'd)
token computed by the client. A server stores the `indexed` member and matches
against it, but cannot invert it back to the plaintext attribute.

Request body:

```json
{
  "profile": "blinded-index",
  "index": "<HMAC key id>",
  "equals": [ { "<blindedName>": "<blindedValue>" } ],
  "has": [ "<blindedName>" ],
  "count": false,
  "limit": 10,
  "cursor": "<opaque token>"
}
```

* `index` (REQUIRED) - a non-empty string naming which blinded index (the HMAC
  key `id`) to search. Matching is scoped to entries under this index.
* Exactly one of `equals` or `has` MUST be present; supplying neither, or both,
  is an [=invalid-request-body=] (`400`) error.
  * `equals` - a non-empty array of objects, each mapping a blinded attribute name
    to a blinded attribute value (all strings). The array is a disjunction (OR
    across its elements); the pairs within one element are a conjunction (AND). An
    empty object element matches nothing.
  * `has` - a non-empty array of blinded attribute name strings; matches every
    document that carries every named attribute under the index, regardless of
    value.
* `count` (OPTIONAL) - a boolean. When `true`, the response is exactly
  `{ "count": <number> }` (the number of matching documents) rather than a
  document page.
* `limit` (OPTIONAL) - the requested page size, handled leniently as for the
  [[[#query-profile-changes]]] profile (a non-finite or `< 1` value falls back to
  a server default; a server MAY clamp to a maximum).
* `cursor` (OPTIONAL) - an opaque pagination token from a prior response (see
  below). A malformed `cursor` is rejected with an [=invalid-cursor=] (`400`)
  error.

Response body (a document page):

```json
{ "documents": [ { } ], "hasMore": false, "cursor": "<opaque token>" }
```

* `documents` - the matching stored documents, verbatim (an encrypted envelope
  passes through untouched), in ascending Resource `id` order.
* `hasMore` - a boolean, `true` when more matching documents follow the page.
* `cursor` - an opaque continuation token, present if and only if `hasMore` is
  `true`. A client resumes by echoing it as the `cursor` of its next request. As
  with pagination cursors elsewhere (see [[[#pagination]]]), a client MUST treat
  the token as opaque and MUST NOT construct, parse, or modify it.

<div class="note">

Cursor pagination is a deliberate PWS extension over the Encrypted Data Vaults
query operation, which offers only `limit` together with a `hasMore` flag and no
means to fetch the next page. A server that also implements the EDV query
operation directly may expose `limit`/`hasMore` there while offering the
`cursor` continuation only through this profile.

</div>

Example -- match documents whose blinded `index-2` attribute equals a blinded
value:

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/query HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "profile": "blinded-index",
  "index": "z1A...hmacKeyId",
  "equals": [ { "aBlindedName": "aBlindedValue" } ],
  "limit": 10
}
```

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "documents": [
    { "id": "urn:uuid:...", "sequence": 0, "jwe": { "ciphertext": "..." } }
  ],
  "hasMore": false
}
```

#### Unique blinded attributes

An `indexed` attribute entry MAY carry `"unique": true`, marking its
`(hmac id, name, value)` triple as a uniqueness claim within the Collection. A
[=server=] MUST reject a Resource content write that would claim a triple already
claimed with `unique: true` by a *different* Resource in the same [=collection=],
with an [=id-conflict=] (`409`) error. A document re-asserting its own existing
claim never self-conflicts, and the same triple carried *without* `unique`
coexists freely.

This uniqueness is enforced at write time, not at query time. As with the
[=id-conflict=] rule elsewhere in this specification (see the id-conflict privacy
note in [[[#error-type-registry]]]), a server MUST verify the caller's
authorization to write at the target before performing this check, so that the
existence-revealing `409` is observable only to a caller already authorized to
write; an under-authorized caller receives the merged [=not-found=] instead. The
check MUST be atomic with the write, so that two concurrent writers cannot both
claim the same triple.

</section>

<section class="appendix">

## EDV-over-PWS Profile v0.1 {#edv-over-pws-profile-v0-1}

This appendix is normative for clients that claim conformance to it.

<div class="note">
This profile is a **client-side layout convention**: it constrains how a client
maps [Encrypted Data Vault](https://identity.foundation/edv-spec/) (EDV)
operations onto ordinary PWS operations. A
conforming PWS server needs nothing beyond the features it already advertises,
and never learns that it is hosting an EDV. Its normative requirements therefore
bind the *client*; a server's obligations are only the ones it already has for
the underlying PWS operations.
</div>

### Purpose {#edv-over-pws-purpose}

An Encrypted Data Vault stores JWE-encrypted documents that its server can query
by blinded index but never decrypt. A client can realize all of that behavior on
a plain PWS server, because a PWS Resource is an opaque byte store and
PWS already carries the pieces an EDV needs. This profile fixes the mapping so
that independent clients interoperate over the same encrypted Collection:

| EDV concept           | PWS realization                                                                    |
|-----------------------|------------------------------------------------------------------------------------|
| Vault                 | [=collection=]                                                                     |
| Document              | Resource (the EDV envelope stored as its content)                                  |
| Document `sequence`   | conditional writes (`If-Match` / `If-None-Match`; see [[[#conditional-requests]]]) |
| Blinded index + query | the `blinded-index` query profile (see [[[#query-profile-blinded-index]]])         |
| Stream chunks         | chunk addressing (see [[[#chunked-resources]]])                                    |

Encryption itself is out of scope of the server entirely: the client holds all
keys and performs all encryption, decryption, and index blinding, as required by
[[[#stored-data-is-opaque-to-the-storage-provider]]]. The server enforces PWS
authorization, maximum-privacy `404`s, and policies over the ciphertext
unchanged; the encryption is defense in depth layered on top, not a replacement
for the PWS authorization model.

### The encryption descriptor {#edv-over-pws-descriptor}

A Collection realizing this profile declares the `encryption` descriptor
`{ "scheme": "edv", "version": 1 }` in its Collection Metadata object (see
[[[#collection-metadata-data-model]]]; a bare `{ "scheme": "edv" }` is
equivalent, since
an absent `version` means `1`). The wire `version` is the integer registry key
of the envelope format, distinct from this appendix's own "v0.1" maturity
label. The `edv` scheme's envelope wire format and the
server's structural fail-closed validation of it are defined by the
[[[#encryption-scheme-registry]]] and are not restated here. A server that
recognizes the `edv` scheme thereby guarantees, without holding any key, that a
plaintext write into the Collection is rejected -- so the profile inherits the
registry's fail-closed property for free.

Key management -- how the JWE's recipient keys are chosen, distributed, rotated,
or organized into epochs for multi-recipient sharing -- is deliberately outside
the scope of both this profile and this specification, exactly as it is for a
native EDV: those keys live in the client, and the server sees only opaque
envelopes. Multi-recipient encryption is carried either directly, in the JWE
`recipients` structure the [[[#encryption-scheme-registry]]] describes, or
through the key-epoch indirection of [[[#key-epochs]]], whose public
bookkeeping (the descriptor's `epochs` roster and the Resource `epoch` stamp)
the server stores and serves without interpreting; the client-side epoch
construction is specified in [[PWS-EC]].

### Document layout {#edv-over-pws-document-layout}

A client stores each EDV document as the EDV Encrypted Document envelope -- the
JSON object `{ id, sequence, indexed?, jwe }` -- as the Resource's content, at
the Resource id equal to the EDV document id. The envelope is the `edv` scheme's
registered wire format (see [[[#encryption-scheme-registry]]]); a client SHOULD
write it under the JWE JSON Serialization media type `application/jose+json`
([[RFC7516]]) where the server registers a parser for it, and MAY fall back to
`application/json`, which an unmodified PWS server accepts. The plaintext type of
the resource, and any user-visible metadata, ride *inside* the JWE and are never
server-visible; the server stores one opaque envelope regardless of what the
decrypted document is.

### Sequence mapping {#edv-over-pws-sequence}

EDV gives every document a monotonic `sequence` and enforces `previous + 1`
atomically server-side. This profile maps that onto PWS conditional writes (see
[[[#conditional-requests]]]):

* A fresh insert is a `PUT` carrying `If-None-Match: *`, so a collision with an
  existing document surfaces as [=precondition-failed=] (`412`) rather than a
  silent overwrite.
* An update pre-reads the stored envelope, advances `sequence` to `previous + 1`,
  and writes it back with `If-Match` pinned to the `ETag` observed on that read,
  so a concurrent writer's stale update is a `412` rather than a lost update.

Both preconditions are honored on any conformant PWS server, since conditional
writes are a baseline server requirement (see [[[#conditional-requests]]]).

### Chunked streams {#edv-over-pws-chunked-streams}

A large or streamed EDV document is stored as chunks, using PWS chunk
addressing (see [[[#chunked-resources]]]) against a server that implements
[[PWS-EC]]:

1. The client writes the document envelope **first**, as an ordinary Resource
   (satisfying the [[[#store-chunk-operation]]] rule that the parent Resource must
   exist before any of its chunks). The envelope carries `stream` metadata --
   `{ sequence, chunks }`, where `chunks` is the total chunk count -- so a reader
   can learn the extent of the stream from the document alone.
2. For each chunk `i` in `0 .. chunks - 1`, the client serializes the EDV chunk
   object `{ index, offset, sequence, jwe }` to JSON and `PUT`s it to chunk index
   `i` under the `application/octet-stream` content type. The octet-stream type is
   deliberate: it routes the chunk through the server's raw-binary write path,
   which is bounded by the backend's `maxUploadBytes` (tens of MiB) rather than
   the much smaller body-size limit servers typically impose on JSON-parsed
   requests (the reference server's is 1 MiB), which a full encrypted chunk
   would exceed. The server stores the bytes verbatim and never parses them, so
   a reader decodes and parses the chunk object back client-side.
3. A reader fetches the document envelope, reads `stream.chunks`, fetches chunk
   indexes `0 .. chunks - 1` (see [[[#read-chunk-operation]]]), and decrypts each
   `jwe` client-side to reassemble the stream.

<div class="note">
**Security consideration (stream integrity).** The chunk framing above
authenticates the *bytes* of each chunk (each `jwe` is an AEAD ciphertext) but
does **not** authenticate a chunk's *position* in the stream or the stream's
*total length*: the index, offset, and count are carried in plaintext framing and
in the (individually authenticated but not cross-linked) chunk objects. A
malicious server that reorders, drops, duplicates, or truncates chunks is
therefore **not** reliably detectable by the client with this framing. A hardened
framing -- a per-stream authenticated header, an index-bound AAD on each chunk,
and an authenticated total length, in the manner of a Cryptomator-style file
header -- closes this gap and is planned upstream in the EDV specification. When
it lands, this profile will adopt it under a new scheme `version`; until
then, a deployment that does not trust its storage provider for availability and
ordering integrity SHOULD treat streamed documents accordingly.
</div>

### Search and uniqueness {#edv-over-pws-search}

Content search over encrypted documents uses the `blinded-index` query profile
(see [[[#query-profile-blinded-index]]]): a client attaches blinded (HMAC'd)
`indexed` attributes to the envelope and queries them via
`POST /space/{space_id}/{collection_id}/query`. Blinded-attribute uniqueness
(`unique: true`) is enforced server-side by that profile (see
[[[#query-profile-blinded-index]]], *Unique blinded attributes*). Both require
a server whose PWS-EC version entry advertises `blinded-index-query` (see
[[[#service-description-data-model]]]); a client SHOULD gate their use on it.

### What this profile does not provide {#edv-over-pws-limits}

Relative to a dedicated EDV server, this profile reaches full parity for the
affordances a given server advertises, and has these limitations otherwise:

* **Server-side blinded-index query** and **`unique: true` enforcement**
  require a server whose PWS-EC version entry advertises
  `blinded-index-query`. Without it, a client cannot query blinded attributes
  at the server or rely on it to enforce uniqueness; it must fetch-and-filter
  client-side (or, as the reference codec does, mint restrict-mode document ids
  and keep all searchable metadata inside the envelope), and a uniqueness
  constraint is at best a racy client-side read-then-write.
* **Stream ordering / truncation integrity** is not authenticated by the current
  chunk framing (see the security consideration in
  [[[#edv-over-pws-chunked-streams]]]).

For a small single-writer Collection (for example, a credential wallet) these
limitations rarely matter; a large or multi-writer Collection wants a server
that advertises the blinded-index affordance above.

</section>

<section class="appendix">

## Error Type Registry {#error-type-registry}

This appendix is normative.

This specification uses [[RFC9457]] Problem Details for HTTP APIs for error
responses (see [[[#error-handling]]]). The `type` property of a problem
response is a URI identifying the _kind_ of problem. Following [[RFC9457]],
a `type` is reused across operations: a single kind such as
[=invalid-id=] is emitted by Create, Update, and Read operations
across Spaces, Collections, and Resources alike. The per-occurrence specifics
belong in the `errors` array (`detail` and an optional `pointer`), and the
short human-readable summary in `title`.

Each `type` URI is a fragment anchor into this registry. The status codes
listed below are typical; a single kind MAY be returned with more than one
status code depending on the operation.

| `type` URI                                                  | Anchor                                                                      | Typical status | Description                                                                                                                                                                                                                                                                                                                                                                                                      |
|-------------------------------------------------------------|-----------------------------------------------------------------------------|----------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `https://w3id.org/pws#not-found`                     | <dfn id="not-found">not-found</dfn>                                         | 404            | The resource (Space, Collection, or Resource) does not exist, or the caller is not authorized to access it. These two conditions are deliberately indistinguishable -- see the privacy note below.                                                                                                                                                                                                               |
| `https://w3id.org/pws#invalid-id`                    | <dfn id="invalid-id">invalid-id</dfn>                                       | 400            | A Space, Collection, or Resource `id` is missing or not URL-safe.                                                                                                                                                                                                                                                                                                                                                |
| `https://w3id.org/pws#reserved-id`                   | <dfn id="reserved-id">reserved-id</dfn>                                     | 409            | A client-supplied `id` collides with a [[[#reserved-path-segment-registry]]] segment.                                                                                                                                                                                                                                                                                                                            |
| `https://w3id.org/pws#id-conflict`                   | <dfn id="id-conflict">id-conflict</dfn>                                     | 409            | A client-supplied `id` in a `POST` create operation already exists. Also returned when a write would violate a `unique: true` blinded-attribute claim (see [[[#query-profile-blinded-index]]]) or a `unique: true` [=plaintext index=] claim, and when a Collection update adds a `unique: true` plaintext index over Resources that already violate it (see [[[#collection-metadata-data-model]]]). (Create-or-replace by `id` is done idempotently via `PUT`, which does not conflict.)                                                                                                                                                                                                                                                         |
| `https://w3id.org/pws#invalid-request-body`          | <dfn id="invalid-request-body">invalid-request-body</dfn>                   | 400            | The request body is missing or invalid (e.g. a required property is absent). Entries in `errors` SHOULD carry a `pointer` to the offending field.                                                                                                                                                                                                                                                                |
| `https://w3id.org/pws#invalid-cursor`                | <dfn id="invalid-cursor">invalid-cursor</dfn>                               | 400            | A pagination `cursor` query parameter is malformed or can no longer be honored (e.g. an expired snapshot). See [[[#pagination]]].                                                                                                                                                                                                                                                                                |
| `https://w3id.org/pws#missing-content-type`          | <dfn id="missing-content-type">missing-content-type</dfn>                   | 400            | A required `Content-Type` header is missing.                                                                                                                                                                                                                                                                                                                                                                     |
| `https://w3id.org/pws#missing-authorization`         | <dfn id="missing-authorization">missing-authorization</dfn>                 | 401            | Required `Authorization` / `Capability-Invocation` headers (or proof of possession) are missing.                                                                                                                                                                                                                                                                                                                 |
| `https://w3id.org/pws#invalid-authorization-header`  | <dfn id="invalid-authorization-header">invalid-authorization-header</dfn>   | 400            | An `Authorization`, `Capability-Invocation`, or `Digest` header is malformed, unparseable, or failed verification.                                                                                                                                                                                                                                                                                               |
| `https://w3id.org/pws#controller-mismatch`           | <dfn id="controller-mismatch">controller-mismatch</dfn>                     | 400            | The capability invocation in a Create Space request is not currently authorized by the `controller` supplied in the request body: it is neither signed by that DID nor accompanied by a valid, unexpired delegation chain rooted in it. Servers SHOULD differentiate the cause (chain rooted elsewhere, expired delegation, failed proof) in the `detail` string where they can; see [[[#create-space-errors]]]. |
| `https://w3id.org/pws#unsupported-backend`           | <dfn id="unsupported-backend">unsupported-backend</dfn>                     | 409            | A requested `backend` id is not in the space's [[[#space-backends-available]]] list.                                                                                                                                                                                                                                                                                                                             |
| `https://w3id.org/pws#encryption-immutable`          | <dfn id="encryption-immutable">encryption-immutable</dfn>                   | 409            | A Collection update tried to change the `scheme`, decrease or remove the `version`, or clear an existing `encryption` descriptor; or it tried to change the `id` or `type` of the descriptor's `hmac` member, or remove that member. The descriptor is set-once, version-monotonic: declaring it on a Collection that lacks one is allowed (and re-declaring the standing values is a no-op), but changing its `scheme`, moving its `version` backward, or clearing it on a populated Collection would corrupt the stored, client-encrypted Resources, and replacing or dropping the blinding key would orphan every blinded index. See [[[#collection-metadata-data-model]]] and [[[#blinding-key-member]]]. |
| `https://w3id.org/pws#encryption-history-log-governed` | <dfn id="encryption-history-log-governed">encryption-history-log-governed</dfn> | 409            | A Collection update carried an `encryption` member on a Collection whose descriptor is governed by its history log. The member is read-only on that path: it is derived from the log's head entry, and changes by an append to the log at the Collection's `meta/log` sub-resource. See [[[#collection-governing-history-log]]]. |
| `https://w3id.org/pws#encryption-scheme-mismatch`    | <dfn id="encryption-scheme-mismatch">encryption-scheme-mismatch</dfn>       | 422            | A write into an encrypted Collection -- a Resource's content, or the `custom` object of a Resource's or the Collection's own Metadata -- had a body (or `Content-Type`) that does not conform to the Collection's declared `encryption` scheme envelope profile. Reachable only by a caller already authorized to write -- see [[[#encryption-scheme-registry]]].                                                                                                                                           |
| `https://w3id.org/pws#unsupported-encryption-scheme` | <dfn id="unsupported-encryption-scheme">unsupported-encryption-scheme</dfn> | 400            | A Collection create/update declared an `encryption` `scheme` (or a `version` of one) the server does not recognize or support. See [[[#encryption-scheme-registry]]].                                                                                                                                                                                                                                                                    |
| `https://w3id.org/pws#precondition-failed`           | <dfn id="precondition-failed">precondition-failed</dfn>                     | 412            | A conditional write's `If-Match` / `If-None-Match` precondition evaluated false: the Resource's current version did not match, or a create-if-absent target already exists. Header-driven and distinct from the `409` conflict kinds. See [[[#conditional-requests]]].                                                                                                                                           |
| `https://w3id.org/pws#quota-exceeded`                | <dfn id="quota-exceeded">quota-exceeded</dfn>                               | 507            | A write was rejected because the target backend's storage quota is exhausted. See [[[#quotas]]].                                                                                                                                                                                                                                                                                                                 |
| `https://w3id.org/pws#payload-too-large`             | <dfn id="payload-too-large">payload-too-large</dfn>                         | 413            | An upload exceeds the target backend's `maxUploadBytes` constraint (see [[[#quotas]]]). Note that unlike [=quota-exceeded=], this rejection is per-request: smaller uploads may still succeed.                                                                                                                                                                                                                   |
| `https://w3id.org/pws#unsupported-operation`         | <dfn id="unsupported-operation">unsupported-operation</dfn>                 | 501            | An optional operation that this server or the target backend does not support (for example, a per-collection quota report on a backend without per-collection accounting).                                                                                                                                                                                                                                       |
| `https://w3id.org/pws#invalid-import`                | <dfn id="invalid-import">invalid-import</dfn>                               | 400            | An uploaded archive is not a valid PWS space export. The import operation itself is reserved and not yet specified (see [[[#reserved-path-segment-registry]]]); this kind is registered ahead of it so implementations converge on one error shape.                                                                                                                                                              |
| `https://w3id.org/pws#storage-error`                 | <dfn id="storage-error">storage-error</dfn>                                 | 500            | An underlying storage operation failed.                                                                                                                                                                                                                                                                                                                                                                          |
| `https://w3id.org/pws#internal-error`                | <dfn id="internal-error">internal-error</dfn>                               | 500            | An unexpected server-side fault with no more specific kind.                                                                                                                                                                                                                                                                                                                                                      |

**Privacy: the `not-found` kind is intentionally merged.** Under the principle
of maximum privacy (see [[[#error-handling]]]), an unauthorized client MUST NOT
be able to discover the existence of a Space, Collection, or Resource from an
error response. The [=not-found=] kind therefore covers both
"resource absent" and "invalid authorization"; implementations MUST NOT split
it into distinguishable `type` values, and MUST NOT otherwise let the response
(status code, `title`, or `detail`) reveal whether the resource exists.

This merging applies to "insufficient-authorization" and "absent-authorization"
against an existing target. It does not apply to request- or
credential-validation failures, which describe the request rather than the
target and so MAY use their own precise `type`s -- see [[[#error-handling]]].

**Privacy: `id-conflict` is existence-revealing by nature** -- a `409` confirms
that the supplied id is taken. For Create Collection and Create Resource,
servers MUST therefore verify the caller's authorization before checking for
a conflict, so that only callers already authorized to create at that level can
observe the signal; an under-authorized caller receives the merged
[=not-found=] instead. For Create Space, where any caller permitted to attempt
creation learns the same fact from whether creation succeeds, the disclosure is
inherent -- see the note under [[[#create-space-errors]]].

### Error Response Examples

Canonical example responses for the error kinds defined above, referenced by
the per-operation "Errors" lists throughout this specification. The `title`
strings are illustrative, not normative: for example, the noun in a
[=not-found=] `title` varies with the target (Space, Collection, or Resource),
and a `title` may name the operation that failed. Kinds not shown here
([=missing-content-type=], [=invalid-authorization-header=],
[=invalid-import=], [=storage-error=], [=internal-error=]) follow the same
shape; they will gain examples as their corresponding sections are drafted.

[=not-found=] -- a missing target, or missing or insufficient authorization
(deliberately indistinguishable, see the privacy note above):

```http
HTTP/1.1 404 Not Found
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#not-found",
  "title": "Resource not found or insufficient authorization."
}
```

[=invalid-id=] -- an `id` that is not URL-safe (see [[[#identifiers]]]):

```http
HTTP/1.1 400 Bad Request
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#invalid-id",
  "title": "Invalid Space id.",
  "errors": [
    {
      "detail": "Space 'id' must be URL-safe.",
      "pointer": "#/id"
    }
  ]
}
```

[=reserved-id=] -- a client-supplied `id` that collides with a segment from the
[[[#reserved-path-segment-registry]]]:

```http
HTTP/1.1 409 Conflict
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#reserved-id",
  "title": "Invalid collection id (from reserved list)."
}
```

[=id-conflict=] -- a `POST` create operation supplying an `id` that already
exists:

```http
HTTP/1.1 409 Conflict
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#id-conflict",
  "title": "A Collection with this id already exists.",
  "errors": [
    {
      "detail": "Use PUT to create-or-replace a Collection at a chosen id.",
      "pointer": "#/id"
    }
  ]
}
```

[=precondition-failed=] -- a conditional `PUT` whose `If-Match` precondition did
not match the Resource's current version (a concurrent write landed first):

```http
HTTP/1.1 412 Precondition Failed
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#precondition-failed",
  "title": "The resource was modified by another write.",
  "errors": [
    {
      "detail": "Re-read the current resource, re-apply your change, and retry."
    }
  ]
}
```

[=invalid-request-body=] -- a missing or invalid request body (here, a Create
Space request without the required `controller` property):

```http
HTTP/1.1 400 Bad Request
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#invalid-request-body",
  "title": "Invalid Create Space body.",
  "errors": [
    {
      "detail": "'controller' property is required.",
      "pointer": "#/controller"
    }
  ]
}
```

[=missing-authorization=] -- required authorization (here, a proof of
possession signature on a Create Space request) is missing:

```http
HTTP/1.1 401 Unauthorized
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#missing-authorization",
  "title": "Invalid Create Space request.",
  "errors": [
    {
      "detail": "Valid proof of possession of the 'controller' DID must be provided."
    }
  ]
}
```

[=controller-mismatch=] -- the invocation is not currently authorized by the
DID specified in the `controller` property of a Create Space request body
(the `detail` here is the generic catch-all; servers that can tell SHOULD name
the specific cause instead, e.g. "The delegation chain is rooted in a DID
other than the body's 'controller'." or "The delegated capability has
expired."):

```http
HTTP/1.1 400 Bad Request
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#controller-mismatch",
  "title": "Invalid Create Space request.",
  "errors": [
    {
      "detail": "The invocation must be authorized by the 'controller' DID in the request body: signed by it, or via a delegation chain rooted in it.",
      "pointer": "#/controller"
    }
  ]
}
```

[=unsupported-backend=] -- a `backend` id that is not part of that space's
[[[#space-backends-available]]] list:

```http
HTTP/1.1 409 Conflict
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#unsupported-backend",
  "title": "Unsupported backend id, check the space's 'backends available' list."
}
```

[=encryption-immutable=] -- a Collection update tried to change the `scheme`,
decrease or remove the `version`, or clear an
existing `encryption` descriptor (it is set-once, version-monotonic; see
[[[#collection-metadata-data-model]]]):

```http
HTTP/1.1 409 Conflict
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#encryption-immutable",
  "title": "Collection encryption descriptor is immutable."
}
```

[=encryption-history-log-governed=] -- a Collection update carried an
`encryption` member on a Collection whose descriptor is governed by its
history log; append to the log instead (see
[[[#collection-governing-history-log]]]):

```http
HTTP/1.1 409 Conflict
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#encryption-history-log-governed",
  "title": "Collection encryption descriptor is governed by its history log."
}
```

[=encryption-scheme-mismatch=] -- a write into an encrypted Collection whose
body (a Resource's content, or the `custom` object of a metadata write) is
not an envelope of the Collection's declared scheme (observable only by a
caller already authorized to write; see [[[#encryption-scheme-registry]]]):

```http
HTTP/1.1 422 Unprocessable Content
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#encryption-scheme-mismatch",
  "title": "Body does not conform to the Collection's encryption scheme."
}
```

[=quota-exceeded=] -- a write rejected because the target backend's storage
quota is exhausted (see [[[#quotas]]]):

```http
HTTP/1.1 507 Insufficient Storage
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#quota-exceeded",
  "title": "Storage quota exceeded for backend 'default'."
}
```

[=payload-too-large=] -- an upload exceeding the target backend's
`maxUploadBytes` constraint:

```http
HTTP/1.1 413 Content Too Large
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#payload-too-large",
  "title": "Upload exceeds the backend's maximum upload size.",
  "errors": [
    {
      "detail": "Upload size 209715200 exceeds 'maxUploadBytes' of 157286400 for backend 'dropbox'."
    }
  ]
}
```

[=unsupported-operation=] -- an optional operation this server or backend does
not support (here, a per-collection quota report):

```http
HTTP/1.1 501 Not Implemented
Content-type: application/problem+json

{
  "type": "https://w3id.org/pws#unsupported-operation",
  "title": "Backend 'default' does not support per-collection quota reports."
}
```

</section>

<section class="appendix informative">

## Goals and Requirements {#goals-and-requirements}

This storage specification is intended to support the following goals and
requirements.

### Local-first and Offline capable storage

Users and apps need to be able to use (provision, set up, and start reading and
writing to) storage spaces without being connected to the internet.

### Storage and sharing of public, permissioned, and private encrypted data

Although the local-first offline functionality is necessary, writing data to
stable internet-accessible URLs for the purposes of sharing them is one of the
primary use cases of this specification.

* A user needs to be able to write data (that is intended to be world-readable)
  to a cloud-accessible URL, and be able to send that URL to intended recipients
  via any out-of-band mechanism such as email, chat, and so on.

* User needs to be able to change or revoke permissions at any point after
  sharing. Note that changed permissions apply only to operations that come after
  the change (this spec is not intended to solve the general problem of DRM).

* The sharing and permission system needs to be primarily based on authorization
  capabilities (zCaps). It also needs support storage-side authorization policies
  (even if only as a way for an authorized client to receive an appropriate zcap)

* The sharing mechanism needs to be flexible and granular. For example, a given
  data resource needs to be: world-readable, or readable by groups or categories,
  or by only those possessing the required authorization capabilities, or by
  no one except the author or controller, etc

* Advanced sharing conditions are also desirable (such as "this share expires
  after X amount of time" or "this is a one-time share and will expire after
  the first successful read request")

### Stored data is opaque to the storage provider {#stored-data-is-opaque-to-the-storage-provider}

* The spec needs to support (though not require) end-to-end client side
  encryption of the space. For plausible deniability, this might need to include
  all data (even marked as public-readable) is encrypted at rest

### Replication to user-controlled local and cloud servers

* Replication reconciles the first two requirements (data reads and writes must
  be offline-capable, but the data must eventually be able to be shared on the
  web via traditional URLs)

* Replication also provides critical availability and disaster recovery
  functionality

* Replication needs to be multi-primary (to reflect the multi-device and
  multi-client user environment)

* Multi-primary replication requires support for a versioning or conflict
  resolution mechanism

* Data, metadata, and permissions all need to be replicated

* Authorship and data provenance (the ability to tell which user or service
  created or edited a given set of data) must work in this permissioned
  multi-primary-write environment

### Serve as a General Purpose application storage backend

Intended to serve as a storage backend for credential wallets, and any other
client-side (Single Page Applications), server side, desktop, and mobile apps
and services.

### Data Portability

Data written to storage spaces using this specification needs to be portable:

* Authorized agents need to be able to export or backup all the data written,
  including all corresponding metadata and permissions

* The sharing and storage system needs to be able to support web domain
  independent identifiers. That is, a user must be able to share data at a given
  URL, then be able to migrate to a different storage service provider
  (potentially operating on a different web domain than the previous one), and
  the shared permissions to that data must not break after service migration

* While portability (and the not breaking of URLs) is relatively easy to achieve
  via redirect mechanisms (such as HTTP 301 and 302 redirect codes), this
  requires the previous service provider to be alive, available, and cooperative.
  However, this is true only of public-readable URLs, and the moment permissions
  are involved, cross-domain redirects become almost impossible to implement.
  In addition, portability from "dead servers" is also required. That is, if a
  cloud-based service provider disappears (or is otherwise unavailable), but a
  user still has a backup/export available, they should be able to set up another
  storage server (on another web domain or network address), and import/restore
  the data from backup, without shares and permissions breaking. Agents that
  the data was previously shared with must still be able to find the data at
  the new storage server location, and their permissions must still work.

### A Plurality of Data Formats and Protocols

* Spec needs to support the storage of data in any format -- binary files and
  objects, structured documents such as JSON or CBOR, contents of relational
  database tables, graphs, and anything else, all using the same unified
  metadata, sharing, and permission mechanisms.

* Storage-side schema enforcement is available but not required.

* Spec needs to be able to support multiple protocols and APIs, such as HTTP,
  JSON-RPC, DIDComm, local client APIs, and more.

### Permissioned Query and Search functionality

* Where appropriate (such as for unstructured text, structured documents, RDBMSs
  etc), storage needs to be queryable or searchable

* Any query/search mechanism needs to work well with the sharing/permission and
  replication requirements

### Upgradeable and legislation-compliant cryptography

All cryptography has a half-life.

* Any cryptographic operations (such as hashing, signatures, and encryption)
  used in this specification must be able to be obsoleted or upgraded, as
  techniques and algorithms break. To put it another way, the spec cannot
  "hardcode" any given algorithm (although it can recommend current best
  practices)

* Implementations of this spec need to be usable with FIPS-compliant
  cryptographic algorithms

### Anti-Goals

#### Use cases do not include "zero trust" environments

In a "zero trust" storage environment, the sync and replication nodes are
assumed to be untrusted: they hold only ciphertext, enforce no authorization of
their own, and encryption alone serves as the access control mechanism.

This storage specification is intentionally positioned to _not_ be used in such
environments. All encryption has an unpredictable half-life, and some use cases
do not permit relying on encryption only for access control.

What this specification pursues instead is a _combination_ of the two:
encryption protects data at rest and in transit, while minimally trusted
[=servers=] independently enforce the applicable [=policy=] on every request.
A broken cipher alone therefore does not grant an attacker access.

</section>

<section class="appendix informative">

## IANA Considerations {#iana-considerations}

This section will be submitted to the Internet Engineering Steering Group (IESG)
for review, approval, and registration with IANA.
</section>

