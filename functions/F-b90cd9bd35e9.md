<!-- Trusted provenance: F-b90cd9bd35e9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b90cd9bd35e9

## Purpose

F-b90cd9bd35e9 conditionally prepares a four-word working record associated with an object, invokes a gated sequence of dependencies, and returns the record’s first two words as a 64-bit value.

The normal path requires the object’s 32-bit field at byte offset 0xA8 to be zero. A nonzero value selects a separate path that calls two dependencies and has no established normal return.

The routine also performs two optional virtual calls before preparing the working record. A later processing sequence depends on a dependency result, an object field’s low eight bits, and an externally referenced byte.

These statements describe the supplied static evidence. The application-level meaning of the object, fields, flags, and processing operations is not established.

## Interface

### Inputs

The visible interface has four arguments:

| Argument | Visible representation | Established use |
|---|---|---|
| Object | Pointer to storage addressed in 32-bit integer units | Supplies fields, receives writes, participates in virtual dispatch, and is passed to dependencies. |
| Second argument | 32-bit opaque value | No explicit use in the supplied body. |
| Third argument | 32-bit opaque value | Initially stored in the third word of the working record. |
| Fourth argument | 32-bit opaque value | Initially stored in the fourth word of the working record. |

No explicit null-pointer, bounds, alignment, object-type, or lifetime checks appear.

The object must support the accesses actually reached. The normal path directly accesses storage through offset 0xA8 and constructs an address at offset 0x80. Optional virtual dispatch additionally requires a usable dispatch-table reference in the object’s first word. Dependencies may require more storage or stronger invariants; their requirements are unknown.

### Object layout used directly

Offsets are byte offsets from the object address.

| Offset | Width or interpretation | Use |
|---|---|---|
| 0x00 | Dispatch-table reference in the supplied representation | Used for optional indirect calls. |
| 0x34 | 32-bit value, interpreted as an address-like value when nonzero | Captured before F-e57b05520b65 and potentially adjusted by 0xD0 for F-ee8733a08210. |
| 0x48 | 32-bit flag word | Controls the two optional virtual calls. |
| 0x80 | Interior object address | Placed in the working record and passed twice to F-e57b05520b65. |
| 0xA0 | 32-bit field, narrowed to eight bits for the condition | Must have zero low eight bits to permit later processing. |
| 0xA4 | 32-bit field | Written with zero on the normal path. |
| 0xA8 | 32-bit field | Initial branch discriminator; written with zero on the normal path. |

The storage at 0x80 is not otherwise characterized. Taking its address does not establish its type, capacity, ownership, or contents.

### Working record and result

The routine constructs an automatic four-word record with this initial layout:

| Record offset | Initial 32-bit contents |
|---|---|
| 0x00 | Object address |
| 0x04 | Object address plus 0x80 |
| 0x08 | Third argument |
| 0x0C | Fourth argument |

The supplied representation uses 32-bit address-bearing words. It does not establish a portable interface for wider pointers.

On normal completion, the 64-bit result is formed from the record’s first two words **as they exist at return**:

- Bits 31–0 come from the word at record offset 0x00.
- Bits 63–32 come from the word at record offset 0x04.

This is a bitwise composition, not a numerical sum of the two words and not an established ownership transfer. Dependencies receiving the record’s address may change the returned words.

## Behavior

### 1. Entry discriminator

The first established object access reads the 32-bit field at offset 0xA8.

If that value is nonzero:

1. F-282eebd7c413 is called with no explicit arguments.
2. F-feee9156a257 is then called with no explicit arguments, provided the first call returns.

The supplied body marks F-feee9156a257 as non-returning. This path does not reach record construction or the normal result assembly. It also does not perform the normal-path zero writes before calling F-282eebd7c413.

If the initial value at offset 0xA8 is zero, the routine continues on the normal path. It explicitly writes zero to offset 0xA8, even though the preceding read found zero, and then writes zero to offset 0xA4. The order of these two writes is established.

There is no subsequent explicit revalidation of offset 0xA8. A dependency changing it later does not, by itself, cause the entry decision to be repeated.

### 2. Optional virtual calls

After the zero writes, the routine examines the 32-bit flags at offset 0x48.

The optional virtual-call sequence is selected when both conditions hold:

- Masking with 0x1800 produces exactly 0x0800.
- Bit 0x0010 is set.

Equivalently, the first condition requires bit 0x0800 to be set and bit 0x1000 to be clear. Other bits do not participate in this decision.

The conditions are short-circuited: the bit-0x0010 condition is considered only when the mask comparison succeeds.

