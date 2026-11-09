<!-- Trusted provenance: F-cbc8c552d27e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cbc8c552d27e

## Purpose

F-cbc8c552d27e performs an indefinitely repeating sequence of memory updates and opaque dependency calls. Its application-level purpose is not established. The supplied body exposes neither an ordinary return nor a success or error result.

The observable contract includes access widths, overlapping writes, sampling order, dependency selection, a global-byte assignment, and a breakpoint carrying immediate value 4. Equivalent final memory contents alone are insufficient: an implementation that combines writes, samples through a replacement pointer too early, or substitutes the global byte for a retained snapshot would change the established behavior.

This specification distinguishes:

- **Established static evidence:** behavior directly supported by the supplied body.
- **Conditional interpretation:** behavior requiring stated representation, arithmetic, or execution assumptions.
- **documented_only:** behavior that the available evidence cannot resolve.

The memory-oriented terminology below describes observable relationships, not application data structures or ownership.

## Interface

### Explicit inputs

| Input | Width or representation | Established role |
|---|---|---|
| First | Opaque 32-bit quantity | No explicit use in the body. |
| Second | Pointer to an unsigned 32-bit word | Destination for the original third input. |
| Third | Unsigned 32-bit quantity | Initializes address calculations and retained state. |
| Fourth | Opaque 32-bit quantity | No explicit use in the body. |
| Fifth | Signed 32-bit quantity | Contributes to an entry-time address and stored value. |

The body also reads three quantities without defining their initial values: a word pointer, a signed 32-bit address quantity, and an unsigned 32-bit mask. Their origin, initialization, and calling-context interpretation are **documented_only**. Describing their uses does not establish that a valid invocation supplies them.

The first and fourth explicit inputs have no demonstrated direct effect. Their significance outside the displayed body is unknown.

### Results and execution requirements

There is no ordinary returned result. Effects are exposed through memory, dependency calls, global state, and breakpoint handling.

Each reached access requires a suitable accessible destination or source. The body provides no explicit validation, error reporting, rollback, or recovery. An access fault, a dependency that does not return, or breakpoint intervention can prevent all later events.

Address zero is reachable by explicit computation and assignment. A contract that simply forbids zero addresses would exclude displayed behavior rather than explain it.

No visible allocation, release, ownership transfer, or lifetime extension occurs. A pointer obtained from a dependency is not thereby proven valid, writable, allocated, or owned.

### Width and representation boundaries

Word reads and writes are four bytes; narrow writes are one or two bytes. One result is written as eight bytes. A separate write and read have pointer width.

Unsigned 32-bit arithmetic retains the low 32 bits. This rule does not resolve signed overflow, invalid signed shifts, pointer arithmetic outside valid objects, or implementation-dependent conversions.

The numeric address examples use a **conditional 32-bit pointer model**. Under that model:

- A pointer-width access occupies four bytes.
- Converting a 64-bit result directly to a pointer selects its low 32 bits.
- The low and high halves of a consumed 64-bit result are each 32 bits.

The displayed integer widths support these calculations, but an independent ABI and pointer-representation specification was not supplied.

## Behavior

### Entry transaction and initial state

Let the **entry value** mean the original third input. It remains a captured value even if later writes alter its output destination.

Three four-byte writes occur before the header accesses, in this order:

| Destination relationship | Stored value |
|---|---|
| Incoming signed address quantity plus 44 bytes | That incoming signed quantity |
| Twice an entry sum | The entry sum |
| Second explicit input | Entry value |

The entry sum combines the initial word pointer’s signed numeric conversion, the fifth input, and eight. Both that calculation and its doubling use signed arithmetic. The first destination calculation also uses signed addition. Overflow and conversion boundaries are **documented_only**.

The **initial header address** is the entry value multiplied by \(2^{24}\), reduced to 32 bits. Before the first header transaction, a byte at that address plus **20 bytes** receives the low byte of eight times the entry value minus two.

The following retained quantities define subsequent behavior:

