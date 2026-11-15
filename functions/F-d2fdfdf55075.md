<!-- Trusted provenance: F-d2fdfdf55075; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d2fdfdf55075

## Purpose

F-d2fdfdf55075 accepts two 32-bit address bases and returns an unsigned 32-bit word. Its established behavior consists of state-dependent dispatch, ordered global and input-relative reads, calls to opaque dependencies, byte transformations, and arithmetic over retained state.

The result is not established as Boolean, a comparison ordering, or an error code. Neither a fixed processing length nor termination is established.

**Inferred intent:** Paired byte accesses and an auxiliary operation using fields from one input suggest structured-data comparison or validation. That interpretation is not an acceptance requirement.

This contract separates established static behavior from inferred intent and unresolved matters. **documented_only** marks behavior not determined by the supplied evidence. No runtime tests have been performed.

## Interface

### Inputs and observed layout

The two arguments are represented as signed 32-bit integers and used as address bases. They are called the **first base** and **second base**.

| Base | Byte offset | Established use |
|---|---:|---|
| First | 236 | Prepared byte-view address |
| First | 1000 | Prepared byte-view address |
| First | 1296 | 32-bit field read |
| Second | 32 | Prepared byte-view address |
| Second | 1020 | Prepared byte-view address |
| Second | 1040 | Auxiliary argument and retained byte-view address |
| Second | 1296 | 32-bit field read, including auxiliary argument preparation |
| Second | 1300 | 32-bit field read for auxiliary argument preparation |
| Second | 1304 | Auxiliary argument and retained byte-view address |

Address preparation alone does not establish a dereference. These offsets do not define complete structures, allocation sizes, region lengths, alignment, or byte order.

The inputs are not established as strings, immutable records, owned allocations, or handles. Aliasing between them is not excluded.

### Output and preconditions

The return destination returns the current retained 32-bit result word without normalization. The result initially contains zero, but numerous destinations overwrite it.

Normal completion requires valid accesses on the selected path, satisfaction of each reached dependency’s unknown requirements, returning dependencies, and eventual selection of the return destination.

There is no established minimum allocation size sufficient for every possible execution. Zero-base tests do not establish generally safe null-input handling.

## Behavior

### Entry captures and dispatch

Most working state starts at zero, including saved bases, prepared and active views, indices, and the result word. Global entry captures occur in this order:

1. A selector-related value used to form the next selector in destination 8.
2. A helper-state seed used by the first helper call in destination 8 and by later helper preparation.
3. A selector-state seed subsequently XOR-updated and used to form the selector in destination 21.
4. Another helper-state seed used by the second helper call in destination 8 and by later helper preparation.
5. Another selector-state seed subsequently XOR-updated and used in saved-second-base selection at destination 7.
6. The initial dispatch selector.

Thus the entry captures alternate helper-state and selector-state roles after the initial selector-related capture; they are not three helper-state seeds.

The supplied bases are saved only in destination 8, after two ordered calls to F-870c4fcf7295. Until then, the saved bases remain zero. Preparation-before-use is not established.

Every dispatch iteration:

- Reads a global arithmetic coefficient into a per-iteration snapshot.
- Subsequently reads the global selector offset.
- Selects a destination using the current selector plus that offset as a 32-bit unsigned sum.

These reads occur even when the selected destination immediately returns. Destinations are numbered 0 through 27. Destination 2 and values outside the explicitly handled destinations take the fallback path.

Selectors depend on retained state, global observations, saved-base zero tests, arithmetic sign information, and secondary results from F-870c4fcf7295. Without global values and dependency contracts, concrete traversal, reachability, iteration count, and termination remain **documented_only**.

### Captures across opaque calls

A completed scalar capture remains retained even if a later dependency changes its originating global. A later explicit read is a new observation. No dependency is assumed to preserve globals.

The following boundaries are established; helper-call numbers are local to the indicated destination.

