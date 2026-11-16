<!-- Trusted provenance: F-ad0c89b25f6a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ad0c89b25f6a

## Purpose

F-ad0c89b25f6a performs an ordered sequence of indirect invocations, fixed-width field transformations, global captures, and memory updates, followed by an unresolved indirect transfer.

The supplied static representation does not establish a domain-level purpose. Printable constants do not establish serialization, formatting, a string, or user-visible output.

This contract distinguishes:

- **Established static evidence:** represented operations, data origins, access widths, and ordering.
- **[INFERENCE]:** interpretations that are not acceptance requirements.
- **documented_only:** behavior unresolved by the supplied evidence.

Requirements are conditional on execution reaching the relevant operation. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has three positional inputs and no declared return value.

| Input | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit body use |
| Second | 32 bits | No explicit body use |
| Third | Unsigned 32 bits | Bits 10–19 supply a dependency argument |

The third input contributes a ten-bit unsigned field, scaled by 64. The resulting argument ranges from 0 to 65,472 and has its low six bits clear.

The absence of explicit uses of the first two inputs does not establish complete execution independence from them. Unresolved invocation signatures may consume incoming registers or other machine state.

### Implicit inputs

Execution also depends on indirect targets, incoming control state, dependency result channels, global storage, and memory reached through arithmetic-derived addresses.

Three global captures are particularly significant:

| Capture | Established role |
|---|---|
| Global at 0x05 | Converted to an unsigned 32-bit address-like value early in the direct-memory phase |
| Global integer at 0x087FBBBC | Supplies a retained base for the long repeated-write region |
| Global at 0xA9 | Supplies a retained word-sized value after that region |

These are direct state dependencies, not hypothetical effects of dependencies.

The declarations of the globals at 0x05 and 0xA9 are unavailable. Their storage-access widths must not be inferred solely from their addresses, names, conversions, or destination containers. The later base capture is represented as a 32-bit integer; its underlying declaration and machine load are also unavailable.

### Outputs, preconditions, and ownership

There is no established scalar success or failure result. Observable outputs include invocations, direct reads and writes, and the terminal transfer. Ordinary return is conditional on that transfer returning.

Complete traversal requires executable indirect targets, suitable calling conventions, accessible memory of the required widths, meaningful derived and low addresses, and supported alignment. No validation establishes these conditions.

No allocation, release, ownership transfer, reference counting, object boundary, or lifetime guarantee is established. Dependencies must not be assumed to preserve globals or retain or release memory.

## Behavior

### Overall ordering

The represented body contains a long straight-line invocation phase, a direct-memory phase, and a terminal indirect transfer. No explicit input-dependent branch or loop appears. This does not establish complete recovered control flow or dependency termination.

Invocation order remains significant even when a result is unused. Dependencies may mutate memory, globals, machine state, or external state. No direct data-memory store is explicitly represented before the direct-memory phase; dependency-internal stores remain possible.

### Widths and arithmetic

Most working values are represented as 32-bit quantities. Some dependency results occupy 64-bit containers whose low and high 32-bit portions are consumed separately. This does not establish a public 64-bit return type or memory byte order.

The invocation phase repeatedly passes shifted result words, separate high-result words, zeros, masked fields, and multiple transformations of retained results. Visible masks include 24-, 22-, 21-, 20-, 19-, 18-, 17-, 14-, 13-, 12-, and 11-bit selections.

Under a 32-bit bitvector interpretation, left shifts and multiplication discard bits above bit 31. Truncation at each operation matters; a later right shift does not restore discarded bits.

Established bounds include:

- A low 24-bit field scaled by 64 fits within 30 bits.
- A low 22-bit field scaled by 256 fits within 30 bits.
- A low 19-bit field scaled by four fits within 21 bits.
- Unsigned right shifts zero-fill.

Some operations use signed represented intermediates. Overflow, signed left shifts, and signed right shifts do not establish portable language behavior. Target semantics remain documented_only where those distinctions affect the result.

### Opening direct-memory effects

The opening updates use dependency-derived bases, including a value scaled by \(2^{25}\), a doubled masked field, and a low result word scaled by 256.

The established opening order is:

- A word store at offset 0x14 from a dependency’s high-result base.
- A byte store at offset 9 from another derived base.
- A word store at offset 0x54 from the low-result base scaled by 256.
- Another byte store to the preceding offset-9 destination.
- A zero word store at offset 0x14 from the \(2^{25}\)-scaled base.
- An unsigned byte read at offset 5 from that scaled base.

The offset-5 byte is subsequently zero-extended into a word store at offset 0x34. A zero store through the doubled-field base occurs between that byte read and word store.

The function then reads the scaled base’s offset-0x14 word and retains it. That read cannot automatically be replaced by zero: intervening destinations may alias the cleared word. Another zero word store through the doubled-field base and two duplicate byte stores at its offset 0x15 precede the first significant global capture.

### Early global capture and retained address dependency

The global at 0x05 is read and converted to an unsigned 32-bit address-like value **before** the fixed-address byte write at 0x01.

This conversion establishes the retained value’s width and unsigned interpretation, not the original global load width. In particular, the source does not justify calling the global a byte or treating the capture as a pointer-sized memory read.

The retained value determines:

- A byte destination at offset 0x15.
- A word-read address at offset 0x54.
- The destination of a subsequent word overwrite at that same offset 0x54.

Its timing is significant. After the capture, the byte at 0x01 receives a truncated doubled-base value. The offset-0x15 byte destination selected by the capture then receives a truncated masked-field value.

Before the captured base’s offset-0x54 read, the representation also writes a shifted word’s low byte at the scaled base’s offset 0x11, stores that shifted word at offset 0x54 from the earlier retained offset-0x14 value, and clears the scaled base’s offset-0x11 byte.

The offset-0x54 read therefore observes memory at its own later point in the sequence. Its result is retained as a principal update value. Only after this read does the function overwrite the captured base’s offset-0x54 word with the earlier offset-0x14 value.

Neither capture may be moved freely:

- Deferring the 0x05 capture until after the 0x01 write may change the captured value if the accesses overlap.
- Moving the offset-0x54 read before intervening writes may change the principal update value.
- Re-reading the global at 0x05 instead of retaining its converted value introduces a different state dependency.

### Traversal and mixed-width updates

The principal update value drives subsequent destinations and arithmetic, including bases scaled by four and eight, a fixed word write at 0x54 containing its doubled value, and an adjusted quantity formed by subtracting 150 from its fourfold value.

Word reads at offsets 0x64 and 0x74 supply further bases. One offset-0x74 traversal uses a zero-extended byte read at offset 5 as its base. That is a byte-derived integer address, not a full-width pointer loaded from offset 5.

Repeated address-bearing reads remain distinct observations. For an update that reads an offset-0x74 field both to obtain a destination base and to obtain a stored value, the representation does not establish a single reusable snapshot. Relative evaluation order within that individual compound update is not resolved here.

The direct-memory layout includes:

| Offset | Represented accesses |
|---|---|
| 0x01 | Byte store |
| 0x04 | Word reads and unsigned byte read |
| 0x05 | Unsigned byte reads |
| 0x08, 0x09, 0x0D | Byte stores |
| 0x10 | Byte and word stores |
| 0x11 | Byte stores |
| 0x14 | Word reads and stores |
| 0x15, 0x19 | Repeated byte stores |
| 0x20 | Repeated word stores |
| 0x24, 0x34 | Word stores |
| 0x40 | Word read |
| 0x44, 0x54 | Word reads and stores |
| 0x64 | Word reads |
| 0x74 | Word reads and stores |

“Word” means 32 bits in these explicitly represented accesses. These observations do not establish a structure declaration.

### Constants and later global-base capture

A constant-writing region stores numeric 0x3A through three distinct accesses: an 8-bit store at a fourfold derived address, a 16-bit store at the corresponding eightfold address, and a 32-bit store at offset 0x20 from the fourfold address.

Another word destination at offset 0x20 receives, in order:

1. 0x20.
2. 0x2F.
3. A zero-extended byte read from offset 4 of its base.
4. 0x20 again.

The byte read occurs after the first two stores and before the third. Aliasing can therefore affect it.

**After this complete four-write sequence**, the global integer at 0x087FBBBC is captured as the repeated-region base. **After that capture**, fixed-address storage at 0x60 receives numeric 0x20.

