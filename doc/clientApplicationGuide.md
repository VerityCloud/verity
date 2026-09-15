# Verity client application guide

This is the implementation guide for applications built on Verity.
It provides best practices suitable for most applications.

## Mental model

- A **Cube** is a fixed-size network/storage block with typed fields.
- A **Veritum** is an application-level record. Under the hood it is represented
  by one or more Cubes, but applications usually don't need to care about this.
- **Veritable** is a general interface for any object that has the commons
  properties expected of a Veritum or Cube.
- A **Key** identifies each record in Verity, e.g. any Veritable or an Identity.
  Store and exchange these keys to refer to data in Verity.
  Each key is 32 Bytes and represented either as a 32 byte `Buffer` or as a
  64-character hex string.
- A **VerityField** is a standardized CCI field such as application, payload,
  media type, name, description, or relationship.
- A **Relationship** is a typed link e.g. from one Veritum to another or for
  example from an Idetity to a Veritum.
- An **Identity** is a mutable, signed profile and index of authored posts and
  public subscriptions.
- A **VerityNode** owns storage, retrieval, and networking.
  Application usually don't need to care about node-level stuff; the import
  part is that an application should have exactly one VerityNode object.
- A **Cockpit** combines a node with the current Identity (e.g. the logged in user)
  and is the normal and preferred application façade.

## General rules

1. **Use the highest-level API that solves the problem.**
   Prefer APIs like `Cockpit`, `Identity`, and `Veritum`.
   Use intermediate-level APIs like `Cube` or `VeritumRetriever` only if there's
   a specific reason.
   Don't use the low level Core APIs (`verity/core`).
2. **Model application data as Verita.**
   A Veritum is one logical record.
   Avoid using `Cube`, which is a low level block of exactly 1024 Bytes of data.
3. **Use a `Cockpit` for data associated with the active Identity.**
   It publishes, indexes posts, adds authorship hints, adds back-references,
   and decrypts for its Identity. Do not reproduce that orchestration in application code.
4. **Reuse Identity objects.**
   Every application context should share one
   `IdentityStore`; call `retrieveIdentity()` instead of `new Identity(...)`.
5. **Pass option objects through.**
   Option objects should generally be passed through the call chain.
   When passing options down the call chain, often data (e.g. defaults) is added
   to the options object at the different levels.
   As a callee, copy the options object before adding data unless
   your added data should specifically be passed up to the caller.
   Conversely, as a caller, copy your options object before passing them on unless
   you want the callee to be able to modify your options object.
6. **Accept interfaces at boundaries.**
   Depend on `VeritumRetrievalInterface`,
   `VerityNodeIf`, `Veritable`, or the narrowest retrieval interface needed, not
   `Cockpit`, `VerityNode`, or another concrete implementation.
7. **Treat network content as hostile.**
   Validate application and media fields,
   bound recursion and displayed content, await relationship resolution before
   trusting it, verify authorship, and sanitize rich text before inserting it in
   the DOM.
8. **Make lifecycle and asynchronous work observable.**
   Return promises, cancel
   subscriptions/generators where supported, and shut down nodes, stores, and
   controllers. Controllers register background promises with
   `registerAsyncOp()` so tests can await `lastAsyncOp` or `allAsyncOps`.

## Problem-to-primitive index

| Problem | Preferred primitive | Avoid |
|---|---|---|
| Publish an application record | `Cockpit.publishVeritum()` | don't handle Cubes manually |
| Build before publishing | `Cockpit.prepareVeritum()` | don't handle Cubes manually |
| Retrieve a logical record | `Cockpit.getVeritum()` | Avoid lower level `CubeRetriever` etc |
| Retrieve without a logged-in user | `VeritumRetriever.getVeritum()` | don't handle Cubes manually |
| Publish/retrieve encrypted data | `Cockpit.publishVeritum()` with `recipients` option | don't handle crypto manually |
| Represent a user/profile | `Identity` | avoid application-specific implementations |
| Resolve an Identity key | `IdentityStore.retrieveIdentity()` | avoid constructing duplicate `Identity` instances |
| Discover Identities on a channel | Use `notifyingIdentities()` the retrieve Identities referencing a common notification key | avoid parsing notifications yourself |
| Read an Identity's content | `Identity.getPosts()` | don't retrieve Cubes or Verita manually, avoid traversing relationships manually |
| Follow authors | `Identity.addPublicSubscription()` | avoid application-specific implementations |
| Attach semantic links | Use `VerityField.RelatesTo()` to create a `VerityField` representing the `Relationship`, then include this field in your Veritum, e.g. by including it in your `field` list when calling `cockpit.publishVeritum()` | raw bytes or invented standard codes |
| Resolve linked content | retrieval option `resolveRels` | avoid hand-written recursive fetches |
| Build Web UI behavior | `VerityController` plus a `VerityView` | avoid custom UI implementations unless your project specifically chose not to use the standard Verity UI classes |

