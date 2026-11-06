<!-- Trusted provenance: F-294b5cbdf97e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-294b5cbdf97e

## Purpose

F-294b5cbdf97e selects one of five candidate sources associated with a caller-supplied object, passes the selected source to an output-related dependency, and records a 32-bit selection value in the object.

Selection has strict priority. Three candidates are considered through predicate calls. If all three predicates return zero, two additional candidates are considered through signed 32-bit count fields. At most one candidate is emitted per invocation.

The inferred intent is prioritized selection and output assignment. The established behavior is narrower: the function makes particular dependency calls, sometimes using a temporary region, and writes a selection value after those calls return. The supplied evidence does not establish what the sources represent or whether the output-related dependency copies, retains, transforms, or otherwise processes its source.

If no candidate qualifies, the function returns without making an output call or directly updating the selection field.

## Interface

### Inputs

The function receives four arguments:

| Position | Width and interpretation | Role |
|---|---|---|
| First | 32-bit address-bearing value | Base address of the object containing candidate fields and selection state. |
| Second | 32-bit opaque value | Passed unchanged as the first argument to F-46f0b3cb316e. Its concrete type and validity requirements are unknown. |
| Third | 32-bit opaque word | Initial contents of the first word of a local temporary region. |
| Fourth | 32-bit opaque word | Initial contents of the second word of that temporary region. |

The first argument is used in address arithmetic. It is not treated as a signed quantity for selection purposes. The evidence does not establish portable behavior for invalid addresses, address wraparound, or misaligned accesses.

The third and fourth arguments are not explicitly dereferenced by this function. They seed local storage that is exposed to dependencies only on either of the two count-based paths.

### Return

There is no explicit return value. Observable results consist of dependency interactions, their effects, and a possible 32-bit write to the object’s selection field.

### Object layout used by this function

Offsets below are byte offsets from the first argument. They identify observed access locations, not a complete object schema.

| Offset | Direct use |
|---|---|
| `0x24` | Address passed as the highest-priority source to F-46f0b3cb316e. |
| `0x28` | 32-bit word loaded and passed to F-d8cbf9aa79a9. |
| `0x3c` | Address passed as the third predicate-selected source. |
| `0x40` | 32-bit word loaded and passed to F-d8cbf9aa79a9. |
| `0x44` | Address passed as the second predicate-selected source. |
| `0x48` | 32-bit word loaded and passed to F-d8cbf9aa79a9. |
| `0x6c` | Address passed to F-e1d0a9f8dcb0 when the first count-based candidate qualifies. |
| `0x70` | Signed 32-bit count used for the first count-based candidate. |
| `0x9c` | Address passed to F-017a009221aa when the second count-based candidate qualifies. |
| `0xa0` | Signed 32-bit count used for the second count-based candidate. |
| `0xb4` | 32-bit selection field written after a successful selection path completes its calls. |

A source address and a nearby predicate word may belong to the same larger representation, but that relationship is not established. In particular, passing an address at one offset does not establish how many bytes a dependency reads or writes there.

### Preconditions

For a normally completing invocation:

- Every direct load reached by the selected control path must be valid.
- If a selection is made, the four-byte destination at offset `0xb4` must be writable when the final store occurs.
- Every invoked dependency must receive arguments satisfying its own requirements.
- Object storage and any storage reachable through dependency arguments must remain valid for the accesses actually performed.

These requirements are path-dependent. For example, a nonzero first predicate prevents this function from directly reading either count field.

## Behavior

### Local initialization and predicate priority

The function first initializes an eight-byte local region with the third argument in its first four bytes and the fourth argument in its next four bytes. No address of this region is exposed on the three predicate-selected paths or on the no-selection path.

It then considers candidates in this order:

1. The source at offset `0x24`, controlled by the word at `0x28`.
2. The source at offset `0x44`, controlled by the word at `0x48`.
3. The source at offset `0x3c`, controlled by the word at `0x40`.
4. The candidate associated with offset `0x6c`, controlled by the signed count at `0x70`.
5. The candidate associated with offset `0x9c`, controlled by the signed count at `0xa0`.

For each of the first three candidates, the function loads the corresponding 32-bit word and passes that value—not its address—to F-d8cbf9aa79a9. A nonzero return selects the candidate. Zero permits consideration of the next candidate. Negative predicate returns therefore select just as positive nonzero returns do.

