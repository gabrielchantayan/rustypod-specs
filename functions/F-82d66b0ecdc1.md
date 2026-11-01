<!-- Trusted provenance: F-82d66b0ecdc1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-82d66b0ecdc1

## Purpose

F-82d66b0ecdc1 reads four consecutive 32-bit input words, obtains one integer result per word from F-0ef463575bea, and converts those results into four consecutive destination bytes.

The operation has two ordered phases:

1. Four input reads interleaved with four dependency calls.
2. Four result conversions and destination-byte assignments, in corresponding order.

Each dependency call receives the current input word, zero, and 65,536. Each returned result is multiplied by 511 and shifted right by 17 bits. **The shifted value is then explicitly converted to plain char before assignment to the destination byte.** That intermediate conversion is a distinct behavioral step; it must not be replaced in this contract by an unconditional low-eight-bit extraction.

**Inferred intent:** the constants suggest conversion of four bounded fixed-point components into byte-sized components. If every dependency result is between zero and 65,536 inclusive, the pre-character-conversion shifted values are between zero and 255 inclusive. This does **not**, by itself, establish that all those values are stored unchanged: plain-char signedness and out-of-range conversion behavior remain unresolved.

Neither component meanings nor dependency range enforcement is established.

## Interface

### Arguments and layouts

The function accepts two pointer-like arguments:

- **Destination address:** the beginning of four consecutive byte locations.
- **Input address:** the beginning of four consecutive 32-bit words.

| Position | Input offset | Input read width | Destination offset | Destination write width |
|---|---:|---:|---:|---:|
| First | 0 bytes | 32 bits | 0 bytes | 8 bits |
| Second | 4 bytes | 32 bits | 1 byte | 8 bits |
| Third | 8 bytes | 32 bits | 2 bytes | 8 bits |
| Fourth | 12 bytes | 32 bits | 3 bytes | 8 bits |

The input words are passed to F-0ef463575bea without a visible numerical transformation. Their application-level interpretation and signedness are not established. A high-bit-set input word must not automatically be treated as either a negative scalar or a large positive scalar at the dependency boundary.

The dependency results are held as signed integer values in the supplied representation. Arithmetic bounds below use the apparent 32-bit signed-integer model. Independent target-width evidence is not supplied.

### Result

There is no explicit return value. On normal completion, the direct result is four destination-byte assignments.

The function does not return a status, a completion count, an allocation, or the intermediate dependency results.

### Preconditions

Successful completion requires:

- Each input word to be readable when its read occurs.
- Each destination byte to be writable when its assignment occurs.
- Accesses to satisfy applicable alignment, representation, and memory-access rules.
- All four dependency calls to return normally.
- The dependency’s own preconditions to hold.
- Arithmetic and plain-character conversions to permit execution to continue through all four assignments.

No pointer, length, alignment, or numerical-range validation is visible.

Disjoint input and destination regions are not an explicit requirement of the visible operation. Overlap must be evaluated using the access order below, dependency effects, and applicable target rules.

## Behavior

### Input reads and dependency calls

The function reads the first 32-bit input word and calls F-0ef463575bea with:

1. That word.
2. Zero.
3. 65,536.

After that call returns, it reads the second word and makes the corresponding call with the same two constants. The third and fourth positions follow the same ordering.

Thus:

- The first call completes before the second input read.
- The second call completes before the third input read.
- The third call completes before the fourth input read.
- All four calls complete before the first destination assignment.

The function does not capture all four words before invoking the dependency. An earlier dependency call may, if it has suitable access, change a later input word before that word is read.

Each returned integer is retained for its corresponding conversion. There is no visible additional dependency call during the destination-assignment phase.

The constants zero and 65,536 are established. Their interpretation as bounds is not established without the dependency contract.

### Arithmetic before character conversion

For each dependency result, the visible arithmetic is multiplication by 511 followed by a right shift of 17 bits.

For a nonnegative result \(r\) whose product is representable, the shifted value \(q\) is:

\[
q=\left\lfloor\frac{511r}{131072}\right\rfloor
\]

This is truncation of a nonnegative quantity, not rounding to the nearest integer. No rounding bias is visibly added.

Under the apparent signed 32-bit model, results satisfying

\[
0\leq r\leq65536
\]

produce representable products. The largest such product is 33,488,896. Their shifted values satisfy:

\[
0\leq q\leq255
\]

These bounds describe the **pre-character-conversion shifted value**, not an unconditional stored-byte range.

