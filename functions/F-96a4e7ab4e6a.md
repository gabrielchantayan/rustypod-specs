<!-- Trusted provenance: F-96a4e7ab4e6a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-96a4e7ab4e6a

## Purpose

F-96a4e7ab4e6a describes a fixed, ordered sequence of indirect dependency invocations, arithmetic transformations of selected dependency results, and four explicit single-byte memory writes.

A higher-level application purpose is not established. In particular, the available evidence does not establish that the routine initializes a device, decodes a format, manages an allocation, or implements a complete application operation.

The supplied representation ends at an invalid-instruction boundary rather than an ordinary return. Its control-flow warning limits confidence in the recovered function boundary and in the apparent completeness of the sequence. This contract therefore distinguishes:

- **Established static evidence:** operations and ordering explicitly represented in the supplied material.
- **Inferred interpretation:** a possible 32-bit machine-word interpretation of arithmetic and addresses.
- **documented_only:** behavior that cannot be determined without additional evidence.

No runtime tests have been performed.

## Interface

### Entry inputs

The displayed interface has no explicit parameters and no declared return value. It nevertheless uses three pieces of ambient entry state:

| Input | Represented width | Use |
|---|---:|---|
| Entry control-address value | Apparently 32 bits | Determines the first indirect invocation target by adding `0xFFE`. |
| Entry fallback word | Apparently 32 bits | Supplies the first store-address calculation when a selected dependency result has bit 30 clear. |
| Entry payload word | Unsigned 32 bits | Supplies the byte values for the first and second explicit stores. |

These are descriptive interface names, not source identifiers. Their actual calling-convention roles, initialization requirements, and availability to an ordinary caller are **documented_only**.

The ambient values are referenced as entry-state values in the supplied representation. Whether that representation correctly captures their lifetime across intervening calls is not independently established.

### Dependency results

Selected invocations provide results interpreted as:

- Signed 32-bit quantities for most scaling operations.
- An unsigned 32-bit quantity for the bit-30 branch.
- Opaque 32-bit quantities for three forwarding operations.

A result whose value is not subsequently used is not evidence that its invocation has no effect.

### Outputs

There is no established ordinary return value or successful return path.

The directly visible outputs are:

1. The ordered dependency-invocation trace.
2. Four attempted one-byte writes at computed addresses.
3. Arrival at the represented invalid-instruction boundary, if all preceding operations complete normally.

### Preconditions

Following the entire represented trace requires:

- Each reached indirect target to be callable under the actual execution convention.
- Every preceding invocation to return normally.
- Each computed store destination to permit a one-byte write.
- The arithmetic and address conversions to be meaningful in the actual execution environment.

The routine does not visibly validate these conditions. These are prerequisites for observing the complete trace, not checks promised by the function.

## Behavior

### Dependency-reference convention

The identifiers **F-D001** through **F-D234** below are opaque, document-local references to consecutive indirect invocation sites. They are not recovered dependency identities. Different references must not be assumed to represent different implementations, and matching behavior must not be assumed merely because sites have similar shapes.

Only F-D001 has a target derived from ambient entry control state. Subsequent targets are represented as fixed code-location-derived addresses with an added displacement of `0xFFE`. Exact executable addresses and target identities remain outside this contract.

An invocation described as having “no explicit arguments” has no argument expressions in the supplied representation. That does not establish that the callee receives no machine-state inputs.

### Repeated scaling behavior

A **five-invocation scaling group** has this observable contract:

- Three dependency invocations occur in order, with no explicit arguments and no direct use of their results.
- A fourth invocation supplies a result.
- A fifth invocation receives one explicit argument: that result shifted left by two bit positions.

Each occurrence uses its own dependency sites. The three leading calls cannot be removed, reordered, or merged on the evidence available.

For results whose signed arithmetic is representable, the explicit argument is four times the producer result. Results involving negative signed operands or signed overflow require the arithmetic qualification below.

A **five-invocation forwarding group** has the same leading three invocations, but its fourth result is forwarded without arithmetic to the fifth invocation. One forwarding group supplies that same result in two argument positions.

### Ordered phases

The following table specifies the complete represented invocation order and the placement of direct memory effects. All phases are conditional on earlier phases completing normally.

