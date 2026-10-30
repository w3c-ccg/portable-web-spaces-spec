<div class="remove">

# Replication for Portable Web Spaces v0.1

**Abstract:** This document defines the replication profile for Portable Web
Spaces, a pull-only transport by which a Space held on one server is copied
to, and kept converged with, a Space on another server. The Space's
controller writes the registration that drives the pull.

**Status:** Experimental W3C CCG draft, undergoing regular revisions.
Rendered version: https://w3c-ccg.github.io/portable-web-spaces-spec/replication/

</div>

## Introduction {#introduction}

Portable Web Spaces (PWS) [[PWS]] is a permissioned storage API. A Space
lives on one server. Every capability a controller delegates names that
server's URL as its target, and every write lands there. This document makes
a Space live on more than one server. Each server accepts writes to its own
copy, and the servers converge.

Replication here is a pull. A replica server holds a capability that lets it
read a source Space, and it reads: the source's changes feed, then each
changed representation, which it stores through the same path and under the
same checks as a write a client would make. Nothing is pushed. No server
accepts a request that says "store this record under this stamp". A server
with no pull capability for a Space learns nothing about it, and a server
holding one holds read authority alone.

The core specification already carries what makes two copies comparable.
Every versioned record carries a write stamp, minted at the origin of the
write and stored verbatim by every copy. Every deleted Collection, Resource,
and policy leaves a tombstone. A changes feed lists every record kind. Those
are baseline requirements of [[PWS]], since a client's own deduplication
reads them whether or not any server replicates. This document is the
transport over them. It defines how a controller registers a source on a
replica server, what the replica server checks before it pulls, how it
pulls, and how it applies what it reads.

