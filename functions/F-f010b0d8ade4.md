<!-- Trusted provenance: F-f010b0d8ade4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f010b0d8ade4

## Purpose

The supplied static representation describes a routine that makes two indirect calls, performs address-derived memory accesses, updates global storage, and ends at a bad-instruction termination marker. It does not establish an application-level purpose or successful outcome.

The representation includes warnings about invalid instruction data and overlapping global symbols. Its operations must not be recast as a valid initialization, reset, serialization, or device-control protocol without further evidence.

This contract describes the represented sequence, conditional on execution reaching each operation and the representation identifying it correctly. It distinguishes explicitly typed memory operations from accesses to globals whose declarations are absent. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has four arguments and no expressed return value.

| Input | Established representation | Observed role |
|---|---|---|
| First argument | Signed 32-bit integer | Supplies the initial destination and product, and a shifted value written through the fourth argument. |
| Second argument | Uninterpreted 32-bit value | No explicit use appears in the body. |
| Third argument | Unsigned 32-bit integer | Used numerically and as an address; its logical right shift supplies another address. |
| Fourth argument | Pointer to a signed 32-bit integer | Receives a 32-bit write; its numeric address participates in calculations; the pointer is stored elsewhere. |

Pointer-to-integer conversions and pointer stores follow the represented pointer model. They do not independently establish the target ABI or physical pointer width.

The absence of an explicit use of the second argument does not establish its irrelevance to the original machine execution or incomplete indirect-call interfaces.

### Implicit inputs

The routine also uses:

- An ambient base-register value, represented as signed 32-bit.
- An ambient return-link value, represented as signed 32-bit.
- An incoming pointer associated with stack displacement `0x138`.
- The address of that incoming stack slot, distinct from its pointer contents.
- A **late-source global** associated with `0x0897EA50`.
- A **pointer-product global** associated with `0x0897EA80`.
- A **reset-fragment global**, used first as an assignment destination and later as an operand source.
- Numeric addresses associated with dependencies.
- State potentially changed by the indirect calls.

An incoming 16-bit stack item at displacement `0x13C` is declared but not explicitly used.

The declarations of the three descriptive globals above are absent. Their storage widths, signedness, access widths, and conversion behavior are therefore unresolved. In particular:

- Reading the late-source global into a signed 32-bit local establishes the local’s width and interpretation, not the global’s.
- Assigning a signed 32-bit product to the pointer-product global does not establish that global’s width or that it retains the complete product.
- Assigning a 16-bit expression to the reset-fragment global does not establish a 16-bit storage access.
- Using that global as a 16-bit concatenation operand establishes an expression requirement, not its underlying read width.

Descriptive labels identify references, not separate allocations or nonaliasing guarantees.

### Outputs and preconditions

There is no represented normal return, result status, or structured error report. The visible sequence ends at a bad-instruction marker; actual trap, halt, recovery, or continuation behavior is unknown.

Continued execution requires each reached access to be permitted and both indirect calls to return. No explicit bounds, null, alignment, permission, or arithmetic-validity checks appear. Destinations include low absolute addresses and addresses derived from arguments, stack storage, globals, and dependency addresses.

## Behavior

### Interpretation rules

The following order is the represented statement-level order, not a measured instruction or bus trace.

Explicitly typed word, halfword, and byte operations have widths of 32, 16, and 8 bits. Pointer-store width remains dependent on the unresolved pointer model. Access widths of undeclared globals are not inferred from symbol spelling, address, assigned constant, or surrounding expression.

Signed 32-bit multiplication and left shifts do not establish unrestricted wraparound behavior. Negative signed right shifts require target semantics before a particular result can be asserted. Unsigned 32-bit arithmetic follows its represented width, but the resulting numeric address still requires a compatible address model.

Separate represented accesses must remain distinct. The representation does not establish the evaluation order of the two pointer-product global references within the self-address assignment.

### 1. Initial writes and first call

The routine first writes a signed 32-bit product of the first argument and `0x20000` to the address formed by multiplying that argument by `0x40000`. Both calculations are represented as signed; overflow behavior is unresolved.

