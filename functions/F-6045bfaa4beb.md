<!-- Trusted provenance: F-6045bfaa4beb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6045bfaa4beb

## Purpose

F-6045bfaa4beb accepts two 32-bit words and performs a state-driven operation involving a 16-byte automatic buffer, three dependencies, arithmetic transformations, and conditional byte stores. It returns a signed 32-bit result only when dispatch selects index 5.

**Established static evidence:** the routine can stage its inputs for one dependency, pass the buffer to two dependencies, prepare a pointer for subsequent byte stores, and derive dispatch values from arithmetic and dependency outputs.

**Inferred intent:** the buffer exposure and byte transformations suggest buffer processing. They do not establish encryption, validation, initialization, or any application-level purpose.

**Unknown:** external values and dependency implementations are unavailable. Actual stage reachability, termination, store coverage, and result meanings remain unresolved.

## Interface

### Inputs and output

The inputs are two uninterpreted 32-bit words. They are retained separately from two argument slots that initially contain zero. Index 4 copies the retained inputs into those slots, preserving input order. F-ab4565f926b1 receives the slots, not necessarily the original inputs.

The pending return value initially equals zero. Index 5 returns its current signed 32-bit value. Other stages may replace it without returning.

No input-pointer interpretation, ownership transfer, nullability rule, or success/error convention is established.

### Storage and state

The following labels describe behavioral roles, not implementation identifiers.

| State | Initial contents or role |
|---|---|
| Buffer | 16 contiguous bytes of automatic storage; no visible initialization |
| Count | 32-bit integer, initially zero |
| Source pointer; active pointer | Initially null |
| Staged inputs | Two 32-bit words, initially zero |
| Pending result | Signed 32-bit integer, initially zero |
| Basis A; basis B | Independently read external seeds |
| Worker A; worker B | Signed 32-bit state, initially zero |
| Mixer A; mixer B | 32-bit state; A initially zero, B externally seeded |
| Route A; route B | 32-bit state, initially zero |
| Stage key | Externally seeded 32-bit word; not subsequently assigned |
| Transformed result | Signed 32-bit state, initially zero |
| Accumulator | Unsigned 32-bit state, initially zero |
| Encoded value; byte input | Signed 32-bit state, initially zero |
| Dispatch word | Externally seeded unsigned 32-bit word |

Stage descriptions identify the routine’s explicit assignments. An absence of such an assignment is not a guarantee about arbitrary dependency side effects.

### Widths and preconditions

Dispatch subtraction, unsigned bitwise calculations, and logical selector shifts operate on 32-bit quantities. Dispatch subtraction wraps modulo \(2^{32}\). Both selectors are extracted by a logical right shift of 31, producing zero or one.

Other arithmetic includes signed 32-bit operands. The static representation does not establish portable behavior for signed overflow, out-of-range character conversion, or pointer-to-integer conversion. Equations below describe represented arithmetic relationships; they do not impose an unsupported universal wraparound model.

Defined execution requires valid reached memory accesses and compatible dependency behavior. In particular:

- A reached byte store requires a writable calculated address.
- A dependency reading the buffer requires a valid basis for reading bytes not initialized by this routine.
- Address-based arithmetic in index 6 requires the target’s pointer-to-32-bit-integer semantics.
- Automatic storage remains valid only during the invocation.

## Behavior

### External reads and coefficient identity

Initialization reads five external seeds in this order: stage key, basis A, basis B, mixer B, dispatch word. Other initializations occur between some reads. Both inputs are retained before dispatch begins.

The recurring coefficient labels below denote external locations, not immutable values:

- **Shared scale** and **shared offset** are used by several arithmetic stages.
- **Default multiplier** and **default subtrahend** are used to transform the result of F-ab4565f926b1.
- **Multiplier**, **bias**, and **step** participate in other arithmetic and iteration-entry snapshots.

The default multiplier and default subtrahend are distinct locations from the shared scale and shared offset. No equality between their values is established. In particular, the default-stage transformation must not be rewritten using the shared coefficients.

Each dispatch iteration reads, in order:

1. Step, retained for that iteration.
2. Bias, retained for that iteration.
3. Multiplier, retained for that iteration.
4. The dispatch-base word.

