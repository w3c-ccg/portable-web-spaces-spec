<div class="remove">

# Portable Web Spaces Authorization Profile v0.1

**Abstract:** The baseline authorization profile for Portable Web Spaces:
object capabilities (zCaps) invoked over HTTP Signatures, rooted in a Space's
controller DID, together with the `PublicCanRead` access control policy.

**Status:** Experimental W3C CCG draft, undergoing regular revisions.
Rendered version: https://w3c-ccg.github.io/portable-web-spaces-spec/authz-profile/

</div>

## Introduction {#introduction}

Portable Web Spaces (PWS) [[PWS]] is a permissioned storage API. Its core
specification defines the contract every authorization profile meets (private
by default, the Space `controller` as the root of trust, and "not found" in
place of "not authorized") and the socket a profile plugs into (the `/policy`
auxiliary resource). It leaves the mechanism to an authorization profile. This
document is that profile for the baseline: it defines how a caller proves it may
act on a PWS target, and the one access control policy type that lets a
controller open a target to callers in general.

Like many authorization specifications, this profile tries to address opposing
tensions. On the one hand, to cover the full range of use cases, it needs to be
delegatable, revocable, secure, flexible, and thus capability-based. On the
other hand, for ease of implementation and adoption, and for maximum developer
usability, the profile must make the most common operations as simple and
friction-free as possible.

To that end, the profile offers the following layered mechanisms.

1. **Root Access**: For basic admin CRUD operations, use the Space's `controller`
   DID directly to sign API calls with HTTP Signatures.
