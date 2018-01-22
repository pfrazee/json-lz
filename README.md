# JSON LZ

Pronounced "JSON Lazy." A toolset for app developers to share JSON data between applications without creating conflicts in meaning. Includes tools to:

 - Identify the schemas of JSON documents,
 - Transform between schema-vocabularies, and
 - Detect ambiguities and abort.

**Status:** Work in progress.

## Background

JSON-LZ was created as part of [this discussion in the Beaker community](https://github.com/beakerbrowser/beaker/issues/820) between members of the p2p Web, microdata, and W3C Social WG communities. This toolset was largely inspired by [Robin Berjon](https://twitter.com/robinberjon)'s criticism of [JSON-LD](https://json-ld.org) titled [Don't Make Me Think (About Linked Data)](https://web.archive.org/web/20130814103818/http://berjon.com/blog/2013/06/linked-data.html).

See [DESIGN.md](DESIGN.md) for more information.

## Example objects

Here are some JSON objects which demonstrate how JSON-LZ's metadata works.

### One vocabulary

All attributes in this object are part of the `"alice-calendar-app"` vocabulary.

```json
{
  "@schema": "alice-calendar-app",
  "type": "event",
  "name": "JSON-LZ Working Group Meeting",
  "startDate": "2018-01-21T19:30:00.000Z",
  "endDate": "2018-01-21T20:30:00.000Z"
}
```

### Two vocabularies

Most attributes in this object are part of the `"alice-calendar-app"` vocabulary. The `"requested"` and `"deadlineDate"` attributes are part of the `"bob-rsvps"` vocabulary.

```json
{
  "@schema": [
    "alice-calendar-app",
    {"name": "bob-rsvps", "attrs": "rsvp.*", "required": true}
  ],
  "type": "event",
  "name": "JSON-LZ Working Group Meeting",
  "startDate": "2018-01-21T19:30:00.000Z",
  "endDate": "2018-01-21T20:30:00.000Z",
  "rsvp": {
    "requested": true,
    "deadlineDate": "2018-01-18T19:30:00.000Z"
  }
}
```

This JSON also uses the `"required": true` flag on the `"bob-rsvps"` vocabulary. See ["Vocabulary metadata"](#vocabulary-metadata) for more information.

## API

### jlz.detectSupport()

TODO document more

```js
var support = JSONLZ.detectSupport(obj, ['alice-calendar-app', 'bob-rsvps'])
if (support.full) {
  // 100% supported
}
if (support.partial) {
  // some vocabs are missing but should work fine
}
if (support.incompatible) {
  // unable to process object because required vocabs are missing
  // (in practice this is the only result we MUST worry about)
}
if (support.inconclusive) {
  // the object has no JSON-LZ metadata
}
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

**name**. The name is the only required attribute. It may be any string value, including URLs. You should try to use long and non-generic names to avoid accidental collisions. For instance, instead of just `'status-update'`, you should prepend your org name, eg `'genericorp-status-update'`.

You should use a URL for the name if you plan to publish and maintain documentation at the location. If not, then you should use a string that's unique enough that it will be easy to search against (and therefore discover the docs via a search engine).

**attrs**. The `attrs` attribute describes which attributes in the object use the vocabulary. It should be an array of attribute paths. (See [Attribute Paths](#attribute-paths).)

When multiple attribute paths match, the most specific (that is, longest) will be used. Therefore, for the object `{foo:{bar:{baz: true}}}`, the path `foo.bar` will match everything inside for `bar` (include `baz`). However, if the path `foo.bar.baz` is present, then that will take precedent over the `foo.bar` selector. If there are multiple matching selectors of the same length, the first to match will be used (vocabulary objects are processed in order).

If `attrs` is set to `undefined`, then the vocabulary will be used as the default vocabulary for any attribute that does not have an explicitly-set vocab. Only the first vocabulary-object with an undefined `attrs` will be the default.

**required**. If true, the vocabulary must be supported by the reading application for the JSON to have meaning. An application which does not support a required vocab must not use the input JSON.

### Attribute Paths

The "Attribute Path" is a very simple language for identifying attributes within the JSON object. It supports two special characters:

 - `.` Separates attribute references.
 - `*` Wildcard. Matches all attribute names

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

## How to use JSON-LZ

JSON-LZ uses metadata to "paint" the attributes of a JSON document with vocabulary definitions. Vocabs are then used to help identify the meanings of attributes, transform between different schemas, and fallback to safe defaults in the case ambiguous meaning. It is *not* an alternative to validation.

See ["When to use JSON-LZ"](DESIGN.md#when-to-use-json-lz) for more information about why JSON-LZ exists and when to use it in your app.

### Validating objects

Applications should validate their input JSON by checking the structure. Tools such as [JSON-Schema](http://json-schema.org/) are useful for accomplishing this. JSON-LZ does not help with validation - it only helps with checking for ambiguities in declared vocabularies and for transforming between vocabs.

### Transforming between vocabularies

Sometimes there are competing vocabularies that encode the same data. If you think you can still use the data, you can transform the data to fit the vocabulary/structure you use. This is sometimes called "munging" the data.

For example, suppose you have two "RSVP" vocabularies: `'bobs-rsvps'` and `'carlas-rsvps'`.

The `'bobs-rsvps'` schema looks like:

```json
{"rsvp": {"requested": true, "deadlineDate": "..."}}
```

While the `'carlas-rsvps'` schema looks like:

```json
{"rsvpIsRequested": true, "rsvpDeadline": "..."}
```

We can convert from `'carlas-rsvps'` to `'bobs-rsvps'` by iterating over each attribute in the `'carlas-rsvps'` vocabulary:

```js
obj.rsvp = obj.rsvp || {}
JSONLZ.iterate(obj, 'carlas-rsvps', (key, value, path) => {
  if (key === 'rsvpIsRequested') {
    obj.rsvp.requested = value
  }
  if (key === 'rsvpDeadline') {
    obj.rsvp.deadlineDate = value
  }
})
```

Here's an example object that would work with this technique:

```js
{
  "@schema": [
    "alice-calendar-app",
    {"name": "carlas-rsvps", "attrs": ["rsvpIsRequested", "rsvpDeadline"]}
  ],
  "type": "event",
  "name": "JSON-LZ Working Group Meeting",
  "startDate": "2018-01-21T19:30:00.000Z",
  "endDate": "2018-01-21T20:30:00.000Z",
  "rsvpIsRequested": true,
  "rsvpDeadline": "2018-01-18T19:30:00.000Z"
}
```

And here is what the output object would look like:

```js
{
  // ...
  "type": "event",
  "name": "JSON-LZ Working Group Meeting",
  "startDate": "2018-01-21T19:30:00.000Z",
  "endDate": "2018-01-21T20:30:00.000Z",
  "rsvp": {
    "requested": true,
    "deadlineDate": "2018-01-18T19:30:00.000Z"
  }
}
```

### Detecting schema support

JSON-LZ includes a method `detectSupport()` for detecting the compatibility between your app and a JSON's declared schemas. This method is important for avoiding bugs.

In most cases, partial support of a schema is not an issue, and apps can happily ignore the unsupported attributes, or perhaps warn the user about them if needed.

However, there are rare cases where ambiguous interpretations of data will result in a critical mistake. This is called a "fatal ambiguity," and it's problematic enough that it should be actively avoided. Proper use of `detectSupport()` and JSON-LZ metadata can accomplish this.

#### An example of "fatal ambiguity"

Suppose you have a social media application with "status update" JSONs, and one of the users' applications extends the JSON to include an `"audience"` field. The field's goal would be to control the visibility of a message; for instance, "only show this status-update to Bob." If that field is not interpretted correctly by a client, the message would be visible to the wrong audiences.

This is fatal ambiguity caused by partial support; the client understood the parts of the JSON that meant "status update" but it didn't understand the part that said "only show this to Bob."

#### Why does it matter?

Fatal ambiguities make it difficult for developers to freely extend their schemas. Unless they can signal the danger of partial support, they can only hope that all other clients will add full support in the near future. This will stifle developers and lead to bad user experiences.

#### How do I avoid fatal ambiguities?

As a schema developer, you use the `"required": true` attribute in your vocabulary object. This signals that the JSON data would be *misunderstood* without fully supporting that vocabulary.

```js
{
  "@schema": {
    {
      "name": "my-critical-vocabulary",
      "required": true // this schema MUST be supported!
    }
  }
  // ...
}
```

As an app developer, you use the `detectSupport()` method on inputs and you pass in the list of vocabularies that you fully support. If the returned object has the `.incompatible` flag set, you should ignore the JSON, or perhaps save it in debugging storage for the user to diagnose.

```js
var support = JSONLZ.detectSupport(obj, ['alice-calendar-app', 'bob-rsvps'])
if (support.full) {
  // 100% supported
}
if (support.partial) {
  // some vocabs are missing but should work fine
}
if (support.incompatible) {
  // unable to process object because required vocabs are missing
  // (in practice this is the only result we MUST worry about)
}
if (support.inconclusive) {
  // the object has no JSON-LZ metadata
}
```

#### How often should I use `"required": true` in my JSON?

**Very rarely!** The only time you should include it is if the misinterpretation (or non-interpretation) of a field would create a major issue. In most cases, partial support of an object's vocabs will not create issues, so you should leave the vocab as unrequired.

Required vocabs are a way to say "hide this object if you don't understand this vocab fully." Use it selectively.