The unsigned difference between the current dispatch word and the freshly read dispatch base selects the stage. Differences 0 through 9 select their corresponding indices. Index 1 and every unmatched difference select the same stage.

The iteration-entry snapshots are used by index 7’s encoded-value update. Other occurrences of the same coefficient labels denote fresh reads unless expressly identified as snapshots. Dependencies may change external locations between reads. Ordering between separate statements is established; evaluation order among operands within one compound expression is not.

### Index 0: route transition and worker update

The dispatch word becomes route A exclusive-or a freshly read transition word. This occurs before the dependency call.

F-870c4fcf7295 then receives worker B minus a freshly read adjustment as its first argument. Its secondary output replaces worker B. It does not provide this stage’s dispatch value.

No other persistent state receives an explicit assignment here.

### Index 1 and unmatched indices: two-input dependency

F-ab4565f926b1 receives the two staged words, in order, followed by the buffer address.

After that call returns, the routine reads the **default multiplier** and **default subtrahend** and replaces the transformed result with

\[
\text{default multiplier}\times\text{dependency result}
-\text{default subtrahend}.
\]

These are not shared-scale or shared-offset reads. Their relative evaluation order within the expression is not established, but both occur after the dependency returns. Only after this transformation is the buffer address assigned to the source pointer.

Two F-870c4fcf7295 calls follow:

- The first uses basis A minus a freshly read adjustment. Its secondary output is retained for dispatch.
- After that call, mixer A is exclusive-or updated using a fresh external mask.
- The second uses the existing worker A minus another freshly read adjustment. Its secondary output will replace worker A.

After the second call, external reads supply, in order, a worker-B replacement, a route-B mask, and a route-A mask. Between the latter two reads, the routine installs the second secondary output as worker A and the first as dispatch.

Route A and then route B are exclusive-or updated using their respective masks.

Basis A is not replaced. The first helper output does not feed the second helper argument. Basis B and mixer B receive no explicit assignments. No dependency result is immediately interpreted as an error or return instruction.

### Index 2: mixer transition

Dispatch becomes mixer B exclusive-or a freshly read transition word. There is no dependency call or other persistent-state assignment.

### Index 3: fixed-operation dependency and selection

F-6f0d307cf172 receives the 32-bit operation word `0x14000002`, zero, and the buffer address. Its result initially replaces worker A.

Subsequent fresh reads establish these relationships:

- The transformed result is shared scale times the returned result, less shared offset.
- The accumulator is transformed result times multiplier, plus bias.
- A selector is obtained by complementing the bitwise OR of the accumulator and a separate affine transformation of the transformed result, then logically shifting by 31. That separate transformation includes another bias read.
- The pending result becomes shared scale times accumulator, less shared offset.

These calculations do not use the iteration-entry coefficient snapshots.

Three F-870c4fcf7295 calls follow:

1. The first uses basis A plus a freshly read adjustment. After return, external reads supply a route-B replacement, route-A replacement, worker-A replacement, mixer-A mask, and mixer-B mask, in that order. Both mixers are exclusive-or updated. The secondary output replaces basis A.
2. The second uses basis B minus a fresh adjustment. Its secondary output is the selector-one candidate.
3. The third uses the same unchanged basis B minus a different fresh adjustment. Its secondary output is the selector-zero candidate.

The one-bit selector chooses the second call’s output when one and the third call’s output when zero.

Worker B and basis B are not explicitly assigned. The later external worker-A replacement supersedes the fixed-operation dependency’s return. This stage does not assign the source pointer.

### Index 4: input staging and basis updates

The retained inputs replace the staged words. Dispatch then becomes stage key exclusive-or a freshly read transition word.

The first helper call uses basis B plus a fresh adjustment. Its secondary output immediately replaces basis B.

The second helper call uses basis A minus a fresh adjustment. After return, an external mixer-B mask is read and applied, followed by an external mixer-A replacement. The second secondary output then replaces basis A.

Neither helper supplies dispatch. Workers and routes receive no explicit assignments.

### Index 5: return

The current pending result is returned.

The iteration-entry step, bias, multiplier, and dispatch-base reads have already occurred. No stage-local dependency call or further result transformation occurs.