| Destination | Capture and consumption boundaries |
|---|---|
| 8: input capture | The next selector is formed from the entry selector-related capture and a fresh mask before call 1. Between calls 1 and 2, the first retained selector state is XOR-updated, one helper state is loaded, the second retained selector state is XOR-updated, and another helper state is loaded, in that order. Both input bases are saved after call 2. |
| 4: symmetric view preparation | One retained state is XOR-updated between calls 1 and 2; another between calls 3 and 4; two more between calls 5 and 6. A final global state is captured after call 6. |
| 16: symmetric reset | The symmetric index is reset and the next selector formed before call 1. One XOR update occurs between calls 2 and 3; two occur between calls 4 and 5. Further XOR updates and global-state loads follow call 5. |
| 13: asymmetric view preparation | A selector state is XOR-updated between calls 1 and 2. Two masks are captured after call 2 and before calls 3 and 4. A third mask is captured after call 4. All three are applied at the shared continuation. |
| 10: asymmetric reset | The asymmetric index is reset and the next selector formed before call 1. Two masks are captured, in order, between calls 1 and 2. Calls 2 and 3 follow. A third mask is captured after call 3; all three are then applied at the shared continuation. |
| 19: first-base field processing | Two selector states are XOR-updated before call 1. A global state is captured after call 2 and before calls 3 and 4; those later calls do not cause it to be refreshed. |
| 3: second-base field processing | Field-derived arithmetic precedes two calls. Subsequent global-state loads and XOR updates occur after both calls. |
| 7: saved-second-base selection | Both calls precede the subsequent XOR updates, global-state loads, and selector-candidate reads. |
| Fallback | Two XOR updates precede call 1. Two global states are captured between calls 2 and 3. Calls 3, 4, and 5 do not refresh those captures. |

For destinations 10 and 13, the shared continuation updates three selector states in a fixed order:

1. The state used by asymmetric byte-condition selection.
2. The state used by destination 27, which clears a retained arithmetic operand.
3. The state used for auxiliary-operation completion.

In destination 10, **the first captured mask is applied to the first state XOR-updated at this continuation; the second captured mask is applied to the second state**. The mask captured after call 3 applies to the third state.

Destination 13 uses the same capture-to-target relationship. Its first two masks are captured after call 2, and its third after call 4.

Each helper call has its own argument preparation and secondary result. Related arguments do not justify combining calls or reusing earlier global observations.

These boundaries do not impose an otherwise unstated evaluation order on separate global occurrences within one expression. They do not establish atomicity, synchronization, or memory barriers.

### Field-dependent processing

Separate destinations read a 32-bit field at offset 1296 from each saved base. The field is multiplied by global coefficients and processed through word arithmetic, shifts, and bitwise combinations. Derived high-bit information affects selector formation.

This is not established as a test of the original field against zero. Coefficients and intermediate widths matter.

The auxiliary destination independently rereads fields at second-base offsets 1300 and 1296. Earlier field reads are not reusable substitutes: intervening dependencies may have changed the memory.

### Views and shared working state

The **symmetric family** prepares first-base offset 236 and second-base offset 32. A separate destination copies the prepared first address into the active first view.

The auxiliary destination instead assigns second-base offset 1040 to that active first view. After its call and result processing, it assigns the retained second-base offset 1304 address to the symmetric second view.

The **asymmetric family** prepares first-base offset 1000 and second-base offset 1020. A later destination copies these into active views. Auxiliary preparation can overwrite its retained first address with second-base offset 1304, so a subsequent copy uses that replacement.

Before its byte reads, symmetric byte processing copies its second view into the active asymmetric second view.

The two families have distinct primary index storage but share other state:

- Each has a destination resetting its index to zero.
- Asymmetric byte processing overwrites the symmetric-index storage with arithmetic derived from its first byte before reading its second byte.
- Intermediate arithmetic values written by one destination can feed later index or result calculations.

Consequently, the families cannot be modeled as independent loops. All views initially contain zero, and no universal preparation-before-use sequence is proven.

### Byte processing and index arithmetic

Both families load unsigned bytes. Values 128 through 255 are not sign-extended.

The first byte read precedes the second. Each address combines the current active view with the relevant index multiplied by a freshly read global stride coefficient. Separate stride reads are not established to return equal values.

For the symmetric family, both bytes are multiplied by 245 and reduced to eight bits.

For the asymmetric family:

- The first byte is multiplied by 57, increased by 116, and reduced to eight bits.
- The second byte is multiplied by 245 and reduced to eight bits.

