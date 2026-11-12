<!-- Trusted provenance: F-c04bae1d49ac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c04bae1d49ac

## Purpose

F-c04bae1d49ac performs an ordered sequence of 53 indirect invocations, one separate direct invocation of F-3de949054bdd, ten coprocessor-transfer operations, and a terminal 16-bit memory write. The supplied static representation ends at an invalid-instruction boundary; it establishes neither an ordinary return nor successful completion.

No application-level purpose is established. Descriptions such as initialization, decoding, device configuration, or memory management would be inferred intent, not supported behavior.

This contract distinguishes three evidence levels:

- **Established static evidence:** represented operations, operand relationships, and their order.
- **Conditional arithmetic deductions:** consequences of interpreting those operations as fixed-width machine operations.
- **documented_only unknowns:** dependency contracts, platform behavior, instruction validity, and other unresolved properties.

All downstream behavior is conditional on earlier invocations and architectural operations completing and permitting continuation. The terminal write is not established as reachable on an actual system.

## Interface

### Declared inputs

| Input | Represented width | Explicit consumption |
|---|---:|---|
| First argument | 32 bits, uninterpreted | None |
| Second argument | 32 bits, uninterpreted | None |
| Third argument | Signed 32 bits | Shifted left by 22 positions for F-d009 |

The first two arguments have no explicit uses in the supplied representation. This does not establish observational independence: hidden calling-convention inputs and dependency behavior remain unknown.

### Additional entry state

The body also consumes:

- Two incoming 32-bit general-purpose values, designated **A** and **B**.
- An incoming link value used to derive F-d001’s target.
- Seven incoming 32-bit coprocessor-related operands, distinguished by slots 1, 3, 5, 6, 9, 10, and 15.
- Any additional register, memory, global, or device state consumed by dependencies.

These are environmental inputs, not additional declared parameters. The slot numbers distinguish operand identities; they are not assertions that the operands contain those numerical values.

F-d005 additionally receives an auxiliary 32-bit register value exposed by the preceding invocation sequence. Its producer and initialization requirements are unresolved.

### Output and termination

No ordinary return value or normal return path is established. The last represented ordinary-memory operation writes the numerical halfword `0x00CD`. An invalid-instruction boundary follows it.

That boundary does not establish a deliberate processor halt, a particular exception, an exit status, or any continuation.

### Widths and arithmetic

For the conditional deductions below:

- Words, double words, and halfwords have widths of 32, 64, and 16 bits respectively.
- Left shifts retain only the destination-width bit pattern.
- Unsigned right shifts insert zero bits.
- Signed right shifts are interpreted arithmetically where identified.
- Extracting either word of a double word preserves that word’s 32-bit pattern.
- Address arithmetic uses the represented machine-width interpretation.

These assumptions describe bit-level behavior, not the definedness of corresponding signed expressions in a high-level language.

No buffer, string, structure, allocation layout, or object-relative addressing scheme is established.

## Behavior

### Invocation identities and ordering

The indirect invocation sites are designated **F-d001 through F-d053**, in execution order. These are site identities, not proof of 53 distinct destination functions. There is also one direct invocation of **F-3de949054bdd**.

An invocation described without arguments has no explicitly represented arguments. It may still consume hidden machine state. Result widths below describe the representation’s use of results, not independently verified dependency signatures.

The phase descriptions collectively specify the complete represented order. Transfers occur at the positions stated, not as a separate batch.

### Initial argument processing

F-d001’s target is derived from the incoming link value plus `0xEB6`. The source does not establish target validity, instruction mode, or successful invocation.

F-d002 receives A shifted left by 14. F-d003 and F-d004 follow. F-d005 then receives, in order:

1. A shifted left by 8.
2. The auxiliary 32-bit register value.
3. B shifted left by 26.

F-d006 and F-d007 follow before F-d008 produces a retained 32-bit result.

F-d009 receives that result unchanged, the third declared argument shifted left by 22, and the F-d008 result shifted left by 5. The last of these values is retained for later use.

F-d010 produces a 32-bit result forwarded to F-d011. The separate direct invocation of F-3de949054bdd follows F-d011 and precedes F-d012.

### Forwarding and first transfer group

F-d012 produces a 32-bit result forwarded to F-d013. F-d014 similarly supplies F-d015.

F-d016 receives B shifted left by 13 and a signed transformation of the retained F-d008 result: its 32-bit left-by-5 intermediate is shifted arithmetically right by 5. Under the stated arithmetic model, this sign-extends the original result’s low 27 bits; it is not generally an identity. F-d016’s 32-bit result is forwarded to F-d017.

F-d018 produces a 64-bit result. Its lower word supplies transfer 1. Its upper word, shifted left by 1, is the explicit argument to F-d019.

F-d020 then produces a 32-bit result used by both transfers 2 and 3. Those transfers are consecutive and differ in an incoming operand. F-d021 follows and produces another retained 32-bit result.

### High-bit value and intervening calls

Define **H** as the 32-bit result of shifting the F-d021 result left by 27. Before F-d022, a separate represented intermediate is explicitly set to zero.

