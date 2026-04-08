# Patching the Core Schema With Extensions
Paul Agbabian,
August 2024

Extensions have been around since the earliest days of OCSF.  We knew that it was impossible to cover every type of event in a standard way, and that vendors will have special event classes, attributes and objects that only pertain to their products.  Customers may have their own enrichment pipelines with specific attributes that need to be type-checked, etc.

A somewhat subtle and hidden feature of the extension mechanism is how the core schema itself can be patched, meaning added to, without having to create new classes and new objects.  That is to say, an extension can have new attributes and new objects added to existing core classes without having to create an extension class.  An extension can add new attributes to a core object or class without having to create a new object or class.  An extension can even create new data types for new attributes that can be added to existing core classes and objects.

Before we learn how to patch the core schema, we will review standard schema extensions.  You will see later that patching the core schema is much simpler than having to create a standard extension, if all you want to do is add some attributes and constraints to existing classes or objects.

## Standard Approach for Extensions

Here is an example of a normal extension in a folder named `ocsf-extension` that adds an attribute, `a1` of type `string_t` via a new object that extends the core `metadata` object:

#### Extension example registration in `ocsf-extension/extension.json` file.  The name, version and uid are example names.

```json
{
    "caption": "Generic Extension",
    "name": "extension",
    "version": "1.1.0",
    "uid": 500
}
  
```

The `uid` and `version` are arbitrary here. For a real extension, you should request a unique extension ID via a Pull Request to the ocsf-schema repository.  This prevents collisions with other extensions.  Note that having a reserved public extension ID does not mean your actual extensions need be made public.

#### Dictionary entry in `ocsf-extension/dictionary.json` file.

```json
{
    "caption": "Generic Extension Attribute Dictionary",
    "description": "The Attribute Dictionary defines schema attributes and includes references to the events and objects in which they are used.",
    "name": "dictionary",
    "attributes": {
      "a1": {
        "caption": "An attribute",
        "description": "A generic extension attribute.",
        "is_array": false,
        "type": "string_t"
      }
    }
}
```

#### Object entry `extra_metadata.json` in `ocsf-extension/objects` subdirectory.
```json
{
    "caption": "Extra Metadata",
    "description": "The Generic Extension Extra Metadata object.",
    "name": "extra_metadata",
    "extends": "metadata",
    "attributes": {
      "a1": {
        "requirement": "recommended"
      }
    }
  }
  
```

This metaschema code will add a new object `extra_metadata` to the schema with a new attribute `a1` when the schema server loads and compiles the extension `ocsf-extension`.  If you are running your own schema server, you can do this at startup with the environment variable, or you can issue a reload at the Elixir command prompt: `Schema.reload(["extensions", "../ocsf-extension"])`, for this example, if your extension folder is parallel to the OCSF Schema repo folder.

If you do not include a caption or a description, the default is to use the caption or description of the object being extended.  The same holds for extension classes.

But, you may ask, how do I use this new `extra_metadata` object I created in my extension?  Although it is now visible to the schema server when loaded, it can only be used within a new extension class.  You would need to create a new class or extend an existing class, likely in the core schema, so that you can add it to that class.  Let's say you want to create a new Base class, and add it to that class.

First, you would need to add an attribute, for example `new_metadata`, to the extension dictionary of type `extra_metadata`:

```json
{
    "caption": "Attribute Dictionary",
    "description": "The Attribute Dictionary defines schema attributes and includes references to the events and objects in which they are used.",
    "name": "dictionary",
    "attributes": {
        ...
      "new_metadata": {
        "caption": "New Metadata",
        "description": "A new metadata object that can work with a new base class.",
        "type": "extra_metadata"
      }
    }
}
```

There are two ways of creating a new base class: either extend the core Base event class, or create a new base class for your extension.

To create a new base class, you would write something like the following metaschema code.  This example copies some of the core Base class attributes but not all, for example purposes.  Note the `new_metadata` attribute as well as the `"uid": 0"` statement.  As this is an extension, you must give the class an ID, which will be used to calculate a concrete class ID based on the extension master ID, in this case 500.  The core schema Base event class defaults to 0, as the core schema doesn't have an extension ID.  In most cases it is likely you will not be starting with a new base class in your extension, but rather extending the core Base event class, or some other core event class.

```json
{
    "caption": "New Base Event",
    "category": "other",
    "uid": 0,
    "description": "The new base event is a generic and concrete event. It also defines a set of attributes available in most event classes. As a generic event that does not belong to any event category, it could be used to log events that are not otherwise defined by the schema.",
    "name": "new_base_event",
    "attributes": {
      "$include": [
        "includes/classification.json",
        "includes/occurrence.json"
      ],
      "message": {
        "group": "primary",
        "requirement": "recommended"
      },
      "new_metadata": {
        "group": "context",
        "requirement": "required"
      },
      "raw_data": {
        "group": "context",
        "requirement": "optional"
      },
      "severity": {
        "group": "classification",
        "requirement": "optional"
      },
      "severity_id": {
        "group": "classification",
        "requirement": "required"
      },
      "status": {
        "group": "primary",
        "requirement": "recommended"
      },
      "status_code": {
        "group": "primary",
        "requirement": "recommended"
      },
      "status_detail": {
        "group": "primary",
        "requirement": "recommended"
      },
      "status_id": {
        "group": "primary",
        "requirement": "recommended"
      },
      "unmapped": {
        "group": "context",
        "requirement": "optional"
      }
    }
  }
  
```

