<!-- Trusted provenance: F-40fcaf87b7ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-40fcaf87b7ef

## Purpose

F-40fcaf87b7ef selects a quantity from a caller-supplied position and a descriptor’s length-like field, constructs two 32-bit address-like words, and calls F-c8e9cf68c1b6. It maps that dependency’s result to a 32-bit status.

If the dependency returns zero, this function rereads the descriptor reference, writes the original position into the selected descriptor, and returns zero. If the dependency returns nonzero, this function returns three without performing that write.

**Inferred intent:** the quantity cap of 16,384, position scaling by four, and added displacement of 16 suggest a bounded operation over four-byte units following a prefix. The evidence does not establish those semantic units, the nature of the address-like value, or the operation performed by the dependency.

This contract distinguishes behavior established by the supplied static representation from overflow-sensitive outcomes and dependency behavior that remain **documented_only**.

## Interface

### Inputs

The function receives two 32-bit inputs:

| Input | Interpretation established by use |
|---|---|
| State reference | An address-like value used to access a mutable state object |
| Position | A signed 32-bit integer used in quantity selection, two multiplication expressions, and the success-path write |

No local input validation is visible.

### Accessed layouts

The state object has the following accessed fields:

| Byte offset | Width | Use |
|---|---:|---|
| `0x20` | 32 bits | Descriptor reference, read at multiple stages |
| `0xE0` | 32 bits | Low base word, read separately for the low result and first carry |
| `0xE4` | 32 bits | High base word used in the dependency’s high address argument |

The descriptor has these accessed fields:

| Byte offset | Width | Use |
|---|---:|---|
| `0x00` | 32 bits | Signed length-like value used in quantity selection |
| `0x04` | 32 bits | Destination of the original position after dependency success |
| `0x08` | 32 bits | Value supplied as the dependency’s second argument |

These tables describe accessed locations, not complete object layouts. They establish neither the meaning of intervening storage nor the total allocation sizes.

The descriptor reference is not retained for every subsequent access. The initial length read, dependency-argument read, and final write may select different descriptors if the reference changes between those stages.

### Output

The return value is 32 bits:

- **0:** the dependency returned zero, and the subsequent position write completed.
- **3:** the dependency returned nonzero.

The dependency’s particular nonzero result is discarded. No other normal return value is visible.

An invalid access or a dependency that does not return normally is not shown being converted into either status.

### Preconditions and ownership

Normal completion requires:

- Readable state fields at the accessed offsets.
- A readable length-like field in the initially selected descriptor.
- A readable field at offset `0x08` in the descriptor selected during argument preparation.
- Following a zero dependency result, a writable field at offset `0x04` in the descriptor selected after the call.
- A normally returning dependency.
- An arithmetic interpretation sufficient to evaluate the expressions involved; overflow-sensitive cases are qualified below.

There is no visible allocation, deallocation, reference-count operation, or explicit ownership transfer performed locally. Passing the state reference to the dependency does not establish whether that dependency preserves, retains, transfers, or releases any referenced storage.

## Behavior

### Initial read and quantity selection

The function first reads the descriptor reference at state offset `0x20`, then reads the signed 32-bit length-like value at descriptor offset `0x00`.

It compares the position plus 16,384 against that length-like value using a strict signed less-than comparison:

- If the sum is less than the length-like value, the selected quantity is 16,384.
- Otherwise, the selected quantity is the length-like value minus the position.

The addition and subtraction are signed 32-bit expressions. Their overflow behavior is not independently established by the supplied representation. The numerical rules above are unconditional only where the relevant signed operations are representable, or where a separately established execution model defines the overflow.

For nonnegative positions no greater than the length, with representable arithmetic, the quantity is the smaller of 16,384 and the remaining length.

Boundary behavior includes:

- A remaining length of exactly 16,384 selects the subtraction branch and produces 16,384.
- Position equal to length produces zero.
- Position greater than length can produce a negative quantity.
- Negative positions and negative length-like values are not locally rejected.
- A zero or negative selected quantity does not bypass the dependency call.

The quantity is selected before the call. Later changes to the length-like field do not cause a visible recalculation.

### Multiplication and its arithmetic qualification

Address construction contains **two occurrences** of signed 32-bit multiplication of the position by four:

1. One participates in the initial low-word addition.
2. The other participates in the first carry calculation.

The multiplication is representable for positions from −536,870,912 through 536,870,911, inclusive.

Within that range, the product’s 32-bit representation supplies the operand used by the low-word addition and carry calculation. A representable negative product is treated through its 32-bit unsigned word representation for those operations.

