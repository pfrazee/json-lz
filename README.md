# JSON LZ

Pronounced "JSON Lazy." An alternative to JSON-LD that's designed to work how developers work.

Provides toolsets to:

 - Identify the schemas of JSON documents,
 - Transform between schema-vocabularies, and
 - Detect "fatal ambiguities" and fall back to smart defaults.

**Status:** Work in progress. See [DESIGN.md](DESIGN.md).

## Background

JSON-LZ was created as part of [this discussion in the Beaker community](https://github.com/beakerbrowser/beaker/issues/820) between members of the p2p Web, microdata, and W3C Social WG communities. This toolset was largely inspired by [Robin Berjon](https://twitter.com/robinberjon)'s criticism of [JSON-LD](https://json-ld.org) titled [Don't Make Me Think (About Linked Data)](https://web.archive.org/web/20130814103818/http://berjon.com/blog/2013/06/linked-data.html).

## Example usage

Here is a processor function which demonstrates how to use JSON-LZ.

```js
import * as JSONLZ from 'json-lz'

function processJsonObject (obj) {
  // ...
}
```

**Step 1. Detect support**

We want to make sure that we don't misunderstand the JSON's data. A misunderstanding is called a "fatal ambiguity" and it's an error condition. To avoid that, we do vocab support-detection.

```js
var support = JSONLZ.detectSupport(obj, ['alice-calendar-app', 'bob-rsvps'])
if (support.full) {
  // 100% supported
}
if (support.partial) {
  // some features missing but should work fine
}
if (support.incompatible) {
  // unable to process object because required features are missing
  // (in practice this is the only result we need to worry about)
}
if (support.inconclusive) {
  // the object has no JSON-LZ metadata
}
```

**Step 2. Validate the object**

We expect the object to fit a specific structure. If the structure differs, we'll fail validation or modify the object to make it fit.

```js
if (obj.type !== 'event') throw new Error('.type must be "event"')
if (!obj.name || typeof obj.name !== 'string') throw new Error('.name is required')
if (!obj.startDate || typeof obj.startDate !== 'string') throw new Error('.startDate is required')
obj.endDate = obj.endDate || obj.startDate
if (!(obj.rsvp && typeof obj.rsvp === 'object')) {
  obj.rsvp = {} // make sure a .rsvp object exists
}
obj.rsvp.required = typeof obj.rsvp.required === 'boolean' ? obj.rsvp.required : false
```

**Step 3. Transform between vocabularies**

Sometimes there are competing vocabularies that encode the same data. If we think we can still use the data, we can transform the data to fit the vocabulary/structure we use. We sometimes call this "munging" the data.

In this example, we have two "RSVP" vocabularies. The `'bobs-rsvps'` looks like:

```
{"rsvp": {"required": true, "deadlineDate": "..."}}
```

While the `'carlas-rsvps'` looks like:

```
{"rsvpIsRequired": true, "rsvpDeadline": "..."}
```

We're going to convert from `'carlas-rsvps'` to `'bobs-rsvps'`:

```js
JSONLZ.iterate(obj, 'carlas-rsvps', (key, value, path) => {
  switch (key) {
    case 'rsvpIsRequired':
      obj.rsvp.required = value
      break
    case 'rsvpDeadline':
      obj.rsvp.deadlineDate = value
      break
  }
})
```

All that's left to do to finish our `processJsonObject` method is to return the result:

```js
return obj
```

## Example objects

Here are some JSON objects which demonstrate how JSON-LZ's metadata works.

### One vocabulary

```json
{
  "@schema": "alice-calendar-app",
  "type": "event",
  "name": "JSON-LZ Working Group Meeting",
  "startDate": "2018-01-21T19:30:00.000Z",
  "endDate": "2018-01-21T20:30:00.000Z"
}
```

All attributes in this object are part of the `"alice-calendar-app"` vocabulary.

### Two vocabularies

```json
{
  "@schema": [
    "alice-calendar-app",
    {"name": "bob-rsvps", "attrs": "rsvp.*"}
  ],
  "type": "event",
  "name": "JSON-LZ Working Group Meeting",
  "startDate": "2018-01-21T19:30:00.000Z",
  "endDate": "2018-01-21T20:30:00.000Z",
  "rsvp": {
    "required": true,
    "deadlineDate": "2018-01-18T19:30:00.000Z"
  }
}
```

Most attributes in this object are part of the `"alice-calendar-app"` vocabulary. The `"required"` and `"deadlineDate"` attributes are part of the `"bob-rsvps"` vocabulary.

### Two vocabularies, with support metadata

```json
{
  "@schema": [
    "alice-calendar-app",
    {
      "name": "bob-rsvps",
      "attrs": "rsvp.*",
      "required": true
    }
  ],
  "type": "event",
  "name": "JSON-LZ Working Group Meeting",
  "startDate": "2018-01-21T19:30:00.000Z",
  "endDate": "2018-01-21T20:30:00.000Z",
  "rsvp": {
    "required": true,
    "deadlineDate": "2018-01-18T19:30:00.000Z"
  }
}
```

This object is semantically the same as the "Two vocabularies" example above, however it adds a requirement that the `"bob-rsvps"` vocabulary is supported by an app that reads the JSON. If that vocabulary is not understood by the reading app, the app should treat it as "fatally ambiguous" and avoid using the JSON input.

Developers should be careful about when they use the `required` keyword. See the section [Fatal ambiguity](#fatal-ambiguity).

## How to use JSON-LZ

JSON-LZ uses metadata to "paint" the attributes of a JSON document with vocabulary definitions. Vocabs are then used to identify the meanings of attributes, transform between different structures, and fallback to safe defaults in the case ambiguous meaning.

### Identifying the vocabulary of an attribute

TODO

### Transforming between vocabularies

TODO

### Fatal ambiguity

Ambiguous interpretations of data can sometimes result in critical mistakes. This is rare, but problematic enough that it should be actively avoided.

Avoiding fatal ambiguity is one of the primary purposes of JSON-LZ.

#### An example of "fatal ambiguity"

In a social media application, a "status update" JSON might be extended to include an `"audience"` field. The field's goal would be to control the visibility of a message; for instance, "only show this status-update to Bob." If that field is not interpretted correctly by a client, the message would be visible to the wrong audiences. This is fatal ambiguity caused by partial support; the client understood the parts of the JSON that meant "status update" but it didn't understand the part that said "only show this to Bob."

#### Why does it matter?

Fatal ambiguities make it difficult (if not impossible) for developers to freely extend their schemas. Because they have no way to signal the danger of partial support, they can only hope that all other clients will add full support in the near future. This will stifle developers and lead to bad user experiences.

#### How do I avoid fatal ambiguities?

As a schema developer, you use the `"required": true` attribute in your vocabulary object. This signals that the JSON data would be *misunderstood* without fully supporting that vocabulary, and should therefore be ignored or logged away for debugging if support isn't available.

As an app developer, you use the `checkSupport()` method on inputs and you pass in the list of vocabularies that you fully support. If the returned object has the `.incompatible` flag set to true, you should ignore the JSON, or perhaps save it in debugging storage for the user to diagnose.

#### How often should I use `"required": true` in my JSON?

Very rarely! The only time you should include it is if the misinterpretation (or non-interpretation) of a field would create a major issue. In most cases, partial support of an object's vocabs will not create issues, so you should leave the vocab as unrequired.

Required vocabs are a way to say "hide this object if you don't understand this vocab fully." Use it selectively.

## Vocabulary metadata

The JSON-LZ metadata is placed in the toplevel `"@schema"` attribute. It is an array of objects which describe the vocabularies of the attributes.

```js
{
  "@schema": [/* your schema vocab objects */],
  /* the rest of the data */
}
```

The values in the `"@schema"` array are called "vocabulary objects." They follow the following schema:

```
{
  name: String.
  attrs: optional Array<String>. Default undefined.
  required: optional Boolean. Default false.
}
```

If a string value is given in `@schema`, it will expand to an object with the default values. For instance, the string 'foo-vocab' will expand to the following object:

```js
{
  name: 'foo-vocab',
  attrs: undefined,
  required: false
}
```

**name**. The name is the only required attribute. It may be any string value, including URLs. You should try to use long and non-generic names to avoid accidental collisions. For instance, instead of just `'status-update'`, you should prepend your org name, ie `'genericorp-status-update'`.

You should use a URL for the name if you plan to publish and maintain documentation at the location. If not, then you should use a string that's unique enough that it will be easy to search against (and therefore discover the docs via a search engine).

**attrs**. The `attrs` attribute describes which attributes in the object use the vocabulary. It should be an array of attribute paths. (See [Attribute Paths](#attribute-paths).)

When multiple attribute paths match, the most specific (that is, longest) will be used. Therefore, for the object `{foo:{bar:{baz: true}}}`, the path `foo.bar` will match everything inside for `bar` (include `baz`). However, if the path `foo.bar.baz` is present, then that will take precedent over the `foo.bar` selector. If there are multiple matching selectors of the same length, the first to match will be used (vocabulary objects are processed in order).

If `attrs` is set to `undefined`, then the vocabulary will be used as the default vocabulary for any attribute that does not have an explicitly-set vocab. Only the first vocabulary-object with an undefined `attrs` will be the default.

**required**. If true, the vocabulary must be supported by the reading application for the JSON to have meaning. An application which does not support a required vocab must not make regular use of the input JSON.

### Attribute Paths

The "Attribute Path" is a very simple language for identifying attributes within the JSON object. It supports two special characters:

 - `.` Separates attribute references.
 - `*` Matches all attribute names

Examples:

```
name           => 'Bob'
location       => {state: 'Texas', city: 'Austin'}
location.state => 'Texas'
location.*     => ['Texas', 'Austin']
friends        => [{name: 'Alice'}, {name: 'Carla'}]
friends.*      => [{name: 'Alice'}, {name: 'Carla'}]
friends.0      => {name: 'Alice'}
friends.0.name => 'Alice'
friends.1.name => 'Carla'
friends.*.name => ['Alice', 'Carla']
```

Input object:

```
{
  name: 'Bob',
  location: {
    state: 'Texas',
    city: 'Austin'
  },
  friends: [
    {name: 'Alice'},
    {name: 'Carla'}
  ]
}
```

This path-language is intended to be so simple that an implementation can be written in less than twenty lines of code.

## API

### jlz.checkSupport()

TODO document more

```js
var doc = {
  '@schema':[
    {
      name: 'bll-fritter',
      required: true,
      fallback: 'discard'
    },
    {
      name: 'https://www.w3.org/ns/activitystreams',
      attrs: ['@type', 'content', 'inReplyTo', 'published']
    }
  ],
  // ...
}

jlz.checkSupport(doc, ['bll-fritter', 'https://www.w3.org/ns/activitystreams'])
/* => {
  passes: true,
  supported: [
    {
      name: 'bll-fritter',
      required: true,
      fallback: 'discard'
    },
    {
      name: 'https://www.w3.org/ns/activitystreams',
      attrs: ['@type', 'content', 'inReplyTo', 'published']
    }
  ],
  unsupported: []
} */

jlz.checkSupport(doc, ['bll-fritter', 'something-else'])
/* => {
  passes: true,
  supported: [
    {
      name: 'bll-fritter',
      required: true,
      fallback: 'discard'
    }
  ],
  unsupported: [
    {
      name: 'https://www.w3.org/ns/activitystreams',
      attrs: ['@type', 'content', 'inReplyTo', 'published']
    }
  ]
} */

jlz.checkSupport(doc, ['something-else'])
/* => {
  passes: false,
  fallback: 'discard',
  supported: [],
  unsupported: [
    {
      name: 'bll-fritter',
      required: true,
      fallback: 'discard'
    },
    {
      name: 'https://www.w3.org/ns/activitystreams',
      attrs: ['@type', 'content', 'inReplyTo', 'published']
    }
  ]
} */
```

### jlz.getSchemaFor()

TODO document more

```js
jlz.getSchemaFor(doc, 'text') // => 'bll-fritter'
jlz.getSchemaFor(doc, 'content') // => 'https://www.w3.org/ns/activitystreams'
```

### jlz.iterate()

TODO document more

```js
jlz.iterate(doc, 'https://www.w3.org/ns/activitystreams', (key, value, path) => {
  // example values for {"inReplyTo": {"summary": "Previous note"}}
  console.log(key) // => 'summary'
  console.log(path) // => 'inReplyTo.summary' 
  console.log(value) // => 'Previous note'
})
```





