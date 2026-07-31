# Tamper-Evident, Attributable Event Records
Jeff Leva

OCSF describes *what happened* — an authentication, a policy decision, a file deletion, an AI agent invoking a tool. But a record rarely stays where it was born. It is produced on an endpoint, enriched at a gateway, then stored and analyzed in a SIEM or data lake. By the time someone reads it as *evidence* — for an audit, an incident investigation, or a regulated or legal review — two questions matter that the schema historically could not answer from the record itself:

1. **Was it altered?** Between the moment it was produced and the moment it is read, did any field change — by mistake, or on purpose?
2. **Is its source verifiable?** Can a third party confirm where it came from, without having to trust every hop in the pipeline that carried it?

Today, a record quietly edited in storage — or a forged event slipped into the stream — looks identical to a genuine one. Nothing in the OCSF record reveals the difference. The `record_integrity` profile and its `attestation` object close that gap. They let a producer attach cryptographic proof that a record has not been altered since it was written, and — when signed — that it came from the holder of a verifiable identity. Crucially, that proof is checkable by anyone, offline, with nothing but the exported JSON.

This article walks through the mental model, where the construct lives in the schema, what the object carries, and how a third party verifies it.

## A chain: each record seals the one before it

The core idea is a hash chain. Each event carries a fingerprint of its own canonical bytes, and a reference to the event before it that includes that event's fingerprint:

