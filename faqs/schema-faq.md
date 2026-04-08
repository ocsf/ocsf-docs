# Schema FAQ

This document answers common questions about how to use the OCSF Schema

## How do I create a typical OCSF event?

Depending on the type of event, a data producer or data mapper should first determine what event class best suits your event. Start with the OCSF category to narrow down the choices. For example, an endpoint security product would likely choose an event class from the System Activity category, for example, File System Activity for an AV product. Every event class has an `activity_id` enumeration which narrows down the intended activity of the event. Sometimes these are simple CRUD activities, but often they are more specific to the class, such as `Logon` for the `Authentication` class in the `Identity and Access Management` category.

Since endpoint security products typically send alert events when malware is detected, the producer or mapper would apply the Security Control profile, which adds important attributes to the File System Activity event class, e.g. a Malware object, a MITRE ATT&CK object, the disposition etc. These profiles have their own attributes that must be populated.

If your endpoint security product also has network security capabilities, you would choose an event class from the Network Activity category, for example the general Network Activity event class. Given that the endpoint product will have information about the host system, you would apply the Host profile, as well as the Security Control profile. The Host profile includes attributes about the device and the actor (e.g. process or user) on the host.

Every OCSF event must have all of its event class Required attributes populated, and should have its Recommended attributes populated, if possible. This includes any of the embedded objects, such as the Malware, Process and Device objects above.

All OCSF events have a set of required classification attributes from the Base Event class: the `class_uid` the `category_uid` the `activity_id` and the derived `type_uid`. Their associated `*_name` attributes are optional.

In addition to the classification attributes, a number of other Base Event class attributes are required and must be populated: the `time` `metadata` and `severity` attributes. The `metadata` attribute is an object that itself requires the `product` and associated `version` of the reporting event, as well as the version of the OCSF schema adhered to with the event.

Note that the product should be the originating event producer (i.e. not the mapping system, nor any intermediary event processing systems) in order to best represent the origin of the event. The `time` should be the time that the event actually occurred assuming that information is known, or the earliest possible time available to the event producer or mapper.

Although the `observables` array attribute is optional, populating it can make things easier for event consumers and analysts. Each Observable object surfaces an important attribute of the event in a common location in a simple tuple: name, value, type. For example, if the event class has a `device` `user` and `process` populated, an array of three Observable objects will refer to them in a common location to all OCSF events.

---

## How would I populate the `observables` array?

There are three important attributes of the `Observable` object, and the Base Event class allows for an array of these objects with the `observables` attribute: `name`, `type_id` and `value`. The first two are required attributes, while `value` is optional. Why it is optional will become clear soon. There can be multiple observables within an event, even of the same type. This is why `observables` is an array.

The required `name` attribute of the `Observable` object should be the fully qualified attribute name within the event. E.g. `fingerprint.value` or `actor.process.file` or `actor.process.file.name`. In other words, `observable.name` is the locator of that observable within the instance of the event. Note that the observable attribute can be a scalar, like `device.ip`, or it can be an object, like `actor.process.file`.

When the `type_id` of the observable indicates that the observable's `name` attribute is of object type, e.g. Fingerprint, the observable's `value` attribute is not populated. When the `type_id` indicates the observable's `name` is a scalar, e.g. File Hash or File Name, then the observable's `value` should be populated with the value of that attribute, that is, a copy of the value from the event.

---

## When should I use a Finding event class?

A Finding in OCSF represents the result of some type of enrichment, correlation, aggregation, analysis or other processing of one or more events or alerts, producing a derived insight. Most security events and alerts are activity events with a disposition (e.g. Blocked), for example when using the Security Control profile. Findings in OCSF are not always alerts themselves, although alerts may be triggered by findings or findings might be added to an incident further downstream.

For example, an email security product may determine that a user has been phished or an email attachment is malicious. It would send an email activity event (from its standpoint an alert) containing the user and sender, supplemented by the Security Control profile with a disposition of Blocked, and information about the Malware, to its management console which in turn sends it to a SIEM.

