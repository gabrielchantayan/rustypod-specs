<!-- Trusted provenance: F-10e2de1bbd1a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-10e2de1bbd1a

## Purpose

F-10e2de1bbd1a specifies a condition-dependent operation with two possible memory writes, one coprocessor-store request, three conditional software-interrupt requests, and a terminal bad-instruction boundary.

The supplied static representation establishes this sequence of possible effects, but not its higher-level purpose. It does not establish a valid standalone entry point, a complete calling convention, an allocation or release operation, an ordinary return path, or the meanings of the interrupt selectors.

This contract separates directly represented behavior from a conditional 32-bit arithmetic interpretation and from unresolved environmental behavior. It does not establish that the recovered sequence executes successfully on any particular processor.

## Interface

### Explicit inputs

| Input | Represented width and interpretation | Use |
|---|---|---|
| First argument | Unsigned 32-bit integer | Logically shifted right by ten bits when the incoming zero condition is set. |
| Second argument | Signed 32-bit integer | Subtracted from the shifted first argument; also supplied to the coprocessor-store request. |
| Third argument | Opaque 32-bit value | Written unchanged through an incoming word pointer when the word-store condition is true. |

No argument validation, normalization, or range restriction is shown. The third argument is treated as a four-byte payload, not as an established integer, pointer, or owned resource.

### Additional incoming state

The operation also consumes state that is not supplied by the three explicit arguments:

- A zero-condition indicator.
- A negative-condition indicator.
- An overflow-condition indicator.
- A working quantity represented as a signed 32-bit integer.
- A pointer to four-byte storage.
- An opaque four-byte coprocessor-associated value.

No initialization of this additional state is shown. The three explicit arguments therefore do not determine the behavior by themselves.

The pointer participates both as a write destination and, after conversion to a signed 32-bit integer, as an arithmetic operand. That conversion is explicit in the representation. A complete pointer representation, address-space model, and external calling convention are not established.

The one-byte negative and overflow state carriers are tested through Boolean conversions. Their documented roles are conditions; the supplied representation does not establish the physical encoding of incoming processor flags.

### Outputs and termination

No returned value or ordinary return path is shown.

If execution continues through the selected operations, it reaches a terminal bad-instruction marker. The actual runtime consequence—exception, signal, halt, transfer to a handler, or some other outcome—is unknown.

Earlier memory accesses, the coprocessor request, or an interrupt can prevent later operations from being reached. No local recovery path is represented.

### Preconditions for completing selected operations

The listing contains no explicit checks for writable destinations, alignment, address validity, privilege, or availability of the coprocessor and interrupt mechanisms.

Consequently, acceptance examples that reach later events are conditional on the earlier selected operations completing and permitting continuation. They are not promises that arbitrary input states are safe.

## Behavior

### Conditional arithmetic update

The incoming zero condition is tested first.

If it is clear, the working quantity and the negative and overflow conditions are not replaced by this initial block. Their incoming values remain relevant to subsequent decisions.

If it is set, the representation performs these state updates in order:

1. Replace the overflow condition with the four-byte signed-subtraction-overflow result for the shifted first argument minus the second argument.
2. Replace the working quantity with that subtraction result.
3. Replace the negative condition with the test that the resulting working quantity is less than zero.

The shift is logical because the first argument is unsigned. Its result lies between zero and 4,194,303, inclusive.

The overflow calculation represents signed subtraction overflow, not unsigned borrow. Because the shifted operand is nonnegative, a sufficiently negative second argument can produce a mathematical result above the signed 32-bit maximum.

Under the 32-bit bit-vector interpretation used in the arithmetic examples, the stored subtraction result is the low 32 bits of the mathematical result, and its negative condition is determined by bit 31. Exact execution semantics for overflowing recovered expressions remain unresolved; the source is not a complete architectural specification.

### Overflow-controlled byte write

The current overflow condition controls the next block. It is either the incoming condition or the replacement just calculated.

If this condition is clear, the byte write, coprocessor request, and negative-condition replacement in this block are skipped.

If it is set, the operation first derives a destination by subtracting 2,336, or 0x920, from the working quantity. It then writes one byte to that destination.

The value expression converts the derived address itself to a byte. It does not represent a read of the existing byte at that address.

Under a low-eight-bit conversion model:

- The written byte is the derived address modulo 256.
- Since 0x920 has remainder 0x20 modulo 256, the byte is **the working quantity minus 0x20, modulo 256**.
- It is not generally the low byte of the unadjusted working quantity.

The representation does not independently settle address wraparound, the exact address-to-byte conversion rule, or behavior when signed address arithmetic exceeds its range. The low-byte formula is conditional on the stated conversion model.

There is no explicit null, bounds, accessibility, or alignment check before this write.

