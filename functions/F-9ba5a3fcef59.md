<!-- Trusted provenance: F-9ba5a3fcef59; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9ba5a3fcef59

## Purpose

F-9ba5a3fcef59 specifies a sequence of memory reads and writes driven by four explicit arguments, incoming machine state, fixed global storage, and two dependency addresses. The supplied static representation does not establish an application-level purpose.

The observable behavior includes overlapping storage, low numeric write addresses, dependency-address-derived writes, repeated accesses, unresolved shifts, and two paths ending at unestablished instruction data. It does not establish initialization, allocation, successful object modification, or normal return.

This contract distinguishes:

- **Established static evidence:** represented access widths, calculations, ordering, and branches.
- **Inferred intent:** possible interpretations, not acceptance requirements.
- **documented_only:** behavior that the supplied material does not resolve.

All behavioral sequences below are conditional on execution reaching them and preceding operations completing.

## Interface

### Explicit inputs and result

The represented interface has four 32-bit arguments.

| Argument | Established use |
|---|---|
| First | Address base and input to arithmetic and address calculations |
| Second | No explicit use in the represented body |
| Third | Address base; its low byte is stored at offset 3 from the initially captured word interpreted as an address |
| Fourth | Address base; its complete 32-bit value is stored at first-argument offset 64 |

The third argument is **not** represented as being stored as a complete 32-bit word. The word stored at third-argument offset 116 is the initially captured word, not the third argument.

Using an argument as an address does not establish valid storage, alignment, permissions, allocation, ownership, or lifetime.

The interface supplies no return value. Neither represented endpoint establishes normal completion.

### Implicit inputs

Additional inputs include:

- Three incoming register values used to form the first two read addresses.
- An incoming stack-associated pointer.
- The address of the stack location associated with that pointer.
- Existing memory contents at calculated and fixed addresses.
- The 32-bit address representations of F-e47d7b2e9c88 and F-8afdaab91693.

The stack-location address and the pointer value associated with it are distinct. Their origins and calling-convention requirements are **documented_only**.

### Widths and terminology

Explicit memory operations distinguish 8-bit bytes, 16-bit halfwords, and 32-bit words. Represented pointers and address-bearing values are 32 bits.

Narrowing retains low-order bits. An unsigned byte or halfword used as a numeric address is zero-extended. A signed halfword used in subsequent signed arithmetic is sign-extended.

The following descriptive terms apply:

| Term | Definition |
|---|---|
| Adjusted value | First argument minus 56 |
| Displacement | First argument minus 224, using unsigned 32-bit arithmetic |
| Masked displacement | Displacement with bit 31 cleared |
| Reduced displacement | Masked displacement shifted right by one |
| Halfword base | Twice the displacement, modulo \(2^{32}\) |
| Unresolved destination | Fourth-argument-offset-40 word, shifted left by two and then subjected to a signed right shift by 32 |
| Working base | Signed 32-bit address of F-e47d7b2e9c88, shifted right by 32 |
| Incoming pointer | The stack-associated pointer value |
| Product global | Fixed 32-bit storage at `0x0897EA80` |
| Reset halfword | The explicitly 16-bit global assigned during the final phase and read again later |

The displacement calculation and its multiplication by two use unsigned 32-bit arithmetic. The masked displacement lies between zero and `0x7FFFFFFF`; the reduced displacement lies between zero and `0x3FFFFFFF`.

A right shift by 32 of a 32-bit operand is not assigned a concrete result here. Signed overflow, signed-left-shift behavior, address calculation overflow, and target-specific shift semantics remain unresolved where applicable. Those uncertainties must not be replaced by an assumed shift-count mask or arithmetic result.

## Behavior

### Initial captures and argument-associated accesses

The first two represented accesses are reads:

1. Read a word at the sum of two incoming register values.
2. Read a byte at the sum of the shared register base and the third incoming register value.

These captured values precede every represented write.

The next accesses occur in this order:

1. Store the captured word at fourth-argument offset 8.
2. Store the fourth argument as a word at first-argument offset 64.
3. Store the captured word at third-argument offset 116.
4. Read a word at fourth-argument offset 40.
5. Read a word at third-argument offset 92.
6. Store the third argument’s low byte at captured-word address plus 3.
7. Store the initially captured byte at third-argument offset 16.
8. Store the low byte of the word read at third-argument offset 92 at numeric address equal to the initially captured byte plus 28.

The last destination ranges from 28 through 283. It is not the original byte-source address plus 28.

The two field reads occur after the three initial word stores. They may therefore observe those stores if addresses alias. Conversely, later uses of the initial captures retain the captured values rather than rereading their original sources.

### Displacement-derived accesses

After deriving the displacement-related values and unresolved destination, the represented memory sequence is:

1. Store the reduced displacement’s low byte at the previously read fourth-argument-offset-40 word plus `0x0897E8B8`.
2. Assign `0xB8` to fixed global storage.
3. Store the adjusted value’s low byte at first-argument offset −50.
4. Store the adjusted value as a word at the unresolved destination.
5. Store halfword `0xFFFF` at halfword-base offset 4.
6. Store halfword `0xFFFF` at halfword-base offset 6.
7. Store byte `0xB8` at unresolved-destination offset 4.
8. Store halfword `0xFFFF` at halfword-base offset 2.
9. Assign the low 16 bits of the halfword-base address to another fixed global.
10. Read an unsigned halfword at the halfword base.
11. Zero-extend that halfword to an address and store a word there containing the adjusted value masked with `0x897E8B80`.

The halfword read in step 10 occurs after all preceding writes. It is not an entry-state capture and may observe an earlier write through aliasing.

Where the representation narrows an assigned value but does not declare the global storage type, that narrowing establishes the value’s width, not necessarily the complete width of the global write.

### First dependency address and conditional endpoint

F-e47d7b2e9c88 is used as an address value, not invoked.

Two values are derived from its signed 32-bit address:

- The working base, through the unresolved right shift by 32.
- A halfword obtained by right-shifting the signed address by 31 and narrowing.

The routine first stores this sign-derived halfword at working-base offset 4. It then tests whether the dependency address is negative as a signed 32-bit value.

On the negative path, the representation ends at unestablished instruction data. No specific exception, termination mechanism, recovery action, or normal return is established.

On the nonnegative continuation, accesses occur in this order:

1. Store the sign-derived halfword at working-base offset 18.
2. Store the dependency address as a word at that same dependency address.
3. Read a signed halfword at working base plus reduced displacement.
4. Repeat the halfword write at working-base offset 18.
5. Repeat the dependency-address self-write.
6. Sign-extend the captured halfword, shift it right by 16, and store the resulting word at working-base offset 8.

Under arithmetic signed-right-shift semantics, the final stored word is zero for a nonnegative halfword and `0xFFFFFFFF` for a negative halfword. This conditional interpretation does not resolve either shift by 32.

The repeated writes are distinct observable accesses.

### Selector and low-address writes

The routine next assigns zero to fixed global storage.

A selector is derived from bit 2 of F-e47d7b2e9c88’s address:

- Bit clear: selector 88.
- Bit set: selector 92.

The subsequent accesses are ordered as follows:

1. Store the working base as a word at selector plus 4: numeric address 92 or 96.
2. Assign the masked displacement shifted right by 17, narrowed to 16 bits, to another fixed global.
3. Assign the reduced displacement’s low byte, widened to 16 bits, to an overlapping-global aggregate.
4. Store the complete reduced displacement as a word at four times the selector plus 16: numeric address 368 or 384.
5. Assign the complete reduced displacement to another fixed global.

These low numeric destinations are literal calculated addresses. The representation does not establish that they are valid or writable.

### Incoming-pointer and stack-location-derived accesses

The incoming pointer is then used in this sequence:

1. Store the selector as a halfword at the pointer.
2. Store the working base shifted left by eight as a word at the pointer, overwriting the first word there.
3. Store the same shifted value as a word at pointer offset 20.
4. Overwrite the first word at the pointer with the incoming pointer’s own numeric address.
5. Read a word at twice the incoming pointer’s numeric address.
6. Store that captured word’s low halfword at the incoming pointer.
7. Store the captured word as a word at the address represented by that captured word.

These are separate accesses, not an atomic update. The halfword write in step 6 does not itself replace the complete word written in step 4; byte-level effects depend on the unresolved memory layout.

Next, a word is stored at 32 times the associated stack-location address plus 24. Its value is the incoming pointer interpreted as signed and shifted right by 31. Under arithmetic-right-shift semantics, that value is zero or `0xFFFFFFFF`.

A byte component is then read from overlapping fixed global storage. The routine assigns a concatenation of the 16-bit value 5 and that byte to the corresponding aggregate. Before destination-width conversion, the result has a 24-bit representation and numeric value `0x000500` plus the byte.

Finally, the incoming pointer is stored as a word at four times its numeric address plus 24.

The stack-location address used in the scaled-by-32 destination must not be substituted with the pointer value used in the scaled-by-four destination.

### Product-global and final derived writes

The routine captures a word from fixed storage at `0x0897EA50`. Subsequent calculations described as using this capture do not reread that storage.

F-8afdaab91693’s address is taken as data. No invocation is visible.

The routine assigns four times the incoming pointer’s numeric address to the product global. It then:

1. Uses the product global as both the destination address and stored word value for a self-addressed write.
2. Reads the product global again after that write.
3. Stores the reduced displacement as a word at four bytes beyond the value obtained by that later read.

The first expression references the product global for both destination and value. Their relative evaluation order and exact machine-load multiplicity are **documented_only**. The later read for the offset-4 destination is nevertheless distinct and must not be replaced by the originally assigned product.

The remaining derived accesses occur in order:

1. Store four times the captured `0x0897EA50` value as a word at the address represented by that product.
2. Store four times F-8afdaab91693’s address as a word at a destination formed by shifting the captured value left by six, shifting that signed result right by 17, and adding the captured value.
3. Assign the captured value’s upper 16 bits to the reset halfword.
4. Assign zero to another fixed global.
5. At eight times F-8afdaab91693’s address, store a zero word.
6. Store another zero word four bytes beyond that destination.
7. At offset 16 from that destination, store a byte equal to the derived address’s low byte plus eight, narrowed to eight bits.

Arithmetic and address-validity uncertainties continue to apply.

### Final fixed-global sequence

The final represented assignments occur in this order:

1. Assign `0x897EA700` to fixed global storage.
2. Assign `0x66` to another fixed global.
3. Assign `0xEAEC` to another fixed global.
4. Assign `0x1A` to another fixed global.
5. Assign a global’s own address to its overlapping aggregate storage.
6. Read the reset halfword and assign a combined value with that halfword as the upper 16 bits and `0xEA70` as the lower 16 bits.
7. Assign `0x5C` to another fixed global.

Complete destination widths are not supplied for every assignment.

The reset-halfword read occurs after the self-address global assignment and before the final `0x5C` assignment. Its value is not established to equal the earlier upper-halfword capture: intervening accesses and unresolved storage relationships must not be assumed to preserve it.

The represented path then ends at unestablished instruction data. Normal return is not established.

## Side effects

Writes may target caller-associated storage, fixed globals, low numeric addresses, incoming-pointer-derived addresses, stack-location-derived addresses, dependency addresses, and addresses loaded from memory.

No allocation, release, ownership transfer, lifetime extension, or object-construction protocol is established. Loading, storing, or calculating an address does not establish any such transfer.

No explicit null check, bounds check, alignment check, permission check, rollback, or error result is visible. An invalid operation may prevent later effects. The resulting fault mechanism and partial-execution reporting are **documented_only**.

