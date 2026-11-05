<!-- Trusted provenance: F-9d25eab73bd1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9d25eab73bd1

## Purpose

F-9d25eab73bd1 has a statically represented prefix containing two dependency calls and five explicit memory writes. It obtains a selector byte through the first dependency’s result, updates two indexed destinations, derives a value from a stack address, and performs additional selector-based and stack-value-based writes.

An initialization or table-update role is **inferred intent**, not an established requirement. No object schema, protocol, ownership transfer, or success condition is established.

The supplied representation reports invalid instruction data and truncated control-flow recovery. This contract describes the represented prefix, conditional on execution following that prefix. It does not establish a complete executable function, normal return, or actual runtime halt at the terminal evidence boundary.

## Interface

### Explicit arguments

| Argument position | Represented width | Established use |
|---|---:|---|
| First | 32-bit signed integer | Passed to the first dependency, shifted left by 16 for another call argument, and multiplied by 256 to form two destination offsets. |
| Second | 32 bits; interpretation unspecified | Passed unchanged in the represented expression to the first dependency. |
| Third | 32 bits; interpretation unspecified | No explicit use in the represented prefix. |
| Fourth | 32-bit signed integer | Supplies the base for the first indexed write. |

No argument validation is represented. The lack of a visible use for the third argument does not establish that callers can omit it or that its incoming machine state is irrelevant outside the recovered expressions.

### Implicit inputs

The prefix also depends on:

- An incoming return-link value used to derive the first call target.
- An incoming register-held address used for an unsigned byte read.
- A stack-associated 32-bit value identified by relative offset `0x14`.
- A stack-associated 32-bit value identified by relative offset `0x2e8`.
- The address of a stack location identified by relative offset `0x38c`.
- Memory at fixed and computed addresses.

These are represented frame references, not a verified external calling convention. The evidence does not establish that the offsets are measured from the caller’s incoming stack pointer.

The incoming register-held address and stack-associated values may indicate an unusual calling convention, incomplete boundary recovery, or damaged analysis. None of those explanations is selected by the available evidence.

### Outputs and completion

No ordinary return value is represented, and no normal return is visible. The established outputs are the calls and memory effects of the prefix.

The terminal invalid-instruction marker does not establish an error code, termination API, processor exception, or guaranteed nonreturning behavior.

### Width and arithmetic model

The representation distinguishes unsigned 8-bit reads, a 16-bit read of unresolved signedness, and 32-bit word operations. The first write stores a pointer; its four-byte size is conditional on the apparent 32-bit address model.

For address calculations and stored values, this contract gives arithmetic examples under a **conditional 32-bit machine-word model**: results are reduced modulo \(2^{32}\) when necessary. This model describes potential machine-word results; it does not make signed overflow or an invalid shift well-defined in the source-level expressions.

Important distinctions:

- The first argument’s left shift and indexed signed arithmetic have source-level validity constraints not established for all inputs.
- The selector’s unsigned multiplication by `0x400000` fits within 32 bits for every possible selector.
- Subtracting `0x48` from the unsigned shift result uses unsigned arithmetic.
- The later byte-minus-`0x44` expression first uses signed integer promotion, then converts the result for the unsigned 32-bit store.

Byte order, permitted alignment, and architectural treatment of oversized shift counts remain unresolved.

## Behavior

### 1. Invoke the initial dependency and read the selector

The first represented operation invokes **F-000000000001** at a target formed by adding `0xcfe` to the incoming return-link value.

The arguments, in order, are:

1. The first explicit argument.
2. The second explicit argument.
3. The first argument shifted left by 16 bits.

Under the conditional 32-bit word interpretation, the third supplied value has zeroes in its lower 16 bits and the original argument’s lower 16 bits in its upper half. This describes the truncated bit pattern only; it does not establish that the signed shift is valid for every input.

The returned value is used as an address basis. The next represented access reads one unsigned byte at that value plus `0x1e`. This byte is the **selector**, ranging from 0 through 255.

No null check, range check, error-result check, or accessibility check is represented before this read. Continuing execution requires the returned address basis to permit the access.

### 2. Perform the first indexed write

The function writes the fixed pointer value `0x08983eb0` at:

- The fourth explicit argument, plus
- The first explicit argument multiplied by `0x100`.

This is a 256-byte stride. It does not establish an allocated array, an element size, or a valid index range.

Under the conditional 32-bit model, the destination is the low 32 bits of that sum. Source-level signed multiplication or addition may be invalid for some inputs; no guard is represented.

### 3. Read the incoming-address byte and perform the second indexed write

After the first indexed write, the function reads one unsigned byte from the incoming register-held address. This is the **sample byte**, retained for the later word write.

