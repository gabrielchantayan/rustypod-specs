<!-- Trusted provenance: F-7ebdba6d3d94; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7ebdba6d3d94

## Purpose

F-7ebdba6d3d94 conditionally applies sequential records from an external resource to a linked collection reachable from its input.

Application is positional: one successfully read record updates the current node, and traversal then follows that node’s successor. No record identifier lookup is visible. Record processing requires successful acquisition, a successful header-read status, and two matching 32-bit header fields.

A referenced static string has a label suggesting playback-count metadata. This suggests the resource’s intended role but does not establish the string’s actual contents, encoding, punctuation, separators, or spaces. Neither the resource format’s application-level meaning nor the meanings of most updated fields are established.

A zero return is not proof that any node changed. Header rejection and an empty collection also return zero.

## Interface

### Input and object layout

The function accepts one value used as a base address in the supplied 32-bit address model. No explicit null, bounds, alignment, or object-type checks are visible.

After acquisition succeeds, the function resolves the collection metadata through these accesses:

1. Read a 32-bit pointer at input offset `0xF64`.
2. Read a second 32-bit pointer from the address obtained in step 1.
3. Read a third 32-bit pointer at offset `0x0C` from the object obtained in step 2.
4. From the metadata object obtained in step 3, capture:
   - A 32-bit expected header value at offset `0x24`.
   - A 32-bit first-node pointer at offset `0x28`.

The expected header value is captured before the first-node pointer. Both are captured before the header-buffer preparation call and the header-read call. Later modifications to those metadata fields do not replace the captured values used by this invocation.

Each visited node must provide accessible storage through offset `0xAB`, including a 32-bit successor pointer at offset `0x04`. These are addressability requirements for the visible accesses, not evidence of a complete object definition.

### Return value

The result is a 32-bit integer status.

| Condition | Result |
|---|---|
| Acquisition returns nonzero | Acquisition status |
| Header-read result has a nonzero low 32-bit word | That low word |
| Header read reports zero, but either header comparison fails | Zero |
| Header is accepted and the captured first-node pointer is null | Zero |
| Record reads succeed until traversal reaches null | Zero |
| A record read reports nonzero | The 32-bit record-read status consumed by this function |

The header-read result is represented as 64 bits, but only its low 32 bits control success and the returned status. No meaning for its upper half is established.

The release dependency’s result is ignored.

### Normal-execution preconditions

All accessed storage must remain valid when accessed. The function provides no visible synchronization, lifetime protection, pointer validation, or fault recovery.

A cyclic node chain is not detected. If record reads continue succeeding, a cycle can cause repeated updates and prevent normal completion. A read failure can still terminate such traversal.

## Behavior

### Preparation and acquisition

The function uses automatic storage for:

- A 48-byte resource-state region.
- A 260-byte descriptor region.
- A 96-byte header scratch region.
- A 32-bit transfer-length cell.

Record storage overlaps the header scratch region as described below.

An unidentified two-argument preparation dependency receives the resource-state address and size 48, then the descriptor address and size 260. Its memory effects are unknown; zero initialization must not be assumed.

The function then explicitly writes zero to the descriptor’s first two bytes. F-e7c5183babcf receives:

- A destination beginning two bytes into the descriptor.
- A reference to a static string whose label suggests playback-count metadata.

The destination has 258 bytes remaining within the descriptor. No capacity argument is visible at this call. The string’s actual bytes, encoding, termination, and the dependency’s bounds behavior are `documented_only`.

After these preparation calls, the function reads a value from static storage for the fourth acquisition argument. This is a call-time read, not a value captured before preparation and not an established immutable constant. Earlier dependencies may have modified that storage.

F-2b2edb028a06 receives, in order:

1. The descriptor address.
2. The resource-state address.
3. A zero-valued argument.
4. The value just read from static storage.

The fourth argument’s numeric value and meaning are unknown. Its declared storage type and width are not independently supplied.