The SIEM might receive other related events or alerts, for example for other users in the same circumstance or for general email activity from the same sender. The SIEM might enrich the events with information from a Threat Intelligence Platform or threat feed pertaining to the email sender. The result of the aggregation, and enrichment would constitute an OCSF Finding. The SIEM might create an incident that includes or refers to the finding, in the event that there are remediation steps required.

Note that in a more complex processing architecture, there may be layered findings. That is, the original event may go to product A which eventually triggers a finding. Product B meanwhile may take in a lot of other events and findings (including those from product A) and make its own findings. In the example above, the originating email alert might have been a finding from the producer's standpoint if the event was enriched by its management system before being collected by the SIEM, which then produced a more complete finding.

---

## When should I use metadata.correlation_uid?

When an event producer or mapper emits multiple events that have some grouping characteristic, or similarity of any form, it should populate the `metadata.correlation_uid` attribute with a constant identifier. This allows consumers and analysts of the set of events to more easily aggregate and correlate the events.

A simple example would be a vulnerability scanner that emits events at the start of a scan of a system, at the end of the scan, and separate events for each vulnerability discovered. If these are separate events, they would all have their `metadata.correlation_uid` set to the same value.

It is possible for an intermediary system to determine the grouping characteristic as well, populating the attribute after collection of the events, although when OCSF events are immutable a copy of the original events would be made with added correlation information. See the next question.

---

## Can Finding events be correlated with each other too?

Yes, they are also events with a base class metadata object that can follow the same pattern.
E.g. a SIEM that creates findings may have enough knowledge and state to tie multiple findings together with a metadata.correlation_uid.

---

## How do I use the Actor object?

The Actor object is intended for use in event classes when knowledge of one entity that is initiating or causing some action on another entity.  
For example, a process deleting a file is the actor in a Filesystem Activity event.

From a structural standpoint, the `actor` attribute avoids name collisions with the other end of the activity in cases where a process acts on another process, as those attribute names would be in contention at the same level within the class.

Currently the Actor object has a `process` and `user` attribute, where one or the other is in the role of the actor in the activity. It also has Optional attributes for Session, `authorizations`, `idp`, and `invoked_by`.

The `idp` is populated in IAM category event classes, when the actor's identity provider is known and logged with Authentication and related events.

The `authorizations` attribute is an array of information pertaining to what permissions and privileges the actor has at the time of the event, if known.

The `invoked_by` attribute is populated with the name of the service or application through which the actor's activity was initiated.

---

## When should I use the session attribute?

The `session` attribute is usually paired with the `user` attribute. A Session object has information pertaining to a particular user session through which the activity was initiated. User is an entity object that isn't always associated with a session, and isn't always an actor, hence Session isn't part of the User object, but is included with the Actor object for actor semantics.

Related to this, the `process` attribute of type Process has a User object which represents the user's account that the process runs under or is impersonating. Hence, the Process object also has a `session` attribute paired with its `user` attribute.

Often, User and Session objects will be paired in many event classes.

---

## When should I use the unmapped attribute?

The `unmapped` attribute is a catchall for event producers and mappers when there is data that doesn't populate the more specific attributes of the class. For example, product specific data that is extracted into fields and values from a log that aren't mapped.

Where `unmapped` is best used, is for a mapper who is mapping events from multiple vendors where each vendor may have unique fields not common to other vendors for the same type of data source.

However, using `unmapped` is not recommended for event producers. A native event producer should extend the schema to properly capture the data that can't be mapped. For product specific data, an extension is preferred, using either a vendor developed profile, or in some cases a new event class if the core event class doesn't adequately represent the event due to data that can't be naturally mapped, or activities not captured by the core class.

---

## unmapped is of Object type. What does that mean and is it different from JSON or a String type?

Object is the empty complex data type from which all OCSF objects extend with JSON formatted attributes, requirements, and descriptions. Think of `unmapped` as if it were an OCSF object that you created on the fly. In the Java programming language, it would be like an inner class that doesn't need to be declared externally or globally. That is to say, it is used within the instance of an OCSF class only, and not part of the schema.