### Index 6: pointer and encoded-value preparation

Count becomes 16, then the source pointer is copied into the active pointer.

The encoded value is constructed from the address of the count object through nested externally parameterized arithmetic. This calculation includes fresh shared-scale, shared-offset, multiplier, and bias reads. The address is represented as a signed 32-bit integer. The calculation neither reads count through that address nor passes the address to a dependency.

Dispatch then becomes mixer A exclusive-or a fresh transition word.

Two helper calls follow:

- The first uses worker A plus a fresh adjustment.
- A route-A mask is read after that call.
- The second uses worker B plus another fresh adjustment.
- A route-B mask is read after that call.
- The first secondary output replaces worker A; the second replaces worker B.
- Route A and then route B are exclusive-or updated using the retained masks.

The second call does not consume the first output. Neither basis nor mixer is explicitly assigned.

Setting count to 16 does not establish sixteen iterations or sixteen stores.

### Index 7: count update and byte store

The current byte input is first converted to character width and retained for byte construction. Count is then replaced by an externally parameterized arithmetic transformation.

The destination index is calculated from the updated count using fresh external coefficients, including fresh multiplier and bias reads. The destination is the active pointer plus that calculated index. There is no explicit null check or bounds check.

Under an 8-bit-character, low-byte-truncation model, let \(x\) denote the retained byte value. Define

\[
q=\bigl(127((2x+1)\mathbin{\mathrm{OR}}139)+69+x\bigr)\bmod256.
\]

The stored byte is

\[
\bigl((x-144)q-127x+112\bigr)\bmod256.
\]

This conditional byte formula does not establish address validity or resolve the target’s complete signed-arithmetic semantics.

After the store, the encoded value is updated using the full, untruncated byte input. The nested transformation uses:

- The iteration-entry step snapshot with the byte input.
- The iteration-entry multiplier and bias snapshots.
- Fresh reads of other coefficients, including shared scale and shared offset.

Thus the store-index calculation and encoded-value update can observe different multiplier and bias values in the same iteration.

Finally, dispatch becomes route B exclusive-or a freshly read transition word. No dependency is called.

### Index 8: byte-input preparation and selection

The encoded value is transformed into the byte input using two external operands.

Separately, count is transformed into an intermediate used by the selector calculation. The accumulator is replaced by an affine transformation of the transformed result. Further external arithmetic and bitwise operands produce a one-bit selector through a logical right shift of 31. All these calculations precede the helper calls.

Two helper calls use:

- Worker A minus one fresh adjustment.
- The same unchanged worker A minus a different adjustment read after the first call.

The first secondary output is retained across the second call. Selector one chooses the first output as dispatch; selector zero chooses the second.

Neither output replaces worker A. The calls do not form an output-to-input chain. Dependency changes to external state can nevertheless affect the second adjustment read.

Count, encoded value, transformed result, worker B, bases, mixers, and routes receive no explicit assignments.

### Index 9: pending-result construction

The pending result becomes

\[
\text{shared scale}\times
(\text{accumulator}\times\text{result coefficient}
+\text{result addend})
-\text{shared offset},
\]

using fresh external reads.

Only after this assignment is an adjustment read for a helper call using worker B minus that adjustment. Its secondary output replaces dispatch; worker B is unchanged.

This stage does not return.

## Side effects

The routine writes its automatic state and attempts one byte store each time index 7 is reached. The destination is calculated; containment within the 16-byte buffer is not established.

No explicit direct assignment to an external data word appears. Nevertheless, external preservation cannot be promised: dependencies may change external words, other memory, or process state. Reads after a call must not be replaced by earlier observations merely because they reference the same location.

The buffer is exposed without visible initialization. The source pointer is assigned only after F-ab4565f926b1 returns and its result transformation completes. Passing the buffer to F-6f0d307cf172 does not prepare that pointer.

No explicit allocation or release operation appears. The buffer has automatic lifetime. Passing or retaining its address does not establish ownership transfer or extend its lifetime. Dependency allocation, release, retention, and access behavior are unknown.

There is no dedicated error path or iteration limit. Invalid accesses, arithmetic-semantic problems, dependency failures, and nontermination are not visibly translated into defined error results.

