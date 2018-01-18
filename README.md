# JSON LZ

Pronounced "JSON Lazy." An alternative to JSON-LD that's designed to maximize compatibility between the schemas of projects with a minimal amount of forethought.

Provides:

 - A metadata schema for describing vocabularies in JSON documents.
 - Toolsets to...
   - Detect schema-vocabulary support,
   - Transform between schema-vocabularies, and
   - Provide smart fallbacks when support isn't possible.

**Status:** Work in progress.

## Background

JSON-LZ was created as part of [this discussion in the Beaker community](https://github.com/beakerbrowser/beaker/issues/820) between members of the p2p Web, microdata, and W3C Social WG communities. This toolset was largely inspired by [Robin Berjon](https://twitter.com/robinberjon)'s criticism of [JSON-LD](https://json-ld.org) titled [Don't Make Me Think (About Linked Data)](https://web.archive.org/web/20130814103818/http://berjon.com/blog/2013/06/linked-data.html).

**Philosophy**:

 - "[Munging](https://en.wikipedia.org/wiki/Mung_(computer_term)) after" is preferable to "coordinating before." (See [Worse is better](https://en.wikipedia.org/wiki/Worse_is_better).)
 - Applications should be liberal in what they accept. (See [Robustness Principle](https://en.wikipedia.org/wiki/Robustness_principle).)

## How to use JSON-LZ

TODO

### Modes of conformance

#### Default naive mode

The default form of conformance in JSON-LZ is not to conform. This is the easiest of all approaches, and it is called the **"default naive"** mode. 

A default-naive app should manage conformance by [duck-typing](https://en.wikipedia.org/wiki/Duck_typing). If a JSON fits its schema-shape, then the app should assume vocabulary conformance.

#### Full-muckity mode

When an app decides to handle multiple vocabularies, it should manage conformance by JSON-LZ's detect/transform/fallback toolset. This is called the **"full-muckity"** mode (as in: "I've decided to get into the muckity muck of schema vocabularies").

Here, a JSON should be transformed to fit the target vocabulary, and if it fails any requirements it should use a well-defined fallback (which may include total rejection). Additionally, a fully muckity app should include the vocabulary metadata in its outputted JSON.

#### Half-muckity mode

Before adopting multiple vocabularies, an app may want to include the vocabulary metadata in its outputted JSON in order to maximise its possible conformance with other apps. This is known as the **half-muckity** mode and it's recommended but not required.

## Requirements

### Good ergonomics within Javascript

JSON-LZ should be fun and easy to use. Developers should not have to futz with strange keynames or read long documents like this one prior to writing their applications. Further, it should involve minimal tooling to leverage; an app should not have to import a library to support JSON-LZ.

JSON-LD attempts to support "minimal tooling" by saying "all you need to do is put the @context field on your JSON." This fails in practice during reading because schemas can take so many forms; depending on how the writing-app constructs the JSON-LD, some attributes may or may not have scopes in front of them (eg `foaf:relationship`). This can only be solved with a "normalization" step.

To address this, JSON-LZ includes non-conformance as a form of conformance (take that, philosophy majors!). This is called "default naive." The more advanced modes of schema-conformance use a separate metadata object, and therefore require no changes to the structure or key-naming conventions of the object.

### Post-hoc compatibility

Compatability between schemas must be doable as an afterthought. Otherwise, then too much upfront coordination will be required and we won't get compatible data.

### Well-defined fallback behaviors

Schemas must be open to arbitrary extension while still providing well-defined fallback behaviors. If a schema adds a feature which *must* be supported by an application for the message to have meaning, then the schema needs to be able to trigger one of the fallback behaviors. (Fallbacks may range from "render the simple form" to "render a descriptive error" to "render nothing at all.")

### Arbitrary vocabulary identifiers

Schema vocabularies may be identified by any string; URL identifiers are not required.

**Reasoning**: Publishing a schema is time-consuming and requires the developer to maintain the document at the given URL, which most developers won't bother with. Developers should instead endeavor to use "unique-enough" identifiers which are unlikely to collide, and which will provide good information on a search.

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