A nonzero acquisition status is returned immediately. That path performs no explicit input-graph reads, header reads, node updates, or release call.

### Metadata capture and header transfer

After acquisition returns zero, the function resolves the metadata object and captures the expected header value and initial node pointer.

It next passes the 96-byte header region to the preparation dependency. It writes 96 (`0x60`) to the transfer-length cell and calls F-74745fd09174 with:

1. The resource-state address.
2. The transfer-length-cell address.
3. The header-region address.

Only the low 32 bits of the returned result determine read success. If that word is nonzero, no header field is accepted and record processing is skipped.

If the low word is zero, the function reads a 32-bit marker from static storage for comparison with the header word at offset `0x00`. This marker read occurs **after the header-read dependency returns successfully** and before the marker comparison. It is not an established constant or a value captured before the read dependency. That dependency, or an earlier dependency, may have changed the static storage.

Header acceptance requires:

| Header offset | Width | Requirement |
|---|---:|---|
| `0x00` | 32 bits | Equal to the marker read from static storage after successful header reading |
| `0x0C` | 32 bits | Equal to the expected value captured earlier from metadata offset `0x24` |

The second comparison is reached only if the first comparison succeeds. The static marker is not read on the header-read failure path.

The source establishes these read points relative to dependency calls. It does not establish a separate ordering between the two operand reads used in the marker comparison.

No other header bytes are explicitly validated. The function does not inspect the transfer-length cell after the call. A zero status therefore does not, by itself, establish that all 96 requested bytes were written.

Header rejection skips all record processing, calls release, and returns zero. The marker and metadata comparisons are not repeated between records.

### Record layout and scratch reuse

For every nonnull current node after header acceptance, the function requests 28 (`0x1C`) bytes into record scratch storage.

That storage is the final 28 bytes of the 96-byte header region, beginning at header offset `0x44`. It is not a separate allocation. There is no visible preparation call or clearing operation between record reads.

| Record offset | Width | Local use |
|---|---:|---|
| `0x00` | 32 bits | Added to node field `0x90` |
| `0x04` | 32 bits | Assigned to node field `0x98` |
| `0x08` | 32 bits | Assigned to node field `0x68` |
| `0x0C` | 32 bits | Narrowed to one byte and assigned at node offset `0x7E` |
| `0x10` | 8 bits | Nonzero condition for setting node bit `0x4000` at offset `0x1C` |
| `0x11` | 8 bits | Exact-one condition for node bit `0x08` at offset `0x8F` |
| `0x12`–`0x13` | 2 bytes | Not explicitly consumed |
| `0x14` | 32 bits | Added to node field `0xA0` |
| `0x18` | 32 bits | Assigned to node field `0xA8` |

No explicit byte-order conversion is visible. These are native-width accesses in the supplied representation; this function alone does not establish a portable external byte order.

### Successful-record update order

Before each record read, the function resets the transfer-length cell to 28. It calls F-74745fd09174 with the resource-state address, transfer-length-cell address, and record-region address.

A nonzero consumed status stops traversal before this function performs any field update for that record. Earlier successful updates are retained. This does not restrict writes performed by the dependency itself.

After a zero status, the explicit node operations occur in this order:

1. Read the 32-bit node field at `0x90`, add the record value at `0x00`, and write the result back.
2. Write the record’s 32-bit value at `0x08` to node offset `0x68`.
3. Write the record’s 32-bit value at `0x04` to node offset `0x98`.
4. Narrow the record’s 32-bit value at `0x0C` to one byte and write it at node offset `0x7E`.
5. Read the node byte at `0x8F`. Set its bit `0x08` if record byte `0x11` equals exactly one; otherwise clear that bit. Write the resulting byte back, preserving its other bits.
6. Read the 32-bit node field at `0xA0`, add the record value at `0x14`, and write the result back.
7. Write the record’s 32-bit value at `0x18` to node offset `0xA8`.
8. If record byte `0x10` is nonzero, read the 32-bit node field at `0x1C`, set bit `0x4000`, and write it back. If the byte is zero, this step performs no explicit read or write of that field.
9. Read the 32-bit successor pointer at node offset `0x04` and make it the current node.