## Dependencies

| Opaque ID | Established interface and use | Unresolved contract |
|---|---|---|
| F-870c4fcf7295 | Two arguments; every visible second argument is 2,147,483,647. A represented secondary output supplies state or a dispatch candidate. | Primary-result meaning, secondary-output ABI, arithmetic operation, failures, and side effects |
| F-ab4565f926b1 | Two staged 32-bit words followed by the buffer address; returns an integer transformed using the independent default coefficients. | Input meanings, buffer access extent, initialization, ownership effects, and result semantics |
| F-6f0d307cf172 | Operation word `0x14000002`, zero, and the buffer address; returns an integer transformed using shared scale and shared offset. | Operation meaning, buffer access extent, ownership effects, and result semantics |

The repeated helper argument does not establish division, remainder, or modular-reduction semantics. Secondary-output dataflow is established by the representation, but the underlying ABI and dependency guarantees are not supplied.

## Acceptance examples

These are conditional static criteria, not executed observations.

1. **Immediate return.** If the first dispatch difference is 5, the result is zero. Initialization seed reads and the first iteration’s snapshot/base reads still occur. No dependency is called.

2. **Input staging.** If index 4 precedes the default stage without intervening changes to the staged words, F-ab4565f926b1 receives the original inputs in order. Without prior staging, its first two arguments are zero.

3. **Unmatched dispatch.** A difference outside 0 through 9 selects the index-1 behavior, not an invalid-state return.

4. **Independent result coefficients.** Suppose F-ab4565f926b1 returns 7, and the subsequent default coefficient reads yield 3 and 2. The transformed result becomes 19, assuming defined arithmetic. Shared-scale and shared-offset values do not replace those observations, even if they differ.

5. **Default-stage output flow.** If the helper secondary outputs are \(a\) and \(b\), respectively, dispatch receives \(a\), worker A receives \(b\), and basis A receives no explicit replacement.

6. **Index-8 argument independence.** If worker A is \(w\), both helper arguments are formed from \(w\). The second uses its later adjustment read, not the first helper output.

7. **Mutable coefficients.** If index 7’s multiplier or bias snapshots differ from its later reads, the store index uses the later observations while the encoded update uses the snapshots.

8. **Prepared pointer.** After normal completion of the default stage and subsequent index 6, the active pointer identifies the buffer, provided no intervening side effect alters the relevant state.

9. **Unprepared pointer.** Index 7 does not explicitly reject a null active pointer. It attempts the calculated store; validity depends on the resulting address and environment.

10. **Zero byte input.** Under the stated byte model, \(x=0\) gives \(q=58\) and stored byte 208. Address validity remains separate.

11. **Deferred return.** Index 9 updates the pending result and calls the helper. A return requires a later selection of index 5.

## Uncertainties

The following remain **documented_only**:

- External values, including any coincidental equality between distinct coefficient locations.
- External mutation by dependencies or environmental activity.
- Actual stage sequences, termination, call counts, and store counts.
- Final addressing signedness and whether calculated stores remain within the buffer.
- Dependency read/write extents and initialization behavior.
- Input meanings, result meanings, and success/error conventions.
- Secondary-output ABI and preservation across subsequent calls.
- Pointer width and address-to-signed-32-bit conversion semantics.
- Character signedness, out-of-range conversions, and signed-overflow behavior.
- Operand evaluation order within compound expressions.

These unknowns do not erase established ordering between separate operations, retained snapshot identity, or the distinction between default-stage coefficients and shared coefficients.

## Evidence

This contract uses only the supplied static representation. It establishes storage sizes, explicit initialization, dispatch destinations, dependency arguments, represented secondary-output flow, stage-local assignments, and ordering where separate operations establish it.

The default-stage result multiplication and subtraction refer to external locations distinct from the shared scale and offset used elsewhere. Their reads occur after F-ab4565f926b1 returns. No static evidence equates their values.

Dependency implementations, external values, and instruction-level arithmetic evidence were not supplied. Application-level purpose and complete execution paths remain unresolved.

No runtime tests, dependency experiments, memory traces, or dynamic path observations have been performed.
