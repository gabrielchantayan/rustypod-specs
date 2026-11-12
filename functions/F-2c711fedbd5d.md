<!-- Trusted provenance: F-2c711fedbd5d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2c711fedbd5d

## Purpose

F-2c711fedbd5d performs an ordered sequence of indirect calls and direct memory writes, then enters an unconditional loop containing further calls and a byte clear.

No application-level purpose is established. The supplied representation does not establish initialization, serialization, device control, allocation, or cleanup. Numerical address patterns alone do not identify an object layout or protocol.

This contract describes the caller’s visible behavior. It does not prescribe the internal behavior of unknown dependencies. Its applicability is conditional on the supplied control-flow representation being accurate: that representation reports invalid instruction data and removal of unreachable regions.

## Interface

### Explicit inputs

| Position | Represented width | Visible use |
|---|---|---|
| First | Unsigned 32-bit word | Determines the initial word-store address and value, the first dependency argument, and the first byte-selector address |
| Second | Opaque 32-bit word | No visible use |
| Third | Unsigned 32-bit word | Logically shifted right by seven and supplied to one dependency |
| Fourth | Unsigned 32-bit word | Stored through a dependency-derived address; logically shifted right by seven for a byte-store address and a later dependency argument |

The absence of a visible use for the second input does not establish its irrelevance to an external calling convention.

### Implicit inputs

Execution also depends on:

- Incoming link/return context used to derive the first indirect-call target.
- Fixed code references used to derive other call targets.
- Contents and accessibility of computed memory locations.
- Results returned by dependencies.
- Hidden state that dependencies may inspect or modify.

No dependency is assumed to preserve globals, leave memory unchanged, or acquire or release resources.

### Output and termination

The represented interface has no return value. Its normal path reaches an unconditional loop without a visible return.

This does not guarantee infinite execution. A dependency may not return, and calls or memory accesses may fault or otherwise transfer control. No recovery path is visible.

### Widths, arithmetic, and memory layout

Bits are numbered from zero at the least significant bit. Word stores write 32 bits; byte stores write eight bits. Byte reads used for selection are unsigned, producing values from zero through 255.

Unsigned 32-bit arithmetic and shifts have their represented width. In particular:

- The initial address expression uses unsigned multiplication and addition, reduced modulo 2³².
- The first selector calculation extracts bits 7 through 30 of the first input, then adds one.
- The third and fourth inputs are logically shifted right by seven.
- Left shifts of the unsigned byte-value sources by two retain the low 32 bits.

Other expressions involve signed 32-bit values, including dependency-derived address arithmetic and several left shifts. Overflow, negative signed shifts, and conversions between unsigned bit patterns and signed values are **documented_only** where the representation does not establish their machine-level interpretation. The formulas below do not silently impose wrapping semantics on those expressions.

There is no supplied guarantee about byte order, alignment requirements, valid pointer ranges, or whether a location is ordinary memory or mapped I/O. No structured object layout is established beyond the explicit access widths and offsets.

## Behavior

### Dependency labels and selector operations

Dependency IDs are local opaque labels, not verified external identities. Labels F-00000001 through F-00000060 describe the initial call sequence and loop roles. F-00000061 identifies the different target used at the loop head after the first iteration. Distinct labels do not prove distinct underlying implementations.

A range of dependency IDs means calls in increasing order. Unless arguments are specified, the representation supplies no explicit arguments. An unused result does not imply an effect-free call.

The **first selector address** is one plus the first input after masking off bit 31 and shifting right by seven. Its numerical range is one through 16,777,216.

A **first-selector write** consists of:

1. Reading one unsigned byte at the first selector address.
2. Adding seventeen to that byte.
3. Writing one byte at the resulting address.

The destination therefore lies between seventeen and 272 inclusive. Every occurrence performs a fresh selector read.

A **second-selector clear**, introduced later, uses the same read-and-add-seventeen selection rule but a different selector address.

### 1. Initial word store and first calls