| Phase | Invocation sites | Required visible behavior |
|---|---|---|
| Initial scaling | F-D001–F-D010 | Two consecutive five-invocation scaling groups. F-D001 uses the entry-derived target. |
| First address selection | F-D011–F-D015 | F-D011–F-D013 have no explicit arguments. F-D014 supplies the unsigned branch result. The first byte write occurs before F-D015, which receives two identical selected-word arguments. |
| First forwarding | F-D016–F-D020 | One forwarding group; F-D020 receives the opaque 32-bit result from F-D019 in one explicit argument position. |
| Scaling run | F-D021–F-D065 | Nine consecutive five-invocation scaling groups. |
| Payload-bit write | F-D066–F-D070 | Three invocations precede producer F-D069. The second byte write occurs before F-D070, whose two arguments are differently scaled forms of that producer result. |
| Extended lead-in | F-D071–F-D080 | Eight invocations have no explicit arguments and no directly used results. F-D079 supplies a result; F-D080 receives that result shifted left by two. |
| Scaling run | F-D081–F-D120 | Eight consecutive five-invocation scaling groups. |
| First zero write | F-D121–F-D125 | Three invocations precede producer F-D124. The third byte write occurs before F-D125, which receives two copies of the producer result shifted left by fourteen. |
| Second forwarding | F-D126–F-D130 | One forwarding group; F-D130 receives the opaque 32-bit result from F-D129 in one explicit argument position. |
| Scaling run | F-D131–F-D175 | Nine consecutive five-invocation scaling groups. |
| Second zero write | F-D176–F-D180 | Three invocations precede producer F-D179. The fourth byte write occurs before F-D180, which receives two copies of the producer result shifted left by fourteen. |
| Duplicated forwarding | F-D181–F-D185 | One forwarding group; F-D185 receives the opaque 32-bit result from F-D184 in two explicit argument positions. |
| Final scaling run | F-D186–F-D230 | Nine consecutive five-invocation scaling groups. |
| Terminal sequence | F-D231–F-D234 | Four invocations with no explicit arguments and no directly used results, followed by the represented invalid-instruction boundary. |

There is no represented loop, retry, early return, or result-dependent omission of these invocations. The only visible conditional selects the first store address and the arguments to F-D015; it does not select whether that invocation occurs.

### First byte write: conditional address source

F-D014 supplies an unsigned 32-bit result. Bit 30, selected by mask `0x40000000`, determines the selected word:

- **Bit 30 set:** use that result shifted left by two.
- **Bit 30 clear:** use the entry fallback word shifted left by ten.

The function then attempts to write the low eight bits of the entry payload word to an address obtained by doubling the selected word.

Only after that write does F-D015 receive the selected word in both explicit argument positions.

Consequences:

- Bit 31 is not the tested bit.
- The branch result does not itself supply the written byte.
- The store address is twice the value passed to F-D015.
- The function does not read the destination byte before writing it.
- There is no represented null-address exclusion or bounds check.

The unsigned shift used by the bit-set branch discards high bits under the represented 32-bit unsigned interpretation. Subsequent conversion to a signed value and doubling need additional machine-semantics evidence for cases outside the signed range.

### Second byte write: extracted payload bits

F-D069 supplies a signed result. The function attempts a one-byte write at an address formed by shifting that result left by three.

The written byte comprises bits 22 through 29 of the unsigned entry payload word. Bits 30 and 31 do not contribute to that byte.

After the write, F-D070 receives:

1. The producer result multiplied by four.
2. The producer result shifted left by ten.

Both arguments derive from the same producer result. They are not the store address, and neither argument is the extracted payload byte.

### Third and fourth byte writes: zero values

The third write uses F-D124’s result; the fourth uses F-D179’s result.

For each:

- The destination is formed by shifting the corresponding producer result left by three.
- Exactly one byte with value zero is written.
- The immediately following consumer receives the producer result shifted left by fourteen in both explicit argument positions.

These are separate writes at separately computed addresses. The evidence does not establish that their destinations differ from each other or from either earlier destination.

### Arithmetic and address interpretation

The represented arithmetic uses 32-bit integers and one-byte stores. It is not a contract for arbitrary-precision multiplication or shifting.

**Established static evidence:**

- The branch result and payload word are unsigned 32-bit quantities.
- The tested mask selects bit 30.
- The payload extraction uses a logical right shift of 22 positions followed by narrowing to one byte.
- Other scaling expressions predominantly use signed 32-bit intermediates.
- Computed integer quantities become store destinations without visible validation.

**Inferred machine interpretation:**

If execution uses 32-bit word arithmetic with discarded high bits, left shifts and multiplications can be modeled by retaining the low 32 result bits before interpreting an address or argument. This interpretation explains a possible machine trace; it is not independently established for every expression.

**documented_only:**

The supplied material does not establish portable behavior for negative signed left shifts, signed overflow, out-of-range unsigned-to-signed conversion, or arbitrary integer-to-address conversion. Acceptance examples below either avoid these cases or state the additional interpretation explicitly.

No byte order is needed to describe an individual one-byte store. No larger object layout, record format, or buffer extent is established.

### Termination and errors

No explicit error result, status channel, exception translation, or rollback is visible.

If a dependency fails to return, later behavior is not established. If a direct memory write faults, there is no represented local recovery path.

After F-D234, the supplied representation reports an invalid-instruction boundary. This is not evidence of an intentional application-level halt API, a particular exception type, or a reliable normal return.

## Side effects

The four direct byte writes occur in this order:

1. Low payload byte at the conditionally selected destination.
2. Payload bits 22–29 at the F-D069-derived destination.
3. Zero at the F-D124-derived destination.
4. Zero at the F-D179-derived destination.

Each precedes its associated two-argument consumer invocation.

No direct data-memory read is represented for those destinations. In particular, none of the stores is visibly a read-modify-write operation.