It retains the unsigned third argument logically shifted right by five for later use.

A 32-bit write places `0x0897E9C8`, the represented address of F-dd66fecc047e, at the ambient base plus eight.

The first indirect call, labeled F-indirect-01 here, targets the ambient return-link value plus `0x0C`. Its single displayed argument is the signed numeric address of F-dd66fecc047e shifted right by 16. For the positive address shown, that expression yields `0x0897`.

This is not a represented invocation of F-dd66fecc047e. Its address is used as data. F-indirect-01’s full interface, effects, preservation rules, and return behavior are unknown.

After that call returns, the routine writes the first argument shifted left by 21 through the fourth argument as a 32-bit value.

### 2. Second call and third-argument accesses

The second indirect call, labeled F-indirect-02 here, has displayed target `0x0897EA31` and no displayed explicit arguments. Its result is represented as 64 bits, subsequently separated into low and high 32-bit portions. This does not independently establish its original ABI or return type.

After it returns, the following operations appear in order:

1. Write the third argument’s low 16 bits at its numeric address plus `0x12`.
2. Write the retained shifted address to itself as a 32-bit word.
3. Write the low 32 result bits at the third argument plus `0x10`.
4. Repeat the halfword write at offset `0x12`.
5. Repeat the shifted-address self-store.
6. Interpret the low result portion as signed 32-bit, shift right by 16, and write 32 bits at the third argument plus four.
7. Repeat the halfword write at offset `0x12`.
8. Repeat the shifted-address self-store.
9. Read a signed 16-bit value from the ambient base plus the fourth argument’s signed numeric address.
10. Repeat the halfword write at offset `0x12`.
11. Repeat the shifted-address self-store.
12. Widen the retained signed halfword to signed 32-bit, shift right by 16, and write 32 bits at the third argument plus eight.

There are four halfword writes at offset `0x12` and four shifted-address self-stores.

The word at offset `0x10` overlaps the halfword at offset `0x12`. Under little-endian interpretation, the later halfword replaces the word’s upper 16 bits; under big-endian interpretation, it replaces the lower 16 bits. Other aliases can further affect the result.

Under arithmetic signed-right-shift semantics, the final write at offset eight is zero for a nonnegative sampled halfword and `0xFFFFFFFF` for a negative one.

### 3. Selector and pointer distribution

The routine assigns zero to an abort-related global. Its storage declaration and access width are absent.

It forms a selector by retaining bit two of the shifted address at its positional value, zero or four, and adding `0x58`. The selector is therefore `0x58` or `0x5C`.

The following writes occur:

1. Write the ambient base as 32 bits at selector plus four: `0x5C` or `0x60`.
2. Store the fourth argument’s pointer at the high result portion plus four.
3. Store that pointer at four times the selector plus `0x10`: `0x170` or `0x180`.
4. Store that pointer at the high result portion plus eight.

No disjointness between these destinations or other storage is established.

### 4. Incoming-pointer-derived accesses

Using the incoming stack-associated pointer, the routine:

1. Writes the selector as 16 bits at the pointed-to address.
2. Writes the ambient base shifted left by eight as 32 bits at that same address.
3. Writes that shifted ambient value as 32 bits at byte offset `0x14`.
4. Stores the incoming pointer at an address formed from its numeric value plus its bitwise intersection with the high result portion.
5. Reads 32 bits from twice the incoming pointer’s signed numeric value.
6. Writes the low 16 bits of that read value at the incoming pointer’s destination.
7. Treats the complete read value as an address and writes that same 32-bit value there.

The word write in step 2 overlaps the preceding halfword write. Both remain separate accesses.

Next, the routine writes the incoming pointer’s signed numeric value shifted right by 31 as 32 bits to an address calculated from the **address of the incoming stack slot**, multiplied by `0x20`, plus `0x18`. This destination is not calculated from the pointer contained in the slot.

Storage associated with absolute address `0x00000005` then receives five; the access width is unresolved. A pointer store places the incoming pointer at four times its signed numeric value plus `0x18`.

### 5. Late-source conversion and pointer-product references

