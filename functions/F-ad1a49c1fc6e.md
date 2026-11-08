<!-- Trusted provenance: F-ad1a49c1fc6e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ad1a49c1fc6e

## Purpose

F-ad1a49c1fc6e describes a repeating sequence of indirect calls and address-derived memory writes. Its initial action updates a carried accumulator and stores that result through a supplied destination. Subsequent operations obtain paired results from dependencies, interpret those results as both addresses and data, and conditionally repeat either the inner processing sequence or the initial accumulator update.

The supplied representation establishes no normal return path. Execution can continue indefinitely, encounter a fault, enter a dependency that does not return, or reach a software breakpoint whose continuation behavior is unknown.

No higher-level purpose—such as initialization, allocation, device reset, or record construction—is established. In particular, the name of the global byte written by this function does not establish the meaning of that byte.

## Interface

### Declared inputs

The visible interface contains three arguments:

- A 32-bit opaque argument with no explicit use in the supplied body.
- A destination interpreted as a pointer to a 32-bit integer.
- A 32-bit unsigned token.

The absence of an explicit use of the first argument does not establish that the original executable ignores it across undocumented calling conventions or hidden-state interactions.

### Additional incoming state

The visible body also consumes state not represented as declared arguments:

- A carried accumulator, used in addition and later as the basis of an address.
- A carried base value, used to derive several write addresses.
- A carried cursor, interpreted as a pointer to a 32-bit unsigned value and later as integer data.
- A carried continuation value, used to derive the first indirect call target.

These values are used before the supplied body establishes them. The visible three-argument interface is therefore insufficient to invoke the function safely or reproduce its behavior.

The contract below calls these quantities the **accumulator**, **base**, **cursor**, and **continuation**. These are descriptive roles, not claims about an original public interface.

### Results

There is no explicit return value and no visible normal return.

Observable results consist of:

- Memory stores through caller-supplied and computed addresses.
- Reads through the carried cursor and dependency-derived addresses.
- Writes to a global byte, called the **control byte** here.
- Dependency invocations and their arguments.
- A software breakpoint with immediate value 4.
- Changes to carried state used by later iterations.

### Width and layout assumptions

The representation supports the following working interpretation:

| Quantity | Width or interpretation |
|---|---|
| Token and ordinary word stores | 32 bits |
| Halfword stores | 16 bits |
| Byte stores and control byte | 8 bits |
| Paired dependency results | 64 bits, accessed as low and high 32-bit halves |
| Address-bearing integers | Apparently 32 bits |
| Three-word output region | Words at byte offsets 0, 4, and 8 |
| Full paired-result store | 8 bytes |

The representation does not establish byte order, alignment rules, pointer validity, or whether the displayed integer operations faithfully express the underlying instruction semantics. Where those details affect behavior, they remain documented-only uncertainties.

## Behavior

The ordering below describes explicit operations in the supplied body. Dependency-internal accesses are not included and can occur between the listed operations.

### Initial update and restart point

On initial entry, the function:

1. Adds the current token to the accumulator.
2. Stores the resulting accumulator as a 32-bit value through the current destination.

A later branch can return to this same update point. On that branch, the destination, accumulator, and token have already been replaced by values obtained or derived during processing. The initial update is therefore not limited to the original caller-supplied destination.

The exact treatment of overflowing addition is not established by the mixed signed and unsigned representation. A 32-bit machine interpretation would retain the low 32 bits, but that interpretation requires confirmation from instruction-level evidence.

### Start of a processing pass

Each processing pass begins with a zero argument prepared for F-00000001.

At the beginning of every inner iteration, the function saves the low eight bits of the current token. This saved byte is used later in the same iteration, even though the token is subsequently replaced.

It then performs these operations in order:

1. Calls F-00000001 with one explicit argument: the current pass argument.
2. Treats the low 32 bits of the returned pair as an address and stores the low 16 bits of that same returned value at that address.
3. Multiplies the carried base by 64, uses the product as an address, and stores that product there as a 32-bit word.
4. Reads a 32-bit token through the current cursor.
5. Stores a 16-bit value at base plus 4. The value is the cursor’s low 16 bits plus 4, truncated to the halfword width.
6. Replaces the cursor by loading a pointer-sized value from the first result’s high-half address plus 80.
7. Calls F-00000002 with no explicit arguments.
8. Calls F-00000003 with no explicit arguments.
9. Calls F-00000004 with the token read in step 4.
10. Replaces the carried continuation with a fixed internal continuation value.
11. Calls F-00000005 with the token shifted right logically by 3.