The formula is not interchangeable with multiplication by 255 followed by division by 65,536. For example, a dependency result of 257 produces a shifted value of 1 under the visible arithmetic, whereas truncating \(255\times257/65536\) produces 0.

No separate post-dependency saturation step is visible.

For negative results or overflowing products, the supplied representation does not establish a portable numerical result. Negative right-shift behavior and signed multiplication overflow require target or execution-model evidence.

### Explicit plain-character conversion

Before each destination assignment, the corresponding shifted integer is explicitly converted to plain char.

Plain char is not established to be unsigned. Its representable range must therefore be distinguished from the destination byte’s storage width.

- Shifted values from 0 through 127 are representable under either signed or unsigned 8-bit plain-char semantics.
- Shifted values from 128 through 255 are representable under unsigned 8-bit plain-char semantics.
- Those same values are outside the representable range under signed 8-bit plain-char semantics.

For an out-of-range conversion to signed plain char, the relevant language rules permit an implementation-defined result or an implementation-defined signal. Consequently, an eight-bit destination does not prove that a shifted value of 255 reaches it unchanged.

There is also a subsequent assignment from the character-conversion result to the destination-byte representation. The supplied representation does not independently establish every target detail of that assignment.

A target may preserve the expected byte patterns through these steps, but that behavior requires evidence. It must not be assumed merely because the final storage unit is eight bits wide.

### Destination assignments

After all four dependency calls return, the function processes the retained results in order:

- Convert and assign the first result to destination offset 0.
- Convert and assign the second result to destination offset 1.
- Convert and assign the third result to destination offset 2.
- Convert and assign the fourth result to destination offset 3.

Each assignment uses its corresponding dependency result, not a new input read.

The source does not establish an aggregate four-byte store, atomic publication, rollback, or completion flag. Where individual writes are observable, an updated prefix may coexist with an older suffix.

Arithmetic or character-conversion failure can occur during this phase. In particular, an implementation-defined signal associated with a later character conversion may occur after earlier destination assignments have completed.

### Aliasing

The function’s own destination assignments cannot change an input word before this function reads it: all four input reads precede the first destination assignment.

That ordering alone does not make arbitrary overlap safe:

- A dependency call may modify later input storage.
- A dependency call may invalidate input or destination storage.
- Concurrent access may impose separate synchronization requirements.
- Target aliasing and alignment rules may impose additional constraints.

The established local property is only that the four final destination assignments do not overwrite an input word that this function has yet to read.

## Side effects

### Direct memory effects

On normal completion, the visible direct memory operations are:

- Four 32-bit input reads, interleaved with dependency calls.
- Four byte assignments to consecutive destination locations.

No direct write through the input argument is visible. No other explicit memory write appears in the function body.

This is the function’s direct footprint, not a guarantee about the entire call including dependency behavior.

### Dependency-mediated effects

F-0ef463575bea is called exactly four times on the normal path. Its memory, global-state, synchronization, resource, and exceptional effects are unknown.

The contract does not assume that the dependency:

- Preserves globals or either supplied region.
- Is pure or deterministic.
- Preserves allocation lifetimes.
- Avoids allocation, release, retention, callbacks, or external I/O.

No separate dependency status is inspected. Any normally returned integer is used as conversion data.

### Ownership and lifetime

The function visibly neither allocates nor releases storage. It does not visibly retain either pointer argument beyond its execution.

The caller provides the storage for the direct reads and assignments. No ownership transfer is established by this interface.

These observations do not constrain allocations, releases, or retention performed by the dependency through external state.

### Abnormal completion

No local error handling or rollback is visible.

If a dependency call does not return normally, no destination assignment by this function has yet occurred. Earlier dependency side effects may nevertheless remain.

If execution is interrupted during arithmetic, character conversion, or destination assignment, an earlier destination prefix may already have changed. No all-or-nothing guarantee is established.

An out-of-range plain-character conversion is therefore relevant both to the value eventually stored and to whether execution reaches that store or subsequent stores.

## Dependencies

### F-0ef463575bea

**Established use:** four ordered calls. Each receives one 32-bit input word, followed by zero and 65,536. Each normally returned integer supplies one component’s conversion data.

A stronger dependency contract would require:

- Interpretation of the first argument.
- Meaning of the two constants.
- Valid input domain.
- Return-value range and exact integer representation.
- Error, signal, and non-return behavior.
- Global and memory side effects.
- Ownership and lifetime effects.

**Inferred intent:** the dependency may constrain a scalar using the two constants as bounds. This is not adopted as established behavior.

Even a proven return range of zero through 65,536 would resolve only the arithmetic range discussed above. It would not independently resolve plain-char signedness or conversion behavior for shifted values from 128 through 255.

