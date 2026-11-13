<!-- Trusted provenance: F-13b791aee2fb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-13b791aee2fb

## Purpose

F-13b791aee2fb handles a message associated with one of two records. Depending on the message selector, it initializes a control mask, changes record data, invokes an endpoint operation, submits a request, or completes processing for the record.

The operation is **not transactional**. Record changes, dependency calls, and comparison-word updates can occur before a return indicating rejection or an unsuccessful request. No rollback is established.

**Established static evidence:** the supplied representation supports the layouts, branch conditions, explicit dependency arguments, retained addresses, and observable accesses described below.

**Inferred intent:** the mask and progression markers appear to coordinate a field exchange. This interpretation does not establish a protocol specification. In particular, accepting a field does not visibly add its marker to the mask.

**Unknown:** dependency internals, hidden arguments, storage ownership, and behavior outside the represented arithmetic’s defined range. No runtime tests have been performed.

## Interface

### Inputs

The visible interface has four argument positions:

| Position | Contract |
|---|---|
| First | Opaque 32-bit context, retained for completion operations |
| Second | Descriptor address |
| Third | 32-bit value with no visible use |
| Fourth | 32-bit incoming value whose initial contents do not visibly affect a decision; the argument storage is reused |

The descriptor contains these accessed fields:

| Byte offset | Width | Meaning within this function |
|---|---:|---|
| 0 | 32 bits | Endpoint value retained for later calls |
| 8 | 32-bit address | Message address |
| 12 | 16 bits | Input to F-a2715f79cdcc |

F-5f5f14cedda3 derives a 32-bit lookup key from the descriptor. F-a2715f79cdcc receives that key and the descriptor’s 16-bit field. Its represented 64-bit result supplies the record index in its low 32 bits and the declared message size in its high 32 bits.

Only record indices 0 and 1 are accepted. Size comparisons use unsigned 32-bit values.

### Shared record layout

Separate shared pointer slots identify the active-status bytes, control-mask words, comparison words, handles, and record bodies. Control masks, comparison entries, and handles occupy 32 bits each.

Record bodies have a 292-byte stride:

| Record-relative offset | Accessed data |
|---|---|
| 0 | Sequence destination for selector 1 |
| 64–66 | Three bytes for selector 4 |
| 67–69 | Three bytes for selector 5 |
| 70–72 | Three bytes for selector 2 |
| 74–99 | Thirteen 16-bit entries |
| 100 | Sequence destination for selector 6 |
| 164 | Sequence destination for selector 7 |
| 228 | Sequence destination for selector 8 |

The spacing of sequence destinations does not establish dependency write limits or allocation sizes.

### Outputs and preconditions

The represented normal return paths produce:

| Result | Meaning |
|---:|---|
| 9 | Rejected input or record state, absent progression, or unsupported progression |
| 28 | Request-result accumulator is zero; this also covers paths that issue no request |
| 0 | Nonzero request result processed, or completion callback reached |

These values do not guarantee termination for unresolved arithmetic cases.

Every executed descriptor, message, shared-storage, object-table, and temporary-buffer access requires valid storage. The function supplies no explicit null, alignment, or allocation-bound validation. A sequence scan can read one byte beyond the declared message extent; that access requires readable storage even though a terminator at that position is rejected.

## Behavior

### Entry and address retention

The lookup-key dependency runs before the endpoint and message address are read. The descriptor’s 16-bit field is then obtained for F-a2715f79cdcc. After that call, the function captures the record-body base followed by the control-mask base.

It rejects an index greater than 1 before accessing the selected active-status byte. A nonzero active byte rejects the record. A zero declared size is rejected next, before the selector byte is read.

Capturing a base retains an address, not the contents of its storage. Later dependency calls may change contents at retained addresses. Later reads of shared pointer slots may also identify different storage.

### Field acceptance and immediate effects

Each recognized selector starts with no progression. Unless stated otherwise, failing a selector’s conditions leaves progression absent and ultimately returns 9.