If you really just wanted to add the `new_metadata` attribute to the existing Base event class, you could extend the Base event class in your extension, rather than create a new one:

```json
{
    "caption": "Extended Base Event",
    "category": "other",
    "uid": 1,
    "description": "The Extended Base event adds new attributes to the core Base event.",
    "extends": "base_event",
    "name": "extended_base",

    "attributes": {
        "new_metadata": {
            "group": "context",
            "requirement": "required"
          }    
    }
}

```

To keep things separated, this extended class uses a different `uid` value.  Note that only the `new_metadata` attribute was needed with this approach, as all of the standard Base event class attributes will still be present.

However, now you will have two metadata attributes, the core `metadata` attribute of type `metadata` as well as the extended metadata object `extra_metadata`.  Since `extra_metadata` extended `metadata` you now have two versions of most of the `metadata` attributes.  This is probably not what you wanted.  Maybe that's why you created a new class and copied most but not all of the attributes, so that you would only have one extended metadata object in the class.  You just wanted to add some extra attributes to the `metadata` object in the core schema's Base event class.  That's where Patching Extensions comes in.

## A Patching Approach for Extensions

Let's now say that we just want the attribute `a1` to be directly added to the core `metadata` object.

```
{
    "caption": "Metadata Extension",
    "description": "The Generic Extension metadata object.",
    "extends": "metadata",
    "attributes": {
      "a1": {
        "requirement": "recommended"
      }
    }
}
```
The key enabling feature is the omission of the `name` field.  This indicates to the schema compilation process that no new class or object should be generated.

The metaschema code above will add a new attribute `a1` directly into the existing `metadata` object when the schema server loads the extension `ocsf-extension` and compiles the schema.  In this case, the caption and description in the extension object will be ignored by the server but are useful for documentation purposes.

I think you will agree this is a much simpler way to add attributes to an object in the core schema.  When your extension is loaded and compiled, they will be added to the core schema.  *There is no need to create a new class to add your attributes - they will be added directly into core.*

The same approach can be used to add attributes directly to a core event class.  You would just extend the core class without declaring a new `name` and add the attributes from your extension dictionary.

```json
{
    "caption": "Patched Event",
    "description": "The Extended Base event adds new attributes to the core Base event.",
    "extends": "base_event",

    "attributes": {
        "a1": {
            "group": "context",
            "requirement": "recommended"
          }
    }
}

```

The core Base event class will now have a new attributes from the `ocsf-extension` extension dictionary, `a1`, directly added to the class, rather than to the `metadata` object (you would need to remove the `metadata` patch extension otherwise you would also have `a1` in the object).  Pretty simple.  Again, note that `name` is omitted and the `caption` and `description` are useful for documentation only - they will not be used in the schema.

More generally, a patching extension will do the following things:
- Profiles are merged
- Attributes are merged
- Class and object level observable definitions override / replace
- Constraints override / replace
- Caption and description are not patched but are good for documentation

## Constraints in Patching Extensions

As of OCSF Schema Server vs. 2.72.0 patching extensions can add overriding constraints to core classes and objects.  This was always possible with ordinary extension classes and objects, but was not supported by the server for patching constraints until 2.72.0.  An example of this for `metadata` is shown below with two attributes, `a1` from the prior examples and `a2` so that a constraint on the two can be added.

```
{
    "caption": "Attribute Dictionary",
    "description": "The Attribute Dictionary defines schema attributes and includes references to the events and objects in which they are used.",
    "name": "dictionary",
    "attributes": {
      "a1": {
        "caption": "An attribute",
        "description": "A generic extension attribute.",
        "is_array": false,
        "type": "string_t"
      },
      "a2": {
        "caption": "A second attribute",
        "description": "A second generic extension attribute",
        "is_array": false,
        "type": "integer_t"
      }
    }
}
```

```
{
    "description": "The Generic Extension metadata object.",
    "extends": "metadata",
    "attributes": {
      "a1": {
        "requirement": "recommended"
      },
      "a2": {
        "requirement": "recommended"
      }
    },

    "constraints": {
        "just_one": [
          "a1",
          "a2"
        ]
    }
 }
```

Note that if existing parent class or object constraints exist, they will be removed and replaced unless they are also included in the extension.  Also note that multiple constraints may be applied, as long as they don't conflict.  For example:

```json
{
    "caption": "Patched Event",
    "description": "The Extended Base event adds new attributes to the core Base event.",
    "extends": "base_event",

    "attributes": {
        "a1": {
            "group": "context",
            "requirement": "recommended"
          },
          "a2": {
            "group": "context",
            "requirement": "recommended"
          },
          "b1": {
            "group": "context",
            "requirement": "recommended"
          },
          "b2": {
            "group": "context",
            "requirement": "recommended"
          }    
    },
    "constraints": {
        "just_one": [
          "a1",
          "a2"
        ],
        "at_least_one": [
            "b1",
            "b2"
        ]
    }

}
```

As can be seen in the above example, the same capability is possible when patching event classes; you can add constraints in the patching extension class and they will replace any constraints of the class being extended.

## Conclusion
Patching extensions are a powerful yet simple way to add attributes to existing core schema classes and objects rather than having to introduce new extension classes with distinct names and IDs.  With the most recent OCSF Schema Server update, patching extensions also support constraints which replace any constraints of the extended class or object.  Existing queries will not need to change with the caveat that new patching constraints need to be carefully considered as they can change the validation of the core classes.
