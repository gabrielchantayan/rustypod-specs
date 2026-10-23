<!-- Trusted provenance: F-c7a4c2faed1f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c7a4c2faed1f

## Purpose

The supplied static representation describes a routine that attempts an indirect invocation, then a 16-bit memory write to an address derived from its third argument. The recovered path ends at an undecodable-instruction boundary; no ordinary return is established.

No application-level purpose is established. In particular, the evidence does not identify the write as device configuration, buffer modification, metadata maintenance, or any other higher-level operation.

This contract distinguishes behavior expressed by the recovered representation from behavior that requires confirmation against the original instruction stream or execution environment. It does not establish that every displayed operation is a correctly decoded machine instruction.

## Interface

The recovered interface has three arguments and no declared return value:

| Input | Recovered representation | Established use |
|---|---|---|
| First argument | 32-bit quantity, otherwise untyped | No explicit use in the displayed body |
| Second argument | 32-bit quantity, otherwise untyped | No explicit use in the displayed body |
| Third argument | Signed integer, modeled here as 32 bits | Used to derive the destination address of a 16-bit write |

The first two arguments are not explicitly read by the displayed expressions. That does not establish that their incoming machine locations are invisible to the indirect dependency, or that they may safely be omitted by a caller. The calling convention and exact machine-level argument handling are undocumented.

An additional implicit input is an incoming execution-context value associated with a link or return-address location. It is not supplied as one of the three recovered arguments. The indirect invocation target is represented as this value plus `0x7BE`.

The following execution conditions are necessary for the displayed path to progress:

- The computed indirect target must be executable and callable under the actual instruction set and calling convention.
- The indirect dependency must return to the continuation that performs the write.
- The computed destination must support the represented 16-bit store, including any alignment, mapping, privilege, or device-access requirements.
- Execution must reach the relevant instructions despite the explicitly reported decoding problem.

These are conditional execution requirements, not validated preconditions checked by the routine. No checks for them are visible.

There is no established output value. The potentially observable outputs are the indirect dependency’s effects, the memory write if reached and completed, and whatever occurs at the subsequent undecodable boundary.

## Behavior

### 1. Indirect invocation precedes the explicit write

The first represented action is an indirect invocation. Its target is obtained by adding `0x7BE` to the implicit link-context value.

The representation supplies no explicit argument expressions to this invocation. It does not establish a true zero-argument ABI contract: arguments, context, or other inputs may remain available in registers, stack locations, or shared state.

No return value from the dependency is explicitly consumed. This does not establish that the dependency’s return is otherwise irrelevant. Its control flow and side effects determine whether execution reaches the later write.

If the dependency does not return normally, the displayed continuation does not establish that the write occurs. Possible dependency behavior—including faults, nonlocal transfers, indefinite execution, memory changes, and changes to execution context—is undocumented.

The width and wrapping rules for the target-address addition are not established independently of the recovered representation. The offset must not be interpreted as a complete portable pointer-construction rule.

### 2. Destination-address derivation

The destination expression combines a signed right shift, a conversion to unsigned form, a logical right shift, and addition.

Under the recovered 32-bit model:

- Start with the signed third argument.
- Shift it right by 11 positions to obtain a signed intermediate.
- Interpret that intermediate as an unsigned 32-bit value for the second contribution.
- Logically shift the unsigned value right by two positions.
- Add the unsigned representation of the intermediate to that second contribution.

For a conventional arithmetic signed right shift, let \(q\) be the signed result of shifting the third argument right by 11 positions, and let \(u\) be the unsigned 32-bit interpretation of \(q\). The destination address represented by the mixed signed/unsigned expression is then:

\[
A = \left(u + (u \mathbin{\mathrm{logical\ right\ shift}} 2)\right)
\bmod 2^{32}.
\]

The addition is therefore not an unbounded signed addition. Negative values of \(q\) participate through their unsigned 32-bit representations, and the final sum wraps at 32 bits under this model.

The signed right shift requires special qualification. For nonnegative inputs, the displayed calculation has an unambiguous ordinary interpretation. For negative inputs, the recovered expression suggests an arithmetic right shift, but neither the actual instruction semantics nor the accuracy of this recovery has been independently established. Negative-input examples below are conditional on arithmetic right shift and a 32-bit integer model.

There is no visible base-pointer argument or separate base-address addition. The computed integer is used directly as the destination address. It must not be reinterpreted as a checked index into a caller-provided buffer.

### 3. A 16-bit constant is written

After the indirect invocation returns, the representation attempts to write the 16-bit value `0x00CD` to address \(A\).

This is a 16-bit write, not an eight-bit write of `0xCD`. The byte order of the resulting storage is not established by the supplied evidence. It depends on the target environment.

No explicit read of the destination precedes the store. No masking, merging, bounds check, null check, or alignment adjustment is visible.

Some inputs produce odd addresses. The expression does not round them to an even address. Whether a resulting unaligned store succeeds, traps, is split into smaller transactions, or has device-specific behavior is undocumented.

The evidence does not establish atomicity, memory-ordering guarantees, or synchronization with other observers. A displayed 16-bit assignment alone is insufficient to promise a single indivisible externally visible transaction.

