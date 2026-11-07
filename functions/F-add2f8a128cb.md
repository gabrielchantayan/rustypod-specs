<!-- Trusted provenance: F-add2f8a128cb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-add2f8a128cb

## Purpose

F-add2f8a128cb is represented as a low-level, memory-mutating routine that performs a fixed sequence of stores, interspersed with memory reads and address calculations, before reaching an invalid-instruction termination marker.

The supplied static evidence does not establish a conventional application-level purpose. In particular, it does not support describing this routine as an initializer, allocator, interrupt handler, or reset handler. Although some referenced storage has interrupt- or reset-related labels, those labels alone do not establish hardware behavior.

The behavioral contract is therefore conditional: it describes the accesses and value transformations represented by the supplied material, provided execution reaches each operation and the relevant addresses and arithmetic are meaningful on the target. It does not establish that the entire represented sequence is valid executable behavior.

## Interface

### Explicit inputs

The represented interface has four arguments:

| Argument | Represented width and interpretation | Observed use |
|---|---|---|
| First | Pointer to 32-bit storage, with a represented 32-bit address | Store destination, stored pointer value, and operand in several address calculations |
| Second | Unsigned 32-bit integer | Bitwise mask operand used to construct one store address |
| Third | Opaque 32-bit value | No represented use |
| Fourth | Signed 32-bit integer | Operand in two store-address calculations and the value narrowed for one store |

The first argument is not merely a pointer to a single object. Its numeric address participates in accesses far outside the initially addressed storage. A valid four-byte object at that address is insufficient to satisfy the routine’s access requirements.

The fourth argument likewise is not used solely as an ordinary integer: it helps determine two memory destinations.

### Additional entry state

The body consumes three values not supplied through the four explicit arguments:

- One 16-bit inherited value, used in the first store.
- One 32-bit inherited value, used in two subsequent stores.
- A second 32-bit inherited value, used later in a store adjacent to a computed address.

Their entry-state origin and calling-convention requirements are not established. They must not be treated as initialized constants or synthesized from the explicit arguments.

### Outputs and termination

No return value is represented. The visible sequence ends at an invalid-instruction termination marker rather than a normal return.

The marker does not establish a specific processor exception, process exit status, recovery path, or externally callable error API. Earlier memory-access failures may prevent execution from reaching it.

### Notation and width model

The following notation describes values rather than implementation identifiers:

- **P**: numeric address of the first argument.
- **M**: second argument.
- **N**: fourth argument.
- **H**: inherited 16-bit value.
- **W** and **Z**: the two inherited 32-bit values, in order of their first use.
- **L**: value read from storage at address `0x0897EA50`.
- **S₁** and **S₂**: the first and second numeric uses of the symbol associated with dependency F-8afdaab91693.

The representation uses 32-bit integers, 32-bit pointer values, 16-bit halfwords, and 8-bit bytes. Arithmetic involving signed values has important unresolved semantics, detailed below. Mathematical expressions in this document do not independently authorize wrapping arithmetic.

## Behavior

### Ordered memory operations

The following order is established by the supplied representation. A later operation occurs only if execution continues through all preceding operations. No atomic grouping, rollback, or transactional behavior is shown.

1. **Write inherited values through the first argument.**
   - Store H as 16 bits at P.
   - Store W as 32 bits at P.
   - Store W as 32 bits at P + 20 bytes.

   The second store overlaps the entirety of the first store’s destination. That does not make the first store unobservable: memory-mapped devices, faults, or external observation could distinguish the two writes.

2. **Write the first argument’s pointer value through a masked address.**
   - Construct the destination as P + (M bitwise-AND P).
   - Store the 32-bit pointer value P at that destination.

   The mask operates on the pointer’s numeric representation, not on the contents at P.

3. **Read a word through a doubled address and use it twice.**
   - Read a 32-bit value from address 2P.
   - Store that value’s low 16 bits at P.
   - Treat the full loaded value as an address and store the same 32-bit value at that address.

   The last operation is a self-address store: the loaded numeric value determines both the destination and the stored word. The loaded value is retained for these uses; the representation does not show a second read from 2P.

