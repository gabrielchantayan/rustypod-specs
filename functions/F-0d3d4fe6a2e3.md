<!-- Trusted provenance: F-0d3d4fe6a2e3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0d3d4fe6a2e3

## Purpose

F-0d3d4fe6a2e3 conditionally copies a 16-bit memory value, evaluates a chain of state-dependent gates, and enters an unconditional dispatch-and-interrupt loop. No normal return is represented.

The supplied static representation does not establish an application-level purpose. In particular, it does not justify describing the operation as a decoder, allocator, initializer, or error handler.

A significant distinction applies:

- **Established static evidence:** the representation contains conditional memory operations followed by two indirect calls and a software interrupt inside an endless loop.
- **Conditional arithmetic conclusion:** under a consistent 32-bit address model, most of the later conditional memory operations cannot be reached. Their presence does not establish that they occur.
- **Unknown:** whether the representation accurately captures the executable’s entry boundary, calling convention, register state, and instruction semantics.

## Interface

### Explicit inputs

The represented interface has five positional inputs:

| Position | Represented width | Observed use |
|---|---:|---|
| First | 32 bits | No explicit use |
| Second | 32-bit unsigned | Controls the initial gate through its low four bits |
| Third | 32 bits | No explicit use |
| Fourth | 32 bits | No explicit use |
| Fifth | 16 bits | Its storage address is used; its incoming value is not explicitly read |

The address of the fifth input’s storage matters independently of its value. Consequently, the behavior cannot be specified solely as a function of the five input values.

### Additional entry state

The representation also consumes incoming machine state not supplied through those five inputs. For this contract, the following descriptive labels identify that state:

- **Source address:** an incoming address used for a 16-bit read.
- **Byte offset:** a 32-bit quantity added to the source address.
- **Retreat amount:** a 32-bit quantity subtracted when forming an internal updated address.
- **Comparison word:** a 32-bit quantity used in equality gates.
- **Auxiliary word:** a 32-bit quantity used in an addition gate.
- **Dispatch base:** an incoming address used to form the first indirect-call target.
- **Parameter-storage address:** the address of the fifth input’s 16-bit storage.

These are semantic labels, not additional declared parameters. No standard calling convention or initialization guarantee for them is established.

### Outputs and completion

There is no explicit returned value. The represented control flow never reaches a normal return.

Observable outcomes can include:

- A conditional 16-bit memory copy.
- An exceptionally constrained additional 16-bit write.
- Calls to computed executable addresses.
- A software interrupt with immediate value `0x5B`.
- Faults, traps, or non-returning behavior caused by memory accesses, calls, or the interrupt.

No success indicator, error code, or recoverable failure path is represented.

## Behavior

### Arithmetic and access model

The representation uses 16-bit unsigned loads and stores, 32-bit integer arithmetic, and address values treated as 32-bit quantities.

The arithmetic conclusions below assume:

- Address calculations use 32-bit wrapping arithmetic.
- An increment of one through a 16-bit-element address advances two bytes.
- Conversion from an address to a 32-bit quantity retains its address bits.
- Explicit byte displacements remain byte displacements.

These assumptions provide a coherent interpretation of the supplied representation. They are not independently verified properties of the executable.

The second input multiplied by `0x10000000`, reduced to 32 bits, is zero exactly when that input’s low four bits are zero. This is the initial behavioral split.

### Initial gate and memory copy

If the second input’s low four bits are nonzero:

- The initial memory copy is skipped.
- The subsequent conditional chain does not become active.
- Execution reaches the dispatch loop with the incoming dispatch base.

If the second input’s low four bits are zero, one memory transfer occurs before the later gates:

1. A 16-bit unsigned value is read from the incoming source address.
2. That value is written as a 16-bit quantity to the source address plus the byte offset.

The read precedes the write. The representation contains no intervening external call.

The implementation also forms an updated source address equal to the original source address plus the byte offset minus the retreat amount. That updated address is used in later comparisons; it is not the destination of another unconditional transfer.

No alignment, accessibility, overlap, or range check is represented. If source and destination coincide, the read and write still occur. If they overlap at a one-byte displacement, the entire represented 16-bit read precedes the represented 16-bit write; byte order remains unspecified.

