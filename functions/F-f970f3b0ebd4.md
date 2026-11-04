<!-- Trusted provenance: F-f970f3b0ebd4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f970f3b0ebd4

## Purpose

F-f970f3b0ebd4 selects between pointers to two invocation-local 32-bit objects, conditionally updates caller-addressed state, invokes a final dependency with the selected pointer, and returns a 64-bit value assembled from two local words.

The returned words are not necessarily the words accessible through the selected pointer. In particular, the alternative selection path passes a pointer to the local initialized from C, but still returns the words initially initialized from A and B.

**Inferred intent:** The comparison, conditional helper call, and increment-by-two update suggest management of a position or resource. No domain interpretation is established. The requirements below concern numeric values, addresses, local objects, and observable operation order.

## Interface

### Inputs and output

| Input | Width and interpretation | Established use |
|---|---|---|
| A | Unsigned 32-bit value | Initializes the return-low local and is passed separately to the final dependency. |
| B | Unsigned 32-bit value | Initializes the return-high local, supplies the base for state-address arithmetic, and is passed to the conditional helper. |
| C | Unsigned 32-bit value | Initializes the alternative-target local and supplies the initial comparison candidate. |
| D | Opaque 32-bit value | Initializes a fourth local object. No direct arithmetic use is established. |

The output is a 64-bit bit pattern:

\[
R = 2^{32}H + L,
\]

where \(L\) and \(H\) are the unsigned 32-bit contents of the return-low and return-high locals after the final dependency returns.

This establishes word significance, not a signed result interpretation, success indicator, pointer type, byte order, or ownership transfer.

### Local objects and pointer arguments

Four separate 32-bit local objects are explicitly initialized:

| Contract label | Initial value | Established subsequent use |
|---|---|---|
| Return-low local | A | Its address is selected on the update path; its final contents supply the low return word. |
| Return-high local | B | Replaced with the candidate on the update path; its final contents supply the high return word. |
| Alternative-target local | C | Its address is passed to the first dependency and selected on the alternative path. |
| Fourth local | D | Initialized before the first dependency; no later direct read or write is shown. |

These labels describe separate objects, not fields of an established array or record. The supplied representation does **not** guarantee their physical adjacency, order in memory, or offsets relative to one another.

Consequently:

- Passing the address of the alternative-target local establishes a pointer to that object. It does not establish that D is accessible as a following word.
- Passing the address of the return-low local establishes a pointer to that object. It does not establish that the return-high local is accessible as a following word.
- The two-word return composition is established independently of any physical adjacency between its source objects.
- A dependency’s actual access extent and ability to affect other local objects remain unknown.

### State addresses and arithmetic

For each directly accessed field, the address is obtained by adding its offset to B **as unsigned 32-bit arithmetic**, then converting the resulting value to a pointer:

\[
\operatorname{address}(k)=\operatorname{pointer}\bigl((B+k)\bmod 2^{32}\bigr).
\]

The addition therefore wraps before pointer conversion. This is not established as pointer arithmetic within a conventional allocated object.

| Offset \(k\) | Access width | Direct interpretation |
|---|---|---|
| 4 | 32 bits | Unsigned comparison bound |
| 12 | 32 bits | Unsigned selection value; conditionally overwritten |
| 16 | 32 bits | Signed value for the positive test and subsequent conditional decrement |

References below to an “offset-12 field,” for example, mean the 32-bit cell at \(\operatorname{address}(12)\), including the specified wrapping address calculation.

Address arithmetic is established; address validity is not. Normal completion requires the actual accessed cells to support the relevant accesses:

- Offset 12: readable after the first dependency returns.
- Offset 4: readable when the observed selection value is nonzero.
- Offset 16: readable on the update path.
- Offset 12: writable on the update path.
- Offset 16: readable again and writable after the conditional helper returns.

Dependencies may impose additional requirements. There is no visible validation of nullness, mapping, alignment, accessible extent, or writable storage.

## Behavior

### Initialization and first dependency

Before any direct state-field read, the function initializes the four local objects in this order:

1. Return-low local from A.
2. Return-high local from B.
3. Alternative-target local from C.
4. Fourth local from D.

It then calls F-e511432e9f02 with the address of the alternative-target local. No result from that call is directly consumed.

The pointed-to local is not established as read-only. Its contents may differ when the call returns. The call does not, merely by receiving this pointer, have an established two-word argument containing C and D.

The comparison candidate initially remains the original value C. The function does not directly reload that candidate from the alternative-target local after the call. A change to that local alone therefore does not replace the candidate.

No preservation of state fields, globals, ownership, or other reachable memory is assumed across this dependency.

### Selection reads

After the first dependency returns, the function reads the unsigned 32-bit offset-12 field. Call this observation the **selection value**.

