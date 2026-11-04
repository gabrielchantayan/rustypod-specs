<!-- Trusted provenance: F-46f8baffa44d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-46f8baffa44d

## Purpose

F-46f8baffa44d writes a five-word record to caller-designated storage, reads two halfword samples, and performs further writes selected by whether its second explicit input is zero. If execution completes normally, it returns its fifth explicit input unchanged.

This is an ordered memory-access contract, not a validated record-construction interface. The record is written before the sample reads. A zero second input still causes a read at address 4; it does not bypass address-dependent operations.

Application-level intent is unresolved. Exception-associated names appear among the global destinations, but they do not establish handler installation, an active exception-vector layout, or execution in an exception context.

All established behavior below comes from static evidence. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input position | Represented width and kind | Role |
|---|---|---|
| First | Unsigned 32-bit integer | Supplies the highest four bits for the first record word. |
| Second | Signed 32-bit integer | Determines one sample address and selects the zero or nonzero outcome. |
| Third | Pointer to unsigned 32-bit storage | Identifies the record destination; its address is also encoded in the record. |
| Fourth | Signed 32-bit integer | Supplies the quantity reduced by 31 for the third record word. |
| Fifth | Opaque 64-bit value | Supplies the unchanged normal return value. |

The pointer’s machine width is not independently established. Its stored representation occupies 32 bits. Preservation of any wider address bits must not be assumed.

For normal completion, the record destination must support five consecutive 32-bit stores spanning 20 bytes. There is no represented capacity argument, null check, alignment check, address validation, or overlap check.

The second input is an integer used in address formation, not an optional pointer with an established null convention. Zero has a branch-selection meaning but does not mean “do not access.”

### Ambient inputs

Three additional values are consumed without a definition established inside this function:

- A signed 32-bit **base seed**, used to derive an address base.
- An unsigned 32-bit **record-only value**, stored in the fourth record word.
- An unsigned 32-bit **shared value**, stored in the fifth record word and assigned to a global destination on the zero-input outcome.

These describe required incoming machine state in the supplied representation. Their producers, availability at entry, and relationship to the actual calling convention are unresolved. They must not be treated as zero-initialized values or as evidence of additional ordinary parameters.

### Result and failure interface

Normal completion returns all 64 bits of the fifth input unchanged.

There is no separate status result, explicit rejection outcome, or represented recovery mechanism. Successful return is conditional on execution continuing through all required operations; the returned value is not a validation result.

A memory fault or arithmetic condition outside the established domain may prevent normal completion. The evidence does not specify fault delivery, recovery, continuation, or rollback.

## Behavior

### Record layout and construction boundary

The record has five adjacent unsigned 32-bit words:

| Byte offset from record start | Construction value |
|---|---|
| 0 | The first input’s highest four bits, interpreted as an unsigned value from 0 through 15. |
| 4 | The record’s starting address converted to an unsigned 32-bit value. |
| 8 | The fourth input minus 31, converted to an unsigned 32-bit value. |
| 12 | The ambient record-only value. |
| 16 | The ambient shared value. |

The address field refers to the start of the record, not to the address field itself. No sixth word, terminator, size field, or additional initialization is established.

The five stores occur in increasing offset order. All five precede both sample reads and every branch-dependent write. This describes the values written during construction; later overlapping writes can change the same storage before return.

The subtraction uses a represented signed 32-bit operand. Its mathematical result is representable when the fourth input is between −2,147,483,617 and 2,147,483,647, inclusive. Within that interval, the result is converted to unsigned 32-bit storage. Thus a fourth input of 30 produces `0xFFFFFFFF`.

Below that interval, signed subtraction exceeds the represented range. Unconditional wraparound is not established.

### Derived addresses and sample reads

The derived base is the ambient base seed multiplied by 268,435,456. Its computation follows record construction in the supplied representation.

The two samples are unsigned 16-bit values:

| Sample | Address expression |
|---|---|
| Input-derived sample | Second input plus 4 |
| Base-derived sample | Derived base plus 40 |

The input-derived read precedes the base-derived read. Both precede all writes specific to either outcome.

Both reads are required regardless of whether their values ultimately contribute to a write:

- A nonzero second input still requires the input-derived read, although that sample has no subsequent explicit use on that outcome.
- A zero second input still requires both reads, including the read at address 4.
- The base-derived sample is read before the zero-input outcome selects a constant for its terminal store.

Both samples are unsigned. Their use in explicitly 32-bit stores is by zero extension, not sign extension.