The incoming fifth-input value does not participate in this transfer.

### Conditional gate chain

After the initial transfer, successive stages are enabled only by a zero result from the preceding enabled stage. Skipping a stage does not allow later stages to reactivate independently.

Although several stages contain variable-shift expressions, their shift operand is zero whenever those stages are enabled: the preceding gate requires the relevant result to be zero. Accordingly, those shifts do not supply a nonzero shift count on the established path.

Under the 32-bit model, reaching the exceptional memory operation requires all of the following:

1. The second input’s low four bits are zero.
2. The incoming dispatch base is zero.
3. The dispatch base plus the parameter-storage address shifted right by one is zero.
4. The byte offset equals the comparison word.
5. The updated source address equals the retreat amount.
6. The auxiliary word is zero.
7. The parameter-storage address plus twice the byte offset is zero modulo \(2^{32}\).

The carry entering the final addition is zero on this path. The source contains explicit carry calculations, but they do not add an alternative route around these conditions.

These requirements are much narrower than the initial copy condition.

### Exceptional memory operation

If all the preceding gates succeed, the representation performs an additional operation involving the dispatch base:

- It obtains a 16-bit unsigned value from the address represented by that base.
- It uses that 16-bit value, zero-extended, as a destination address.
- It stores the value plus `15` there as a 16-bit quantity, truncating modulo \(2^{16}\).

The rendered expression uses the same source location for the destination-address value and the stored-value calculation. It does not reliably establish whether the executable performs one physical load or more than one, or an ordering between those loads. Any necessary source read must precede the dependent store.

The complete gate conditions further imply:

- The parameter-storage address is zero.
- The byte offset is either `0` or `0x80000000`.
- The comparison word equals that byte offset.
- The incoming dispatch base is zero.

The parameter-storage conclusion follows because its logical right shift by one must be zero, allowing only addresses `0` and `1`; the required zero result from adding twice a word excludes the odd address.

Thus the exceptional operation requires a read from address zero. In an environment where address zero is inaccessible, execution faults at that operation rather than completing the store.

The representation also changes an internal dispatch-related address by a displacement derived from the parameter-storage address. Under the conditions needed to reach this point, that displacement is zero.

### Later conditional region

The supplied representation contains additional guarded loads and stores, including:

- A pointer-width read through an address derived from a signed byte conversion.
- A signed 16-bit read followed by two 32-bit stores.
- Another signed 16-bit read followed by two 32-bit stores at addresses formed using fixed displacements.

These operations must not be presented as established reachable effects.

Under the same 32-bit model, the exceptional operation advances an internal 16-bit-element address by one element, making it `2`. The next comparison stage then produces `2` rather than zero. That stops the gate chain before the later guarded operations.

When the exceptional operation is not entered, those later gates remain disabled as well. Consequently, these additional memory operations are unreachable in the coherent arithmetic model described here.

Their executable reachability remains **documented_only** if that model or the supplied representation is inaccurate.

### Dispatch-and-interrupt loop

Execution that reaches the final loop performs the following sequence:

1. Call the executable address at the current dispatch base plus `0xEB6`.
2. If that call returns, call executable address `0x0857A94B`.
3. If that call returns, execute software interrupt `0x5B`.
4. If execution continues after the interrupt, replace the dispatch base with `0x083D4CC2`.
5. Repeat.

The first call receives three represented 32-bit argument values:

- `0`
- `0`
- `0xFFFFFFA5`

The third value has unsigned value `4294967205` and signed two’s-complement interpretation `−91`. Its interpretation by the target is unknown.

The second call has no explicit arguments in the supplied representation. This does not establish that it receives no meaningful register state.

On the first iteration, the first target is the incoming dispatch base plus `0xEB6`. Under the established path analysis, the conditional prelude does not change that effective base.

After a completed interrupt, the first target on subsequent iterations is `0x083D5B78`.

These are direct computed code addresses, not represented reads of function pointers from a table. Neither call has a captured return value.

## Side effects

### Memory