The successor is read after the record-read dependency and all explicit updates, not before them. A dependency’s modification of the successor can therefore affect traversal.

The additions use 32-bit fields. There is no explicit overflow check, saturation, carry reporting, or overflow status. Ordinary addition is established when the mathematical sum fits signed 32-bit representation. Behavior beyond that range is `documented_only`; this contract does not impose portable wraparound or saturation.

The narrowing operation produces a one-byte store. Portable signed-conversion semantics for out-of-range source values are not established.

### Termination and release

Record processing ends when the current node is null or a record read reports nonzero.

The expected header value is used only for equality checking. It is not used as a node count or record limit. The function neither checks a processed-record count against it nor reads ahead to detect surplus records after reaching a null successor.

On every normal return path after successful acquisition, F-49a51558071a is called once with the resource-state address. This includes:

- Header-read failure.
- Header rejection.
- An accepted header with an empty collection.
- Traversal reaching a null successor.
- Record-read failure.

This normal-path release behavior does not establish cleanup after a fault or another abnormal interruption.

## Side effects

Explicit persistent effects are the ordered node-field writes above. Updates are incremental and nontransactional: later read failure does not roll back earlier nodes.

The two flag operations have different semantics:

- Bit `0x08` in node byte `0x8F` is set or cleared for every successful record.
- Bit `0x4000` in the 32-bit field at `0x1C` is conditionally set and is never explicitly cleared by this function.

The function also writes its automatic descriptor, scratch regions, and transfer-length cell, and passes their addresses to dependencies. No explicit dynamic allocation, node insertion, node removal, or node deallocation is visible.

Dependency effects are not bounded by the visible body. The contract does not assume that dependencies preserve globals, leave reachable nodes untouched, retain no pointers, or acquire or release any particular memory ownership.

In particular, the acquisition argument and header marker are read from static storage at different times. Their contents may be affected by intervening calls. Neither value is established as immutable.

The automatic buffers are available during this invocation. Passing their addresses does not establish a valid post-return lifetime for retained pointers or prove that callees do not retain them.

## Dependencies

| Opaque function ID | Observed interaction | Limits of established behavior |
|---|---|---|
| F-e7c5183babcf | Receives descriptor offset `0x02` and a referenced static string | String bytes, encoding, bounds, termination, ownership, and other effects unknown |
| F-2b2edb028a06 | Receives descriptor, resource-state storage, zero, and a static-storage value read after preparation | Zero enables processing; nonzero is returned directly |
| F-74745fd09174 | Receives resource-state storage, a transfer-length-cell address, and a destination for header or record data | Consumed zero status enables use of destination contents; completeness and failure-time writes unknown |
| F-49a51558071a | Receives resource-state storage once on normal post-acquisition exit | Result ignored; exact release and ownership effects unknown |

An additional preparation dependency receives a buffer address and byte count for the 48-, 260-, and 96-byte regions. No opaque ID was supplied for it. Its identity and semantics remain `documented_only`; no invented ID or clearing guarantee is assigned.

“Acquisition,” “transfer,” and “release” describe observed control-flow roles, not independently verified implementations.

## Acceptance examples

These examples are conditional static consequences, not executed tests. Unless stated otherwise, they assume valid storage, normal dependency returns, and no independent dependency writes to the node fields being illustrated.

### Acquisition failure

If F-2b2edb028a06 returns a nonzero status, the function returns that status. It does not subsequently resolve the input graph, read the header, process records, or call release.

### Preparation changes the acquisition argument

Suppose the static acquisition-argument storage contains one value on entry, and a preparation dependency changes it before returning. The acquisition call receives the value read after preparation, not an entry-time snapshot.