If the selection value is nonzero, the function next reads the unsigned 32-bit offset-4 field and replaces the candidate with that observed bound.

If the selection value is zero, this function does not directly read offset 4.

The update path is selected exactly when:

- The selection value is nonzero; and
- The selection value is strictly less than the candidate under an unsigned 32-bit comparison.

Thus every successful update comparison uses the bound read from offset 4, rather than C. C cannot cause the update path to be taken when the selection value is zero.

Equality and unsigned greater-than select the alternative path.

### Update path

The function first reads the offset-16 field as a signed 32-bit value and tests whether it is strictly positive.

If the value is zero or negative:

- F-b2262fcb3198 is not called.
- No direct offset-16 write occurs.
- The candidate remains the bound previously read from offset 4.

If the tested value is strictly positive:

1. F-b2262fcb3198 is called with B and the earlier selection value.
2. Its 32-bit result becomes the candidate.
3. After the call returns, the offset-16 field is read again.
4. That newly read signed value is decremented by one and written to offset 16.

The decrement uses the post-call observation, not the value that passed the positive test. The helper may have changed it to a positive, zero, or negative value. There is no second positive test.

For post-call values other than the minimum signed 32-bit value, the decrement has the ordinary representable signed result. If the reread value is \(-2^{31}\), the supplied representation does not establish a portable signed-overflow outcome. A wrapping result at that boundary must not be assumed.

The helper result is not checked against the original selection value or bound.

After the optional helper and decrement, the function performs these direct operations in order:

1. Rereads the unsigned offset-12 field.
2. Stores that observation in the return-low local.
3. Writes candidate plus two to offset 12.
4. Stores the candidate in the return-high local.

The offset-12 reread matters: its value may differ from the selection value used in the comparison and helper argument.

Candidate-plus-two is unsigned 32-bit arithmetic:

\[
\text{written value}=(\text{candidate}+2)\bmod 2^{32}.
\]

The return-high local receives the candidate itself, not the incremented value.

The selected pointer on this path is the address of the return-low local. This selection does not establish a contiguous two-word record at that address.

### Alternative path

When the update condition fails, the selected pointer is the address of the alternative-target local.

On this path, the function:

- Does not directly read offset 16.
- Does not call F-b2262fcb3198.
- Performs no direct state-field write.
- Does not directly replace either return-source local after initialization.

Absent dependency effects on those return-source locals, they therefore remain A and B. The alternative-target local may have been modified by F-e511432e9f02.

### Final dependency and return

On both paths, F-2aaa2ea360f2 is called with:

- Original A as its first argument.
- The selected local-object pointer as its second argument.

A remains the first argument even when the update path has replaced the return-low local with another value.

After this call returns, the function forms its 64-bit result from the return-high and return-low locals. No result returned by F-2aaa2ea360f2 is directly consumed.

Changes to either return-source local, if a dependency actually makes them, affect the result. However, the selected pointer alone establishes neither access to the other return-source local nor a route to unrelated locals. Such access requires additional layout or dependency evidence.

## Side effects

The function’s direct effects consist of:

- Four local initializations.
- One call to F-e511432e9f02.
- The conditional state reads described above.
- On the update path, replacement of both return-source locals and a write to offset 12.
- On the signed-positive subpath, a call to F-b2262fcb3198 followed by the offset-16 reread and decrementing write.
- One call to F-2aaa2ea360f2 after the preceding operations complete normally.
- Assembly of the return value from the two return-source locals.

The ordering is significant. The first dependency precedes every direct state read. The conditional helper precedes the offset-16 decrement and the later offset-12 reread and write. The final dependency follows those direct state updates.

No atomic transaction, rollback, synchronization, or preservation guarantee is established. A final dependency may change state previously written by this function, so a direct write is not necessarily the state’s value at return.

There is no explicit allocation or release operation in the function. This does not establish that dependencies leave memory ownership unchanged.

Pointers passed to dependencies designate invocation-local objects. No supported post-return lifetime for dereferencing retained pointers is established.

No explicit error status, recovery branch, or cleanup branch is shown. A dependency that does not return normally prevents the subsequent described operations from being guaranteed. Fault behavior for invalid accesses is unknown.

## Dependencies

| Opaque ID | Invocation and established contribution | Unresolved contract |
|---|---|---|
| F-e511432e9f02 | Called once, before direct state reads, with the address of the local initialized from C. No returned value is directly used. | Interpretation, access extent, mutation, pointer retention, global effects, ownership, and failure behavior. Receipt of D as an adjacent word is not established. |
| F-b2262fcb3198 | Called only after a successful unsigned selection comparison and a signed-positive offset-16 test. Receives B and the earlier selection value; supplies the 32-bit candidate. | Result meaning, state changes, other memory effects, ownership, and failure behavior. Later field rereads prevent assuming preservation. |
| F-2aaa2ea360f2 | Called with original A and the address of either the return-low local or the alternative-target local. Runs before return-word observation. No returned value is directly used. | Pointer interpretation, access extent, mutations, retention, globals, ownership, and failure behavior. Receipt of a two-word record is not established. |

