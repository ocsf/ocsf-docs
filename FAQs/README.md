# Frequently Asked Questions

## What is OCSF?
Open Cybersecurity Schema Framework (OSCF)
is an open-source effort to create a common schema
for security events across the cybersecurity ecosystem.

See [this whitepaper](https://github.com/ocsf/ocsf-docs/blob/main/Understanding%20OCSF.pdf)
for more info.

## What Problems does OCSF solve for?
One of the primary challenges of cybersecurity analytics
is that there is no common and agreed-upon format
and data model for logs and alerts.
As a result, pretty much everyone in the space creates
and uses their own format and data model
(IE sets of fields).  
There are *many* such models that exist,
including some open ones like
STIX, OSSEM, and the Sigma taxonomy.
The challenge to date is that none of these
models have become widely adopted by practitioners
for logging and event purposes,
and thus it requires a lot of manual work
in order to derive value.
This poses a challenge to
detection engineering, threat hunting,
and analytics development,
not to mention AI – as Rob Thomas said,
“There is no AI without IA”.
Despite the issues this causes in the industry,
there has been no significant progress on the problem space,
because until now there has been lack of a “critical mass”
of major players willing to tackle the problem head-on, and
with efforts like this, timing is everything.
With OCSF,
we are now at a moment where we have 
that critical mass as well
as a real willingness to tackle these challenges.

## How can I contribute to OCSF?
See the
[OCSF Contribution Guide](https://github.com/ocsf/ocsf-schema/blob/main/CONTRIBUTING.md)

## What is OCSF Governance Model?
See [OCSF Governance](https://github.com/ocsf/governance/blob/main/Governance.md)

## How does OCSF relate to STIX?
OCSF and STIX are compatible and complementary.  While STIX is focused on threat intelligence, campaigns and actors, OCSF is focused on events representing the activities on computer systems, networks and cloud platforms that may have security implications.  Observables represented OCSF can be matched with IOCs from STIX, for example, to determine whether a threat or malicious actor has compromised a system or enterprise environment.

Structured Threat Information Expression (STIX™)
is a open-source language and serialization format
used to exchange cyber threat intelligence (CTI).
For more info on STIX, see
[this info](https://oasis-open.github.io/cti-documentation/stix/intro.html)
or the
(spec itself](https://docs.oasis-open.org/cti/stix/v2.1/csprd01/stix-v2.1-csprd01.html)

## How does OCSF relate to the Sigma taxonomy?
Sigma is a SIEM language format for detection rules.
Sigma rules can be written against OCSF events and complement OCSF.  The
essence of Sigma is the logic of what to look for
within events to yield security findings.

See
[Sigma Taxomomy](https://github.com/SigmaHQ/sigma/wiki/Taxonomy)
for more info on it.

## How does OCSF relate to Kestrel?
OCSF and Kestrel are complementary, solving different problems.

The Kestrel Threat Hunting Language
provides an abstraction for threat hunters
to focus on what to hunt instead of how to hunt.
See their
[repo](https://github.com/opencybersecurityalliance/kestrel-lang)
for more information.
 
## How does OCSF relate to OSSEM?

Open Source Security Events Metadata (OSSEM)
is a community-led project focused
primarily on the documentation,
standardization and modeling of security event logs.
See [OSSEM repo](https://github.com/OTRF/OSSEM).

## How does OCSF relate to OpenC2?
OCSF and OpenC2 are complementary.

OpenC2 is a standardized language
for the command and control of technologies
that provide or support cyber defenses.
By providing a common language
for machine-to-machine communication,
OpenC2 is vendor and application agnostic,
enabling interoperability
across a range of cyber security tools and applications.
The use of standardized interfaces and protocols
enables interoperability of different tools,
regardless of the vendor that developed them,
the language they are written in
or the function they are designed to fulfill.
For more info on OpenC2, see
[info](https://openc2.org/).