The first call argument is zero after entry into a processing pass. If execution reaches the breakpoint continuation described below and resumes the inner sequence without starting a new pass, that argument has a different, computed value.

“No explicit arguments” is a statement about the supplied representation. It does not prove that a dependency consumes no hidden state.

### Classification of the second paired result

The high 32 bits returned by F-00000005 become the **record address**.

The low 32 bits are multiplied by 32 to form the **scaled address**. The function immediately stores the scaled address as a 32-bit word at the scaled address.

Only after that store does it compare the scaled address with unsigned decimal 20.

If the scaled address is below 20, the inner loop ends and a new processing pass begins. This restart:

- Resets the first-call argument to zero.
- Does not repeat the initial accumulator update or destination store.
- Retains the other carried state as left by the preceding operations.

Under a wrapping 32-bit multiplication interpretation, the scaled address is always a multiple of 32, so “below 20” is equivalent to zero. This is not a protective null check: the store through that address occurs before the comparison.

If the scaled address is not below 20, processing continues.

### Extended writes and third paired result

On the non-small-address branch, the function:

1. Stores the current base as a 32-bit word at scaled address plus 88.
2. Stores the cursor’s low 16 bits at base plus 14.
3. Replaces the token with the base shifted left by 2.
4. Writes the previously saved byte to the global control byte.
5. Calls F-00000006 with the cursor’s integer representation shifted right logically by 2.
6. Replaces the carried continuation with another fixed internal continuation value.
7. Calls F-00000007 with no explicit arguments.

The high 32 bits of F-00000007’s result replace the destination. The low 32 bits become the **object address**.

The following memory operations occur in this order:

1. Store the saved byte at twice the object address.
2. Store the low eight bits of the new destination at object address plus 8.
3. Store the record address as a pointer-sized value at twice the current accumulator.
4. Store the complete 64-bit result from F-00000007 at the object address.

The fourth store writes the same paired result from which the object address and destination were obtained. These addresses are not independently validated.

### Record population and carried-state replacement

After those writes, the function replaces:

- The accumulator with the record address shifted left by 2.
- The base with the cursor’s integer representation shifted left by 8.

It then writes three consecutive 32-bit words at the record address:

| Byte offset | Stored value |
|---|---|
| 0 | Current token |
| 4 | Newly derived base |
| 8 | Cursor interpreted as a 32-bit integer |

The current token was derived from the earlier base, before the base replacement. The first record word therefore does not use the newly shifted cursor-derived base.

The function next calculates a **branch value** using a problematic shift sequence: the cursor is interpreted as a signed integer, shifted left by 31, shifted right by 32, converted to an unsigned quantity, and then shifted right by 20.

The exact result is not established. The displayed operation shifts a nominally 32-bit quantity by 32 and also involves signed shifting. No portable arithmetic interpretation can safely be imposed on this expression.

### Zero branch

If the branch value is zero:

1. The control byte is explicitly written as zero.
2. Execution returns to the initial update point.

At that point:

- The destination is the high half returned by F-00000007.
- The accumulator is the shifted record address.
- The token is the earlier base shifted left by 2.

The next explicit operations are the accumulator addition and a 32-bit store through that replacement destination.

### Nonzero branch and breakpoint

If the branch value is nonzero:

1. The branch value is stored as a 32-bit word at token multiplied by 256, plus 32.
2. The carried continuation is replaced with a third fixed internal continuation value.
3. A new first-call argument is calculated by interpreting token multiplied by 256 as signed and shifting it right by 29.
4. Software breakpoint 4 is executed.
5. If execution continues, the base is replaced by the signed interpretation of the destination shifted right by 18.
6. The inner processing sequence repeats.

This route does not perform the initial accumulator update. It also does not reset the first-call argument to zero before the next F-00000001 invocation.

Breakpoint handling, signed-right-shift behavior, and resumability are unknown.

## Side effects

### Memory ordering and aliasing

The body supplies no disjointness guarantees. The destination, cursor, derived addresses, record, object, and control byte may overlap.

Consequently:

- An earlier store may alter a later cursor read.
- The halfword store at base plus 4 may affect the subsequent cursor replacement if their address ranges overlap.
- The byte write at twice the object address may overlap the later 64-bit object store.
- Record writes may overwrite earlier stores.
- A dependency may observe memory written before its invocation.
- A dependency may change memory read after its invocation.

Any consumer relying on non-overlap must establish that condition externally. The explicit write order must not be replaced with a simultaneous-update interpretation.

### Ownership and lifetime

No allocation, ownership transfer, reference acquisition, or release is established for the function itself.

The function treats dependency results as usable addresses, but that does not establish that those addresses are owned, live, writable, or suitably aligned. No ownership behavior may be attributed to a dependency without additional evidence.

