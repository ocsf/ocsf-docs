# Representing Process Parentage
Mitchell Wasson
February 2025

Effectively representing endpoint process parentage is frequently discussed, because the OCSF schema has several fields that support this use case (`actor.process`, `process.parent_process`, `process.lineage` and `process.ancestry`).
This article clarifies and expands on those discussions to provide prescriptive guidance for representing process parentage within the OCSF schema.

## Actor/Creator or Parent

[Confusion on this topic]((https://github.com/ocsf/ocsf-schema/discussions/1194)) arises from the fact that the OCSF schema enables the simultaneous expression of `.actor.process` and `.process.parent_process` in `Process Activity: Launch` events.
People are usually wondering if they should put the launched process's parent in `.actor.process`, `.process.parent_process` or both.
Again, prescriptive guidance will be provided, but it is important to understand the difference between the actor (aka the "creator") and the parent in a process launch/creation event.

The creator is the process that initiated the creation of a new process with the endpoint operating system.
The parent is the process that the newly created process inherits properties from according to operating system rules.
The creator and the parent are _usually_ the same process.
Additionally, on many platforms they are guaranteed to always be the same.

However, this guarantee is not present on Windows.
[Pavel Yosifovich's blog on "Parent Process vs. Creator Process"](https://scorpiosoftware.net/2021/01/10/parent-process-vs-creator-process/) shows exactly how one can create a process on Windows with a parent different from the creator.
This is a straightforward technique and has many legitimate use cases.

Note this situation shouldn't be confused with creating a process through a layer of indirection or communicating with another common process that will create a process for you.
The above distinction of creator vs parent applies to mechanisms natively supported through operating system APIs that can't be modelled otherwise.
Process creation through a layer of process indirection (e.g. starting a shell to start your program) is modelled through two `Process Activity: Launch` events.
Asking another process to create a process is modelled through some sort of communication event and a single `Process Activity: Launch` event.

For `Process Activity: Launch` events, one should set both `.actor.process` and `.process.parent_process` if the ability to know both is present.
This will provide the visibility to know when they differ.
However, your endpoint software must be aware of this difference in order to effectively populate both locations.
If your endpoint software only reports on parent, then only set `.process.parent_process`.
However, if your query patterns demand that `.actor.process` be set, you can duplicate the parent information there knowing that this information would be the same the majority of the time anyway.
Inform your downstream data consumers this approach is being taken so they do not rely on being able to detect when creator and parent differ.

Note that there is no explicit attribute for creator inside the process object.
As mentioned above, the creator and the parent will be the same process the majority of the time.
We currently believe it is sufficient to only provide one location in which this difference is expressed: the `Process Activity: Launch` event.

Depending on the situation, the creator may be desired instead of the parent.
When a data consumer is specifically interested in the creator, they should consult the `Process Activity: Launch` event for the process in question.
It is possible to add attributes for creator to the process object in the future.
However, the added value will need to be carefully weighed against the incurred bloat.

## Extended Ancestry

OCSF primarily models process ancestry through process object recursion with the `process.parent_process` attribute.
In theory this recursion ends once you get to the root of the ancestry tree.
In practice, endpoint software must stop closer to the process in question.

Current guidance is to only populate `process.parent_process` for the top-level process object in an event.
This guidance is given in order to prevent deep nesting in events.
Additionally, there are diminishing returns to going further and further up the process tree.

The `parent_process` attribute in the top-level process object should be set if possible as the primary mechanism for communicating ancestry.
Parent process and all the fields in the process object are often critical context in security investigations.

When going beyond immediate parent, the OCSF 1.4 `process.ancestry` attribute should be used.
This attribute provides the ability to supply references to processes going up the process ancestry tree (e.g. parent, grandparent, great grandparent, ...).
The process entity objects in this array contain a small subset of process object attributes.
These fields are meant to enable a lookup of full process details and enable a basic preview of the process.
It is left up to the implementer to determine how far back to report process ancestry.

Prior to OCSF 1.4, the `process.lineage` field (now deprecated) enabled a preview of ancestry.

## `Process Activity: Launch` Sample Event

Here is a `Process Activity: Launch` event that adheres to the above guidance on Actor/Creator, Parent and Ancestry.
Note that the creator and parent are different.

```json
{
  "activity_id": 1,
  "activity_name": "Launch",
  "actor": {
    "process": {
      "ancestry": [
        {
          "cmd_line": "C:\\windows\\System32\\cmd.exe",
          "created_time": 1738156431386,
          "pid": 43548,
          "uid": "3831f89a-5b2c-8fc2-8396-794f0d877672"
        },
        {
          "cmd_line": "\"C:\\Program Files\\WindowsApps\\Microsoft.WindowsTerminal_1.21.3231.0_x64__8wekyb3d8bbwe\\WindowsTerminal.exe\" ",
          "created_time": 1737662946236,
          "pid": 10464,
          "uid": "9263fade-d82f-8780-95da-203a95408580"
        },
        {
          "cmd_line": "C:\\windows\\Explorer.EXE",
          "created_time": 1737662110682,
          "pid": 7956,
          "uid": "2d7576cb-b5f9-8eee-b209-2022749157d1"
        }
      ],
      "cmd_line": "cuckoo.exe  7956 powershell -command \"echo Hello.\"",
      "created_time": 1738271124947,
      "file": {
        "hashes": [
          {
            "algorithm": "SHA-256",
            "algorithm_id": 3,
            "value": "3dab543070797f16f874caf518807ca9d3e81376daf9eccce34d69bec2e8a7a5"
          }
        ],
        "name": "cuckoo.exe",
        "path": "C:\\Users\\test\\source\\repos\\cuckoo\\x64\\Release\\cuckoo.exe",
        "size": 169472,
        "type": "Regular File",
        "type_id": 1
      },
      "name": "cuckoo.exe",
      "pid": 61320,
      "uid": "2b2de22a-2818-8819-8bdb-67379aa6b98b"
    }
  },
  "category_name": "System Activity",
  "category_uid": 1,
  "class_name": "Process Activity",
  "class_uid": 1007,
  "device": {
    "hostname": "test hostname",
    "type": "Desktop",
    "type_id": 2
  },
  "message": "Process 61320 (cuckoo.exe) created process 14148 (powershell.exe) as a child of process 7956 (explorer.exe).",
  "metadata": {
    "product": {
      "name": "A creator-aware endpoint security product"
    },
    "version": "1.4.0"
  },
  "process": {
    "ancestry": [
      {
        "cmd_line": "C:\\windows\\Explorer.EXE",
        "created_time": 1737662110682,
        "pid": 7956,
        "uid": "2d7576cb-b5f9-8eee-b209-2022749157d1"
      }
    ],
    "cmd_line": "powershell -command \"echo Hello.\"",
    "created_time": 1738271124954,
    "file": {
      "hashes": [
        {
          "algorithm": "SHA-256",
          "algorithm_id": 3,
          "value": "3247bcfd60f6dd25f34cb74b5889ab10ef1b3ec72b4d4b3d95b5b25b534560b8"
        }
      ],
      "name": "powershell.exe",
      "path": "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
      "size": 450560,
      "type": "Regular File",
      "type_id": 1
    },
    "name": "powershell.exe",
    "parent_process": {
      "cmd_line": "C:\\windows\\Explorer.EXE",
      "created_time": 1737662110682,
      "file": {
        "hashes": [
          {
            "algorithm": "SHA-256",
            "algorithm_id": 3,
            "value": "6b45e1b4d3af9ae92c6aec095579571317924f50d12f13f0ed6a82b91c6fab83"
          }
        ],
        "name": "explorer.exe",
        "path": "C:\\Windows\\explorer.exe",
        "size": 5575536,
        "type": "Regular File",
        "type_id": 1
      },
      "name": "explorer.exe",
      "pid": 7956,
      "uid": "2d7576cb-b5f9-8eee-b209-2022749157d1"
    },
    "pid": 14148,
    "uid": "a5f0e1f1-8e89-8b58-98ea-3d5ba1a3d07e"
  },
  "severity": "Informational",
  "severity_id": 1,
  "time": 1738271124958,
  "type_name": "Process Activity: Launch",
  "type_uid": 100701
}
```

## Pre-existing Processes

Endpoint security software may report on the existence of processes that were created while the security software was not running.
For example, some processes will be created before endpoint security software is started at system boot.
Care should be taken to correctly represent process ancestry in these situations.

First, the type of event used to report process existence is important.
`Process Query: Query` events should be used when the existence of a process is reported, and `Process Activity: Launch` events should be used when a directly observed process creation is reported.
If these two situations can't be distinguished in an endpoint dataset, then one may use `Process Activity: Launch` events for both cases.

Communicating this distinction through `Process Query: Query` and `Process Activity: Launch` event types is greatly preferred,
because more information is available to endpoint security software if it directly observes a process creation.
Most importantly, information on process creator is only available at process creation time.
Additionally, if reporting on an already existing process, that process's parent may have terminated already and there will be little information available about it.
Using two different event types allows one to easily communicate a different schemas to data consumers based on event type.

When reporting on already existing processes, `.actor.process` should not be supplied to reflect that no information on process creator is available.
`Process Query: Query` does not have the `actor` attribute by default, so this advice only applies if enabling the Host profile.
`.process.parent_process` should be set and populated if the reported process's parent hasn't terminated yet.
If the parent process has terminated, then it is likely that only the parent's PID will be available.
In this situation `.process.parent_process` may be set with only the `pid` attribute supplied, or the PID can be reported in the `process.ancestry` array.

## `Process Query: Query` Sample Event

Here is a `Process Query: Query` event that adheres to the above guidance pre-existing processes where creator information is not available.

```json
{
  "activity_id": 1,
  "activity_name": "Query",
  "category_name": "Discovery",
  "category_uid": 5,
  "class_name": "Process Query",
  "class_uid": 5015,
  "message": "Process 8732 (svchost.exe) is pre-existing.",
  "metadata": {
    "product": {
      "name": "A creator-aware endpoint security product"
    },
    "version": "1.4.0"
  },
  "process": {
    "ancestry": [
      {
        "cmd_line": "C:\\Windows\\system32\\services.exe",
        "created_time": 1738348318013,
        "pid": 864,
        "uid": "88b1f102-1c47-8051-8f81-728b01bd7343"
      },
      {
        "cmd_line": "wininit.exe",
        "created_time": 1738348317989,
        "pid": 952,
        "uid": "06b14610-be3d-8a4a-b7c1-e41c25f0fbe3"
      }
    ],
    "cmd_line": "C:\\Windows\\System32\\svchost.exe -k swprv",
    "created_time": 1738349093722,
    "file": {
      "hashes": [
        {
          "algorithm": "SHA-256",
          "algorithm_id": 3,
          "value": "6fc3bf1fdfd76860be782554f8d25bd32f108db934d70f4253f1e5f23522e503"
        }
      ],
      "name": "svchost.exe",
      "path": "C:\\Windows\\System32\\svchost.exe",
      "size": 57528,
      "type": "Regular File",
      "type_id": 1
    },
    "name": "svchost.exe",
    "parent_process": {
      "cmd_line": "C:\\Windows\\system32\\services.exe",
      "created_time": 1738348318013,
      "file": {
        "hashes": [
          {
            "algorithm": "SHA-256",
            "algorithm_id": 3,
            "value": "1efd9a81b2ddf21b3f327d67a6f8f88f814979e84085ec812af72450310d4281"
          }
        ],
        "name": "services.exe",
        "path": "C:\\Windows\\System32\\services.exe",
        "size": 716544,
        "type": "Regular File",
        "type_id": 1
      },
      "name": "services.exe",
      "pid": 864,
      "uid": "88b1f102-1c47-8051-8f81-728b01bd7343"
    },
    "pid": 8732,
    "uid": "52bc2416-2c6b-811c-a031-43ba9b58ec7e"
  },
  "query_result": "Exists",
  "query_result_id": 1,
  "severity": "Informational",
  "severity_id": 1,
  "time": 1738792775574,
  "type_name": "Process Query: Query",
  "type_uid": 501501
}
```