Address arithmetic has represented signed 32-bit operands. The multiplication is mathematically representable only for base seeds from −8 through 7. The second-input addition is representable only when that input is at most 2,147,483,643. Representable arithmetic does not by itself establish a valid memory address: integer-to-address interpretation, mapping, permissions, and alignment remain environmental requirements.

Under an independently established low-32-bit wrapping interpretation, only the base seed’s lowest four bits affect the derived base’s bit pattern. This conditional arithmetic fact is not a guarantee that wrapping applies.

### Nonzero-input outcome

When the second input is nonzero, the function performs one further explicitly represented write after the two sample reads:

| Destination | Access width | Value |
|---|---|---|
| Derived base plus 48 | 32 bits | Zero-extended base-derived sample |

None of the zero-input outcome’s five explicit global assignments occurs.

This absence does not guarantee that global storage remains unchanged. The record destination or the indirect destination can alias global storage.

After the store, normal completion returns the fifth input unchanged.

### Zero-input outcome

For this outcome, the five named global destinations are identified below by neutral labels. The labels distinguish assignment targets; they do not assert distinct addresses, particular storage widths, or operational roles.

| Destination | Assigned value |
|---|---|
| Global 1 | Derived base minus 5 |
| Indirect destination at derived base plus 59 | Zero-extended base-derived sample, stored as 32 bits |
| Global 2 | The literal value represented as `0xFFFFFB00` |
| Global 3 | Unsigned input-derived sample promoted to a 32-bit value |
| Global 4 | Derived base |
| Global 5 | Ambient shared value |
| Terminal destination at address 48 | 141, stored as 32 bits |

The assignment targets have this strict represented precedence:

- Global 1 precedes the indirect store at derived base plus 59.
- That indirect store precedes Global 2.
- Global 2 precedes Global 3, which precedes Global 4, which precedes Global 5.
- Global 5 precedes the terminal store at address 48.

All these effects follow both sample reads. Normal return follows the terminal store.

The two indirect stores are separate requirements. In particular, the write at derived base plus 59 is not replaced by the terminal store. The terminal store uses address 48 and value 141 independently of the base-derived sample.

The declarations of the globals are unavailable. The assignment expressions above are established, but final storage conversion, access width, and byte effects for those globals are not. For Global 2, the literal’s bit pattern must not be expanded into a claim about the global’s signed interpretation or storage width.

The subtraction of 5 introduces an additional signed-range boundary. A base seed of −8 gives the minimum signed 32-bit derived base, so the Global 1 expression falls outside that range. Consequently, the multiplication’s representable seed range alone is insufficient to establish all arithmetic on the zero-input outcome.

## Side effects

### Ordering and partial completion

The represented memory-access order has three boundaries: record construction, ordered sampling, then outcome-specific writes.

If all record stores succeed but the first sample read faults without continuation, the complete construction writes precede the failure; the second sample read and subsequent writes have not been reached. If the second sample read fails, the first has already occurred. A later failure can leave a prefix of the outcome-specific writes completed.

No rollback is represented. These observations describe execution that reaches the relevant operation with preceding arithmetic and accesses completed; they do not assign behavior to earlier arithmetic outside its established domain.

The ordering is evidence about the supplied representation. It does not establish volatile qualification, compiler barriers, hardware ordering, atomicity, or the sequence visible to another processor or external device.

### Aliasing

No separation among the record, sample locations, indirect destinations, and global storage is enforced.

A sample that overlaps the record reads storage after record construction, not necessarily its entry contents. The numeric halfword obtained depends on the overlap and the target’s byte order.

Later stores can overwrite constructed record bytes or previously written global storage. Therefore, construction values are not unconditional post-return contents.

The supplied warning establishes that some named globals overlap smaller symbols at the same address. It does not identify a complete overlap map or prove that the five listed global destinations are mutually independent.

### Access widths and alignment

The five record stores and the two outcome-dependent indirect destinations are explicitly 32-bit accesses. The two sample reads are explicitly 16-bit accesses. Global assignment widths remain unresolved.

When the derived base has the represented multiple-of-268,435,456 address interpretation, adding 59 produces an address not divisible by four. Normal completion of the zero-input outcome therefore requires that this word access be permitted or that the environment otherwise allow execution to continue. No alignment repair or replacement by byte accesses is established.

The zero-input outcome also requires the halfword read at address 4 and the word store at address 48. Their low numeric addresses do not establish mapping or permission.

### Ownership and concurrent activity

No allocation, release, ownership transfer, or lifetime extension is represented. Storing the record’s address inside the record does not establish registration or retention.

No synchronization protocol is present. Concurrent modification, device-memory semantics, and fault-handler effects require information not supplied here. No preservation guarantee for unrelated state follows from this contract.

