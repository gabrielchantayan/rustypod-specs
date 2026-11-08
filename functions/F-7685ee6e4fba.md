<!-- Trusted provenance: F-7685ee6e4fba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7685ee6e4fba

## Purpose

F-7685ee6e4fba writes one to four integer results by summing selected words from caller-provided input storage. An unsigned selector determines the selected words and the number of output assignments.

For selectors 1 through 4, output position \(j\), counted from zero, sums the input words whose positive index has bit \(j\) set. The input word at index zero does not participate.

The actual branch behavior includes two important boundary cases:

- Selector 0 takes the four-output path, just like selector 4.
- Every selector greater than 4 first invokes F-540af0155702. If that invocation returns normally, the function proceeds to the four-output path.

**Established static evidence:** the selector comparisons, conditional dependency invocation, metadata argument reads, input offsets, sum membership, and output assignment order are visible in the supplied representation.

**Inferred intent:** the calculation resembles aggregation over nonempty subsets, with the selector describing a dimension. The diagnostic dependency appears intended to report an excessive selector. Neither interpretation establishes application-level meaning or guarantees that an excessive selector is rejected.

## Interface

### Arguments

The function receives:

1. **Input base:** an integer-valued address used to locate input words at positive byte offsets. The base value itself is not a numerical operand in the sums.
2. **Output address:** a pointer to consecutive integer result words.
3. **Selector:** an unsigned integer used for the upper-bound check and path selection.

The representation is consistent with four-byte signed integer data words and a 32-bit unsigned selector. This contract uses that interpretation for word layouts and numeric examples. An independently verified ABI, pointer width, and complete target integer model were not supplied.

Under the 32-bit unsigned-selector interpretation, converting a negative signed value to the selector type produces a large unsigned value. Such a value takes the greater-than-4 path; it does not behave like selector 0.

### Input and output layout

Let \(X_k\) denote the integer word at byte offset \(4k\) from the input base. Let \(Y_j\) denote the output word at byte offset \(4j\) from the output address.

| Selector or continuation | Input words used | Highest input word start offset | Output words written |
|---|---|---:|---:|
| 1 | \(X_1\) | 4 | 1 |
| 2 | \(X_1\) through \(X_3\) | 12 | 2 |
| 3 | \(X_1\) through \(X_7\) | 28 | 3 |
| 0 or 4 | \(X_1\) through \(X_{15}\) | 60 | 4 |
| Greater than 4, after normal dependency return | \(X_1\) through \(X_{15}\) | 60 | 4 |

A four-byte word beginning at offset 60 occupies offsets 60 through 63. Conventional contiguous input allocations of 8, 16, 32, or 64 bytes from the base are therefore sufficient for the respective paths, provided the required words are readable and address calculations are valid. This does not mean that every byte in those extents is accessed.

The required writable output extents are respectively 4, 8, 12, and 16 bytes. Words beyond the selected output extent are not directly written by this function.

The input word at offset zero is not selected by any calculation. This is a statement about the function’s explicit input-word accesses, not about possible accesses by F-540af0155702.

### Result and preconditions

No return value or status code is represented. Normal completion communicates results through output storage.

For ordinary defined execution:

- Each selected input word must be readable when its assignment evaluates it.
- Each output word must be writable when stored.
- Base-plus-offset arithmetic and conversion to usable addresses must be valid under the execution environment.
- Integer accesses must meet the environment’s alignment requirements.
- Every evaluated signed addition must be representable unless target-specific overflow behavior has been established separately.
- For selectors greater than 4, both global metadata locations used as dependency arguments must also be readable when evaluated.

There are no explicit null, bounds, alignment, or overflow checks. There is no represented validation of the two metadata values.

## Behavior

### Selector handling and metadata reads

For selectors 0 through 4, the function does not invoke F-540af0155702 and does not perform the two global metadata reads associated with that invocation.

For a selector greater than 4, the dependency receives three arguments:

- A reference to diagnostic string storage.
- A value read from one global metadata location.
- A value read from a second global metadata location.

The two metadata locations are statically identified storage locations. Their contents are **not established constants**. Nothing supplied establishes their immutability, particular values, or semantic roles.

Both global reads occur during argument evaluation, before the dependency invocation. Their relative evaluation order is not established by the supplied representation. The contract does not require either metadata location to be read first or promise that the two values form an atomic snapshot.