The routine reads the late-source global and assigns the result to a signed 32-bit local. Subsequent late-source calculations use this retained local value.

The source access width and signedness are unknown. The local assignment may involve extension, truncation, or another type-dependent conversion. Consequently, no particular local value follows solely from a proposed byte pattern at the global’s address.

The routine assigns four times the incoming pointer’s signed numeric value to the pointer-product global. The product expression is signed 32-bit; the destination storage type is unknown. Assignment may truncate or otherwise convert the product.

The following self-address assignment contains two separate references to that global:

- One supplies the address for an explicitly 32-bit destination.
- The other supplies the value converted for that 32-bit store.

Their evaluation order is not established. Their underlying access widths and conversions are also unresolved. Neither reference may be replaced unconditionally with the originally assigned product.

After this store, another reference to the pointer-product global supplies the value used in a destination expression adding four. The routine stores the fourth argument’s pointer at that resulting address. The global’s missing declaration prevents establishing the addition’s complete arithmetic type and conversion rules. The preceding store may also have changed the global through aliasing.

Using the retained signed 32-bit late-source local, the routine then:

1. Writes four times that local as 32 bits at the address denoted by the same product.
2. Writes `0x225FA918`, four times the represented address of F-8afdaab91693, as 32 bits to the address formed by adding the local to the result of shifting it left by six and then right by 17.

These signed calculations retain the arithmetic limitations stated above. F-8afdaab91693 is not invoked.

The routine logically shifts the retained late-source local’s unsigned 32-bit representation right by 16, converts the result to a 16-bit expression, and assigns that expression to the reset-fragment global. This establishes the assigned expression’s bits, not the global’s access width or eventual stored representation.

It next assigns zero to an interrupt-related global of unresolved width.

### 6. Dependency-address writes and final assignments

Eight times the represented address of F-8afdaab91693 is `0x44BF5230`. The routine writes consecutive 32-bit zero words at that base and base plus four.

At base plus `0x10`, it writes one byte calculated from the base address’s low-byte conversion plus eight. Under the represented address conversion, the stored byte is `0x38`.

The remaining operations are:

1. Assign `0x897EA700` to global storage associated with `0x00000086`.
2. Assign `0x66` to storage associated with `0xEA700000`.
3. Assign `0xEAEC` to storage associated with `0x0897EB26`.
4. Assign `0x1A` to storage associated with `0x12FD4C34`.
5. Store address value `0x897EA65C` into storage associated with that same address.
6. Reference the reset-fragment global again and use its value as the upper 16-bit operand of a concatenation whose lower operand is `0xEA70`; assign the resulting 32-bit expression to a reset-related global.
7. Assign `0x5C` to storage associated with `0x12FD4CB8`.

The reset-fragment reference is fresh, not a reuse of the earlier assigned expression. Its storage read width and conversion into the concatenation operand remain unknown. Likewise, the 32-bit concatenation result does not establish the destination global’s width; assignment may convert or truncate it.

Declarations for the other final globals are also absent. Their actual access widths and stored results cannot be derived solely from these expressions.

The visible sequence then reaches the bad-instruction termination marker.

## Side effects

Established explicit memory operations include 32-bit, 16-bit, and byte writes; a signed 16-bit read using the ambient base and fourth argument; and a 32-bit read at twice the incoming pointer’s numeric value.

Separately represented global references include:

- The late-source read converted into a retained signed 32-bit local.
- The pointer-product assignment.
- Two pointer-product references within the self-address assignment.
- A further pointer-product reference for the following destination.
- The reset-fragment assignment and its later reread.
- Additional global assignments whose declarations are unavailable.

These global references must not be described as established 32-bit or 16-bit storage accesses. Missing declarations leave read extension, assignment truncation, and other conversion behavior unresolved.

Aliasing may connect any destination with later read sources, globals, stack storage, or callee-visible data. Unknown calls may independently alter memory or machine state. Neither callee may be assumed to preserve globals or register-derived values.

No allocation, release, ownership transfer, locking, or cleanup is established. Pointer stores alone do not transfer ownership or establish later retention or release.