When selected, two indirect calls occur in this order:

1. The dispatch-table entry at byte offset 0x11C, with the object as its explicit argument.
2. The dispatch-table entry at byte offset 0x114, again with the object as its explicit argument.

The second dispatch expression uses the object’s dispatch-table reference after the first call. The contract must therefore allow the first call to change which table or target is used by the second call. It must not assume that either virtual call preserves object fields or other accessible state.

If either call fails to return normally, later processing is not reached.

### 3. Record preparation and initial dependency

After the optional virtual calls return, or after they are skipped, the four-word record is initialized in the order shown in the interface: object address, object-plus-0x80 address, third argument, fourth argument.

F-3fac671648b8 is then called with no explicit arguments. Its result is tested as a 32-bit zero/nonzero value.

The absence of explicit arguments is significant. The supplied evidence does not establish that this dependency formally receives the record, the object, or the unused second argument. It also does not justify assuming that the dependency preserves global or object state.

### 4. Three-stage processing gate

The later processing sequence is entered only if all of the following succeed in order:

1. F-3fac671648b8 returned nonzero.
2. The low eight bits of the object’s 32-bit field at offset 0xA0 are zero.
3. A byte reached through an external pointer is nonzero.

The object field is read after F-3fac671648b8 returns. Its higher 24 bits do not affect this gate. For example, a 32-bit value of 0x00000100 passes the low-eight-bit test.

The external byte is read only after the first two conditions succeed. If the dependency result is zero, neither the offset-0xA0 test nor the external-byte test is reached. If the low-eight-bit test fails, the external-byte test is not reached.

The gate uses the state available at these later reads, not necessarily the object or external state present on entry.

### 5. Gated processing sequence

When all three gate conditions succeed, the following order is established:

1. Capture the object’s 32-bit value at offset 0x34.
2. Call F-e57b05520b65 with:
   - The working record’s address.
   - The object-plus-0x80 address.
   - The same object-plus-0x80 address again.
3. If the captured value from step 1 was nonzero, call F-ee8733a08210 with:
   - The working record’s address.
   - The captured value adjusted by 0xD0.
4. Call F-ffe0d92c57b7 with the object.
5. If its 32-bit result is nonzero, call F-c9b5d17d8d1e with that result and the working record’s address.
6. After that call returns, call F-4d037ecf0f82 with the same saved result and the working record’s address.

The offset-0x34 value is captured before F-e57b05520b65. Consequently, a change to that object field during F-e57b05520b65 does not change whether F-ee8733a08210 is selected or which captured value supplies its adjusted argument.

Likewise, the result from F-ffe0d92c57b7 is saved for both subsequent calls. There is no second invocation of F-ffe0d92c57b7 between them.

The two object-plus-0x80 arguments to F-e57b05520b65 alias exactly. A behavioral description that treats them as distinct buffers would contradict the visible call.

### 6. Normal completion

Whether gated processing runs or is skipped, normal completion returns the current first two record words in the 64-bit arrangement described above.

There is no explicit final cleanup call, record validation, restoration of object fields, or conversion of dependency results into a separate status code.

## Side effects

The routine’s unconditional direct writes on the normal path are confined to:

1. Zero at object offset 0xA8.
2. Zero at object offset 0xA4.
3. Initialization of the automatic working record.

Additional object, global, external-memory, or resource effects may occur through dependencies and virtual calls. Their extent is not established here.

The record’s address is passed to several dependencies. This establishes access during those calls, not a right to retain the address after this routine returns. Because the record is automatic storage, retaining and later using its address would require an external lifetime arrangement not shown in the evidence.

No explicit allocation, release, reference-count adjustment, or ownership transfer appears in this body. Those operations must not be inferred absent merely because they are not visible: callees may perform them. In particular, the nonzero value at offset 0x34 and the nonzero result from F-ffe0d92c57b7 have no established ownership semantics.

There is no visible synchronization. Concurrency guarantees, atomicity, and validity under concurrent mutation are documented_only unknowns.

## Dependencies

| Opaque function ID | Visible invocation and role |
|---|---|
| F-282eebd7c413 | Called with no explicit arguments on the nonzero entry-discriminator path. Return value, if any, is ignored. |
| F-feee9156a257 | Called next on that path; marked non-returning in the supplied evidence. |
| F-3fac671648b8 | Called with no explicit arguments after record initialization. Its zero/nonzero result controls access to the remaining gate. |
| F-e57b05520b65 | Receives the record address and two identical object-interior addresses. Called once when the full gate passes. |
| F-ee8733a08210 | Receives the record address and the saved offset-0x34 value adjusted by 0xD0, only when that saved value is nonzero. |
| F-ffe0d92c57b7 | Receives the object. Its zero/nonzero result controls the final pair of calls. |
| F-c9b5d17d8d1e | Receives the saved nonzero result and record address; precedes F-4d037ecf0f82. |
| F-4d037ecf0f82 | Receives the same saved result and record address after F-c9b5d17d8d1e returns. |