## Fields: describing an application record

Use `VerityField` factories rather than numeric types or manually encoded
buffers:

```ts
import { VerityField } from "verity/cci/cube/verityField";
import { MediaTypes } from "verity/cci/cube/cube.definitions";

const fields = [
  VerityField.Application("MYAPP"),
  VerityField.MediaType(MediaTypes.TEXT),
  VerityField.ContentName("A concise title"),
  VerityField.Description("A short plain-text or Markdown summary"),
  VerityField.Payload("The complete content"),
];
```

Selection guide:

| Information | Field factory | Guidance |
|---|---|---|
| Protocol/application discriminator | `Application(id)` | Always validate before interpreting foreign data. Use a stable, versionable identifier. Identity defaults to `ID`; a separate application Identity may use e.g. `ID/MYAPP`. |
| Main bytes/text | `Payload(value)` | Accepts `Buffer` or UTF-8 string. Do not assume text unless media type says so. |
| Short title/file-style name | `ContentName(name)` | Display metadata, not a substitute for validation. |
| Summary | `Description(text)` | Plain text or controlled Markdown. Sanitize before rendering. |
| Common media discriminator | `MediaType(code)` | Use a defined `MediaTypes` value. Do not infer solely from a name. |
| User display name | `Username(name)` | Normally managed by `Identity`, not ordinary content. |
| Link to another record | `RelatesTo(type, key)` | Use a standard `RelationshipType` when semantics match. |
| Application extension | `new VerityField(0x30..0x3f, bytes)` | Only variable-length application fields belong in this range. Prefer common fields first. |

Read with `getFirstField(FieldType.X)` and handle absence or invalid lengths.
Use `getRelationships(type)` for repeated links. Never assume remote content is
CCI merely because retrieval succeeded; validate its application-level schema.

## Application schemata

### Application-specific fields
Prefer using commonly defined CCI fields for application data, as defined
at `doc/cci.md`.

If the shared vocabulary cannot express a concept, use application-specific
fields in the 0x30..0x3f range, which are variable-length.

Avoid deviating from the standard CCI field schema, i.e. avoid custom
`FieldDefinition`, `CubeFamilyDefinition` etc.

### Subrecords expressed by field groups
Field groups are useful when one logical record contains an ordered
sequence of small subrecords. Pick a clear start field for each group, slice the
field set by that marker, and reject the whole record if any group is malformed.
If later records depend on that order, validate the exact count and expected
type of every element before merging them into local state.

### Separate records connected by relationships
Use relationships for separately authored, separately encrypted, independently
updated, or independently discoverable records. A child record should include a
`REPLY_TO`, replacement, or application-specific relationship to its parent as
appropriate.

### Mutable data
For mutable application concepts, prefer immutable records connected by
`REPLACES` / `REPLACED_BY` (or a documented application relationship).

While Verity has limited support for mutable data records in the form of
`MUC` and `PMUC` Cube types, you should avoid using them for regular data as
they are not compatible with the standard Veritum/Cockpit data publishing model.

### Private datas
Private per-Identity settings can be modeled as ordinary encrypted posts to
self. This keeps across-device synchronization on the same Verity path as
public content. Decide whether such records should be indexed as posts, notified
on a private key, or kept discoverable only by an application-maintained reference.

## Publishing

### Normal record or post

```ts
import { Cockpit } from "verity/cci/cockpit";
import { VerityField } from "verity/cci/cube/verityField";
import { MediaTypes } from "verity/cci/cube/cube.definitions";

const veritum = await cockpit.publishVeritum({
  fields: [
    VerityField.Application("MYAPP"),
    VerityField.MediaType(MediaTypes.TEXT),
    VerityField.Payload(text),
  ],
});
return veritum.getKey();
```