Outside that range, the supplied representation alone does **not** establish that either multiplication wraps or yields the low 32 bits of the mathematical product. The unsigned surrounding operations do not, by themselves, settle the overflow behavior of the signed multiplication.

Accordingly:

- The address rules below apply directly when both signed multiplications are representable.
- Under an independently assumed wrapping machine-arithmetic model, each multiplication supplies the mathematical product reduced modulo \(2^{32}\).
- Without such a model, overflow-sensitive address results are **documented_only**. Neither the low-word calculation nor the first carry calculation has an unconditional truncated-product guarantee.

### Low address word

After quantity selection, the function reads the low base word at state offset `0xE0`.

It adds the first multiplication’s 32-bit operand to that base word using unsigned 32-bit addition. The retained intermediate word is the sum modulo \(2^{32}\), subject to the multiplication qualification above.

The low address argument supplied to the dependency is that retained intermediate word plus 16, reduced modulo \(2^{32}\).

The intermediate is retained: the later carry for adding 16 uses this value rather than a newly read low base word.

### High address word

The high address argument combines three contributions:

1. The 32-bit word read at state offset `0xE4`.
2. A carry from unsigned 32-bit addition of a **second read** of state offset `0xE0` and the operand from the **second signed multiplication** by four.
3. A carry for adding 16 to the retained intermediate low word.

The first carry is one when the corresponding unsigned sum exceeds `0xFFFFFFFF`; otherwise it is zero. This rule remains conditional on the second multiplication having defined or independently specified behavior.

The second carry is one exactly when the retained intermediate word is at least `0xFFFFFFF0`; otherwise it is zero. Thus:

- `0xFFFFFFEF` generates no second carry.
- `0xFFFFFFF0` generates a second carry.

The high-word additions retain a 32-bit result. No further address word is supplied if that result overflows.

Both carry contributions can be one in the same invocation. Conversely, if the low base word changes between reads, the first carry need not correspond to the addition that produced the retained intermediate.

With stable base fields and defined products, the construction resembles adding the unsigned 32-bit representation of the product, then 16, to a two-word base. It does not establish a full-width signed multiplication or sign extension of a negative product into the high word.

### Dependency invocation

The function calls F-c8e9cf68c1b6 with five arguments in this order:

| Argument | Supplied value |
|---|---|
| 1 | Original state reference |
| 2 | 32-bit descriptor field at offset `0x08`, using a descriptor reference read during call preparation |
| 3 | Computed low address word |
| 4 | Computed high address word |
| 5 | Selected quantity |

There is one visible dependency invocation on every normal path reaching the call. Zero and negative quantities are passed through without a local shortcut or rejection.

The initial descriptor and length reads precede quantity selection. The first low-base read and retained-intermediate calculation precede the call. Reads within argument preparation must complete before dependency entry, but the supplied representation does not establish a total order among the descriptor argument read, high-base read, and repeated low-base read.

### Result handling

If the dependency returns zero, the function:

- Rereads the descriptor reference at state offset `0x20`.
- Writes the original position’s 32-bit representation at offset `0x04` in that descriptor.
- Returns zero.

The write therefore targets the descriptor selected **after** the dependency returns. It is not tied to either earlier descriptor selection.

If the dependency returns nonzero, the function returns three without performing this position write.

There is no visible retry, rollback, second dependency invocation, or recovery path for an invalid post-call destination.

## Side effects

### Direct memory effects

The directly visible reads are:

- The initial descriptor reference and its length-like field.
- The low base word used to form the retained intermediate.
- Another descriptor reference and its field at offset `0x08`.
- The high base word.
- Another low base word used for the first carry.
- On dependency success, the post-call descriptor reference.

The only explicit data write by this function is the success-path 32-bit position write at descriptor offset `0x04`.

Returning three does **not** imply unchanged memory. The dependency receives the state reference and may modify memory or external state.

### State and lifetime across the call

No preservation guarantee is established for:

- The state’s descriptor reference.
- Descriptor fields.
- Either base word.
- Global state.
- The lifetime or ownership of referenced objects.

Changes during the call cannot retroactively alter scalar arguments already supplied. They can affect the descriptor selected for the final write.

If the dependency returns zero after invalidating the state or selected descriptor, the function has no visible check before its post-call accesses. Any resulting access failure is not shown being mapped to status three.

### Concurrency

No locking, atomic operation, barrier, or coherent-snapshot mechanism is visible.

The repeated reads are structurally distinct and need not be described as a single cached snapshot. This does not establish that unsynchronized concurrent mutation is supported or has defined execution semantics.

## Dependencies

### F-c8e9cf68c1b6

**Established static behavior at this call site:**