The base capture must not be moved before the constant sequence or after the 0x60 store without additional aliasing assumptions. Throughout the following repeated region, the captured base is retained rather than repeatedly loaded from its global storage.

### Repeated-write region

Each recurring group comprises four ordered effects:

- A word store at offset 0x54 from a scaled destination.
- A byte store at offset 0x19 from the captured global base, containing the low byte of the adjusted fourfold address.
- A byte store at offset 0x11 from that adjusted fourfold address, containing the captured base’s low byte.
- A word store at offset 0x34 from the captured base.

The initial group uses the principal update value scaled by eight for the first destination base and final stored value. Subsequent repeated groups use the adjusted quantity scaled by eight. Their first stored word comes from the earlier traversal result scaled by four.

The byte payloads are retained truncated values. Writes that alias the global base’s storage do not, by themselves, cause the function to recapture that base or recompute its retained byte.

Duplicate groups remain distinct ordered writes. A final-state-only replacement is not justified where access count, intermediate states, or device semantics are observable. This document describes the recurring behavior without enumerating every occurrence.

### Post-repetition global-value capture

Immediately after the repeated region, the global at 0xA9 is captured into a retained 32-bit value container.

This capture precedes both:

- A word read at offset 0x74 from the traversal-derived base scaled by four.
- The fixed-address byte write at 0x3A.

The 0x3A byte is derived from eight times the adjusted quantity plus 0x30, with byte truncation. Its value does not establish a character encoding or message.

The 0xA9 capture is independent of the earlier global-base capture. Its retained value later supplies a byte write at 0x67 and a word write at offset 0x44 from the adjusted fourfold base. Those uses do not re-read 0xA9. The global’s declaration, original access width, and any extension into the retained container remain documented_only.

### Live reads at 0x67

The final region distinguishes the retained 0xA9 value from **live byte-valued reads of storage at 0x67**.

After several derived-memory writes, the function writes the retained value’s low byte to 0x67. It then reads a derived offset-0x44 word before reading 0x67 for the byte write at fixed address 0x21.

Further derived-memory operations occur before two later reads of 0x67. Each later read supplies a separate byte store at offset 9 from the adjusted fourfold base.

Between the 0x21 use and the first later offset-9 use, the represented effects include:

- Byte and word writes at derived offsets 0x10 and 0x54.
- A shift-derived byte write at offset 1.
- A word read at offset 0x40.
- Word writes at derived offsets 0x74, 0x10, and 0x44.

Between the two later offset-9 uses, the function writes a byte at offset 9 from the doubled traversal base, writes the retained 0xA9 value at the adjusted fourfold base’s offset 0x44, and reads that base’s offset-0x54 word.

Consequently, there are three distinct live consumers of 0x67: the 0x21 write and the two later offset-9 writes. They need not observe the byte initially written there or agree with one another. Intervening derived writes may alias 0x67. The first later offset-9 write may itself alias it.

A subsequent offset-9 store uses a truncated derived address, not another 0x67 read.

### Final handoff

Other final effects include an 8-bit store of 0x6C, a 16-bit store of 0x65 at a doubled derived address, a 32-bit store of 100, retained-word copies into offset-0x54 destinations, and a word store at a doubled address obtained through an offset-0x64 read.

One byte payload comes from shifting a signed represented word right by the low eight bits of the adjusted fourfold address. Counts above 31 are possible. Signed-shift and oversized-count behavior remain documented_only.

The terminal indirect transfer has four represented arguments:

1. Numeric 0x72.
2. A retained word obtained through an offset-0x64 read.
3. A word captured earlier in the final region from offset 0x14 of the doubled traversal base.
4. A word captured late from offset 0x74 of a subsequently obtained base.

The third argument is not the opening direct-memory phase’s offset-0x14 capture. No explicit rollback or restoration follows the transfer.

## Side effects

Direct effects include writes to derived memory and fixed storage at 0x01, 0x15, 0x21, 0x3A, 0x54, 0x60, and 0x67. Fixed symbolic word destinations at 0x54 and 0x60 lack declarations in the supplied material; exact storage-access widths must not be invented from their assigned values alone.

