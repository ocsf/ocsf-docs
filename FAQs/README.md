# Frequently Asked Questions

Welcome to the OCSF FAQ section! Here you'll find answers to common questions about the Open Cybersecurity Schema Framework.

## 📋 Available FAQs

- **[Schema FAQ](schema-faq.md)** - Technical questions about the OCSF schema

---

## 🔍 General Questions

### What is OCSF?

The **Open Cybersecurity Schema Framework (OCSF)** is an open-source effort to create a common schema for security events across the cybersecurity ecosystem.

📖 **Learn More**: See our [Understanding OCSF guide](../overview/understanding-ocsf.pdf)

### What problems does OCSF solve?

One of the primary challenges of cybersecurity analytics is that there is no common and agreed-upon format and data model for logs and alerts. As a result, pretty much everyone in the space creates and uses their own format and data model (IE sets of fields).

There are many such models that exist, including some open ones like STIX, OSSEM, and the Sigma taxonomy. The challenge to date is that none of these models have become widely adopted by practitioners for logging and event purposes, and thus it requires a lot of manual work in order to derive value. This poses a challenge to detection engineering, threat hunting, and analytics development, not to mention AI – as Rob Thomas said, “There is no AI without IA”. Despite the issues this causes in the industry, there has been no significant progress on the problem space, because until now there has been lack of a “critical mass” of major players willing to tackle the problem head-on, and with efforts like this, timing is everything. With OCSF, we are now at a moment where we have that critical mass as well as a real willingness to tackle these challenges.

OCSF brings together major industry players to create a standardized approach.

### How can I contribute to OCSF?

See the [OCSF Contribution Guide](https://github.com/ocsf/ocsf-schema/blob/main/CONTRIBUTING.md).

### What is the OCSF governance model?

See [OCSF Governance](https://github.com/ocsf/governance/blob/main/Governance.md) for details.

---

## 🔗 How OCSF Relates to Other Standards

### STIX (Structured Threat Information Expression)

**Relationship**: Compatible and complementary

OCSF and STIX™ are compatible and complementary. While STIX is focused on threat intelligence, campaigns and actors, OCSF is focused on events representing the activities on computer systems, networks and cloud platforms that may have security implications. Observables represented OCSF can be matched with IOCs from STIX, for example, to determine whether a threat or malicious actor has compromised a system or enterprise environment.

📖 **Learn More**: [STIX Documentation](https://oasis-open.github.io/cti-documentation/stix/intro.html) | [STIX Specification](https://docs.oasis-open.org/cti/stix/v2.1/csprd01/stix-v2.1-csprd01.html)

### Sigma Taxonomy

**Relationship**: Complementary

Sigma is a SIEM language format for detection rules. Sigma rules can be written against OCSF events and complement OCSF. The essence of Sigma is the logic of what to look for within events to yield security findings.

📖 **Learn More**: [Sigma Taxonomy](https://github.com/SigmaHQ/sigma/wiki/Taxonomy)

### Kestrel Threat Hunting Language

**Relationship**: Complementary, solving different problems

The Kestrel Threat Hunting Language provides an abstraction for threat hunters to focus on what to hunt instead of how to hunt.

📖 **Learn More**: [Kestrel Repository](https://github.com/opencybersecurityalliance/kestrel-lang)

### OSSEM (Open Source Security Events Metadata)

**Relationship**: Similar goals, different approaches

Open Source Security Events Metadata (OSSEM) is a community-led project focused primarily on the documentation, standardization and modeling of security event logs.

📖 **Learn More**: [OSSEM Repository](https://github.com/OTRF/OSSEM)

### OpenC2

**Relationship**: Complementary

OpenC2 is a standardized language for the command and control of technologies that provide or support cyber defenses. By providing a common language for machine-to-machine communication, OpenC2 is vendor and application agnostic, enabling interoperability across a range of cyber security tools and applications. The use of standardized interfaces and protocols enables interoperability of different tools, regardless of the vendor that developed them, the language they are written in or the function they are designed to fulfill.

📖 **Learn More**: [OpenC2 Information](https://openc2.org/)

---

## 🆘 Need More Help?

- **Schema-specific questions**: Check the [Schema FAQ](schema-faq.md)
- **Getting started**: Visit our [Getting Started guide](../getting-started/)
- **Technical articles**: Browse our [Articles section](../articles/)
