# Design

## Philosophy

 - "[Munging](https://en.wikipedia.org/wiki/Mung_(computer_term)) after" is better than "planning before." (See [Worse is better](https://en.wikipedia.org/wiki/Worse_is_better).)
 - Applications should be liberal in what they accept. (See [Robustness Principle](https://en.wikipedia.org/wiki/Robustness_principle).)

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

## Why don't we just use JSON-LD?

We considered this quite a bit! We tried designs that were partially compliant with JSON-LD, and designs which were subsets. Ultimately, we abandoned that approach because (naturally) we were worried about partial support creating ambiguities. Our current thinking is, by using a separate metadata format, we make it *easier* to interop with JSON-LD because the two formats can coexist (`@context` next to `@schema`).

JSON-LD has more features than we need. The graph data model is not useful in our work and adds a lot of complex features. Also, the requirement to use URLs as vocabularity identifiers is very tedious for developers.

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
