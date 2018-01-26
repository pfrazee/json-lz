# JSON LaZy (LZ)

Simple self-describing JSON. Includes tools to:

 - Identify the schemas of JSON documents,
 - Transform between schemas, and
 - Detect ambiguities and abort.

See [DESIGN.md](DESIGN.md) for more information.

**Status:** Work in progress.

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Example objects](#example-objects)
  - [No schema](#no-schema)
  - [One schema](#one-schema)
  - [Two schemas](#two-schemas)
- [API](#api)
  - [jlz.detectSupport(obj, schemaIds)](#jlzdetectsupportobj-schemaids)
  - [jlz.getSchemaFor(obj, attrPath)](#jlzgetschemaforobj-attrpath)
  - [jlz.iterate(obj, schemaId, fn)](#jlziterateobj-schemaid-fn)
- [Schema metadata](#schema-metadata)
  - [Attribute Paths](#attribute-paths)
- [How to use JSON-LZ](#how-to-use-json-lz)
  - [Validating objects](#validating-objects)
  - [Transforming between schemas](#transforming-between-schemas)
  - [Detecting schema support](#detecting-schema-support)
    - [An example of "fatal ambiguity"](#an-example-of-fatal-ambiguity)
    - [How do I avoid fatal ambiguities?](#how-do-i-avoid-fatal-ambiguities)
    - [How often should I use `"required": true` in my JSON?](#how-often-should-i-use-required-true-in-my-json)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Example objects

Lazy is a way to describe what schemas your JSON is using. It's an optional convention. It's helpful when you start working with other developers but don't have any way to coordinate with them (for instance, because you're not on a team together). By adding schema information, you give other developers hints about what your objects are.

### No schema

Schemas are optional. This JSON has no schema declaration.

```json
{
  "type": "event",
  "name": "JSON-LZ Working Group Meeting",
  "startDate": "2018-01-21T19:30:00.000Z",
  "endDate": "2018-01-21T20:30:00.000Z"
}
```

### One schema

A schema can help other developers read your data. This object uses the `"alice-allisons-calendar-app"` schema.

```json
{
  "schema": "alice-allisons-calendar-app",
  "type": "event",
  "name": "JSON-LZ Working Group Meeting",
  "startDate": "2018-01-21T19:30:00.000Z",
  "endDate": "2018-01-21T20:30:00.000Z"
}
```

### Two schemas

Multiple schemas can be combined on one JSON. Most attributes in this object are part of the `"alice-allisons-calendar-app"` schema. The `"requested"` and `"deadlineDate"` attributes are part of the `"bob-bunsens-rsvps"` schema.

```json
{
  "schema": [
    "alice-allisons-calendar-app",
    {"name": "bob-bunsens-rsvps", "attrs": "rsvp.*", "required": true}
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

This JSON also uses the `"required": true` flag on the `"bob-bunsens-rsvps"` schema. See ["Schema metadata"](#schema-metadata) for more information.

## API

When you have the `"schema"` convention in your JSON, you can leverage the information to run some helpful operations.

### jlz.detectSupport(obj, schemaIds)

This method is a simple way to detect whether your app can handle some JSON.

There are 4 possible responses:

 - `.full === true` You support all of the JSON's declared schemas.
 - `.partial === true` You support some of the declared schemas.
 - `.incompatible === true` You don't support one of the required schemas.
 - `.inconclusive === true` The JSON doesn't declare its schemas.

```js
var support = JSONLZ.detectSupport(obj, ['alice-allisons-calendar-app', 'bob-bunsens-rsvps'])
if (support.full) {
  // 100% supported
}
if (support.partial) {
  // some schemas are missing but should work fine
}
if (support.incompatible) {
  // unable to process object because required schemas are missing
  // (in practice this is the only result we MUST worry about)
}
if (support.inconclusive) {
  // the object has no JSON-LZ metadata
}
```

### jlz.getSchemaFor(obj, attrPath)

This helper lets you find the schema for a given attribute. If the JSON describes its own schema, you'll be given an identifier.

```js
jlz.getSchemaFor(doc, 'text') // => 'bll-fritter'
jlz.getSchemaFor(doc, 'content') // => 'https://www.w3.org/ns/activitystreams'
```

### jlz.iterate(obj, schemaId, fn)

This helper finds all attributes in the JSON of a given schema. It will iterate each of the attribute, calling `fn` with identifying information.

```js
jlz.iterate(doc, 'https://www.w3.org/ns/activitystreams', (key, value, path) => {
  // example values for {"inReplyTo": {"summary": "Previous note"}}
  console.log(key) // => 'summary'
  console.log(path) // => 'inReplyTo.summary' 
  console.log(value) // => 'Previous note'
})
```

## Schema metadata

The JSON-LZ metadata is placed in the toplevel `"schema"` attribute. It is an array of objects which describe the schemas of the attributes.

```js
{
  "schema": [/* your schema objects */],
  /* the rest of the data */
}
```

The values in the `"schema"` array are called "schema objects." They follow the following schema:

```
{
  name: String.
  attrs: optional Array<String>. Default undefined.
  required: optional Boolean. Default false.
}
```

If a string value is given in `schema`, it will expand to an object with the default values. For instance, the string 'foo-schema' will expand to the following object:

```js
{
  name: 'foo-schema',
  attrs: undefined,
  required: false
}
```

**name**. The name is the only required attribute. It may be any string value, including URLs. You should try to use long and non-generic names to avoid accidental collisions. For instance, instead of just `'status-update'`, you should prepend your org name, eg `'genericorp-status-update'`.

You should use a URL for the name if you plan to publish and maintain documentation at the location. If not, then you should use a string that's unique enough that it will be easy to search against (and therefore discover the docs via a search engine).

**attrs**. The `attrs` attribute describes which attributes in the object use the schema. It should be an array of attribute paths. (See [Attribute Paths](#attribute-paths).)

When multiple attribute paths match, the most specific (that is, longest) will be used. Therefore, for the object `{foo:{bar:{baz: true}}}`, the path `foo.bar` will match everything inside for `bar` (include `baz`). However, if the path `foo.bar.baz` is present, then that will take precedent over the `foo.bar` selector. If there are multiple matching selectors of the same length, the first to match will be used (schema objects are processed in order).

If `attrs` is set to `undefined`, then the schema will be used as the default schema for any attribute that does not have an explicitly-set schema. Only the first schema-object with an undefined `attrs` will be the default.

**required**. If true, the schema must be supported by the reading application for the JSON to have meaning. An application which does not support a required schema must not use the input JSON.

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

JSON-LZ uses metadata to "paint" the attributes of a JSON document with schema declarations. Schemas are then used to help identify the meanings of attributes, transform between different schemas, and fallback to safe defaults in the case ambiguous meaning. It is *not* an alternative to validation.

See ["When to use JSON-LZ"](DESIGN.md#when-to-use-json-lz) for more information about why JSON-LZ exists and when to use it in your app.

### Validating objects

Applications should validate their input JSON by checking the structure. Tools such as [JSON-Schema](http://json-schema.org/) are useful for accomplishing this. JSON-LZ does not help with validation - it only helps with checking for ambiguities in declared schemas and for transforming between schemas.

### Transforming between schemas

Sometimes there are competing schemas that encode the same data. If you think you can still use the data, you can transform the data to fit the structure you use. This is sometimes called "munging" the data.

For example, suppose you have two "RSVP" schemas: `'bobs-rsvps'` and `'carlas-rsvps'`.

The `'bobs-rsvps'` schema looks like:

```json
{"rsvp": {"requested": true, "deadlineDate": "..."}}
```

While the `'carlas-rsvps'` schema looks like:

```json
{"rsvpIsRequested": true, "rsvpDeadline": "..."}
```

We can convert from `'carlas-rsvps'` to `'bobs-rsvps'` by iterating over each attribute in the `'carlas-rsvps'` schema:

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
  "schema": [
    "alice-allisons-calendar-app",
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

JSON-LZ includes a method `detectSupport()` for detecting the compatibility between your app and a JSON's declared schemas. This method is important for avoiding "fatal ambiguity."

#### An example of "fatal ambiguity"

Suppose you have a social media application with "status update" JSONs, and one of the users' applications extends the JSON to include an `"audience"` field. The field's goal would be to control the visibility of a message; for instance, "only show this status-update to Bob." If that field is not interpretted correctly by a client, the message would be visible to the wrong audiences.

This is fatal ambiguity caused by partial support; the client understood the parts of the JSON that meant "status update" but it didn't understand the part that said "only show this to Bob."

#### How do I avoid fatal ambiguities?

As a schema developer, you use the `"required": true` attribute in your schema object. This signals that the JSON data would be *misunderstood* without fully supporting that schema.

```js
{
  "schema": {
    {
      "name": "my-critical-schema",
      "required": true // this schema MUST be supported!
    }
  }
  // ...
}
```

As an app developer, you use the `detectSupport()` method on inputs and you pass in the list of schemas that you fully support. If the returned object has the `.incompatible` flag set, you should ignore the JSON, or perhaps save it in debugging storage for the user to diagnose.

```js
var support = JSONLZ.detectSupport(obj, ['alice-allisons-calendar-app', 'bob-bunsens-rsvps'])
if (support.full) {
  // 100% supported
}
if (support.partial) {
  // some schemas are missing but should work fine
}
if (support.incompatible) {
  // unable to process object because required schemas are missing
  // (in practice this is the only result we MUST worry about)
}
if (support.inconclusive) {
  // the object has no JSON-LZ metadata
}
```

#### How often should I use `"required": true` in my JSON?

**Very rarely!** The only time you should include it is if the misinterpretation (or non-interpretation) of a field would create a major issue. In most cases, partial support of an object's schemas will not create issues, so you should leave the schema as unrequired.

Required schemas are a way to say "hide this object if you don't understand this schema fully." Use it selectively.


