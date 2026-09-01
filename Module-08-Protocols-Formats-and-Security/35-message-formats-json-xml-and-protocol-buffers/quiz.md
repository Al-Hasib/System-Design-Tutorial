# Practice & Interview Questions

**1. What is serialization, and why does every distributed system need a shared, agreed-upon format for it?**
Serialization converts in-memory data into a byte sequence for transmission or storage. Two services can only understand each other's bytes if they both agree in advance on exactly how those bytes are structured — that agreement is the message format. Without it, the receiver would have no reliable way to reconstruct the sender's data.

**2. Why is JSON more verbose than Protocol Buffers for the same logical data?**
JSON repeats every field name as a text string in every single message (`"userId"`, `"userName"`, etc.), while Protocol Buffers replace field names with small numeric tags defined once in a shared schema — the field names never appear on the wire at all, only in the `.proto` definition both sides already have.

**3. What does it mean for Protocol Buffers to be "schema-first," and what does that buy you compared to JSON?**
The message structure (field names, types, numeric tags) is defined up front in a `.proto` file, and both sender and receiver generate strongly-typed code from that same schema. This catches type mismatches at compile time and guarantees both sides agree on structure, whereas JSON's fields are loosely typed and unenforced unless you bolt on a separate validation layer like JSON Schema.

**4. Why does XML tend to appear in enterprise/legacy systems today rather than in new API designs?**
XML offers strong native schema and validation support (XSD) and namespaces, which mattered for large enterprise integrations with strict contracts, but its tag-based syntax is more verbose than JSON for the common case. Most new systems now prefer JSON's simplicity and smaller payloads, leaving XML mostly in legacy systems and specific enterprise/SOAP integrations you might still need to interoperate with.

**5. In Protocol Buffers, what makes adding a new field to a message "safe," and what makes changing an existing field's number "unsafe"?**
Adding a new field with a new, unused tag number is safe because old code simply ignores tags it doesn't recognize. Reusing or renumbering an existing field's tag is unsafe because it makes old and new code disagree about what that tag means — one side might read completely wrong data for that field.

**6. Why can't you embed raw binary data (like an image) directly inside JSON or XML?**
Both are text formats, so binary bytes must be encoded into a text-safe representation first — typically base64 — which adds roughly 33% to the data's size. Protocol Buffers, being a binary format natively, can include raw bytes directly without this encoding overhead.

**7. Scenario: You're designing a public API that third-party developers will integrate with, many of whom will debug it by hand with curl or Postman. Which message format would you choose, and why?**
JSON — it's human-readable, universally supported across languages and tools, and lets developers inspect and debug responses directly without needing your schema definitions or generated code, which matters a lot for a public integration surface.

**8. Scenario: Two services you own make a few million calls to each other per day, and you've noticed serialization overhead showing up in profiling. What would you consider changing, and why?**
Moving from JSON/REST to Protocol Buffers over gRPC — since both services are internal and under your control, you can share the `.proto` schema and generated code, and the smaller binary payloads plus faster (de)serialization would reduce both bandwidth and CPU time at that call volume.

**9. What is schema evolution, and why does it matter for services that are deployed independently?**
Schema evolution is the set of rules/practices for changing a message format over time (adding, removing, or changing fields) without breaking compatibility between services running old and new versions simultaneously. It matters because independently-deployed services can't always be upgraded in lockstep — a producer and consumer may run different schema versions for some period during a rollout, and the format needs to tolerate that.

**10. True or False: JSON enforces that a given field always has the same data type across all messages.**
False. JSON itself has no schema enforcement — a field could technically hold a string in one message and a number in another unless the application layer (or an added-on tool like JSON Schema) validates it. Protocol Buffers, by contrast, enforce field types at the schema and generated-code level.