| Selector | Required size | Additional condition | Immediate effect | Progression |
|---:|---:|---|---|---:|
| 0 | 5 | Existing mask zero; decoded value has bit 0 set | Replace mask | 1 |
| 1 | 2–65 | Accepted terminator position | Sequence dependency at offset 0 | Derived after dependency |
| 2 | 8 | Decoded identifier matches object-provided value | Update offsets 70–72 | 4 |
| 3 | 4 | Entry index below 13 | Update one 16-bit entry | 8 |
| 4 | 4 | None | Update offsets 64–66 | 16 |
| 5 | 4 | None | Update offsets 67–69 | 32 |
| 6 | 2–65 | Accepted terminator position | Sequence dependency at offset 100 | Derived after dependency |
| 7 | 2–65 | Accepted terminator position | Sequence dependency at offset 164 | Derived after dependency |
| 8 | 2–65 | Accepted terminator position | Sequence dependency at offset 228 | Derived after dependency |
| 9 | 3 | Decoded unsigned value exceeds 127 | Endpoint operation | 512 |

Unrecognized selectors return 9 immediately.

**Selector 0.** The existing mask is checked through a fresh shared mask-base read. A nonzero word returns 9 without decoding. Otherwise, F-c25a2fcc75b5 receives message offset 1. If its 32-bit result has bit 0 clear, the function writes zero through the entry-captured mask base and returns 9. If bit 0 is set, it writes the complete decoded word there and establishes progression 1; higher bits are not validated.

Both target addresses are fixed by bases obtained before the decoder. A decoder-induced pointer-slot change redirects neither access. The checked and written words can differ if the shared pointer changed between the two base captures, although no intervening dependency call or direct pointer-slot write is visible. The decoder may change the checked contents; no second check follows it.

**Selector 2.** F-c25a2fcc75b5 decodes from message offset 1. F-cdd33263ee89 supplies an object, and D-04 supplies the value compared with that decoded 32-bit value. On equality, message bytes 5, 6, and 7 are read and stored at offsets 70, 71, and 72 respectively. Each read/store pair precedes the next. These writes use the entry-captured body base despite the intervening calls.

**Selector 3.** Message byte 1 identifies the entry. Its validation read and destination-selection read are separate in the representation; stable contents must not be assumed. A fresh shared body-base read supplies the destination. The stored 16-bit numeric value is 256 times message byte 2 plus message byte 3, covering 0–65535. This numeric relationship does not establish host byte order or a total ordering of the expression’s source reads.

**Selectors 4 and 5.** A fresh body-base read supplies the destination. Message bytes 1–3 are transferred to the respective three-byte region in increasing order, with each source read followed by its destination store. Aliasing can therefore affect subsequent reads.

**Selector 9.** F-64e2dfdd4a5d receives message offset 1. Its result is treated as unsigned 32-bit data. Only a value greater than 127 invokes F-382ffcbbd77a with the retained endpoint and decoded value. Progression 512 is established after that call.

### Sequence fields

The accepted size interval is 2–65 inclusive, established by an unsigned subtraction-based check.

Scanning starts at message offset 1. A zero byte is accepted only at an offset strictly less than the declared size. Without an earlier terminator, the scan also reads the byte at offset equal to the size. A zero there is too late; a nonzero byte causes one further index increment before scanning stops. The scan index is reduced to eight bits after increments, but does not wrap within this size interval.

An accepted sequence invokes D-03 with the selected destination address and message offset 1. There is no explicit length argument. The destination comes from a fresh body-base read and the **original** selector. Empty sequences are accepted when byte 1 is zero.

After D-03 returns, the selector byte is reread to derive progression. Consequently, dependency mutation can make the destination correspond to one selector and progression to another.

The represented shift has a signed literal one as its left operand; conversion to unsigned progression occurs afterward. Counts 0–30 establish the corresponding powers of two. Unchanged selectors 1, 6, 7, and 8 therefore establish 2, 64, 128, and 256.

Counts 31–255 are **documented_only**. In particular, ordinary signed-32-bit rules do not define the count-31 sign-bit shift. The unsigned destination does not repair that uncertainty.

### Progression and selection

After field handling, the comparison-array base is captured. The mask is then read through the entry-captured mask base. A zero mask or absent progression returns 9 without undoing preceding effects.

Progression 8 has special selection behavior. D-01 supplies a context, and F-62d1a359629f receives that context address plus 4 and the retained lookup key. Its low 32 bits are compared with the selected comparison word through the captured comparison base. Inequality selects marker 8 directly, whether or not mask bit 3 is set. Equality uses ordinary forward selection.

Ordinary selection considers successive doubled markers using unsigned 32-bit arithmetic. Below 1024, it rereads the mask through the retained base and chooses the first candidate present in that mask. Candidate 1024 enters completion without checking mask bit 10. A candidate above 1024 returns 9. Thus a field’s own marker is ordinarily skipped rather than selected.