Dependency effects are unrestricted by the available evidence. A dependency may read or modify globals, access the same destinations, perform external I/O, or alter state needed by later dependencies. Consequently:

- The four explicit stores do not determine final memory contents.
- No global-preservation guarantee is established.
- No atomicity, synchronization, or thread-safety guarantee is established.
- The absence of an explicit allocation or release does not prove that no allocation or release occurs inside dependencies.

Ownership is **documented_only**. There is no established ownership transfer for any argument, computed address, or result. Arithmetic scaling must not be interpreted as allocation sizing, indexing, or pointer ownership without further evidence.

## Dependencies

| Opaque references | Required relationship | Unknown contract |
|---|---|---|
| F-D001 | First indirect target depends on entry control state plus `0xFFE`. | Calling convention, validity, effects, and return behavior. |
| F-D002–F-D234 | Later indirect targets use represented fixed code-location-derived values plus `0xFFE`. | Actual identities, aliasing between targets, and argument interpretation. |
| Scaling-group producers and consumers | Producer result is transformed and supplied to the next consumer as specified. | Whether the value denotes an address, count, offset, flag, or unrelated scalar. |
| F-D014 and F-D015 | Bit-selected word feeds the first store calculation and duplicated consumer arguments. | Meaning of the branch bit and selected value. |
| F-D069 and F-D070 | One producer supplies a store address and two differently scaled arguments. | Destination ownership and consumer effects. |
| F-D124/F-D125 and F-D179/F-D180 | Each producer supplies a zero-store address and duplicated shifted arguments. | Whether these operations act on related objects. |
| F-D019/F-D020, F-D129/F-D130, F-D184/F-D185 | Opaque 32-bit results are forwarded unchanged, with duplication only for F-D185. | Result type, lifetime, validity, and ownership. |

No dependency may be treated as pure, non-failing, memory-preserving, or memory-owning solely from this representation.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Address examples assume the specified destinations are writable and all preceding dependencies return normally.

### Bit-clear address selection

Suppose F-D014 returns zero, the entry fallback word is 3, and the entry payload word is `0x123456AB`.

Expected direct behavior:

- The selected word is 3,072.
- The first write places byte `0xAB` at address 6,144.
- F-D015 subsequently receives 3,072 in both explicit argument positions.

Using the branch result itself as the destination would not satisfy the represented behavior.

### Bit-set address selection with unsigned truncation

Suppose F-D014 returns `0x40000001`.

The represented unsigned left shift by two yields 4 after discarding high bits. The entry fallback word does not select the result in this branch.

Expected direct behavior:

- The first store destination is address 8.
- Its byte is the low byte of the entry payload word.
- F-D015 receives 4 in both explicit argument positions.

### Extracted byte and distinct scaling factors

Suppose F-D069 returns 5 and the entry payload word is `0x2AC00000`.

Expected direct behavior:

- The second write places `0xAB` at address 40.
- F-D070 then receives 20 and 5,120, in that order.

Forwarding address 40 as either argument would not satisfy this relationship.

### Zero-write ordering

Suppose F-D124 returns 2.

Expected direct behavior:

- A zero byte is written at address 16.
- Only afterward does F-D125 receive 32,768 twice.

If the write cannot complete, invocation of F-D125 is not guaranteed.

### Forwarding without interpretation

Suppose F-D184 returns an opaque 32-bit value.

F-D185 must receive that same value in both explicit argument positions, with no visible arithmetic transformation. This does not establish that the value is a pointer or that either invocation transfers ownership.

### Aliasing destinations

Suppose the third and fourth store calculations produce the same valid address.

Both zero writes remain part of the trace, separated by the intervening dependency invocations. Their equal values do not justify removing either write, especially because dependencies may observe or modify that address.

## Uncertainties

The following remain **documented_only**:

- Actual function boundaries and executable validity, given the control-flow warning.
- Machine architecture, instruction-state rules, and calling convention.
- Dependency identities and their complete inputs, outputs, side effects, and failure behavior.
- Whether ambient entry values are correctly recovered across calls.
- Signed-overflow and address-conversion behavior outside unambiguous arithmetic cases.
- Memory mappings, permissions, ownership, alignment requirements, and device-memory semantics.
- Whether computed destinations alias dependency-managed state or executable memory.
- The externally observable effect of reaching the invalid-instruction boundary.
- Any intended application-level meaning of the repeated invocation patterns.

The repeated fourfold scaling may suggest a unit conversion, but that is inferred intent only. It does not establish an element size or data structure.

## Evidence

This contract is based solely on the supplied static representation.

That representation supports the ordered inventory of 234 indirect invocation sites, selected result-to-argument relationships, one bit-dependent address selection, four explicit byte stores, and a terminal invalid-instruction boundary.

The accompanying control-flow warning is evidence of uncertainty in the representation, not an instruction and not proof of a particular runtime failure.

No dependency implementations, independent instruction decoding, memory map, calling-convention specification, or execution trace were available. No runtime tests have been performed.