4. **Perform two accesses derived from the fourth argument.**
   - Store a 32-bit sign-derived value at address 32N + `0x18`. The value is the signed 32-bit interpretation of P shifted right by 31 positions.
   - Store the low 16 bits of N at address N + 4.

   Under an arithmetic-right-shift interpretation, the first stored value is zero when P’s most significant bit is clear and all ones when it is set. That interpretation is conditional, not an independently established target guarantee.

5. **Perform an absolute-location store and another pointer store.**
   - Assign numeric value 5 to storage labeled at address `0x00000005`.
   - Store pointer value P as 32 bits at address 4P + `0x18`.

   The width of the first assignment is not supplied. Its exact affected byte range is therefore unknown.

6. **Capture one memory value and two dependency-symbol values.**
   - Read L from storage at `0x0897EA50`.
   - Obtain S₁ from the symbol associated with F-8afdaab91693.
   - Assign 4P to storage at `0x0897EA80`.
   - Form a value equal to four times a second numeric use, S₂, of that dependency symbol.

   These are uses of a symbol as a numeric value. The visible body does not invoke F-8afdaab91693.

7. **Perform stores through the address held at `0x0897EA80`.**
   - Using the value held in that storage as an address, write that value as a 32-bit word to its own address.
   - Using the address held in that storage for the next operation, store Z as 32 bits four bytes beyond it.

   The supplied expression accesses this global storage rather than exclusively using a retained calculation of 4P. Consequently, it is unsafe to assume that the second destination is always 4P + 4. The first indirect write may alias and modify the global storage itself.

   No ordering between multiple reads within a single assignment expression is established here beyond what is needed to describe its operands.

8. **Perform two stores derived from L.**
   - At address 4L, store the 32-bit value 4L.
   - At address L plus the result of shifting L left by six positions and then shifting that result right by seventeen positions, store the value derived from 4S₂.

   The right shift in the second address expression is represented on a signed integer. It must not be silently replaced with a logical shift, division, or an unsigned bit-field extraction.

9. **Update reset- and interrupt-labeled storage.**
   - Convert L to an unsigned 32-bit value, shift right by 16 positions, narrow to 16 bits, and assign the result to a reset-associated upper-half storage location.
   - Assign zero to interrupt-associated storage.

   The names do not prove that either store changes actual processor interrupt or reset state. The assignment width of the interrupt-associated storage is not supplied.

10. **Clear two words and write a later byte through an S₁-derived base.**
    - Form a retained base address B = 8S₁.
    - Store a 32-bit zero at B.
    - Store a 32-bit zero at B + 4.
    - Store one byte at B + 16. Its represented value is the byte-sized result of converting B to a character-sized value and adding 8.

    The byte destination is sixteen bytes beyond B, not four bytes beyond it. No clearing of the intervening bytes at B + 8 through B + 15 is represented.

11. **Perform the final fixed-location assignments in order.**
    - Assign `0x897EA700` to storage labeled at `0x00000086`.
    - Assign `0x66` to storage labeled at `0xEA700000`.
    - Assign `0xEAEC` to storage labeled at `0x0897EB26`.
    - Assign `0x1A` to storage labeled at `0x12FD4C34`.
    - Assign the address value `0x897EA65C` to storage labeled at that same address.
    - Read the reset-associated upper-half value at this point, concatenate it above the 16-bit constant `0xEA70`, and assign the resulting value to reset-associated storage.
    - Assign `0x5C` to storage labeled at `0x12FD4CB8`.

    Declaration widths for these destination symbols are absent. The concatenation constructs a 32-bit value, but the supplied material does not independently establish the destination’s declared width.

12. **Reach the invalid-instruction termination marker.**

### Aliasing and intermediate state

Every read observes memory at its position in this sequence, subject to the target’s actual access semantics. It must not automatically observe entry-state memory.

For example:

- An early store may change the word later read at 2P.
- A computed store may change storage at `0x0897EA50` before L is captured.
- A write through the address stored at `0x0897EA80` may modify that address storage.
- A later fixed-location assignment may overlap reset-associated storage before the final reset value is assembled.

The representation explicitly warns of overlapping global symbols. Distinct labels are not proof of distinct byte ranges.

## Side effects

The routine writes through caller-derived addresses, memory-loaded addresses, a dependency-symbol-derived address, and fixed storage locations. There is no bounds check, null check, range validation, or access-permission check in the visible sequence.