JSON is more free form data, hence the `data` attribute is of type JSON. It can be anything encapsulated within JSON and does not need to look like an OCSF object. It should not be used for unmapped extracted fields, but rather other data that may be captured with the event. It is used, for example, within the Enrichment object (the `enrichments` array attribute of the Base Event) to augment one or more of the mapped or unmapped attributes.

A String type is reserved for unformatted text, such as the `raw_data` attribute of the Base Event class. Binary data is Base64 encoded in an attribute of `bytestring_t` type, currently not used in the core schema but may be used in extensions or within the `unmapped` object.

---

## When should I use Authorize Session from Identity and Access Management vs. Web Resource Access Activity from the Application category?

These two event classes are complementary. Changes to a security principal's permissions, privileges, roles are authorization activities, while the access of web resources by a security principal is logged as Web Access Activity. IAM category authorization or change events are independent of a particular resource access, while enforcement of authorization restrictions is made at access time and is logged as such. For example, when a new Logon session is created, authorization checks are made and if logged, belong in the Authorize Session class. However, when the user or process that has those permissions accesses a web resource, and it is granted or denied, the Web Access Activity class is used.

---

## When should I use HTTP Activity vs. Web Resource Access Activity?

HTTP Activity is information focused on the network protocol, and not the gating of the resource. While access to a resource is often requested via a web service or REST APIs, the HTTP Activity is the protocol activity for that access, not the activity of the gating service to the resource, which might be via the HTTP server nevertheless. And of course access activity in general is not uniquely via HTTP: Kerberos and LDAP servers grant and deny access to resources over their respective protocols.

---

## Can you explain Profiles to me?

Profiles in OCSF are a way to uniformly add a set of attributes to one or more event classes or objects. Event classes provide the basic structure and type of an event, while objects provide the structure of complex types. Their definitions can indicate that additional attributes may be included with an event instance via profiles specified with the class or object definition. In effect, adding a profile or profiles to the definition gives you the permission to dynamically include those attributes. When constructing an event, you would add an OCSF profile name to the `metadata.profiles` array to mix-in the additional attributes with the event.

An event that has that profile applied is then a kind of that profile, as well as a kind of the event class. For example, if the `Host` profile was applied to the `HTTP Activity` class to add the `actor.process` making a request, the event would be queryable either via the metadata.profiles[] as `Host` or via class_name as `HTTP Activity`. If using `Host` other events from `System Activity` could also be returned with the same actor.

Not all of the attributes from the profile need be added together. For example, a profile with attributes A, B, C can be defined within the definition of class D and object E. Class D can include A and B, while object E can include attribute C. You can also build in a profile, by adding the attributes of the profile directly into your class, and referencing the profile in your class definition. In this case, as with class and object extensions, the profile defined requirements, group or description can be overridden within the definition of the class or object, although this is not recommended. Only the attribute data type and constraints cannot be overridden.

---

## Is there a similarity between OCSF and LDAP (and X.500)?

Yes there is, although OCSF is considerably simpler. At a fundamental level LDAP consists of attributes and object classes, while OCSF consists of attributes and event classes. Attributes in LDAP have syntaxes and in OCSF have data types (OCSF objects are complex data types). An event class is similar to an LDAP structural object class; it defines the basic structure of an event, as the LDAP object class defines the structure of an entry. Like LDAP, an OCSF event class can be constructed via extending a super class to inherit attributes. And an OCSF profile is similar to an LDAP auxiliary class which can be applied to a structural object class so that an entry can mix in additional attributes, independent of structural hierarchy of the entry.

---

## How should the attribute suffixes `_uid` and `_id` be used and what are "siblings?"

These are naming conventions rather than metaschema or data type validation factors. `_id` is the convention for OCSF enumerated attributes. These attributes can be integer data types, or string data types, although OCSF favors integer data types with string labels. Every integer enum attribute SHOULD have standard values of `0` for `Unknown` and `99` for `Other`. There is no requirement that the integers stay within those bounds, or that they increment by `1`. Every enum attribute SHOULD have a string sibling attribute of the same name but without the `_id` suffix. A sibling is declared within the attribute definition of the enum attribute. When the logged value is not mappable within the enum listed values, `Other` can be set and a source specific label can populate the sibling attribute. The exception to this convention is when an enum attribute mirrors an external standard, for example with the `dns_query` object's `opcode_id` which mirrors the values requested from a resolver. It is recommended that the sibling attribute is populated with the enum label so that human queries can be made against a more easily remembered string, rather than a number.

