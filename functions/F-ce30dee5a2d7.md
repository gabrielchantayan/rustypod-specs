<!-- Trusted provenance: F-ce30dee5a2d7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ce30dee5a2d7

## Purpose

F-ce30dee5a2d7 performs a state-dependent dispatch operation. Its visible behavior combines a memory-derived value with a dependency result, publishes several values to memory, and invokes an ordered series of dependencies. Execution ends with an indirect transfer whose target and return behavior are unresolved.

**Established static evidence:** the supplied representation shows one unsigned 16-bit memory read, three 32-bit memory writes, one byte write, thirteen dependency invocations before the terminal transfer, and no explicit conditional branches or loops.

**Inferred intent:** the operation may coordinate another component through shared storage and calls. The evidence does not establish an application-level purpose, the significance of the constants, or the meaning of the written fields.

This contract specifies the visible accesses, argument relationships, and ordering constraints. It does not establish a complete conventional calling interface or guarantee that the recovered entry and exit boundaries are complete.

## Interface

### Explicit inputs and output

The visible interface accepts three 32-bit words.

| Input | Established use |
|---|---|
| First word | No direct use is displayed. |
| Second word | No direct use is displayed. |
| Third word | Published by a 32-bit write beginning at absolute address `0x4E`. |

The first two words are not visibly validated or rejected. Their absence from explicit expressions does not prove that dependencies cannot observe them through machine state.

No explicit return value is specified. The displayed endpoint is a return following the terminal indirect transfer. Reaching that endpoint requires the terminal transfer to return; the evidence does not establish that it does.

### Implicit execution state

The operation also requires state outside the three explicit inputs:

- Two 32-bit address-related words determine the location of the initial read.
- A signed 32-bit arithmetic word contributes to two difference calculations and the byte store.
- An incoming return-link value determines the target of the first dependency invocation.
- Two auxiliary 32-bit values are consumed after particular dependency invocations.
- An unresolved callable target is used for the terminal transfer.

The auxiliary values are not established as ordinary return values or caller-supplied parameters. Their production and calling conventions remain unresolved.

For this contract, the **source location** is the address formed from the two address-related words. The **arithmetic state** is the word used in the differences involving `0x3D` and `0x46`. These names describe observable roles, not additional formal parameters.

### Execution preconditions

Normal completion requires readable storage for the 16-bit source access, writable storage for every direct store, and valid targets for all invoked dependencies. In particular:

- Absolute addresses `0x41`, `0x45`, and `0x4E` must support the displayed 32-bit stores.
- F-d03 must return an address for which an additional five-byte offset identifies writable byte storage.
- The return-link-derived target and terminal target must be executable under the applicable machine conventions.
- References supplied to dependencies must be meaningful under those dependencies’ contracts.

The body does not check these conditions. It does not establish alignment tolerance, address-space mapping, pointer validity, executable-state selection, or fault handling. Supplying the three explicit words alone is insufficient to satisfy the complete execution preconditions.

## Behavior

### Observable ordering

The visible path is unconditional, but every later event depends on earlier accesses completing and earlier calls returning.

The source read precedes publication of the third explicit word. That publication precedes F-d01. The destination byte is written only after F-d03 returns. The two remaining fixed-location stores occur after that byte store and before F-d04.

Dependency invocations have the following total order:

F-d01 → F-d02 → F-d03 → F-d04 → F-d05 → F-d06 → F-d07 → F-d08 → F-d09 → F-d10 → F-d11 → F-d12 → F-d13 → F-d14.

F-d14 denotes the unresolved terminal transfer. This ordering does not constrain additional operations performed inside any dependency.

### Source-dependent dispatch arguments

The source access reads an unsigned 16-bit value. Subtracting `0x3E` produces a retained 32-bit unsigned adjustment. This is not a subtraction confined to 16 bits: source values below `0x003E` produce the corresponding negative difference represented in the 32-bit unsigned domain.

F-d01 receives the numeric argument `0x36` and fixed data reference D-01. Its callable target is derived by adding `0xF66` to the incoming return-link value. The representation does not establish safe wrapping behavior for an overflowing target calculation.

F-d01 supplies a result represented as 64 bits. F-d02 receives three arguments with these relationships:

- Its first argument is the low 32 bits of that result, combined by bitwise exclusive-or with the retained source adjustment, then increased by `0x32`. The exclusive-or and addition operate in the 32-bit unsigned domain.
- Its second argument comes from the high 32 bits of the result and is presented through a signed 32-bit interpretation.
- Its third argument is callable reference F-c01.

The upper result word is selected independently of the lower-word mixing. If its top bit is set, the precise numeric meaning of the signed conversion depends on the applicable conversion convention; the evidence does not supply that convention.

F-c01 is passed onward, not directly invoked by the visible body.

### Arithmetic-state publication

A full-width difference between the arithmetic state and `0x3D` is captured after F-d01 returns and before F-d02 is invoked. That captured value supplies the eventual 32-bit store at `0x41`.