Passing the diagnostic-string reference does not itself establish that this function reads the string’s contents. Any inspection of those contents by the dependency belongs to that dependency’s unknown behavior.

The dependency invocation precedes this function’s explicit input-data reads and output assignments. If argument evaluation cannot complete, the supplied representation establishes no completed dependency invocation or subsequent calculation.

If F-540af0155702 does not return normally, the calculation continuation is not reached. If it returns normally, the selector still selects the four-output path. There is no represented clamping, replacement, or recovery status.

### Output values

For nonoverlapping input and output storage, no relevant intervening mutation, and representable arithmetic, the results are:

| Selector | Results |
|---|---|
| 1 | \(Y_0=X_1\) |
| 2 | \(Y_0=X_3+X_1\), \(Y_1=X_3+X_2\) |
| 3 | \(Y_0=X_7+X_5+X_3+X_1\), \(Y_1=X_7+X_6+X_3+X_2\), \(Y_2=X_7+X_6+X_5+X_4\) |
| 0 or 4 | The four results below |
| Greater than 4, after normal dependency return | The four results below, using the input state reached after that invocation |

The four-output results are:

\[
Y_0=X_{15}+X_{13}+X_{11}+X_9+X_7+X_5+X_3+X_1
\]

\[
Y_1=X_{15}+X_{14}+X_{11}+X_{10}+X_7+X_6+X_3+X_2
\]

\[
Y_2=X_{15}+X_{14}+X_{13}+X_{12}+X_7+X_6+X_5+X_4
\]

\[
Y_3=X_{15}+X_{14}+X_{13}+X_{12}+X_{11}+X_{10}+X_9+X_8
\]

There is no weighting, division, normalization, saturation, or contribution from \(X_0\). Selector 1 copies one selected word without performing an addition.

### Access ordering and overlap

Output assignments occur in increasing index order: \(Y_0\), then \(Y_1\), then \(Y_2\), then \(Y_3\), stopping at the selected output count.

Each assignment obtains its input operands before storing that assignment’s result. Later assignments evaluate their own selected input locations after the preceding output assignment has completed. No complete input snapshot or temporary input copy is represented.

The order of terms in a displayed sum does not establish a temporal order for its individual memory reads. The representation establishes the separation between successive assignments, not an exact machine-load sequence within one assignment.

No overlap rejection is shown. If output storage overlaps selected input storage, an earlier output assignment can change a word used by a later assignment. Consequently, the equations identify the locations used by each assignment; they do not universally describe entry-time input values.

For example, with selector 2 and the output base at the location of \(X_2\), storing \(Y_0\) replaces \(X_2\) before the second assignment reads it. Whether an overlapping arrangement is otherwise valid depends on the environment’s memory-access rules.

### Arithmetic boundaries

The results occupy four-byte integer words under the stated width interpretation. No explicit wider accumulator is represented.

For signed 32-bit data, the mathematical equations apply when all evaluated intermediate sums lie between \(-2^{31}\) and \(2^{31}-1\). A final mathematical sum within that interval is not by itself sufficient if an intermediate addition overflows.

The supplied representation does not establish a portable result for signed overflow. Modulo-\(2^{32}\) wrapping, saturation, exceptions, or a reported arithmetic error must not be promised.

Address arithmetic also requires care: the input base is represented as an integer-valued argument, with positive offsets added before the input accesses. Valid backing storage alone does not establish that every such integer calculation and address interpretation is valid.

## Side effects

For selectors 0 through 4, the only explicit state changes are the selected output writes. Input storage changes directly only where output storage overlaps it.

For selectors greater than 4, the function additionally reads two global metadata locations during dependency argument evaluation and invokes F-540af0155702. Those reads are conditional on the unsigned upper-bound check; they are not unconditional function-entry reads.

The dependency’s side effects are unknown. It may modify input storage, output storage, the metadata locations, other globals, or control flow. Normal return does not establish preservation of any such state.

This function shows no direct allocation, deallocation, ownership transfer, or persistent-state update beyond output storage. The caller supplies storage usable for the required accesses. Passing the diagnostic reference establishes neither ownership transfer nor a guarantee that the dependency retains or releases nothing.

Execution is not transactional. Earlier output assignments precede later assignments, and no rollback is represented. If execution stops after some stores, the contract supplies no rollback guarantee for those stores.

No synchronization or atomicity mechanism is shown.