For normal field markers, doubling does not overflow. Sequence-derived counts 10–30 produce an initial candidate greater than 1024 and return 9.

A separate conditional consequence remains relevant: **if** an independently established execution produces progression 0x80000000 and reaches ordinary selection, unsigned doubling produces zero. Zero remains zero and cannot intersect any mask, leaving no visible loop exit. This does not establish that the unresolved count-31 shift produces that progression.

### Request submission

Selected request markers have these payloads:

| Marker | Payload |
|---:|---|
| 2 | One-byte tag 1 |
| 4 | Eight-byte special payload |
| 8 | Two-byte comparison payload, if a qualifying difference exists |
| 16, 32, 64, 128, 256, 512 | One-byte tags 4, 5, 6, 7, 8, 9 respectively |

F-cd7e1905512f receives the retained endpoint, two zero arguments, operation value 39, a temporary payload address, and payload size. Its result becomes the request-result accumulator, initially zero.

For marker 4, the temporary payload region comprises adjacent 32-bit words initialized to 2 and zero. F-cdd33263ee89 supplies an object; D-04 is called with that object explicitly. Its result is passed to F-b9d9c0b9bfab with a destination at payload offset 1. D-05 subsequently receives the same retained object and three four-byte output buffers. Their first bytes populate payload offsets 5, 6, and 7 in argument order. Eight bytes are submitted. A four-byte serialization at offsets 1–4 fits this layout, but the serializer’s actual extent and encoding remain unknown.

For marker 8, the template word is captured before D-01 and another F-62d1a359629f call. This is distinct from any earlier selection comparison. Bits 0–12 are compared in ascending order against a comparison word reread for each bit. The external result remains retained.

The first difference produces a two-byte payload containing the template’s low byte and the differing bit index. After submission, the stored comparison word is reread, incremented by that bit’s power of two, and written back. This is 32-bit modular addition, not bitwise setting; an already-set bit can carry. The update occurs even when submission returns zero. Only one difference is handled. If bits 0–12 agree, no request is submitted.

### Request-result processing

A zero accumulator returns 28.

A nonzero accumulator is passed to F-256e8a77aa59 with zero as its second argument. D-02 then supplies a service context. Before the first handle operation, the function retains a handle-array base for later use and obtains the first handle through another current shared-base read. F-23aadb16b673 receives that context and handle.

D-02 supplies a second context before the second handle value is read through the retained base. F-8becc2a0337c receives that context and handle. The function then returns 0.

A pointer-slot change during the first handle operation cannot redirect the retained second base; mutations to its contents remain observable.

### Completion

Completion obtains a service context from D-02, reads the current handle base and selected handle, and invokes F-23aadb16b673.

A fresh mask-base read determines whether bit 3 enables indexed completion. If enabled, indices 0–12 are visited in order. Each index obtains a new D-01 context and F-62d1a359629f result. When that result’s low 32 bits contain the index’s bit, the corresponding 16-bit record entry is loaded through the entry-captured body base, and F-d4ed7d447da8 is called with the original first argument and index as its explicit arguments.

A zero low-32-bit result from F-d4ed7d447da8 marks completion unsuccessful and stops this iteration. Delivery of the loaded entry is not explicit; no hidden argument is asserted.

An independent fresh mask-base read then checks bit 2, even after indexed failure. If enabled, record bytes 70–72 are read through the entry-captured body base, stopping at the first value 255. If all three differ from 255, F-4ab31ce53300 receives the original first argument.

The active-status byte is written as 1 only if indexed completion remained successful. The conditional operation’s return does not independently change that decision. F-74710b3c8aff finally receives the original first argument and lookup key. Completion returns 0 even after indexed failure.

## Side effects

Explicit writes affect control masks, record bytes, 16-bit entries, comparison words, and active-status bytes. Dependency effects are additional and unspecified.

No allocation or release is explicit. Passing descriptors, objects, handles, request results, or temporary addresses establishes neither ownership transfer nor release responsibility. Request payloads are invocation-local; whether submission copies or retains them is unknown.

No disjointness, synchronization, global-preservation, or reentrancy guarantee is established. Aliasing and dependency mutation can affect subsequent reads, including reads through retained addresses.

## Dependencies

