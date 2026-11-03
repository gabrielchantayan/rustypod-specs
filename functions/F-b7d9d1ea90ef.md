<!-- Trusted provenance: F-b7d9d1ea90ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b7d9d1ea90ef

## Purpose

F-b7d9d1ea90ef performs three indirect calls, passes a 32-bit result from the first call into the second, and then attempts a 16-bit memory write at an address calculated from its third explicit input. The supplied representation ends at an invalid-instruction boundary rather than showing a normal return.

The function’s higher-level purpose is unknown. The evidence does not establish that the calculated address belongs to an allocated object, a device register, or any other particular address space. It also does not establish that the terminal boundary represents an intentional trap.

This contract describes the visible call sequence and memory operation. It does not assign application meaning to them or reconstruct behavior beyond the invalid-instruction boundary.

## Interface

### Explicit inputs

The visible interface contains four inputs, in order:

| Input | Visible width and interpretation | Visible use |
|---|---|---|
| First input | 32-bit opaque value | No explicit use in the supplied body |
| Second input | 32-bit opaque value | No explicit use in the supplied body |
| Third input | Signed 32-bit integer | Determines the destination of the final 16-bit store |
| Fourth input | 32-bit opaque value | Supplied as the second argument of the second indirect call |

The first and second inputs are not explicitly forwarded or inspected. This does not establish that their incoming machine-level locations are cleared before a dependency executes. Because the indirect-call signatures are incomplete, dependencies’ observations of residual register or stack contents remain unknown.

The third input is used as an integer, not visibly dereferenced as a pointer. Its derived value subsequently becomes a memory address.

### Implicit input

The first indirect-call target depends on an incoming link-register value. The target is formed by adding the byte offset `0xD2C` to that value.

Consequently, the four explicit inputs do not fully determine execution. Correct execution also depends on the implicit register state, executable target availability, and the applicable calling convention.

### Outputs and completion

The visible interface declares no return value. No normal return appears in the supplied body.

The following observable outcomes are possible, subject to the execution environment:

- A dependency may transfer control elsewhere, fail, or not return.
- An indirect call may fail before entering its target.
- The final memory store may succeed or fault.
- If execution reaches the represented terminal boundary, subsequent behavior is not established.

A result from the first dependency is used internally as a 32-bit argument to the second dependency. It is not a return value of F-b7d9d1ea90ef.

### Preconditions for reaching the final store

Reaching the store requires all three indirect calls to complete and return control normally. Their targets and calling conventions must be compatible with execution.

Successfully completing the store additionally requires the calculated address to permit a 16-bit write under the target’s memory-protection and alignment rules. No visible validation enforces these requirements.

## Behavior

### Dependency-call sequence

On the visible path, the function performs these actions in order:

1. It computes a target from the incoming link-register value and invokes the first dependency.
2. If that dependency returns, the function retains a 32-bit result from it.
3. It invokes the second dependency with two visible arguments: the retained result, followed by the fourth explicit input.
4. If the second dependency returns, it invokes the third dependency.
5. If the third dependency returns, it attempts the calculated 16-bit store.
6. The supplied representation then reaches an invalid-instruction boundary.

There is no visible conditional branch, retry, status check, or alternative path between these operations.

The first and third calls have no explicit arguments in the supplied representation. This is not proof that the actual callees have zero-argument interfaces. Their real signatures and any implicit inputs are unknown.

No result from the second or third call is visibly consumed. This establishes only the absence of a visible result-dependent decision; it does not establish that either dependency is side-effect-free.

### Call-target selection

The first target is dynamically derived from incoming register state.

The second target is formed from a literal address and a fixed offset. Its visible selection does not depend on the four explicit inputs.

The third target is formed from a code-associated anchor and a fixed offset. Its visible selection likewise does not depend on the explicit inputs. The exact identity and interpretation of that anchor are not established.

The three call sites are distinguishable, but the evidence does not prove that they always reach distinct implementations.

### Address calculation

The destination calculation uses the third input and has three relevant operations:

- A signed right shift by 11 bits produces a signed intermediate.
- That intermediate is converted to a 32-bit unsigned value and logically shifted right by 2 bits.
- The two terms are added to form the destination address.

Under the conventional 32-bit interpretation, let \(x\) be the signed third input and let \(s\) be its arithmetic right shift by 11. The destination is:

\[
A =
\left(
\operatorname{uint32}(s)
+
\left(\operatorname{uint32}(s) \mathbin{\mathrm{logical\ right\ shift}} 2\right)
\right)
\bmod 2^{32}.
\]

The unsigned conversion is important for negative inputs. The second term is not a signed division of \(s\) by four. For a negative \(s\), it is derived from the unsigned representation of that negative value.

For nonnegative inputs, arithmetic right shift corresponds to division by \(2048\), rounded down. Under the conventional arithmetic-shift interpretation, negative inputs also round toward negative infinity rather than toward zero.

The supplied representation supports this calculation, but the exact instruction semantics and address-width rules are not independently verified. The conditional examples below explicitly state the interpretation they require.

### Final memory write

The function attempts to store the 16-bit value `0x00CD` directly at address \(A\).

There is no visible:

- Object base added to the calculated address.
- Bounds check or null-address rejection.
- Alignment adjustment.
- Read-modify-write operation.
- Destination-content check.
- Recovery path for an invalid address.

The requested write width is two bytes. It is not a 32-bit write of the same numeric value.

The byte layout depends on target byte order:

| Byte order | Byte at \(A\) | Byte at \(A+1\) |
|---|---|---|
| Little-endian | `0xCD` | `0x00` |
| Big-endian | `0x00` | `0xCD` |

The evidence does not establish byte order, store atomicity, or the behavior of a faulting or unaligned store. In particular, it does not guarantee that a failed store leaves both destination bytes unchanged.

### Terminal boundary

The supplied representation stops immediately after the store at a boundary marked as invalid instruction data.

This is evidence of incomplete or invalidly decoded control flow, not proof of an intentional application-level halt. There is no basis for specifying a successful return, a particular exception type, process termination, or a continuation beyond that point.

## Side effects

### Direct effects

The only explicit data-memory modification in the visible body is the 16-bit store described above.

No explicit reads of the destination’s previous contents appear. The fourth input is forwarded as a value rather than visibly dereferenced.

The function also transfers control through three indirect calls. Those calls precede the direct store in program order.

### Dependency effects and ordering

The dependencies may read or modify memory, global state, device state, or other execution state. Their effects are unknown. No preservation of globals or external resources may be assumed.

The visible ordering guarantees are conditional on normal returns:

- The first dependency executes before the second is invoked.
- The second executes before the third is invoked.
- The third executes before the direct store is attempted.

These are control-flow guarantees, not independently established inter-thread synchronization or memory-barrier guarantees.

A dependency could modify memory that later becomes the destination of the direct store. The function does not visibly inspect such modifications before attempting its own write.

### Ownership

No explicit allocation, release, ownership transfer, or lifetime extension appears in the body.

The fourth input and the first dependency’s result are opaque words. They might encode addresses or handles, but the evidence does not establish that they do. Passing them to a dependency does not establish whether that dependency acquires, borrows, retains, or releases any associated resource.

The destination address is calculated numerically. No owned buffer or valid object extent is identified.

### Errors

No explicit error result or recovery path is present.

A dependency’s 32-bit result is forwarded without a visible success check. The other dependency results are not visibly tested. There is also no visible validation of call targets or of the final destination address.

## Dependencies

The following opaque identifiers are documentary labels for unresolved call sites. They do not assert recovered function identities or distinct implementations.

| Opaque ID | Invocation and visible role | Unknown contract |
|---|---|---|
| F-opaque-01 | First indirect call; target is incoming link-register value plus `0xD2C`; provides a 32-bit result | Actual arguments, calling convention, side effects, failure behavior, resource ownership |
| F-opaque-02 | Second indirect call; receives the first call’s result and the fourth input, in that order | Meaning of arguments, result type, side effects, failure behavior, resource ownership |
| F-opaque-03 | Third indirect call; target is derived from a code-associated anchor; no explicit arguments shown | Identity, actual arguments, result type, side effects, failure behavior, resource ownership |