No allocation, release, ownership transfer, or reference-count operation is represented. Reading a value as an address does not establish ownership of the addressed storage. Callers must not infer that the routine acquires or releases memory.

Successful execution requires every reached access to have whatever mapping, permissions, alignment, and device-access properties the target requires. These requirements extend beyond the first argument’s nominal pointee.

No error result, cleanup path, restoration of earlier writes, or recovery behavior is shown. The invalid-instruction marker does not undo preceding effects.

## Dependencies

### F-8afdaab91693

This dependency is referenced numerically:

- Its first represented numeric value contributes to base address B.
- Its second represented numeric value contributes to a stored word.

No invocation is visible. There is no basis for attributing callee side effects, memory ownership, global preservation, or return behavior to these references.

The precise numeric meaning of the symbol—such as a callable address or another target-specific representation—is not established. Equality of S₁ and S₂ requires that both uses denote the same stable numeric value.

### Execution environment

The routine also depends on:

- The three inherited entry values.
- The address space and storage represented by the absolute locations.
- The meanings and overlap of reset- and interrupt-associated storage.
- Target integer, pointer, character-conversion, and shift semantics.
- The environment’s response to invalid instruction data.

These are prerequisites or unknowns, not additional identified function dependencies.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Zero mask.**  
   If M is zero, the masked-address store targets P and writes pointer value P. It follows the two W stores and precedes the read at 2P.

2. **Low-half replacement.**  
   If the word read at 2P is `0x12345678`, the next halfword store at P carries bit pattern `0x5678`. The following word store targets address `0x12345678` and writes `0x12345678`, provided that address is accessible.

3. **Captured memory value.**  
   If L is captured as `0x12345678`, its unsigned upper-half extraction produces `0x1234`. Later changes to storage at `0x0897EA50` do not change the retained L.

4. **Final reset composition without intervening aliasing.**  
   If the reset-associated upper-half location still contains `0x1234` when the final composition reads it, the constructed 32-bit value is `0x1234EA70`. If intervening writes changed that location, the then-current value must be used instead.

5. **Word-versus-byte destinations.**  
   If B is `0x00001000` and the byte conversion preserves low-eight-bit patterns, the routine writes zero words at `0x1000` and `0x1004`, then byte `0x08` at `0x1010`. This does not establish zeros at `0x1008` or `0x100C`.

6. **Unused third argument.**  
   Changing only the third explicit argument introduces no represented change to the body’s calculations or accesses. This does not resolve missing calling-convention information.

7. **Inaccessible intermediate address.**  
   If execution cannot complete the store through the value loaded from 2P, the contract does not promise any subsequent store or the final termination marker. Earlier writes are not represented as rolled back.

## Uncertainties

The following remain **documented_only**:

- Whether the represented instruction stream is valid. The supplied material reports bad instruction data and truncated control flow.
- Exact declarations, widths, and overlaps of fixed-location globals.
- Target endianness and alignment requirements.
- Whether absolute locations are ordinary memory, device registers, protected storage, or unmapped.
- Signed overflow behavior for address-related additions, multiplication, and left shifts.
- Right-shift behavior for negative signed values.
- Signedness and out-of-range conversion behavior of the character-sized value used in the byte store.
- Exact pointer-to-integer and integer-to-pointer semantics.
- The source and ABI significance of inherited values.
- The precise numeric interpretation of the dependency symbol.
- The architectural effect of the termination marker.

In particular, the signed calculations must not be unconditionally specified as modulo-$2^{32}$ machine operations. That may be a plausible target interpretation, but it is not established by the supplied representation.

## Evidence

**Established static evidence:** four explicit arguments; three inherited values; the represented sequence of memory accesses and arithmetic; numeric references to F-8afdaab91693 without a visible call; and termination at an invalid-instruction marker.

**Inferred intent:** none is required for this contract. Symbol labels may suggest low-level machine-state manipulation, but that interpretation is not an acceptance requirement.

**Evidence limits:** the supplied material warns both of bad instruction data and overlapping global symbols. Missing declarations and target semantics prevent a complete byte-level or executable-validity guarantee.

No runtime tests have been performed. The acceptance examples express consequences of the static representation under stated conditions, not observed execution results.