F-d022 receives four explicit arguments: zero, zero, H, and zero. The next invocations are F-d023, F-d024, F-d025, F-d026, and F-d027, in that order.

F-d028 produces the 32-bit data for transfer 4. F-d029, F-d030, and F-d031 follow. Transfer 5 then receives H shifted left by 24. F-d032 receives that same shifted value and zero.

F-d033 follows. F-d034 supplies the 32-bit data for transfer 6, after which F-d035 is invoked.

Under 32-bit truncating shifts, H can contain set bits only in positions 27–31. Consequently:

- H shifted left by 24 is zero.
- H masked with `0x007FFFFF` is zero.

Thus transfer 5 receives zero, and both explicit arguments to F-d032 are zero. These conclusions do not depend on F-d021’s result.

### Second result-derived transfer group

The separate intermediate set to zero before F-d022 has no represented reassignment before F-d036. F-d036 receives that intermediate shifted left by 22, the same intermediate shifted left by 11, and zero. All three explicit arguments are therefore zero in the represented dataflow.

Let **R** be the retained 32-bit result from F-d036. Transfer 7 receives R shifted left by 1. Define **S** as R shifted left by 4.

F-d037 receives:

1. R shifted left by 1.
2. H masked with `0x007FFFFF`, then shifted left by 4.
3. S.
4. Zero.

Its second explicit argument is zero under the stated model.

F-d038 and F-d039 follow. Transfer 8 receives zero. A **selector value** is then formed by masking S with `3` and shifting the result left by 16. Transfer 9 receives zero.

F-d040 receives one explicit argument, zero, and produces a retained 32-bit result. Transfer 10 receives S shifted left by 8. F-d041 then receives S shifted left by 8, S shifted left by 12, and the F-d040 result shifted left by 9.

Because S was formed by a four-position left shift, its low four bits are zero. The selector value is therefore zero. This is a local arithmetic deduction, not a claim that intervening dependencies preserve arbitrary registers or globals.

### Final forwarding chains

F-d042 produces a 64-bit result. F-d043 receives its lower word, its upper word, and its lower word shifted left by 1.

F-d044 receives the selector value shifted left by 8, which is zero under the stated model. Its 32-bit result is forwarded to F-d045; F-d045’s 32-bit result is forwarded to F-d046. F-d047 follows.

F-d048 produces a 64-bit result. F-d049 receives its lower word, its upper word, and its upper word shifted left by 11. F-d049 itself produces a retained 64-bit result.

An unsigned right shift of that result by 41 produces a value with at most 23 bits. Separately, the selector value is shifted left by 25 for later use; this remains zero.

F-d050 receives the lower word of the F-d049 result. F-d051 then supplies a 32-bit result to F-d052, whose second explicit argument is the shifted selector value. F-d053 follows F-d052.

### Terminal destination and store

Let **E** be the unsigned 64-bit F-d049 result shifted right by 41. Its range is zero through `0x007FFFFF`.

The terminal address calculation first interprets E as signed 32-bit and shifts it right by 11. This conversion cannot make E negative because its highest possible set bit is bit 22. Thus the signed shift has the same numerical result as a nonnegative right shift.

Let **n** be that result. The destination is the numerical address:

**n + floor(n / 4)**

Equivalently, n is the integer represented by bits 52–63 of the F-d049 result.

The resulting bounds and layout properties are:

- n ranges from 0 through 4095.
- The address ranges from 0 through 5118, inclusive.
- The maximum is `0x13FE`.
- These bounds do not imply that every address in the interval is attainable.
- No object or allocation base is added.
- Address zero and odd addresses are possible.
- The addition cannot overflow a 32-bit word within these bounds.

The function attempts a 16-bit store of `0x00CD` at this address. The representation provides no address, permission, or alignment check before the store. The invalid-instruction boundary follows.

## Side effects

### Ordinary memory and state

The terminal halfword store is the only explicit ordinary-memory write in the represented body. There are no explicit ordinary-memory data reads outside dependency execution.

This does not describe all execution effects. Dependencies may read or write memory, globals, registers, devices, or other state. Instruction fetching and coprocessor operations may have additional architectural effects.

The invocation and transfer order is observable in principle. No reordering across unknown dependencies is justified.

### Coprocessor operations

There are **ten** represented coprocessor-transfer operations:

| Transfer | Position | Data operand | Fixed operand triple | Incoming operand slots |
|---:|---|---|---|---|
| 1 | After F-d018, before F-d019 | F-d018 lower word | `(0, 6, 7)` | 3, 15 |
| 2 | After F-d020 | F-d020 result | `(0, 3, 7)` | 6, 15 |
| 3 | Immediately after transfer 2 | Same F-d020 result | `(0, 3, 7)` | 15, 15 |
| 4 | After F-d028, before F-d029 | F-d028 result | `(0, 1, 7)` | 1, 15 |
| 5 | After F-d031, before F-d032 | H shifted left by 24 | `(0, 5, 7)` | 5, 15 |
| 6 | After F-d034, before F-d035 | F-d034 result | `(0, 6, 7)` | 9, 15 |
| 7 | After F-d036, before F-d037 | R shifted left by 1 | `(0, 5, 7)` | 10, 15 |
| 8 | After F-d039 | Zero | `(0, 7, 7)` | 9, 15 |
| 9 | After selector formation, before F-d040 | Zero | `(0, 0, 7)` | 15, 15 |
| 10 | After F-d040, before F-d041 | S shifted left by 8 | `(0, 2, 7)` | 9, 15 |