This example does not establish the actual stored value or any particular dependency’s ability or intent to change it.

### Header reading changes the marker

Suppose the static marker initially contains value A. The header-read dependency changes it to B, writes B into header offset `0x00`, and returns a zero low word. The marker comparison uses B.

If header offset `0x0C` also equals the captured metadata value, the header is accepted. A differing earlier value A does not itself cause rejection.

### Header rejection

If the header read reports zero but its first word differs from the marker read after that call, release occurs and the result is zero. There are no explicit node updates.

The same outcome occurs when the marker matches but the header word at `0x0C` differs from the captured expected value.

### Captured metadata remains the comparison input

If the metadata value is captured before header preparation, and a later dependency changes metadata offset `0x24`, the header comparison still uses the earlier captured value. The function does not reload that field.

### Accepted header and empty collection

If both comparisons pass and the captured first-node pointer is null, no record-read call occurs. Release is called and the result is zero.

### One successful node

Assume a node has:

- Field `0x90` equal to 10.
- Field `0xA0` equal to 20.
- Byte `0x8F` equal to `0xA0`.
- Field `0x1C` equal to `0x00000002`.
- A null successor.

A successful record supplying increments 3 and 4, byte `0x11` equal to one, and byte `0x10` nonzero produces:

- Field `0x90`: 13.
- Field `0xA0`: 24.
- Byte `0x8F`: `0xA8`.
- Field `0x1C`: `0x00004002`.

The assignment fields receive their corresponding values. A record value of 127 at offset `0x0C` produces a one-byte value of 127 at node offset `0x7E`, without requiring an out-of-range conversion assumption.

Traversal then reaches null, release occurs, and zero is returned.

### Flag boundaries

If node byte `0x8F` is `0xFF` and record byte `0x11` is two, the resulting node byte is `0xF7`. Nonzero is insufficient for this flag; the input must equal one.

If record byte `0x10` is zero, this function does not clear an existing `0x4000` bit or explicitly access that field at the conditional flag step.

### Failure after partial progress

If one record succeeds and the next read reports nonzero, the first node’s explicit changes remain. The second node receives no explicit record updates from this function. Release occurs and the failing status is returned.

### Short transfer reported as success

If a read reports zero while supplying fewer requested bytes, this function does not reject it by examining the resulting length cell. Consumed values depend on the dependency’s writes and existing scratch contents.

For a later record, those contents may include bytes from an earlier record; for the first record, they may include bytes left in the overlapping header region. No complete-transfer guarantee is established here.

## Uncertainties

The following remain `documented_only`:

- The referenced static string’s exact contents, encoding, termination, and bounds behavior.
- The acquisition argument’s value, declared width, meaning, and mutability.
- The marker’s numeric value and which dependencies, if any, modify it.
- The preparation dependency’s identity and memory effects.
- Transfer completeness, length-cell semantics, and failure-time writes.
- Meanings of nonzero status values and the header result’s upper 32 bits.
- External byte order, out-of-range narrowing semantics, and signed-overflow behavior.
- Application-level meanings of most metadata and node fields.
- Callee global effects, pointer retention, ownership, and exact resource-release semantics.

Playback-count processing is inferred intent. Timed static reads, argument flow, status branches, explicit memory accesses, and update order are established static behavior.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. No runtime tests have been performed.

Static evidence establishes the buffer sizes, pointer-chain offsets, captured metadata, overlapping scratch layout, consumed field widths, header gates, update order, successor-read timing, partial-progress behavior, and normal-path release calls.

The representation exposes a string-symbol reference rather than its underlying bytes. It also places the acquisition-argument read after preparation and the marker read after a zero-low-word header-read result. Those observations support timed reads of potentially mutable static storage, not literal string reconstruction or immutable-constant assumptions.

No dependency implementation, independent format definition, or runtime observation was supplied to resolve the documented unknowns.