With an Identity, defaults are intentionally opinionated:

- `addAsPost: true` adds the Veritum key to the Identity and stores its update.
- `addAuthorHint` follows `addAsPost` and adds the forward hint.
- `includePreviousPostRefs` defaults to the CCI minimum, improving discovery of
  older posts through distributed back-references.

Keep these defaults for public authored content;
override deliberately only if required.

Do not mutate a Veritum after publication: content changes its key; publish a
replacement and express sthe relationship instead.

### Prepare, modify, then publish

If for any reason you need to modify a Veritum before publishing, use `prepareVeritum()`:

```ts
const draft = cockpit.prepareVeritum({ fields });
draft.insertFieldBeforeBackPositionals(
  VerityField.RelatesTo(RelationshipType.REPLY_TO, parentKey),
);
/* insert custom modifications to draft here*/
const published = await cockpit.publishVeritum(draft, publishOptions);
```

Do not call `compile()` yourself before `publishVeritum()`. Publishing compiles,
stores every chunk, and performs Identity integration.

If an application model wraps publication, make the wrapper additive: normalize
and pass through caller options, insert model fields in the correct positional
area, and only override Cockpit defaults when the model deliberately owns the
corresponding behavior. When replacing an older posts, update the post list
consistently: add the new key, remove the superseded key, and store the Identity
once after all changes.

### Single-Cube records

Only drop to `Cube.Create()` when the one-Cube constraint is part of the design.
Choose the Cube type through options and still use CCI fields. Finalize it before
using its key and add it to `node.cubeStore`. This path does **not** provide
Cockpit's post indexing, authorship, continuation, or automatic encryption.

## Relationships

Use the semantic type, not whichever link happens to be convenient.
Standard relationship types are defined in `doc/cci.md`.
Application-specific relationship codes are 128 and above and should be used
when the standard types do not express the intended semantics.

## Retrieval and relationship resolution

Verity should be retrieved through a `Cockpit`.
A `Cockpit` can be bound to an Identity (by passing it during construction)
or used anonymously. Usually, whenever a user is logged in, the application
should use an Identity-bound cockpit.

Identity-bound cockpits perform common tasks in the background, such as
automatically decrypting Verita for which the Identity is a recipient.


### One record

```ts
const veritum = await cockpit.getVeritum(key);
if (!veritum) return undefined;
```

Returns a plain `Veritum`.
This default form is recommended when you only need the record itself.

### Linked records

Use `metadata: true` when the caller needs retrieval enhancements rather than
just the record. Metadata mode wraps the retrieved record in `main`, adds a
`done` promise, and is enabled automatically when using `resolveRels`.

```ts
const result = await cockpit.getVeritum(key, {
  ...retrievalOptions,
  metadata: true,
  resolveRels: "recursive",
  relTypes: [RelationshipType.REPLY_TO],
  maxRecursion: 5,
});

await result.done;
if (!result.allResolved) return undefined;
const post = result.main;
const parent = await result[RelationshipType.REPLY_TO]?.[0];
```

Note that `await result.done` is a simplification used in this example;
applications can and should await each record individually and process them
as they arrive whenever sensible.

Use `resolveRels: true` for direct relationships only. Use
`resolveRels: "recursive"` when you need a bounded graph, such as a reply chain
or dependency tree. Always set `relTypes` and `maxRecursion` for network data.

### Existing notifications versus future notifications

- `getNotifications(key)` is a finite async generator over known notifications.
- `subscribeNotifications(key)` yields only notifications received after the
  subscription starts and returns a cancellable generator.
- If both are needed, establish the subscription and combine/deduplicate the
  results.
- Default `VeritumRetriever` notification format is `RetrievalFormat.Veritum`;
  request `Cube` only when the first Cube is intentionally sufficient.
  For example, this would be appropriate to avoid downloading large records
  when the first 1024 bytes are sufficient to determine whether the record is relevant.

## Identities

