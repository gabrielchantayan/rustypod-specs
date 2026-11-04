<!-- Trusted provenance: F-ad81d5a8fb8d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ad81d5a8fb8d

## Purpose

F-ad81d5a8fb8d transforms two word-sized values using a four-word parameter region, eight iterations of arithmetic and bitwise mixing, and two externally stored arithmetic parameters. It passes the first resulting word to an output dependency and explicitly stores the second resulting word as four bytes in most-significant-byte-first order.

The structure suggests a reduced-round, TEA-like inverse transformation. That is inferred intent, not an established cryptographic contract. The supplied evidence does not establish the arithmetic-parameter values, input byte order, first-word output encoding, or compatibility with a named algorithm.

This contract distinguishes:

- **Established behavior:** operations and sequencing visible in the supplied representation.
- **Inferred interpretation:** a 32-bit word model and block-oriented cryptographic purpose.
- **documented_only unknowns:** dependency behavior, external values, and machine details absent from the evidence.

No runtime tests have been performed.

## Interface

The function has three arguments and no explicit return value:

1. **Parameter-region reference.** A reference to signed-integer elements. The function directly reads elements at indices 0 through 3. Under a four-byte integer layout, these occupy byte offsets 0, 4, 8, and 12.
2. **Input address value.** An integer-valued address passed to F-d36d6556e6bc, first unchanged and then increased by 4.
3. **Output address value.** An integer-valued address passed to F-977f3c45daf8. The function subsequently writes individual bytes at offsets 4, 5, 6, and 7 from this address.

The supplied representation uses signed integers for the address arguments, unsigned integers for the two transformed values, and a signed integer for the running arithmetic state. It does not supply an ABI or type definitions.

The behavioral equations below use the likely target interpretation:

- unsigned words are 32 bits;
- unsigned arithmetic wraps modulo \(2^{32}\);
- a byte is 8 bits;
- signed-integer elements occupy four bytes.

These widths are strongly suggested by the shifts, constants, and output layout, but are not independently established by the excerpt. Behavior on a different-width target must not be assumed equivalent.

### Preconditions

Successful execution requires all of the following:

- The parameter-region reference permits the integer reads actually performed.
- Both calls to F-d36d6556e6bc satisfy that dependency’s input requirements.
- F-977f3c45daf8 accepts the supplied output address and result value.
- Output offsets 4 through 7 permit byte writes.
- The two external arithmetic-parameter locations can be read.
- Address arithmetic and signed arithmetic have valid target semantics.

There are no explicit null, alignment, length, overlap, or address-range checks. The function has no size arguments and exposes no error result.

An eight-byte readable input region and eight-byte writable output region may be the intended interface, but the exact access footprint of either dependency is unknown. Those region sizes are therefore not sufficient, by themselves, to establish every precondition.

## Behavior

### Input acquisition and initial state

The observable call sequence starts with:

1. F-d36d6556e6bc receives the input address. Its return value becomes the first working word.
2. F-d36d6556e6bc receives the input address plus 4. Its return value becomes the second working word.

The first call completes before the second begins. The initial external arithmetic parameter is read after both calls have completed. Consequently, a change to that parameter made by either call can affect the transformation.

The function does not visibly obtain either input word through a direct load. Input byte order, access width, alignment handling, and any additional effects belong to F-d36d6556e6bc and remain unknown.

After these calls, the function sets an iteration count of eight and initializes the running arithmetic state from an external location.

### Mixing rule

For description, designate the four parameter elements as \(K_0\) through \(K_3\), in increasing index order. These names identify locations, not values guaranteed to have been snapshotted.

Under the 32-bit interpretation, define the mixing expression

\[
M(x,p,q,s)
=
\bigl(p + 16x\bigr)
\mathbin{\oplus}
\bigl(x+s\bigr)
\mathbin{\oplus}
\bigl(q+(x\mathbin{\mathrm{logical}\gg}5)\bigr).
\]

Each addition and multiplication in this expression uses unsigned 32-bit arithmetic after the applicable integer conversions. Each of the three parenthesized terms is formed before the XOR combination. The right shift of the unsigned working word is logical, introducing zeros at the high end.

Each of the eight iterations performs the following state transition:

- The second working word is reduced by the mixing expression formed from the current first working word, parameter elements 2 and 3, and the current running state.
- The iteration counter is decremented.
- The first working word is reduced by the mixing expression formed from the **new second working word**, parameter elements 0 and 1, and the same running state.
- The external decrement parameter is read and subtracted from the running state.
- The loop continues only if the decremented counter is greater than zero.

Thus, the second-word update precedes the first-word update in every iteration. The first-word update must not use the second word from the start of that iteration.

Both working-word subtractions wrap modulo \(2^{32}\) under the stated interpretation. Negative signed parameter elements are converted as required by their participation in unsigned expressions; they are not treated as arbitrary-precision negative operands.

### Running arithmetic state

The running state is initialized once. Each iteration uses its current value for both word updates, then subtracts the external decrement parameter.

If the initial value is \(S\), the decrement remains \(D\), and target arithmetic behaves as wrapping 32-bit arithmetic, the values used by successive iterations are:

\[
S,\ S-D,\ S-2D,\ \ldots,\ S-7D.
\]

The subtraction after the eighth iteration is still present, even though its result is not subsequently used by the visible transformation.

The excerpt represents this running state as signed. It does not establish that signed overflow wraps. Accordingly, the formula above is conditional on valid arithmetic or the target’s wrapping behavior; it is not a claim that portable signed arithmetic permits overflow.

The decrement is referenced during each iteration rather than explicitly copied into a local snapshot before the loop. Its numeric value and stability are not established.

### Parameter reads and evaluation order