The priority of offsets `0x44` and `0x3c` is significant: the candidate at the higher address, `0x44`, is tested first.

### First predicate-selected candidate

The first object read is the 32-bit word at offset `0x28`.

If F-d8cbf9aa79a9 returns nonzero for that word:

- F-46f0b3cb316e is called with the second input argument and the object address at offset `0x24`, in that order.
- After the call returns, the function writes the 32-bit value `0` at offset `0xb4`.
- The function returns.

No lower-priority predicate or count is directly inspected by this function on this path.

### Second predicate-selected candidate

If the first predicate returns zero, the function loads the word at offset `0x48` and calls F-d8cbf9aa79a9 again.

If that call returns nonzero:

- F-46f0b3cb316e receives the second input argument and the object address at offset `0x44`.
- After the call returns, the function writes the 32-bit value `1` at offset `0xb4`.
- The function returns.

The word at offset `0x40` and both count fields are not directly read on this path.

### Third predicate-selected candidate

If both preceding predicates return zero, the function loads the word at offset `0x40` and calls F-d8cbf9aa79a9.

If that call returns nonzero:

- F-46f0b3cb316e receives the second input argument and the object address at offset `0x3c`.
- After the call returns, the function writes the 32-bit value `2` at offset `0xb4`.
- The function returns.

Neither count field is directly read on this path.

### First count-based candidate

Only after all three predicate calls return zero does the function load the signed 32-bit count at offset `0x70`.

A value greater than or equal to `1` selects this candidate. The function does not add to, subtract from, or otherwise normalize the count.

The selected path performs these interactions in order:

1. F-e1d0a9f8dcb0 receives the object address at offset `0x6c` and a second argument of zero.
2. Its 32-bit result is passed as the second argument to F-b9acea4b06ad. The first argument is the address of the initialized local temporary region.
3. F-46f0b3cb316e receives the second input argument and that same temporary-region address.
4. F-289ec4311c64 receives the temporary-region address.
5. After these calls return, the function writes the 32-bit value `3` at offset `0xb4`.

The count at offset `0xa0` is not directly read on this path.

The zero passed to F-e1d0a9f8dcb0 is consistent with an element position, but its meaning is not established by this function alone.

### Second count-based candidate

If the signed count at offset `0x70` is less than `1`, the function loads the signed 32-bit count at offset `0xa0`.

If this second count is greater than or equal to `1`, the function performs:

1. F-017a009221aa with the object address at offset `0x9c` and a second argument of zero.
2. F-947f8289e35f with the temporary-region address and the preceding call’s 32-bit result.
3. F-46f0b3cb316e with the second input argument and the temporary-region address.
4. F-289ec4311c64 with the temporary-region address.
5. A 32-bit write of `4` at offset `0xb4`, after all these calls return.

The two count-based paths use different retrieval and preparation dependencies. Their representations and semantics must not be assumed interchangeable.

### No-selection path

If all three predicates return zero and both signed counts are less than `1`, the function returns immediately.

It makes no call to F-46f0b3cb316e, performs no temporary preparation or finalization call, and makes no direct write to offset `0xb4`.

This is not a guarantee that all caller-visible state remains unchanged: the preceding predicate calls may have side effects.

## Side effects

The only explicit write through the object’s base address is the four-byte selection write at offset `0xb4`. On every selected path, it occurs after the output-related call returns. On the count-based paths, it also occurs after F-289ec4311c64 returns.

Consequently:

- This function has not yet written the new selection value while F-46f0b3cb316e is executing.
- On a count-based path, it has not yet written the new selection value while F-289ec4311c64 is executing.
- If a dependency changes the selection field and returns normally, the subsequent explicit selection store overwrites that change.
- A dependency that does not return normally can prevent the later store.

The dependency calls may have additional effects on object memory, output storage, global state, or other reachable storage. None is established as side-effect-free.

Reads occur incrementally rather than from an initial snapshot. A dependency can therefore affect values loaded later. For example, after the first predicate returns zero, the next predicate receives the then-current word at offset `0x48`.