### Errors and abnormal completion

The body contains no explicit error return, bounds check, recovery mechanism, or rollback.

Potentially failing operations include every computed memory access, indirect call, and breakpoint. If execution stops partway through an iteration, earlier completed writes are not visibly undone.

The small-scaled-address branch does not prevent the preceding potentially invalid store. Likewise, the record address is written through without a preceding validity test.

## Dependencies

The following opaque IDs are analytical labels for distinct call sites. They do not assert distinct implementations or membership in a verified dependency catalog.

| ID | Visible invocation and use |
|---|---|
| F-00000001 | Indirect target derived from the carried continuation. Receives the current pass argument; returns a 64-bit pair used for an immediate halfword store and cursor replacement. |
| F-00000002 | Fixed target derived through an internal constant; no explicit arguments or consumed return value. |
| F-00000003 | Another fixed derived target; no explicit arguments or consumed return value. |
| F-00000004 | Fixed derived target; receives the cursor-loaded token. |
| F-00000005 | Directly represented call; receives the token shifted right by 3 and returns the scaled-address source and record address. |
| F-00000006 | Fixed derived target at this call site; receives the cursor representation shifted right by 2. |
| F-00000007 | Fixed derived target; returns the object address and replacement destination. |

For every dependency, global effects, register effects beyond the represented dataflow, ownership behavior, failure behavior, and return guarantees are unknown. In particular, F-00000006 may change the control byte after the caller writes it.

## Acceptance examples

These are conditional static examples, not runtime results.

### Initial store

Given accumulator 7, token 5, and a valid writable destination, reaching the initial update without arithmetic complications stores the 32-bit value 12 through that destination before the first dependency invocation.

### Saved-byte behavior

Given an inner iteration starting with token hexadecimal 123456AB, the saved byte is hexadecimal AB.

If the cursor subsequently supplies token 16, F-00000004 receives 16 and F-00000005 receives 2. If the extended branch is taken, the caller’s control-byte assignment still writes hexadecimal AB, not hexadecimal 10.

This does not guarantee the control byte remains AB after later dependencies.

### Small-address ordering

If F-00000005 returns a low half of zero, the scaled address is zero. The next explicit action attempts to store a zero word at address zero.

If that access succeeds, the small-address branch starts a new pass. If it faults, the branch is not reached. An implementation that skips the store because the address is zero would not preserve the visible ordering.

### Record layout

Suppose the extended branch is reached with base 3 and cursor representation 256, and all required accesses and calls complete without altering the relevant carried values.

The token becomes 12. The replacement base becomes 65,536. The record receives, in order:

- Offset 0: 12.
- Offset 4: 65,536.
- Offset 8: 256.

This example deliberately makes no prediction about the later ambiguous branch calculation.

### Aliasing of object writes

If the object address is 8, the byte write at twice that address and the byte write at object address plus 8 target the same byte, address 16.

The destination-derived byte is written second and therefore supersedes the saved byte at that location, absent intervening external modification. The later 64-bit store begins at address 8 and covers bytes 8 through 15.

### Conditional restart

If the branch value evaluates to zero under the actual instruction semantics, the caller writes zero to the control byte and resumes at the accumulator update, not merely at the first indirect call.

If it evaluates to nonzero, the caller performs the additional word store and reaches breakpoint 4 instead.

## Uncertainties

The following remain **documented_only**:

- The intended purpose and valid invocation environment.
- The actual calling convention and initialization of carried state.
- Exact instruction semantics for overflowing arithmetic, signed left shifts, and especially the right shift by 32.
- Endianness, address width confirmation, alignment requirements, and memory permissions.
- Whether computed addresses refer to ordinary memory, mapped devices, or invalid regions.
- Dependency identity, effects, ownership, and return behavior.
- Whether computed call targets are valid in the required execution mode.
- Breakpoint handling and whether execution resumes afterward.
- Synchronization, interrupts, concurrent mutation, and access atomicity.
- Any supported termination condition outside the visible body.

A record-building or low-level control routine is a possible interpretation, not an established purpose.

## Evidence

This contract is based solely on the supplied static representation.

Established static evidence includes the explicit call ordering, access widths, address relationships, paired-result extraction, control-byte assignments, repeated control-flow paths, and absence of a visible normal return.

The 32-bit address model and wrapping arithmetic interpretations are identified where used rather than treated as universally established. The ambiguous shift calculation is intentionally unresolved.

No implementation was accessed independently. No runtime tests, dependency execution, fault observations, or breakpoint-resumption experiments have been performed.