All identifiers below are opaque. D-01 through D-05 name otherwise unnamed callable dependencies within this contract.

| Identifier | Visible role |
|---|---|
| F-5f5f14cedda3 | Derive descriptor lookup key |
| F-a2715f79cdcc | Supply record index and declared size |
| F-c25a2fcc75b5 | Decode a 32-bit field value |
| F-64e2dfdd4a5d | Decode the threshold-tested value |
| F-cdd33263ee89 | Supply an object |
| F-b9d9c0b9bfab | Write a special-request representation |
| F-62d1a359629f | Supply comparison data |
| F-cd7e1905512f | Submit a request and return its result |
| F-256e8a77aa59 | Process a nonzero request result |
| F-23aadb16b673 | First handle operation |
| F-8becc2a0337c | Second handle operation |
| F-382ffcbbd77a | Threshold-triggered endpoint operation |
| F-d4ed7d447da8 | Indexed completion operation |
| F-4ab31ce53300 | Conditional completion operation |
| F-74710b3c8aff | Final completion callback |
| D-01 | Comparison-context provider |
| D-02 | Service-context provider |
| D-03 | Sequence operation |
| D-04 | Object-dispatched operation at table byte offset 48 |
| D-05 | Object-dispatched operation at table byte offset 64 |

D-04 has an explicit object argument during request construction but no explicit argument in the selector-2 comparison. That difference remains unresolved rather than normalized into an assumed calling convention.

## Acceptance examples

These are conditional static criteria, not executed tests. Unless mutation is specified, they assume stable relevant inputs and valid storage.

- **Unavailable record:** A valid index with a nonzero active byte returns 9 before reading the selector.
- **Rejected after writing:** A valid selector-4 message with a zero post-field mask updates offsets 64–66 before returning 9.
- **Mask decoding:** A zero existing mask and decoded value 2 cause a zero mask write and return 9, because bit 0 is clear.
- **Decoder mutation:** Changing the shared mask pointer inside the selector-0 decoder does not redirect its subsequent write. Changing the checked word’s contents does not cause revalidation.
- **Forward completion:** Selector 0 decoding mask 1 establishes progression 1. With no intervening mutation, no later request marker is enabled, so selection reaches completion without requiring bit 10.
- **Sequence destination versus progression:** Selector 6, size 2, and byte 1 zero invoke D-03 at record offset 100. If D-03 changes byte 0 to 9, progression becomes 512; a nonzero post-field mask permits completion.
- **Late terminator:** A first zero at offset equal to declared size is read but does not authorize D-03.
- **Unresolved shift:** Mutation of the selector to 31 has no unconditional marker or return criterion. The zero-doubling loop applies only if progression 0x80000000 is independently established.
- **Comparison carry:** If bit 2 is the first differing bit and the reread stored word is 4, the post-request addition stores 8, not 4. A zero request result still permits that update before return 28.
- **High-bit-only mismatch:** A full-word comparison can select marker 8, yet differences confined above bit 12 produce no request and return 28.
- **Indexed failure:** A zero indexed-completion result prevents active marking but does not suppress the independent bit-2 processing or final callback. Return remains 0.

## Uncertainties

The following remain **documented_only**:

- Sequence-induced shift counts 31–255 and their machine-level outcomes.
- Decoder encodings, serializer extent, and narrower dependency result guarantees.
- Hidden arguments, secondary outputs, and delivery of loaded completion entries.
- D-03’s copying, bounds, overlap, and termination semantics.
- Object validity, allocation bounds, alignment, ownership, and payload retention.
- Dependency mutation, aliasing, concurrency, and reentrancy effects.
- Application-level meanings of selectors, masks, handles, and status values.

The apparent string-like sequence operation and exchange-stage interpretation are inferred intent, not dependency contracts.

## Evidence

This contract is based on the supplied static representation. Earlier prose and reviewer feedback are not independent evidence. No runtime tests, machine-level verification, or dependency inspections have been performed.

The representation establishes explicit widths, field conditions, dependency arguments, separate pointer captures, rereads after calls, and normal return paths. It does not establish a total memory-access order for operands within an expression.

The signed-left-operand shift limits unconditional progression claims to counts 0–30. The later unsigned doubling supports the conditional zero-loop consequence without resolving that earlier shift. Retained-address and fresh-base distinctions are preserved throughout without assuming that any dependency preserves shared state.