The order is observable when addresses alias. If this read overlaps the preceding pointer write, it observes the memory state after that write, subject to the platform’s access and byte-order rules. Moving the read before the write would not preserve the represented behavior.

The function then performs a 32-bit write at:

- The stack-associated value identified by offset `0x2e8`, plus
- The first explicit argument multiplied by `0x100`.

The value written is the 32-bit **contents** of fixed address `0x0898401c`, not the address itself.

The operation follows the sample-byte read. The representation does not resolve the relative evaluation order of the fixed-address source load and acquisition of the stack-associated base within this operation. It also does not provide a complete instruction-level schedule for reads of recovered stack values.

The first dependency must not be assumed to preserve either the fixed source word or stack-associated memory.

### 4. Derive a value from a stack address

The function obtains the address of the stack location identified by offset `0x38c` and interprets that address as an unsigned word.

The shift count is the address’s low eight bits, giving a represented count from 0 through 255. The operation is a logical right shift of the unsigned address word by that count.

For counts from 0 through 31, the conditional 32-bit interpretation determines the result. Counts from 32 through 255 are **documented_only**: the source-level expression does not establish a defined result, and the supplied evidence does not establish an architectural substitute.

In particular, this contract does not assume count masking, a zero result, or an exception for oversized counts.

The resulting **shift value** supplies a later byte write and the first argument of the second dependency.

### 5. Read a selector-addressed halfword and write a byte

The selector is multiplied by `0x400000` to form the **selector basis**. This establishes a 4 MiB stride.

For selectors from 0 through 255, the basis ranges from `0x00000000` through `0x3fc00000`. The multiplication fits within a 32-bit word, and its maximum also fits within the represented signed 32-bit destination.

The function next reads a 16-bit quantity beginning at the absolute address:

- Selector plus `0x3c`.

This is not an offset into the selector basis or the first dependency’s result. Possible starting addresses range from `0x3c` through `0x13b`. Odd selectors produce odd halfword addresses; no alignment adjustment is represented.

After that read, the function writes one byte at:

- Selector basis minus `0x92`.

Under the conditional byte-storage model, the stored byte is the low eight bits of the shift value. This write occurs before the second dependency call.

For selector zero, the destination is `0xffffff6e` under the conditional 32-bit address model. There is no represented special handling for this wraparound.

### 6. Invoke the second dependency

The function invokes **F-000000000002** with four arguments, in order:

1. The shift value minus `0x48`.
2. The constant `0x9d`.
3. The previously read 16-bit quantity.
4. The fixed address `0x0898417c`.

The first argument is derived by unsigned subtraction, wrapping modulo \(2^{32}\) under the conditional word model.

The 16-bit memory pattern is established, but its wider argument interpretation is not. The evidence does not resolve sign extension, zero extension, or other calling-convention treatment.

No return value from this dependency is visibly consumed. Nothing establishes the type, extent, mutability, lifetime, or ownership of the data referenced by the fourth argument.

### 7. Perform the post-call word write

After the second dependency returns, the function writes an unsigned 32-bit value at:

- Selector basis minus `0x5a`.

The stored value derives from the previously captured sample byte, not a new byte read after the call.

The arithmetic has two distinct stages:

1. The unsigned 8-bit sample undergoes integer promotion to signed 32-bit integer because all values from 0 through 255 fit that type. Subtracting `0x44` produces a signed intermediate ranging from **−68 through 187**.
2. Assignment to the unsigned 32-bit destination converts that intermediate modulo \(2^{32}\).

Thus, negative intermediate results become large unsigned stored words. This is not an unsigned-byte-to-unsigned-word promotion before subtraction.

The destination is two bytes past a four-byte boundary: the selector basis is divisible by four, whereas `0x5a` is not. Whether this word write is permitted is platform-dependent.

The second dependency may affect the destination’s contents before this write. It must not be assumed to preserve unrelated memory or external state.

### 8. Perform the final byte write

The final represented write uses the stack-associated 32-bit value identified by offset `0x14`:

- Destination: that value plus `0x1e`.
- Stored byte: the low eight bits of that value under the conditional byte-storage model.

The value itself supplies both the destination basis and the byte. There is no represented dereference of that value to obtain the stored byte.

The representation does not establish that the associated stack storage remains unchanged across either dependency call, or exactly when the recovered value is materialized.

No further valid operation or normal return is established.

## Side effects

There are **five explicit writes**, in this represented order:

| Order | Destination | Stored content |
|---:|---|---|
| 1 | Fourth argument plus first argument times `0x100` | Fixed pointer `0x08983eb0` |
| 2 | Stack-associated base at offset `0x2e8`, plus the same indexed offset | Word read from `0x0898401c` |
| 3 | Selector basis minus `0x92` | Shift-derived byte |
| 4 | Selector basis minus `0x5a` | Sample-minus-`0x44` result converted to unsigned 32 bits |
| 5 | Stack-associated value at offset `0x14`, plus `0x1e` | Low byte of that value |

