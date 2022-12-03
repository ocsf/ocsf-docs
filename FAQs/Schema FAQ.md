# Schema FAQ
This document answers common questions about how to use the OCSF Schema

## How do I create a typical OCSF event?
Depending on the type of event, a data producer or data mapper should first determine what event class best suits your event.  Start with the OCSF category to narrow down the choices.  For example, an endpoint security product would likely choose an event class from the System Activity category, for example, File System Activity for an AV product.

Since endpoint security products typically send alert events when malware is detected, the producer or mapper would apply the Malware profile, which adds important attributes to the File System Activity event class, e.g. a Malware object, a MITRE ATT&CK object, etc.  These objects have their own attributes that must be populated.

If your endpoint security product also has network security capabilities, you would choose an event class from the Network Activity category, for example the general Network Activity event class.  Given that the endpoint product will have information about the host system, you would apply the Host profile, as well as the Malware profile.  The Host profile includes attributes about the device and the actor (e.g. process or user) on the host.

Every OCSF event must have all of its event class Required attributes populated, and should have its Recommended attributes populated, if possible.  This includes any of the embedded objects, such as the Malware, Process and Device objects above.

All OCSF events have a set of required classification attributes from the Base Event class: the `class_uid` the `category_uid` the `activity_id` and the derived `type_uid`.  Their associates `*_name` attributes are optional.

In addition to the classification attributes, a number of other Base Event class attributes are required and must be populated: the `time` `metadata` and `severity` attributes.  The `metadata` attribute is an object that itself requires the `product` and associated `version` of the reporting event.

Note that the product should be the originating event producer (i.e. not the mapping system, nor any intermediary event processing systems) in order to best represent the origin of the event.  The `time` should be the time that the event actually occurred assuming that information is known, or the earliest possible time available to the event producer or mapper.

Although the `observables` array attribute is optional, populating it can make things easier for event consumers and analysts.  Each Observable object surfaces an important attribute of the event in a common location in a simple tuple: name, value, type.  For example, if the event class has a `device` `user` and `process` populated, an array of three Observable objects will refer to them in a common location to all OCSF events.

---

## When should I use a Security Finding event class?

A Finding in OCSF represents the result of some type of enrichment, correlation, aggregation, analysis or other processing of one or more events or alerts, producing a derived insight.  Most security events and alerts are activity events with a dispostion (e.g. Blocked).  Findings in OCSF are not alerts, although alerts may be triggered by findings or findings might be added to an incident further downstream.

For example, an email security product may determine that a user has been phished or an email attachment is malicious.  It would send an email activity event (from its standpoint an alert) containing the user and sender, supplemented by the Malware profile with a disposition of Blocked, and information about the Malware, to its management console which in turn sends it to a SIEM.  

The SIEM might receive other related events or alerts, for example for other users in the same circumstance or for general email activity from the same sender.  The SIEM might enrich the events with information from a Threat Intelligence Platform or threat feed pertaining to the email sender.  The result of the aggregation, and enrichment would constitute an OCSF Finding.  The SIEM might create an incident that includes or refers to the finding, in the event that there are remediation steps required.

Note that in a more complex processing architecture, there may be layered findings.  That is, the original event may go to product A which eventually triggers a finding. Product B meanwhile may take in a lot of other events and findings (including those from product A) and make its own findings. In the example above, the originating email alert might have been a finding from the producer's standpoint if the event was enriched by its management system before being collected by the SIEM, which then produced a more complete finding.

---

## When should I use metadata.correlation_uid?

When an event producer or mapper emits multiple events that have some grouping characteristic, or similarity of any form, it should populate the `metadata.correlation_uid` attribute with a constant identifier.  This allows consumers and analysts of the set of events to more easily aggregate and correlate the events.

A simple example would be a vulnerability scanner that emits events at the start of a scan of a system, at the end of the scan, and separate events for each vulnerability discovered.  If these are separate events, they would all have their `metadata.correlation_uid` set to the same value.

It is possible for an intermediary system to determine the grouping characcteristic as well, populating the attribute after collection of the events, although when OCSF events are  immutable a copy of the original events would be made with added correlation information.  See the next question.

---

## Can Finding events be correlated with each other too?

Yes, they are also events with a base class metadata object that can follow the same pattern.
E.g. a SIEM that creates findings may have enough knowledge and state to tie multiple findings together with a metadata.correlation_uid.

## How do I use the Actor object?
The Actor object is intended for use in event classes when knowledge of one entity that is initiating or causing some action on another entity.  For example, if one process spawns another process, or deletes a file, the first process is the actor in the activity.

From a structural standpoint, the `actor` attribute avoids name collisions with the other end of the activity in cases where a process acts on another process, as those attribute names would be in contention at the same level within the class.

Currently the Actor object has a `process` and `user` attribute, where usually one or the other is in the role of the actor in the activity.

---

## When should I use the unmapped attribute?
The `unmapped` attribute is a catchall for event producers and mappers when there is data that doesn't populating the more specific attributes of the class.  For example, product specific data that is extracted into fields and values from a log that aren't mapped.

Where `unmapped` is best used, is for a mapper who is mapping events from multiple vendors where each vendor may have unique fields not common to other vendors for the same type of data source.

However, using `unmapped` is not recommended for event producers.  A native event producer should extend the schema to properly capture the data that can't be mapped.  For product specific data, an extension is preferred, using either a vendor developed profile, or in some cases a new event class if the core event class doesn't adequately represent the event due to data that can't be naturally mapped.

