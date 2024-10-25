# Common Process Identifier (CPID)

The Common Process Identifier (CPID pronounced "see-pid") is a specification for deterministically calculating an identifier that uniquely identifies an endpoint process across an organization.

This document introduces the CPID by:
1. Exposing the problem space
2. Elaborating on additional properties desired in a solution
3. Specifying the CPID calculation procedure

## Problem Exposition
[OCSF Proposal #1205](https://github.com/ocsf/ocsf-schema/discussions/1205) introduced this topic to the OCSF community with a brief problem explanation.
This section gives a more thorough explanation.

### Unique Endpoint Process Identification
Unique endpoint process identification is a problem because major operating system families (Windows, MacOS, Linux) do not implement process identification that is rigorous enough for adversarial cybersecurity use cases.

The operating system process identifier (PID) is an integer, usually no larger than 32 bits, which is assigned to a process by the operating system at process creation.
PIDs uniquely identify a process within a given endpoint during the lifetime of that process.

However, after a process terminates, the operating system can and will reuse the terminated process's PID.
PID reuse is encountered frequently in normal circumstances depending on the operating system.
Furthermore, PID reuse can be exacerbated or induced.
An example of this is a software bug that leaves many zombie processes running on a system.
This would shrink the available PID pool and result in more PID reuse.

Given that PIDs are reused, a popular technique to achieve unique process identification within an endpoint is to combine the PID with the Process Creation Time (PCT).
A PID is unique for a process during its lifetime and the PCT identifies the start of that lifetime.

The next step is usually to add more identifying information to achieve organizationally or globally unique identification.
The most straightforward way to do this is to combine PID and PCT with some sort of unique endpoint identifier.
There are well-known examples of this approach like [Sysmon GUIDs](https://learn.microsoft.com/en-us/answers/questions/234884/structure-of-process-guids-used-in-sysmon-etw-even).
Hostname is frequently used for this purpose, but it comes with issues.

Generic unique endpoint identification is not a solved problem.
Endpoint identifiers often don't end up being unique in practice due to system misconfiguration or virtualization complications.
Additionally, some endpoint identifiers can change through administrative action during the lifetime of a process, making process identification less reliable.
These issues encourage endpoint security software implementations to rely on their own endpoint identification systems.

Still, on some platforms there can be issues with approaching process identification using endpoint identity, PID and PCT.
For example, PCT needs to be expressed as a "wall clock" value to achieve endpoint uniqueness.
Depending on the operating system, administrative clock changes can affect the reporting of this value.

Given the above complications, many endpoint security software implementations choose to generate a random unique process identifier at process creation.
There is no doubt that this approach achieves unique endpoint process identification.

### Shared Unique Endpoint Process Identification

The previous section describes how unique endpoint process identification can be achieved.
However, the two widely adopted implementation routes create identification schemes specific to the implementing software.
This disrupts downstream data consumers (e.g. SOC data analysis) when multiple endpoint security software implementations are present.
Here are a couple examples of when the problem arises:
- an organization runs 2 competing EDR products side by side
- an incident response team uses tooling in addition to their organization's EDR solution
- datasets from different endpoint focus areas (EDR, DLP, Device Management, Identity) need to be correlated

In these scenarios, the data consumer must work through the unique endpoint process identification problem using whatever data they have on hand.
PID is a universal data point that is usually present in endpoint data sets.
PCT is usually present in EDRs, but the timestamp resolution reporting can vary in addition to the timestamp source.
Assuming these two datapoints are present and processable, then data consumers are usually still left relying on hostname for endpoint identification.
This can often be good enough depending on your datasets, but it is not a general solution, and the weaknesses show when considering many endpoints.

A comprehensive solution requires a process identification system shared by the different dataset producers.

It is theoretically possible for endpoint security software creators to implement a unique process identifier consensus mechanism.
Such a mechanism could facilitate using a randomly generated identifier that guarantees unique identification.
However, synchronous coordination between software creators and the implementation of such a consensus mechanism create significant barriers.

Ideally, operating systems would implement more rigorous process identification that could be consumed by endpoint security software.
While possible, it is unlikely that all major operating systems implement this.
Additionally, this would only be a solution for operating system versions after the addition of improved process identification.

We believe a good solution is possible by rigorously specifying how to deterministically arrive at a process identifier using only information provided by the operating system.
This still requires adoption by endpoint security software creators.
However, barriers are reduced because software creators can independently adopt the specification without coordination between creators.
Additionally, the technical implementation becomes simpler since no on-endpoint communication between different endpoint security software is required.

## Solution Properties

Before getting into the CPID specification, this section covers some additional properties that are desired in a deterministic process identification scheme.

### Unique

Covered in depth above, endpoint processes should be uniquely identified.
Therefore, the process identification scheme should attempt to minimize the probability of random or systemic collisions.
There should be strong uniqueness guarantees for process identification within an organization.
Global uniqueness guarantees are desirable but not essential.
Note the trade-off between uniqueness and efficiency since larger identifiers are required to achieve greater uniqueness.

### Easy to Use

The identification scheme should be easy to use for identifying endpoint processes above all else.
There are multiple facets to this.

First, the identifier should have identical usage patterns regardless of the process's platform.
The process is a fundamental entity on all widely deployed operating systems.
Maintaining consistent experience across platforms enables data consumers to rely on the abstraction of an endpoint process instead of platform-specific details.

Second, the identifier should maximize its compatibility with software that could use it.
Making the identifier easy to consume in other software maximizes the value that data consumers can realize.

Related to maximizing compatibility is the privilege required to source identifier information from the host endpoint.
An identifier scheme requiring high-level privilege or access may be harder to adopt.
Therefore low-privilege data sources should be preferred where possible.

Another "ease of use" consideration is the ease of implementation in endpoint software.
A specification that lends itself to bugs will result in non-matching identifiers across different implementations.

### Efficient

The identification scheme should be efficient both in terms of resources required for computation and the resulting identifier size.

The process identifier size can have a large impact on the datasets it is used with.
Modern endpoint security datasets are process-focused.
Events to describe the creation of endpoint processes are fundamental.
Furthermore, other system activities are usually expressed in terms of an "actor" process that performed the activity with the actor process referred to by its unique identifier.
We want to avoid unnecessarily bloating datasets since process identifiers are used so often.

The unique process identifier should also be efficiently calculatable and usable by endpoint security software.
Endpoint security software must operate in a manner that does not disrupt the host endpoint through excessive resource consumption.
Identifier size plays a role here as endpoint software will need to keep a set of process identifiers in memory.
However, the computational resources used to calculate the identifiers has the potential to be more disruptive.
Identifier calculation will need to occur during most process creations.
Depending on the system, normal operation could involve continuously spawning many processes (e.g. a software build server).
Unique identifier calculation for every process should be enabled without adversely impacting system performance.

### Reliable

Identifiers should be reliably calculatable across the entire lifetime of a process.
This helps guarantee deterministic calculation across endpoint software that may attempt to calculate an identifier at different points in a process's lifetime.
Furthermore, identifier calculation inputs should not be (easily) alterable by attacks or other means.
Immutable inputs that are available for the lifetime of a process are best.
Inputs whose modification is detectable are preferred in the absence of immutable inputs.

### Safe

Process identifiers should be safe to rely on in downstream systems despite adversarial action.
An example of this is relying on unique process identifiers as a database primary key.
Depending on the identifier construction, it may be possible for an attacker to induce database performance issues through PID reuse or some other attack.
The identifier design should strive to avoid these possibilities.

### Private

Finally, identifiers shouldn't leak information about the endpoint or process without good justification.
It is highly likely that datasets containing process identifiers will be shared.
Risk of undesired information exposure should be minimized.

## CPID Specification

At a high level, Common Process Identifiers (CPIDs) are calculated by:
1. Sourcing information about an endpoint process from the endpoint operating system
2. Feeding this information into a SHA-256 digest
3. Retaining the first 128 digest bits and interpreting them according to the platform-native binary representation of a UUID
4. Setting the version and variant bits [according to RFC 9562 UUID Version 8](https://datatracker.ietf.org/doc/html/rfc9562)

#### Sourcing Endpoint Process Information

Subsequent sections of the specification detail the exact endpoint process information that should be sourced on each platform.
This specification is clear about which information is needed and gives instruction for how to retrieve the information both in text and through a reference implementation.
However, it is not required that the information be retrieved from the reference locations if the same binary content can be produced.

Generally, the sourced information solves the unique process identification problem by building on a PID and PCT solution.
However, the CPID differs from usual approaches by attempting to uniquely identify booted kernels / operating systems instead of solving for unique endpoint identification.
This still works for process identification since an endpoint process belongs to a single operating system boot session.

This change comes with the following advantages:
- Virtual machines with incorrectly cloned device identifiers are booted independently, and each booted instance is uniquely identified
- Linux directly offers a boot identifier, resulting in optimal uniqueness guarantees
- Unique process identification within a boot is required instead of within an endpoint, enabling the use of boot-relative monotonic clocks for PCT.

Note: The CPID generally behaves correctly in the face of virtual machine cloning that includes memory copying.
Often this is done to migrate a live virtual machine to a different physical host.
In these scenarios the same identifier value is desired before and after migration.

#### Using a Cryptographic Digest

Feeding process identifying information through a digest adds strength to the CPID's uniqueness properties.
Much of the process identifying information is not random, but there is still entropy to be effectively used.
A digest harvests this entropy effectively by mixing input information together.
Using a _cryptographic_ digest provides this property with rigor.
The result is that any entropy from inputs is used as effectively as possible for a given final identifier size.

A cryptographic digest also contributes privacy and safety to the CPID.
Preimage resistance provides privacy as it becomes infeasible to recover information from the CPID.
Second preimage resistance provides safety as it becomes infeasible to meaningfully control the final CPID information content.

Finally, using a digest provides several "ease of use" advantages.
The digest input format is not important for the final identifier if it is formatted consistently.
Therefore, the input format can be optimized for easy and efficient implementation.
One such optimization is using platform-native memory representations.
This choice removes endianness considerations from endpoint software implementations, maximizing the chance of correct implementation.
Additionally, performance benefits are possible since conversion operations are minimized, and native implementations can use packed (no padding) structs as the digest input buffer.
Another optimization is using 64-bit integers so the packed representations of digest input structs match default compiler packing behavior on 64-bit platforms when optimizing for performance.

A possible downside of using a cryptographic digest is the additional computation.
This is at least somewhat mitigated by using a fast digest function (SHA-256) that has support in modern CPU instruction sets.

Another consequence of digest use is the inability to extract information.
Extracting information from an identifier is a nice convenience.
However, it conflicts with several desired identifier properties.
Additionally, the process identifying information can be reported separately if needed.

#### Using a UUID

The CPID is an RFC-compliant UUID.
UUIDs are an easy-to-use standard that strike a good balance between uniqueness and efficiency.

RFC-compliant UUIDs have 122 bits that can be effectively used.
6 bits function as "version" and "variant" indicators that are dictated by the type of UUID.
A 128-bit binary format can be used when space saving is critical.
Otherwise, the larger string representation may be used when readability or interoperability are more important.

As a uniqueness case study, consider a very large organization with 2,000,000 endpoints.
Assume that:
- on average, these endpoints see 10,000 new processes per day that need identification
- processes should be uniquely identified across a 5-year period
- the 122 UUID bits can be effectively saturated

This example really stretches the boundaries of an organizational dataset.

Determining the probability of collision is an instance of the "birthday problem" drawing ~2^45 samples from 2^122 items.
The probability of at least one collision is 10^-10.
It is not certain that collisions will be avoided, but there is still a strong probabilistic guarantee.
Note that ability to meaningfully saturate 122 bits depends on the process identifying information sourced on a given platform.

The ubiquity of UUIDs provides "ease of use" benefits as well.
Many data storage and data processing systems have a built-in UUID type that allows easy use of UUIDs while also enabling compact binary storage.
Most programming languages also have a standard library UUID type or a well-used community library providing UUID functionality.

Given that the CPID uses a cryptographic digest, a natural choice would have been to use SHA-1 and UUIDv5.
However, RFC 9562 enables other "name-based UUIDs" through the new UUIDv8 format.
CPID's use of SHA-256 and UUIDv8 closely follows the [RFC 9562 "Example of a UUIDv8 Value (Name-Based)" appendix](https://datatracker.ietf.org/doc/html/rfc9562#name-example-of-a-uuidv8-value-n) with some deviation depending on platform.

CPID UUID construction also follows [RFC 9562 "UUID Best Practices"](https://datatracker.ietf.org/doc/html/rfc9562#name-uuid-best-practices) to maximize ease of use.
Highlights from this section:
- Monotonicity and Counters: UUIDv7 was considered to have monotonicity in the CPID. However, exposing bare information in the final identifier carries safety risks. Monotonicity could not be relied on in the face of adversarial action.
- Collision Resistance: The CPID has what RFC 9562 describes as "Low Impact" if a collision does occur.
- UUIDs That Do Not Identify the Host: This directly relates desires for privacy properties. Using a cryptographic digest alleviates these concerns.
- Opacity: Using a digest makes the CPID inherently opaque.

### Linux

The Linux kernel generates a random unique identifier per boot and makes this identifier available in `/proc/sys/kernel/random/boot_id`.
[This identifier is a UUID](https://docs.kernel.org/admin-guide/sysctl/kernel.html#random), appearing to be an RFC 9562 UUIDv4 in practice.
This reinforces the notion that the identifier is randomly generated and satisfies requirements for boot identifying information.

For boot-specific process identifying information, Linux PID namespaces present a challenge.
When a process is running in a PID namespace (e.g. inside a Linux container), the process is visible to all PID namespaces above it in the PID namespace hierarchy.
The same CPID should be produced no matter what PID namespace the CPID is calculated from.
This is accomplished by using the following 3 pieces of information for process identification:
- The PID namespace identifier for the PID namespace the process was created in. This is the deepest PID namespace in the PID namespace hierarchy associated with the process.
- The operating system assigned PID in the above PID namespace. The process will have a PID in every namespace is associated with it, up to the root PID namespace. The desired PID is from the deepest PID namespace in the PID namespace tree associated with the process.
- The PCT.

The phrase "operating system assigned PID" carries ambiguity in Linux though.
[Different threads of the same process will have a different "PID" if viewed from kernel space because they are independently scheduled](https://stackoverflow.com/a/9306150).
In kernel space, the desired value is the Linux Thread Group ID (TGID) for the process from the deepest PID namespace it belongs to.
This distinction only matters if sourcing CPID inputs in the kernel instead of userspace.
The TGID will be reference where applicable to avoid ambiguity.

The above information is exposed by Linux kernel in [the `/proc` filesystem](https://docs.kernel.org/filesystems/proc.html).

First, the PID namespace identifier for the PID namespace the process was created in is exposed in `/proc/<pid>/ns/pid` where `<pid>` is the process identifier in the PID namespace the filesystem is being accessed from.
The contents of this link are always the same no matter what PID namespace it is accessed from.
However, elevated permissions are needed to read the link if the process in question is not owned by the querying user.

Next, the operating system assigned TGID in the above PID namespace is available in `/proc/<pid>/status` where `<pid>` is the process identifier in the PID namespace the filesystem is being accessed from.
The line staring with `NStgid` gives the list of TGIDs for the process from shallowest to deepest in the PID namespace hierarchy.
This list will be longer or shorter depending on the PID namespace this file is accessed from since the TGIDs from namespaces higher up the hierarchy won't be visible.
However, the line will always end with the TGID from the deepest PID namespace in the PID namespace hierarchy that is associated with the process.
This is the desired TGID value.

Finally, the PCT is available in `/proc/<pid>/stat` where `<pid>` is the process identifier in the PID namespace the filesystem is being accessed from.
[This value is given "jiffies" or clock ticks since system boot, depending on the Linux kernel version]((https://man7.org/linux/man-pages/man5/proc.5.html)).
To simplify calculation, this value is use directly instead of normalizing to some standard time measurement.

Uniquely identifying a boot gives the CPID protection against clock modifications by using PCT relative to boot.
A wall clock PCT is created on Linux by adding the boot-relative PCT to the wall clock boot time given in `/proc/stat`.
This wall clock boot time changes with administrative action performed on the system clock (e.g. `sudo date -s ...`).
Therefore, the use of a monotonic PCT provides reliability for the CPID across the lifetime of a process.

These values are combined in the following 40-byte layout to form the CPID digest input.
```
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
| linux boot uuid                     | 128-bit RFC 9562 binary |
| process namespace id                | 64-bit unsigned integer |
| process creation time jiffies/ticks | 64-bit unsigned integer |
| namespace-specific tgid             | 64-bit unsigned integer |
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
```

The [RFC 9562 binary representation](https://datatracker.ietf.org/doc/html/rfc9562#name-uuid-format) is the binary UUID format natively supported on Linux and is implemented by `libuuid`.
Additionally, the 64-bit integer representation is the platform-native memory representation for unsigned 64-bit integers on the platform that the hash input is constructed on.
These formats enable using the memory representation of the following C struct as the digest input.
```C
#include <uuid/uuid.h>

#pragma pack(push, 1)
typedef struct {
    uuid_t boot_uuid;
    uint64_t process_namespace_id;
    uint64_t process_creation_time_ticks;
    uint64_t namespace_tgid;
} digest_input_content_t;
#pragma pack(pop)

_Static_assert(40 == sizeof(digest_input_content_t), "Linux digest_input_content_t size should be 40 bytes.");
```

`boot_uuid` can also be `uint8_t[16]` instead of `uuid_t` as long as the RFC 9562 binary representation is adhered to.

The platform-native binary representation of this data is used as the SHA-256 digest input.
The first 16 bytes of the 32-byte SHA-256 digest are kept as the base for the final UUID.
These bytes are cast to the platform-native UUID representation.

Finally, the platform-native UUID representation dictates how to properly:
- set the RFC 9562 UUID version
- set the RFC 9562 UUID variant
- convert the UUID binary representation to an RFC 9562 string

Here is an example process identifier calculation.
Consider running a container on a Linux system.
The CPID of a process inside the container is desired from outside the container.
The process has PID 39174 in the root PID namespace (outside the container).
Note that the root namespace PID is not used as an input to the SHA-256 digest.
However, it is likely used to ask the operating system for the process-specific digest input information.
```
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
| linux boot uuid             | 2899dae4-4fa4-4eef-95b6-6bc95325f61a  |
| process namespace id        | 4026532263                            |
| process creation time ticks | 55558                                 |
| namespace-specific tgid     | 29                                    |
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
```

Following the above calculation procedure gives `b770a0ed-8463-822c-b5f6-30d9081ddbd9` as the CPID.
The reference implementation provides code for realizing these operations with the above struct definition.

### Windows

Windows does not provide a boot identifier.
Therefore a synthetic boot identifier is needed.
Windows provides a canonical endpoint identifier that is reliable, but not perfect.
Therefore, unique boot identification can be achieved by building off this identifier.

The canonical Windows endpoint identifier is a GUID provided by the operating system in the Windows registry.
It is the data for the registry key `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Cryptography\MachineGuid`.
This data is an RFC 9562 UUIDv4 string, supplying 122 random bits to identify the endpoint.

Properly administered environments will use [Sysprep](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep--generalize--a-windows-installation?view=windows-11) on virtual machine images to ensure unique values.
However, this can't always be relied on.
This registry value can also be modified, but this behavior is detectable and highly abnormal.

Boots of the same endpoint are distinguished by using the operating system boot time.
Windows doesn't explicitly provide a system boot time.
Instead, the PCT of a core system process is relied on.
For Windows, this is the "System" quasi-process (PID 4).

Windows does not provide a boot-referenced PCT based on a monotonic clock.
Process start times are received as a Windows FILETIME which is a 64-bit value representing the number of 100-nanosecond intervals since January 1, 1601 (UTC) (Windows ticks).
This still serves the CPID use case well with the caveat that a system clock change can cause new processes to be created with PCTs that were previously encountered.

[Windows documentation on FILETIME](https://learn.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-filetime#remarks) does not recommend performing arithmetic on a FILETIME or casting it to another interpretation.
Therefore, FILETIME values are converted to a straightforward 64-bit integer representation of the PCT in Windows ticks before the digest calculation.
The Windows reference implementation shows this conversion in detail.

All the above information is easily retrievable through the Windows `RegGetValue` and `GetProcessTime` APIs.

These values are combined together in the following 40-byte layout to form the digest input.
```
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
| windows machine guid                 | windows guid data structure            |
| system (PID 4) process creation time | 64-bit unsigned integer windows ticks  |
| process creation time                | 64-bit unsigned integer windows ticks  |
| process identifier                   | 64-bit unsigned integer                |
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
```

The [Windows GUID binary representation](https://learn.microsoft.com/en-us/windows/win32/api/guiddef/ns-guiddef-guid) is the binary UUID format natively supported on Windows.
Additionally, the 64-bit integer representation is the platform-native memory representation for unsigned 64-bit integers on the platform that the hash input is constructed on.

These formats enable using the memory representation of the following C struct as the digest input.

```C
#include <guiddef.h>

#pragma pack(push, 1)
typedef struct {
    GUID machine_guid;
    uint64_t system_creation_time_windows_ticks;
    uint64_t process_creation_time_windows_ticks;
    uint64_t pid;
} digest_input_content_t;
#pragma pack(pop)

_Static_assert(40 == sizeof(digest_input_content_t), "Windows digest_input_content_t size should be 40 bytes");
```

The platform-native binary representation of this data is used as the SHA-256 digest input.
The first 16 bytes of the 32-byte SHA-256 digest are kept as the base for the final UUID.
These bytes are cast to the platform-native UUID representation.

Finally, the platform-native UUID representation dictates how to properly:
- set the RFC 9562 UUID version
- set the RFC 9562 UUID variant
- convert the UUID binary representation to an RFC 9562 string

Here is an example CPID calculation.

```
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
| windows machine guid                 | b3b44fe1-8a3b-4191-a91e-d3581e766fac |
| system (PID 4) process creation time | 133494576686106382                   |
| process creation time                | 133494576996587731                   |
| process identifier                   | 4992                                 |
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
```

Following the above process gives `ec88c71a-1d67-853c-a76c-3f10f2acdb6e`.

Windows has some process isolation capabilities.
That is, Windows containers running in "Process" isolation mode.
However, when a process or set of processes in run in isolation, [they still have the same PID in the isolated context as they do outside the isolated context](https://thomasvanlaere.com/posts/2021/06/exploring-windows-containers/).
For this reason, CPID calculation does not need be aware of process isolation on Windows.
Note that Windows containers with "Hyper-V" isolation have their own kernel and establishing the same identifier inside and outside the container is not supported.

### MacOS
MacOS does not provide a boot identifier.
Therefore, a synthetic boot identifier is needed.
MacOS provides canonical endpoint identifiers that are reliable, but not perfect.
Therefore, unique boot identification can be achieved by building off these identifiers.

The most widely used endpoint identifiers are the MacOS "Hardware UUID" and Serial Number.
The Hardware UUID is a SHA-1 RFC 9562 UUIDv5, with unspecified inputs (presumably based on system hardware).
The Serial Number is an alpha-numeric string that is typically 10-12 characters long, but this is not guaranteed or published by Apple.
Original plans were to use only the Hardware UUID, but the Hardware UUID and Serial Number are treated differently by different virtualization software.
That is, some vary the Hardware UUID, and some vary the Serial Number.
Therefore, there is value in using both.

There are several ways to access the Hardware UUID and Serial Number.
The reference implementation uses the MacOS IOKit library.
The digest input uses a larger constant-length buffer to hold the Serial Number ASCII string.
Unused bytes at the end of this buffer are set to the null character (`\0`).

Boots of the same endpoint are distinguished by using the operating system boot time.
MacOS provides a system boot time.
However, this value is sensitive to system clock is changes.
Reported PCTs in MacOS do not change for a given process even if the system clock is changed.
Therefore, "boot time" is based on the PCTs of always-present processes.
There are two processes on MacOS that are always present on the system:
- kernel_task (PID 0)
- launchd (PID 1)

While these processes are created immediately after boot, their creation times are not equal, so the difference provides a little more boot identifying information for the CPID calculation.
The reference implementation sources PCT values from `sysctl`.

MacOS does not provide PCTs based on a boot-relative monotonic clock.
PCTs are received from MacOS as Unix timestamps with 64 bits for Unix epoch seconds and 32 bits for a microsecond offset.
This still serves the CPID use case well with the caveat that a system clock change can cause new processes to be created with PCTs that were previously encountered.

Finally, a process is identified within a boot by its PCT and PID.

These values are combined in the following 88-byte layout to form the digest input.
```
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
| macos serial number                                     | 16 byte ascii string with unused terminating bytes set to `\0`  |
| macos hardware identifier                               | 128-bit rfc 9562 binary                                         |
| kernel_task (PID 0) process creation time seconds       | 64-bit unsigned integer                                         |
| kernel_task (PID 0) process creation time mircos offset | 64-bit unsigned integer                                         |
| launchd (PID 1) process creation time seconds           | 64-bit unsigned integer                                         |
| launchd (PID 1) process creation time mircos offset     | 64-bit unsigned integer                                         |
| process creation time seconds                           | 64-bit unsigned integer                                         |
| process creation time mircos offset                     | 64-bit unsigned integer                                         |
| process identifier                                      | 64-bit unsigned integer                                         |
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
```

The [RFC 9562 binary representation](https://www.ietf.org/archive/id/draft-ietf-uuidrev-rfc4122bis-14.html#name-uuid-format) is the binary UUID format natively supported on MacOS, implemented by `libuuid`.
Additionally, the 64-bit integer representation is the platform-native memory representation for unsigned 64-bit integers on the platform that the hash input is constructed on.
A constant 16-byte buffer is used for the MacOS Serial Number in order to enable easier memory allocation of a digest input structure.

These formats enable using the memory representation of the following C struct as the digest input.

```C
#include <uuid/uuid.h>

#pragma pack(push, 1)
typedef struct {
    uint64_t unix_epoch_seconds;
    uint64_t micros_offset;
} process_creation_time_t;

typedef struct {
    char serial_number[16];
    uuid_t hardware_uuid;
    process_creation_time_t kernel_task_creation_time;
    process_creation_time_t launchd_creation_time;
    process_creation_time_t process_creation_time;
    uint64_t pid;
} digest_input_content_t;
#pragma pack(pop)

_Static_assert(88 == sizeof(digest_input_content_t), "Windows digest_input_content_t size should be 88 bytes");
```

`hardware_uuid` can be `uint8_t[16]` instead of `uuid_t` as long as the RFC 9562 binary representation is adhered to.

The platform-native binary representation of this data is used as the SHA-256 digest input.
The first 16 bytes of the 32-byte SHA-256 digest are kept as the base for the final UUID.
These bytes are cast to the platform-native UUID representation.

Finally, the platform-native UUID representation dictates how to properly:
- set the RFC 9562 UUID version
- set the RFC 9562 UUID variant
- convert the UUID binary representation to an RFC 9562 string

Here is an example CPID calculation.

```
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
| macos serial number                                     | "T2T3GKP272\0\0\0\0\0\0"              |
| macos hardware identifier                               | 8e923375-9510-5729-a6cc-2f66444573c9  |
| kernel_task (PID 0) process creation time seconds       | 1703173115                            |
| kernel_task (PID 0) process creation time mircos offset | 212514                                |
| launchd (PID 1) process creation time seconds           | 1703173115                            |
| launchd (PID 1) process creation time mircos offset     | 282857                                |
| process creation time seconds                           | 1703174125                            |
| process creation time mircos offset                     | 741886                                |
| process identifier                                      | 1330                                  |
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
```

Following the above process gives `e6d6d95b-5d2b-8bb5-8296-02908ea24c8f`.