## Acceptance examples

These examples are conditional specifications, not runtime observations. Unless stated otherwise, they assume valid storage, normal dependency returns, the apparent 32-bit arithmetic model, and no interfering external effects. Returned values are specified directly; no claim is made about which input words produce them.

### Endpoints and representative values

If the dependency returns the following values in order:

| Position | Dependency result | Established shifted value | Destination result |
|---|---:|---:|---|
| First | 0 | 0 | Byte value 0 |
| Second | 257 | 1 | Byte value 1 |
| Third | 32,768 | 127 | Byte value 127 |
| Fourth | 65,536 | 255 | Conditional on target character-conversion and destination-assignment semantics |

The fourth shifted value is exactly 255. A stored byte value of 255 is an acceptance requirement only under target semantics that carry this value through the explicit plain-character conversion and destination assignment with that resulting byte value.

For example, unsigned 8-bit plain char with value-preserving destination assignment supports that outcome. Signed 8-bit plain char does not establish it without additional conversion evidence.

If the fourth conversion instead raises an implementation-defined signal that prevents further execution, the first three assignments have already occurred, but the fourth assignment is not established to occur.

### Adjacent quantization values

A dependency result of 256 produces a product of 130,816 and a shifted value of 0.

A dependency result of 257 produces a product of 131,327 and a shifted value of 1.

Both shifted values are representable in plain char regardless of its signedness under the stated eight-bit model. With ordinary value-preserving destination assignment, their stored byte values are respectively 0 and 1.

### First shifted value outside signed 8-bit plain-char range

A dependency result of 32,833 produces:

\[
511\times32833=16777663
\]

Its shifted value is 128. The arithmetic is nonnegative and safely representable in signed 32 bits.

That does not establish a stored byte value of 128 on an unknown target. Unsigned 8-bit plain char can represent it directly; signed 8-bit plain char cannot. The latter case requires implementation-defined conversion behavior to be documented before a stored value or continued execution can be required.

### Changed later input

If the first dependency call changes the second input word before returning, the second call receives the word read after that first return, not an entry-time snapshot.

This example is conditional on the dependency having such access; it does not assert that the dependency actually performs the write.

### Non-returning dependency

If the third dependency call does not return normally, the fourth call is not reached and none of this function’s four destination assignments has occurred. Earlier dependency effects are not rolled back.

### Interrupted output phase

If the first two destination assignments complete and the third conversion or assignment prevents further execution, the first two bytes are not restored by any visible mechanism. Atomic four-byte replacement is not supported by the evidence.

## Uncertainties

The following remain **documented_only**:

- **Dependency semantics:** no established clamp behavior, input domain, return range, or side-effect contract.
- **Integer model:** signed 32-bit arithmetic is apparent in the supplied representation, not independently established by target evidence.
- **Negative results:** right-shifting a negative product requires execution-model evidence.
- **Multiplication overflow:** signed wraparound must not be assumed.
- **Plain-char signedness:** the target’s choice is unknown.
- **Out-of-range character conversion:** uncertainty begins outside the target plain-char range, not merely outside 0–255. On signed 8-bit plain-char targets, shifted values 128–255 already require implementation-defined behavior and may trigger an implementation-defined signal.
- **Destination assignment:** exact byte results after a non-value-preserving character conversion require target representation and assignment semantics.
- **Input representation:** byte order, alignment requirements, and ABI details are not independently supplied.
- **External state:** dependency access to globals, aliased storage, resources, and concurrent activity is unknown.
- **Application meaning:** component names, channel order, color interpretation, and fixed-point units are not established.

The arithmetic examples with nonnegative, representable products establish their shifted values. Only examples whose shifted values are representable in plain char avoid the identified out-of-range character-conversion uncertainty. The upper-endpoint stored byte remains conditional.

## Evidence

This contract is based solely on static inspection of the supplied function representation.

Established evidence includes:

- Two pointer-like arguments.
- Four consecutive 32-bit input reads.
- Four ordered calls to F-0ef463575bea with constants zero and 65,536.
- Completion of all four calls before destination assignments.
- Multiplication by 511 and right shift by 17.
- An explicit conversion to plain char before each destination-byte assignment.
- Four ordered, consecutive byte assignments.
- No explicit return value, local validation, or rollback.

Fixed-point conversion and range normalization are inferred intent, not established dependency semantics.

No runtime tests have been performed. No dependency implementation, independent ABI evidence, or target evidence resolving plain-char signedness and out-of-range conversion behavior was supplied.
