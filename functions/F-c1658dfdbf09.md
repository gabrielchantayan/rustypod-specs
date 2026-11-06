<!-- Trusted provenance: F-c1658dfdbf09; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c1658dfdbf09

## Purpose

F-c1658dfdbf09 selects between two substantially different execution paths using an incoming overflow-condition value that is separate from its two explicit arguments.

The supplied static representation establishes indirect-call ordering, arithmetic dependencies, and two caller-side memory writes on one path. It does not establish an application-level purpose or complete control flow. The overflow-selected path ends at an undecodable instruction boundary. The other path ends with an unresolved indirect transfer represented as a call.

This contract distinguishes:

- **Established static evidence:** operations, types, dependencies, and ordering visible in the supplied representation.
- **Inferred intent:** plausible interpretations that do not constitute required behavior.
- **documented_only unknowns:** behavior that the supplied material does not resolve.

In particular, neither memory write establishes an object format, allocation protocol, or ownership transfer.

## Interface

### Explicit inputs

| Input | Apparent width and interpretation | Visible use |
|---|---|---|
| First argument | Unsigned 32-bit integer | Logical right shift by 25 on the overflow-selected path |
| Second argument | Unsigned 32-bit integer | Logical right shift by 29, followed by unsigned subtraction of 59, on that path |

These widths use the apparent 32-bit integer model of the supplied representation. An independently verified machine ABI is unavailable.

Neither explicit argument appears in a caller-side expression on the non-overflow path. This does not prove that dependencies cannot observe incoming registers or other machine state.

### Additional incoming context

Execution also depends on:

- An incoming overflow-condition value, tested for zero.
- An incoming link-register-derived value used to select one dependency on each path.
- An unresolved code-address-like value used as an arithmetic operand and write-address basis on the overflow-selected path, and as the final transfer target on the other path.
- A separately referenced data item read while preparing the final transfer.

The unresolved code-address-like value has no visible initialization. It is not an established third argument.

The separately referenced data item has **no supplied declaration establishing its type or access width**. Its use in a left-shift expression does not, by itself, establish a 32-bit memory read.

### Outputs and completion

The recovered interface has no explicit return value.

- The overflow-selected path has no established normal return. Its visible sequence ends at an undecodable boundary.
- The non-overflow path shows a return after the final transfer, if that transfer behaves as a returning call. The transfer’s actual control-flow semantics remain **documented_only**.

### Preconditions for completing the visible operations

Reaching the displayed effects without an earlier fault requires suitable execution context:

- Reached indirect targets must be executable under the actual calling convention.
- The first store destination must permit the represented 16-bit write.
- The address derived from F-01’s result must permit the represented 32-bit write at byte offset 8.
- The separately referenced data item must be readable using its actual, unrecovered access width.
- Address calculations and any signed arithmetic must have behavior supported by the actual execution environment.

No visible validation establishes these conditions. Alignment, executable-address tagging, pointer representation, and fault handling are unresolved.

## Behavior

### Path selection

The incoming overflow condition is tested before any visible call or memory write.

- Nonzero selects the overflow path.
- Zero selects the non-overflow path.

This condition is not visibly calculated from either explicit argument. Replacing it with argument overflow detection or a range check would change the established selection rule.

### Overflow-selected path: subtraction and conversion

The first argument is logically shifted right by 25. Under the apparent unsigned 32-bit model, the result is an integer from 0 through 127.

The unresolved code-address-like value is converted to a signed integer before participating in subtraction. The meaning of that pointer-to-integer conversion is not independently established.

The subtraction itself must be distinguished from its later assignment:

1. The shifted first argument has unsigned 32-bit type.
2. Under the apparent model’s arithmetic conversion rules, the signed integer obtained from the address-like value is converted to unsigned 32-bit form for subtraction.
3. The subtraction therefore occurs **modulo 2³²**.
4. The resulting unsigned value is converted into a signed 32-bit value for subsequent use.

For example, if the signed integer obtained from the address-like value is zero and the shifted argument is one, the subtraction result is the unsigned value 0xFFFFFFFF. That modular result is established under the stated model. Its subsequent conversion into signed storage is a separate question.

An unsigned result from 0 through 0x7FFFFFFF is representable as a signed 32-bit integer. The supplied material does not independently establish the treatment of larger unsigned results. Interpreting 0xFFFFFFFF as −1 is a conventional machine interpretation, but it is not an unconditional conversion guarantee supplied by this representation.

Thus, the following are distinct:

- Pointer-to-signed-integer conversion.
- Unsigned modular subtraction.
- Conversion of the unsigned result into signed storage.

Only after those stages is the signed derived value available for the store, dependency argument, and later multiplication.

### Overflow-selected path: first write and dependency call

Before F-01 is called, the derived signed value is converted to a signed 16-bit quantity and written at byte offset 36 from the unresolved address-like value.