Each reduced value undergoes further word-sized processing involving multiplication by 93, complementing that product, forcing the complemented word’s low eight bits to one, and combining it with the reduced value and global coefficients.

The symmetric family combines the two transformed values through arithmetic and bitwise operations. A derived high bit selects between secondary results from two ordered helper calls.

The asymmetric family forms the transformed first value minus the transformed second value. Derived sign information selects between retained-state/global combinations.

Established width facts:

- Each reduced byte is between 0 and 255.
- Its explicit signed 16-bit interpretation remains nonnegative.
- Its product with 93 is at most 23,715.
- The subsequent complements, arithmetic, and selector processing are word-sized, not eight-bit operations.

Neither derived condition is established as ordinary byte equality.

Separate destinations perform bounds-like arithmetic involving constants near 20 and 200. They also prepare active views and select between helper results. Other destinations update indices using global-coefficient products and retained intermediates.

These operations do not establish fixed comparison lengths, unit stride, unit increments, monotonic indices, early mismatch exit, or safe access bounds.

### Auxiliary operation

Destination 24 calls F-6c2a60ff6df7 and transforms its return.

Before the call, it captures an adjusted coefficient by XORing a global coefficient observation with a signed-right-shift-by-30 observation of that coefficient’s global. Separate occurrences are not established as one atomic observation. It also prepares retained arithmetic using fresh global reads, distinct from the dispatch-head coefficient snapshot.

It retains second-base offsets 1304 and 1040. Input-relative observations occur in this order:

1. Read the 32-bit field at second-base offset 1300 and complete preparation of the first argument’s value.
2. Read the 32-bit field at second-base offset 1296 while preparing the fourth argument.
3. Invoke the dependency.

| Argument | Established value |
|---:|---|
| 1 | Low-32-bit product of two global coefficients and the offset-1300 field |
| 2 | Retained second-base offset 1304 address |
| 3 | Retained second-base offset 1040 address |
| 4 | Low-32-bit product of the corresponding global coefficients and the later offset-1296 field |
| 5 | Zero |
| 6 | Saved first base |

The coefficient occurrences for arguments 1 and 4 are separate reads, not guaranteed equal observations.

Post-call processing combines the return with pre-call working state, the dispatch-head coefficient snapshot, and the pre-call adjusted coefficient. Neither captured coefficient is refreshed to reflect dependency changes.

The transformation includes signed right shifts by 4, 2, and 1. After those stages, a selector XOR mask is captured from a global. Final result arithmetic then performs additional global reads, including repeated occurrences of a scaling coefficient.

That selector mask remains captured while final result arithmetic executes. The next selector is formed from the mask and retained auxiliary-completion selector state. The auxiliary return is not a direct operand of this selector expression.

Thus a changed dependency return can affect result arithmetic without directly changing the immediate selector. Dependency changes to later-read globals can independently affect the selector or result. The following dispatch iteration starts with its own coefficient read and selector-offset read.

### Integer and address semantics

Inputs, retained addresses, principal working state, and output are 32-bit quantities. Unsigned operations wrap modulo \(2^{32}\); the auxiliary argument products retain their low 32 bits.

Signed interpretations and shifts must be preserved. Signed right shifts propagate the sign; unsigned right shifts insert zeros. A signed shift by 30 is not merely extraction of the sign bit.

The supplied representation does not establish portable signed-overflow behavior for every signed expression. Substitution of unlimited-precision arithmetic, narrower arithmetic, or a different overflow model is not justified.

Address usability, alignment, byte order, and behavior on unusable addresses remain **documented_only**.

## Side effects

The body directly performs global reads, input-relative reads, private working-state updates, dependency calls, and a scalar return.

No explicit input-relative store, global store, allocation, release, reference-count update, or ownership transfer is visible. Retaining an address does not itself modify the target.

Both dependencies are opaque. Neither is proven to preserve globals, pointed-to memory, resource lifetimes, or external state. Passed addresses are not established as read-only. Retention, allocation, and release behavior cannot be inferred.

A dependency cannot retroactively change a captured scalar. It may change a location observed by a later read, or affect the contents and validity of a retained address.

