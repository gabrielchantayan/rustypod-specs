<!-- Trusted provenance: F-55dea9620dee; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-55dea9620dee

## Purpose

F-55dea9620dee conditionally delegates an operation based on a context-held handle, two lookups, and, when the first lookup succeeds, an indirect readiness result and a signed comparison of two object fields.

**Established behavior:** the function returns a 32-bit result. Its default result is `0xFFFFFFFF`. It replaces that result only with the result of one of two downstream operations:

- F-eadf6abacb1d, when the first lookup returns zero and the second lookup returns nonzero.
- F-b3f4a7a52609, when the first lookup returns an object, an indirect call returns a nonzero low 32-bit word, and the signed field at object offset `0x104` is greater than the signed field at offset `0x100`.

**Inferred intent:** the object fields may delimit an available range, and the indirect call may indicate readiness. These interpretations are not established. The contract depends on the observed words, their signed ordering, and the call sequence—not on a presumed meaning such as length, capacity, or resource availability.

## Interface

### Input

The function accepts one argument represented as a 32-bit address of a context.

The context must support a 32-bit read at byte offset `0x38`. That word is used as a handle or lookup receiver. The function does not establish what the word identifies.

The context address itself is not checked for zero or validity before the read. A zero word at offset `0x38` is handled; an invalid context address is not converted into a defined error result by this function.

### Output

The result is a 32-bit word:

| Condition | Result |
|---|---|
| Initial context word is zero | `0xFFFFFFFF` |
| First lookup is zero and second lookup is zero | `0xFFFFFFFF` |
| First lookup is zero and second lookup is nonzero | Result of F-eadf6abacb1d |
| First lookup is nonzero but the indirect result has a zero low word | `0xFFFFFFFF` |
| First lookup is nonzero, indirect low word is nonzero, and the field comparison fails | `0xFFFFFFFF` |
| First lookup is nonzero, indirect low word is nonzero, and the field comparison succeeds | Result of F-b3f4a7a52609 |

The bit pattern `0xFFFFFFFF` is not independently established as an error code. Nor is it established that either downstream result excludes that same value. A caller cannot distinguish a default return from an identical delegated result solely by inspecting the returned word.

### Reachable object layout

A nonzero first-lookup result is treated as an object address with the following relevant layout:

| Byte offset | Access | Established use |
|---|---|---|
| `0x000` | Address-sized table reference | Selects an indirect call table |
| `0x100` | 32-bit signed word | Lower operand in the final signed ordering test |
| `0x104` | 32-bit signed word | Upper operand in the final signed ordering test |

The indirect call target is obtained from byte offset `0x188` in the referenced table. The object address is passed as the call’s explicit argument.

These offsets identify only the portions accessed here. They do not establish the complete object size, type, ownership model, or other fields.

## Behavior

### Initial gate

The function first reads the 32-bit context word at offset `0x38`.

If this read produces zero, the function returns `0xFFFFFFFF`. It performs neither lookup and makes no further calls.

A nonzero result enables the lookup sequence. The initially read value is not established as a cached handle for subsequent calls.

### Ordered lookups

Two calls to F-682adfd5a561 occur in a fixed order:

1. The first call receives a fresh read of the context word at offset `0x38` and the selector `0x1700`.
2. After the first call returns, the second call receives another read of the context word at offset `0x38` and a 32-bit value obtained from a fixed data location.

The numeric value and meaning of the second selector are not supplied.

The second lookup is performed regardless of whether the first lookup returns zero. It is also performed before any indirect call on an object returned by the first lookup.

The handle is therefore read three times on this path: once for the initial gate, once for the first lookup, and once for the second lookup. A dependency may change the context-held handle between these reads. The implementation does not reapply the initial nonzero gate before each lookup.

The relative evaluation order of the second lookup’s handle read and fixed-data selector read is not established. Both supply that call’s arguments.

### First lookup returns zero

When the first lookup returns zero, the second lookup result determines the remaining action:

- If the second result is zero, the function returns `0xFFFFFFFF`.
- If the second result is nonzero, the function calls F-eadf6abacb1d with no explicit arguments and returns its 32-bit result.