The representation warns that global symbols overlap smaller storage. Distinct names do not establish distinct storage. Endianness, global access widths, and derived-address aliasing determine interactions.

Captured values and live memory are different state sources. In particular, the retained 0x05-derived address, repeated-region base, and 0xA9-derived value remain snapshots; the three later 0x67 uses are live reads.

No synchronization, atomicity, reentrancy guarantee, transaction, validation, retry, or cleanup path is established. Faults and non-returning dependencies may leave preceding effects in place.

## Dependencies

The supplied map resolves only F-ad0c89b25f6a. It provides no opaque IDs for the indirect dependencies; none are fabricated here.

| Dependency category | Established contribution | Unresolved contract |
|---|---|---|
| Entry-state-relative invocation | First indirect invocation | Target, signature, incoming-state consumption |
| Subsequent indirect invocations | Ordered arguments and result-channel use | Identity, complete signatures, effects, ownership |
| Terminal indirect transfer | Four represented arguments | Target, dispatch, return and failure behavior |

A displayed argument-free invocation does not prove absence of implicit inputs. A two-word result representation does not establish a public aggregate return type.

## Acceptance examples

These are conditional static examples, not executed tests.

### Third-input boundaries

| Third input | Required extracted argument |
|---:|---:|
| 0 | 0 |
| 1,023 | 0 |
| 1,024 | 64 |
| 1,048,575 | 65,472 |
| 1,048,576 | 0 |
| 0xFFFFFFFF | 65,472 |

Changing bits outside 10–19 preserves this argument, not necessarily the whole execution.

### Early capture versus later memory

If the converted 0x05 capture is 0x1000, its destinations are based on 0x1000, including 0x1015 and 0x1054. A later overlapping write that changes the global does not change that retained address. The read at 0x1054 still occurs after the intervening writes and before its overwrite.

### Repeated-region base capture

If the constant-write sequence changes storage overlapping the global base, the capture observes state after that sequence. If the subsequent 0x60 store changes that global’s storage, the repeated region nevertheless uses the already captured base.

### Separate late capture

If the 0xA9 capture yields 0x123456AB, the later 0x67 write receives 0xAB and the corresponding word write receives the retained 0x123456AB. Changes to 0xA9 after capture do not replace those retained payloads.

### Live 0x67 observations

Suppose the initial 0x67 write stores 0xAB. If intervening aliasing changes it to 0xCD before the 0x21 use, that use observes 0xCD. If it later contains 0x12 and 0x34 at the two offset-9 reads, those stores receive 0x12 and 0x34 respectively. Reusing 0xAB for all three is insufficient.

### Width and truncation

An unsigned offset-5 byte of 0xFE promotes to 254, not a negative word. A 16-bit store of 0x65 must not become a byte store.

Under 32-bit bitvector semantics, 0x01000001 shifted left eight and then logically right eight yields 1.

### Partial completion

An inaccessible derived address has no established graceful rejection. A non-returning dependency prevents requirements for later operations from becoming applicable. Reaching the terminal transfer requires first argument 0x72 and the specified remaining data origins.

## Uncertainties

The following remain **documented_only**:

- Domain purpose, caller expectations, and complete control flow.
- Dependency identities, signatures, implicit state, and ownership behavior.
- Missing global declarations and unresolved global access widths or conversions.
- Endianness, alignment, low-address mapping, and alias relationships.
- Target overflow, signed-shift, and oversized-shift behavior.
- Ordinary-memory versus device or otherwise access-observable semantics.
- Concurrency, object boundaries, lifetimes, faults, and terminal return behavior.

**[INFERENCE]:** masks suggest field manipulation, and offset-based traversals suggest structured state. Neither establishes an application model or permits replacing ordered accesses with a cleaner abstraction.

## Evidence

This contract uses the supplied static representation and dependency map, not the prior draft or feedback as behavioral authority.

The global capture timing, retained address dependencies, repeated-region base capture, separate 0xA9 capture, and three live 0x67 consumers are directly represented. Missing declarations prevent stronger claims about several symbolic global accesses.

No dependency bodies, callers, executable observations, runtime tests, smoke runs, or memory traces were available or performed. Acceptance examples are analytical conditions only. No snapshot hash is asserted.