Before any dependency call, write the first input as a 32-bit word at the address given by four times that input plus twenty, using the unsigned 32-bit address calculation described above.

Then call, in order:

1. F-00000001 with the first input.
2. F-00000002.
3. F-00000003 with the result of F-00000002.
4. F-00000004.
5. F-00000005 with the result of F-00000004.
6. F-00000006 through F-00000009.

Retain the result of F-00000009. At four times that result plus twenty, write the fourth input as a 32-bit word.

Call F-00000010 with four times the retained result.

The latter address and argument calculations involve a result represented as signed. No range or address-validity check precedes either word store.

### 2. Two stores to a repeated word destination

Call F-00000011 through F-00000014. Retain the result of F-00000014 and derive an **address base** equal to sixteen times that result.

At the address base plus twenty, write a 32-bit value equal to four times the retained result.

Call F-00000015 with these three explicit arguments:

1. The address base.
2. The same address base.
3. The third input logically shifted right by seven.

Call F-00000016 through F-00000019 and retain the result of F-00000019.

At the address base plus twenty, write the address base itself as a 32-bit value. This is the same computed destination as the preceding store. Five dependency calls, F-00000015 through F-00000019, separate the stores and may observe or alter that memory.

Call F-00000020 with the result of F-00000019 shifted left by two. This is a signed shift in the representation; its defined arithmetic cases correspond to multiplication by four.

The third input does not directly determine either word-store value or destination in this stage.

### 3. Two value writes through the first selector

Call F-00000021 through F-00000024. Retain the unsigned 32-bit result of F-00000024 as the **first byte-value source**.

Shift that source left by two at unsigned 32-bit width. Perform a first-selector write of the low eight bits of the shifted value.

Call F-00000025 with three equal arguments, each carrying the shifted value. The representation converts this unsigned shift result to a signed 32-bit intermediate before the call; interpretation of out-of-range conversions remains **documented_only**.

Next:

1. Call F-00000026.
2. Pass its result to F-00000027.
3. Call F-00000028 through F-00000031.
4. Retain the result of F-00000031.

Perform another first-selector write. Its value is obtained by masking the first byte-value source to its low thirty bits, logically shifting right by twenty, and retaining the low eight bits. The stored byte therefore contains original bits 20 through 27.

Call F-00000032 with the result of F-00000031 shifted left by two, subject to the signed-shift qualification.

The two selector reads are separated by dependencies. Their destinations need not match.

### 4. Fourth-input write and two first-selector clears

Call F-00000033 through F-00000036. Retain the unsigned result of F-00000036 as the **second byte-value source**.

Shift that source left by two at unsigned 32-bit width. Preserve the resulting 32-bit value for the later second-selector calculation, subject to its represented conversion to signed form.

Perform these operations in order:

1. At the address given by the fourth input logically shifted right by seven, plus twenty-eight, write the low eight bits of the shifted source.
2. Perform a first-selector write of zero.

The first write may modify the first selector byte itself. The following selector read must occur after that write, rather than being resolved in advance.

Then:

1. Call F-00000037 with the fourth input logically shifted right by seven.
2. Call F-00000038.
3. Call F-00000039 with the second byte-value source masked to thirty bits and logically shifted right by six.
4. Call F-00000040 through F-00000043.
5. Retain the result of F-00000043.

The F-00000039 argument contains original bits 6 through 29 in positions 0 through 23.

Perform another first-selector write of zero.

Call F-00000044 with the result of F-00000043 shifted left by two, subject to the signed-shift qualification.

### 5. Establishing loop state

Call F-00000045 through F-00000048.

From the unsigned result of F-00000048, retain bits 25 through 29 as the **initial loop field**. Its range is zero through thirty-one.

Form the **second selector address** from the preserved shifted second byte-value source:

- Multiply that preserved intermediate by 1,024.
- Add one.

This multiplication and addition use signed intermediates in the supplied representation. Where every operation and conversion is representable, the address equals 4,096 times the second byte-value source, plus one. For other inputs, that simplification is not an established overflow contract. Under a separately assumed wrapping 32-bit machine interpretation, it describes the corresponding low-32-bit result.