- `fingerprint` — the fingerprint of *this* event's canonical serialization, as a standard OCSF `fingerprint` object (algorithm, encoding, and the canonical-serialization scheme that produced the bytes).
- `prev_event` — a reference to the previous event in the chain: its `uid` (the previous event's `metadata.uid`, which is how OCSF refers to another event), its `type_uid` (which directs a consumer to the class, and therefore the store, where that event lives), and its `fingerprint`. The locator attributes serve retrieval; the fingerprint is what binds the link to content. Absent on the first, or genesis, event of a chain.
- `chain_uid` — an identifier for the chain as a whole (for example, a single agent session or an append-only audit log), so a verifier can locate and order its members. It names the chain itself; it is not a pointer to anything outside the event.
- `uid` — identifies this individual attestation, distinct from the chain it belongs to.
- `authority_uid` — identifies the authority that produced the attestation. Because signing credentials rotate and expire, this is what lets a verifier confirm a signature came from the *expected* party rather than from some other holder of an otherwise-valid credential.

Because each event commits to the one before it, changing *any* event changes its `fingerprint`, which no longer matches the `prev_event.fingerprint` of the event that follows — and every later link breaks in turn. Deletion and reordering are detectable for the same reason: the chain of fingerprints no longer lines up.

```
Event N-1              Event N                          Event N+1
fingerprint: 7c1b92…   prev_event.fingerprint: 7c1b92…  prev_event.fingerprint: 46775c…
                       fingerprint:            46775c…  fingerprint:            e5e15e…

   N's prev_event == (N-1)'s fingerprint      N+1's prev_event == N's fingerprint
```

What makes this hold up is *where* the link sits. The canonical serialization covers the entire event including the attestation's own `authority_uid`, `chain_uid`, and `prev_event`, and excludes only the `fingerprint` and `signatures` themselves — an output cannot be part of its own computation. Because `prev_event` is inside the signed and fingerprinted content, an attacker cannot quietly delete an event and re-link its neighbours: doing so changes the following event's canonical bytes, breaking both its fingerprint and any signature over it.

## Where it lives in OCSF: an optional profile on the base event

Record integrity is not a new class and not an AI-specific feature. It is a domain-agnostic **profile** applied at `base_event`, so that *any* event class can opt in to carrying an attestation, and none is forced to.

```
base_event  +  record_integrity profile  =  any event can attest
(every class      optional; adds the           API Activity, Findings,
 inherits it)     attestation_list attribute   IAM, File System Activity…
```

A class carries an attestation only when its producer chooses to populate one — the profile is opt-in and `attestation_list` is optional. The attribute is an array, which is what lets independent attesters cover the same event: an endpoint agent at write time, an ingest gateway, and a downstream analytics platform each contribute a *separate entry*. That is a different axis from multiple signers of a *single* attestation — a producer plus a notary or witness over the same bytes — who are carried in that attestation's own `signatures` array.

The same object works equally well for endpoint telemetry, a detection finding, a configuration file, or an AI agent's activity. Agent activity (the `ai_operation` profile) is simply the *first consumer* that motivated the work; when attesting agent events, a producer applies both profiles together, because the attestation itself is not AI-specific.

This placement mirrors how OCSF profiles are meant to work: a focused, reusable set of attributes that cuts across categories and classes, rather than a proliferation of new classes.

## What the object carries: two jobs, "who" and "unaltered"

The `attestation` object does two independent jobs, and it is worth being precise about which mechanism provides which property, because the object is valid with either one alone.

| Field(s) | Job | Property provided | Keys required? |
|---|---|---|---|
| `signatures[]`, `authority_uid` | **Who** | Producer **authenticity** and **non-repudiation** | Yes |
| `fingerprint`, `prev_event`, `chain_uid` | **Unaltered** | **Integrity** and tamper-evident **ordering** | No |

- **`signatures`** is one or more `digital_signature` objects, each computed over this event's `fingerprint` and thereby over the event's canonical serialization. The first is typically the producer; additional entries carry a co-signature, notary, or witness over the same event. Signatures are what bind an event to a verifiable identity — the *authenticity* and *non-repudiation* half.
- **The hash-chain fields** provide *integrity* and *ordering* with no key management at all. A hash chain is tamper-*evident*: given an independent reference point — a signature anchoring the chain, or a verifier who already holds a trusted earlier fingerprint — any in-place edit, deletion, or reordering becomes detectable.

Because the two capabilities are independent, the object requires at least one of them rather than both:

```json
"constraints": {
  "at_least_one": ["fingerprint", "signatures"]
}
```

An attestation must therefore carry a fingerprint, a signature, or both. A pure hash chain gives you tamper-evidence without any PKI — enough to detect accidental alteration or corruption; adding signatures gives you attribution on top. Signed *and* chained is the strongest combination — a verifier holding nothing but the exported JSON can confirm both who produced each event and that its position in the chain was not rewritten, because the signature's canonical input includes the chain coordinates.

A minimal signed, chained attestation on an event looks like this:

```json
"attestation_list": [
  {
    "uid": "att-9f2c…",
    "authority_uid": "gateway-prod-01",
    "chain_uid": "session-7c1b92e4",
    "prev_event": {
      "uid": "3f0a1c7e-…",
      "type_uid": 600302,
      "fingerprint": {
        "algorithm_id": 3,
        "encoding_id": 1,
        "value": "7c1b92…"
      }
    },
    "fingerprint": {
      "algorithm_id": 3,
      "encoding_id": 1,
      "serialization_id": 2,
      "value": "46775c…"
    },
    "signatures": [
      {
        "algorithm_id": 1,
        "serialization_id": 1,
        "created_time": 1782000000000,
        "certificate": { "…": "per the OCSF digital_signature object" }
      }
    ]
  }
]
```

Two `serialization_id` values appear here and they are answering different questions. On the `fingerprint`, it declares how the structured event was turned into a deterministic byte sequence before hashing — `JCS` (2) for RFC 8785 canonical JSON, or `Flat` (1) where the fingerprinted data is an opaque byte sequence such as file content to which no canonicalization applies. On the `digital_signature`, it declares the signing-envelope rule — `Flat` (1) for a raw signature over the fingerprint bytes, or `JWS`, `COSE`, `DSSE` for enveloped forms. A verifier needs both to reproduce the computation: one tells it which bytes were hashed, the other tells it what was signed.

### A note on identity and framing

The verifiable identity in an attestation is a **technical credential** — a key or a certificate. It is deliberately *not* an attribution of any person, organization, or state. Where an event also carries a `user` attribute, that is the technical entity and remains optional. This keeps the construct neutral and avoids overreaching into claims about real-world actors — it proves that the holder of a particular key produced a particular record, and nothing more.

## The use cases the community already named

The construct is generic, and the demand for it showed up across domains before it was AI-specific:

- **Telemetry provenance.** *"How do I know this telemetry really came from that router or host?"* A signature binds the record to the source's verifiable identity.
- **Detection-finding origin.** *"Prove this finding really came from that security product, unaltered."* The producer signs the finding; consumers verify without trusting the transport.
- **Endpoint-to-cloud relay.** An event is produced on an endpoint, then relayed to the cloud. The endpoint signs it; the cloud co-signs that it received *this exact record*, unaltered — a verifiable hand-off, i.e. a chain of custody.
- **Agent audit and regulated data.** An AI agent deletes a HIPAA-governed record. A tamper-evident chain of signed records establishes the full chain of custody as court-defensible evidence.

## It works — and you can check it yourself

The point of "checkable by anyone, offline" is that verification requires no special infrastructure — no access to the producer, no trust in the pipeline, just the exported records and a public key. A standalone verifier, using only a language's standard library, recomputes each record's canonical bytes, checks the fingerprint, and follows the chain.

Given an intact three-record chain, verification passes:

```
Entry #1  read_ticket   ✓
Entry #2  lookup_order   ✓
Entry #3  issue_refund   ✓

CHAIN INTACT ✓  3/3 verified
```

Now alter a single field in record #3 after the fact — flip a denied action to look successful — and re-run the same verifier:

```
Entry #3  issue_refund
  CHAIN BROKEN ✗
  claimed:  8a1cc9a1…
  computed: e5e15e60…
  ALTERED after written.
```

The edit is not prevented — nothing stops someone with write access from changing stored bytes. What changes is that the edit is now *evident*, and the original record remains *attributable*. This same shape runs in production today against live OCSF event chains, and the verifier reproduces the result with nothing but a standard library.

The following is a sanitized window from a production agent-gateway audit log. Control-plane actions (an agent created, a key rotated) and the agent's own tool calls land in one chained trail, each bound to an actor and agent. Note row 214: a `read_file` tool call is **denied** by policy (`no_active_policy`, 403) — the denial itself becomes a chained, verifiable record, not just a transient error. Each row's previous-event reference carries the prior row's fingerprint; alter or delete any row and every later link breaks.

![Sanitized production agent-gateway audit log: agent lifecycle and tool calls in one tamper-evident chain, including a policy deny at row 214](images/record-integrity-production-log.png)

## What's next: proving one of a million in O(log N)

A hash chain is tamper-evident, but proving that a *single* event is committed means walking the chain — O(N) — with one signature per record. A signed **Merkle checkpoint** over a batch of records addresses both costs, and it is purely additive: nothing in the base construct changes.

| | Chain attestation | + checkpoint layer |
|---|---|---|
| Prove one event | walk the chain — O(N) | inclusion proof — O(log N) (~20 hashes for a million) |
| Signatures | one per event | one covers the whole batch |
| Verifier needs | the chain | one signed root + the public key |

This follows the RFC 6962 transparency-log model: the Merkle leaves *are* the existing per-event `fingerprint` values, a signed checkpoint carries the `merkle_root` and `tree_size`, and an inclusion proof carries the `audit_path`. It is a natural follow-on rather than part of the core construct, and a standalone offline verifier for it already runs.

## Summary

The `record_integrity` profile gives OCSF a domain-agnostic, opt-in way to make records tamper-evident and attributable:

- A **hash chain** (`fingerprint`, `prev_event`, `chain_uid`) provides integrity and ordering with no keys.
- **Signatures**, together with `authority_uid`, provide producer authenticity and non-repudiation.
- An `at_least_one` constraint means an attestation must carry at least one of these; together they are strongest.
- The chain link lives *inside* the fingerprinted and signed content, so deleting or re-linking an event is detectable rather than silent.
- It applies at `base_event` as `attestation_list`, so any class can adopt it, and independent attesters each get their own entry; `ai_operation` is simply the first consumer.
- Verification is standalone and offline — anyone holding the events and a public key can check them.

OCSF has always described what happened. Record integrity lets the record also prove that it is *intact* and *authentic* — the property that matters most exactly when a record becomes evidence.

---

*Reference: the `attestation` object and `record_integrity` profile — [ocsf/ocsf-schema #1661](https://github.com/ocsf/ocsf-schema/pull/1661), merged 2026-07-17 (`2a244bc9`), shipping in OCSF 1.9.*