No explicit invalid-address recovery, exception handler, error callback, or dedicated failure return is visible. A nonreturning dependency prevents subsequent processing. Other failures follow the execution environment’s behavior.

## Dependencies

### F-870c4fcf7295

Receives two 32-bit arguments. The first is formed from retained state and fresh global observations, commonly by addition or subtraction. The second is always 2,147,483,647.

The caller consumes a secondary register result associated with each call. The ordinary return is not the evident transition-data source.

The secondary result’s meaning is **documented_only**. It must not be assumed to be a remainder, random value, status, or normalized selector. Distinct results, call order, and intervening observations must be preserved.

### F-6c2a60ff6df7

Receives the six arguments described above and provides a 32-bit return consumed by arithmetic.

Argument meanings, access extents, ownership, retention, global effects, failure behavior, and return interpretation are **documented_only**. The observed products and addresses do not establish copying, comparison, allocation, or release.

## Acceptance examples

These are conditional requirements, not executed tests.

### Entry capture roles

If the second helper-state capture and the intervening selector-state capture have different values, they remain distinct: the selector-state value supplies the state later used by destination 21, not an additional helper seed. Entry capture order remains the six-step sequence above.

### Return selected immediately

If the initial selector and selector-offset observation select destination 17, the dispatch-head coefficient read still precedes the selector-offset read. The initially zero result is returned without input-relative reads by that destination.

### Destination 10 mask order

Suppose the two masks captured between helper calls 1 and 2 are distinct. At the shared continuation, the first mask updates the asymmetric-byte selector state first; the second updates the destination-27 selector state second. They must not be swapped. The post-call-3 mask updates the auxiliary-completion selector state third.

If calls 2 or 3 change the globals supplying the first two masks, the captured values still apply.

### Input capture across calls

If helper call 2 in destination 8 changes a global already read between calls 1 and 2, the completed selector update or helper-state capture is not recomputed. The supplied bases are saved only after call 2 returns.

### Auxiliary snapshots and field order

If the auxiliary dependency changes globals underlying pre-call coefficient captures, subsequent arithmetic still uses those captures. Explicit post-call reads remain separate observations.

The offset-1300 field read precedes the offset-1296 field read, and both precede the dependency call.

If only the dependency return changes while retained selector state and relevant later global observations remain identical, the immediate selector expression remains unchanged. Result arithmetic may differ.

### Unsigned bytes and replaced views

For byte 255, multiplication by 245 followed by eight-bit reduction yields 11. Multiplication by 57, addition of 116, and eight-bit reduction yields 59. Neither load means −1.

If auxiliary preparation replaces the retained asymmetric first address with second-base offset 1304, a later asymmetric view copy uses that replacement, not first-base offset 1000.

### Zero bases

A zero supplied base establishes no universal safe result. Behavior depends on argument capture, selected destinations, retained views, dependencies, and actual accesses.

## Uncertainties

The following remain **documented_only**:

- Global values, concurrency rules, and dependency-mediated mutations.
- Concrete traversal, reachability, preparation-before-use, and termination.
- Dependency contracts and secondary-register semantics.
- Complete layouts, region extents, alignment, byte order, and address validity.
- Concrete stride, index progression, and processing limits.
- Whether transformed relations express equality, ordering, or another property.
- Results for equal, unequal, aliased, zero-filled, or otherwise particular inputs.
- Ownership and lifetime across dependency calls.
- Application-level result meaning and reachable result range.
- A portable signed-overflow interpretation for all represented arithmetic.

These unknowns do not erase established ordering, widths, captures, or state-sharing relationships. They prevent a fully executable behavioral specification without additional evidence.

## Evidence

The supplied static body establishes the entry capture sequence and each capture’s subsequent role, dispatch-head read order, input-relative offsets, unsigned byte loads, byte reductions, retained-view replacement, shared working state, and ordered dependency calls.

It also establishes destination 10’s capture-to-target mask relationship, snapshots spanning helper calls, auxiliary field-read order, pre-call versus post-call coefficient observations, and the absence of direct auxiliary-return dataflow into the immediate selector expression.

No direct input/global writes or explicit ownership operations are visible. Dependency-mediated effects remain unresolved.

Comparison or validation intent is inference only. No runtime tests, execution traces, or dependency measurements have been performed.
