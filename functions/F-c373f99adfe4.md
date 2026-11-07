<!-- Trusted provenance: F-c373f99adfe4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c373f99adfe4

## Purpose

F-c373f99adfe4 performs an ordered sequence of memory accesses and indirect dependency calls, then selects between an exceptional path and a three-word writeback path using a captured signed 32-bit record field.

The exceptional path obtains additional memory values, invokes a third dependency, and requests software interrupt number 72 if that dependency returns and execution proceeds. The writeback path stores three consecutive 32-bit words at a computed address.

The supplied static representation truncates both paths at invalid-instruction boundaries. It does not establish a normal return, a conventional return value, or intentional non-returning termination.

A record-processing, dispatch, or diagnostic role is **inferred intent**, not an established application-level purpose.

## Interface

### Explicit inputs

| Input | Established interpretation | Visible use |
|---|---|---|
| First argument | Address of a record containing 32-bit fields | Reads at byte offsets 0, 4, 8, 16, and 20 |
| Second argument | 32-bit value | No direct use is visible |
| Third argument | 32-bit address-valued input | Destination base for a 16-bit store at byte offset 8 |

The complete record type, allocation extent, ownership, and required alignment are unknown. No access to the record field at offset 12 is visible.

The lack of a visible use for the second argument does not establish that it can be omitted under the unresolved calling convention.

### Additional incoming context

The body also consumes incoming machine context not represented by the three explicit arguments:

- A base address used for a 32-bit read at byte offset 44.
- A 16-bit value stored through the third argument.
- A dispatch anchor used to select the first dependency.

The first dependency target is the incoming dispatch anchor plus `0x8D6`, using the visible 32-bit address model. No alignment mask is applied to this target expression.

The origins and setup requirements of these context values are unknown. The three explicit arguments alone are not established as a sufficient invocation interface.

### Post-call writeback input

The first writeback word comes from a 32-bit register value exposed after the second dependency call.

Its producer and calling-convention meaning are unresolved. It is not established to be an entry value preserved across calls, a half of the first dependency’s result, the second dependency’s argument, or a conventional return value.

The established requirement is narrower: the first writeback store consumes that post-call register value.

### Widths and arithmetic

The visible operations establish:

- 32-bit record reads, a 32-bit context read, and 32-bit writeback stores.
- One 16-bit early store.
- One signed 16-bit read, sign-extended to 32 bits.
- One eight-bit read on the exceptional path.
- A first-dependency result consumed as two 32-bit halves.

The values used for exceptional arguments 2 and 4 come from identified fixed locations, but their storage widths, read widths, signedness, and argument conversions are not established.

Address calculations and integer arithmetic follow the visible 32-bit model. In particular, multiplication by 1,024 and addition of 161 retain their low 32 bits.

The branch field is interpreted as signed. The first dependency’s low result half is interpreted as signed before an arithmetic right shift by 30. The possible shifted values are −2, −1, 0, and 1.

### Outputs and preconditions

No conventional return or error-reporting interface is established.

Observable outputs are the early 16-bit store, dependency invocations and their effects, conditional writeback stores, and the conditional interrupt request. Every operation after a dependency call is conditional on that call returning control.

Each reached memory access requires memory suitable for its actual width and operation. Each indirect call requires a callable target compatible with the actual machine-level invocation. The representation supplies no validation establishing these conditions.

## Behavior

### Initial context read and early store

The first visible memory operation reads a 32-bit word from the incoming context base plus 44. This captured word is later used as exceptional argument 6.

The function next writes the incoming 16-bit context value to the third argument plus 8.

This write precedes all visible record reads. If it overlaps a record field that is subsequently read, the later read observes memory after this store, subject to the execution environment’s memory semantics. There is no visible alias check or earlier record snapshot.

### Record capture and first dependency

The function captures these 32-bit record fields in order:

1. Offset 8: the destination-forming value.
2. Offset 16: the second writeback word.
3. Offset 20: the signed branch input and arithmetic writeback input.

The record fields at offsets 0 and 4 then supply the first dependency’s two arguments, in that argument order. Their relative read order is not established by the supplied call expression. Both reads follow the early store and the three captures above.

F-000000000001 is selected at:

**incoming dispatch anchor + `0x8D6`**

The dependency is invoked with the fields at offsets 0 and 4. Its result is consumed as two 32-bit halves:

- The high half supplies an address base.
- The low half supplies the signed-shift input for the next dependency.

After the dependency returns, the function reads a signed 16-bit value from the high-half address plus 180 and sign-extends it to 32 bits. The resulting value is between −32,768 and 32,767.

No address-validity check precedes that read.

### Destination formation and second dependency

The destination address is the low 32 bits of the captured record field at offset 8 multiplied by 1,024. No allocation base is visibly added. The field is therefore an address-forming value, not an established index into a known allocation.

F-000000000002 is selected by clearing the low two bits of `0x08A01EBF` and adding `0x2FC`:

**`0x08A01EBC` + `0x2FC` = `0x08A021B8`**

It receives one visible argument: the arithmetic right shift by 30 of the signed low half of the first dependency’s result.