Use `identity.key` (binary `Buffer` format) or `keyString` (ASCII-represented hex)
as the stable identifier, `identity.name` as untrusted display text,
and `identity.avatar.render()` for the generated safe-to-show avatar.
A custom profile picture is network content and needs the
same trust, type, size, and sanitization policy as any other media.

### One store per application context

```ts
const identity = await identityStore.retrieveIdentity(identityKey, {
  ...identityOptions,
});
if (!identity) return; // unavailable or invalid
```

The store ensures each Identity is represented by a single object.
In Verity Web UI controllers use `this.identityStore`; do not create
another. Pass it in all `IdentityOptions` where an API constructs identities.

### Create, load, and persist owned Identities

- `Identity.Create(retriever, username, password, options)` creates from
  credentials.
- `Identity.Load(retriever, {username, password, ...})`, `{masterKey}`, or
  `{recoveryPhrase}` loads an owned Identity.
- `Identity.New(retriever, options)` creates a random passwordless Identity;
  preserve its recovery material.
- `Identity.Retrieve(retriever, options)` lists locally persisted owned
  Identities whose root Cubes are already local.
- `IdentityPersistence.Construct()` is the async persistence factory. Share one
  persistence object per node and pass it through options.

After direct Identity mutations, await `identity.store()`. Cockpit already does
this when indexing a published post. Do not alter password-hardness or key
derivation contexts after deployment: doing so changes derived identities.
Avoid awaiting `fullyParsed` on latency-sensitive production paths; extension
retrieval may require multiple network timeouts.

### Posts and subscriptions

In Verity, any data published by an Identity (except standard fields like name
and avatar) is called a **post**.

#### Retrieving a single Identity's posts

```ts
for await (const post of identity.getPosts({
  subscriptionDepth: 0,
})) {
  if (isValidApplicationPost(post)) render(post);
}
```

#### Retrieving subscribed Identities' posts

Set `subscriptionDepth` above zero to include posts from public subscriptions.
`subscriptionDepth: 1` includes directly subscribed Identities;
`subscriptionDepth: 2` also includes their subscriptions, and so on. Keep the
depth as small as the UI needs, because each level expands the amount of
network content to retrieve and validate.

#### Using `subscribe: true`
By default, `getPosts()` yields existing posts and then exits. With
`subscribe: true`, the generator stays open and yields newly discovered posts as
they arrive. In subscription mode, `existingYielded` resolves once the initial
backlog has been yielded:

```ts
const posts = identity.getPosts({ subscribe: true });
await posts.existingYielded;

for await (const post of posts) {
  if (isValidApplicationPost(post)) render(post);
}
```

#### Using `metadata: true`
Use `metadata: true` when you need post metadata in addition to the post
itself. Most commonly, this supplies `post.author`, the verified Identity from
whose post list the content came. Use it when the UI attributes content to an
author, groups posts by author, or needs the author's key for follow-up actions.

```ts
for await (const post of identity.getPosts({ metadata: true })) {
  if (!isValidApplicationPost(post.main)) continue;

  renderPost({
    veritum: post.main,
    authorName: post.author.name,
    authorKey: post.author.keyString,
    avatar: post.author.avatar.render(),
  });
}
```

#### Using `resolveRels`
`resolveRels` is available the same way as when using `cockpit.getVeritum()`.

It implies metadata mode and yields relationship promises on the post metadata
object. Restrict the relationship types and recursion depth to the graph the UI
actually needs.

```ts
for await (const post of identity.getPosts({
  resolveRels: true,
  relTypes: [RelationshipType.REPLY_TO],
})) {
  await post.done;
  if (!post.allResolved || !isValidApplicationPost(post.main)) continue;

  const repliedTo = await post[RelationshipType.REPLY_TO]?.[0];
  renderReply({
    reply: post.main,
    parent: repliedTo,
    author: post.author,
  });
}
```

#### Managing subscriptions

In Verity, a **subscription** is a reference one Identity makes to another and
is usually used then this Identity typically retrieves the other's posts.
Subscriptions are **public** and discoverable by other Identities; do not use
them for confidential relationships.

To manage subscriptions, use `addPublicSubscription(identityOrKey)` and
`removePublicSubscription(identityOrKey)`, then `await identity.store()`.
Query with `hasPublicSubscription()`.

### Discover Identities advertised on a notification key