## Dependencies

### F-540af0155702

**Invocation condition:** the selector is greater than unsigned value 4.

**Argument sources:**

1. A diagnostic-string reference.
2. The value obtained by reading one global metadata location.
3. The value obtained by reading another global metadata location.

The second and third arguments are sampled values, not established fixed constants. Their source locations are identified statically, but their contents, widths, signedness, mutability, and interpretation are not sufficiently established to define a complete external interface.

**Ordering:** the two global reads occur during argument evaluation before the invocation. Their relative order is unspecified by the supplied representation. The invocation occurs before the caller’s explicit input-word calculations and output stores.

**Return handling:** no return value is used. A normal return permits the four-output calculation to proceed. Nonreturning behavior prevents that continuation.

**Unknown behavior:** logging, termination, exceptions or other control transfer, memory access, global changes, retention of references, and ownership-related actions are not established.

The diagnostic association supports an inference that the dependency reports a failed selector constraint. It does not prove termination or rejection.

## Acceptance examples

These are conditional deductions from static evidence, not runtime observations. Unless otherwise stated, assume valid nonoverlapping storage, representable arithmetic and address calculations, and no relevant concurrent mutation.

### Ordinary selectors

With \(X_k=k\) for indices 1 through 15:

| Selector | Expected output words |
|---|---|
| 1 | \(1\) |
| 2 | \(4,\ 5\) |
| 3 | \(16,\ 18,\ 22\) |
| 4 | \(64,\ 68,\ 76,\ 92\) |
| 0 | \(64,\ 68,\ 76,\ 92\) |

Selector 0 writes four words and does not invoke the dependency.

### Unused leading word and output extent

With selector 3, changing only \(X_0\) does not change the three results.

With selector 2 and at least three writable output words, the function assigns the first two words. It does not directly write the third.

### Sequential overlap

Assume selector 2, initial values \(X_1=1\), \(X_2=2\), \(X_3=3\), and an output base at \(X_2\).

The first result is \(4\), replacing \(X_2\). The second assignment then uses that updated value and produces \(7\), replacing \(X_3\).

The output pair is therefore \(4,7\), not the nonoverlapping result \(4,5\), subject to valid overlapping accesses.

### Conditional metadata access

With selector 4, neither global metadata location is read for dependency argument evaluation, and no dependency invocation occurs.

With selector 5, suppose the first metadata location contains value \(A\) and the second contains value \(B\), both remaining stable during argument evaluation. The dependency receives the diagnostic reference, \(A\), and \(B\) in their respective argument positions. Either metadata read may occur first.

If those locations instead contain different stable values on another invocation, those new values are passed. Their fixed locations do not make their contents constant.

### Excessive selector and continuation

With selector 5, suppose argument evaluation succeeds and the dependency returns normally without changing relevant input or output data. For \(X_k=k\), the function writes \(64,68,76,92\).

If the dependency changes relevant input values before returning, the original values no longer determine the results. If it does not return normally, the function does not reach its subsequent output assignments.

## Uncertainties

The following remain **documented_only**:

- Independently verified ABI, pointer width, byte order, alignment requirements, and exact target integer model.
- Behavior of overflowing additions or invalid address arithmetic.
- Exact machine-load ordering within an output assignment.
- Exact ordering of the two metadata argument reads.
- Metadata value widths, signedness, contents, mutability, and semantic roles.
- Full diagnostic-string contents and the dependency’s complete interface.
- Whether the dependency returns normally, modifies globals or buffers, retains references, or performs ownership-related actions.
- Behavior for inaccessible, misaligned, or insufficient storage.
- Atomicity, synchronization, and effects of concurrent access.
- Application-level interpretation of the integer values.

The four-byte input offsets and output spacing are established by the representation. The subset-aggregation interpretation remains inferred intent.

## Evidence

The supplied static representation establishes the unsigned upper-bound check, conditional dependency invocation, diagnostic reference argument, two global value arguments, selector branches, selected input offsets, sequential output assignments, and absence of a returned result.

The global argument expressions establish conditional reads before invocation, not constant metadata contents or a relative read order. The continuation establishes that selector 0 reaches the four-output path directly and that selectors greater than 4 can reach it after a normal dependency return.

No runtime tests have been performed. No dependency implementation, independent machine-level verification, or external ABI documentation was supplied. The acceptance examples are deductions under their stated conditions, not observed executions.