After this call, the unresolved post-call register value supplies the first writeback word if that path is selected. The static evidence does not establish a conventional result signature for this dependency.

The later branch and writeback use the previously captured record fields. Offsets 8, 16, and 20 are not visibly reread after either dependency. This does not imply that either dependency preserves record memory, globals, or other machine state.

### Branch selection

After the second dependency returns, the function adds 161 to the captured field from record offset 20, retaining the low 32 bits.

The exceptional path requires both:

- The 32-bit sum is nonzero.
- The captured field, interpreted as signed, is greater than −162.

These conditions select exactly the signed range **−160 through 2,147,483,647**.

The writeback path covers **−2,147,483,648 through −161**:

- −162 fails the strict signed comparison.
- −161 produces a zero sum.
- −160 satisfies both conditions.

At the upper signed boundary, the sum wraps without becoming zero. No overflow error or recovery path is visible.

Neither branch bypasses the earlier calls or the dependent signed 16-bit read.

### Exceptional path

The function invokes F-000000000003 with seven visible arguments:

| Position | Established argument |
|---|---|
| 1 | The byte read from destination address plus 31 |
| 2 | The value at fixed location `0x08A02250` |
| 3 | The address `0x08A01FAC` |
| 4 | The value at fixed location `0x08A0223C` |
| 5 | The sign-extended dependent 16-bit value |
| 6 | The 32-bit context word captured before the early store |
| 7 | The fixed 32-bit token `0x08A01EBF` |

The three fixed locations have distinct roles:

- `0x08A02250` supplies a value.
- `0x08A01FAC` is passed as an address, without a visible dereference at that location.
- `0x08A0223C` supplies a value.

The declarations and explicit load-width conversions for arguments 2 and 4 are absent. Their source-memory access widths, signedness, parameter types, passing widths, and conversions remain unresolved. Neither the address model nor the argument positions establish four-byte reads.

The destination read for argument 1 has an established width of eight bits. Its argument-passing width and any extension rule are not established.

F-000000000003 is selected by clearing the low two bits of `0x08A01EC9` and adding `0x6F8`:

**`0x08A01EC8` + `0x6F8` = `0x08A025C0`**

The relative evaluation order of the destination-byte read and the two fixed-location value expressions is not established. These evaluations occur on the exceptional path after the second dependency. They must not be treated as values captured before that call.

If the exceptional dependency returns and execution proceeds, the function requests software interrupt number 72, equivalently `0x48`. The representation then reaches an invalid-instruction boundary.

The local three-word writeback does not occur on this path. Dependencies and interrupt handling may nevertheless write memory; their effects are unknown.

### Writeback path

The function performs three separate 32-bit stores in this order:

| Destination offset | Stored value |
|---|---|
| 0 | The unresolved post-second-call register value |
| 4 | The field captured from record offset 16 |
| 8 | The low 32 bits of the field captured from record offset 20 plus 161 |

The first word must not be substituted with an assumed entry value or an inferred conventional return value.

The stores are sequential, not an established atomic transaction. No rollback is visible if a later store fails after an earlier one completes.

After the third store, the representation reaches another invalid-instruction boundary. A normal return is not established.

## Side effects

The established major-operation order is:

1. Read the context word.
2. Perform the early 16-bit store.
3. Capture record fields at offsets 8, 16, and 20.
4. Obtain the first dependency’s two record arguments and invoke it.
5. Perform the dependent signed 16-bit read.
6. Invoke the second dependency.
7. Select the exceptional call-and-interrupt path or the ordered writeback path.

Consequences of this order include:

- Early-store aliasing can affect the subsequently captured record fields.
- Dependency mutations of record memory do not cause visible rereads of the three captured fields.
- The exceptional destination byte and fixed-location values can reflect changes made by earlier dependencies.
- Earlier writes and dependency effects are not visibly undone after a later fault or failure.

No local allocation, release, reference-count adjustment, lock operation, or ownership transfer is visible. Passing addresses to dependencies does not establish whether those dependencies borrow, retain, modify, allocate, or release memory.

There are no visible null, capacity, range, or alignment checks. Invalid-access behavior is unknown. Byte order, permitted unaligned accesses, concurrency semantics, and atomicity guarantees are not established.

The two fixed-location value accesses require suitable memory, but their necessary extent cannot be specified because their access widths are unknown.

## Dependencies

The opaque IDs below identify dependency roles, not established semantic identities.

| Opaque ID | Target selection | Visible interface |
|---|---|---|
| F-000000000001 | Incoming dispatch anchor plus `0x8D6` | Two 32-bit record-field arguments; result consumed as two 32-bit halves |
| F-000000000002 | (`0x08A01EBF` with its low two bits cleared) plus `0x2FC`, resolving to `0x08A021B8` | One signed argument from −2 through 1; followed by consumption of an unresolved register value on writeback |
| F-000000000003 | (`0x08A01EC9` with its low two bits cleared) plus `0x6F8`, resolving to `0x08A025C0` | Seven argument positions; followed by interrupt 72 if execution returns and proceeds |

The target calculations are established call-selection behavior. The targets’ semantic identities and calling conventions remain unknown.