```ts
for await (const identity of notifyingIdentities(
  retriever,
  applicationIdentityNotificationKey,
  identityStore,
  { subscribe: true },
)) {
  consume(identity);
}
```

Use this when Identities should be discoverable through a shared rendezvous
point instead of through one user's subscription list. Common examples include
public channels, group directories, app-wide discovery, event or project
membership, and opt-in publication of service accounts. Treat discovery as
public: advertising on a shared notification key reveals that the Identity wants
to be found there.

### Authorship

Never trust display authorship from `AUTHORHINT` alone, it is unauthenticated data.
Prefer metadata returned by `Identity.getPosts()`.
For arbitrary content use `identityStore.findAuthor(veritable)`; it returns a
cancellable task whose `.promise` resolves to a verified Identity sor `undefined`.

## Encryption

Encrypt at publication by supplying recipient Identities (preferred) or their
encryption public keys:

```ts
await cockpit.publishVeritum({
  fields,
  recipients: [alice, bob],
  addAsPost: false,
});
```

Retrieve through a cockpit bound to the recipient:

```ts
const plaintext = await recipientCockpit.getVeritum(key);
```

Guidelines:

- Include the publishing Identity in `recipients` if the author must read the
  result later; self-access is not added automatically.
- `addAsPost` is independent of confidentiality. Decide whether publishing an
  encrypted record's key in the public Identity index leaks metadata.
- Do not set nonce/header/sender-key options or call chunk encryption directly.
- Do not use `excludeFromEncryption`, `symmetricKey`, or pre-shared parameters
  without a reviewed protocol. Incorrect nonce/key agreement is dangerous and
  plaintext exclusions leak metadata.
- Failure to decrypt is normal for a non-recipient; handle absent/failed content
  without exposing internals.

## Web UI controllers and views

Use the standard VerityController/VerityView architecture for browser applications unless
the project has deliberately chosen another UI architecture.

- Controllers own application state, validation, retrieval, and async work.
- Views render and forward user intent; they do not perform network operations.
- Obtain `cockpit`, `node`, `identity`, and `identityStore` from the inherited
  `VerityController` getters.
- Code controllers so no attached view is valid. Call optional view methods as
  `this.contentAreaView?.showResult?.(value)`.
- Return a promise from every async action. If an event listener cannot return
  it to the caller, register it:

```ts
onSubmit(text: string): void {
  const operation = this.publish(text).then(result => {
    this.contentAreaView?.showPublished?.(result);
  });
  this.registerAsyncOp(operation);
}
```

Tests can await `controller.lastAsyncOp` for the latest action or
`controller.allAsyncOps` for current background work.

For long-lived async generators, keep a reference, cancel it when supported,
and stop consuming it on navigation/controller shutdown. After every
`await`, re-check state that could have changed (login, navigation, or whether a
record was already displayed).

Before rendering network content:

1. Check the `APPLICATION` discriminator.
2. Require all mandatory fields and validate lengths/media types.
3. Enforce byte/record/graph limits.
4. Await required relationship resolution and reject unresolved dependencies.
5. Verify authorship when the UI attributes an author.
6. Sanitize markup with a restrictive allow-list; never interpolate into HTML.

For workflows that transform form state into records, run the same model-level
sanity checks before preview, publish, and import. Disable or otherwise guard
the submitted form while publication is in flight, then redraw from stored or
retrieved data instead of assuming the optimistic local object is authoritative.

## Testability checklist

- Inject a retrieval interface or Cockpit; do not construct network globals in
  business logic.
- Use `dummyCockpit()`/`dummyVerityNode()` and set zero difficulty only in tests.
- Return/track every promise and await publication before asserting storage.
- For subscription tests, await `existingYielded` before distinguishing backlog
  from live events.
- Test missing fields, wrong application/media type, unavailable continuation,
  failed relationship resolution, cycles/depth limits, forged `AUTHORHINT`,
  non-recipient encryption, duplicate notifications, and shutdown.
- Test application parsers with malformed repeated groups, wrong element counts,
  invalid enum values, mismatched relationship targets, and replacement chains.
- Test controllers without a view and with a minimal fake view.
- Shut down generators, Identity stores, identities, persistence, and nodes in
  teardown to prevent listeners/timers leaking across tests.
