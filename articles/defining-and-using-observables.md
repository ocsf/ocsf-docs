# Defining and Using Observables
Original Author: Rick Mouritzen, August 2024

Updates: Rajas Panat, October 2025

Observables provide a way to enrich OCSF events so that important data can be easily found and queried. They help avoid the need to navigate the complex and deeply nested structure of an event.

Observables are not intended to store new information. They should reference data that already exists elsewhere in the event. In this way, observables act as an optional query optimization that is common enough to have direct support in the OCSF schema.

For example, when searching across all events for a set of IP addresses known to be indicators of compromise, instead of scanning every field of type ip_t, or every field ending with _ip, one can simply query each event’s observables array for entries with type_id equal to 2 and the relevant IP addresses.

Observables are defined in event class and object definitions in the OCSF metaschema. This is done by associating a data type, data attribute, event class, or object with an observable `type_id`. 

## Defining Observables
The following ways to define observables are supported:

1. Observable by dictionary type. All attributes of this type become observables. The generated observables include the attribute values.
2. Observable by dictionary attribute. All instances of this attribute become observables. The generated observables include the attribute values.
3. Observable by object. All attributes of this object type become observables. The generated observables do _not_ include a value.
4. Observable by event class attribute. The attribute in the event class or its subtypes become an observable. Note that these are attributes defined at the base of the event, and not in nested structures. Also note that the attribute type can be any valid attribute type: a primitive, a primitive subtype, or an object.
5. Observable by object attribute. The attribute in all instances of the object or its subtypes become observables. Note that these are attributes defined at the base of the object, and not in nested structures. Also note that the attribute type can be any valid attribute type: a primitive, a primitive subtype, or an object.
6. Observable by class-specific attribute path. Attributes on the specified attributes path become observables. (The plural wording is used because a path element may refer to an array, resulting in multiple observables.)

In all cases, the definition of an observable is an integer number of OCSF type `integer_t`, a 32-bit signed integer. This number becomes the `observable` object's `type_id` value. The only values with a special meaning are the typical OCSF enum integer values of `0` (Unknown) and `99` (Other). There is no other special meaning or special ranges of values.

As with most things in OCSF, these definitions can be in the base of the core schema, one of the core schema extensions, or any other private extension (those extensions outside of the core schema).

As a historical note, definition types 1 and 3 have been in use since schema version 1.0, and the rest became available for use since 1.2.

### Definition Example: Observable by Dictionary Type
Defining an observable by dictionary type is, naturally, done in a metaschema `dictionary.json` file. The definition is done by adding an `observable` field to a type definition.

This, along with defining observable objects (which are also a kind of type) are the broadest ways to define observables. All attributes of this type (regardless of attribute name) become observables.

