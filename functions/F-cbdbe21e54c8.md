<!-- Trusted provenance: F-cbdbe21e54c8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cbdbe21e54c8

## Purpose

F-cbdbe21e54c8 performs an ordered sequence of memory reads and writes through incoming bases, loaded addresses, and arithmetic-derived destinations, then invokes an unresolved indirect dependency.

No application-level purpose is established. Initialization, object linking, message handling, and device control are possible interpretations, not supported conclusions.

Two distinct behaviors must remain separate:

- **Narrowing for stored payloads:** several address-valued quantities are reduced to a byte before being written to memory.
- **Address formation:** arithmetic-derived destinations use retained 32-bit values or the converted result of a character read. The supplied representation does **not** explicitly use a truncated pointer value to construct an address.

The contract describes the supplied static representation. Its correspondence to executable instructions, the actual calling convention, and the execution environment remains unverified.

## Interface

### Explicit inputs and result

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit use in the body. |
| Second argument | 8 bits | Payload of two consecutive stores to the same context byte. |
| Third argument | 32 bits | No explicit use in the body. |
| Fourth argument, **request** | 32 bits | Full-width word payload, narrowed byte payload, and full-width operand in a final destination calculation. |

The represented return type is void. The indirect dependency’s result, if any, is not consumed. Normal return occurs only if all preceding operations complete and the dependency returns normally.

### Implicit inputs

Four incoming machine-state values are used without initialization in the represented body:

| Contract label | Role |
|---|---|
| **Context base** | Principal base for field accesses; also stored as an address-valued payload and narrowed for byte payloads. |
| **Control base** | Base of one word destination. |
| **Root base** | Starting point of the first two-stage address lookup. |
| **Incoming word** | Word payload written into the context before a later read of that field. |

Their provenance and calling convention are **documented_only**. The four explicit arguments alone are insufficient to describe a callable interface.

The body also reads three external 32-bit storage cells, designated **External A**, **External B**, and **External C**. These labels identify data dependencies, not callable dependencies. Their values and relationships to other accessed storage are unknown.

An unresolved indirect target is additionally required.

### Widths, offsets, and storage requirements

Offsets are measured in bytes. Byte, halfword, and word accesses have represented widths of 8, 16, and 32 bits respectively. “Low byte” and “low halfword” designate the least significant bits of a retained value, not an endian-dependent read from its storage.

The representation uses 32-bit integer quantities as addresses and includes a pointer-valued self-address store. The account below follows that represented address model; native pointer width and pointer-conversion rules are not independently established.

Normal completion requires every accessed destination and source to support its stated direction and width. This includes destinations formed by addition or doubling and destinations derived from a character value. No checks establish validity, bounds, alignment, or object lifetime.

## Behavior

### Ordering and aliasing

The following phases preserve the represented order of memory operations. Within each phase, prose order is execution order.

A retained value is a snapshot taken at its stated read. Subsequent writes do not change that snapshot. Different descriptive labels do not establish disjoint storage: context fields, linked locations, external cells, and computed destinations may overlap.

Repeated reads and writes are significant to this description. Whether they are independently visible to another thread, device, or observer is unknown.

### Initial context updates and primary lookup

The context byte at offset `0x0D` receives the second argument twice consecutively. Next, the incoming word is written at context offset `0x64`, followed by word `0x00000074` at control offset `0x30`.

The **primary object** is selected by reading a word at root offset `0x04`, then reading a word at offset `0x74` from that loaded base.

Only after this lookup does the function reread context offset `0x64`. Its low byte becomes the **incoming-byte snapshot**, which is written at context offset `0x1D`. This snapshot is not necessarily the low byte of the original incoming word: an earlier write may have overlapped the context field.

A second lookup reads context offset `0x44` and then offset `0x14` from the resulting base, yielding the **first linked address**. The full request is then written at primary-object offset `0x34`, and the first linked address is written as a word at context offset `0x14`.

The word at context offset `0x54` is retained as the **previous context word**. The low byte of the context base becomes the **context-byte payload** and is written at first-linked-address offset `0x09`. This narrowing supplies stored data; the destination still uses the full retained first linked address.

### Secondary lookup and linked-byte updates

The **secondary object** is obtained by reading primary-object offset `0x04`, then offset `0x74` from the loaded base.

The following updates occur in order: the low byte of request is written at secondary-object offset `0x15`; the context base is written as a word at context offset `0x54`; the low byte of the primary-object address is written at context offset `0x11`; and the low byte of the previous context word is written at secondary-object offset `0x11`.