For F-000000000002, the post-call dataflow neither proves a conventional return signature nor establishes that the dependency itself produces the first writeback word. Ignoring that consumed register value would also omit established behavior.

For F-000000000003, the identified argument locations do not resolve the unknown widths or conversions of the two fixed-location values.

No dependency has an established contract for memory preservation, register preservation, ownership, allocation, release, reentrancy, exceptional behavior, or termination.

Software interrupt 72 is a separate environmental dependency. Its handler, effects, and continuation behavior are unknown.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Each assumes all earlier required accesses succeed and dependencies return wherever continuation is necessary.

### Dependency selection

If the incoming dispatch anchor is `0x10000000`, F-000000000001 is selected at `0x100008D6`.

The second dependency target is `0x08A021B8`, and the exceptional dependency target is `0x08A025C0`. Selecting different targets merely because they appear semantically equivalent would not preserve the established call-selection behavior.

### Early-store aliasing

If the third argument plus 8 overlaps record offset 20, the branch input is captured after the 16-bit store. Capturing the entire record before the store would change the established ordering.

The resulting numeric field depends on byte order and the exact overlap.

### Signed dependent read

If the dependent 16-bit read yields bit pattern `0xFFFF`, its extended value is −1. If the exceptional call is reached, argument 5 receives −1, not 65,535.

### Signed-shift argument

| First result’s low-half bit pattern | Argument to F-000000000002 |
|---|---:|
| `0x00000000` | 0 |
| `0x40000000` | 1 |
| `0x80000000` | −2 |
| `0xFFFFFFFF` | −1 |

Unsigned extraction does not satisfy these cases.

### Writeback boundaries

For a captured branch field of −161, destination offset 8 receives zero.

For −162, destination offset 8 receives `0xFFFFFFFF`.

In both cases, destination offset 4 receives the captured record field from offset 16, and destination offset 0 receives the unresolved post-call register value. Both earlier dependencies and the signed dependent read still occur.

If the consumed post-call register value differs from an entry register value, storing the entry value would violate the established dataflow.

### Exceptional boundary and fixed locations

For a captured branch field of −160, the exceptional path is selected. The function obtains the destination byte at offset 31 and supplies:

- The value at `0x08A02250` as argument 2.
- Address `0x08A01FAC` as argument 3.
- The value at `0x08A0223C` as argument 4.

Passing either value location’s address instead of its value, or dereferencing `0x08A01FAC` to construct argument 3, would not preserve the established distinctions.

A requirement for exactly four-byte reads at the two value locations is unsupported. Even establishing a 32-bit argument slot would not determine the source-memory read width. Those widths and conversions remain **documented_only**.

If F-000000000003 returns and execution proceeds, interrupt 72 is requested. The local three-word writeback is absent.

### Destination wraparound

A destination-forming field of `0x00400000` produces address zero after multiplication by 1,024 at 32-bit width. No visible check rejects that result. Subsequent behavior depends on the validity of the reached accesses.

### Upper signed boundary

For a captured branch field of 2,147,483,647, the exceptional path is selected. The wrapped sum is `0x800000A0`, which is nonzero.

## Uncertainties

The following remain **documented_only**:

- Completeness and accuracy of the supplied control-flow representation.
- Execution beyond either invalid-instruction boundary.
- Whether those boundaries represent actual traps, omitted continuation, or incorrectly interpreted instructions.
- Origins and setup requirements of the additional incoming context.
- Producer and calling-convention meaning of the register value used for the first writeback word.
- Actual dependency identities, calling conventions, and memory, register, ownership, and control-flow effects.
- Storage widths, access widths, signedness, and conversions for values at `0x08A02250` and `0x08A0223C`.
- Argument-passing widths and conversions not explicitly established by the representation.
- Contents, extent, and semantic type of the object at `0x08A01FAC`.
- Byte order, alignment constraints, concurrency behavior, and memory-fault handling.
- Interrupt handling and continuation.
- Any public return or error-reporting contract.

The known target addresses and fixed data locations are not unresolved. Their semantic meanings and the missing interface details are.

A diagnostic or validation role for the exceptional dependency is plausible **inferred intent**. The evidence does not establish an assertion handler, logger, allocator, destructor, or exception raiser.

## Evidence

This contract is based only on the supplied static representation.

**Established static evidence** includes the explicitly sized accesses, record offsets, early-store ordering, captured fields, dependency-target expressions, argument construction, signed halfword extension, signed right shift, branch conditions, ordered writeback stores, post-second-call register consumption, and interrupt number 72.

The exceptional argument expressions distinguish values at `0x08A02250` and `0x08A0223C` from address `0x08A01FAC`. They do not establish the two value accesses’ widths or conversions.

**Arithmetic deductions** include the resolved targets `0x08A021B8` and `0x08A025C0`, the signed branch ranges, the shift-result range, and the wraparound examples. These are not runtime measurements.

**Inferred intent** is limited to possible record-processing, dispatch, or diagnostic roles. None is required by the contract.

The representation reports invalid instruction data and truncates both visible endpoints. Behavior beyond them remains unresolved.

No runtime tests have been performed.