`_uid` suffix attributes are for unique identifier values within the schema, or external identifier values, e.g. coming from a public cloud resource or similar entity. For this reason `_uid` suffix attributes are usually strings, in order to accommodate any type of alphanumeric format, but they MAY be integers (or longs). Within OCSF Classification attributes, `_uid` attributes are integers or longs (see `class_uid` or `type_uid`). The sibling for `_uid` attributes is an attribute of the same base name with the `_name` suffix (see `class_name` or `type_name`). The exception for Classification attributes is `activity_id` which is an enum rather than a singular identifier. However its sibling is also of suffix `_name`: `activity_name` following the convention for `_uid` attributes of the Classification group.

Note that sibling string attributes can be used standalone, i.e. without an associated enum or unique identifier.

---

## How is backwards compatibility managed?

OCSF follows the [semver](https://semver.org/) versioning scheme.

From the semver documentation:

> Given a version number MAJOR.MINOR.PATCH, increment the:
>
> MAJOR version when you make incompatible API changes
> MINOR version when you add functionality in a backward compatible manner
> PATCH version when you make backward compatible bug fixes

In terms practical to OCSF users, this means:

- PATCH version increments may change documentation values like `description`.
- MINOR version increments may add new schemata like attributes or events.
- MAJOR version increments may add and remove anything.

So any version in the 1.x line should be backwards-compatible with previous 1.x versions.

---

## What changes are not backwards compatible?

1. The removal of an event, object, attribute, data type, or enum member.
   1. The `name` of an event or object is missing in the NEW schema.
   2. The dictionary key of an attribute or data type (its implied `name`) is missing in the NEW schema.
   3. The dictionary key of an enum member (its value) is missing in the NEW schema.
   4. The `uid` or `class_uid` of an event is missing in the NEW schema.
2. Renaming an event, object, attribute, or enum member.
   1. A special case of removal in which the same `caption` belongs to an element with a different `name`, key, or `class_uid`; or the same `class_uid` belongs to an event with a different name.
3. Changing the data type of an attribute **unless**:
   1. The data type is changing from `int` to `long`. This exception is allowed on the basis that _nearly_ all encodings use variable lengths by default, meaning that data written in nearly all encodings as an `int` can be safely interpreted as a `long`.
   2. Changing a scalar type when the underlying type (e.g. `string_t`) remains the same and there are no constraints on the new type.
4. Changing the `requirement` value of an attribute from `optional` or `recommended` to `required`.
5. Making the `constraints` of a data type more restrictive.
6. Adding a `required` attribute to an existing event or object.
7. Changing the `caption` of an event, enum member, or category.

---

## When should I use `status` and when should I use `state` when adding to the schema?

The convention we try to stick to when authoring OCSF classes and objects is to use `status_id` and its sibling `status` for the result of an activity, usually as a class attribute, and use `state_id` and its sibling `state` for the state of an object. The latter might sound obvious but it may not be obvious to not use `status` for objects. The reasoning is that an object exist independent of time or an activity or action, and therefore it has a state. It could have just as easily had a status, over an indeterminate period of time, but we have tried to distinguish between the two situations by reserving `status` for the point in time result of an activity or action.

## When should I use a `Module Activity: Load` event and when should I use a `Process Activity: Inject` event

Confusion exists here because both `Module Activity: Load` and `Process Activity: Inject` with `injection_type_id = 2 (Load Library)` can be used to represent an endpoint process loading a module.
`Process Activity: Inject` with `injection_type_id = 2 (Load Library)` represents when an actor process acts on a target process to cause that target process to load a module.
`Module Activity: Load` covers the more general case of an endpoint process loading a module which is something that endpoint processes routinely do.
Given that `Process Activity: Inject` with `injection_type_id = 2 (Load Library)` is more specific, it should be used if applicable.