The word at secondary-object offset `0x64` is then retained as the **secondary-word snapshot**. The incoming-byte snapshot is written at context offset `0x09`.

Primary-object offset `0x04` is now read again, producing the **byte-access object**. This is a fresh read, not reuse of the earlier lookup value. Intervening writes may have changed it.

At that newly selected object, the function writes the context-byte payload at offset `0x09`, reads and retains the word at offset `0x50` as the **object-word snapshot**, writes the low byte of the secondary-word snapshot at offset `0x11`, and repeats the context-byte store at offset `0x09`.

After those accesses, the secondary-word snapshot is written as a word at context offset `0x14`. The low byte of the byte-access-object address is written at context offset `0x0D`. Finally, the low byte of the object-word snapshot is written at byte-access-object offset `0x00`.

Neither narrowed address payload is used as a replacement address for these operations.

### Branch selection, self-address storage, and constants

A word read at primary-object offset `0x74` supplies the **byte-index displacement**. The next read, at primary-object offset `0x14`, selects the **branch object**.

Before following the branch, the function stores the byte-access-object address at that same object’s offset `0x54`. It then reads branch-object offset `0x04` to select the **terminal object**. This ordering matters: an overlapping self-address store can affect the terminal selection.

The terminal byte at offset `0x10` receives `0x64`. Next, context offset `0x54` receives word `0x00000064`, followed by the same word at terminal-object offset `0x40`.

The remaining updates in this phase occur in this order:

- Context offset `0x0D` receives the low byte of the secondary-word snapshot.
- Branch-object offset `0x09` receives the low byte of the terminal-object address.
- Context offset `0x24` receives the full secondary-word snapshot.
- Branch-object offset `0x11` receives the context-byte payload.

The earlier context self-address store at offset `0x54` is therefore later replaced by the constant word, absent additional effects from aliasing.

### Halfword writes and character-derived addressing

External A is read and retained first. The word at context offset `0x74` is then read as the **context displacement**; its low halfword is retained as the **displacement payload**.

The displacement payload is written as a halfword at context base plus context displacement. Next, the low halfword of External A is written at the address obtained by doubling the retained terminal-object value.

The function then reads one plain-character byte at context base plus byte-index displacement. Conversion of that character to a 32-bit unsigned value produces the **expanded character**.

Two more halfword writes follow: the displacement payload is written at External A plus context displacement, then written again at context base plus context displacement. The payload comes from the displacement, not from either calculated destination.

Finally, word `0x00000045` is written at expanded character plus `0x45`.

These destination calculations use retained full-width quantities. In particular:

- Doubling uses the retained terminal-object value, not its earlier byte payload.
- The External A addition uses its retained 32-bit value, not the halfword previously stored from it.
- Context-based displacement addition uses the retained context base and displacement.
- Character-derived addressing uses the converted character read, not a truncated pointer.

Plain-character signedness is unresolved. For example, byte pattern `0x80` produces `0x00000080` under unsigned-character interpretation. Under signed-character interpretation with ordinary conversion of negative values to 32-bit unsigned, it produces `0xFFFFFF80`. That difference affects both subsequent destinations and dispatch.

### Final snapshots, writes, and dispatch

The function reads External B, then External C, then the word at context offset `0x44`. The last value is retained as the **dispatch context word**.

Three stores occur after those snapshots:

1. Byte `0x44` at expanded character plus `0x44`.
2. External C as a word at the address obtained by doubling External B.
3. The low halfword of External B at request plus External B.

The final destination calculation uses the full request and full retained External B. Narrowing External B supplies only the halfword payload.

F-unknown-indirect is then invoked with three represented 32-bit arguments, in this order:

1. Expanded character bitwise OR `0x00000044`.
2. Retained External B.
3. Dispatch context word.

The first argument uses bitwise OR, not addition. It is not reloaded from either character-derived destination. The third argument likewise remains the earlier snapshot even if one of the final stores overlaps context offset `0x44`.

## Side effects

Direct effects include byte, halfword, word, and address-valued stores through incoming bases, loaded addresses, and arithmetic-derived destinations.

The body contains no explicit allocation, release, reference-count operation, synchronization operation, validation branch, or error-reporting mechanism. This establishes only the absence of those explicit operations in the representation. It does not constrain the indirect dependency.

No ownership transfer is established by storing an address or a self-address. Ownership, retention rights, and lifetime requirements remain **documented_only**.