The initial 16-bit copy is controlled only by the second input’s low four bits. It can access addresses derived from unconstrained incoming state.

The exceptional 16-bit store requires the stronger conditions above and can target any address representable by a 16-bit unsigned value. Its stored value is reduced modulo \(2^{16}\).

There is no established buffer layout, allocation size, object type, or ownership transfer. The function performs no explicit allocation or deallocation.

### External state and control

The indirect calls and software interrupt may modify memory, device state, registers, or process state. Their contracts are unavailable. They must not be assumed to preserve globals, preserve arbitrary registers, retain ownership, release memory, or return.

No rollback is represented. If the initial copy succeeds and a later operation faults, the earlier write is not explicitly undone.

### Failure behavior

There is no represented validation for:

- Null or misaligned addresses.
- Unmapped or non-writable destinations.
- Non-executable call targets.
- Unsupported interrupts.
- Arithmetic wraparound.

A fault is not converted into a status result. The observable outcome depends on the execution environment.

## Dependencies

No external dependency has a supplied opaque function ID. The only provided mapping identifies F-0d3d4fe6a2e3 itself.

The function nevertheless depends on three unresolved facilities:

| Dependency | Established interaction | Unknown contract |
|---|---|---|
| First indirect target | Called at dispatch base plus `0xEB6`, with three explicit argument values | Identity, calling convention, effects, return behavior |
| Second indirect target | Called at `0x0857A94B` after the first call returns | Identity, implicit inputs, effects, return behavior |
| Interrupt facility | Invoked with immediate `0x5B` after both calls return | Handler, privilege requirements, effects, resumption behavior |

No opaque IDs are invented for these unresolved dependencies.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests.

### Nonzero low nibble

Given a second input of `1` and a dispatch base of `0x1000`:

- No pre-loop data-memory transfer is performed.
- The first represented call target is `0x1EB6`.
- If that call returns, the next call target is `0x0857A94B`.
- The interrupt follows only if both calls return.

### Copy with nonzero dispatch base

Given a second input of `16`, a nonzero dispatch base, a source containing `0xBEEF`, and a byte offset of `6`:

- `0xBEEF` is read from the source.
- A 16-bit `0xBEEF` is written six bytes beyond the source.
- The exceptional memory operation is skipped because its dispatch-base condition fails.
- Dispatch follows, provided the copy completes.

### Copy to the same address

Given a second input with low four bits zero and a byte offset of zero:

- The source is read before the same address is written.
- This remains an observable read/write pair even though ordinary memory contents are unchanged.
- No claim is made that this is harmless for memory-mapped devices.

### Exceptional-address boundary

If all exceptional-path conditions hold and the 16-bit value read at address zero is `0xFFFF`:

- The exceptional destination is address `0x0000FFFF`.
- The stored 16-bit value is `0x000E`.
- The later guarded 32-bit stores remain unreachable under the stated arithmetic model.

This example requires an environment permitting the relevant accesses; it does not assert that address zero is normally valid.

### Interrupted progression

If the first indirect target never returns, neither the second call nor the interrupt occurs. If the interrupt does not resume execution, the next iteration does not occur.

## Uncertainties

The following remain **documented_only**:

- Actual entry-state provenance and calling convention.
- Executable fidelity of the supplied representation.
- Endianness, alignment behavior, and memory-map constraints.
- Exact physical load count in the exceptional operation.
- Effects and preservation guarantees of both indirect targets.
- Interrupt handling and return behavior.
- Meaning of the represented special machine-state assignment in the exceptional region.
- Reachability conclusions outside the stated 32-bit arithmetic model.

No application-level intent or memory-ownership convention can be established from the available material.

## Evidence

This contract is based exclusively on the supplied static representation.

Directly supported observations include the declared input widths, conditional 16-bit transfer, dependence on incoming machine state, address-of-parameter arithmetic, chained zero-result gates, uncaptured indirect-call results, interrupt immediate, and unconditional loop.

The exceptional-path restrictions and exclusion of later guarded stores are arithmetic deductions under the explicitly stated 32-bit model, not runtime observations.

No runtime tests, executable inspection, dependency inspection, or independent implementation verification have been performed.
