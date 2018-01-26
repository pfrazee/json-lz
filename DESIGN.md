# Design

A key goal of the Dat and SSB networks is to make software development as open and free as possible. This includes enabling free extension to JSON schemas so that new apps can add features without lengthy community planning. This document explores a set of principles, patterns, and conventions for creating extensible, self-describing JSON.

Background reading:

 - [Don't Make Me Think (About Linked Data)](https://berjon.com/linked-data/)
 - [Beaker#820](https://github.com/beakerbrowser/beaker/issues/820)

## Principles

### Users cannot be forced

In an open network, there are no leaders or authorities. Every application is free to read and write data according to their own needs. Developers can only be expected to follow their incentives. There is no "Right Way," only useful patterns.

### Munging is better than planning

Because it is impossible to plan for all possible schemas, developers should expect to transform between multiple different semantics to create interoperability. This is known as ["munging."](https://en.wikipedia.org/wiki/Mung_(computer_term))

Compatability between schemas must be doable as an afterthought. Otherwise, then too much upfront coordination will be required and we won't get compatible data.

### Ergonomics matter

Developers should not have to futz with strange keynames or read long documents prior to writing their applications. Also, the less tooling (external libraries) the better.

URLs should not be used as the identifiers for schemas. Publishing a schema is time-consuming and requires the developer to maintain the document at the given URL, which most developers won't bother with. Developers should instead be allowed to use "unique-enough" identifiers which are unlikely to collide, and which will provide good information on a search.

## When to use JSON-LZ

Apps are not expected to use JSON-LZ until they need to support multiple vocabularies. This is in keeping with our philosophy that "munging after" is better than "planning before." Put another way: you can't get every developer to do the "Right Thing," so you should make your apps safely handle the "Wrong Thing."

The default assumption is that JSON will be created without any vocabulary metadata. JSON-LZ apps are **always expected** to use [duck-typing](https://en.wikipedia.org/wiki/Duck_typing) and custom identifying attributes (such as a `type`) as their primary solution to validating their inputs. If a JSON input fits its schema-shape, then apps can assume vocabulary conformance.

As communities grow and integrate more applications around a shared dataset, they will encounter issues due to ambiguities in schemas. At this point, they can add JSON-LZ metadata and tooling. (Of course, you can add JSON-LZ as early as you want.) The level of strictness used can increase as the complexity of the issues increases. The goal is to be as relaxed as possible and therefore support the most possible inputs without introducing "fatal ambiguities." When ambiguities begin to degrade the UX, the strictness around vocabularies can be increased.

## Requirements

### Good ergonomics within Javascript

JSON-LZ should be fun and easy to use. Developers should not have to futz with strange keynames or read long documents like this one prior to writing their applications. Further, it should involve minimal tooling to leverage; an app should not have to import a library to support JSON-LZ.

### Post-hoc compatibility

Compatability between schemas must be doable as an afterthought. Otherwise, then too much upfront coordination will be required and we won't get compatible data.

### Well-defined fallback behaviors

Ambiguous interpretations of data can sometimes result in critical mistakes. This should be avoided.

For instance, in a social media application, a "status update" JSON might be extended to include an `"audience"` field. The field's goal would be to control the visibility of a message; for instance, "only show this status-update to Bob." If that field is not interpretted correctly by a client, the message would be visible to the wrong audiences. This is fatal ambiguity caused by partial support; the client understood the parts of the JSON that meant "status update" but it didn't understand the part that said "only show this to Bob."

Fatal ambiguities make it difficult (if not impossible) for developers to freely extend their schemas. Because they have no way to signal the danger of partial support, they can only hope that all other clients will add full support in the near future. This will stifle development.

To address this, applications need a mechanism to identify ambiguities and fall back to well-defined behaviors. If a schema adds a feature which *must* be supported by an application for the message to have meaning, then the JSON-LZ tooling should to be able to identify the ambiguity and trigger one of the fallback behaviors. (Fallbacks may range from "render the simple form" to "render a descriptive error" to "render nothing at all.")

### Arbitrary vocabulary identifiers

Publishing a schema is time-consuming and requires the developer to maintain the document at the given URL, which most developers won't bother with. Developers should instead be allowed to use "unique-enough" identifiers which are unlikely to collide, and which will provide good information on a search.

## Concerns

### Fatal ambiguity

**Description**: Errors caused by partial support for a schema.

**Example**: *Bob* extends a schema used by *Alice* with an `audience` field to convey which users should be shown the message. *Alice* does not understand the new field and therefore shows the message to the wrong audience. *Bob* needs a way to convey the importance of the new `audience` field so that *Alice* ignores the message if she doesn't understand that field.

**Discussion**: This can be avoided by fields which describe required semantics. In SSB, the `type` field does this. I propose using a separate `schema` field to convey semantics, because `type` is separately useful for conveying what a JSON represents (kind/class). Conveying semantics in the `type` would incentivize *Alice* to change type from `"message"` to (eg) `"message-with-audience"` and we'd lose the kind/class equivalence. A `schema` field can also convey "unrequired" semantics in addition to "required" semantics, which makes it less strict.

### Keyname collisions

**Description**: Collisions between two schemas which make interoperation impossible.

**Example**: Independently of each other, *Alice* and *Bob* add a `date` field to an `Event` schema. *Alice*'s `date` field is a `String` which indicates when the event should occur, while *Bob*'s `date` field is a `Boolean` which indicates whether the event is a romantic encounter. Later, *Alice* tries to integrate *Bob*'s schema and discovers the collision makes this impossible.

**Discussion**: The main purpose of semantics is to create namespaces which disambiguate collisions such as this. The proposed `schema` field makes it possible for *Alice* to identify the exact namespace of a given `date` and process it accordingly. However, AFAIK there's no simple way to merge the two schemas together. Either *Alice* or *Bob* will need to change their schema. The "Field iteration by namespace" pattern makes this less of a problem.

### Validation assumptions

**Description**: Unstated assumptions about how to validate a schema result in unexpected breakages.

**Example**: AppA tolerates a scalar OR array for `{"foo":}` while AppB only accepts a scalar. AppB therefore fails validation more frequently and thus appears to "break randomly."

**Discussion**: This will definitely happen but I don't know of any good ways to solve it. The cure would probably be worse than the disease. I suggest we don't worry about this as spec-writers.

## Patterns

### Duck-typing

**Description**: Validate inputs by examining the structure of the JSON. If the structure does not match expectations, or is missing key information, the JSON should be ignored as nonconformant.

### Conventional `type` field

**Description**: Identify kind/class of a JSON input using a conventional toplevel  `type` field. Example values: `"note"`, `"status-update"`, `"event"`, `"profile"`, `"code-module"`.

```jsx=
{
  "type": "event",
  "name": "JSON-LZ Working Group Meeting",
  "startDate": "2018-01-21T19:30:00.000Z",
  "endDate": "2018-01-21T20:30:00.000Z"
}
```

**Discussion**: In SSB, the type is used to convey both kind/class and semantics. I suggest using a separate `schema` field to convey semantics, because `type` is separately useful for conveying kind/class (see the discussion in "Fatal Ambiguity"). It might work to suggest that `type` conveys semantics if `schema` is missing.

### Conventional `schema` field

**Description**: Identify the semantics of an object using a conventional toplevel `schema` field. This field is [described in detail here](README.md#vocabulary-metadata).

```jsx=
{
  "schema": "alice-calendar-app",
  "type": "event",
  "name": "JSON-LZ Working Group Meeting",
  "startDate": "2018-01-21T19:30:00.000Z",
  "endDate": "2018-01-21T20:30:00.000Z"
}
```

**Discussion**: The [full spec](README.md#vocabulary-metadata) for this field makes it possible to "paint" every attribute with an informal namespace ID, which addresses the "Keyname collisions" concern. This field can also mark the namespaces which are required, solving the "Fatal ambigiuties" concern.

### Field iteration by namespace

**Description**: The conventional `schema` field makes it possible to iterate the fields in a namespace. See [this example](README.md#transforming-between-vocabularies) for reference. This is useful for normalizing and transforming between schemas. 

```jsx=
jlz.iterate(doc, 'activitystreams', (key, value, path) => {
  // example values for {"inReplyTo": {"summary": "Previous note"}}
  console.log(key) // => 'summary'
  console.log(value) // => 'Previous note'
  console.log(path) // => 'inReplyTo.summary' 
})
```


## Why don't we just use JSON-LD?

JSON-LD has more features than we need. The graph data model is not useful in our work and adds a lot of complexity. Also, the requirement to use URLs as vocabularity identifiers is very tedious for developers.

More importantly than those two issues, JSON-LD adds flexibility in the wrong places.

Consider the two following JSON-LD objects:

```js
const obj1 = {
  "@context": {
    "schema": "http://schema.org/",
  },
  "@type": "schema:Person",
  "schema:name": "Manu Sporny",
  "schema:knows": "http://foaf.me/gkellogg#me"
}
const obj2 = {
  "@context": "http://schema.org/",
  "@type": "Person",
  "name": "Manu Sporny",
  "knows": "http://foaf.me/gkellogg#me"
}
```

These objects are schematically-equivalent but have differing structures. This is a problem. Javascript developers rely on structure to validate and process their inputs. Before either of these objects can be processed, they need to be normalized into a standard form.

The potential to transform structures automatically with expand/compact is neat, but in practice it means that reliable use of JSON-LD *requires* expansion and compaction because multiple object structures can map to the same schemas. That's a lot more tooling than we want to require for good support.