Two additional dependencies are selected dynamically from dispatch-table byte offsets 0x11C and 0x114. Their opaque function IDs are not supplied. Their signatures beyond the explicit object argument, return behavior, and effects are unknown.

Dependency return values are ignored except for the two zero/nonzero decisions expressly identified above. No preservation of globals, memory, or ownership is assumed.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

1. **Nonzero entry discriminator.**  
   Given a nonzero field at offset 0xA8, the first dependency call is F-282eebd7c413. If it returns, F-feee9156a257 follows. The normal-path zero writes, optional virtual calls, and record preparation are not reached.

2. **Optional virtual-call selection.**  
   Given an initially zero discriminator and flags 0x0810, the routine writes the two zero fields and then invokes dispatch offset 0x11C before offset 0x114. Flags 0x1810 do not select these calls because the 0x1000 bit defeats the mask comparison. Flags 0x0800 also do not select them because bit 0x0010 is absent.

3. **Second dispatch observes changed dispatch state.**  
   If the first virtual call replaces the object’s dispatch-table reference and returns, the second call uses the reference obtained afterward. It is not required to use the original table.

4. **Early gate rejection.**  
   If F-3fac671648b8 returns zero, no later direct gate read or gated-processing call occurs. Normal completion still assembles its result from the current first two record words.

5. **Eight-bit rather than whole-word gate.**  
   If F-3fac671648b8 returns nonzero and offset 0xA0 contains 0x00000100, the object-side gate passes. If that field instead contains 0x00000101, it fails and the external byte is not read.

6. **Saved field controls the adjusted call.**  
   Suppose all gates pass and the captured offset-0x34 value is 0x00002000. Even if F-e57b05520b65 changes that object field to zero, F-ee8733a08210 remains selected and receives the adjusted value 0x000020D0. Conversely, a captured zero skips that call even if the field becomes nonzero afterward.

7. **Final pair shares one result.**  
   If F-ffe0d92c57b7 returns a nonzero value, F-c9b5d17d8d1e and then F-4d037ecf0f82 receive that same saved value. If it returns zero, both calls are skipped.

8. **Return reflects record mutation.**  
   If normal completion is reached with record word zero equal to 0x11223344 and record word one equal to 0x55667788, the returned bit pattern is 0x5566778811223344. The initial pointer values do not override intervening record changes.

## Uncertainties

- **documented_only — Application intent.** The field meanings and semantic purpose of the preparation and processing sequence are unknown. A lifecycle or resource-management interpretation would be inferred intent, not an established contract.
- **documented_only — Calling conventions.** Calls shown without explicit arguments may have contextual requirements not represented in the supplied body. No hidden parameters are asserted.
- **documented_only — Arithmetic boundaries.** The address-like adjustment uses a 32-bit captured value and a displacement of 0xD0. No overflow check is visible. Defined behavior for values near the 32-bit boundary, signed-overflow interpretation, and address validity are not established. The acceptance example deliberately avoids those boundaries.
- **documented_only — External byte.** Its location, initialization, lifetime, mutability, and semantic meaning are not supplied. Only its gated nonzero test is established.
- **documented_only — Non-returning path.** The final dependency is marked non-returning, but termination, exception, transfer of control, and other mechanisms are not distinguished.
- **documented_only — Dependency effects.** Memory retention, mutation, allocation, release, exceptions, reentrancy, and global-state changes remain unknown unless directly visible at the call boundary.
- **documented_only — Physical memory accesses.** The ordering here describes the supplied representation’s evaluation and call order. It is not a claim about individual machine-load instructions, volatile behavior, or a memory model.
- **documented_only — Endianness.** The return’s high/low bit arrangement is established; its byte layout in memory is not.

## Evidence

The supplied static body establishes the field offsets and widths, entry branch, explicit zero writes, flag conditions, virtual-dispatch order, four-word record initialization, short-circuit gates, dependency call order, captured-value reuse, and 64-bit result composition.

The dependency map supplies only the opaque function identities used above. It supplies no dependency implementations or preservation guarantees.

No runtime tests have been performed. The acceptance examples are conditional review criteria, not observed executions.
