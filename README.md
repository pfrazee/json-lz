# JSON LZ

Pronounced "JSON Lazy." An alternative to JSON-LD that's designed to maximize compatibility between the schemas of projects with a minimal amount of forethought.

Provides:

 - A metadata schema for describing vocabularies in JSON documents.
 - Toolsets to...
   - Identify the vocabularies of JSON documents,
   - Transform between schema-vocabularies, and
   - Provide smart fallbacks when support isn't possible.

**Status:** Work in progress.

## Background

JSON-LZ was created as part of [this discussion in the Beaker community](https://github.com/beakerbrowser/beaker/issues/820) between members of the p2p Web, microdata, and W3C Social WG communities. This toolset was largely inspired by [Robin Berjon](https://twitter.com/robinberjon)'s criticism of [JSON-LD](https://json-ld.org) titled [Don't Make Me Think (About Linked Data)](https://web.archive.org/web/20130814103818/http://berjon.com/blog/2013/06/linked-data.html).

**Philosophy**:

 - "[Munging](https://en.wikipedia.org/wiki/Mung_(computer_term)) after" is better than "planning before." (See [Worse is better](https://en.wikipedia.org/wiki/Worse_is_better).)
 - Applications should be liberal in what they accept. (See [Robustness Principle](https://en.wikipedia.org/wiki/Robustness_principle).)

## When to use JSON-LZ

Apps are not expected to use JSON-LZ until they need to support multiple vocabularies. This is in keeping with our philosophy that "munging after" is better than "planning before." Put another way: you can't get every developer to do the "Right Thing," so you should make your apps safely handle the "Wrong Thing."

The default assumption is that JSON will be created without any vocabulary metadata. JSON-LZ apps are **always expected** to use [duck-typing](https://en.wikipedia.org/wiki/Duck_typing) and custom identifying attributes (such as a `type`) as their primary solution to validating their inputs. If a JSON input fits its schema-shape, then apps can assume vocabulary conformance.

As communities grow and integrate more applications around a shared dataset, they will encounter issues due to ambiguities in schemas. At this point, they can add JSON-LZ metadata and tooling. (Of course, you can add JSON-LZ as early as you want.) The level of strictness used can increase as the complexity of the issues increases. The goal is to be as relaxed as possible and therefore support the most possible inputs without introducing "fatal ambiguities." When ambiguities begin to degrade the UX, the strictness around vocabularies can be increased.

## How to use JSON-LZ

JSON-LZ uses metadata to "paint" the attributes of a JSON document with vocabulary definitions. Vocabs are arbitrary strings (not necessarily URLs) which declare the schema's origins. They can be used to identify the meanings of attributes, transform between different structures, and fallback to safe defaults in the case ambiguous meaning.

### "Painting" attributes

TODO

#### Via duck-typing

TODO

#### Via JSON-LZ metadata

TODO

#### Via JSON-LD metadata

TODO

### Identifying the vocabulary of an attribute

TODO

### Transforming between vocabularies

TODO

### Detecting ambiguities and falling back to safe defaults

TODO


## Requirements

### Good ergonomics within Javascript

JSON-LZ should be fun and easy to use. Developers should not have to futz with strange keynames or read long documents like this one prior to writing their applications. Further, it should involve minimal tooling to leverage; an app should not have to import a library to support JSON-LZ.

JSON-LD attempts to support "minimal tooling" by saying "all you need to do is put the @context field on your JSON." This fails in practice because schemas can take so many forms; depending on how the writing-app constructs the JSON-LD, some attributes may or may not have scopes in front of them (eg `foaf:relationship`). This creates a problem for apps that read the JSON, and it can only be solved with a "normalization" step (expand and contract into a predictable form).

JSON-LZ avoids this by not using scoped attributes. All vocabulary is applied by selectors in the metadata object, and follows an easy-to-understand structure. The goal is to make a tool that developers *actually like to use* and which doesn't raise weird questions like "what the hell does @id mean."

### Post-hoc compatibility

Compatability between schemas must be doable as an afterthought. Otherwise, then too much upfront coordination will be required and we won't get compatible data.

### Well-defined fallback behaviors

Ambiguous interpretations of data can sometimes result in critical mistakes. This should be avoided.

For instance, in a social media application, a "status update" JSON might be extended to include an `"audience"` field. The field's goal would be to control the visibility of a message; for instance, "only show this status-update to Bob." If that field is not interpretted correctly by a client, the message would be visible to the wrong audiences. This is fatal ambiguity caused by partial support; the client understood the parts of the JSON that meant "status update" but it didn't understand the part that said "only show this to Bob."

Fatal ambiguities make it difficult (if not impossible) for developers to freely extend their schemas. Because they have no way to signal the danger of partial support, they can only hope that all other clients will add full support in the near future. This will stifle development.

To address this, applications need a mechanism to identify ambiguities and fall back to well-defined behaviors. If a schema adds a feature which *must* be supported by an application for the message to have meaning, then the JSON-LZ tooling should to be able to identify the ambiguity and trigger one of the fallback behaviors. (Fallbacks may range from "render the simple form" to "render a descriptive error" to "render nothing at all.")

### Arbitrary vocabulary identifiers

Schema vocabularies may be identified by any string; URL identifiers are not required.

**Reasoning**: Publishing a schema is time-consuming and requires the developer to maintain the document at the given URL, which most developers won't bother with. Developers should instead endeavor to use "unique-enough" identifiers which are unlikely to collide, and which will provide good information on a search.

### Flexibility in vocabulary metadata

JSON-LZ's philosophy is to solve all things with munging, and it'd be hypocritical if we didn't munge metadata too.

JSON-LZ has a pluggable vocab-metadata processor which can be used to support alternative formats such as JSON-LD and [HAL](http://stateless.co/hal_specification.html). This can also be used by applications to detect vocabularies by duck-typing.

## API

### jlz.extract()

```js
var doc = {
  meta: {
    uses: [
      'bll-fritter', // default vocabulary
      {
        vocab: 'https://www.w3.org/ns/activitystreams',
        for: ['@type', 'content', 'inReplyTo', 'published']
      }
    ]
  },

  // fritter
  text: 'Hello world',
  threadRoot: 'dat://alice.com/posts/33.json',
  threadParent: 'dat://bob.com/posts/15.json',
  createdAt: 1516309496622,

  // activity streams
  '@type': 'Note',
  content: 'Hello world',
  inReplyTo: 'dat://bob.com/posts/15.json',
  published: '1/18/2018, 3:04:56 PM'
}

jlz.extract(doc, 'bll-fritter') /* => {
  text: 'Hello world',
  threadRoot: 'dat://.../',
  threadParent: 'dat://.../',
  createdAt: 1516309496622
} */

jlz.extract(doc, 'https://www.w3.org/ns/activitystreams') /* => {
  '@type': 'Note',
  content: 'Hello world',
  inReplyTo: 'dat://bob.com/posts/15.json',
  published: '1/18/2018, 3:04:56 PM'
} */

jlz.extract(doc, ['bll-fritter', 'https://www.w3.org/ns/activitystreams']) /* => {
  text: 'Hello world',
  threadRoot: 'dat://.../',
  threadParent: 'dat://.../',
  createdAt: 1516309496622,
  '@type': 'Note',
  content: 'Hello world',
  inReplyTo: 'dat://bob.com/posts/15.json',
  published: '1/18/2018, 3:04:56 PM'
} */

jlz.extract(doc, 'some-other-spec') // => {}
```

With `strict: false`, any attributes without vocabularies ("vocab-free" attributes) will be returned. With `strict: true`, the vocab-free attributes are excluded.

```js
var doc = {
  meta: {
    uses: [
      // no default vocabulary specified
      {
        vocab: 'https://www.w3.org/ns/activitystreams',
        for: ['@type', 'content', 'inReplyTo', 'published']
      }
    ]
  },

  // vacab-free attributes
  foo: 'bar',
  hello: 'world',

  // activity streams
  '@type': 'Note',
  content: 'Hello world',
  inReplyTo: 'dat://bob.com/posts/15.json',
  published: '1/18/2018, 3:04:56 PM'
}

jlz.extract(doc, 'https://www.w3.org/ns/activitystreams', {strict: true}) /* => {
  '@type': 'Note',
  content: 'Hello world',
  inReplyTo: 'dat://bob.com/posts/15.json',
  published: '1/18/2018, 3:04:56 PM'
} */

jlz.extract(doc, 'https://www.w3.org/ns/activitystreams', {strict: false}) /* => {
  foo: 'bar',
  hello: 'world',
  '@type': 'Note',
  content: 'Hello world',
  inReplyTo: 'dat://bob.com/posts/15.json',
  published: '1/18/2018, 3:04:56 PM'
} */
```

### jlz.getAttrVocab()

```js
jlz.getAttrVocab(doc, 'text') // => 'bll-fritter'
jlz.getAttrVocab(doc, 'content') // => 'https://www.w3.org/ns/activitystreams'
```

### jlz.iterate()

```js
jlz.iterate(doc, 'https://www.w3.org/ns/activitystreams', (key, value, path) => {
  // example values for {"inReplyTo": {"summary": "Previous note"}}
  console.log(key) // => 'summary'
  console.log(path) // => 'inReplyTo.summary' 
  console.log(value) // => 'Previous note'
})
```

### jlz.convert()

Example of converting from activity streams to bll-fritter:

```js
jlz.convert(doc, 'https://www.w3.org/ns/activitystreams', {
  content: 'text',
  inReplyTo: 'threadParent',
  published: 'createdAt'
})
```

Sometimes you need to put up type-guards, or deal with multiple value types. You can handle that by specifying a `value` function:

```js
jlz.convert(doc, 'https://www.w3.org/ns/activitystreams', {
  content: 'text',
  inReplyTo: {
    key: 'threadParent',
    value: v => {
      if (typeof v === 'string') return v
      if (v && typeof v.url === 'string') return v.url
      return undefined // no usable value
    }
  },
  published: 'createdAt'
})
```

## Notes

The metadata about a schema's vocabulary serves primarily to identify the vocabulary that a given attribute belongs to. With that metadata, we can 1) recognize the specific meaning of an attribute, 2) determine how to correctly transform data from one schema to another, 3) 