Example in a `dictionary.json` file:
```jsonc
{
  "name": "dictionary",
  // ... other dictionary fields (caption, description, attributes)
  "types": {
    // ... other "types" fields (caption, description)
    "attributes": {
      // ... other types
      "email_t": {
        
        // This is the observable definition for the email_t type, a subtype of string_t
        "observable": 5,

        "caption": "Email Address",
        "description": "Email address. For example: <code>john_doe@example.com</code>.",
        "regex": "^[a-zA-Z0-9!#$%&'*+-/=?^_`{|}~.]+@[a-zA-Z0-9-]+\\.[a-zA-Z0-9-.]+$",
        "type": "string_t",
        "type_name": "String"
      },
      // ...
    }
  }
}
```

### Definition Example: Observable by Dictionary Attribute
Defining an observable by dictionary attribute is also done in a metaschema `dictionary.json` file. The definition is done by adding an `observable` field to an attribute definition. 

Definitions done this way limit the creation of observable instances of this specific attribute, regardless of where it is used.

Example in a `dictionary.json` file:
```jsonc
{
  "name": "dictionary",
  // ... other dictionary fields (caption, description, types)
  "attributes": {
    // ... other attributes
    "cmd_line": {
      "caption": "Command Line",
      // ... other attribute fields

      // This is the observable definition for the cmd_line attribute
      "observable": 13
    },
    // ...
  },
  // ...
}
```

### Definition Example: Observable by Object
Defining an observable by object is done in a metaschema object definition file. The definition is done by adding an `observable` field directly in the object definition object. This also works for object definitions that extend another, including the special patch case. For both regular and patch extends cases, the observable definition adds or replaces any existing object-level observable definition.

This, along with defining observables by dictionary type, are the broadest ways to define observables. All instances attributes of this type (regardless of attribute name) become observables.

Example in object definition file `objects/container.json`:
```jsonc
{
  "name": "container",

  // This is the observable definition for this object
  "observable": 27,

  // ... other object definition fields
}
```

### Definition Example: Observable by Event Class Attribute / Observable by Object Attribute
Observables can be defined in event class or object attributes. These are very similar and so are described together here. Defining observables this way limits the scope where these observables will occur. Note that definitions of this type are for attributes directly defined in the event class or object, not for attributes in a nested structure.

Defining observables in event class or object attributes works just as with any other attribute definition at these levels: the fields defined override the same field defined in the dictionary or any event classes / objects the current item is derived from.

Definitions of this type work for event class and object `extends` definitions both for the normal case (subclass / subtype) as well as the "patch extends" case, however "hidden" event classes and objects are not supported due to the potential of colliding observable `type_id` values. See [Appendix 2: Hidden Types](#appendix-2-hidden-types) for more about this issue.

Example in object definition file `objects/cve.json`:
```jsonc
{
  "name": "cve",
  // ... other object definition fields
  "attributes": {
    // ... other attributes
    "uid": {
      "caption": "CVE ID",
      // ... other attribute fields

      // This defines uid as an observable when directly inside a cve object
      "observable": 18
    },
    // ...
  },
  // ...
}
```

Event classes work identically: attribute observables are defined inside attribute details.

### Definition Example: Observable by Class-Specific Attribute Path
This last observable definition type allows defining an observable for an attribute inside a nested structure for an event class, though it can be used for attributes directly defined in the class (in this case being a alternative to defining event class attribute observables). 

This type of definition is done by adding a top-level `observables` field to the event class definition whose value is a JSON object that maps from attribute paths to observable `type_id` values.

Class-specific attribute path definitions also work for event class `extends` definitions both for the normal case (subclass / subtype) as well as the "patch extends" case. In the extends cases, the class-specific observable definitions replace a prior definition or add a new definition. 

Hidden event classes are not supported due to the potential of colliding observable `type_id` values. See [Appendix 2: Hidden Types](#appendix-2-hidden-types) for more about this issue. 

The attribute paths use a simple dotted notation. See [Appendix 1: Attribute Paths](#appendix-1-attribute-paths) for details about how attribute paths are defined in OCSF. Notably these attribute paths do _not_ contain array references. In this context it means all array items in the attribute path are considered.

The following is an example showing the definition pattern for observables by class-specific attributes in an example "Bag" event class that holds an array of items, where each item has a `type_id` field that we want to become observables.

Example in event class definition file `events/bag.json`:
```jsonc
{
  "name": "bag",
  // ... other event class definition fields
  "attributes": {
    // ... other attributes
    
    // Here we add the items array: an array of item objects
    "items": {
      "requirement": "required"
    }
  },

  // Here we define the class-specific attribute observable as type_id 20
  "observables": {
    "items.type_id": 20
  }
}
```

This definition causes the `type_id` value of each item object to become an observable, but _only_ for events of the `bag` event class, and nowhere else.

### Definition Nuances
Defining observables has a few nuances, described in the next few sections.

#### Definition Nuance: Avoid Collisions
When defining a new observable, be mindful of collisions. One can use the OCSF Server running with the core extensions plus any additional extensions that may be used in a specific environment, and then view all of the existing observable `type_id` values on the `observable` object page (https://schema.ocsf.io/objects/observable). Also note that for additions to the core schema, the commit process detects collisions by running the [OCSF Server](https://github.com/ocsf/ocsf-server) and running the [OCSF Validator](https://github.com/ocsf/ocsf-validator).

Developers of private extensions should be extra wary to avoid collisions. Unlike most other unique identifier integer values in extensions, observable values are _not_ modified with tricky multiplication. Consider using using high integer numbers related to the extension's `uid`. For example for extension `uid` `999`, the observable numbers could start from `999000` (extension `uid` times `1000` plus observable number). This sort of precaution could become important as OCSF gains acceptance across the industry, and publishing and accepting OCSF events generated with private extensions begins to occur.

#### Definition Nuance: Precedence and the "Use the Most General" Rule
The definition types, as listed in the [Defining Observables](#defining-observables) section, establish a precedence with 1 being the most general to 6 being the most specific.

In cases where an OCSF event has an attribute that is affected by more than one observable definition, _the most general_ should be used.

#### Definition Nuance: Extends
Observable definitions can be overridden in extensions (via `extends`) in event class and object definitions, including the special "patch" type of extends. This is also mentioned above, though repeated here to emphasize that this is a general rule for all types of observable definitions.

#### Definition Nuance: Hidden Types Are Not Supported
Defining observables in definitions of hidden event classes or object is not supported as this would lead to colliding observables `type_id` values. See [Appendix 2: Hidden Types](#appendix-2-hidden-types) for more information. This is also mentioned above, though repeated here to emphasize that this is a general rule for all types of observable definitions.

#### Definition Nuance: Profiles
Profile definitions can override attribute observables (as with any other attribute field), though cannot modify object or class-specific attribute path observables (the top-level `observable` definitions). In other words, with regard to observables profiles can only affect observables by attributes.

> [!NOTE]
> Implementation-wise, this restriction may not be difficult to overcome, however it does become difficult to conceptualize and visualize the effect of a profile if it can affect a top level property of event class or object this way. Constraints, another top-level event class / object concept, are similarly not controllable by profiles.

#### Definition Nuance: New Object Observables Discouraged
The OCSF community currently discourages defining objects as observables. The object observable merely indicates the presence of the object in the event on a specific path. A second query would be needed to interrogate the object. It is not terribly useful.

#### Definition Nuance: Observable Values Are Strings
The `observable` object's `value` field is used for attributes that are primitive types (strings, numbers, booleans), subtypes of primitive types, and arrays of primitive types or primitive subtypes. The type of the `observable` object's `value` field is always of type `string_t`, and so string conversion may be needed. The `value` field is not populated for objects or arrays of objects.

For more, see [Populating The Value Field](#populating-the-value-field).

#### Definition Nuance: Event Class Observables Are Not Supported
Defining an event class as an observable is not supported. This would be essentially redundant with the `type_uid` field, which already uniquely identifies an OCSF event's event class. In other words, a query for an event class's observable ID might as well query for event classes `type_uid`. Further, the OCSF community is trying to move away from observables by object, which this would be similar to, since these do not populate the `observable` object's `value` field.

The only thing this sort of definition would enable would be to detect events that are in an event class inheritance subtree at a finer grain than categories; a fairly esoteric use-case.

## Using Observables
Observables can be generated automatically, though of course can also be created manually. Creating observables automatically requires walking an event's structure along with a compiled schema, looking for observable definitions at each level of the structure, as well at each leaf (each primitive value).

A concrete example of this code can be found in the [`Schema` class](https://github.com/ocsf/ocsf-java-tools/blob/main/ocsf-schema/src/main/java/io/ocsf/schema/Schema.java) in the [`ocsf/ocsf-java-tools` repo](https://github.com/ocsf/ocsf-java-tools). Look at the `enrich(Map<String, Object>, boolean, boolean)` method and follow it along. As an aside, this same class and approach can be used to add enum sibling values, and indeed can (and should) be done in the same pass as adding observables.

Whether creating observables manually -- a part of mapping process -- or automatically, care must be taken while populating the `name` and `value` fields.

### Populating Observable Path References
The `observable` object's `name` attribute is an attribute path reference. See [Appendix 1: Attribute Paths](#appendix-1-attribute-paths) for details.

### Populating the Value Field
The `observable` object's `value` field should be populated for all primitive types (strings, numbers, booleans), _and_ for arrays of primitive types (see next paragraph). The `value` field is specifically not meant to be used for observable objects, nor for arrays of objects.

For arrays of primitive types, one `observable` object should be created for each element of the array with the `value` field being set to the array element's value.

#### All Observable Values Are Strings
The `observable` object's `value` is defined as a type `string_t`. A primitive value that is not a string (either `string_t` or a subtype of `string_t`) must be converted to string.

Suggested conversions of non-string values:
* `integer_t` and `long_t`: base 10 string.
* `float_t`: base 10 string using common standard library conversions, including exponential notation and "NaN".
* `boolean_t`: the strings `"true"` and `"false"`.
* Null should not be converted to string, but rather the encoding's equivalent of `null`. In other words, a null remains a null. In JSON encoded events, use `"value": null` and not `"value": "null"`.

> [!NOTE]
> About `null`, it's weird. Don't overthink it. OCSF does not have a null type. In practice this means OCSF does not distinguish between a field that has a `null` value and a missing field. For observables, when creating them for primitive fields (like strings and numbers), if the field's value is `null`, then you may either set the `observable` object's `value` to `null` or not set the `value` field -- the meaning of each is equivalent. (This is not true in general. For those of that remember the XML era, distinguishing `null` from missing was one of the consistently annoying edge cases you'd have to always keep in mind.)

## Appendix 1: Attribute Paths
Attribute paths occur in two places: in the `observable` object's `name` attribute as a path reference to a field in the event, and in class-specific attribute observable definitions. In both cases the paths are the same. (Note: this is the only use of a JSON path-like capability in OCSF.)

The general pattern is dot-separated attribute names, for example `foo.bar`. Using the dot (".") as a separator works well because OCSF does not use dots in attribute names. There is no special notation for arrays, so these paths only tell us that a reference is for _one of_ the items along a path that includes one or more arrays.

These attribute path references are similar (at least in spirit) to [JSON Pointer](https://www.rfc-editor.org/rfc/rfc6901), [JSONPath](https://www.rfc-editor.org/rfc/rfc9535), and the syntax used by the [`jq` command-line tool](https://github.com/jqlang/jq), though simpler and notably without array notation.

### Path Notation Variations

While OCSF attribute paths use a simplified dot notation, producers may represent attribute paths in several legitimate ways when populating the `name` field of observable objects. This flexibility accommodates different producer capabilities and existing tooling preferences across various structural contexts.

#### Basic Path Notation
- **Simple dot notation**: `user.name` - Standard OCSF path notation
- **Nested object paths**: `user.profile.email` - Multi-level object traversal

#### Array Path Notation
- **Simple notation**: `resources.uid` - References the attribute across all array elements
- **Array bracket notation**: `resources[].uid` - Explicitly indicates array traversal  
- **Indexed notation**: `resources[0].uid` - References a specific array element by index
- **Root-relative JSONPath notation**: `$.resources[0].uid` - Full JSONPath notation starting from document root

All these notations are considered valid representations of the same logical path when referencing attributes within nested structures. Validation systems should accept any of these formats as legitimate observable names.

Let's say we have a event with a nested structure as follows:
```jsonc
{
  // ... other event fields
  "cloud": {
    "account": {
      "uid": "111122223333"
    }
  },
  "resources": [
    {
      "type": "AWS::KMS::Key",
      "uid": "arn:aws:kms:us-west-2:111122223333:key/example1"
    },
    {
      "type": "AWS::KMS::Key",
      "uid": "arn:aws:kms:us-west-2:111122223333:key/example2"
    }
  ]
}
```

In this example, `uid` is an observable by dictionary type with `type_id` of `10`. The following shows what the observables for this event look like, demonstrating the various legitimate notation styles:

```jsonc
{
  // ... other event fields
  "resources": [
    // ... as above
  ],
  "observables": [
    {
      "name": "cloud.account.uid",      // Basic, non-array path
      "type_id": 35,
      "value": "111122223333"
    },
    {
      "name": "resources.uid",         //Simple notation
      "type_id": 10,
      "value": "arn:aws:kms:us-west-2:111122223333:key/example1"
    },
    {
      "name": "resources[].uid",         // Array bracket notation
      "type_id": 10, 
      "value": "arn:aws:kms:us-west-2:111122223333:key/example2"
    },
    {
      "name": "resources[0].uid",        // Indexed notation
      "type_id": 10,
      "value": "arn:aws:kms:us-west-2:111122223333:key/example1"
    },
    {
      "name": "$.resources[1].uid",      // Full JSONPath notation with index
      "type_id": 10,
      "value": "arn:aws:kms:us-west-2:111122223333:key/example2"
    }
  ]
}
```

Notice that while different notation styles are used in the `name` fields, they all represent the same logical path to array elements. The positions in the `resources` array may be explicitly referenced (as in the indexed examples) or omitted.

> **Note for Implementers**: When validating observable names, systems should normalize these different array notations to recognize them as equivalent paths. The choice of notation may depend on the producer's capabilities, preferred representation style, or compatibility with existing JSONPath tooling. Validation logic should treat `resources.uid`, `resources[].uid`, `resources[0].uid`, `resources[*].uid`, and `$.resources[0].uid` as semantically equivalent when referencing array element attributes.

## Appendix 2: Hidden Types
It's a bit tedious to keep saying "event classes and objects". In Computer Science terms, these are both abstract data types, and specifically in object-oriented programming terms, their definitions are like classes. The OCSF terminology is a bit loose here. In this section, event class definitions and object definitions will simply be called types. Just note that OCSF also has primitive types (unstructured types) such as `string_t`, including subtypes of their primitive types like `email_t`.

> [!NOTE]
> Hidden types work like "inheritance for implementation". They exist so commonalities can be placed in a shared definitions, however these definitions are removed from the final compiled schema. Hidden event class definitions occur for definitions _without_ a `uid`, other than the special `base_event` definition which doesn't have a `uid` defined, but ends up with an effective `uid` of `0`.  Hidden object definitions occur for object definitions where the `name` value has a leading underscore, for example `_hidden`.

The net effect of hidden types is that each type that is derived from a hidden type gets all of the inherited information as if it was copied in by hand, essentially replicating the information in the hidden type. For observables, this would mean the `type_id` values would be replicated, and thus cause collisions among the type derived from the hidden type. (The only case where this wouldn't happen would be a hidden event class or object with only a single derivation. This isn't a useful case in practice, however.)

This hidden type observable collision is detected and blocked for each of these cases:
* A hidden event class definition with one or more attributes that define `observable`.
* A hidden object definition with one or more attributes that define `observable`.
* A hidden event class definition with class-specific attribute path observables defined via the top-level `observable` field.
* A hidden object definition defining itself an observables via the top-level `observable` field.

### Example Hidden Object
This is a concrete example using a hidden object that tries to define itself an observable by object type. The other cases work similarly.

Let's say we have a hidden object definition with `name` `_foo`, as well as `bar` and `baz` object definitions that extend the hidden `_foo` object. Now let's say we want all instances of the `_foo` object to be an observable with `type_id` of `42`, so we add `"observable": 42,` to the `_foo` object's definition. Let's show this more concretely:

`objects/_foo.json` (by convention, file names match the `name` attribute in the definition):
```jsonc
{
  "name": "_foo",
  "observable": 42,
  // ... other object definition fields
}
```

`objects/bar.json`
```jsonc
{
  "name": "bar",
  "extends": "_foo",
  // ... other definition fields
}
```

`objects/baz.json`
```jsonc
{
  "name": "baz",
  "extends": "_foo",
  // ... other definition fields
}
```

After compilation, `_foo` disappears, and both `bar` and `baz` are defined as observables by object with the `type_id` value `42`. This is a collision and if done manually would be flagged an an error between `bar` and `baz`.

What actually happens in this case is that when the hidden type definition like `_foo` is encountered, the OCSF Server and OCSF Validator ensure that it does not attempt to define observables of any kind.
