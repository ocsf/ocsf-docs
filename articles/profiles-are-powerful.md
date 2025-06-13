# Profiles are Powerful
Paul Agbabian

I’ve mentioned OCSF Profiles in blogs, but I want to go into more detail here, as they are becoming more important and sometimes misunderstood as to how they can be constructed.  There are four ways of modeling using profiles:

1. Augmentation profiles
2. Native profiles
3. Partially native profiles
4. Hybrid profiles

An OCSF Profile is a framework construct that cuts across categories and classes to augment classes and objects with focused ‘mix-in’ attributes that better describe aspects of activities and findings in certain situations.  Rather than have an explosion of classes that combine attributes for these situations, profiles are an elegant way of reusing the semantics of fundamental classes without extending them with new classes. If you are a Java or C++ developer, they will resemble implementing additional interfaces on top of a class, and similarly, in OCSF, Profiles are an event type that cuts across event classes.

Hence a profile is two things: a mix-in attribute set and an alternate typing of the event class or object where it is registered.  This is accomplished via a “profiles” array at the head of the class or object.  The OCSF schema server will take care of filtering or augmenting classes and objects appropriately.  In this way, a related set of attributes can be added selectively independent of class or category when its type cross-cuts the structural taxonomy.  For example, the Host profile can be applied to the Network Activity category classes for host-based network activity coming from an EDR security agent.  Querying on events WHERE “Host” IN metadata.profiles[] retrieves all events from the System Activity category and the Network Activity classes.

```
{
 "description": "The attributes that identify host/device attributes.",
 "meta": "profile",
 "caption": "Host",
 "name": "host",
 "annotations": {
   "group": "primary"
 },
 "attributes": {
   "device": {
     "requirement": "recommended"
   },
   "actor": {
     "requirement": "optional"
   }
 }
}
```

## Augmentation Profiles

The most common way of designing and using a profile is to define it in the metaschema profiles folder via a profile name and the profile attributes, as above; then declare the profile in the class or object, and finally include the profile to bring in its attributes, as below; the attributes will be added when the profile is applied to an event class or object.  

```
{
 "caption": "Network",
 "category": "network",
 "description": "Network event is a generic event that defines a set of attributes available in the Network category.",
 "extends": "base_event",
 "name": "network",
 "profiles": [
   "host",
   "network_proxy",
   "security_control",
   "load_balancer"
 ],
 "attributes": {
   "$include": [
     "profiles/host.json",
     "profiles/network_proxy.json",
     "profiles/security_control.json",
     "profiles/load_balancer.json"
   ],
...
```

This is the augmentation profile approach.  When the profile is enabled in the schema browser, the respective classes and objects are augmented with the profile attributes, and schema samples will include the profile name in the metadata.profiles[] array, effectively typing the event or object as a kind of the profile.  

```
{
  "type_name": "Network Activity: Open",
  "activity_id": 4,
  "type_uid": 400104,
  "class_uid": 4001,
  "category_uid": 4,
  "class_name": "Network Activity",
  "metadata": {
    "version": "1.1.0",
    "profiles": ["host"]
  }
  "category_name": "Network Activity",
  ...
```

All events matching the profile will be returned if an event is queried by its profile name, irrespective of class or category.  However, there are three other ways to use profiles in the schema.

## Native Profiles

The second approach is where the attributes of a profile definition are already natively defined within the event class or object.  Think of this as the built-in or native profile approach.  For the profiles system and typing to be consistent, those classes and objects must declare the profile within the class as with the augmentation approach. Still, there is no need to include the profile in the attributes section since those attributes (in the case of the Host profile, actor and device) are already defined there.

```
{
 "caption": "System Activity",
 "category": "system",
 "extends": "base_event",
 "name": "system",
 "profiles": [
   "host",
   "security_control"
 ],
 "attributes": {
   "$include": [
     "profiles/security_control.json"
   ],
   "actor": {
     "group": "primary",
     "requirement": "required"
   },
   "device": {
     "group": "primary",
     "requirement": "required"
   }
 }
}
...
```

## Partially Native Profiles

What happens when only some of the attributes of the profiles are native to an event class or object?  This is the partially native profile approach.  Using the augmentation profile approach, where the profile is $included into the class or object, the schema server will remove the native attributes when the profile is not applied, which isn’t what you would want.  For these cases, a “profile”: null statement should be added to the potentially affected native attribute, which tells the server to leave it alone regardless of the profile application.  In the example below, actor is native to the Authentication class, but device is not.  When the profile is applied, only device will be added, and when not applied, actor will stay put.

```
{
 "caption": "Authentication",
 "extends": "iam",
 "name": "authentication",
 "uid": 2,
 "profiles": [
   "host"
 ],
 "attributes": {
   "$include": [
     "profiles/host.json"
   ],
   "actor": {
     "description": "The actor that requested the authentication.",
     "group": "context",
     "profile": null
   },
...
```

## Hybrid Profiles

Finally, what if a class or object wants to be considered as part of the profile family but wants to add new attributes that are only relevant to the one particular class or object?  This may sound a bit esoteric, but it has already been used in the resource_details object for the Cloud profile. When the Cloud profile is applied to classes with attributes of the resource_details object type, for example, API Activity, the cloud_partition and region attributes defined within the object are added, but only when the Cloud profile is applied to the class.  The event now includes the api and cloud attributes, while the resource_details object of the class adds the other two attributes - effectively creating a custom hybrid profile.

If you $included the profile attributes, as with the augmented profile, you would also get the Cloud profile’s attributes in the object as well as the class. You don’t want to duplicate those attributes applied by the profile to the class into the objects too.  To make the object’s native attributes aware of the profile (such that the server switches them on, and the event validator won’t complain), you add “profile”: <profile name> within your object’s attribute clause, as well as the usual declaration within the profiles array at the head of the class or object.

The example below assigns the Cloud profile to the specific native attributes cloud_partition and region of the Resource Details object.  These attributes are not part of the Cloud profile definition, so only this specific object will include them when the Cloud profile is applied to its enclosing class.  In this way, applying a profile can add its attributes to a class, and different attributes can be added to an object within that class.

```
{
 "caption": "Resource Details",
 "extends": "_resource",
 "name": "resource_details",
 "profiles": ["cloud"],
 "attributes": {
   "agent_list": {
     "requirement": "optional"
   },
   "cloud_partition": {
     "profile": "cloud",
     "requirement": "optional"
   },
   "owner": {
     "description": "The service or user account that owns the resource.",
     "requirement": "recommended"
   },
   "region": {
     "description": "The cloud region of the resource.",
     "profile": "cloud",
     "requirement": "optional"
   },
...
```