No rollback is represented. An invalid later access or a failure in the indirect dependency may occur after earlier writes. Whether such failures produce faults, exceptions, nonlocal exits, or other effects is unknown.

Endianness affects the relationship between overlapping accesses of different widths. Alignment requirements, access atomicity, memory-mapped-device behavior, and concurrent visibility are not established. The ordered account must not be interpreted as a synchronization guarantee.

## Dependencies

### F-unknown-indirect

This opaque label denotes the unresolved indirect target. It does not identify a recovered implementation.

Established call-site behavior:

- One invocation is represented if execution reaches dispatch.
- Three 32-bit arguments are supplied in the order specified above.
- No return value is consumed.
- A void return follows only if the dependency returns normally.

The target’s identity, selection mechanism, full calling convention, additional machine-state inputs, global-state effects, memory effects, ownership behavior, and termination behavior are **documented_only**.

There are no other explicit calls in the supplied body. No preservation of globals or referenced storage may be assumed across this dependency.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Each assumes the required accesses and arithmetic are supported by the execution environment.

### Repeated initial byte store

With second argument `0xA7`, the first two represented stores both write byte `0xA7` at context offset `0x0D`.

This is an event-order expectation, not a final-value guarantee. Later operations write the same field.

### Full request versus narrowed payload

With request `0x12345678` and nonoverlapping relevant fields:

- Primary-object offset `0x34` receives word `0x12345678`.
- Secondary-object offset `0x15` receives byte `0x78`.
- The final halfword destination is calculated using `0x12345678` plus External B, not `0x78` plus External B.

### Address narrowing does not redirect later accesses

If the retained primary-object address is `0x001234AB`, the store at context offset `0x11` carries byte `0xAB`.

Primary-object field reads still use `0x001234AB` as their base. The byte payload does not establish an address base of `0xAB`.

### Fresh pointer selection

If primary-object offset `0x04` changes between the secondary-object lookup and the later byte-access-object read, the later operations use the newly read address.

Caching the earlier value would not preserve the represented behavior under that aliasing condition.

### Halfword payload and repeated destination

With context displacement `0x12345678`, the two writes at context base plus that displacement both carry halfword `0x5678`. The intervening write at External A plus that displacement carries the same payload.

These are repeated writes to a retained destination calculation, not advancement of an output cursor.

### Character-derived destinations and bitwise OR

If the character read yields `0x21`, either character-signedness interpretation gives expanded character `0x00000021`.

The word destination is address `0x66`, and the later byte destination is address `0x65`. The first dispatch argument is `0x00000065`, obtained by bitwise OR.

To distinguish OR from addition, an expanded character of `0x44` gives first argument `0x44`, not `0x88`.

These examples do not establish that low numeric addresses are writable.

### Captured dispatch argument

If the final read of context offset `0x44` yields `0x11223344`, dispatch receives that value as its third argument even if a subsequent direct store changes the field.

The expectation applies only if dispatch is reached; it does not constrain the dependency’s later actions.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and semantic field meanings.
- Provenance and calling convention of the implicit inputs.
- Accuracy of the supplied representation relative to executable instructions.
- Native pointer width, address conversion, and pointer-storage details.
- Plain-character signedness and high-bit character conversion.
- Byte order, alignment, atomicity, and concurrency semantics.
- Overflow behavior for integer additions and doubling.
- Validity of character-derived and other arithmetic-derived destinations.
- Aliasing among all accessed regions, including external cells.
- Whether storage is ordinary memory, shared memory, or device-backed.
- Ownership, lifetime, retention, and mutation permissions.
- Indirect-target identity and complete behavior.

Integer arithmetic is not uniformly established as wrapping. Several address operands are represented as signed integers, whereas expanded-character arithmetic uses an unsigned 32-bit value. The contract therefore does not promise defined behavior for overflowing signed calculations or invalid pointer conversions.

No success indicator, failure result, null-handling policy, or recovery contract is established.

## Evidence

The supplied static representation establishes the access sequence, offsets, literal values, retained snapshots, explicit narrowing operations, arithmetic-derived destinations, and final three-argument indirect invocation described here.

It supports narrowing address-valued quantities for stored byte payloads. It does not establish address formation from those narrowed payloads. Arithmetic-derived destinations instead use retained 32-bit operands or the converted character read.

The object and snapshot labels are descriptive aids only. They assert no types, disjointness, ownership, or application intent.

No runtime tests, execution traces, memory observations, ABI validation, or independent instruction-level verification have been performed. The acceptance examples are conditional static expectations, not observed executions.