## Dependencies

No direct or indirect function call appears in the supplied body. There are no additional opaque function IDs to enumerate.

Execution nevertheless depends on:

- Availability of the three ambient inputs.
- Defined arithmetic and address-conversion behavior for the selected values.
- Accessible storage with suitable permissions for every required access.
- Support for the represented widths and relevant alignment.
- Actual global declarations and overlap relationships.
- The environment’s fault and continuation behavior.

Any mechanism entered because an access faults is outside the established call contract. Its behavior, global-state preservation, and memory ownership effects must not be assumed.

## Acceptance examples

These are conditional consequences of static evidence, not observed executions. Unless stated otherwise, each example assumes defined arithmetic and conversions, successful accesses, and no interfering concurrent activity.

### Nonzero outcome with separate storage

Assume:

- First input: `0xD1234567`.
- Second input: a nonzero value with a valid sample address.
- Fourth input: 40.
- Base seed: 1, giving derived base `0x10000000`.
- Base-derived sample: `0xABCD`.
- Relevant destinations do not overlap.

The construction values are 13, the converted record address, 9, the ambient record-only value, and the ambient shared value.

The terminal 32-bit store writes `0x0000ABCD` at address `0x10000030`. No zero-input global assignment occurs. The return preserves the fifth input’s complete 64-bit value.

Omitting the input-derived read because its value is unused would violate the represented access behavior.

### Zero outcome with permitted low-address and unaligned accesses

Assume the second input and base seed are both zero. The halfword at address 4 supplies `0x8123`, and the halfword at address 40 supplies `0x4567`. Assume the global assignments are supported and all destinations are separate.

After record construction and the two reads:

- Global 1 receives an assignment expression with value −5.
- The 32-bit store at address 59 writes `0x00004567`.
- Global 2 receives the specified `0xFFFFFB00` literal assignment.
- Global 3 receives the positive unsigned sample value 33,059, not a sign-extended negative value.
- Global 4 receives zero.
- Global 5 receives the ambient shared value.
- The terminal 32-bit store at address 48 writes 141.

These effects must respect the precedence specified above. Normal return preserves the fifth input.

Exact stored bytes for the globals remain conditional on their missing declarations.

### Failure at address 4

Assume the second input is zero, all record stores succeed, base derivation completes, and the halfword read at address 4 faults without continuation.

The construction writes have occurred. The base-derived read, global assignments, terminal store, and normal return have not been reached.

Treating zero as permission to omit the read would change the contract.

### Sample overlapping the record

Assume the input-derived sample address equals the record start and no other write interferes.

With first input `0xD1234567`, construction places the unsigned word 13 there before sampling. The sample therefore comes from that newly written word.

With conventional little-endian storage, the sampled halfword is 13. With conventional big-endian storage, it is zero. Neither result is unconditional without a byte-order premise.

### Arithmetic boundaries

A fourth input of 30 requires the third construction word to be `0xFFFFFFFF`.

For the minimum signed 32-bit fourth input, subtraction by 31 exceeds the signed range. If wrapping arithmetic is independently established, the resulting low 32 bits are `0x7FFFFFE1`; otherwise this case remains unresolved.

A base seed of −8 makes base derivation representable, but on the zero-input outcome the subsequent subtraction of 5 is not representable. Defined base multiplication alone does not establish defined completion.

## Uncertainties

The following remain **documented_only**:

- The actual calling convention and completeness of the explicit interface.
- The origin and availability of ambient values.
- Pointer width and pointer/integer conversion rules.
- Behavior of signed arithmetic outside its representable range.
- Byte order and treatment of negative integer address expressions.
- Global declarations, widths, addresses, and complete overlap relationships.
- Whether accessed storage is ordinary memory, device memory, or active exception-related storage.
- Fault handling, continuation, and unaligned-access behavior.
- Atomicity, synchronization, and externally observed ordering.

Possible record-format or exception-management intent is an inference, not an acceptance requirement. No handler-installation, processor-configuration, or ownership protocol is established.

## Evidence

The established evidence is the supplied static representation of F-46f8baffa44d and its warning concerning overlapping global symbols.

It supports the five-word construction layout, ordered stores and sample reads, zero/nonzero selection, distinct zero-input indirect stores, five ordered global assignments, explicitly sized indirect accesses, and unchanged 64-bit normal return.

It does not supply global declarations, a complete calling convention, a memory map, or execution observations. Conclusions requiring wrapping arithmetic, particular address conversions, byte order, or successful unaligned access remain expressly conditional.

No runtime tests have been performed.
