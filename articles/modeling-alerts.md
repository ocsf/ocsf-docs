# How to Model Alerts with OCSF

Through version 1.3 of OCSF the concept of an alert, or alertable signal, was not explicitly defined. In practice any event might be considered an alert, if it was deemed important and in many cases worthy of some additional form of notification. All OCSF events have a required `severity_id` attribute. Elevated values such as `Major` or `Critical` could be interpreted to be alertable signals, indirectly. Examples of alertable signals then might be events with high severity, elevated risk scores, detected malware, MITRE ATT&CK annotations, and rule or policy violations.  Other alertable signals might be the creation of a Finding based on some type of analysis.

`Security Control` profile events, available since version 1.0, are often alerts (the profile was factored out from a series of 'Detection' events). More directly, `Detection Findings` when created might be considered alertable signals. However, there was no way to express the intent of the event producer or event mapper that the event should be interpreted as an alert. And yet alerts are extremely important and prevalent in security event processing.

In OCSF version 1.4, an explicit alertable signal is an event with the `is_alert` attribute set to `true`.  This is a newer attribute that is not required.  The intent of `is_alert = true` is to signal that the event may require immediate attention by its consumer, which might be a stream processor, a SIEM system or the product’s management console where an analyst can be notified, tickets created or the events prioritized.

Not all OCSF events are potential alertable signals, i.e. carry the `is_alert` attribute, and as of this writing only the `Detection Finding`, `Data Security Finding` event classes and the `Security Control` profile carry the `is_alert` boolean attribute.  Note the `Security Control` profile may be applied to many activity oriented classes, as well as the two aforementioned `Finding` classes, hence when applied any instances of these classes can be explicit alertable signals.


## Detection Finding and Security Control Alerts

There is a fundamental difference between `Detection Finding` (or `Data Security Finding`) events and `Security Control` profile augmented events (aside from when the profile is applied to `Detection Finding` or `Data Security Finding`).  

### What is a Security Control?

`Security Control` profile attributes represent the augmentation of ordinary activities monitored by a technical security control program or sensor, or an access control system. This profile has been available since version 1.0 but has been expanded to address access control events and risk scoring.

An Intrusion Detection System (IDS), Intrusion Prevention System (IPS) sensor, firewall, anti-malware agent, or Data Loss Prevention (DLP) agent are security controls.  Role Based Access Control (RBAC) enforcement points are security controls. They monitor normal activities watching for suspicious or malicious activity, or policy violations. These controls usually emit single events that include the attempted activity (the class's `activity_id`) along with the control’s judgements (for example risk level or MITRE Techniques), and disposition of what was done in real time.  

For example, the file open activity was blocked, the control detected malware and quarantined the file. The file access was denied due to policy. A firewall rule was violated and the connection denied. These events are generally consumed by an incident management system or a SIEM in the form of alerts. In these example cases, the `is_alert` attribute should be set to `true` and the `severity_id` set to an appropriate level.  For example, if malware was detected but blocked, the severity might be low but the event may still be considered an alertable signal.

### What is a Detection Finding?

`Detection Finding` is a complex class that evolved from the original `Security Finding` in version 1.0 and has a lifecycle: the `activity_id`s are Create, Update, Close. `Detection Finding`s are best suited to systems that consume other events, perform some analysis on them, detect something suspicious or malicious and then have further investigation and workflow, compiling evidence, updating the finding, possibly including it into an `Incident Finding`, assigning it to an analyst, and ultimately closing it out. An MSSP or a SIEM might consume alerts and convert them into `Detection Finding`s. 

The `Analytic` object is a required attribute of `Detection Finding`, which describes the type of analysis done on the event or events. `related_events` refer to other events that are relevant to the Finding and were analyzed by the analytic algorithm, for example machine learning or anomaly detection.  

Examples of products that create `Detection Finding`s would be User and Entity Behavioral Analysis (UEBA) systems, SIEMs, and Endpoint Detection and Response (EDR) systems. EDRs are systems having both an agent that can monitor activities as well as an analysis system that can apply analytics to the activity events to determine a detection. In some cases the producing agent can make the determination at the point of the threat, while in other cases the associated analysis system will do so.  Hence an EDR agent or similar may need to apply the `Security Control` profile to the `Detection Finding` class to produce a detection but also a disposition.

When a Detection Finding is created (`activity_id = 1 Create`), `is_alert` may be set to `true` to indicate the detection is an alertable signal to a system, e.g. a ticketing system.  However, other lifecycle Finding events such as `Update` and `Close` activities are not likely to be considered alertable signals. `is_alert` would be set to `false` or omitted.

## Conclusion
In conclusion, one can use the `is_alert` attribute and set it to `true` when applying the `Security Control` profile to monitored activities for alertable actions and dispositions.  Set the `is_alert` attribute to `true` with a `Detection Finding` or `Data Security Finding` event when they are created to signal an alertable detection. Apply the `Security Control` profile to `Detection Finding` or `Data Security Finding` when the analytic on multiple events happens at a control or enforcement point.