The second result is tested for zero only. No object access through it is shown.

F-eadf6abacb1d is called only after both lookups have completed. Any global or other implicit state it observes may therefore include effects of both lookups.

### First lookup returns an object

When the first lookup returns a nonzero object address, the second lookup’s return value does not control the branch. Its call still occurs, and its side effects are not dispensable merely because its returned word is unused on this path.

After the second lookup completes, the function obtains the indirect target through the object’s table reference and table offset `0x188`. It invokes that target with the original object address.

The indirect result is represented as 64 bits. Only its low 32-bit word affects subsequent behavior:

- A zero low word prevents both field reads and the call to F-b3f4a7a52609.
- A nonzero low word enables the field reads and comparison.
- The upper 32-bit word does not affect the final branch or result.

A nonzero upper word with a zero lower word therefore does not enable the operation.

### Field reads and signed comparison

When the indirect result’s low word is nonzero, the function reads:

1. The 32-bit word at object offset `0x104`.
2. The 32-bit word at object offset `0x100`.

The comparison uses these captured values. It does not reread the fields when deciding whether to call F-b3f4a7a52609.

The acceptance condition is exactly:

**The signed 32-bit value from offset `0x104` is strictly greater than the signed 32-bit value from offset `0x100`.**

Equality fails the condition.

The observed subtraction and overflow test must be interpreted together. The condition is not equivalent to testing whether a wrapped 32-bit difference is positive. For operands far apart across the signed range, the wrapped difference can have the opposite sign from the mathematical difference. The signed-overflow check compensates for that case.

Consequently:

- Negative values are not categorically rejected.
- A negative value at `0x104` may pass when the value at `0x100` is more negative.
- Unsigned ordering is not the governing rule.
- No nonnegative length, bounded distance, or pointer-range invariant is checked.

If the signed ordering succeeds, the function calls F-b3f4a7a52609 with the original object address and returns that dependency’s 32-bit result. Otherwise, it returns `0xFFFFFFFF`.

There is no fallback to F-eadf6abacb1d after an object has been obtained, even if readiness or the field comparison fails.

## Side effects

### Direct accesses

The function directly performs reads but shows no direct writes to the context, returned object, fixed selector storage, or other externally visible data.

On a nonzero initial gate, relevant direct accesses occur in this order:

1. Context handle read for the first lookup.
2. First lookup.
3. Context handle and fixed-data selector reads for the second lookup.
4. Second lookup.
5. If the first result is nonzero, indirect-target lookup through that object.
6. Indirect call.
7. If the indirect low result word is nonzero, object field read at `0x104`, followed by the read at `0x100`.
8. If the comparison succeeds, F-b3f4a7a52609.

The initial gating read precedes this sequence.

### Dependency effects

All invoked dependencies may have side effects. Nothing supplied establishes that they preserve the context, object, lookup storage, fixed selector value, globals, or external resources.

In particular:

- The first lookup can affect the handle subsequently read for the second lookup.
- The second lookup can affect the object’s table or fields before they are accessed.
- The indirect callee can affect the fields before they are read.
- Neither terminal dependency is known to preserve memory or globals.

These are absence-of-guarantee statements, not claims that the dependencies actually perform such mutations.

### Ownership and lifetime

No explicit allocation, retention, release, or ownership transfer is visible in this function.

A nonzero lookup result is used without an accompanying validity or lifetime check. Successful completion of the object path requires that the returned object, its table, and the selected target remain usable when accessed. After the indirect call returns a nonzero low word, both field locations must remain readable.

The evidence does not establish whether dependencies acquire, transfer, or release ownership internally. In particular, the absence of an explicit release here does not prove that the object remains owned by the caller or by the lookup provider.

### Errors and abnormal completion

No local recovery, retry, exception handling, or validation of dependency-produced addresses is shown.

Invalid memory, an unusable indirect target, or a dependency that does not return may prevent normal completion. Those outcomes are not shown being converted into `0xFFFFFFFF`.

## Dependencies