### 4. Recovered control flow ends

Following the store, the representation reaches a boundary identified as bad instruction data and truncated control flow.

This supports only the conclusion that the supplied recovery does not describe an ordinary continuation. It does not establish a deliberate processor halt, a particular exception, an operating-system termination, or an actual call to a runtime error handler.

A consumer must not depend on a normal return from this function based on this evidence.

## Side effects

The explicit local side effect is the attempted 16-bit write of `0x00CD`, conditional on reaching and successfully executing it.

The indirect dependency runs first and may have additional effects. Its behavior is unavailable, so this contract does not promise preservation of:

- Global or shared memory.
- Memory that will later serve as the store destination.
- Memory mappings, access permissions, or device state.
- Caller-owned resources.
- Execution-context state outside any independently established ABI guarantees.

No explicit allocation, deallocation, ownership transfer, or resource acquisition is shown in the local continuation. That absence does not constrain the dependency’s ownership behavior. The evidence is insufficient to claim that any input-associated or shared allocation remains owned, allocated, or unchanged across the invocation.

The destination address is not validated as belonging to any allocation or device. A write fault is possible when the environment rejects the address or access width. No local recovery path is shown.

The displayed operation order is invocation, then store, then the undecodable boundary. This is not a promise about the ordering of all underlying bus transactions, speculative activity, or concurrent observations.

## Dependencies

| Opaque ID | Role | Known contract |
|---|---|---|
| F-opaque-01 | Dynamically selected indirect dependency | Target is represented as the incoming link-context value plus `0x7BE`; invocation occurs before the explicit store; no explicit return value is used |

F-opaque-01 is a specification-local opaque identifier. It does not identify a resolved implementation, and the target may depend on the incoming execution context.

Its argument contract, return convention, state preservation, memory effects, ownership effects, failure modes, and termination behavior are all `documented_only`.

The bad-instruction boundary is not treated as a resolved callable dependency. The supplied representation does not establish that an actual function invocation implements that boundary.

## Acceptance examples

These are conditional acceptance examples derived from the static expression. They are not execution results.

1. **Third argument is zero.**  
   Assuming the indirect dependency returns, the address calculation produces zero. The next represented operation is a 16-bit store of `0x00CD` to address zero. Successful completion is conditional on that address being writable; no null-address rejection is visible.

2. **Third argument is `2048`.**  
   The first shift produces one, and the unsigned secondary contribution is zero. The destination is address one. The routine does not visibly correct this odd address. Alignment-dependent outcomes remain environment-specific.

3. **Third argument is `8192`.**  
   The first shift produces four, and the secondary contribution is one. The destination is address five. This example distinguishes the represented calculation from simply using the first shifted result.

4. **Third argument is between zero and `2047`, inclusive.**  
   Each such value produces a first-shift result of zero and therefore destination zero. Distinct arguments within this interval do not produce distinct destination addresses.

5. **Third argument is `-1`, under arithmetic signed right shift.**  
   The first shift produces `-1`, whose unsigned 32-bit representation is `0xFFFFFFFF`. The logical secondary shift produces `0x3FFFFFFF`. The wrapped sum is `0x3FFFFFFE`. This example is conditional on the stated negative-shift interpretation.

6. **The indirect dependency does not return.**  
   The displayed continuation provides no basis for expecting the explicit store or the subsequent undecodable boundary to be reached.

7. **The indirect dependency changes the destination’s mapping.**  
   If the dependency returns, the later store encounters the environment then in effect. The contract does not assume that mappings or permissions remain as they were on entry.

8. **The store completes.**  
   The next represented event is the undecodable boundary, not an established normal return. Acceptance must not require a particular return value or a specific termination mechanism.

## Uncertainties

The following remain `documented_only`:

- The actual instruction set, execution mode, byte order, pointer width, and calling convention.
- Whether the recovered three-argument interface matches the real entry convention.
- The origin and validity of the incoming link-context value.
- The exact machine arithmetic used to construct the indirect target.
- The identity and full behavior of F-opaque-01.
- The machine-level handling of the third argument across that invocation.
- The exact semantics and recovery accuracy of signed right shift for negative inputs.
- The destination’s memory domain, alignment requirements, and accessibility.
- The atomicity and external ordering of the represented store.
- Whether the apparent instruction sequence is correctly decoded executable code.
- The runtime outcome at the reported bad-instruction boundary.

A plausible intent is address transformation followed by a narrow write. That interpretation is inferred, not established application semantics. There is insufficient evidence to assign meaning to the offset, the destination regions, or the stored constant.

## Evidence

The supplied static material contains a single recovered function representation with:

- Two 32-bit untyped parameters and one signed integer parameter.
- An indirect invocation based on an incoming link-context value plus `0x7BE`.
- A destination expression using a signed right shift by 11, an unsigned right shift by two, and addition.
- A 16-bit assignment of `0xCD`.
- Explicit warnings about bad instruction data and truncated control flow.

The arithmetic examples are deductions from that representation under the assumptions stated above. They are not independent confirmation of the original machine instructions.

No runtime tests, instruction-byte validation, dependency inspection, or execution-environment inspection have been performed.