No rollback is represented. A fault or nonreturning call can leave preceding effects committed without a structured error result.

## Dependencies

| Opaque identifier | Established use | Unresolved contract |
|---|---|---|
| F-dd66fecc047e | Address `0x0897E9C8` is stored and used to form the first call’s displayed argument. | No invocation or implementation behavior is established. |
| F-8afdaab91693 | Address `0x0897EA46` supplies numeric products and destinations. | No invocation or implementation behavior is established. |
| F-indirect-01 | Indirect call at ambient return-link plus `0x0C`. | Identity, full ABI, effects, preservation, and return behavior. |
| F-indirect-02 | Indirect call at displayed address `0x0897EA31`; represented 64-bit result. | Identity, implicit inputs, actual result ABI, effects, and preservation. |

F-indirect-01 and F-indirect-02 are contract-local opaque labels, not resolved function identities.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Repeated accesses:** If the full third-argument sequence is reached, it includes four halfword writes at offset `0x12` and four shifted-address self-stores. Later overwrites do not erase earlier accesses from the contract.

- **Signed halfword:** If the explicit signed halfword read yields `0x8000` and signed right shift is arithmetic, the write at third argument plus eight is `0xFFFFFFFF`. A sampled `0x7FFF` instead produces zero.

- **Selector:** A clear bit two in the shifted address selects `0x58`, with destinations `0x5C` and `0x170`. A set bit selects `0x5C`, with destinations `0x60` and `0x180`.

- **Overlap:** Under little-endian memory, low result `0x11223344`, third-argument low halfword `0xABCD`, and no other relevant alias, the subsequent halfword write leaves `0xABCD3344` at offset `0x10`.

- **Late-source conversion:** If the undeclared global read, after conversion into the signed 32-bit local, yields `0x12345678`, the expression assigned to the reset-fragment global is the 16-bit value `0x1234`. This does not assert the global’s write width or exact stored representation.

- **Pointer-product ordinary storage:** If the global’s declaration and conversions preserve an assigned product of `0x1000`, both self-assignment references supply that value, and the later addition evaluates to `0x1004`, the self-store writes 32-bit `0x1000` at `0x1000`, followed by the pointer store at `0x1004`.

- **Pointer-product differing references:** If the destination-role reference resolves to address `0x2000` and the value-role reference converts to 32-bit `0x3000`, the self-store writes `0x3000` at `0x2000`. If the later reference and addition resolve to `0x4004`, that is the following pointer-store destination. No global access width is implied.

- **Reset reread:** If the earlier assigned 16-bit expression is `0x1234`, but the final global reference supplies concatenation operand `0x5678`, the concatenation expression is `0x5678EA70`. The destination’s unresolved declaration still determines what is stored.

- **Stack-slot distinction:** The special stack-derived destination uses the slot’s address, not its contained pointer.

- **Early interruption:** If F-indirect-01 never returns, the initial product write and ambient-base-plus-eight write precede it; the write through the fourth argument has not occurred.

## Uncertainties

The following remain **documented_only**:

- Validity and completeness of the recovered instruction stream.
- Application purpose, success conditions, and terminal behavior.
- Original ABI, pointer width, endianness, alignment, and memory semantics.
- Signed overflow, signed left shifts, and negative signed right shifts.
- Indirect-call interfaces, identities, effects, and preservation rules.
- Undeclared globals’ types, access widths, signedness, aliases, and conversion rules.
- Whether storage has read or write side effects.
- Evaluation order of the two pointer-product references within the self-address assignment.
- Actual machine-level loads corresponding to represented references.
- Validity of apparent register-derived values across unknown calls.

An initialization-like intent is conceivable but inferred only; it imposes no acceptance requirement.

## Evidence

The sole evidence is the supplied static representation, including its invalid-instruction and overlapping-global warnings.

Explicit access casts and local declarations establish certain operation and expression widths. They do not supply the missing global declarations. The representation separately shows the pointer-product assignment, its subsequent references, and the reset-fragment reread; those distinctions are retained without inventing storage types.

No dependency inspection, runtime execution, memory tracing, or tests have been performed. No successful execution or normal return is claimed.