### Coprocessor request and negative-condition replacement

After the byte write, the operation issues a coprocessor-store request with:

- Selector 3.
- The incoming coprocessor-associated value.
- The second explicit argument.

The second argument is not validated as an address. The request’s name and operands do not establish its precise memory footprint, access width, privilege requirements, or continuation behavior.

If this request permits execution to continue, the negative condition is replaced by the sign test of the following subtraction:

- Left operand: −1,342,177,279, whose 32-bit representation is 0xB0000001.
- Right operand: the incoming word pointer converted to a signed 32-bit integer.

This sign test does not use the first argument or the earlier subtraction result.

The sign of an overflowing subtraction must not be conflated with the sign of its unbounded mathematical result. Under the 32-bit bit-vector interpretation, the negative condition comes from the wrapped result.

The pointer is used after the coprocessor request in the represented sequence. No independent dependency contract establishes which underlying machine state the request preserves.

### Conditional four-byte write

A separate decision is then formed. It is true only if both conditions hold:

- The current overflow condition is true.
- Subtracting the signed 32-bit pointer value from −1,342,177,279 has signed subtraction overflow.

The second condition uses a distinct overflow calculation. It does not simply reuse the overflow result from the initial arithmetic update.

The conjunction is short-circuiting in the supplied representation: when the first condition is false, the pointer-based overflow calculation is not needed to select the write.

If the decision is true, the third argument is written as four bytes through the incoming word pointer. Otherwise, this write is skipped.

No prior contents of the destination are explicitly read. No allocation, release, ownership transfer, or destination check accompanies the write.

Under the 32-bit signed interpretation, the pointer-based subtraction overflows precisely when the pointer’s signed value is greater than 805,306,369, or 0x30000001. At that boundary itself, the result is the signed minimum and does not overflow.

The byte destination and the word destination are not proven disjoint. If both writes occur and overlap, the later word write can replace bytes affected by the byte write. Exact byte placement depends on endianness and alignment behavior, neither of which is supplied.

### Ordered software-interrupt requests

Following the possible word write, three interrupt sites appear:

| Order | Selection condition | Selector |
|---|---|---|
| First | Negative condition is clear | 0x666663 |
| Second | The four-byte-write decision is true | 0x636564 |
| Third | Negative condition is clear | 0x726564 |

The middle request uses the already-formed decision that selected the word write. It is not preceded by a new pointer-overflow calculation.

The first and third sites refer to the same represented negative-condition value; no intervening assignment to that value is shown. This establishes the represented dataflow, not a general guarantee that interrupt handlers preserve machine flags, registers, globals, or memory.

The requests are ordered and conditional on continuation. If the first selected interrupt does not return to the sequence, later requests are not reached. The same limitation applies to the second selected interrupt.

No interrupt return value, error result, handler behavior, or preservation convention is established.

### Terminal boundary

If execution continues past all selected interrupt sites, it reaches the terminal bad-instruction marker.

The supplied warnings explicitly identify bad instruction data and truncated control flow. This establishes the limit of the available representation. It does not prove that the underlying executable contains no further instructions or that a particular operating-system outcome occurs.

## Side effects

The directly represented effects occur in this possible order:

1. One-byte write at the working quantity minus 0x920.
2. Coprocessor-store request with selector 3.
3. Four-byte write of the third argument through the incoming word pointer.
4. Interrupt request 0x666663.
5. Interrupt request 0x636564.
6. Interrupt request 0x726564.
7. Arrival at the terminal bad-instruction boundary.

Each effect before the terminal boundary is conditional. Earlier faults or non-returning external operations can prevent later effects.

The operation also conditionally replaces its represented working quantity and condition state. Whether those replacements correspond to caller-visible registers or other durable state is not established.

No ordinary memory-data read is explicitly represented. Reading a pointer value for arithmetic does not imply reading its destination. The unknown coprocessor mechanism and interrupt handlers may nevertheless perform reads or additional writes.

No rollback or transactional mechanism is explicitly represented. **Persistence of an earlier write across a later external operation or fault is unknown.** In particular, neither the coprocessor mechanism nor fault handling is established to leave the byte destination unchanged.

Ownership of both potential destinations is unknown. The explicit operations do not acquire, release, retain, or transfer ownership. This does not establish that external mechanisms have no ownership effects.

## Dependencies

The supplied identity mapping contains only F-10e2de1bbd1a. No distinct opaque function IDs are available for the external mechanisms. This contract therefore assigns no invented dependency identities.

The operation relies on:

| Mechanism | Established use | Missing contract |
|---|---|---|
| Four-byte signed-overflow calculation | Determines initial overflow state and the separate word-write decision. | Complete architectural interpretation of the surrounding recovered arithmetic. |
| Coprocessor-store mechanism | Receives selector 3, an incoming four-byte value, and the second argument. | Memory footprint, errors, privileges, state preservation, ownership effects, and continuation. |
| Software-interrupt mechanism | Receives the three listed selectors at ordered conditional sites. | Selector meanings, handler effects, return behavior, and state preservation. |
| Bad-instruction terminal mechanism | Marks the end of represented flow. | Concrete runtime outcome and any continuation outside the recovered representation. |
| Execution environment | Supplies incoming state and resolves memory accesses. | Calling convention, address validity, alignment, endianness, and fault handling. |

None of these unknown contracts may be replaced by an assumption that globals are preserved or memory is acquired, released, or left untouched.

## Acceptance examples

These are conditional static examples, not execution results.

### Arithmetic and optional effects bypassed

Given a clear incoming zero condition, a clear overflow condition, and a set negative condition:

- The initial arithmetic update is skipped.
- Neither explicit memory write is selected.
- The coprocessor request is skipped.
- All three interrupt requests are skipped.
- The represented flow reaches the terminal boundary.

### Two interrupt sites selected from incoming state

Given a clear incoming zero condition, a clear overflow condition, and a clear negative condition:

- Neither memory write nor the coprocessor request is selected.
- The first selected interrupt is 0x666663.
- If execution resumes, 0x726564 is next.
- Interrupt 0x636564 is not selected.

### Initial arithmetic replaces incoming conditions

Given a set incoming zero condition, a first argument of zero, and a second argument of one:

- The subtraction result is −1.
- Signed overflow is clear.
- The negative condition is set.
- Both memory writes, the coprocessor request, and all interrupt requests are skipped.

The incoming negative and overflow conditions do not determine these later selections.

### Correct byte conversion without a word write

Given a clear incoming zero condition, a set overflow condition, and a working quantity of 0x1000:

- The byte destination is 0x6E0.
- Under the low-byte conversion model, the value written is 0xE0, not 0x00.
- The coprocessor request follows the byte write.

If the pointer value used afterward is 0x00001000, the constant-minus-pointer result is −1,342,181,375. It is negative and does not overflow signed 32-bit arithmetic.

Provided earlier operations continue, the negative condition is therefore set, the word-write decision is false, and no interrupt is selected.

### Pointer-overflow boundary

Suppose the overflow-controlled block is selected and preceding operations continue.

For a pointer value of 0x30000001, the constant-minus-pointer result is exactly −2,147,483,648. Under the 32-bit interpretation:

- The negative condition is set.
- The pointer subtraction does not overflow.
- The word write and all interrupts are skipped.

For a pointer value of 0x30000002, the mathematical result is −2,147,483,649. Its low 32 bits are 0x7FFFFFFF:

- The negative condition is clear.
- The pointer subtraction overflows.
- The word write is selected.
- All three interrupt sites are selected in their listed order.

These selections do not establish that either pointer is writable.

### Word write followed by three requests

Suppose the overflow-controlled block is selected and the pointer value used afterward is 0x40000000.

The mathematical subtraction result is −2,415,919,103. Its low 32 bits are 0x70000001. Thus, under the bit-vector interpretation, signed overflow is set and the negative condition is clear.

If both destinations are writable and each external operation permits continuation:

1. The byte write occurs.
2. The coprocessor request occurs.
3. The third argument is written as four bytes.
4. Interrupt requests occur in the order 0x666663, 0x636564, 0x726564.
5. The terminal boundary is reached.

No persistence guarantee across those external operations follows from this example.

## Uncertainties

The following remain **documented_only**:

- Higher-level intent and validity of the recovered entry point.
- The complete calling convention for the additional incoming state.
- Exact architectural handling of overflowing arithmetic and address conversion.
- Address width beyond the represented pointer-to-32-bit conversion.
- Endianness, alignment requirements, and memory-access fault behavior.
- Coprocessor effects, including possible changes to later-consumed state or earlier destinations.
- Interrupt meanings, continuation, state preservation, and ownership effects.
- Persistence of earlier writes across external operations and faults.
- The concrete consequence of the terminal bad-instruction boundary.

A processor-condition interpretation is plausible inferred intent, not an established replacement for these missing contracts.

## Evidence

Established static evidence is limited to the supplied representation: three explicit arguments, additional incoming state, a conditional arithmetic update, two differently sized conditional writes, one coprocessor request, three ordered interrupt sites, and a terminal bad-instruction marker.

The arithmetic constants and conditional examples above are deductions from that representation under the expressly stated width and conversion models. They are not runtime observations.

The supplied bad-instruction and truncation warnings limit claims about completeness and executability. No external mechanism implementation or environmental contract was supplied.

No runtime tests have been performed.
