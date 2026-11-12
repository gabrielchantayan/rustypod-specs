<!-- Trusted provenance: F-ccddd5882ade; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ccddd5882ade

## Purpose

F-ccddd5882ade performs a fixed sequence of indirect calls and calculated-address memory writes, then enters an unconditional loop containing additional calls and byte clears.

No higher-level purpose is established. The evidence does not identify an initializer, serializer, allocator, device operation, or other application-level service. Small destination addresses and unusual call targets do not establish such an interpretation.

This contract describes behavior conditional on the supplied static representation accurately describing executable operations. That representation reports invalid instruction data and removal of unreachable blocks. These warnings limit confidence in the represented execution path.

## Interface

### Explicit inputs

| Position | Represented type and width | Visible use |
|---|---|---|
| First | Signed 32-bit integer | Determines the initial word destination, supplies the initial stored value and first call argument, and is stored again at a dependency-selected destination. |
| Second | Opaque 32-bit value | No explicit use in the visible body. |
| Third | Unsigned 32-bit integer | Logically shifted right by seven and supplied to a three-argument call. |
| Fourth | Unsigned 32-bit integer | Stored at a dependency-selected destination; its right-shifted value also determines a byte destination and supplies a later call argument. |

The absence of an explicit use of the second input does not prove that unknown dependencies cannot observe its incoming machine location.

### Implicit inputs and preconditions

Execution depends on:

- An incoming link-related machine value used to calculate the first indirect-call target.
- Values returned by unresolved dependencies.
- Bytes read from calculated selector addresses.
- The accessibility and behavior of all addressed memory.
- Machine state consumed by indirect calls, including any arguments not represented explicitly.

Successful continuation requires each reached call target and memory access to be valid. The representation supplies no complete calling convention, memory map, bounds contract, alignment contract, or valid-input domain.

### Output and completion

The represented interface has no return value. No normal return operation is visible.

If every reached memory access succeeds and every dependency returns normally, execution reaches an unconditional loop and does not return. Faults, traps, exceptions, nonlocal transfers, and nonreturning dependencies remain possible. Their handling is documented_only.

### Widths and layout

Word stores are four bytes wide. Selector reads and byte stores are one byte wide. No explicit multi-byte reads are shown.

Unsigned masks and right shifts operate on 32-bit values. Byte stores retain the least significant eight bits of their supplied values. Selector bytes are interpreted as unsigned values from zero through 255.

Several intermediate values are represented as signed 32-bit integers. Multiplication, addition, and left shifting can exceed their signed range. Where this contract describes wrapping arithmetic, that description is conditional on 32-bit machine arithmetic with truncation to the low 32 bits; it is not a claim that overflowing signed operations have defined language-level behavior.

Endianness, address width, pointer conversion rules, and alignment requirements are unresolved. The word-address formulas use a 20-byte displacement and either a four-byte or sixteen-byte stride. Those formulas do not establish an array or record layout.

## Behavior

### Dependency notation

Opaque IDs identify unresolved dependencies without assigning recovered names:

- **F-unknown-001:** the first indirect call.
- **F-unknown-002:** the other indirect-call occurrences, except the loop-entry calls distinguished below.
- **F-unknown-003:** the leading call in the first loop iteration.
- **F-unknown-004:** the leading call in subsequent loop iterations.

F-unknown-002 is a collective label, not an assertion that its occurrences share a target, signature, or implementation.

A call with “no explicit arguments” may still consume hidden machine state. A “discarded-result call” has no visibly consumed return value; its effects are not assumed to be absent.

### Initial word stores

The first explicit memory operation stores the first input as a four-byte word at:

**20 + 4 × first input.**

Only afterward does F-unknown-001 receive the first input. Its target is calculated from the incoming link-related value plus 0xee8; the architecture-specific interpretation of that calculation remains unresolved.

The next eight calls comprise, in order:

1. A no-explicit-argument producer.
2. A consumer receiving that producer’s 32-bit result.
3. A second no-explicit-argument producer.
4. A consumer receiving the second producer’s result.
5. Three discarded-result calls with no explicit arguments.
6. An index-producing call with no explicit arguments.

The procedure stores the first input as a four-byte word at **20 + 4 × that index**.

It then passes four times that index to another call. Three discarded-result calls follow, then another index-producing call.

For this latest index, the procedure stores four times the index at **20 + 16 × the index**. The stored 32-bit value is retained for later use as the **early scaled value**.

These are three separately ordered word stores, even if their calculated destinations coincide.

### Fourth-input storage and retained word destination

The early scaled value is passed to another call. The same eight-call structure follows: two producer/consumer pairs, three discarded-result calls, and an index-producing call.

The fourth input is stored as a four-byte word at **20 + 4 × the resulting index**.

Four times that index is passed to another call. Three discarded-result calls and a further index-producing call follow.

For this latest index:

- The **retained word base** is sixteen times the index.
- Four times the index is stored at **retained word base + 20**.

The procedure also derives the **original selector address**:

**1 + ((early scaled value AND 0x7fffffff) logically shifted right by 7).**

Under the represented unsigned interpretation, this address ranges from 1 through 0x01000000 inclusive.

The next call receives three explicit arguments, in order:

1. The retained word base.
2. The retained word base again.
3. The third input logically shifted right by seven.

Three discarded-result calls follow, then a result-producing call.

The procedure then overwrites **retained word base + 20** with the retained word base itself. It uses the retained numerical value, not a value reloaded from that destination. Intervening dependencies may nevertheless have changed the destination before this explicit overwrite.

Four times the latest producer’s result is passed to another call. Three discarded-result calls follow, then a further result-producing call.

### First two selector-based byte stores

The latest 32-bit result becomes the **first byte source**.

The procedure reads an unsigned byte at the original selector address. It adds 17 to that byte’s numeric value and stores the low byte of four times the first byte source at the resulting address.

Every destination selected this way lies between **17 and 272 inclusive**. The read byte is used as a numeric address component, not as a full-width pointer.

The full 32-bit, four-times-first-byte-source value is then supplied as all three explicit arguments to the next call.

The following calls occur in order:

- A no-explicit-argument producer.
- A consumer receiving that producer’s result.
- Three discarded-result calls.
- Another result-producing call.

The procedure rereads the byte at the original selector address and performs a second selected byte store. The stored value is the low byte of:

**(first byte source AND 0x3fffffff) logically shifted right by 20.**

This uses the earlier first byte source, not the most recent dependency result. Before byte truncation, the masked-and-shifted value ranges from zero through 1023.

The fresh selector read matters: intervening calls may have changed the selector, so these two byte stores need not share a destination.

Four times the most recent dependency result is then passed to another call. Three discarded-result calls and a further result-producing call follow.

### Fourth-input destination and original-selector clears

The latest result becomes the **second byte source**.

The procedure stores the low byte of four times that source at:

**28 + (fourth input logically shifted right by 7).**

This unsigned calculation ranges from 28 through 0x0200001b inclusive and does not overflow 32 bits.

Immediately afterward, the procedure rereads the original selector byte and writes zero at **17 + that byte**. There is no intervening indirect call between the fourth-input-derived store and this selector read.

Consequently, if the fourth-input-derived store changes the original selector byte, the following clear uses the newly stored selector value.

The next operations are:

1. A call receiving the fourth input logically shifted right by seven.
2. One discarded-result call.
3. A call receiving **(second byte source AND 0x3fffffff) logically shifted right by 6**.
4. Three discarded-result calls.
5. A result-producing call.
6. A fresh original-selector read and another zero store at the selected address.

The masked-and-shifted argument in step 3 ranges from zero through 0x00ffffff.

Four times the result from step 5 is then passed to another call. Three discarded-result calls and another result-producing call follow.

### Alternate selector and loop initialization

The latest result is masked with 0x3fffffff and logically shifted right by 25. This produces the **initial loop field**, ranging from zero through 31.

An **alternate selector address** is formed as:

**1 + 1024 × (four times the second byte source).**

The four-times value is the retained 32-bit intermediate previously used for the fourth-input-derived byte store. Under wrapping 32-bit arithmetic, this address is equivalently the low 32 bits of the second byte source shifted left by twelve, plus one. This equivalence does not resolve signed-overflow semantics in the representation.

The procedure reads the alternate selector byte and writes zero at **17 + that byte**.

It then calls a dependency with twice the initial loop field. That argument ranges from zero through 62. A subsequent no-explicit-argument producer supplies the **initial loop carrier**.

### Unconditional loop

Each represented iteration has the following observable order:

1. A leading call receives the current loop carrier and the current loop field logically shifted right by eight.
2. Three discarded-result calls occur.
3. A producer returns a result.
4. The alternate selector byte is freshly read, and zero is stored at 17 plus that byte.
5. Another call receives four times the result from step 3.
6. Three discarded-result calls occur.
7. A final producer returns a result.
8. The next loop carrier becomes four times that final result, and the next loop field becomes that result shifted left by 26.

The first iteration’s leading call is F-unknown-003. Subsequent iterations use F-unknown-004. Their represented target calculations differ; they must not be assumed to designate the same dependency.

The alternate selector address remains derived from the pre-loop second byte source. Only its byte contents are reread each iteration.

On the first iteration, the leading call’s second explicit argument is necessarily zero because the initial loop field is at most 31.

On later iterations, under 32-bit shift semantics, that argument depends on the low six bits of the preceding iteration’s final result. Its range is zero through 0x00fc0000, in multiples of 0x00040000.

There is no visible exit condition.

## Side effects

### Explicit memory effects

Before the loop, the representation contains exactly:

- **Six four-byte stores.**
- **Six one-byte stores.**
- **Five one-byte selector reads.**

The six word stores are the initial first-input store, the dependency-selected first-input store, the early-scaled-value store, the fourth-input store, and the two successive stores at the retained word destination.

The six pre-loop byte stores are:

1. The low byte of four times the first byte source, through the original selector.
2. The truncated upper field of the first byte source, through the original selector.
3. The low byte of four times the second byte source, at the fourth-input-derived destination.
4. Zero through the original selector immediately afterward.
5. Zero through the original selector after another call sequence.
6. Zero through the alternate selector before loop initialization.

Four pre-loop selector reads use the original selector address; one uses the alternate selector address.

Each completed loop iteration adds one selector-byte read and one byte store. These counts exclude all memory effects inside dependencies.

### Aliasing and visibility

No separation among word destinations, selector addresses, byte destinations, and dependency-accessed storage is established.

Every represented selector access is a fresh read. Caching a selector byte across intervening calls or stores could change the selected destination.

The two stores at the retained word destination are distinct. Replacing them with only the final store would remove an intermediate value potentially observable by intervening dependencies.

A selected byte store may overwrite its own selector. Its destination is determined from the byte read before that store; later selector reads may observe the changed byte.

No atomicity, synchronization, volatile semantics, cache behavior, or hardware-register interpretation is established.

### Ownership and errors

No explicit allocation, release, ownership transfer, or cleanup is visible. Unknown dependencies are not assumed to preserve globals, preserve pointed-to storage, or acquire or release resources.

There are no visible bounds checks, null checks, status checks, or error branches. Failure behavior for invalid accesses or calls is documented_only.

## Dependencies

| Opaque ID | Established role | Unresolved properties |
|---|---|---|
| F-unknown-001 | Receives the first input after the initial word store; target depends on incoming machine state. | Identity, complete signature, effects, termination, errors. |
| F-unknown-002 | Collectively labels the remaining producers, consumers, discarded-result calls, and other loop calls. | Individual identities, target equivalence, hidden arguments, preserved state, ownership, errors. |
| F-unknown-003 | Leading two-argument call in the first loop iteration. | Identity, complete signature, effects, termination, errors. |
| F-unknown-004 | Leading two-argument call in subsequent loop iterations. | Identity, relationship to F-unknown-003, effects, termination, errors. |

Some dependency results become stored values or address components without visible validation. Other results are forwarded, multiplied, shifted, masked, or truncated. No stronger dependency contract is established.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Initial ordering.** With first input two and a writable initial destination, the first explicit memory operation stores the four-byte value two at address 28. F-unknown-001 is called afterward.

2. **Third-input shift.** With third input 255, the identified three-argument call receives one as its third argument. Its first two arguments both equal the retained word base.

3. **First byte source.** With first byte source 0x00100003, the first selected byte store writes 0x0c. The later upper-field store writes 0x01. Their destinations may differ.

4. **Selector changes between calls.** A selector byte of three selects address 20. If an intervening dependency changes it to nine, the next selector-based store selects address 26.

5. **Fourth-input destination.** With fourth input 128, its derived byte destination is address 29. The later call receiving the shifted fourth input receives one.

6. **Immediate aliasing.** If the original selector address is 29, fourth input is 128, and the second byte source is two, the fourth-input-derived store writes eight at address 29. The immediately following selector read observes eight, so the clear targets address 25, assuming no external interference.

7. **Retained word overwrite.** With retained word index two, the first store at address 52 writes eight. After the specified intervening calls, the explicit overwrite at address 52 writes 32.

8. **Loop transition.** The first iteration’s leading second argument is zero. If that iteration’s final producer returns one, the next carrier is four and the next leading second argument is 0x00040000 under 32-bit arithmetic.

9. **Memory accounting and continuation.** If all pre-loop operations complete, the procedure has performed six explicit word stores, six explicit byte stores, and five explicit selector reads. Each completed iteration adds one selector read and one byte clear. If all accesses succeed and all dependencies return, execution continues looping rather than returning.

## Uncertainties

The following remain documented_only:

- Whether the reported instruction-data and reachability problems invalidate represented operations or ordering.
- Original purpose and intended valid-input domain.
- Dependency identities, full calling conventions, hidden inputs, and preservation rules.
- Architectural address width, alignment, endianness, signed-overflow behavior, and pointer interpretation.
- Whether small addresses designate mapped storage, hardware, or representation artifacts.
- Dependency-induced changes to memory or other state.
- Fault handling, exception propagation, nonlocal exits, and external termination.
- Resource ownership, allocation, lifetime, concurrency, and synchronization.

The formulas establish visible data relationships, not semantic data structures or proof that calculated destinations are valid.

## Evidence

Established static evidence comprises the represented input widths, arithmetic and masks, explicit word stores, byte reads and stores, call argument relationships, call ordering, unused explicit second input, and unconditional terminal loop.

The explicit pre-loop memory operations account consistently for six four-byte stores, six one-byte stores, and five selector-byte reads.

No higher-level intent is inferred as a requirement. Arithmetic examples involving overflow or left shifts are conditional on the stated 32-bit interpretation.

Only the supplied static representation was used. No implementation access, runtime execution, or tests were performed.