2. **Public Read**: For the common "public read" use case (the typical web
   publishing workflow, where a site or a file is shared for anyone to access
   via an HTTP GET), use the simple `{ "type": "PublicCanRead" }` policy, see
   [[[#publiccanread]]].
3. **Advanced Delegatable Capabilities** ("anyone with the link..." style):
   Use [=zCaps=] [[ZCAP]], see [[[#delegation]]].
4. **Policy Based Access Control** (including the familiar "share with this list
   of people or groups" style): Use the Space's `linkset` property to point to
   a linkset that includes a URL to an access control policy document. This
   version of the profile defines one policy type; the socket is defined by
   [[PWS]].

The [[ZCAP-GUIDE]] is the explanatory companion to this profile. It walks
through how capabilities are delegated, invoked, and verified, and how a root
of trust is established. This document states the requirements; the guide
explains them.

### Relationship to the Core Specification {#relationship-to-the-core-specification}

The core specification [[PWS]] names this profile as the baseline: every
server conformant to its Minimal profile implements this document. A server
MAY implement further authorization profiles in addition; see the
[Authorization](https://w3c-ccg.github.io/portable-web-spaces-spec/#authorization)
section of [[PWS]].

This profile's persistent identifier is `https://w3id.org/pws/authz-profile`.
A server that implements it lists the identifier in its service description,
see [[[#service-description-entry]]].

The following are defined by [[PWS]] and only cited here:

* The DID methods a Space `controller` may use, in the
  [Space Controller DID Method Registry](https://w3c-ccg.github.io/portable-web-spaces-spec/#space-controller-did-method-registry),
  and the rules for setting and changing a controller.
* The `/policy` auxiliary resource, its discovery through the linkset, and
  the [policy evaluation contract](https://w3c-ccg.github.io/portable-web-spaces-spec/#access-control-policies).
* The error kinds this profile reports, in the
  [Error Type Registry](https://w3c-ccg.github.io/portable-web-spaces-spec/#error-type-registry);
  see [[[#errors]]].

### Reading This Document {#reading-this-document}

<div class="note">
This subsection is non-normative.

All examples use the Space id `81246131-69a4-45ab-9bff-9c946b59cf2e` on the
host `example.com`, the same Space every example in [[PWS]] uses. Paths are
written from the server root, which may be a subpath of an origin.

Request examples in [[PWS]] abbreviate the credential as a placeholder,
`Authorization: ...`. In every case it stands for a signed capability
invocation constructed as described in [[[#performing-authorized-api-calls]]].
The fully expanded form appears once, in the worked example of
[[[#request-body-integrity-digest-header]]].
</div>

## Terminology

<dl class="termlist definitions" data-sort="ascending">
  <dt><dfn data-lt="capabilities|capability|zcap|zCaps|authorization capability">zCap (Authorization Capability)</dfn></dt>
  <dd>A delegatable, attenuable object capability as defined by [[ZCAP]]: a
    document naming an <code>invocationTarget</code>, the
    <code>allowedAction</code>s it permits, and the <code>controller</code>
    who may invoke or delegate it. See the [[ZCAP-GUIDE]] for an introduction.</dd>

  <dt><dfn data-lt="controllers">controller</dfn></dt>
  <dd>An entity that can make changes to a given object. A Space's
    <code>controller</code> is a DID, see
    [[[#space-controller-and-the-root-of-trust]]].</dd>

  <dt><dfn data-lt="invocation|invocations|capability invocation">capability invocation</dfn></dt>
  <dd>A request that presents a capability and is signed by a key the
    capability authorizes. See [[[#performing-authorized-api-calls]]].</dd>

  <dt><dfn data-lt="root capabilities|root zcap">root capability</dfn></dt>
  <dd>The implied capability for a target whose <code>controller</code> is the
    Space's controller; it is the root of trust from which all other
    capabilities for that target are delegated. See [[[#root-capability]]].</dd>

  <dt><dfn data-lt="action|actions|allowedAction">action (allowedAction)</dfn></dt>
  <dd>The kind of operation a request performs on a target, named by a
    capability so it can be authorized. This profile uses the uppercase HTTP
    method names (<a>GET</a>, <a>POST</a>, <a>PUT</a>, <a>DELETE</a>) as its
    action vocabulary. See [[[#capability-invocation]]].</dd>

  <dt><dfn data-lt="targets|invocationTarget">target (invocationTarget)</dfn></dt>
  <dd>The resource a request acts on, as the full request URL (scheme, host,
    port, and path), and the scope a capability authorizes. A capability's
    <code>invocationTarget</code> MUST match the request target for the
    invocation to be valid, see [[[#capability-invocation]]]. A capability
    MAY instead name a <code>did:webvh</code> DID URL that a server
    dereferences to that URL, see [[[#did-url-targets]]].</dd>
</dl>

## Specification Dependencies at a Glance {#authorization-specification-dependencies-at-a-glance}

This profile uses the following specifications.

1. Identity (for controllers or clients/agents): [DID 1.0](https://www.w3.org/TR/did-1.0/)
2. Capability data model: [[ZCAP]]
3. Protocol for getting authorization: Out of scope (implementers are encouraged
   to use VC-API, OpenId4VP, OAuth2, or GNAP, as appropriate)
4. Proof of Possession / authorization invocation: HTTP Signatures.
   MUST - [[HTTP-SIGNATURES]] (Cavage draft 12),
   MAY - [[RFC9421]] HTTP Message Signatures (a future direction for this
   profile, see the note below)
5. Request body integrity: the `Digest` header, bound to the request
   signature -- see [[[#request-body-integrity-digest-header]]]
6. Access Control / Policy language data model: the `/policy` socket of
   [[PWS]], with the policy type this profile defines in [[[#policies]]]

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

## Space `controller` and the Root of Trust {#space-controller-and-the-root-of-trust}

Conceptually, the Space's controller serves as the root of trust and
authorization for any operations on the Space or its Collections or Resources.
That is, any operation requiring an authorization MUST provide a chain of proof
all the way to the Space controller, by one of the following:

1. Direct: Provide a [=root capability=] invoked directly by the controller, or
2. Delegated: Invoke a capability delegated to some other agent by the
   controller (see [[[#delegation]]]), or
3. Matching Policy: Match an access control policy at the target or inherited
   from a container above it (see [[[#policies]]]). A policy is related to the
   Space controller because it can only be written by the controller or by a
   party the controller delegated to.

The form a `controller` takes (a DID), the DID methods a server accepts, and
the rules for setting or changing a controller are defined by [[PWS]] in its
[Space `controller` and the Root of Trust](https://w3c-ccg.github.io/portable-web-spaces-spec/#space-controller-and-the-root-of-trust)
section and its
[Space Controller DID Method Registry](https://w3c-ccg.github.io/portable-web-spaces-spec/#space-controller-did-method-registry).
Every server implementing this profile supports `did:key` [[DID-KEY]]
controllers with Ed25519 keys in the Multikey encoding of [[CID]], as that
registry requires.

### Current-key-set rule {#current-key-set-rule}

Whatever the DID method, the key material that verifies an invocation or a
delegation is the signer's DID document as resolved at the time of
verification. A signature verifies if and only if its verification method is
present in that document, under the verification relationship the operation
requires: `capabilityInvocation` for an invocation, `capabilityDelegation` for
a delegation. A `did:key` document never changes. A document of a method with
a mutable, verifiable history (see the
[verified-log DID methods](https://w3c-ccg.github.io/portable-web-spaces-spec/#verified-log-did-methods)
of [[PWS]]) changes as that history is extended, so the set of accepted keys
is the current one. A delegation signed by a key that has since been removed
from the delegator's document stops verifying the moment that verification
method leaves the document, even though the capability itself was not revoked.
Every capability delegated onward from it stops verifying with it. Removing a
key from a controller's document is therefore an immediate, server-enforced
withdrawal of everything that key delegated, independent of any capability
revocation mechanism.

This rule applies to every DID that signs in a capability chain, not only to
the Space's `controller`. A capability rooted in a `did:key`-controlled Space
MAY be delegated to a DID of any method the server supports; the server
resolves that delegate's document the same way when it verifies the
delegate's invocation or onward delegation.

### Authorizing Space creation {#authorizing-space-creation}

When a Space is created via an HTTP `POST` or `PUT` operation (see the
[Create Space](https://w3c-ccg.github.io/portable-web-spaces-spec/#create-space-operation)
and [Update (or Create by Id) Space](https://w3c-ccg.github.io/portable-web-spaces-spec/#update-or-create-by-id-space-operation)
operations of [[PWS]]), the controller for that Space is set explicitly. That
is, a client specifies the `controller` as part of the payload of the `PUT` or
`POST` create Space request, and the server MUST verify that the invocation is
authorized by that `controller`, by one of the first two mechanisms above:
either directly -- the signing key (key ID) used in the headers is authorized
in the `capabilityInvocation` section of the `controller`'s DID document -- or
via a capability delegated by the `controller` to the signing DID. (The third
mechanism, matching policy, does not apply: no Space, and therefore no policy,
exists yet.) An invocation that is not authorized by the body's `controller`
is refused with `controller-mismatch` (400), see [[[#errors]]].

## Performing Authorized API Calls {#performing-authorized-api-calls}

Unless otherwise explicitly allowed via access control policy (see
[[[#policies]]]), all PWS API calls require authorization.

This can be done in one of two ways:

1. (for admin-like root access) Use the `controller` DID directly to sign
   HTTP API requests using the HTTP Signatures specification, invoking the
   target's [=root capability=].
2. (for advanced delegatable use cases) Use HTTP Signatures in combination
   with [[ZCAP]], and include a capability invocation header in the API
   request.

In both cases the request carries a `Capability-Invocation` header naming the
capability being invoked and the [=action=] performed, and an `Authorization: Signature ...` header carrying an HTTP
Signature [[HTTP-SIGNATURES]] over the request. The worked example in
[[[#request-body-integrity-digest-header]]] shows both headers in full.

## Request Body Integrity (Digest Header) {#request-body-integrity-digest-header}

When an authorized request carries a body (a Resource write, a Space create,
and so on), this profile binds the body to the request's HTTP Signature so
that the payload cannot be substituted without invalidating the signature:

1. The client MUST include a `Digest` header whose value is the hash of the
   request body, carried as a `mh` (multihash) parameter: a multibase
   base64url-encoded (`u` prefix) multihash of the body's SHA-256 digest
   (Multihash and Multibase as defined in [[CID]]). For example:

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
   with a body is rejected with an `invalid-authorization-header` (400)
   error, see [[[#errors]]].

Bodyless requests (`GET`, `HEAD`, `DELETE`) carry no `Digest` header.

The requirement applies per request. A chunk write (the chunk endpoints
defined by [[PWS-EC]]) therefore carries a `Digest` of that chunk, and a query request
(see the [Query Profile Registry](https://w3c-ccg.github.io/portable-web-spaces-spec/#query-profile-registry))
carries a `Digest` of the query body it signs.

Example authorized write request, showing the `Digest` header and the
covered headers list (the Space's [=controller=] invoking the
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

## Capability Invocation {#capability-invocation}

A [=capability invocation=] names an [=action=] that the invoked capability
must permit. This profile uses the uppercase HTTP method names as its action
vocabulary:

* <dfn id="get-action">`GET`</dfn> -- read a Space, Collection, or Resource. A
  `HEAD` request is authorized as a `GET`.
* <dfn id="post-action">`POST`</dfn> -- create a child item in a container (add a
  Resource to a Collection, a Collection to a Space, or a Space to the Spaces
  Repository).
* <dfn id="put-action">`PUT`</dfn> -- create-by-id or replace: a Resource at
  its own URL, or a Space or a Collection through its `meta` sub-resource.
* <dfn id="delete-action">`DELETE`</dfn> -- delete a Space, Collection, or
  Resource.

The [=target=] of a request is its full request URL: scheme, host, port, and
path. A target covers everything beneath it (see [[[#delegation]]]). Query
parameters select within a target and do not change it, so a capability for a
container authorizes every page of its listing.

A request is authorized by a capability when all the following hold:

1. the capability's `invocationTarget` matches the request's [=target=];
2. the capability's `allowedAction` includes the request's [=action=] (the
   HTTP method); and
3. the invocation is signed by a key the capability authorizes, carried as a
   valid HTTP Signature over the request (see
   [[[#performing-authorized-api-calls]]]), and that key is present in the
   signer's DID document as resolved at verification time (see
   [[[#current-key-set-rule]]]).

### Root Capability {#root-capability}

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

A Space also has a root capability whose target is a DID URL, on a server that
implements [[[#did-url-targets]]]. See [[[#did-url-root-capability]]].

### Delegation {#delegation}

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
covers the governing history log beneath it (see the
[Collection Governing History Log](https://w3c-ccg.github.io/portable-web-spaces-spec/#collection-governing-history-log)
of [[PWS]]). The same holds one level up for a Space.

A capability's `invocationTarget` MAY be a `did:webvh` DID URL in place of an
https URL, so that the grant keeps working when the Space moves to another
provider. [[[#did-url-delegation-chains]]] gives the rules for such a chain.

<div class="ednote">
**Revocation.** This profile does not yet define a revocation
operation, although the goals of [[PWS]] require that a grant can be withdrawn
before it expires. The current PWS implementation stack ships a Space-scoped
revocation endpoint (`POST /space/{space_id}/zcaps/revocations/{revocation_id}`;
every Space-rooted capability verification checks the presented delegation
chain against the recorded revocations). A future revision will specify the
operation and reserve its path segments.
</div>

### The Container Rule {#the-container-rule}

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

## DID URL Targets {#did-url-targets}

This section is OPTIONAL. A server that implements it lists the
`did-url-targets` token in its service description entry (see
[[[#service-description-entry]]]).

Every [=target=] in the sections above is an HTTPS URL on one provider. When a
Space moves to another provider, each capability issued for it names a URL
that no longer serves the Space, and the grants are lost. A `did:webvh` DID
URL [[DID-WEBVH]] can stand in for the https URL. A capability names a target
such as `did:webvh:{scid}:alice.example/photos/cat.jpg`. A `PathHandler`
service in the DID document maps the path to the current https URL. The
Space's owner moves the Space by changing that service's `serviceEndpoint`,
and the grants keep working.

The request is still an HTTPS request. The HTTP Signature covers the HTTPS
request URL, and the request's [=target=] is that URL, as
[[[#capability-invocation]]] defines. Only the capabilities in the chain name
DID URLs. A server dereferences each DID URL to an HTTPS URL and decides on
the result.

### Dereferencing a DID URL {#dereferencing-a-did-url}

A <dfn data-lt="DID URL targets">DID URL target</dfn> is an `invocationTarget`
that is a `did:webvh` DID URL with a path. It MAY carry a query. This version
of the profile defines the version parameters `versionId`, `versionNumber`
and `versionTime` (see [[[#did-url-selectors]]]). A target that carries a
fragment is refused.

<div class="ednote">
**Other query parameters.** This version of the profile gives no meaning to a
query parameter on a DID URL target other than the three version parameters.
A server MAY refuse a target that carries one, and the reference
implementation does. A future revision is expected to define further
parameters. The attenuation rule of [[[#did-url-selectors]]] already covers
them, since it compares the whole query.
</div>

A server dereferences a DID URL target as the DID URL path handling of
[[DID-WEBVH]] describes:

1. Resolve the DID document. The version parameters select the version. A
   target without them is read from the current document.
2. Of the document's services whose `type` includes `PathHandler`, select the
   one whose `path` is the longest complete match from the start of the DID
   URL path. A complete match is equal to the DID URL path, or is a prefix of
   it that ends at a `/` boundary.
3. Remove the matched `path` from the DID URL path and append the rest to the
   service's `serviceEndpoint`.

The result is the <dfn data-lt="dereferenced URLs">dereferenced URL</dfn>. It
carries no query and no fragment. It MUST be in canonical form. A server
refuses a dereferenced URL that a URL parser would rewrite, such as one with
a dot segment, a percent-encoded dot segment or a backslash. A target that
does not resolve, or whose path no `PathHandler` matches, is refused.

For example, this service maps the root path of a DID to a Space:

```json
{
  "id": "#root",
  "type": "PathHandler",
  "path": "/",
  "serviceEndpoint": "https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/"
}
```

Under it, the DID URL target `did:webvh:{scid}:alice.example/messages/`
dereferences to
`https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/`.

Every refusal in this section is reported as the masked `not-found` (see
[[[#errors]]]), except where [[[#did-url-denial-causes]]] says otherwise.

### DID URL Root Capability {#did-url-root-capability}

A Space has a root capability in DID URL form besides the HTTPS form of
[[[#root-capability]]]. Its id is `urn:zcap:root:` followed by the
percent-encoded DID URL:

```json
{
  "@context": "https://w3id.org/zcap/v1",
  "id": "urn:zcap:root:did%3Awebvh%3A{scid}%3Aalice.example%2F",
  "invocationTarget": "did:webvh:{scid}:alice.example/",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

The following rules apply to it.

* The DID URL MUST end in `/`. A Space has one root id per `PathHandler` path
  that maps to it.
* The DID URL MUST NOT carry a query or a fragment. A server
  refuses a root that carries one. Nobody signs a root capability, so a server
  always reads it from the current DID document, and a change to the mapping
  moves every chain beneath it.
* The DID URL MUST dereference to exactly the canonical (trailing-slash) URL
  of the Space the request addresses. A root that dereferences to anything
  else, such as another Space, another host or a Collection, is refused.
* The root's `controller` is the Space's [=controller=], as for the https
  form. The DID document does not supply it.

Unlike the HTTPS form, only a Space has a root in DID URL form. A grant on a
Collection or a Resource is a capability delegated from the Space's root, with
a longer DID URL path.

A server accepts a DID URL root on every operation on an existing Space. A
request that creates a Space takes an https root, since no Space exists yet
for the DID URL to dereference to (see [[[#authorizing-space-creation]]]).

### DID URL Delegation Chains {#did-url-delegation-chains}

A chain is in one form throughout. Every capability delegated beneath a DID
URL root MUST carry a [=DID URL target=], and every capability beneath an
HTTPS root MUST carry an HTTPS target. A server refuses a chain that mixes
the two forms, whichever way it mixes them. A DID URL target and an HTTPS
target never attenuate each other.

Each capability's target MUST dereference to a URL at or beneath the
[=dereferenced URL=] of its parent's target. This is the attenuation rule of
[[[#delegation]]], applied to the dereferenced URLs. The root dereferences to
the Space URL, so every capability in the chain stays inside the Space. A
target that dereferences to another Space, another host or a path above its
parent's URL is refused.

A capability MAY dereference through a different `PathHandler` service than
its parent did. A document can therefore alias a path, for example `/pics`
mapped to the `photos` Collection URL of the same Space. One document can also
serve several Spaces under different paths, and each Space then has its own
root DID URL.

A request is authorized by a chain in DID URL form when the conditions of
[[[#capability-invocation]]] hold, with the invoked capability's
[=dereferenced URL=] in place of its `invocationTarget`. The HTTPS request URL
may extend the path of that URL, and may carry a query, as with an HTTPS
target. The rules of [[[#the-container-rule]]] are also judged on the
dereferenced URL of the invoked capability.

### DID URL Selectors {#did-url-selectors}

The query of a DID URL target is its selector. A version parameter in it
chooses the version of the DID document that maps the path. That version's
`PathHandler` services then map the path as usual. A selector is not required.

A different document version may map the same path to a different Collection,
so a delegate that could change the selector could reach data it was not
granted. Attenuation between two DID URL targets therefore checks the path and
the selector separately:

1. Split each target at the first `?` or `#`. The part before it is the path
   part, and the rest is the selector part. The selector part is the whole
   query, whatever parameters it carries.
2. The path part MUST equal the parent's, or extend it at a `/` boundary.
3. The selector part MUST be byte-identical to the parent's.

One exception applies to rule 3. The capability directly beneath the root MAY
introduce a selector, since the root carries none. The Space's [=controller=]
signs that capability, so the choice of document version is theirs. Every
capability beneath it carries the selector unchanged.

A query on the HTTPS request selects no DID document. A server does not hold
it to the selectors of the capability.

### Trust Model {#did-url-trust-model}

The party that holds the DID's update keys controls the mapping and is
trusted with everything granted under the Space's DID URL roots. The check of
[[[#did-url-delegation-chains]]] bounds each capability by its parent's URL at
the time of the request, and not at the time of issue. A change to the DID
document can therefore do the following to a capability already issued:

* break it by removing or repointing the service that maps its path;
* widen it up to the whole Space its root dereferences to. Suppose a document
  maps `/` to the Space, and a capability on `did:webvh:{scid}:alice.example/trash/`
  allows `DELETE`. A new `PathHandler` that maps `/trash` to the Space URL
  makes that capability dereference to the Space URL, where it satisfies the
  Delete Space exception of [[[#the-container-rule]]];
* move it to another Space of the same controller, by repointing the service
  the root dereferences through.

Two bounds hold whatever the document says. A capability never reaches
outside the Space its root dereferences to. A server also gives the root the
Space's own `controller`, and the chain MUST still be signed from that
controller, so a DID document cannot create authority over a Space of another
controller.

The DID's update keys SHOULD be held by the Space's [=controller=]. A
deployment where they are separate parties trusts the holder of the update
keys with every grant made under a DID URL root.

### Which DIDs a Target May Name {#which-dids-a-target-may-name}

A [=DID URL target=] DID need not be the Space's `controller`, and need not be
hosted by the server. Portability depends on this, since a Space's owner keeps
the DID when the Space moves to a server that never hosted it.

A server that holds a verified copy of the DID's log resolves the target from
that copy. A server MAY fetch the log of a DID it does not host over the
network. A server that does so:

* SHOULD verify the request's signature and every delegation proof before it
  fetches;
* MAY cache the resolved document, and MAY limit how often it fetches a given
  DID;
* MAY refuse to fetch a DID or a host by operator policy.

The cache lifetime and the fetch limits are the implementation's choice. They
set how soon a server sees a move. A server that the Space moved away from MAY
keep accepting grants under its cached document until that document expires.

A refusal masks the status and the body of the response. It does not hide the
fetch. The holder of a valid capability on a DID the server does not host can
see a slower answer when the server fetches, and the host of the log sees the
request.

### Denial Causes {#did-url-denial-causes}

A server that names the cause of a refusal to the holder of a capability, as
`capability-revoked` or `capability-expired`, MUST NOT name it for a chain in
DID URL form unless the chain's root dereferences to the Space the request
addresses. Every other refusal of such a chain is the masked `not-found`.
Without this rule, a capability for one Space presented at another would
reveal its expiry when the two Spaces share a controller.

### Revoking a Capability in DID URL Form {#did-url-revocation}

A server that accepts revocation submissions (see the note in
[[[#delegation]]]) accepts a chain in DID URL form when its root has the form
of [[[#did-url-root-capability]]] and its delegation proofs verify back to
the Space's [=controller=].

* When a key of the Space's controller signs the submission, the server does
  not dereference the root of the submitted chain. This holds whether the
  submission's own invocation is rooted in the https root or the DID URL root.
  A revocation, therefore, does not depend on the current mapping, and it stays
  in force when the Space moves.
* When any other party in the chain submits, the root of the submitted chain
  MUST dereference to the Space the request addresses. Otherwise, a delegate
  could submit its capability at any Space and learn which Spaces share a
  controller.
* The root capability of the revocation URL itself stays in https form.

## Service Description Entry {#service-description-entry}

A server that implements this profile lists it in its service description
(see the [Service Description](https://w3c-ccg.github.io/portable-web-spaces-spec/#service-description)
of [[PWS]]) under the key `https://w3id.org/pws/authz-profile`. A version
entry under that key carries, in addition to the `version` and `url` members
every entry has:

* `signatureAlgorithms` (optional) - An array of the signature algorithms the
  server accepts on capability invocations (see
  [[[#performing-authorized-api-calls]]]), named by their JSON Web Algorithms
  [[RFC7518]] identifiers, so `EdDSA` [[RFC8037]] for Ed25519.
* `zcapCryptosuites` (optional) - An array of the Data Integrity cryptosuite
  names the server accepts on capability delegation proofs, such as
  `eddsa-jcs-2022`.
* `features` (optional) - An array of feature tokens naming the OPTIONAL
  sections of this profile the server implements. The vocabulary is open and
  additive. A client MUST ignore a token it does not recognize, and MUST treat
  an absent token, or an absent array, as "not supported".

This version of the profile defines one token:

* <dfn data-lt="did-url-targets">`did-url-targets`</dfn> - The server accepts
  a `did:webvh` DID URL as a capability's `invocationTarget` and as the target
  of a Space's root capability, as [[[#did-url-targets]]] defines.

A client MUST read the token before it issues a capability in DID URL form. A
server without the token answers an invocation of such a capability with the
masked `not-found`, which gives the holder no reason.

The entry advertises no policy types. Listing this profile means the server
evaluates every policy type this profile defines (see [[[#policies]]]).

```json
{
  "url": "https://example.com/pws/service",
  "specs": {
    "https://w3id.org/pws": [
      { "version": "0.5", "spaces": "https://example.com/pws/spaces/" }
    ],
    "https://w3id.org/pws/authz-profile": [
      {
        "version": "0.1",
        "url": "https://w3c-ccg.github.io/portable-web-spaces-spec/authz-profile/",
        "signatureAlgorithms": ["EdDSA"],
        "zcapCryptosuites": ["eddsa-jcs-2022"],
        "features": ["did-url-targets"]
      }
    ]
  }
}
```

A client that speaks this profile checks for the entry before its first
signed request, as part of the version selection [[PWS]] defines. A server
that lists no authorization profile the client implements is incompatible.

<div class="ednote">
This profile does not yet catalogue which values of `signatureAlgorithms` and
`zcapCryptosuites` a conformant server MUST accept. The example carries the
values the reference implementations exchange today: `EdDSA` over Ed25519
`did:key` keys, and `eddsa-jcs-2022` delegation proofs.
</div>

## Policies {#policies}

[[PWS]] stores an access control policy, a JSON document with a required
`type` property, at the `/policy` auxiliary resource of a Space, Collection,
or Resource, and defines how a policy is evaluated: a policy is consulted
only after any capability the request carries, it can only broaden access,
an absent or unrecognized `type` grants nothing, the most specific level wins,
and the request's action is reduced to a `read` or `write` access kind. See
its [Access Control Policies](https://w3c-ccg.github.io/portable-web-spaces-spec/#access-control-policies)
section for the contract.

Policy `type` values are defined by profiles and registered in
[[[#policy-type-registry]]]. This profile defines one type. A server
implementing this profile MUST evaluate it.

### `PublicCanRead` {#publiccanread}

```json
{ "type": "PublicCanRead" }
```

It grants the `read` access kind to any caller (including unauthenticated ones)
and grants no write access. This is the canonical "public read" pattern -- for
example, hosting an HTML file or an image that anyone may `GET`, while writes
still require a capability. Setting it on a Space makes the whole Space
public-readable (subject to any more specific Collection or Resource policy);
setting it on a single Resource exposes only that Resource.


## Policy Type Registry {#policy-type-registry}

This section is normative.

This registry lists the `type` values an access control policy may carry
(see [Access Control Policies](https://w3c-ccg.github.io/portable-web-spaces-spec/#access-control-policies)
of [[PWS]]) and the specification that defines each one. A server evaluates a
policy by looking its `type` up here. A `type` not in this registry is
unrecognized and grants nothing, per the fail-closed rule. Registering a type
adds a row naming the specification that defines it.

| `type`          | Grants                                                                                 | Defined by            |
|-----------------|----------------------------------------------------------------------------------------|-----------------------|
| `PublicCanRead` | the `read` access kind to any caller, including unauthenticated ones; no write access | [[[#publiccanread]]] |

## Errors {#errors}

This profile registers no error kinds of its own. It reports failures with
the kinds of the
[Error Type Registry](https://w3c-ccg.github.io/portable-web-spaces-spec/#error-type-registry)
of [[PWS]], as [[RFC9457]] problem responses:

* `missing-authorization` (401) - the `Authorization` or
  `Capability-Invocation` header is missing on an operation whose target's
  existence is not sensitive, such as a Create Space request at the Spaces
  Repository. Against an existing Space, Collection, or Resource the
  maximum-privacy rule below applies instead.
* `invalid-authorization-header` (400) - an `Authorization`,
  `Capability-Invocation`, or `Digest` header is present but malformed,
  unparseable, or fails verification (a signature that does not verify, an
  expired or unresolvable delegation chain, a `Digest` that does not match the
  body).
* `controller-mismatch` (400) - a Create Space invocation is not authorized
  by the `controller` supplied in the request body, see
  [[[#authorizing-space-creation]]].
* `not-found` (404) - a well-formed request that lacks sufficient privilege
  for an existing target.

The maximum-privacy rule of [[PWS]]'s
[Error Handling](https://w3c-ccg.github.io/portable-web-spaces-spec/#error-handling)
governs which of these a server may return. A credential that is present but
invalid describes the request, and MAY be reported precisely. A well-formed
request that simply lacks privilege for an existing target MUST be reported as
`not-found`, indistinguishable from the target being absent, and a request
that carries no credential at all MUST NOT learn from the response whether the
target exists.