The invalid-instruction boundary is not treated as a fourth ordinary dependency. Its representation does not establish a callable runtime service.

## Acceptance examples

These are conditional behavioral criteria, not executed tests.

### Result forwarding and order

Assume F-opaque-01 returns the 32-bit word `0x12345678`, the fourth input is `0x89ABCDEF`, and all three dependencies return normally.

The second dependency must receive `0x12345678` as its first visible argument and `0x89ABCDEF` as its second. The third dependency must execute afterward. The direct store must not precede any of these calls.

### Nonreturning dependency

If F-opaque-02 does not return, the visible path does not invoke F-opaque-03 or reach the direct store.

The same principle applies to earlier interruptions: an interruption during the first call prevents all later visible operations.

### Zero-address case

Assume conventional 32-bit arithmetic, arithmetic signed right shift, and a third input of zero.

Both address terms are zero, so the function attempts a 16-bit write of `0x00CD` at address zero. A faithful account must not silently replace this with a null-address check or an early return.

Whether that write succeeds is environment-dependent.

### Shift boundary and alignment

Under the same arithmetic assumptions:

| Third input | Signed shifted value | Calculated address |
|---|---:|---:|
| `2047` | 0 | 0 |
| `2048` | 1 | 1 |
| `4096` | 2 | 2 |
| `8192` | 4 | 5 |

These cases demonstrate both the 11-bit shift boundary and the addition of the unsigned shifted term.

Addresses 1 and 5 are odd. The visible function does not align them before attempting a 16-bit write. Success at either address therefore depends on the platform’s rules.

### Negative input

For a third input of \(-1\), assuming arithmetic signed right shift:

- The signed intermediate is \(-1\).
- Its unsigned 32-bit representation is `0xFFFFFFFF`.
- The logical right-shifted term is `0x3FFFFFFF`.
- The calculated address is `0x3FFFFFFE`.

An interpretation that uses signed division for the second term does not match the visible unsigned conversion.

### Store extent

If address \(A\) is writable and the store completes, the direct operation writes the 16-bit value `0x00CD` at \(A\), with byte placement determined by endianness.

This criterion does not constrain earlier dependency writes. It also does not establish what execution does after the terminal boundary.

## Uncertainties

The following remain **documented_only**:

- The function’s application-level purpose.
- Whether the supplied instruction interpretation is reliable throughout the body.
- Actual machine address width, byte order, instruction-set state, and calling convention.
- Exact signed-shift instruction semantics.
- The identities and full contracts of all indirect dependencies.
- Whether the no-explicit-argument calls have hidden or unrepresented arguments.
- Global-state preservation and ownership behavior across dependencies.
- Whether the calculated destination is ordinary memory, device memory, unmapped memory, or another region.
- Alignment, access-permission, and partial-write behavior.
- The meaning of the invalid-instruction boundary and any continuation beyond it.

The arithmetic examples are conditional because machine-level semantics have not been independently established. None of these unknowns is replaced by an assumed allocator contract, conventional error policy, or intentional-trap interpretation.

## Evidence

**Established static evidence:** The supplied representation shows four explicit 32-bit inputs, a link-register-derived first call, forwarding of its 32-bit result with the fourth input to a second call, a third indirect call, and a subsequent 16-bit store of `0x00CD`. The destination expression contains a signed shift by 11, an unsigned conversion, a logical shift by 2, and addition. The representation ends at an invalid-instruction boundary.

**Inferred interpretation:** Conventional 32-bit arithmetic yields the address formula and conditional numerical examples above. An arithmetic signed right shift supplies the stated behavior for negative inputs.

**Unknown:** Dependency semantics, memory-map meaning, ownership, reliable continuation, and real runtime outcomes are not established by the supplied material.

No runtime tests have been performed.
