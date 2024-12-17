# Representing Process Parentage
Mitchell Wasson
December 2024

Effectively representing endpoint process parentage is a topic that frequently appears in the OCSF slack.
This article clarifies and expands on those discussions to provide presecriptive guidance of representing process parentage within OCSF events and objects.

## Actor or Parent

[Confusion on this topic]((https://github.com/ocsf/ocsf-schema/discussions/1194)) arises from the fact that the OCSF schema enables the simultaneous expression of `.actor.process` and `.process.parent_process` in `Process Activity: Launch` events.
People are usually left wondering where they should put the launched process's parent in `.actor.process`, `.process.parent_process` or both.
Again, presecriptive guidance will be provided, but it is important to understand the difference between the actor (aka the "creator") and the parent in a process launch/creation event.

The creator is the process that initiated the creation of a new process with the endpoint operating system.
The parent is the process that the newly created process inherits properties from according to endpoint operating system specific rules.
The creator and the parent are _usually_ the same process.
Additionally, on many platforms they are guaranteed to always be the same.

However, this guarantee is not present on Windows.
[Pavel Yosifovich's blog on "Parent Process vs. Creator Process"](https://scorpiosoftware.net/2021/01/10/parent-process-vs-creator-process/) shows exactly how one can create a process on Windows with a parent different from the creator.
This is a straightforward technique and has many legitimate use cases.

Note this situation shouldn't be confused with creating a process through a layer of indirection or communicating with another common process that will create a process for you.
The above disinction of creator vs parent applies to mechanisms natively supported through operating system APIs that can't be modelled otherwise.
Process creation through a layer of indirection (e.g. starting a shell to start your program) is modelled through two `Process Activity: Launch` events.
Asking another process to create a process is modelled through some sort of communication event and a single `Process Activity: Launch` event.

For `Process Activity: Launch` events, one should set both `.actor.process` and `.process.parent_process` if you have the ability to know both.
This will provide the visiblity to know when they differ.
However, your endpoint software must be aware of this difference is in order to effectively populate both locations.
If your endpoint software only reports on parent, then I advise only setting `.process.parent_process`.
However, if your query patterns demand that `.actor.process` be set, you can duplicate the parent information there knowing that this information would be the same the majority of the time.
Inform your downstream data consumers this approach is being taken so they do not rely on being able to detect when creator and parent differ.

Note that there is no explicit field for creator inside the process object.
This is somewhat intentional at this point.
As mentioned above, the creator and the parent will be the same process the majority of the time.
I believe it is sufficient to only provide one location in which this different is expressed: the `Process Activity: Launch` event.
Parentage is usually what people are interested in when looking at endpoint data, and duplicating the same process into a creator field would bloat datasets and provide little value.
In cases where the creator is specifcally of interest, the `Process Activity: Launch` event for the process in question can be consulted.

# How Far Up the Tree?

OCSF models the process parentage tree with recursion inside the process object.
In theory this recursion ends once you get to the root of the parentage tree.
In practice, endpoint software must stop somewhere closes to the process in question.

There is no real consensus on what the correct point to stop this recursion is.
Still, there are a few generaly trends across endpoint datasets.

Including the direct parent wherever a process appears is usually good practice.
It provides context that is often useful to security practitioners.
However, if dataset size and normalization were priorities, one could choose to only include the `process.parent_process.uid` field and expect data consumers to consult `Process Activity: Launch` events for parentage information.

Grandparent also provides context that is often useful to security practitioners.
However, full grandparent information is reported less often thorugh `process.parent_process.parent_process`.
If included at all, grandparent or further will be reported with a subset of the attributes reported for the parent.

Generally, there are diminishing returns to going further and further up the process tree.
At some point data consumers will needs to consult multiple `Process Activity: Launch` events when extended parentage is required.

# Parentage Summaries

A shortcut built into the OCSF schema is the idea of process parentage summaries.
Currently this appears in the `process.lineage` field which is an array of the executable image paths for a process's parentage.
This summary is useful to report this most relevant piece of a process's parentage while omitting other details.

