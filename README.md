# JSON LZ

Pronounced "JSON Lazy." An alternative to JSON-LD that's designed to maximize compatibility between the schemas of projects with a minimal amount of forethought.

Provides:

 - A metadata schema for describing vocabularies in JSON documents.
 - Toolsets to...
   - Detect schema-vocabulary support,
   - Transform between schema-vocabularies, and
   - Provide smart fallbacks when support isn't possible.

## Background

JSON-LZ was created as part of [this discussion in the Beaker community](https://github.com/beakerbrowser/beaker/issues/820) between members of the p2p Web, microdata, and W3C Social WG communities. It was largely inspired by [Robin Berjon](https://twitter.com/robinberjon)'s criticism of [JSON-LD](https://json-ld.org) titled [Don't Make Me Think (About Linked Data)](https://web.archive.org/web/20130814103818/http://berjon.com/blog/2013/06/linked-data.html).

**Philosophy**:

 - [Munging](https://en.wikipedia.org/wiki/Mung_(computer_term)) after is preferable to coordinating before. (See [Worse is better](https://en.wikipedia.org/wiki/Worse_is_better).)
 - Applications should be liberal in what they accept. (See [Robustness Principle](https://en.wikipedia.org/wiki/Robustness_principle).)

## Requirements

### Good ergonomics within Javascript

Minimal tooling to support. (An app should not **have** to import a library to support JSON LZ.)

### Post-hoc compatibility

Compatability between schemas must be doable as an afterthought. Otherwise, then too much upfront coordination will be required and we won't get compatible data.

### Well-defined fallback behaviors

Schemas must be open to arbitrary extension while still providing well-defined fallback behaviors. If a schema adds a feature which *must* be supported by an application for the message to have meaning, then the schema needs to be able to trigger one of the fallback behaviors. (Fallbacks may range from "render the simple form" to "render a descriptive error" to "render nothing at all.")

### Arbitrary vocabulary identifiers

Schema vocabularies may be identified by any string; URL identifiers are not required.

**Reasoning**: Publishing a schema is time-consuming and requires the developer to maintain the document at the given URL, which most developers won't bother with. Developers should instead endeavor to use "unique-enough" identifiers which are unlikely to collide, and which will provide good information on a search.

### Versioned vocabularies

TODO

## API

### jlz.extract()

```js
var newDoc = jlz.extract(doc, 'fritter')
```

### jlz.iterate()

```js
jlz.iterate(doc, 'https://www.w3.org/ns/activitystreams', (key, value, path) => {
  // example values for {"inReplyTo": {"summary": "Previous note"}}
  console.log(key) // => 'summary'
  console.log(path) // => 'inReplyTo.summary' 
  console.log(value) // => 'Previous note'
})

// example of converting from activity streams to 
jlz.iterate(doc, 'https://www.w3.org/ns/activitystreams', {
  content:   value => { doc.text         = doc.text || value },
  inReplyTo: value => { doc.threadParent = doc.threadParent || value },
  published: value => { doc.createdAt    = doc.createdAt || value }
})
```
