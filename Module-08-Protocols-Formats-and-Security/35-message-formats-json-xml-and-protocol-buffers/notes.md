# Study Notes: Message Formats — JSON, XML & Protocol Buffers

## Definitions

- **Serialization:** Converting in-memory data into a byte sequence for transmission or storage.
- **Deserialization:** Converting a byte sequence back into usable in-memory data.
- **Schema:** A formal definition of a message's structure — field names, types, and (for protobuf) numeric tags — that both sender and receiver agree to.
- **Schema evolution:** The rules/practices that let a message format change over time (adding/removing fields) without breaking compatibility between old and new versions of communicating services.

## Format Comparison

| Aspect | JSON | XML | Protocol Buffers |
|---|---|---|---|
| Format type | Text | Text | Binary |
| Human-readable | Yes | Yes | No (needs schema + tooling) |
| Schema | Optional (JSON Schema is add-on) | Native, strong (XSD) | Required (`.proto` file), enforced |
| Payload size | Medium (field names repeated) | Large (verbose tags) | Small (field numbers, not names, on wire) |
| Parse/serialize speed | Moderate | Slower (tag parsing overhead) | Fast (binary, no tokenizing) |
| Type safety | Weak (no int/float distinction, no enforced types) | Moderate (via XSD) | Strong (generated typed code) |
| Binary data support | Poor (needs base64, +~33% size) | Poor (same issue) | Native |
| Typical use | Public REST APIs, config, logs | Legacy/enterprise integration, SOAP | Internal service-to-service (gRPC), high-throughput systems |

## Schema Evolution Rules (Protocol Buffers)

- Each field has a permanent numeric tag — this, not the field name, is what's actually encoded on the wire.
- **Safe:** adding a new field with a new tag number (old code ignores unrecognized tags).
- **Unsafe:** reusing/renumbering an existing field's tag (old and new code would then disagree on what a given tag means).
- **Recommended:** when removing a field, mark its tag `reserved` so it's never accidentally reassigned to a different field later.
- JSON/XML handle this more loosely — an old consumer typically just ignores unknown fields and treats missing fields as null/absent, with no enforced contract either way.

## Decision Guide

| If your priority is... | Reach for... |
|---|---|
| Human debuggability / broad tooling / public API | JSON |
| Strong native schema validation, enterprise/legacy interop | XML |
| Minimum bandwidth, fastest (de)serialization, internal RPC | Protocol Buffers (usually via gRPC) |
| Streaming or very high call volume between your own services | Protocol Buffers / gRPC |

## Key Numbers / Facts

- Base64-encoding binary data (needed to embed it in JSON/XML text) adds roughly 33% to its size.
- Protobuf payloads are commonly 3-10x smaller than the equivalent JSON for the same data, depending on field types and repetition.
- Protocol Buffers were developed internally at Google and open-sourced in 2008.

## Summary

- Every message format is a trade-off between human-readability/tooling (JSON, XML) and wire efficiency/type-safety (Protocol Buffers).
- JSON is the pragmatic default for public and browser-facing APIs; XML persists mainly in legacy/enterprise integration; Protocol Buffers dominate high-throughput internal service communication, especially paired with gRPC.
- Schema evolution — how you add/remove fields without breaking existing consumers — is a first-class design concern once services are deployed and versioned independently.
