<!-- Trusted provenance: F-9ca5cf36dbfb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9ca5cf36dbfb

## Purpose

F-9ca5cf36dbfb selects a 32-bit result through two ordered dependency-mediated checks and, if neither check selects an early result, a field-based fallback.

The established selection order is:

1. A zero result from the first comparison selects one global value.
2. Otherwise, a zero result from the second comparison selects a different global value.
3. Otherwise, two reads of an input-relative field determine whether to return its initially read value or a third global value.

This is a selection contract, not an established description of the selected values’ meaning. The supplied representation does not establish whether the results are addresses, handles, identifiers, or ordinary integers. It also does not establish that the comparison dependency performs string comparison, despite a named constant used in its first invocation.

## Interface

### Input

The function accepts one argument: an address-like value represented as a 32-bit integer in the supplied source.

For this contract, the argument is called the **input base**. Its relevant offsets are:

| Offset from input base | Use |
|---|---|
| 40 bytes | Address-like argument passed to F-44362605ab79 |
| 44 bytes | Location of a field read twice on the final selection path |

The operation at offset 40 is address formation followed by a dependency call. The visible body does not itself read the contents of that location before the call.

The operation at offset 44 is a 32-bit integer load. The representation does not establish a complete object layout, the size of any member beginning at offset 40, or whether that member overlaps the field at offset 44.

### Output

The function returns a 32-bit value selected from:

- Global slot G-01.
- Global slot G-03.
- The first value read from the field at input base plus 44.
- Global slot G-04.

The dependency results used for comparison are not directly returned.

### Opaque references

The following labels identify data without attributing unestablished semantics:

| Label | Meaning |
|---|---|
| G-01 | Global result slot selected by the first zero comparison result |
| G-02 | Global comparison operand used in the second comparison |
| G-03 | Global result slot selected by the second zero comparison result |
| G-04 | Global fallback result slot selected by a zero second field read |
| K-01 | Fixed comparison operand associated with the text `CntrlHistoryFn` |

These labels identify storage roles, not distinct values. The slots may contain equal values. No relationship among their contents is established.

### Preconditions

The function has path-dependent requirements rather than an unconditional requirement to dereference the input base.

- The initial dependency calls must be valid for execution.
- If the first comparison returns nonzero, forming the input-relative argument and passing it to F-44362605ab79 must be valid under the target address model and that dependency’s contract.
- If both comparisons return nonzero, the location at input base plus 44 must permit the two represented 32-bit reads.
- A selected global result slot must be readable when its branch reaches the corresponding load.
- G-02 must be readable when the second comparison is prepared.

There is no visible input validation. In particular, there is no explicit null, alignment, range, or object-lifetime check.

## Behavior

### First selection stage

F-9ca5cf36dbfb first calls F-3ad65987df9a without explicit arguments.

Its 32-bit result becomes the first operand of comparison dependency D-01. K-01 is the second operand.

If D-01 returns zero, the function reads G-01 and returns that value immediately. On this path:

- F-44362605ab79 is not called.
- G-02 is not used by the visible body.
- The field at input base plus 44 is not read.
- G-03 and G-04 are not selected.

The visible body therefore permits an early result without using the input base. This does not prove that the dependencies are independent of it or of related global state.

If D-01 returns any nonzero result, execution continues. The sign and magnitude of that result do not otherwise affect the selection.

### Second selection stage

The function forms an address-like value 40 bytes beyond the input base and passes it to F-44362605ab79.

After that dependency returns, its 32-bit result becomes the first operand of another D-01 invocation. The current contents of G-02 supply the second operand.

If this comparison returns zero, the function reads G-03 and returns that value immediately. The field at input base plus 44 is not read by the visible body on this path.

If the comparison returns any nonzero result, execution proceeds to the field-based selection.

The two D-01 invocations are separate calls. The second is not a reuse of the first result, and the function does not visibly assume that D-01 is pure or deterministic.

### Field-based selection

After both comparisons have returned nonzero, the supplied representation contains two reads of the same 32-bit field:

1. The first read captures a candidate return value.
2. The second read determines whether the zero fallback applies.

If the second read is zero, the candidate is replaced by the value read from G-04. Otherwise, the function returns the candidate captured by the first read.

This distinction matters if the two reads can observe different values:

- First read nonzero, second read zero: G-04 is selected.
- First read zero, second read nonzero: the returned candidate is zero.
- Both reads nonzero but different: the first value is returned.
- Both reads zero: G-04 is selected.

For a field that remains unchanged across both reads, this reduces to the familiar rule “return the field when nonzero; otherwise return G-04.” That simplified rule is conditional on stability, not an unconditional replacement for the represented behavior.

Whether the two represented reads correspond to two distinct machine-level memory transactions is not established by the supplied material.

### Widths and arithmetic

The supplied representation treats the input argument, return value, field loads, and intermediate dependency values as 32-bit quantities.

The offsets are byte offsets of 40 and 44. They are not element counts or scaled indices.

The representation does not establish a portable overflow rule for adding those offsets to the signed integer input. This contract therefore does not prescribe wraparound for an input near an address-space boundary. Such cases remain outside the established behavior unless the target arithmetic and address model are supplied.