## Acceptance examples

These are conditional static expectations, not executed tests. Unless specified otherwise, assume all required accesses are valid, dependencies return normally, and dependencies preserve relevant memory except for explicitly stated changes.

### Zero selection value

If the offset-12 read yields zero:

- Offset 4 and offset 16 are not directly read.
- The conditional helper is not called.
- No direct state write occurs.
- The final dependency receives the address of the alternative-target local.
- The return has low word A and high word B.

If the first dependency changes only the alternative-target local from C to 19, the final dependency receives a pointer to an object containing 19. No expectation about a neighboring D word follows.

### Equality and unsigned ordering

If the selection value and bound are both 7, equality selects the alternative path. A positive offset-16 value does not cause a helper call, because offset 16 is not directly inspected on that path.

If the selection value is \(2^{32}-1\) and the bound is 8, the alternative path is also selected. The comparison is unsigned; the selection value is not treated as negative.

### Update without helper

If the selection value is 3, the bound is 8, and the offset-16 value is zero:

- The helper is not called.
- Offset 12 is written with 10.
- Before the final dependency, the return-low local contains 3 and the return-high local contains 8.
- The final dependency receives the address of the return-low local.
- With those locals preserved, the result has low word 3 and high word 8.

The pointer does not independently establish that both values are accessible as adjacent words.

A negative signed offset-16 value produces the same direct branch behavior.

### Helper mutations and rereads

Suppose the initial selection value is 3, the bound is 8, and the signed offset-16 value is 2. Suppose the helper receives B and 3, returns 20, writes 6 to offset 16, and writes 11 to offset 12.

The subsequent direct behavior is:

- Offset 16 is reread as 6 and written as 5.
- Offset 12 is reread as 11.
- The return-low local is set to 11.
- Offset 12 is written as 22.
- The return-high local is set to 20.

With return-source locals preserved by the final dependency, the result has low word 11 and high word 20.

### Candidate increment wraparound

If the helper supplies candidate \(2^{32}-1\), the offset-12 write is 1. The return-high local receives \(2^{32}-1\), not 1.

If the helper instead changes offset 16 to \(-2^{31}\), this example does not establish the outcome of the decrement. That signed-overflow boundary remains documented_only.

### State-address wraparound

For B equal to hexadecimal FFFFFFF8, unsigned 32-bit address calculations produce:

| Field | Numeric value before pointer conversion |
|---|---|
| Offset 4 | FFFFFFFC |
| Offset 12 | 00000004 |
| Offset 16 | 00000008 |

If these converted addresses support the required accesses, the ordinary branch rules apply. Arithmetic wraparound neither guarantees accessibility nor triggers an explicit rejection path.

### Final mutation of the pointed-to local

On the update path, suppose the return-high local contains 20 before the final call, and the final dependency writes 9 through its supplied pointer while preserving the return-high local.

The result has low word 9 and high word 20. This expectation requires no assumption of adjacent local storage.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of inputs, state fields, helper results, and the 64-bit output.
- Physical arrangement, adjacency, padding, and relative addresses of the four local objects. Apparent stack-position cues suggest a possible layout but do not establish one.
- Whether either pointer-taking dependency accesses a neighboring local, and whether such access is valid.
- Dependency behavior beyond the observed argument and result relationships, including global effects, pointer retention, ownership, and failures.
- Integer-to-pointer mapping, alignment requirements, byte order, and accessibility of the computed addresses. The unsigned 32-bit address arithmetic itself is established.
- Signed decrement behavior when the post-helper reread is the minimum signed 32-bit value.
- Concurrent interference, volatile access guarantees, external synchronization, and atomicity.
- Whether callers interpret the return as a scalar, pair, address-like value, or another representation.

## Evidence

Established requirements derive solely from the supplied static function representation: input widths, separate local declarations and initialization, explicit address-taking, dependency calls, unsigned 32-bit offset arithmetic before pointer conversion, conditional reads, unsigned selection comparison, signed-positive test, post-helper rereads, ordered writes, and final high-word/low-word concatenation.

No contiguous local array or two-word pointer argument is explicitly established. Any such physical interpretation is inferred rather than contractual.

The possible resource-management interpretation imposes no acceptance requirement. No dependency implementation, caller contract, runtime trace, or execution result was supplied. No runtime tests have been performed.