- Receives the five arguments listed above.
- Is invoked once on the normal path.
- Returns a value tested for zero versus nonzero.
- A zero result enables the descriptor reread, position write, and return value zero.
- Any nonzero result produces return value three without the local position write.

**Inferred intent:**

The cap and address scaling suggest a bounded range operation. The evidence does not establish whether the dependency reads, writes, seeks, transfers, or performs another operation.

**Documented_only:**

- Exact semantic argument types and units.
- Meaning of the descriptor field at offset `0x08`.
- Meaning and address space of the two address words.
- Interpretation of zero and nonzero results beyond the caller’s branch.
- Treatment of zero or negative quantities.
- Validation, partial progress, allocation, retention, release, and other side effects.
- Global-state preservation and object-lifetime guarantees.
- Whether execution always returns normally.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Unless an example states otherwise, assume valid storage, stable fields during argument preparation, representable signed arithmetic—including **both** multiplications by four—and a normally returning dependency.

### Capped and exact-boundary quantities

With length 50,000 and position 100, the dependency receives quantity 16,384.

With length 20,000 and position 3,616, it also receives 16,384. Equality takes the subtraction branch.

For either case, dependency result zero causes the original position to be written to the post-call selected descriptor and produces return value zero.

### Short remainder and failure

With length 1,000 and position 900, the quantity is 100.

If the dependency returns any nonzero value, the function returns three and performs no position write of its own. Dependency effects are not rolled back.

### End position and beyond

With length 1,000 and position 1,000, the dependency is still called with quantity zero. A zero dependency result causes a write of 1,000 and return value zero.

With length 100 and position 101, the quantity is −1. There is no local clamp or rejection. The dependency’s treatment of that quantity remains unknown.

### Negative position with representable multiplication

With length 100 and position −1, the quantity is 101.

Both multiplications produce the representable signed value −4. Its unsigned 32-bit word representation is `0xFFFFFFFC`. No multiplication-overflow assumption is required for this example.

With both base words zero, the low address word is 12 and the high address word is one: adding 16 to the retained intermediate generates the carry. This is not a sign-extended subtraction of four from a full-width base.

### Final-addition carry boundary

With position zero, low base word `0xFFFFFFF0`, and high base word 7, the address arguments are low zero and high 8.

With low base word `0xFFFFFFEF` instead, they are low `0xFFFFFFFF` and high 7.

### Both carries contribute

With position −1, low base word `0xFFFFFFFC`, and high base word 7:

- The representable product −4 supplies unsigned word `0xFFFFFFFC`.
- The first unsigned addition produces intermediate `0xFFFFFFF8` and a carry.
- Adding 16 produces low word 8 and another carry.
- The high address word is 9.

Both carry contributions are required.

### Multiplication overflow requires an extra assumption

With position `0x40000000`, multiplication by four is outside the signed 32-bit range. The address arguments are not unconditionally established by the supplied representation.

**Only under an assumed wrapping machine-arithmetic model for both multiplication occurrences**, each product supplies word zero. With both base words zero, the resulting address arguments are low 16 and high zero.

Without that model, those address results remain documented_only. The function’s complete overflow-sensitive execution is not established merely because its quantity arithmetic happens to be representable.

### Descriptor replacement during the call

Suppose earlier reads select descriptor A, and the dependency changes the state’s descriptor reference to descriptor B.

If the dependency returns zero and B is writable, the final position write targets B at offset `0x04`.

If the dependency returns nonzero, this function performs no position write to either descriptor.

## Uncertainties

The following remain **documented_only**:

- Overflow behavior of the signed addition and subtraction used for quantity selection.
- Overflow behavior of **each** signed multiplication by four, including the occurrence used for carry calculation.
- Exact ordering of reads within argument preparation.
- Alignment requirements and access-fault behavior.
- Full object layouts and semantic field names.
- Dependency effects, ownership rules, lifetime changes, and nonlocal failures.
- Supported concurrency and mutation rules.

The bounded-range interpretation is inferred intent, not a guarantee of validation, safety, or meaningful handling of every 32-bit input.

## Evidence

This contract uses only the supplied static representation and dependency mapping.

Static evidence establishes the accessed offsets and widths, signed quantity comparison, two signed multiplication occurrences, unsigned low-word arithmetic, two carry contributions, five-argument dependency call, zero-versus-nonzero result mapping, and post-call descriptor reread before the success-path write.

The representation does not independently establish wrapping behavior for overflowing signed multiplication. All such outcomes are either explicitly conditional on a wrapping model or retained as documented_only.

No implementation inspection beyond the supplied material, runtime execution, dependency execution, fault injection, or tests have been performed. The acceptance examples are static expectations, not observed results.