The byte store uses a separately evaluated difference involving the arithmetic state and `0x3D`, after F-d03 returns. Only its low eight bits are written. The full-width capture and the later byte calculation therefore occur at different points, with F-d02 and F-d03 between them.

Their values agree modulo 256 if the relevant arithmetic state is unchanged and both evaluations have defined arithmetic semantics. Such agreement must not be assumed solely from the similar expressions: the supplied evidence does not establish the necessary register-preservation conventions.

A later argument to F-d07 depends on the arithmetic state minus `0x46`, followed by a signed right shift of nine bits. This is another use at a later point in execution, not an explicitly saved copy of the earlier difference.

### Auxiliary and returned-value transformations

Two dependency arguments depend on auxiliary machine state rather than ordinary displayed return values:

- F-d05 receives an unsigned 32-bit right shift by 17 of the first auxiliary value. Its consumption follows F-d04.
- F-d09 receives a signed 32-bit right shift by 18 of the second auxiliary value. Its consumption follows F-d08.

The placement associates these values with the preceding calls, but does not establish complete output interfaces for those calls.

F-d06 supplies a 32-bit word that becomes F-d07’s first argument. F-d09 supplies a signed 32-bit result. F-d10 receives that result as its first argument and the same result shifted right by 14 under signed interpretation as its second argument.

F-d11, F-d12, and F-d13 have no displayed explicit arguments. F-d14 likewise has no displayed explicit arguments. This absence does not establish that these dependencies lack implicit inputs.

### Width and arithmetic limits

The visible data operations distinguish 8-, 16-, 32-, and 64-bit quantities:

| Quantity or operation | Established width or interpretation |
|---|---|
| Source memory access | Unsigned 16 bits |
| Explicit input words | 32 bits |
| Fixed-location stores | 32 bits each |
| Destination-relative store | 8 bits |
| F-d01 result | 64 bits, divided into two 32-bit portions |
| Source adjustment and F-d02 mixing | Unsigned 32 bits |
| First auxiliary shift | Unsigned 32 bits |
| Arithmetic-state differences | Signed 32-bit expressions |
| Other displayed right shifts | Signed 32-bit expressions |

Unsigned 32-bit arithmetic is modulo \(2^{32}\). Signed differences are unambiguous when their mathematical results are representable. Outside that range, a machine-word wrapping interpretation is not established by the supplied representation.

For negative signed operands, the precise right-shift rule is also unresolved. Examples using sign-extending shifts below are conditional on that convention. No saturation, clipping, validation, or recovery operation is visible.

## Side effects

### Direct memory accesses

The following table specifies the direct accesses in their relative order. Dependency calls occur between some accesses, so this is not a claim that the accesses are consecutive.

| Access | Location and extent | Value or observation |
|---|---|---|
| Read | Source location, two bytes | Unsigned 16-bit source value |
| Write | `0x4E` through `0x51` | Third explicit input word |
| Write | F-d03 result plus five bytes | Low byte of the later difference from `0x3D` |
| Write | `0x45` through `0x48` | 32-bit value `0x59000000` |
| Write | `0x41` through `0x44` | Captured full-width difference from `0x3D` |

Byte order is not established. In particular, the 32-bit constant must not be expanded into an asserted byte sequence without an endianness assumption.

The three fixed store ranges do not overlap one another. No non-aliasing guarantee is shown for the source location or F-d03’s returned destination. Either may refer to storage overlapping a fixed range.

The distinction between access order and final contents is material:

- A source read overlapping `0x4E` observes storage before this body publishes the third explicit word.
- A destination byte overlapping `0x41` or `0x45` can subsequently be overwritten by a fixed-location store.
- Dependencies invoked after a store may change that storage again.

The warning about overlapping global storage prevents treating separately named locations as independent objects merely because they have different symbolic descriptions.

### Ownership and dependency effects

No allocation, release, reference-count update, or explicit ownership transfer is directly visible.

F-d03’s result is used as writable storage, but that use establishes neither ownership nor lifetime. The function does not visibly release the returned storage.

D-01, D-02, and F-c01 are supplied to dependencies. Whether recipients read, modify, retain, invoke, or release associated resources is unknown. No dependency is assumed to preserve globals or other shared memory.

### Failure and partial completion

There is no visible local error result, validation branch, retry, rollback, or cleanup path.

A failed memory access, a nonreturning dependency, or a transfer elsewhere prevents any guarantee about subsequent events. Earlier stores may already be observable. Nothing in the visible behavior provides transactionality or restores the entry state.

## Dependencies

These opaque IDs identify invocation roles. They do not establish that all roles resolve to distinct implementations.