Both dependencies may have additional side effects. Their calls are not evidence of memory preservation, ownership transfer, allocation, release, synchronization, or guaranteed completion.

Aliasing is not prohibited. Earlier writes can affect later memory reads, and different address formulas may overlap. The second dependency occurs between the third and fourth explicit writes and may change memory before the final writes.

There is no represented rollback, recovery path, or explicit error report. If a call does not return or a memory access cannot complete, subsequent effects are not established. Earlier completed effects are not shown being undone.

## Dependencies

These IDs are document-local opaque dependency identifiers, not verified external symbol mappings.

| Opaque ID | Invocation | Established interaction |
|---|---|---|
| F-000000000001 | Indirect target derived from the incoming return-link value plus `0xcfe` | Receives three arguments. Its result supplies the basis for a byte read at offset `0x1e`. |
| F-000000000002 | Fixed direct target | Receives the derived unsigned word, `0x9d`, the 16-bit quantity, and fixed address `0x0898417c`. No returned value is visibly used. |

Neither dependency has an established preservation, ownership, failure, or return guarantee.

The terminal invalid-instruction marker is an evidence boundary, not an established callable dependency.

## Acceptance examples

These are conditional arithmetic and ordering criteria, not runtime observations. They assume execution follows the represented prefix, dependencies return where needed, required accesses are permitted, and the stated memory values apply when read.

### Indexed destinations

For first argument `2` and fourth argument `0x20000000`, the first destination is `0x20000200` and receives pointer value `0x08983eb0`.

If the second indexed base is `0x30000000` and the fixed source word contains `0x11223344` when read, the second destination is `0x30000200` and receives `0x11223344`.

### Shifted first-call argument

For first argument `1`, the first dependency receives `0x00010000` as its third argument.

For first argument `0x12345678`, conditional 32-bit truncation gives `0x56780000`. This latter example describes a machine-word bit pattern only; it does not establish validity of the signed source-level shift.

### Selector addressing

For selector `1`:

- The halfword read begins at `0x3d`.
- The byte destination is `0x003fff6e`.
- The later word destination is `0x003fffa6`.

The halfword and word accesses retain their alignment preconditions.

### Stack-address-derived shift

If the relevant stack address is `0x10000808`, its low byte gives count 8. The logical shift produces `0x00100008`.

The selector-based byte write stores `0x08`, and the second dependency’s first argument is `0x000fffc0`.

An address whose low byte is `0x28` supplies count 40. No shift result is specified by this contract for that case.

### Signed intermediate and unsigned storage

| Sample byte | Signed intermediate after subtraction | Unsigned 32-bit stored word |
|---:|---:|---:|
| `0x00` | −68 | `0xffffffbc` |
| `0x43` | −1 | `0xffffffff` |
| `0x44` | 0 | `0x00000000` |
| `0xff` | 187 | `0x000000bb` |

A byte-sized store would not satisfy this word-write behavior.

### Aliasing and final byte

If the incoming register-held address designates a byte overwritten by the first pointer write, the sample-byte read uses the post-write byte. Its numerical value requires the platform’s byte order.

If the value supplying the final write is `0x200012ab`, the destination is `0x200012c9` and the stored byte is `0xab`. This does not assert that an earlier observation of the corresponding stack storage remains valid across dependency calls.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered prefix is an executable path.
- The true function boundary and calling convention.
- Runtime behavior at the terminal invalid-instruction boundary.
- Shift behavior for counts from 32 through 255.
- Native pointer width and arithmetic beyond the conditional 32-bit model.
- Validity of signed shifts, overflow-prone address expressions, and integer-to-address interpretation.
- Byte order and support for unaligned accesses.
- Wider interpretation of the 16-bit call argument.
- Exact instruction-level timing of recovered stack-value reads.
- Contents, types, extents, and accessibility of referenced storage.
- Dependency side effects, memory preservation, ownership behavior, and failure or return guarantees.
- Whether the low-address read and widely separated writes are intentional.

A table-update or initialization interpretation remains inferred intent, not an acceptance requirement.

## Evidence

The supplied static representation establishes the argument uses, five explicit writes, access widths, address formulas, call arguments, integer-promotion sequence for the sample-byte subtraction, and represented operation order.

The same representation explicitly reports invalid instruction data and truncated control-flow recovery. Those limitations are retained rather than replaced with assumed application semantics.

No implementation access, dependency inspection, independent machine-level verification, or runtime tests have been performed. The acceptance examples are conditional consequences of the stated model, not measured results.