| Contract term | Initial value |
|---|---|
| Working value | Eight times the entry value minus two, modulo \(2^{32}\) |
| Shared value | Entry value multiplied by \(2^{25}\), modulo \(2^{32}\) |
| Header value | Entry value shifted left by 28, retaining 32 bits |
| Accumulator | Entry value bitwise AND the incoming mask |
| Primary header address | Initial header address |
| Secondary header address | Initial header address |
| Sampling pointer | Initial word pointer |
| Recurring dependency | F-opaque-01-A |

These terms name retained values, not an externally visible state record.

### Header transaction

A header visit has four ordered write events. At the primary header address, it writes a one-byte zero, then the working value narrowed to two bytes, then the four-byte header value. The accumulator then increases by the shared value using unsigned 32-bit arithmetic, and the updated accumulator is written as four bytes at the secondary header address.

On the first visit, the primary and secondary addresses coincide. The overlapping writes remain distinct events.

Later header visits do not repeat the entry transaction. Their primary address is zero; their secondary address comes from F-opaque-07. Every completed header visit establishes zero control for the next recurring call, without restoring the original recurring dependency.

### Sampling transaction

Each cycle captures the low byte of the current shared value before invoking the selected recurring dependency. This **captured byte** survives subsequent changes to the shared value and the global byte.

The recurring dependency receives the current signed 32-bit control value. Its result is consumed as 64 bits. Before another dependency is called, the following memory events occur in order:

| Event | Required relationship |
|---|---|
| Two-byte write | At the recurring result’s low-half address, storing its low 16 bits |
| Four-byte write | At 64 times the working value, storing that same unsigned 32-bit product |
| Four-byte read | Through the existing sampling pointer, replacing the shared value |
| Two-byte write | At the working-value address plus four bytes, using the existing pointer’s short-width conversion plus four |
| Pointer-width read | At the signed interpretation of the result’s high half plus 80 bytes, replacing the sampling pointer |

The sample and pointer-derived two-byte write both use the **old sampling pointer**. The replacement pointer is acquired only afterward.

Under ordinary low-bit narrowing, the fourth event stores the low 16 bits of the old pointer’s numeric value plus four. The actual pointer-to-short conversion remains **documented_only**.

The replacement-pointer location involves a signed conversion and signed addition. Its behavior at conversion or overflow boundaries is unresolved. The working-value products and unsigned address additions instead have the displayed unsigned 32-bit reduction.

### Dispatch and continuation decision

After pointer replacement, the dependency order is:

1. F-opaque-02, with no displayed explicit argument.
2. F-opaque-03, with no displayed explicit argument.
3. F-opaque-04, receiving the sampled unsigned word.
4. F-opaque-05, receiving that word logically shifted right by three.

F-opaque-01-B becomes the recurring dependency after F-opaque-04 returns and before F-opaque-05 is invoked.

F-opaque-05 supplies two distinct quantities. Its high result half becomes the **candidate address**. Its low result is converted to signed 32-bit form and multiplied by 32; the result is retained as an unsigned 32-bit **threshold**. Signed conversion and multiplication must not be silently replaced with guaranteed wrapping arithmetic.

A four-byte write of the threshold at its own numeric address occurs **before** the threshold comparison.

- **Threshold below 20:** another sampling cycle follows without a header. Control is reset to zero, F-opaque-01-B remains selected, and the sampled shared value and replacement pointer remain current. The working value is unchanged.
- **Threshold at least 20:** execution enters the extended transaction described below.

For representable signed products, the threshold is a multiple of 32. Zero is the only such unsigned value below 20. This does not establish behavior for overflowing signed multiplication.

### Extended transaction and snapshot publication

The extended path first writes the working value as four bytes at the threshold address plus 88 bytes. It then writes the replacement pointer’s short-width numeric conversion as two bytes at the working-value address plus 14 bytes.

The shared value becomes the working value shifted left by two using unsigned 32-bit arithmetic. A visible global byte then receives the captured cycle-entry byte, before F-opaque-06 is invoked.

F-opaque-06 receives the replacement pointer’s unsigned 32-bit numeric conversion logically shifted right by two. On its return, F-opaque-01-C becomes the recurring dependency, and F-opaque-07 is called.

The global assignment does not publish the newly sampled word or the newly shifted shared value. It publishes the earlier snapshot. Subsequent dependencies may change that global; no preservation guarantee is available.