Likewise, the function does not widen the returned field value or visibly reinterpret it as a particular pointer type. The established output is its 32-bit value.

## Side effects

The visible body contains no explicit writes to the input-relative storage or global slots. It also contains no explicit allocation, release, reference-count adjustment, or ownership transfer.

That observation is limited to the body. The called dependencies may read or modify memory, change globals, allocate or release resources, fail, or fail to return. Their contracts are not supplied.

The established sequence of calls and relevant reads is:

1. Call F-3ad65987df9a.
2. Call D-01 with that result and K-01.
3. If the result is zero, read G-01 and return.
4. Otherwise, call F-44362605ab79 with input base plus 40.
5. Obtain G-02 as the second operand of the next D-01 call.
6. Call D-01 with the second dependency result and that operand.
7. If the result is zero, read G-03 and return.
8. Otherwise, perform the two represented field reads.
9. If the second field read is zero, read G-04 and return it; otherwise return the first field value.

Consequently, a selected global is not established as an entry-time snapshot. For example, if a dependency changes G-03 before the second comparison returns zero, the subsequent G-03 load selects its then-current contents.

No ownership conclusion follows from returning a value loaded from a global or field. A caller must not infer that the result is newly allocated, borrowed, retained, or safe to release.

## Dependencies

| Opaque function ID | Established interaction | Unestablished properties |
|---|---|---|
| F-3ad65987df9a | Called once at entry, without explicit arguments; supplies a 32-bit comparison operand | Meaning of result, global access, ownership, side effects, failure behavior |
| F-44362605ab79 | Called only after a nonzero first comparison; receives input base plus 40 and supplies a 32-bit comparison operand | Required object layout, reads and writes, result interpretation, ownership |
| D-01 | Receives two operands; its zero/nonzero result controls each comparison branch | Comparison algorithm, operand types, null handling, memory effects, exact return range |

D-01 is a local opaque dependency label because no published opaque function ID for that dependency was provided. It identifies the same dependency at both call sites.

Only zero versus nonzero is significant to this function. Calling the zero case “equal” would be an inference about D-01, not established behavior.

## Acceptance examples

These examples are conditional expectations from the supplied representation, not observed runtime results. Assume all required calls return normally and all relevant accesses are valid.

### First comparison selects the result

Given that the first D-01 invocation returns zero and G-01 contains 17 when read:

- The result is 17.
- F-44362605ab79 is not called.
- Neither represented field read occurs.

The validity of the input-relative storage is not exercised by the visible body on this path.

### Second comparison selects the result

Given that the first D-01 invocation returns nonzero, the second returns zero, and G-03 contains −8 when read:

- F-44362605ab79 receives input base plus 40.
- The result is −8.
- Neither represented field read occurs.

### Stable nonzero field

Given that both comparisons return nonzero and both field reads produce 73, the result is 73. G-04 is not read by the visible body.

### Stable zero field

Given that both comparisons return nonzero, both field reads produce zero, and G-04 contains 91 when read, the result is 91.

If G-04 instead contains zero, the result is zero. The function does not visibly reject a zero fallback or search for another result.

### Differing field observations

If distinct reads are preserved and valid in the execution model:

- First read 12, second read zero, G-04 equal to 99: result 99.
- First read zero, second read 12: result zero.
- First read 12, second read 34: result 12.

These cases distinguish the represented two-read selection from a single-load implementation.

### Global changed by a dependency

Given that the second D-01 call changes G-03 to 45 and then returns zero, the subsequent global read returns 45, assuming no further intervening change.

This example does not assert that D-01 actually performs such a write. It records why preserving globals across unknown calls cannot be assumed.

## Uncertainties

The following are **documented_only**:

- **Semantic purpose:** A history-related interpretation is suggested by K-01’s text, but the result’s domain meaning is unknown.
- **Comparison semantics:** D-01 is not established as a string comparator or equality predicate.
- **Constant representation:** K-01’s exact storage encoding, terminator, and extent are not supplied.
- **Object layout:** Offsets 40 and 44 do not establish surrounding fields or an object type.
- **Address arithmetic:** Signed overflow, wrapping, address conversion, alignment, and boundary behavior are unspecified.
- **Read fidelity:** Two reads appear in the supplied representation; distinct underlying machine accesses are not independently established.
- **Concurrency:** No synchronization or atomic-access guarantees are shown. The differing-read examples do not establish that unsynchronized concurrent modification is valid.
- **Dependency effects:** No dependency is established as preserving globals, preserving the input object, or retaining resource ownership.
- **Failures:** No explicit recovery path is shown for invalid memory, dependency failure, or exceptional control transfer.
- **Ownership:** Neither dependency results nor the final return value have an established lifetime or release protocol.

## Evidence

The established static evidence consists solely of the supplied function representation: ordered calls, zero/nonzero branches, input-relative offsets, global loads, two represented field reads, and return selection.

The offset calculations are 0x28 = 40 bytes and 0x2c = 44 bytes. No additional structure layout follows from those values.

Interpretations such as string equality, history-name resolution, pointer ownership, or immutable globals are not treated as established facts.

No runtime tests, executable inspection, dependency inspection, or independent machine-level verification have been performed.