The represented store width is 16 bits. For values representable in signed 16-bit form, the numeric narrowing is unambiguous. For values outside that range, the supplied representation does not independently establish the narrowing rule. Under a conventional truncating machine interpretation, the stored bit pattern is the low 16 bits of the derived value.

Byte order and unaligned-store behavior remain unknown.

F-01 is then invoked at the incoming link-register-derived value plus 3892 bytes. Its four visible arguments are:

| Position | Visible value |
|---|---|
| 1 | The derived signed value |
| 2 | 44 |
| 3 | 141 |
| 4 | The unsigned second argument shifted right by 29, then reduced by 59 |

The fourth argument’s subtraction is unsigned under the apparent 32-bit model. The shift yields 0 through 7, so the resulting 32-bit patterns are 0xFFFFFFC5 through 0xFFFFFFCC inclusive.

Those patterns correspond to −59 through −52 if interpreted as signed two’s-complement values. F-01’s actual parameter types are unavailable, so that signed interpretation is not a recovered callee contract.

The first write precedes F-01. A dependency that reads the affected memory can therefore observe the write, subject to the actual memory model.

### Overflow-selected path: returned-address write and terminal boundary

If F-01 returns, its result is represented as a signed machine integer and used as an address basis. A 32-bit write is performed at byte offset 8 from that basis.

The value to be written is the earlier **signed derived value multiplied by two**. Unlike the first subtraction, this multiplication is represented as signed arithmetic.

Under a signed 32-bit model, multiplication by two remains representable when the derived value lies between −1,073,741,824 and 1,073,741,823 inclusive. Outside that interval, signed overflow behavior is not established. A wrapping result must not be required merely because the earlier subtraction was modular.

The returned-address calculation also carries unresolved signed-address arithmetic and integer-to-pointer conversion semantics. No null check, range check, status check, or validity check is visible.

After this write, the representation reaches an undecodable instruction boundary. That boundary does not establish a deliberate abort, portable exception, infinite loop, or normal return. The contract does not require manufacturing any particular terminal mechanism.

### Non-overflow path: preliminary calls

Seven preliminary indirect calls are shown in this order:

| Order | Dependency | Target selection | Explicit result use |
|---:|---|---|---|
| 1 | F-02 | Incoming link-register-derived value plus 3680 bytes | None shown |
| 2 | F-03 | Fixed encoded target | None shown |
| 3 | F-04 | Fixed encoded target | None shown |
| 4 | F-04 | Same encoded target as the preceding call | None shown |
| 5 | F-05 | Fixed encoded target | None shown |
| 6 | F-06 | Fixed encoded target | None shown |
| 7 | F-07 | Fixed encoded target | Supplies the final transfer’s first argument |

The repeated F-04 target is established by the represented target calculations. These are two consecutive invocations, not one invocation and not two independently identified functions.

No explicit arguments are shown for these seven calls. This does not establish that their real signatures have zero parameters. Hidden inputs, retained register contents, and unrecovered calling-convention details remain unknown.

Each later call is reached only if execution continues through the earlier calls.

### Non-overflow path: late data read and final transfer

After F-07 returns, its represented 32-bit result supplies the first visible argument to F-08.

The separately referenced data item is then read for an expression that shifts its value left by one. That expression supplies F-08’s second visible argument.

The source does not declare the data item. Consequently, the following are unrecovered:

- Its stored type.
- Its memory access width.
- Its signedness.
- Any integer promotions affecting the shift.
- The effective width and overflow rules of the shifted expression.

The established ordering is nevertheless clear: the data read occurs **after the seven preliminary calls**, not at function entry. Those dependencies must not be assumed to preserve the item.

**Conditional interpretation:** if the item and effective shift expression are unsigned 32-bit values, shifting left by one discards the original top bit and introduces a zero low bit. This interpretation is not evidence that the actual memory access is 32 bits wide.

F-08 uses the unresolved target. The representation treats this transfer as a call and shows a return afterward, but explicitly reports that the underlying transfer was not recovered reliably. Its actual dispatch, continuation, and return behavior remain **documented_only**.

## Side effects

### Caller-side effects

On the overflow-selected path, the visible order is:

1. A 16-bit write at offset 36 from the unresolved address-like value.
2. The F-01 call.
3. A 32-bit write at offset 8 from the returned address basis, if the call returns and execution reaches the store.
4. Arrival at the undecodable boundary, if preceding operations complete.

No rollback is shown. A fault or nonreturning dependency can leave the first write observable without the second write occurring.

On the non-overflow path, caller-side effects comprise the ordered calls, the late data read of unrecovered width, and the unresolved final transfer. No direct caller-side memory write is shown on that path.

### Dependency effects and ownership

Dependency effects are unknown. The contract does not assume that any dependency:

- Preserves globals, condition flags, or reachable memory.
- Allocates, retains, transfers, or releases memory.
- Returns normally.
- Avoids callbacks or re-entry.
- Uses a particular error convention.

F-01’s result is used as an address basis, but that use does not establish allocation, ownership, lifetime, or object size.

No release operation, reference-count update, synchronization primitive, or transaction boundary is visible. Store atomicity and visibility to concurrent observers remain **documented_only**.

## Dependencies

The identifiers below are opaque references, not recovered names or verified signatures.

| ID | Established role |
|---|---|
| F-01 | Overflow-path dependency with four visible arguments; result becomes a write-address basis |
| F-02 | First non-overflow dependency, selected relative to incoming link-register-derived state |
| F-03 | Second non-overflow dependency |
| F-04 | Third and fourth non-overflow invocations, sharing one encoded target |
| F-05 | Fifth non-overflow dependency |
| F-06 | Sixth non-overflow dependency |
| F-07 | Seventh non-overflow dependency; represented 32-bit result feeds F-08 |
| F-08 | Final unresolved transfer, represented as a call with two visible arguments |

The same unresolved symbolic value appears as an address-like operand on the overflow path and as F-08’s target on the other path. Its runtime origin and stability are not independently established.

No dependency exception or failure is visibly translated into a caller-defined error result.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

### Incoming condition controls selection

For identical explicit arguments, changing only the incoming overflow condition from zero to nonzero selects the other visible path. Argument range checks are not an equivalent substitute.

### Upper-seven-bit dependence

With overflow selected and all other relevant context fixed:

- First arguments 0 and 0x01FFFFFF both contribute zero to the subtraction.
- 0x02000000 contributes one.
- 0xFFFFFFFF contributes 127.

These examples establish the shift contribution, not an unconditional signed interpretation of the subtraction result.

### Modular subtraction versus signed conversion

Assume the address-like operand’s signed integer conversion yields zero, and the shifted first argument is one.

The unsigned subtraction result is 0xFFFFFFFF under the apparent 32-bit model. A conversion rule mapping this pattern to −1 would produce a derived signed value of −1, but that mapping remains conditional. The contract must not merge the modular subtraction and signed conversion into an unqualified signed subtraction.

### Fourth-argument boundaries

With overflow selected, the fourth visible F-01 argument has these patterns under the apparent unsigned 32-bit model:

| Second argument | Fourth-argument pattern |
|---|---|
| 0 | 0xFFFFFFC5 |
| 0x1FFFFFFF | 0xFFFFFFC5 |
| 0x20000000 | 0xFFFFFFC6 |
| 0xFFFFFFFF | 0xFFFFFFCC |

No clamping to zero or rejection is shown.

### Defined signed multiplication

If the derived signed value is 7, F-01 returns normally, and its returned address basis supports the required calculation and write, the subsequent 32-bit store at offset 8 stores 14.

If the derived signed value is 1,073,741,824, doubling exceeds signed 32-bit range. The contract does not require a particular wrapped result.

If F-01 does not return, the second write is not reached.

### Repeated dependency invocation

With overflow clear and continued execution through the preliminary sequence, F-04 is invoked twice consecutively. A stateful dependency therefore has two opportunities to change observable state.

### Late observation with conditional shift width

If a preliminary dependency changes the separately referenced data item, the later read uses the then-current value, subject to actual memory semantics.

If, additionally, the effective shift expression is unsigned 32-bit and the value supplied to it is 0x80000001, the second visible F-08 argument has bits 0x00000002. This example does not establish the item’s actual load width.

## Uncertainties

The following remain **documented_only**:

- Validity of the recovered entry and instruction boundaries.
- Origin and initialization of the unresolved address-like value.
- Actual ABI, hidden arguments, dependency signatures, and calling conventions.
- Pointer-to-integer and integer-to-pointer conversion rules.
- Out-of-range unsigned-to-signed conversion after the modular subtraction.
- Out-of-range signed 16-bit narrowing for the first store.
- Overflow behavior of the later signed multiplication and signed address calculation.
- The final data item’s type, access width, promotions, and shift semantics.
- Alignment, byte order, permissions, atomicity, and concurrent visibility.
- Actual control flow at the final unresolved transfer and beyond the undecodable boundary.
- Dependency side effects, ownership, failure conventions, and termination.

No higher-level purpose is established.

## Evidence

The sole evidence is the supplied static representation. It reports bad instruction data, truncated control flow, and an unresolved indirect transfer.

That representation supports the path condition, apparent explicit argument widths, unsigned subtraction rules under the stated model, subsequent signed conversion and multiplication, store widths and offsets, dependency ordering, repeated target, and late data dependency. It does not supply a declaration for the final data item or establish its load width.

No runtime tests, execution traces, binary validation, or independent dependency inspection have been performed. All acceptance examples are conditional deductions from static evidence, not observed execution results.