### Result-directed storage

F-opaque-07’s low result portion supplies a **storage base**; its high half becomes the secondary header address.

Four ordered writes precede the candidate-address footprint:

| Destination | Width | Value |
|---|---:|---|
| Twice the signed numeric storage base | 1 byte | Captured cycle-entry byte |
| Storage base plus eight bytes | 1 byte | Low byte of F-opaque-07’s high result half |
| Twice the existing accumulator, reduced to 32 bits | Pointer width | Candidate address |
| Storage base | 8 bytes | Entire F-opaque-07 result |

The first destination uses signed multiplication. The second displacement is eight bytes because the base is treated as pointing to eight-byte storage. Neither relationship proves valid object bounds.

After these writes, the retained accumulator is derived from the candidate address’s signed numeric conversion shifted left by two. The working value is derived from the replacement pointer’s signed numeric conversion shifted left by eight.

The candidate address then receives three adjacent four-byte writes, in increasing address order:

| Offset | Value |
|---:|---|
| 0 bytes | Shared value established before F-opaque-06 |
| 4 bytes | Newly derived working value |
| 8 bytes | Replacement pointer’s unsigned 32-bit numeric conversion |

The second stored value depends on a **signed** left shift. It is not unconditionally equivalent to a defined unsigned shift.

These accesses establish a 12-byte footprint, not a record schema or allocation contract. Afterward, the header value becomes the shared value multiplied by 256 using unsigned 32-bit arithmetic.

### Final branch and breakpoint continuation

The final condition depends on the replacement pointer’s signed numeric conversion, a signed left shift by 31, a signed right shift by 32, conversion to unsigned 32-bit form, and a logical right shift by 20.

For a 32-bit signed operand, the right-shift count of 32 is outside ordinary valid bounds. The preceding left shift is also not valid for every input. No portable condition value, parity test, or equivalent pointer-bit predicate is established.

The primary header address is set to zero before the condition is tested. Conditional on an executable interpretation:

| Condition | Required continuation |
|---|---|
| Zero | Revisit the header, retaining F-opaque-01-C. The first attempted header access is a one-byte zero write at address zero. |
| Nonzero | Write the condition as four bytes at the header-value address plus 32 bytes; select F-opaque-01-D; derive control from the signed header value shifted right by 29; invoke breakpoint 4. |

If execution resumes after the breakpoint, the working value becomes the signed secondary-header-address conversion shifted right by 18, retained as an unsigned word. The next cycle has no intervening header and no zero-control reset.

Selecting a dependency does not guarantee that it will be called: intervening accesses, invalid arithmetic interpretation, or interruption may prevent that call.

## Side effects

Writes extend far beyond the explicit output pointer. Destinations derive from incoming state, arithmetic values, dependency results, and zero.

Aliasing affects observations. Entry writes, header writes, dependency side effects, and the first two post-recurring-call writes can change the word eventually sampled. The relevant value is the word present at the specified read boundary, not its earlier contents.

The captured byte is independent of later global contents. Its result-directed write still uses the captured value even if F-opaque-06 or F-opaque-07 changes the global after publication.

No transaction is atomic or reversible in the supplied evidence. Partial completion can leave externally visible writes behind. Access atomicity, byte order, alignment support, volatile behavior, and device-memory semantics are **documented_only**.

Unknown dependencies may mutate memory, alter globals, allocate, release, or invalidate storage. None is assumed to preserve globals or provide ownership.

## Dependencies

Opaque IDs distinguish call roles without assigning application meaning.

| ID | Established interaction |
|---|---|
| F-opaque-01-A | Initial recurring dependency |
| F-opaque-01-B | Selected before F-opaque-05; retained across short continuation |
| F-opaque-01-C | Selected after F-opaque-06; retained across a zero-condition header revisit |
| F-opaque-01-D | Selected before breakpoint 4 |
| F-opaque-02 | No displayed explicit argument; result not explicitly consumed |
| F-opaque-03 | No displayed explicit argument; result not explicitly consumed |
| F-opaque-04 | Receives the sampled unsigned 32-bit word |
| F-opaque-05 | Receives the sample shifted right by three; supplies a consumed 64-bit result |
| F-opaque-06 | Receives the replacement pointer’s unsigned numeric conversion shifted right by two |
| F-opaque-07 | No displayed explicit argument; supplies a consumed 64-bit result |