Repeated accesses and later global reads cannot be collapsed merely because ordinary, non-aliased memory would end with identical contents. No atomicity, thread safety, reentrancy, global preservation, or isolation guarantee is established.

## Dependencies

| Opaque dependency | Established relationship | Unestablished behavior |
|---|---|---|
| F-e47d7b2e9c88 | Address arithmetic, sign test, selector derivation, and writes at its address | Invocation, body execution, ownership, memory preservation |
| F-8afdaab91693 | Address scaling, stored values, and derived write destinations | Invocation, body execution, ownership, memory preservation |

Neither dependency is visibly called. No preservation or memory-management guarantees are assumed from their identities.

The two unestablished instruction endpoints are not established calls to error handlers.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Third and fourth argument distinction

If the third argument is `0x123456AB`, the captured word is address `0x2000`, and execution reaches the relevant byte store, that store writes byte `0xAB` at `0x2003`.

Separately, the word store at first-argument offset 64 uses the complete fourth argument. Substituting the third argument there, or storing the third argument as a word at third-argument offset 116, would not match the representation.

### Captures versus later reads

If an initial store changes the source of the captured word, later uses still use the earlier capture.

If an initial store aliases fourth-argument offset 40, the subsequent read there occurs after that store. Moving the read before the initial stores changes behavior.

### Byte-derived address

If the initial byte is `0xFF`, the later byte destination is numeric address 283, not its original source address plus 28.

### Unsigned displacement

For a first argument of zero:

- Displacement: `0xFFFFFF20`.
- Masked displacement: `0x7FFFFF20`.
- Reduced displacement: `0x3FFFFF90`.
- Halfword base: `0xFFFFFE40`.

These calculations establish neither valid accesses nor the results of signed calculations elsewhere.

### Negative dependency address

If F-e47d7b2e9c88’s represented address has bit 31 set, and execution reaches this phase, the halfword write at working-base offset 4 precedes the conditional endpoint. Testing the sign before that write changes the represented order.

### Aggregate byte combination

If the overlapping byte read yields `0xAB`, concatenating it below the 16-bit value 5 produces `0x0005AB` before destination conversion. It does not produce `0x0500AB`.

### Required global rereads

If the product global’s later read yields a value different from its initial assignment, the reduced-displacement destination is four bytes beyond that later value.

If the reset halfword was earlier assigned `0x1234` but its final read yields `0xABCD`, the final combined value is `0xABCDEA70`, subject to destination representation.

### Completion

Repeated writes on the nonnegative continuation remain separate. Even completion of the final `0x5C` assignment does not establish successful return.

## Uncertainties

The following remain **documented_only**:

- Correct instruction boundaries, function extent, and completeness of represented control flow.
- Calling convention and origins of implicit register and stack inputs.
- Full-width-shift results and relevant signed arithmetic semantics.
- Byte order, alignment requirements, permissions, and valid address ranges.
- Complete widths and overlap relationships of incompletely declared globals.
- Exact load multiplicity and operand evaluation order within the product-global self-write.
- Whether accessed storage is ordinary memory, executable storage, device memory, or another kind.
- Values obtained by later global reads after intervening accesses.
- Runtime behavior at either unestablished endpoint.

Low-level memory or execution-state manipulation is a possible **inferred intent**, not a behavioral requirement.

## Evidence

Established evidence is limited to the supplied static representation: explicit access widths, narrowing and concatenation operations, address expressions, represented ordering, repeated writes, dependency-address uses, and conditional and terminal instruction-data endpoints.

The argument distinction is explicit: the third argument supplies an address base and a narrowed byte; the fourth argument supplies the word stored at first-argument offset 64. The word stored at third-argument offset 116 is the initial word capture.

Global rereads, overlapping storage, and unresolved arithmetic remain explicit rather than being replaced by assumed ordinary-memory behavior.

No runtime tests have been performed. Successful execution, fault behavior, dependency execution, and final memory contents have not been observed.