| Opaque identity | Invocation and role | Limits of established knowledge |
|---|---|---|
| F-682adfd5a561 | Called twice after a nonzero initial gate. Receives a freshly read context handle and a selector each time. | First result is treated as an object-or-zero value; second as zero/nonzero when the first is zero. Lookup semantics and side effects are unknown. |
| F-eadf6abacb1d | Called with no explicit arguments when the first lookup is zero and the second is nonzero. | Its 32-bit result is returned unchanged. Meaning, ownership behavior, and implicit inputs are unknown. |
| F-b3f4a7a52609 | Called with the first lookup’s object after a nonzero indirect low word and successful signed comparison. | Its 32-bit result is returned unchanged. The operation performed is unknown. |

An additional dependency is selected indirectly through the returned object’s table at byte offset `0x188`. Its opaque function identity is unavailable. The established interface is an object argument and a result represented as 64 bits, of which only the low 32 bits are tested. Its broader calling convention and semantics are documented_only unknowns.

## Acceptance examples

These are conditional contract examples, not runtime observations. Each assumes valid required memory and normally returning dependencies.

1. **Initial handle is zero.**  
   The result is `0xFFFFFFFF`. No dependency is invoked.

2. **Both lookups return zero.**  
   With a nonzero initial gate, both lookups execute in order. Neither terminal dependency nor the indirect target is called. The result is `0xFFFFFFFF`.

3. **Only the second lookup succeeds.**  
   If the first lookup returns zero and the second returns any nonzero word, F-eadf6abacb1d is called. If it returns `0x12345678`, that exact word is returned.

4. **Object exists but the indirect low word is zero.**  
   An indirect result with upper word `1` and lower word `0` fails the gate. Neither object field is read by this function, F-b3f4a7a52609 is not called, and the result is `0xFFFFFFFF`.

5. **Equal fields.**  
   If the indirect low word is nonzero and both captured fields equal `7`, the comparison fails. The result is `0xFFFFFFFF`.

6. **Ordered negative fields.**  
   If offset `0x104` contains signed `−2` and offset `0x100` contains signed `−3`, the comparison succeeds. F-b3f4a7a52609 is called and its result is returned.

7. **Signed ordering despite a negative wrapped difference.**  
   If offset `0x104` contains `2,147,483,647` and offset `0x100` contains `−2,147,483,648`, the comparison succeeds. The wrapped 32-bit subtraction result is `0xFFFFFFFF`, but overflow compensation preserves the correct signed ordering.

8. **Reverse extreme ordering.**  
   If those extreme values are reversed, the comparison fails, despite a wrapped subtraction result of `1`.

9. **Second lookup changes the object fields.**  
   If the first lookup returns an object and the second lookup changes its fields, the later reads observe the values present after the second lookup and indirect call. The second lookup’s returned zero/nonzero value does not gate this path.

10. **Handle changes between lookups.**  
    If the first lookup changes the context-held handle, the second lookup receives the subsequently read value, even if it is zero. The initial nonzero gate is not repeated.

## Uncertainties

The following remain documented_only:

- The domain meaning of the context handle, selectors, object fields, and result codes.
- The value and mutability of the second selector’s fixed data location.
- The indirect callee’s opaque identity and full interface.
- Dependency effects on globals, memory, ownership, and resource lifetime.
- Any synchronization or thread-safety guarantees.
- Behavior on invalid addresses, abnormal dependency termination, or calling-convention violations beyond the absence of local recovery.
- Whether the default result denotes failure, absence, ineligibility, or another condition.

No alignment, object-lifetime, or concurrency guarantee should be inferred solely from the displayed accesses.

## Evidence

This contract is based on the supplied static representation of F-55dea9620dee and the supplied opaque dependency mapping.

Established evidence includes the 32-bit return width, default bit pattern, context and object offsets, ordered lookup calls, unconditional second lookup after the initial gate, indirect dispatch location, low-word readiness test, ordered field reads, signed strict comparison, and conditional terminal calls.

The signed comparison conclusion follows from interpreting the 32-bit subtraction’s sign together with signed overflow and the explicit inequality requirement. It does not depend on inferred field meanings.

Readiness, range availability, and lookup-resource interpretations are inferred intent only. No runtime tests have been performed.