| ID | Required visible relationship |
|---|---|
| F-d01 | Receives `0x36` and D-01; supplies the represented 64-bit result; target depends on incoming return-link state. |
| F-d02 | Receives the mixed low-word argument, signed interpretation of the high word, and F-c01. |
| F-d03 | Supplies the address used for the byte store at offset five. |
| F-d04 | Receives `0x59000000`, literal word `0x00857316`, and D-02, in that order. |
| F-d05 | Receives the first auxiliary value after unsigned right shift by 17. |
| F-d06 | Supplies the first argument to F-d07. |
| F-d07 | Receives F-d06’s word and the signed shifted difference involving `0x46`. |
| F-d08 | Has no displayed explicit arguments; precedes consumption of the second auxiliary value. |
| F-d09 | Receives the second auxiliary value after signed right shift by 18; supplies a signed 32-bit result. |
| F-d10 | Receives F-d09’s result and its signed right shift by 14. |
| F-d11 | Executes after F-d10, with no displayed explicit arguments. |
| F-d12 | Executes after F-d11, with no displayed explicit arguments. |
| F-d13 | Executes after F-d12, with no displayed explicit arguments. |
| F-d14 | Terminal unresolved indirect-transfer role; return behavior is unknown. |
| F-c01 | Callable reference passed to F-d02, without a direct invocation here. |

D-01 and D-02 are fixed data references. Their contents, extents, mutability, and lifetimes are unspecified. The second argument to F-d04 is a literal word; interpreting it as an address, tag, or handle would require additional evidence.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, they assume valid accesses, returning dependencies, representable signed differences, and the specified widths.

1. **Neutral source adjustment.**  
   A source value of `0x003E` and an F-d01 low word of `0x00000010` require F-d02’s first argument to be `0x00000042`. The upper result word is handled independently.

2. **Adjustment below zero.**  
   A source value of zero produces unsigned adjustment `0xFFFFFFC2`. With an F-d01 low word of zero, F-d02’s first argument is `0xFFFFFFF4`. A clamped zero result would violate the displayed unsigned arithmetic.

3. **Byte truncation versus word publication.**  
   If the arithmetic state remains `0x0000013D` across the relevant evaluations, the destination byte is `0x00`, while the store at `0x41` writes `0x00000100`.

4. **Different arithmetic-state observations.**  
   If the captured difference is `0x00000100` but the arithmetic state at the byte calculation is `0x0000013E`, the captured word remains `0x00000100` and the byte is `0x01`. This example is conditional on a machine convention that permits those differing observations.

5. **Aliased source and publication.**  
   If the source location overlaps the word beginning at `0x4E`, its read precedes this function’s write there. The numeric source value depends on the preexisting bytes and byte order, not solely on the third explicit input.

6. **Byte store overwritten by a word store.**  
   If F-d03 returns an address whose five-byte offset is `0x45`, the byte write occurs before the 32-bit constant write covering that location. The byte is not required to survive that later store.

7. **Unsigned auxiliary shift.**  
   A first auxiliary value of `0xFFFFFFFF` requires F-d05’s argument to be `0x00007FFF`.

8. **Conditional negative shift.**  
   If F-d09 returns `-1` and signed right shifts extend the sign bit, both F-d10 arguments are `-1`. This expectation is not unconditional evidence of the platform’s shift convention.

9. **Later dependency interference.**  
   If F-d04 overwrites `0x41`, no subsequent direct store in this body restores the captured difference there. Final-memory acceptance must account for dependency effects.

10. **Partial completion.**  
    If F-d03 does not return, publication at `0x4E` has already occurred. The destination byte and the stores at `0x45` and `0x41` have not yet occurred on the visible path.

## Uncertainties

The following remain **documented_only**:

- Complete entry interface, register conventions, and boundary completeness.
- Whether the two explicitly unused words remain observable to dependencies.
- Validity, mapping, and alignment requirements of the low absolute addresses.
- Endianness and effective-address overflow behavior.
- Signed-overflow behavior and negative signed right-shift semantics.
- Numeric conversion behavior for an upper result word exceeding the signed 32-bit positive range.
- Preservation of arithmetic state across calls.
- Exact production and interpretation of the auxiliary values.
- Dependency side effects, failure modes, and global-state preservation.
- Contents and storage contracts of D-01 and D-02.
- Ownership, extent, and lifetime of F-d03’s returned storage.
- Whether F-c01 is retained or invoked.
- Semantic meaning of F-d04’s literal second argument.
- Terminal-target identity, implicit arguments, and return behavior.

These unknowns are not resolved by assigning application-level names or assuming conventional callee behavior.

## Evidence

The contract is based solely on the supplied static representation: displayed access widths, constants, argument relationships, invocation order, and recovery warnings.

Direct memory widths and ordering have stronger support than calling-convention details, auxiliary-value provenance, or terminal control-flow interpretation. The overlapping-storage warning limits conclusions about independent objects. The terminal-transfer warning limits conclusions about normal return and complete function boundaries.

No runtime tests, dependency inspection, binary inspection, or independent memory-layout validation have been performed. The acceptance examples are arithmetic and temporal consequences under their stated conditions, not observed execution results.