Read one unsigned byte at the second selector address and write zero at seventeen plus that byte.

Call F-00000049 with twice the initial loop field. This argument ranges from zero through sixty-two.

Call F-00000050 and retain its result as the initial **loop-carried call argument**.

### 6. Unconditional loop

Each iteration performs the following ordered operations:

1. Call the loop-head dependency with:
   - The current loop-carried call argument.
   - The current loop field logically shifted right by eight.
2. Call F-00000052 through F-00000055.
3. Retain the result of F-00000055.
4. Read one unsigned byte at the second selector address.
5. Write zero at seventeen plus that byte.
6. Call F-00000056 with the retained result shifted left by two.
7. Call F-00000057 through F-00000060.
8. Retain the signed result of F-00000060.
9. Set the next loop-carried call argument to that result shifted left by two.
10. Set the next unsigned loop field from that result shifted left by twenty-six.
11. Continue unconditionally.

The loop-head target is not constant across entry and repetition:

- The first iteration calls F-00000051.
- Every subsequent iteration calls F-00000061.

The supplied target construction establishes different computed targets for these two roles. It does not establish whether they ultimately implement different behavior.

The first loop-head call necessarily receives zero as its second argument because the initial loop field is at most thirty-one.

Subsequent second arguments are not necessarily zero. For example, an F-00000060 result of one produces a next loop-carried argument of four and a next loop field of 67,108,864; the next loop-head second argument is 262,144. These shifts are representable signed operations.

For other results, the signed shift by twenty-six can overflow or involve a negative operand. Under a conditional wrapping 32-bit interpretation, the next second argument places the result’s low six bits in positions 18 through 23. That interpretation is not an unconditional guarantee.

## Side effects

### Direct memory effects

The represented path before the loop contains exactly:

- **Four 32-bit stores.**
- **Six byte stores.**
- **Five one-byte selector reads.**

The six byte stores occur in this order:

| Order | Destination derivation | Stored value |
|---|---|---|
| 1 | First selector | Low byte of the first byte-value source shifted left by two |
| 2 | First selector | Original bits 20 through 27 of that source |
| 3 | Fourth input shifted right by seven, plus twenty-eight | Low byte of the second byte-value source shifted left by two |
| 4 | First selector | Zero |
| 5 | First selector | Zero |
| 6 | Second selector | Zero |

Thus, **four**, not five, pre-loop byte stores use the first selector: two value writes and two clears. The second selector contributes one pre-loop clear. The remaining byte store uses the fourth input directly to derive its destination.

Each completed loop iteration adds one selector-byte read and one byte store of zero.

These are counts of direct operations, not distinct locations or changed values. Dependencies may perform additional accesses.

### Ordering, aliasing, and state

No separation between memory locations is established. Word stores may overlap selector bytes or byte destinations. Selected destinations may coincide with their own selector addresses.

The selector address values are retained or derived from retained scalar values; the memory bytes at those addresses are not cached. Every selector operation rereads memory at its specified point.

Dependencies may modify either selector byte, a future destination, or memory written earlier. No preservation guarantee permits eliminating calls, merging selector reads, or treating repeated writes as redundant.

### Ownership and errors

No allocation, release, reference-count operation, or ownership transfer is established by the caller’s direct operations. A dependency result used in address arithmetic is not thereby established as an allocated pointer or owned resource.

No error-result check, rollback, cleanup path, or transactional boundary is visible. If execution stops partway through, earlier stores and dependency effects have already occurred. Their external persistence depends on the memory and dependency environment.

## Dependencies

All dependency IDs denote opaque call roles. Most targets are derived from fixed code references; F-00000001 instead depends on incoming link/return context. The repeated loop-head role has the entry-versus-repetition distinction described above.

Selected dependency results are:

- Forwarded unchanged to another dependency.
- Used in address or value arithmetic.
- Used as byte-value sources or extracted bit fields.
- Retained as loop-carried state.