| Specification  | Relationship                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
|----------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [[PWS]]        | The storage API and the data model this document moves: the write stamp and its order, the stamp-derived `ETag`, Collection and policy tombstones, the `changes` query profile and its read-only form, the service description this document's [=version entry=] is listed in, the `replicas` and `zcaps` reserved segments, the `replicas` member of the Space Metadata object, and the registration-mapped resolution of a replicated controller log (its Self-hosted histories section). |
| [[PWS-AUTHZ]]  | The baseline authorization profile. A pull capability is a delegated capability under it, invoked over HTTP Signatures by the replica server, and the registration routes are authorized under it.                                                                                                                                                                                                                                                                                              |
| [[PWS-EC]]     | Chunked Resources. A replica server that pulls from a source listing [[PWS-EC]] reads each chunked Resource's chunk set as [[[#apply-chunks]]] describes. Nothing else in that profile is visible to replication, since envelopes are opaque bytes to both servers.                                                                                                                                                                                                                                 |
| [[APP-CONNECT]] | The replica pointer and the predicate under which a source server admits a pull capability signed by a credential rather than by an enrolled client ([[[#pull-capability-signing]]]). This document defines the pointer's service type and cites the predicate rather than restating it.                                                                                                                                                                                                       |
| [[DID-WEBVH]]  | The method of a replica server's own DID ([[[#server-identity]]]) and of a controller whose log a replica carries ([[[#replicated-controller-logs]]]).                                                                                                                                                                                                                                                                                                                                             |

### Relationship to the core specification {#relationship-to-the-core-specification}

This document is a companion to [[PWS]] and is versioned on its own. A
server implements it or not. A server that does not is still a conformant
PWS server. It can be the source of a pull by another server, since the pull
is made of ordinary reads, but it cannot hold a replica.

This document's persistent identifier is `https://w3id.org/pws/replication`.
A server that implements it lists the identifier in its service description
(see [[[#service-description]]]).

The following are defined by [[PWS]] and only cited here:

* The write stamp (`updatedAt`, `updatedAtCounter`, `originId`), its total
  order, and the `ETag` layout derived from it, in the Resource data model
  and the
  [Conditional Requests](https://w3c-ccg.github.io/portable-web-spaces-spec/#conditional-requests)
  section.
* The Collection tombstone, the tombstone-aware listing
  (`GET /space/{space_id}/?include=deleted`), and the Collection Metadata
  object's `created` member, in the
  [Collections](https://w3c-ccg.github.io/portable-web-spaces-spec/#collections)
  section.
* The policy tombstone and `GET .../policy?include=deleted`, in the
  [Access Control Policies](https://w3c-ccg.github.io/portable-web-spaces-spec/#access-control-policies)
  section.
* The `changes` query profile, its change document with the `kind` member,
  and its read-only `GET` form, in the
  [Query Profile Registry](https://w3c-ccg.github.io/portable-web-spaces-spec/#query-profile-changes).
* The `originId` and `serverDid` members of the core
  [version entry](https://w3c-ccg.github.io/portable-web-spaces-spec/#service-description-data-model).
* The `replicas` member of the
  [Space Metadata object](https://w3c-ccg.github.io/portable-web-spaces-spec/#space-metadata-data-model)
  and the `replicas` relation of the
  [Space linkset](https://w3c-ccg.github.io/portable-web-spaces-spec/#space-linkset).
* The `replicas` and `zcaps` segments of the
  [Reserved Path Segment Registry](https://w3c-ccg.github.io/portable-web-spaces-spec/#reserved-path-segment-registry).
* The `replica-refused` kind in the
  [Error Type Registry](https://w3c-ccg.github.io/portable-web-spaces-spec/#error-type-registry);
  see [[[#errors]]].

<div class="ednote">
The core text for the write stamp, the Collection and policy tombstones, and
the widened `changes` feed follows the same decisions as this document and
is still being written. Until it lands, the citations above point at the
sections that will carry it.
</div>

### Reading this document {#reading-this-document}

<div class="note">
This subsection is non-normative.

Examples use two servers. The source server is `source.example`, holding the
Space `81246131-69a4-45ab-9bff-9c946b59cf2e` that every example in [[PWS]]
uses. The replica server is `replica.example`, holding the Space
`3f1c2a9e-7b4d-4c61-9a0e-5d8b2f6c1e44`. The replica server's own DID is
abbreviated `did:webvh:Qm...:replica.example:space:server:id`. Paths are
written from each server's root. `Authorization: ...` stands for a signed
capability invocation as [[PWS-AUTHZ]] defines it.

The words "source" and "replica" name roles in one registration, not kinds of
server. One server is the source of one registration and the replica of
another, and two servers may each register the other as a source for the
same Space, which is what makes the Space multi-primary.
</div>

## Terminology {#terminology}

<dl class="termlist definitions" data-sort="ascending">
  <dt><dfn data-lt="registrations|replica registration|replica registrations">registration</dfn></dt>
  <dd>A record stored on a [=replica Space=] naming one [=source Space=] and
  carrying the [=pull capability=] that reads it. It is the directed edge
  along which the replica server pulls ([[[#registrations]]]).</dd>

  <dt><dfn data-lt="source Spaces">source Space</dfn></dt>
  <dd>The Space a [=registration=] names in <code>fromSpace</code>: the Space
  whose records the replica server reads.</dd>

  <dt><dfn data-lt="replica Spaces">replica Space</dfn></dt>
  <dd>The Space a [=registration=] is stored on, named in
  <code>toSpace</code>: the local Space the replica server writes what it
  reads into.</dd>

  <dt><dfn data-lt="source servers">source server</dfn></dt>
  <dd>The server hosting a [=source Space=], in the context of one
  [=registration=].</dd>

  <dt><dfn data-lt="replica servers">replica server</dfn></dt>
  <dd>The server hosting a [=replica Space=] and running its
  [=pull loop=], in the context of one [=registration=].</dd>

  <dt><dfn data-lt="pull capabilities">pull capability</dfn></dt>
  <dd>A delegated capability [[PWS-AUTHZ]] whose chain roots in the
  [=source Space=]'s root capability, whose <code>controller</code> is the
  [=replica server=]'s [=server DID=], and whose actions lie within
  <code>GET</code> and <code>HEAD</code>. The replica server signs every
  request of the pull loop as an invocation of it
  ([[[#capability-checks]]]).</dd>

  <dt><dfn data-lt="pull loops">pull loop</dfn></dt>
  <dd>The repeated cycle in which a [=replica server=] reads a
  [=source Space=]'s state and applies it ([[[#pull-loop]]]).</dd>

  <dt><dfn data-lt="write stamps|stamp|stamps|stamped">write stamp</dfn></dt>
  <dd>The three members <code>updatedAt</code>, <code>updatedAtCounter</code>
  and <code>originId</code> that [[PWS]] stores on every versioned record,
  minted at the server where the write originated, and the total order over
  them. A received record is applied or skipped by comparing stamps
  ([[[#stamp-comparison]]]).</dd>

  <dt><dfn data-lt="creating stamp|creating stamps">created stamp</dfn></dt>
  <dd>The [=write stamp=] of the write that created a Collection under its
  current generation, served by [[PWS]] as the Collection Metadata object's
  <code>created</code> member. It orders two lives of one Collection id
  ([[[#apply-collection-metadata]]]).</dd>

  <dt><dfn data-lt="server DIDs|serverDid">server DID</dfn></dt>
  <dd>A server's own <code>did:webvh</code> identity, hosted on the server
  itself at the path <code>space:server:id</code> and advertised as the
  <code>serverDid</code> member of its core [=version entry=]
  ([[[#server-identity]]]).</dd>

  <dt><dfn data-lt="replica pointers">replica pointer</dfn></dt>
  <dd>A service entry of type <code>https://w3id.org/pws#ReplicaServer</code>
  in an account's client-annex document [[APP-CONNECT]], naming one replica
  server by its [=server DID=]. A source server reads it when it decides
  whether to admit a credential-signed [=pull capability=]
  ([[[#pull-capability-signing]]]).</dd>

  <dt><dfn data-lt="stalls|stalled">stall</dfn></dt>
  <dd>The state of one Collection of a [=registration=] whose pull cannot
  advance past a record, with a durable reason served at the registration's
  <code>status</code> sub-resource ([[[#stalls]]]).</dd>

  <dt><dfn data-lt="version entries">version entry</dfn></dt>
  <dd>A JSON object under a specification's identifier in the service
  description's <code>specs</code> member, describing one version of that
  specification the server implements. Defined by [[PWS]].</dd>
</dl>

## Service description {#service-description}

A server tells clients which specifications it speaks through the PWS
[service description](https://w3c-ccg.github.io/portable-web-spaces-spec/#service-description):
a server-wide JSON document whose `specs` member maps each specification's
persistent identifier to the versions of it the server implements.

### This specification's identifier {#spec-identifier}

The persistent identifier of this specification is
`https://w3id.org/pws/replication`.

That string is the `specs` key a server lists this specification under. It
names the specification without a version. Clients compare the key as an
opaque string, with no URL normalization.

### The version entry {#version-entry}

A [=version entry=] under this specification's identifier carries the
members [[PWS]] defines for every entry and nothing else:

* `version` (REQUIRED) - The version of this specification the server
  implements, a bare `major.minor` string. This document is version `0.1`.
* `url` (OPTIONAL) - The URL of this document at exactly that version.

Listing the entry is the claim. A server that lists it serves the
registration sub-resource ([[[#registrations]]]), runs the pull loop
([[[#pull-loop]]]) and applies what it reads ([[[#apply]]]) for every
registration it holds. There is no feature token: nothing in this document
is optional for a server that implements it.

A server that lists no entry under the identifier holds no replica. It can
still be a source, since a pull is made of reads any PWS server answers.
A registration naming such a server as its source is refused
([[[#source-check]]]).

A change to what the pull loop accepts, to the registration record, or to the
apply rules is a new version of this specification. A replica server at one
version refuses a source that lists no version it speaks, with a recorded
reason, rather than misapplying what it reads.

### Members of the core entry {#core-entry-members}

Two members of the core `https://w3id.org/pws` [=version entry=], defined by
[[PWS]], are read by this document:

* `originId` - The id of the store that mints the server's write stamps. A
  registration is refused when the source advertises none, or advertises the
  replica server's own ([[[#source-check]]]).
* `serverDid` - The server's own DID ([[[#server-identity]]]). A controller
  delegates the pull capability to this DID. A server that advertises none
  cannot hold a replica.

Example: the `specs` member of a replica server's service description (other
members elided):

```json
{
  "specs": {
    "https://w3id.org/pws": [
      {
        "version": "0.5",
        "originId": "replica-eu-1",
        "serverDid": "did:webvh:Qm...:replica.example:space:server:id"
      }
    ],
    "https://w3id.org/pws/replication": [
      {
        "version": "0.1",
        "url": "https://w3c-ccg.github.io/portable-web-spaces-spec/replication/"
      }
    ]
  }
}
```

## Model {#model}

### The Space is the unit {#space-unit}

A registration names a whole Space. The replica server pulls the Space's own
state (its `name` and its policy), its Collection listing with tombstones,
and every Collection the registration selects, with the Collection holding
the controller's log always among them ([[[#registration-record]]]).

A Collection is not a unit of its own. Space-level state appears in no feed.
A Collection missing from a listing is indistinguishable from a revoked
grant or a lagging source. A new sibling Collection would be invisible to a
per-Collection registration. The `collections` member of a registration gives
the selectivity a deployment needs.

### Transport is pull only {#pull-only}

Every request of the pull loop is a `GET` (or a `HEAD`) the replica server
signs as an invocation of the pull capability. The source server answers it
as it answers any read. No route defined by [[PWS]], by this document, or by
any profile a conformant server implements accepts a write that carries a
stamp to be stored as given. A received record enters the replica through a
storage-level path that runs under the same locks and the same container
checks as a request-layer write, after the one stamp comparison of
[[[#stamp-comparison]]].

The feed the loop reads is the `changes` query profile's read-only form,
`GET /space/{space_id}/{collection_id}/query?profile=changes`, with
`checkpoint` and `limit` as query parameters, verified under the `GET`
action and answering what the `POST` form answers. A pull capability carries
no `POST`, since a `POST` grant on a Space would also create Collections and
Resources and run an import there.

### What is replicated {#what-is-replicated}

Replicated, by the rules of [[[#apply]]]:

* the Space Metadata object's `name`;
* the Space policy, Collection policies, and Resource policies, live or
  tombstoned;
* each selected Collection's Metadata object, including its tombstone;
* each Resource of a selected Collection, whatever its content type, its
  Resource Metadata object, its tombstone, and its chunk set;
* each selected Collection's governing history log, and a controller's
  `did.jsonl` log stored as a Resource.

Not replicated, because each is a fact about one server rather than about
the Space:

* `controller`. A Space is promoted to a `did:webvh` controller on each
  server separately, authorized by the `did:key` stored there. A replicated
  `controller` would hand a compromised or merely different source server the
  replica Space on the next pull, or deadlock the replica under a DID whose
  log has not arrived.
* Revocations. A revocation names a capability that is invocable only at the
  host its target embeds. Retiring a key through the controller's log is the
  cross-replica revoke.
* Backend registrations and the Collection `backend` selection, quotas, and
  keystores.
* Registrations themselves, and the `replicas` member they derive.

### Server identity {#server-identity}

A server that implements this document has an identity of its own: a
`did:webvh` [[DID-WEBVH]] whose log is self-hosted on the server at the path
`space:server:id`, so the DID has the form
`did:webvh:{scid}:{host}:space:server:id`. The server advertises it as the
`serverDid` member of its core version entry.

The server signs every request of the pull loop with the one key its
administrator lists in that log under both `assertionMethod` and
`capabilityInvocation`, as the verification method `{serverDid}#{key}`. There
is no second key and no derived key for replication. A controller therefore
delegates a pull capability to the server DID and learns no key, and the
administrator rotates the key in the log without any re-delegation.

A server with no server DID, or whose log lists no key under
`capabilityInvocation`, MUST refuse every registration
([[[#refusals]]]).

## Registrations {#registrations}

A registration is a sub-resource of the replica Space, under the reserved
`replicas` segment:

| Endpoint                                              | Methods                | Purpose                                              |
|-------------------------------------------------------|------------------------|------------------------------------------------------|
| `/space/{space_id}/replicas`                          | `GET`, `POST`          | List the Space's registrations; add one              |
| `/space/{space_id}/replicas/{replica_id}`             | `GET`, `PUT`, `DELETE` | Read, replace (pause, resume, renew), and remove one |
| `/space/{space_id}/replicas/{replica_id}/status`      | `GET`                  | The registration's runtime state                     |

A method not listed is answered as the
[Methods at Reserved Endpoints](https://w3c-ccg.github.io/portable-web-spaces-spec/#methods-at-reserved-endpoints)
rule of [[PWS]] requires. `HEAD` is implemented wherever `GET` is.

### The registration record {#registration-record}

A registration record is a JSON object with the following members. A client
writes the first six; the server adds the rest.

* `id` (REQUIRED) - The registration's id, unique within the Space, chosen by
  the client. It is a path segment and follows the identifier rules of
  [[PWS]].
* `fromSpace` (REQUIRED) - The canonical trailing-slash URL of the source
  Space, absolute.
* `toSpace` (REQUIRED) - The canonical trailing-slash URL of this Space,
  absolute. A value naming another Space is an `invalid-request-body`
  error.
* `role` (REQUIRED) - The string `source`. It names the direction of the edge
  and is the one value this version defines.
* `capability` (REQUIRED) - The pull capability: a delegated capability
  document as [[PWS-AUTHZ]] defines one, carried verbatim. Its constraints
  are in [[[#capability-checks]]].
* `collections` (OPTIONAL) - An array of `{ "id": "<collection_id>" }`
  objects naming the Collections to pull. Absent means every Collection of
  the source Space. The Collection holding the source's controller log, where
  the controller is a `did:webvh` hosted in the source Space, is pulled
  whether or not it is listed. Space-level state is pulled whatever the list.
* `pausedAt` (OPTIONAL) - Present when the registration is paused. Its value
  is an [[RFC3339]] `date-time` the server records when the pause is
  written; a client supplies the member to pause and omits it to resume, and
  the server ignores the value a client supplies. Absent means not paused
  ([[[#pausing]]]).
* `updatedAt`, `updatedAtCounter`, `originId` - The [=write stamp=] of the
  write that last stored the record, added by the server on the same terms
  as on every other stamped record of [[PWS]]. A value supplied in a request
  body is ignored.

The record's `ETag` is the stamp-derived validator [[PWS]] defines for a
Resource. `PUT` honors `If-Match` and `If-None-Match: *` as the Update (or
Create by Id) Resource operation does, answering `precondition-failed`
(412) when the precondition fails.

A member not defined here is ignored on write and not served.

Example record, as served:

```json
{
  "id": "primary",
  "fromSpace": "https://source.example/space/81246131-69a4-45ab-9bff-9c946b59cf2e/",
  "toSpace": "https://replica.example/space/3f1c2a9e-7b4d-4c61-9a0e-5d8b2f6c1e44/",
  "role": "source",
  "capability": {
    "@context": ["https://w3id.org/zcap/v1", "https://w3id.org/security/suites/ed25519-2020/v1"],
    "id": "urn:uuid:7b1f7a1e-5c2e-4f6a-9d3b-2e8c1a4f6d90",
    "controller": "did:webvh:Qm...:replica.example:space:server:id",
    "parentCapability": "urn:zcap:root:https%3A%2F%2Fsource.example%2Fspace%2F81246131-69a4-45ab-9bff-9c946b59cf2e%2F",
    "invocationTarget": "https://source.example/space/81246131-69a4-45ab-9bff-9c946b59cf2e/",
    "allowedAction": ["GET", "HEAD"],
    "expires": "2027-10-01T00:00:00Z",
    "proof": { "...": "..." }
  },
  "collections": [{ "id": "id" }, { "id": "messages" }],
  "updatedAt": "2026-10-29T15:04:05.000Z",
  "updatedAtCounter": 0,
  "originId": "replica-eu-1"
}
```

### Authorization {#registration-authorization}

Every registration route, the `status` sub-resource included, admits two
kinds of invocation under [[PWS-AUTHZ]]:

* a root invocation by the Space's controller;
* a delegated invocation whose chain roots in the Space's root capability,
  for `GET`, `POST`, `PUT`, and `DELETE`.

A delegated `POST` is admitted only when the registration it adds names a
`fromSpace` equal to the `fromSpace` of a registration already stored on
that Space. A delegated grant can therefore re-register a source the Space
already has, and cannot introduce a new one. A root `POST` is not bounded.
The bound exists because of phished credentials. Given an unbounded
delegated `POST`, the holder of one could register a source of its own
choosing that serves the account's DID as `controller`. That source's
tombstones would then be applied to the user's copy.

The delegated path exists so that an account with no enrolled client can
keep its replica alive. Such an account maintains its promoted replica Space
from a session that holds no key the account document lists under
`capabilityInvocation`. The predicate under which a source server admits
such a grant on the `replicas` route is the replicas-route maintenance
predicate of the client-annex clause of [[APP-CONNECT]].

Reads of the `replicas` listing and of the `status` sub-resource are not
public. A policy on the Space does not open them. A request with no
credential, or one whose credential does not reach the route, is answered as
[[PWS-AUTHZ]] answers any such read, and reveals nothing about the Space's
existence. A delegated `POST` that names a `fromSpace` not already registered
on the Space is a request lacking privilege, and is answered the same way.

### Storage {#registration-storage}

A registration is stored inside the Space it is on. It dies with the Space.
Delete Space removes every registration of the Space and stops its loops. It
is not replicated, so a source Space's registrations say nothing about its
replicas. It is not exported, so an archive of the Space carries none.

Storing or removing a registration advances the Space Metadata object's
validator, since the served `replicas` member changes. A `PUT` of a
registration does not, since none of the three members `replicas` lists can
change by `PUT`.

### List registrations {#list-registrations}

#### (HTTP API) GET `/space/{space_id}/replicas`

Answers a listing object of the form every listing in [[PWS]] takes,
`{ url, totalItems, items }`, whose `items` are the Space's registration
records as [[[#registration-record]]] serves them, capabilities included.
This version does not paginate it.

```http
GET /space/3f1c2a9e-7b4d-4c61-9a0e-5d8b2f6c1e44/replicas HTTP/1.1
Host: replica.example
Accept: application/json
Authorization: ...
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "url": "/space/3f1c2a9e-7b4d-4c61-9a0e-5d8b2f6c1e44/replicas",
  "totalItems": 1,
  "items": [ { "id": "primary", "fromSpace": "https://source.example/space/81246131-69a4-45ab-9bff-9c946b59cf2e/", "...": "..." } ]
}
```

Errors: `not-found` (404) when the Space does not exist or the caller is not
authorized; the two are indistinguishable.

The same relation is advertised in the Space linkset under
`https://w3id.org/pws#replicas`, and the Space Metadata object carries a
server-derived `replicas` member, `[{ fromSpace, toSpace, role }]`, listing
the registrations stored on that Space. Under a one-way pull the source Space
stores none and serves an empty array, and the replica Space names its
source.

### Add registration {#add-registration}

#### (HTTP API) POST `/space/{space_id}/replicas`

The body is a registration record. The server validates its shape, then runs
the checks of [[[#registration-checks]]], then stores it and starts its pull
loop. A registration carrying `pausedAt` is stored and no loop starts.

On success the server answers `201 Created` with the stored record as the
body, a `Location` header naming the registration's URL, and the record's
`ETag`. An `id` already in use on the Space is an `id-conflict` (409).

```http
POST /space/3f1c2a9e-7b4d-4c61-9a0e-5d8b2f6c1e44/replicas HTTP/1.1
Host: replica.example
Content-Type: application/json
Authorization: ...

{
  "id": "primary",
  "fromSpace": "https://source.example/space/81246131-69a4-45ab-9bff-9c946b59cf2e/",
  "toSpace": "https://replica.example/space/3f1c2a9e-7b4d-4c61-9a0e-5d8b2f6c1e44/",
  "role": "source",
  "capability": { "...": "..." }
}
```

```http
HTTP/1.1 201 Created
Location: https://replica.example/space/3f1c2a9e-7b4d-4c61-9a0e-5d8b2f6c1e44/replicas/primary
ETag: "z3kQ9m.1761750245000.0.replica-eu-1"
Content-Type: application/json

{ "id": "primary", "...": "..." }
```

Errors: `invalid-request-body` (400) for a body-shape fault, with a
`pointer` to the member; [=replica-refused=] (409) for a refusal over source
or server state ([[[#refusals]]]); `id-conflict` (409); `not-found`
(404).

### Read registration {#read-registration}

#### (HTTP API) GET `/space/{space_id}/replicas/{replica_id}`

Answers the stored record with its `ETag`. An unknown `replica_id` is
`not-found` (404), as is an unauthorized request.

### Replace registration {#replace-registration}

#### (HTTP API) PUT `/space/{space_id}/replicas/{replica_id}`

The body is a registration record whose `id` equals `replica_id`. Against a
stored record, only `capability` and `pausedAt` may differ; a body whose
`fromSpace`, `toSpace`, `collections` or `role` differs from the stored
value is an `invalid-request-body` (400) error at that member. Against no
stored record, `PUT` creates the registration as `POST` would, with the same
checks, and answers `201 Created`. Otherwise it answers `204 No Content` with
the new `ETag`.

`PUT` serves two purposes:

* Pause and resume. A body carrying `pausedAt` pauses the registration; a
  body omitting it resumes the loop from its checkpoint ([[[#pausing]]]).
* Capability replacement. A body whose `capability` differs from the stored
  one replaces the pull capability in place, keeping the registration's `id`
  and the loop's checkpoint. The new capability MUST pass every check a
  `POST`'s would ([[[#capability-checks]]], [[[#controller-check]]]), which
  the server confirms by reading the source through it before it stores the
  record. A body whose `capability` equals the stored one re-reads nothing.

Replacement in place exists because loop state is kept per registration id.
Renewing a capability by adding a second registration and removing the first
would restart the pull from an empty checkpoint and hold two registrations
of one source for the duration.

Errors: `invalid-request-body` (400); [=replica-refused=] (409);
`precondition-failed` (412); `not-found` (404).

### Remove registration {#remove-registration}

#### (HTTP API) DELETE `/space/{space_id}/replicas/{replica_id}`

Removes the registration and stops its loop. Answers `204 No Content`, and
is idempotent: a `DELETE` of an absent registration answers `204` too.

The server MUST refuse, with [=replica-refused=] (409), to remove the last
registration through which a `did:webvh` controller of a Space on this
server resolves ([[[#replicated-controller-logs]]]). Removing it would leave
that Space with no resolvable controller. A controller who wants the pull to
stop pauses the registration instead.

The copy stays. Removing a registration removes the edge, not the records the
replica Space holds.

### Registration status {#registration-status}

#### (HTTP API) GET `/space/{space_id}/replicas/{replica_id}/status`

Answers the registration's runtime state, under the same authorization as
the registration itself. The response MUST carry `Cache-Control: no-store`.
It is a JSON object:

* `state` - The loop state, one of `idle`, `pulling`, `backing-off`,
  `stalled`, `paused`. `pulling` while a cycle is in flight; otherwise
  `paused` when the registration is paused; otherwise `backing-off` after a
  failed cycle until the next attempt; otherwise `stalled` when any
  Collection is stalled; otherwise `idle`.
* `lastPullAt` (optional) - When the last cycle started.
* `lastSuccessAt` (optional) - When the last cycle completed without a
  failure.
* `nextPullAt` (optional) - When the next cycle is due. Absent while paused.
* `collections` - An array, one item per Collection the loop has seen, each
  with:
  * `id` - The Collection id.
  * `state` - One of `synced`, `syncing`, `stalled`, `skipped`.
  * `lastAppliedAt` (optional) - When a record of the Collection was last
    applied.
  * `stall` (optional) - Present while `state` is `stalled`:
    `{ reason, since, detail }`. `reason` is a token from the closed set of
    [[[#stalls]]], `since` is when the stall began under that reason, and
    `detail` is a human-readable string.

Times are [[RFC3339]] `date-time` strings.

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "state": "stalled",
  "lastPullAt": "2026-10-29T15:10:00.000Z",
  "lastSuccessAt": "2026-10-29T15:10:01.200Z",
  "nextPullAt": "2026-10-29T15:10:06.200Z",
  "collections": [
    { "id": "id", "state": "synced", "lastAppliedAt": "2026-10-29T15:10:01.100Z" },
    {
      "id": "messages",
      "state": "stalled",
      "lastAppliedAt": "2026-10-29T15:09:56.000Z",
      "stall": {
        "reason": "clock-bound",
        "since": "2026-10-29T15:10:01.150Z",
        "detail": "A received write stamp is dated 2026-10-29T15:12:30.000Z, more than 60 seconds ahead of local time."
      }
    }
  ]
}
```

## Registration checks {#registration-checks}

Before a replica server stores a registration, or stores a replacement
capability, it runs the checks below. A registration that fails any of them
is refused ([[[#refusals]]]). The checks read the source server, through the
pull capability where the read is of the Space, so a capability the source
server refuses fails the registration at the first such read.

### The source check {#source-check}

The replica server reads the [=source server=]'s service description, which
needs no credential, and requires:

* an entry under `https://w3id.org/pws/replication` whose `version` is one
  this server speaks;
* an `originId` on the source's core entry;
* that `originId` differs from this server's own. A source carrying this
  server's origin id is this server, or a clone of its store, and the stamps
  the two would exchange are indistinguishable.

The replica server also records whether the source lists [[PWS-EC]], since
that decides whether it reads chunk sets ([[[#apply-chunks]]]).

### The controller check {#controller-check}

The replica server reads the source Space's Metadata object through the pull
capability and compares the source's `controller` with the replica Space's.
The check passes in exactly one of three branches:

1. Equality. The source Space's `controller` equals the replica Space's.
2. The listed-key branch. The source Space's `controller` is a `did:webvh`
   whose log is hosted in the source Space itself, and whose current
   document lists the replica Space's `did:key` controller under
   `capabilityInvocation`. The replica server reads the log's `did.jsonl`
   through the pull capability and verifies it offline, as [[PWS]]'s
   verified-log rules require, and requires the registration to pull the
   Collection holding the log. A key listed under `capabilityDelegation`
   alone does not pass this branch.
3. The bootstrap branch. The replica Space's controller is a `did:key`. The
   source Space's `controller` is a `did:webvh` this server can resolve,
   either through the pull capability's read of a log hosted in the source
   Space, or from a local copy that another registration on this server maps
   ([[[#replicated-controller-logs]]]). The pull capability's chain is
   exactly `[root, capability]`, and its delegation proof's verification
   method is one that the resolved document lists under
   `capabilityDelegation`. No further binding between the replica Space's
   `did:key` and the capability is required.

The bootstrap branch lets a wallet create the replica Space under a `did:key`
it derives for the purpose, register the source, and promote the replica
Space to the account's `did:webvh` after the first pull has brought the log
over. Its local-copy arm covers an auxiliary Space whose controller's log
lives in another Space of the same account: the check resolves that
controller from the account replica's copy. The one-link condition refuses a
re-delegated grant, and the source server's own admission of the capability
([[[#capability-checks]]]) is what gates a credential-signed one
([[[#pull-capability-signing]]]).

### The Space checks {#space-checks}

* The source Space's `type` array, as a set, MUST equal the replica Space's.
  The Space ids need not be equal.
* The replica Space MUST NOT be the server's own `server` Space, the one
  hosting its server DID.
* For each Collection the registration selects that both Spaces hold, the
  creation-only members of its Collection Metadata object (the `encryption`
  scheme, `revisions.resolution`, `revisions.immutable`) MUST be equal on
  both sides, as the merge rule of [[[#apply-collection-metadata]]] would
  otherwise stall it at the first pull. The replica server reads each such
  Metadata object through the pull capability. A Collection absent on either
  side is not compared.
* Once one registration of a source Space stands on this server, a second
  registration of the same source Space that selects a Collection the first
  one selects MUST be refused, on this or any other local Space. Two loops
  writing one copy of a Collection, or two copies of one controller log, give
  the registration-mapped resolution of [[[#replicated-controller-logs]]]
  two answers.

### The capability checks {#capability-checks}

The `capability` member MUST satisfy all of the following:

* Its chain's first element is the root capability of the source Space, as
  [[PWS-AUTHZ]] forms it from the source Space's canonical URL.
* Its `invocationTarget` is the source Space's canonical trailing-slash URL,
  equal to `fromSpace`, or that URL's subtree form (the URL followed by `*`)
  once [[PWS-AUTHZ]] defines the explicit subtree scope on
  `invocationTarget`. A replica server MUST accept both forms where the
  authorization profile defines both. Under a target that covers the Space
  URL alone, every `GET` beneath it is refused, and the registration fails at
  the first read.
* Its `allowedAction` is present, non-empty, and within `{ GET, HEAD }`.
* Its `controller` is this server's server DID.
* The source server admits it. The replica server confirms this by reading
  the source through it during these checks.

<div class="ednote">
The subtree form of `invocationTarget` is pending in [[PWS-AUTHZ]]. Until it
lands, a target is the Space URL itself and covers the Space's subtree under
that profile's current prefix rule.
</div>

### Refusals {#refusals}

A registration whose body is malformed (a missing or mistyped member, an
`id` that is not a valid path segment, a `toSpace` naming another Space, a
`capability` that fails a shape rule of [[[#capability-checks]]]) is answered
with `invalid-request-body` (400), with a `pointer` to the member.

A registration refused over source or server state is answered with
[=replica-refused=] (409), with the cause in the problem's `detail`. The
causes are:

* this server has no server DID, or its log lists no key under
  `capabilityInvocation`;
* the source lists no replication entry at a version this server speaks, or
  no `originId`;
* the source advertises this server's own `originId`;
* the controller check passes in no branch;
* the source Space's `type` set differs;
* a selected Collection's creation-only members differ;
* the replica Space is the `server` Space;
* another registration of the source Space on this server shares a selected
  Collection;
* the capability is not delegated to this server's DID, or the source server
  does not admit it;
* the source server could not be read.

The same kind answers the two refusals that protect a mapped controller log:
removing the last registration that maps it ([[[#remove-registration]]]),
and an Update Space that would change the replica Space's `controller` to a
DID the server could then not resolve.

## The pull loop {#pull-loop}

Each registration that is not paused runs a pull loop on the replica server.
The loop runs a cycle, waits an interval, and runs the next. A cycle is a
sequence of signed reads of the source followed by the apply of
[[[#apply]]], one Collection at a time. The server MAY run one cycle at a
time per registration and MUST NOT run two concurrently against one
registration.

A cycle begins at once when a registration is created or resumed, or when
its capability is replaced.

### Discovery channels {#discovery}

The loop learns what changed through three channels of [[PWS]]:

* Conditional `GET`s of the Space Metadata object and the Space policy, with
  `If-None-Match` carrying the validator last seen, so an unchanged object
  costs a `304`.
* The tombstone-aware Collection listing,
  `GET {fromSpace}?include=deleted`, following `next` pages, which lists
  live Collections and tombstones with their stamps.
* Each selected Collection's `changes` feed, through its `GET` form, which
  lists every Resource write and tombstone whatever its content type, each
  Collection Metadata write, each Collection and Resource policy write or
  tombstone, and each governing-log append, each under its `kind`.

A `next` link that leaves the source Space ends the cycle as a failure.

### The cycle {#cycle}

A cycle proceeds in this order.

1. Read the source Space's Metadata object, conditionally. Apply `name`
   ([[[#apply-space]]]).
2. Read the Space policy with `?include=deleted`, conditionally, accepting
   `404` as "no policy". Apply it ([[[#apply-policies]]]).
3. Read the Collection listing with `?include=deleted`.
4. Apply every tombstone in the listing that names a selected Collection
   ([[[#apply-collection-tombstones]]]).
5. Pull the Collection holding the source controller's log first, if there is
   one, then every other selected Collection in the listing's order.

Pulling one Collection:

1. When the Collection's generation, read from its Metadata object's `ETag`,
   is not the one the loop holds for it, read the Metadata object and apply
   it ([[[#apply-collection-metadata]]]). A `404` leaves the Collection
   pending for the next cycle.
2. Read the feed page by page:
   `GET {fromSpace}{collection_id}/query?profile=changes&limit={n}`, with
   `&checkpoint={checkpoint}` once one is held. For each change document, by
   its `kind`:
   * `resource`: read the Resource's representation (or take the inline
     `data`), read its `/meta`, read its chunk set if the source lists
     [[PWS-EC]] ([[[#apply-chunks]]]), and apply ([[[#apply-resources]]]).
   * `collection-metadata`: re-read the Metadata object and apply it.
   * `policy`: read the policy at the document's `id` with
     `?include=deleted` and apply it.
   * `log`: read the log at the document's `id` and apply it
     ([[[#apply-logs]]]).
   * any other `kind`: skip the document.
   After each document is applied or skipped, hold its `checkpoint`.
3. The Collection is done when a page holds fewer documents than `limit`.

A read whose `ETag` does not match the change document's `etag` is a record
that moved under the loop. The loop re-reads it or leaves it for the next
cycle rather than applying a representation the feed did not describe.

A `400` answering a feed read that carried a checkpoint means the source's
feed is in another generation: the Collection was re-created, or the
checkpoint the loop holds was issued for a feed that no longer exists. The
loop drops the Collection's held generation and checkpoint and reads its
Metadata object and feed from the beginning on the next cycle. This is safe
because every record is applied by stamp comparison and keyed by id.

A page too large to buffer is retried with a smaller `limit`, down to `1`.
A single document that still cannot be buffered stalls the Collection with
reason `quota-exceeded`.

The checkpoint is held per Collection and persisted at least at the end of
each Collection's pull, so a cycle interrupted by a restart resumes near
where it stopped. It survives a capability replacement and a pause. It is
reset by a generation change and by the feed `400` above.

### Failure and backoff {#backoff}

A request to the source server that fails, other than inside a chunk set's
reads ([[[#apply-chunks]]]), ends the cycle. A `404` for the Space counts as
a failure: under the maximum-privacy rule of [[PWS]] it is what a revoked or
expired pull capability looks like, and the loop cannot tell it from an
absent Space. The loop then backs off, doubling its delay from a base up to
a limit, and tries again. The reference values are a 5-second poll
interval, a 5-second base, and a 10-minute limit. A successful cycle resets
the delay. A registration created, resumed, or renewed is attempted at once,
backoff or not.

A failure ends the cycle and leaves no stall: the Collections' states are as
they were, and `status.state` is `backing-off`.

### Pausing {#pausing}

A paused registration makes no request to the source. Its loop state, the
checkpoints included, is retained, and its registration-mapped resolution
([[[#replicated-controller-logs]]]) stays in force, so a `did:webvh`
controller whose log the registration copies keeps resolving on this server
from the frozen copy. `status.state` is `paused`.

Clearing `pausedAt` by `PUT` resumes the loop from its checkpoints, with no
carried-over backoff.

The pause exists because removal cannot always be the way to stop a loop.
The last registration mapping a promoted Space's controller cannot be removed
([[[#remove-registration]]]), and a loop left running against a source that
refuses it backs off on `404` forever.

## Applying received records {#apply}

A received record is not stored verbatim. The loop first reduces the
representation it read to the record's stored projection, dropping every
server-derived member, and then decides by kind whether and how the
projection applies. A record that does not apply is skipped, and the loop
advances past it. A record that cannot be decided stalls its Collection
([[[#stalls]]]).

The apply runs under the same locks and the same container checks as a
request-layer write. A write into a Collection that no longer exists, or
that a tombstone now covers, is refused by the container check like any
other. The loop treats the refusal as [[[#apply-collection-tombstones]]]
says.

### The stamp comparison {#stamp-comparison}

Every stamped record is decided by one comparison. The received stamp and
the held stamp are ordered by `(updatedAt as milliseconds, updatedAtCounter,
originId)`, numerically on the first two and by plain string comparison on
the third, which is the total order [[PWS]] defines. A received record
applies when its stamp is strictly greater than the held one. An equal or
lesser stamp is skipped, which is also what keeps a record the replica
itself originated from coming back changed. A record with no held
counterpart applies.

A received record whose stamp is missing a member, or whose members are not
a valid stamp, is skipped.

Clock bound. A receiver MUST NOT apply a stamp whose `updatedAt` is more than
a bound ahead of its own physical time. The RECOMMENDED bound is 60 seconds.
A stamp within the bound advances the receiver's clock by the hybrid logical
clock receive rule, so the receiver's next own write stamps after it. A stamp
past the bound stalls the Collection with reason `clock-bound` at that
position; the stall clears as local time catches up, since the record is
retried each cycle.

### The Space Metadata object {#apply-space}

The stored projection of a received Space Metadata object is `name` alone.
`url`, `linkset`, `backends`, `replicas`, `createdBy`, `controller` and
`type` are stripped. `name` applies under the stamp comparison on the Space
Metadata object's stamp, and an absent `name` on the source clears the
replica's. The replica Space's `controller` and `type` are its own
([[[#what-is-replicated]]]); `type` is held equal by the registration check.

### Collection Metadata {#apply-collection-metadata}

The stored projection of a received Collection Metadata object strips `url`
and `linkset`, and on a Collection whose descriptor is governed by its
history log, strips the derived `encryption` and `revisions` members too,
since the replica derives them from its own copy of the log. The received
object MUST carry a generation (read from its `ETag`), a stamp and a
[=created stamp=]; one that does not is skipped.

The decision reads the held record, if any, and the generations:

* Nothing held, or a held tombstone whose stamp is less than the received
  `created` stamp: create the Collection under the received generation.
* The same generation held: apply under the stamp comparison, merging as
  below.
* Another live generation held: a generation change is a delete plus a
  create. When the received `created` stamp is greater than the held one,
  the held Collection is tombstoned, its members cascade-removed, and the
  received object is created under its generation. Otherwise the received
  object is skipped. Two creates in one millisecond on two origins are
  ordered by `originId`; an exact tie is broken by the greater generation
  string.

Merging an update onto the same generation:

* `backend` keeps the held selection; the source's is a fact about the
  source server.
* The creation-only members (the `encryption` scheme, `revisions.resolution`,
  `revisions.immutable`) merge forward: a received value equal to the held
  one, or absent, keeps the held one. A received value that differs is a
  fork, and the Collection stalls with reason `fork`.
* `encryption.version` takes the greater of the two values.
* Every other member applies as received.

A received `backend` selection a replica cannot honor is not an error, since
it is not applied. A Collection whose own held `backend` names a backend
this server does not configure stalls with reason `unsupported-backend`
before anything is stored.

### Collection tombstones {#apply-collection-tombstones}

A Collection tombstone in the source's listing carries the deleting write's
stamp and no generation. The decision:

* Nothing held: write the tombstone.
* A live Collection held whose `created` stamp is less than the tombstone's
  stamp: tombstone it and cascade-remove its members, whatever the stamps of
  those members. This is the delete-wins rule, and it is the one stated
  exception to greater-stamp-wins: a write at a later stamp into a Collection
  deleted at an earlier stamp does not resurrect the Collection. A controller
  who wants the member back re-creates the Collection and the member.
* A live Collection held whose `created` stamp is greater than the
  tombstone's: skip. A life created after the delete is not deleted by it.
* A tombstone held with a lesser stamp: take the received stamp.
* Otherwise skip.

A member write received for a Collection the replica holds as a tombstone is
skipped and the checkpoint advances. The container check refuses it, and the
loop treats that refusal as a skip rather than a stall, since the delete
wins.

### Resources and their Metadata {#apply-resources}

A received `resource` change document MUST carry a stamp, a generation, and a
`contentType`; one that does not is skipped. The Resource's content record
applies under the stamp comparison on the document's stamp, a tombstone
(`deleted: true`) exactly as a live revision does. The Resource's Metadata
object is its own record with its own stamp, nested as `meta` on the change
document and served by the `/meta` sub-resource; it applies second, under
its own comparison. A `/meta` read whose stamp differs from the one the
change document carries is a record that moved, and the document is
retried.

A replica applies the representation it read, bytes and content type, and
stores the source's stamp and generation with it, so the Resource's `ETag` is
byte-identical on every replica and a client's `If-Match` works against any
of them.

### Chunked Resources {#apply-chunks}

A chunk write takes no feed position. A replica server that pulls from a
source listing [[PWS-EC]] discovers a Resource's chunk set when the
Resource's `resource` change document applies. It then:

1. reads the Resource's chunk listing
   (`GET .../{resource_id}/chunks/`);
2. reads each listed chunk with a conditional `GET`, skipping a chunk whose
   source `ETag` it already holds staged for this Resource revision;
3. reads the parent Resource again, last. A parent whose `ETag` is no longer
   the change document's was rewritten during the reads, and the document is
   retried.

The set applies as one unit with the parent revision that won the stamp
comparison. A chunk carries no received stamp, and a held chunk's validator
never decides against an incoming set: a held parent revision that wins or
ties already has its set, and only the `/meta` record is then applied. A
chunk-only mutation on the source (a chunk written or deleted with no parent
write) does not replicate until the parent is next written, when the listing
carries the result. A chunk delete leaves no tombstone.

The source's chunk `ETag` is opaque to the replica and used for the
conditional `GET` alone. The replica mints its own validator for each chunk
it stores. A source that does not list [[PWS-EC]] serves no chunk listing,
and the replica reads none from it.

A request inside a chunk set's reads that fails does not end the cycle. It
is retried, and after a bounded number of consecutive retries of the same
document at the same checkpoint (the reference value is 5) the Collection
stalls: with reason `chunk-listing-obsolete` when the parent kept moving
under the reads, cycle after cycle, and `source-unavailable` when a request
kept failing. How a replica holds a set before it applies is its own
concern.

### Policies {#apply-policies}

A Space, Collection, or Resource policy, read with `?include=deleted`,
applies under the stamp comparison on its own stamp, live or tombstoned,
with its generation taken from its `ETag`. A policy tombstone is stored as a
tombstone, so a deleted public-read policy does not return from a replica
that still held it.

### Logs {#apply-logs}

A governing history log, and a controller's `did.jsonl` stored as a
Resource, are not decided by stamp. The received log is compared with the
held one as byte sequences:

* nothing held, or the held log is a strict prefix of the received one:
  apply (a fast-forward);
* the received log equals the held one, or is a prefix of it: skip (a no-op);
* neither is a prefix of the other: a fork. The Collection stalls with
  reason `fork`, and stays stalled until the controller resolves it, since
  the two copies have diverged in a way no stamp can order.

A received deletion of a `did.jsonl` Resource is skipped. A log stored under
a registration gets the same verification, caching, head record, and
fast-forward rules the server applies to any self-hosted log, so a fork is
permanent on both sides.

### Stalls {#stalls}

A [=stall=] is per Collection. The stalled Collection's checkpoint holds at the
record that stalled it, the record is retried each cycle, and every other
Collection of the registration continues. The stall is served at the
`status` sub-resource as `{ reason, since, detail }`
([[[#registration-status]]]), with `since` kept while the same reason stands.

`reason` is one of the following tokens, and the set is closed for this
version:

| `reason`                  | Meaning                                                                                                  |
|---------------------------|----------------------------------------------------------------------------------------------------------|
| `clock-bound`             | A received stamp is dated past the clock bound ([[[#stamp-comparison]]]). Clears as local time catches up. |
| `fork`                    | A log, or a creation-only Collection member, diverged ([[[#apply-logs]]], [[[#apply-collection-metadata]]]). |
| `quota-exceeded`          | The replica's backend refused the write for want of space, or a single document could not be buffered.  |
| `unsupported-backend`     | The held Collection selects a backend this server does not configure.                                   |
| `container-refused`       | The apply path refused the write at the container, other than under a tombstone.                        |
| `chunk-listing-obsolete`  | A chunk set was rewritten on the source faster than the replica could read it, cycle after cycle.        |
| `source-unavailable`      | A request to the source kept failing inside a chunk set's reads.                                         |

A reader MUST treat a `reason` it does not recognize as "stalled for an
unrecognized reason". A new token is a new version of this specification.

## Replicated controller logs {#replicated-controller-logs}

A `did:webvh` controller whose log is hosted in a Space resolves on the
source server from the server's own storage, under the Self-hosted histories
rule of [[PWS]]. The same controller resolves on a replica server from the
replicated copy. That resolution is still a local storage read, with no
network fetch.

A DID with host `H` and path `space:S:C` resolves on a server when exactly
one registration on a local Space `X` names `https://H/space/S/` as its
`fromSpace` and selects Collection `C`. The copy is then read from the
Resource `did.jsonl` of Collection `C` of Space `X`. `X` need not have the
id `S`. A DID with zero or more than one such registration does not resolve
from a copy. The copy is verified, cached, and fast-forwarded as any
self-hosted log is.

This is what lets a replica Space be promoted to the account's `did:webvh`
after its first pull, and keep accepting root invocations by the account's
enrolled keys while the original host is unreachable. It is also why the two
refusals of [[[#refusals]]] protect the mapping: removing the last mapping
registration, or changing a Space's `controller` to a DID that would then
not resolve, would leave a Space with no controller.

A paused registration keeps its mapping ([[[#pausing]]]). A key retired at
the source after the pause keeps its reach on the frozen copy until the
registration resumes or the Space is deleted.

Resolution from the original host over the network is not a substitute. If
that host is lost for good, a cached document can never refresh and the
account can never rotate a key again. The replicated copy is what keeps the
controller resolvable on the surviving server.

## Signing the pull capability {#pull-capability-signing}

A pull capability is delegated by a verification method the source Space's
controller document lists under `capabilityDelegation`. For a wallet account
that is one of two kinds of key:

* an enrolled client's key, listed in the account document. A pull it signs
  is an ordinary delegation, and the source server admits it as it admits
  any delegated read.
* a standing credential's ladder verification method, so that an account
  with no enrolled client, visited only by transient sessions, can still
  register a replica. The source server admits a credential-signed pull
  only under the replica-pull predicate of the client-annex clause of
  [[APP-CONNECT]]. In outline, the predicate requires that:
  * the delegation's sole `controller` equals the `serviceEndpoint` of a
    [=replica pointer=] in the account's client-annex document;
  * its target is the account Space, or the client-annex auxiliary Space
    the account document names;
  * its actions lie within `{ GET, HEAD }`;
  * its parent is the Space's root.

A [=replica pointer=] is a service entry in the client-annex document whose
`type` is
`https://w3id.org/pws#ReplicaServer`, one per replica server. Its
`serviceEndpoint` is the replica server's bare server DID. Its `id` fragment
is `#replica-<multihash>`, where `<multihash>` is the SHA-256 multihash of
the `serviceEndpoint` string as UTF-8 bytes, encoded base64url with no
padding, so the id is the same each time the pointer is restated and a
reader may match a server's entry by id or by endpoint. A reader dispatches
on `type`. The entry carries nothing beyond `serviceEndpoint`.

```json
{
  "id": "did:webvh:Qm...:source.example:space:81246131-69a4-45ab-9bff-9c946b59cf2e:annex#replica-EiB3v...",
  "type": "https://w3id.org/pws#ReplicaServer",
  "serviceEndpoint": "did:webvh:Qm...:replica.example:space:server:id"
}
```

The source server reads a pointer as a fact of a verified log, when it
evaluates the predicate on a delegation it is already verifying. It fetches
nothing from the DID a pointer names, and acts on no pointer as a locator.
Writing a pointer extends the annex log under the credential's authority, so
a credential-signed copy of the account always leaves a logged record.
Removing the pointer ends that server's credential-signed pull of the
account Space.

<div class="ednote">
The normative text of the replica-pull predicate and of the replicas-route
maintenance predicate is held by [[APP-CONNECT]] as decision records ahead
of its profile text. This document cites the predicates by name and does
not restate them.
</div>

## Verifying a replica server's signature {#verifying-replica-server}

Every request of the pull loop is signed by a server DID of another host.
The source server verifies it as [[PWS-AUTHZ]] verifies any invocation, with
one addition: resolving the invoker's DID requires a network fetch of that
server's log, since the log is hosted on the replica server. This is the one
bounded network fetch replication asks of a source server.

A source server MUST fetch a `did:webvh` log from another host only when all
of the following hold:

* the DID is the `controller` of a delegated capability whose chain has
  already verified to the Space's controller, so the fetch is made for a
  party the controller has named;
* the DID's path is `space:server:id`, so it is a server identity;
* the fetch is over `https` on the default port, with no redirect followed;
* the host does not resolve to a loopback, private, link-local, metadata or
  otherwise reserved address range, and the connection is made only to an
  address that passed that check;
* the fetch is rate-limited per host and bounded in concurrency, with a
  request timeout and a body size cap;
* the result is verified as the method requires, cached with a time to live,
  revalidated after it, and refused if it does not extend the last verified
  head.

A DID of any other method hosted elsewhere, and a `did:webvh` that fails any
of the conditions, stays refused under the controller rules of [[PWS]]. The
reference values are a 5-second timeout, a 1 MiB body cap, a 300-second
cache, ten first-contact fetches per host per minute, and eight concurrent
fetches in all.

The foreign server may invoke the capability it is named in. It may not
delegate on, since the pull capability's `controller` is verified at the
leaf and a further link would make the replica server a delegator of the
controller's authority.

## Capabilities stay host-bound {#host-bound-capabilities}

A capability's `invocationTarget` embeds one server's URL, and replication
does not change that. A controller who wants a grant to work on a replica
delegates it again against the replica's URL, every kind of grant included:
app Collection grants, management capabilities, the pull capability itself.
Each is revoked on the replica separately, since revocations are not
replicated ([[[#what-is-replicated]]]). Until the controller has
re-delegated a grant on a replica, only root invocations by the controller's
own keys work there, which the replicated controller log makes possible.

## Errors {#errors}

This document registers one kind in the
[Error Type Registry](https://w3c-ccg.github.io/portable-web-spaces-spec/#error-type-registry)
of [[PWS]], and otherwise reports with the kinds registered there:

* <dfn data-lt="replica-refused">`replica-refused`</dfn> (409) - a
  registration, a capability replacement, the removal of a registration, or
  a Space `controller` change was refused over source or server state, with
  the cause in `detail` ([[[#refusals]]]).
* `invalid-request-body` (400) - a registration body is malformed, with a
  `pointer` to the member.
* `id-conflict` (409) - a `POST` names a registration `id` already in use.
* `precondition-failed` (412) - a `PUT`'s `If-Match` or `If-None-Match: *`
  evaluated false.
* `not-found` (404) - the Space or registration does not exist, or the
  caller lacks authorization; the two are indistinguishable under the
  maximum-privacy rule of [[PWS]].

A failed read of the source inside the pull loop is not an error the replica
server reports to anyone. It is served as state at the `status`
sub-resource.

```json
{
  "type": "https://w3id.org/pws#replica-refused",
  "title": "Replica registration refused.",
  "status": 409,
  "detail": "The source carries this server's own origin id \"replica-eu-1\"."
}
```

## Security considerations {#security-considerations}

### Read authority is all a replica holds {#read-authority}

The pull capability carries `GET` and `HEAD` alone, and the loop invokes
nothing else. A replica server holds no write authority on its source, and a
compromised replica server can disclose what it has read but cannot alter
the source. In the other direction, a source server can serve a replica
anything, which is why the apply path verifies logs, stamps, and container
state rather than trusting the representation.

### What is not replicated protects the replica {#not-replicated-protects}

A replicated `controller` would let a source server take over the replica
Space on the next pull. A replicated revocation set would let it inject
unverified records into the replica's authorization state. Both stay
per-server ([[[#what-is-replicated]]]). The price is the per-replica grant
inventory of [[[#host-bound-capabilities]]].

### The bootstrap key {#bootstrap-key}

Until a replica Space is promoted, its bootstrap `did:key` is root over the
copy, and any party able to derive that key can read or delete it. It cannot
touch the source. A wallet promotes the replica Space as soon as the first
pull has brought the controller's log over, and a replica left unpromoted by
an interrupted run is finished by the next session that holds the key. The
bootstrap branch requires no binding between that key and the pull
capability, so a pull capability that leaks lets its holder register a copy
of the source under a key of their choosing on the server the capability
names, readable by them until the controller revokes the capability at the
source. A credential-signed pull capability is bounded by the replica
pointer, so such a copy is always preceded by a logged pointer write.

### The copy verifies against a copy {#copy-verifies-copy}

After promotion a replica server verifies every invocation on the replica
Space, the `replicas` route included, against its own copy of the account
log, which advances only by pulls. A key enrolled or a credential
established at the source reaches the replica only once the copy holds its
entry. A key retired at the source keeps its reach on the replica until the
retirement is pulled, and indefinitely on a paused registration's frozen
copy.

### A fork is permanent {#fork-permanent}

Two copies of a log that are not prefixes of one another cannot be ordered
by this document. The Collection stalls on both sides and stays stalled. A
controller resolves a fork by choosing a copy and re-creating the Collection
from it. The one-key signing rule, the one-link chain of the bootstrap
branch, and the refusal of two registrations sharing a Collection exist in
part to keep forks rare.

### The outbound fetch {#outbound-fetch}

The source server's fetch of a replica server's log is an outbound request
to a host named, indirectly, by a capability a controller signed. The bounds
of [[[#verifying-replica-server]]] keep it from being abused for request
forgery. The address-range refusal keeps it off internal networks. The path
restriction keeps it to server identities. The rate limit and concurrency
bound keep a flood of registrations from becoming a flood of fetches. The
chain condition means an attacker must first obtain a delegation from the
Space's controller.

### Clocks {#clocks}

The stamp order is a hybrid logical clock. A source whose clock runs far
ahead would, without the clock bound, make every replica's later writes lose
to its earlier ones. The bound of [[[#stamp-comparison]]] stalls such a
source's Collection rather than applying it, at the cost of stalling an
honest source whose clock is more than the bound ahead until the replica's
clock catches up.

## Privacy considerations {#privacy-considerations}

### The served `replicas` member {#replicas-member-disclosure}

The `replicas` member of a Space Metadata object lists the registrations
stored on that Space, and nothing else. A source Space serves an empty array,
so nothing a source serves tells one replica about another. The member's
readers are the controller's own sessions and holders of a Space-wide grant,
who already read the Space.

### Replica pointers {#replica-pointer-disclosure}

The replica pointers live in the client-annex document, which is
capability-gated and read by the account's own sessions, its storage host,
and its replica servers. Once the client-annex auxiliary Space is
replicated, every replica server reads the pointers naming every other
replica server by DID, and the source host reads them whenever it evaluates
the replica-pull predicate. The source host already learns each pulling
server's DID from the invocations it verifies, so what the pointers add is
that the replica servers learn about one another. This is accepted. A
pointer in the account document would instead disclose the set of replica
servers to anyone, since that document is world-readable and permanent. A
blinded pointer would hide nothing from the one party that must read it.

### A copy is a copy {#copy-disclosure}

A replica server holds every record the pull capability can read, under its
own operator's control. A controller choosing a replica server is choosing a
second custodian of the plaintext of every unencrypted Collection it
replicates, and of the ciphertext of every encrypted one. Encrypted
Collections [[PWS-EC]] keep their contents opaque to both servers.

### The source learns the replica {#source-learns-replica}

A source server learns the DID of each server that pulls from it, and when
each pulls, from the invocations it verifies. A replica server learns
nothing about its source beyond what the pull capability lets it read.