The incoming slots identify supplied operands, not fixed selector values. Their interpretation, encoding, privilege requirements, and effects are documented_only.

### Ownership and failure behavior

No allocation, release, ownership transfer, object retention, or lifetime policy is established. Forwarded words must not be assumed to denote pointers or owned resources.

No explicit error check, retry, rollback, cleanup branch, or recovery path appears. A nonreturning dependency, invalid invocation target, unsupported coprocessor operation, or faulting store can prevent subsequent operations. Error signaling and external handling are unknown.

The halfword’s numerical value and width are established. Byte order, write atomicity, and behavior at an unaligned or inaccessible destination are not.

## Dependencies

- **F-d001:** indirect invocation whose target depends on the incoming link value plus `0xEB6`.
- **F-d002–F-d053:** indirect invocation sites with targets formed from represented fixed-location-relative calculations. Their semantic identities and executable validity are unresolved.
- **F-3de949054bdd:** one direct invocation, between F-d011 and F-d012. No explicit arguments or consumed result are represented.

Multiple indirect sites may resolve to the same destination. Explicit argument lists and consumed result widths do not establish complete calling conventions.

No dependency is assumed to preserve globals, preserve arbitrary registers, own or release memory, or return normally. The described local dataflow records the supplied representation; it is not an independent preservation guarantee.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes that execution reaches the relevant operation, dependencies permit continuation, coprocessor operations are supported, and the stated arithmetic model applies.

1. **Third-argument truncation.** A third-argument bit pattern of `1` supplies `0x00400000` as F-d009’s second explicit argument. A pattern of `0x00000400` supplies zero because its shifted set bit falls beyond 32 bits.

2. **Signed reconstruction.** If F-d008 returns `0x04000000`, its left-by-5 intermediate is `0x80000000`. Arithmetic right shift by 5 supplies `0xFC000000` to F-d016, not the original result.

3. **High-bit elimination.** For every F-d021 result, transfer 5 receives zero, F-d032 receives zero and zero, and F-d037’s second explicit argument is zero.

4. **Explicit zero intermediate.** F-d036 receives three explicit zero arguments because the represented intermediate was set to zero and has no intervening represented reassignment. This does not establish the absence of dependency side effects.

5. **Selector elimination.** For every F-d036 result, S has zero low four bits. F-d044’s explicit argument and F-d052’s second explicit argument are therefore zero.

6. **Address independence from lower bits.** Two F-d049 results with identical bits 52–63 yield the same numerical terminal address, regardless of their lower 52 bits. Their earlier forwarded lower words may differ, so this does not imply identical overall execution effects.

7. **Minimum destination.** If bits 52–63 are zero, the terminal store targets numerical address zero. No null-address guard is represented.

8. **Maximum destination.** If bits 52–63 are all set, n is 4095 and the destination is 4095 plus 1023, or `0x13FE`. The stored numerical halfword remains `0x00CD`.

9. **Unaligned destination.** If n is one, the destination is address one. Successful storage, an alignment fault, or other behavior depends on the platform.

10. **Interrupted sequence.** If F-d028 does not return, transfer 4 and every later represented operation are not reached through this sequence. No local recovery path is established.

## Uncertainties

The following remain **documented_only**:

- Whether the represented sequence corresponds to valid executable instructions.
- Whether execution reaches any particular invocation, transfer, or terminal store.
- Actual calling conventions, hidden inputs, and dependency signatures.
- The auxiliary register value’s producer and initialization.
- Indirect destination identities and validity.
- Coprocessor operand meaning, privilege requirements, state effects, and fault behavior.
- Actual machine semantics underlying represented signed shifts and signed left shifts.
- Memory mapping, permissions, alignment requirements, byte order, and write atomicity.
- Whether a terminal address denotes RAM, device memory, unmapped space, or another resource.
- Any continuation or external handling after the invalid-instruction boundary.
- Application intent and caller expectations.

The arithmetic deductions are conditional properties of the represented dataflow. They do not resolve instruction validity or prove runtime reachability.

## Evidence

This contract uses only the supplied static source representation. Prior prose and reviewer feedback are not independent evidence of behavior.

Established evidence comprises three declared 32-bit inputs, 53 indirect invocation sites, one separate direct invocation, ten coprocessor-transfer operations, the represented forwarding and ordering, fixed shifts and masks, the terminal 16-bit store of `0x00CD`, and the invalid-instruction boundary.

Sign extension, zero-valued derived arguments, selector elimination, and terminal address bounds are arithmetic deductions under the explicitly stated machine-width model.

No runtime tests have been performed. No dependency implementation, external interface specification, calling convention, or platform behavior has been independently verified.