For every dependency, the following remain **documented_only**:

- Actual identity and calling convention.
- Hidden arguments and global inputs.
- Memory reads and writes.
- Global-state preservation or mutation.
- Resource acquisition, ownership transfer, and release.
- I/O, faults, exceptional transfers, and normal-return guarantees.

“No explicit arguments” describes only the supplied representation. It does not establish that a dependency has no inputs.

## Acceptance examples

These are conditional static examples, not executed tests. They assume accessible memory and returning dependencies wherever the described prefix requires them.

### Initial store precedes the first call

With a first input of three, the initial operation writes the 32-bit value three at address thirty-two. F-00000001 then receives three.

Calling F-00000001 before this store would violate the represented order.

### First selector ignores bit 31

First inputs zero and hexadecimal 80000000 both produce first selector address one.

They do not produce equivalent executions: the initial stored word and F-00000001 argument differ. This example relies only on the represented unsigned arithmetic, not on signed-overflow assumptions.

### Selector redirection is observed

If the first selector byte is five during the first selected write, its destination is twenty-two. If an intervening dependency changes that byte to nine, the next first-selector write targets twenty-six.

Reusing the earlier destination would violate the fresh-read behavior.

### Byte truncation and extraction differ

If F-00000024 returns hexadecimal 12345678:

- The first selected value is hexadecimal E0.
- The second selected value is hexadecimal 23.

The second value comes from original bits 20 through 27, not from another multiplication.

### A direct write can redirect the immediately following clear

Suppose the first input is 3,456, making the first selector address twenty-eight. Let the fourth input be zero and F-00000036 return two.

The fourth-input-derived write stores eight at address twenty-eight. The immediately following selector read therefore obtains eight and clears address twenty-five, absent any fault. It must not use the selector byte’s previous value.

### Third-input forwarding

If the third input is 255, F-00000015 receives one as its third explicit argument. Its first two arguments remain the repeated address base derived from F-00000014.

### Loop initialization and transition

If F-00000048 returns all one bits, the initial loop field is thirty-one. F-00000049 receives sixty-two, and F-00000051 receives zero as its second argument.

If the first F-00000060 result is one, the following F-00000061 call receives four and 262,144. Repeating the initial loop-head target or forcing the second argument to remain zero would not match the representation.

### Direct-store accounting

A prefix that reaches F-00000050 after completing all earlier operations contains four direct word stores and six direct byte stores. Exactly four of those byte stores use the first selector.

Each subsequently completed loop iteration adds one direct byte store, regardless of whether zero was already present.

## Uncertainties

The following remain **documented_only**:

1. **Control-flow reliability:** invalid instruction data and removed unreachable regions limit confidence in the supplied reconstruction.
2. **Signed arithmetic:** overflowing multiplication, negative signed shifts, overflowing signed left shifts, and out-of-range unsigned-to-signed conversions lack an independently established machine contract.
3. **Address interpretation:** alignment, byte order, address validity, memory type, and pointer-conversion behavior are unspecified.
4. **Dependency behavior:** identities, hidden inputs, memory effects, ownership effects, and return guarantees are unknown.
5. **ABI completeness:** explicit arguments and displayed call arguments may omit machine-level inputs.
6. **Application intent:** no protocol, object model, or device semantics are established.
7. **Runtime behavior:** no runtime tests have been performed.

## Evidence

Established static evidence consists of the supplied representation’s argument widths, direct access widths, masks, shifts, call ordering, retained results, computed call targets, and unconditional loop.

Enumerating its pre-loop byte assignments yields two first-selector value writes, one fourth-input-derived write, two first-selector clears, and one second-selector clear: six byte stores in total, four through the first selector.

The differing first and subsequent loop-head targets follow from the target-reference state at loop entry and at the end of each iteration.

Ranges and acceptance examples are arithmetic deductions from those operations. Application intent remains unknown. No dependency implementation, external memory specification, independently verified instruction stream, or runtime trace was supplied.