The parameter region is not explicitly copied before processing. Each iteration contains reads of elements 2 and 3 for the second-word update, followed by reads of elements 0 and 1 for the first-word update.

The expression does not establish the relative evaluation order of the two parameter-element reads within one update. A contract requiring, for example, element 2 to be read before element 3 would exceed the evidence.

There are no visible calls inside the loop. The function itself does not directly write the parameter region or either external arithmetic parameter.

The source-level references do not establish volatile access, atomicity, or a physical memory-load count after compilation. Concurrent mutation has no defined synchronization contract here.

### Output emission

After eight iterations, the function calls F-977f3c45daf8 with:

- the output address; and
- the final first working word.

Only after that call returns does it explicitly store the final second working word at output offsets 4 through 7:

| Output offset | Stored bits of the second word |
|---|---|
| 4 | 31 through 24 |
| 5 | 23 through 16 |
| 6 | 15 through 8 |
| 7 | 7 through 0 |

These four byte stores occur in increasing offset order in the supplied representation. Under the eight-bit-byte interpretation, they encode the second word in big-endian order.

The function then returns without an explicit value. There is no visible success flag or result comparison.

The evidence does not establish how F-977f3c45daf8 represents the first word, whether it writes exactly four bytes, or whether it touches any other memory.

## Side effects

The function directly performs four byte writes at output offsets 4 through 7. Other memory effects are delegated to the two dependencies.

The explicit processing order is important:

- Both input calls complete before the arithmetic loop.
- All visible parameter reads used by the transformation finish before the output call.
- The output call completes before the four trailing-byte stores.

This ordering does not establish unrestricted overlap safety. For example, F-d36d6556e6bc might modify memory that the second call or the loop subsequently reads. Likewise, F-977f3c45daf8 might write beyond the first word. If it writes output offsets 4 through 7 and returns normally, the later explicit stores overwrite those locations.

No allocation, release, ownership transfer, locking, or direct external-state write is visible in the body. This observation does not establish that the dependencies refrain from such actions. Their ownership and global-state behavior remain unknown.

There is no rollback. If the output dependency writes memory and then fails to return normally, the explicit trailing-byte stores are not reached. If a later byte store faults, effects of the dependency and earlier completed stores may already be externally visible.

No explicit error handling or recovery path is present. Invalid accesses and dependency failures are not translated into a defined status.

## Dependencies

### F-d36d6556e6bc

Called twice, serially, with the input address and input address plus 4. Each return value initializes one unsigned working word.

**Established:** call order, argument relationship, and use of the returned values.

**documented_only:** return-width details beyond the receiving type, byte order, read footprint, alignment requirements, failures, ownership effects, and changes to globals or caller-accessible memory.

### F-977f3c45daf8

Called once after the loop with the output address and the final first working word.

**Established:** its position before the four explicit byte stores. No returned value is used.

**documented_only:** encoding, write footprint, alignment requirements, failures, ownership effects, and changes to globals or caller-accessible memory.

### External arithmetic parameters

Two external values participate:

- an initial running-state value, read after both input calls;
- a decrement value, referenced at the end of each iteration.

Their numeric values, declared storage types, mutability, and synchronization properties are absent. They must not be replaced with constants from a presumed algorithm.

## Acceptance examples

These examples are conditional behavioral checks, not observed runtime results.

### Zero-valued transformation

Assume 32-bit wrapping word arithmetic, both input calls return zero, all four parameter elements are zero whenever read, and both external arithmetic parameters are zero.

Both working words remain zero for all eight iterations. The output dependency receives zero as the first-word result. After it returns, output offsets 4 through 7 each receive a zero byte.

This does not establish the final contents of output offsets 0 through 3.

### Unsigned underflow

Assume the first working word begins at zero, the second begins at zero, the running state and decrement are zero, parameter element 2 is one, and parameter element 3 is zero.

The first second-word update subtracts one from zero. Under the 32-bit interpretation, its result is \(2^{32}-1\), not a saturated zero and not an error. The following first-word update uses that updated value.

### Trailing-word byte order

If the final second working word is `0x12345678` and the output dependency returns normally, the explicit stores place `0x12`, `0x34`, `0x56`, and `0x78` at output offsets 4, 5, 6, and 7 respectively.

This example says nothing about how the first word is encoded.

### Dependency changes to initialization

If the second input call changes the external initial-state value before returning, initialization observes the value available at the later initialization read, subject to the target’s memory semantics. The function does not visibly snapshot that value before calling the dependencies.

### Output failure ordering

If F-977f3c45daf8 does not return normally, none of the four subsequent explicit byte stores is reached. If it returns normally, those stores are reached in offset order unless execution is interrupted by a fault or another external event.

## Uncertainties

The following remain **documented_only**:

- Exact ABI widths and address representation.
- Numeric values and declared types of both external arithmetic parameters.
- Signed-overflow behavior of the running-state subtraction.
- Validity of signed address additions near representational limits.
- Input decoding and first-word output encoding.
- Dependency access footprints, failures, ownership effects, and global mutations.
- Guarantees for overlapping regions.
- Behavior under concurrent mutation, unusual mappings, or access faults.
- Cryptographic identity, intended security properties, and correspondence to a complete block-processing protocol.

The evidence supports a precise conditional 32-bit transformation, but not a fully instantiated input-to-output byte mapping without these missing definitions.

## Evidence

The supplied static representation establishes the two ordered input calls, four parameter-element references per iteration, eight iterations, second-word-before-first-word updates, running-state initialization and decrements, one output call, and four explicit trailing-byte stores.

It also establishes the absence of explicit validation, error returns, allocation, release, and recovery logic in this function body.

The 32-bit interpretation and cryptographic characterization are inferences from the arithmetic structure and output layout. Dependency contracts and external values were not supplied. No runtime execution, dependency inspection, or test results are available.