Aliasing is not checked. The second argument may designate storage overlapping the object if dependencies permit it. Such overlap can affect dependency-visible contents; this function still performs its final selection store after the selected path’s calls return.

The temporary region has invocation-local lifetime. F-46f0b3cb316e receives its address only on the count-based paths, before F-289ec4311c64 is called. The evidence does not establish whether either dependency copies or retains any referenced data, or whether F-289ec4311c64 releases resources.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-d8cbf9aa79a9 | Receives one loaded 32-bit word; zero/nonzero return determines whether selection continues. Called one to three times. |
| F-46f0b3cb316e | Receives the second input argument and a selected source address. Called exactly once when a candidate is selected, otherwise not called. |
| F-e1d0a9f8dcb0 | Receives the address at offset `0x6c` and zero; supplies a 32-bit value for the first count-based preparation call. |
| F-b9acea4b06ad | Receives the temporary-region address and the result from F-e1d0a9f8dcb0. |
| F-017a009221aa | Receives the address at offset `0x9c` and zero; supplies a 32-bit value for the second count-based preparation call. |
| F-947f8289e35f | Receives the temporary-region address and the result from F-017a009221aa. |
| F-289ec4311c64 | Receives the temporary-region address after the count-based output call and before the selection store. |

No dependency return is explicitly checked for failure other than interpreting the predicate’s return as zero or nonzero. The retrieval results are forwarded without validation. No retry, rollback, or explicit error-reporting path is present.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. They assume valid reached accesses and normal dependency returns.

1. **Highest-priority selection:** If the first predicate returns any nonzero value, F-46f0b3cb316e receives the source address at `0x24`; the final selection value is `0`. Later predicates and counts are not directly read.

2. **Priority is not address order:** If the first predicate returns zero and the second returns nonzero, the source address is `0x44` and the final selection value is `1`. The candidate at `0x3c` is not tested.

3. **Third predicate selection:** If the predicate returns are zero, zero, and nonzero, the source address is `0x3c` and the final selection value is `2`.

4. **Positive first count:** If all predicates return zero and the signed count at `0x70` is `1`, the first retrieval/preparation sequence is used. Output precedes temporary finalization, which precedes writing selection value `3`. The second count is not read.

5. **Negative counts do not qualify:** If all predicates return zero, the count at `0x70` is `−1`, and the count at `0xa0` is `1`, the second retrieval/preparation sequence is used and the final selection value is `4`.

6. **Signed-width boundary:** A count word with bit pattern `0x80000000` is negative as a signed 32-bit value and does not qualify. A count of `0x7fffffff` qualifies. No count arithmetic occurs.

7. **No selection:** If all predicates return zero and both counts are zero or negative, there is no output call and no direct selection-field write. Any predicate side effects remain possible.

8. **Later reads observe intervening changes:** If the first predicate changes the word at `0x48` and returns zero, the second predicate receives the changed word, assuming it remains readable.

9. **Final-store precedence:** If the selected output call writes another value at `0xb4` and returns normally, the function subsequently replaces that value with its branch’s selection value. On a count-based path, finalization also runs before that replacement.

## Uncertainties

The following remain **documented_only**:

- Concrete object types, field names, and meanings of selection values `0` through `4`.
- Whether predicate nonzero means valid, available, nonempty, or something else.
- The second argument’s representation and output semantics.
- The interpretation of zero passed to the retrieval dependencies.
- Preparation dependencies’ read/write extents within or beyond the temporary region.
- Whether the third and fourth arguments remain semantically significant after preparation.
- Ownership transfer, retention, allocation, reference counting, and release behavior.
- Dependency effects on globals, object lifetime, and aliased storage.
- Exceptional, faulting, or nonlocal-exit behavior.
- Concurrency guarantees, byte order, and alignment requirements.

The observed order does not establish an atomic transaction. There is no visible synchronization or rollback mechanism in this function.

## Evidence

Established static evidence consists of the supplied function body and opaque dependency mapping. It supports the argument widths, listed offsets, signed count comparisons, branch priority, dependency argument order, local initialization, final-store timing, and no-selection return described above.

Prioritized output assignment and temporary finalization are inferred interpretations of those interactions, not independently established dependency contracts.

No runtime tests have been performed. The acceptance examples are static behavioral obligations conditional on the stated preconditions and dependency behavior.