Every recurring dependency receives the current signed 32-bit control and supplies a result consumed as 64 bits.

“No displayed explicit argument” does not exclude implicit inputs. An unused result does not establish absence of side effects. Dependency termination, memory effects, ownership, and global preservation remain **documented_only**.

## Acceptance examples

These are conditional criteria, not executed tests. Each assumes that preceding accesses complete and the necessary dependencies return.

### Entry displacement and overlapping accesses

For entry value 3, the output word receives 3, the initial header address is \(0x03000000\), and the working value is 22.

The entry byte write targets \(0x03000014\) and stores 22. A write at \(0x03000005\) would not satisfy the contract. The following header transaction requires distinct one-, two-, four-, and four-byte writes even where their destinations coincide.

### Sampling before replacement

Suppose the old sampling pointer yields 40 at the actual sampling boundary, while the replacement pointer refers to a word containing 99. F-opaque-04 receives 40, and F-opaque-05 receives 5.

The replacement location’s contents do not determine that cycle’s sample. Earlier aliasing is allowed; 40 is specified at the read itself.

### Threshold storage precedes continuation

A zero low result from F-opaque-05 produces a threshold of zero and requires a four-byte zero write at address zero before continuation is decided. If that write fails, no next cycle is required.

If it completes, the next cycle uses F-opaque-01-B with zero control and no header. If the retained sampled shared value is 40, the next captured byte is 40.

A low result of 1 produces threshold 32 without signed multiplication overflow and takes the extended path.

### Snapshot and storage displacement

If the shared value’s low byte at cycle entry is \(0xAB\), the extended-path global assignment writes \(0xAB\), regardless of the subsequent sample.

For storage base \(0x2000\), the first two result-directed byte destinations are \(0x4000\) and \(0x2008\). The first still receives \(0xAB\) if an intervening dependency changed the global.

### Candidate footprint

With candidate address \(0x1000\), replacement pointer \(0x4000\), and shared value 88, the relevant signed shifts are representable. The ordered four-byte candidate writes are:

- 88 at \(0x1000\);
- \(0x00400000\) at \(0x1004\);
- \(0x00004000\) at \(0x1008\).

The replacement pointer supplies values; it is not the destination of these three writes.

### Conditional final outcomes

An execution interpretation yielding zero requires the next header’s one-byte write at address zero, retaining F-opaque-01-C.

An interpretation yielding nonzero requires the condition write before breakpoint 4. F-opaque-01-D receives the derived control only if execution resumes and reaches its call.

Neither criterion gives portable meaning to the invalid shift count.

## Uncertainties

The following remain **documented_only**:

- Origin and validity of the three read-before-defined quantities.
- Actual ABI, pointer width, pointer representation, and dependency calling conventions.
- Pointer-to-integer conversions, out-of-range signed conversions, and short-width conversions.
- Signed overflow, signed-left-shift validity, and negative signed-right-shift behavior.
- Executable interpretation of the final shift by 32.
- Memory mapping, object bounds, alignment, endianness, and access atomicity.
- Dependency termination, mutation, ownership, and allocation or release behavior.
- Breakpoint handling and whether execution resumes.
- Whether the unusual displayed control flow faithfully represents executable behavior.
- Application purpose and any successful completion criterion.

**Inferred intent only:** the adjacent candidate writes resemble structured storage. That resemblance establishes no schema, ownership, lifetime, or release obligation.

## Evidence

**Established static evidence:** the supplied body supports the explicit input uses, retained-state relationships, access widths and ordering, dependency sequence and selection changes, old-pointer sampling boundary, captured-byte behavior, global assignment, threshold-address write, candidate footprint, and breakpoint immediate described above.

**Conditional evidence:** numeric address examples require the stated pointer model and valid arithmetic interpretations. Unsigned low-bit calculations do not resolve signed overflow, pointer validity, or invalid shifts.

**Unavailable evidence:** no independent ABI, dependency contracts, memory map, breakpoint policy, or executable observations were provided.

No runtime tests have been performed.
