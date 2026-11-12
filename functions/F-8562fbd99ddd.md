<!-- Trusted provenance: F-8562fbd99ddd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8562fbd99ddd

## Purpose

F-8562fbd99ddd performs a sequence of calls through computed targets, derives addresses and small stored values from their results, and conditionally writes a short record through a pointer supplied by F-c260788e6b26.

The supplied static representation establishes memory operations and call ordering, but not an application-level purpose. Describing this function as an allocator, renderer, parser, or initializer would exceed the evidence.

There is one explicit ordinary return path that is feasible under the displayed integer widths. It follows a byte-dependent calculation whose result is necessarily nonnegative. Other paths reach an invalid-instruction boundary, or appear in a later continuation that is inconsistent with that calculation.

This contract distinguishes:

- **Established static evidence:** operations and dependencies represented in the supplied material.
- **Inferred interpretation:** target-width arithmetic and packed-return conventions needed to interpret that material.
- **documented_only:** unresolved behavior, including unreliable control flow and unknown callee effects.

No runtime tests have been performed.

## Interface

### Entry and result

The displayed interface has no explicit parameters and no declared return value.

Nevertheless, execution depends on implicit machine state:

- An incoming link-related value participates in constructing the first call target.
- An incoming 32-bit register value controls the main conditional path.
- Calls produce values in additional result registers, including pairs represented as a single 64-bit quantity.
- Memory supplies a global 32-bit base value and several referenced data objects.

The absence of explicit parameters is therefore not evidence that the function is independent of its caller.

Successful completion on the feasible ordinary-return path provides no specified return payload. The observable result consists of ordered calls, memory writes, and any effects of the callees.

### Widths and interpretation

The representation uses:

| Item | Represented width |
|---|---:|
| Byte read or write | 8 bits |
| Halfword read or write | 16 bits |
| Word read or write | 32 bits |
| Packed result and first record write | 64 bits |
| Address used in displayed arithmetic | 32 bits |

A packed result is interpreted by taking its low 32 bits as one value and its high 32 bits as another. This interpretation does not establish a public ABI for any dependency.

Narrow stores retain only their destination width. In particular, converting a value to a halfword retains its low 16 bits; storing that halfword as a byte retains its low 8 bits.

Signed comparisons and signed-overflow detection affect control flow. The representation also contains arithmetic that could overflow a signed 32-bit value. Exact machine wraparound, invalid pointer behavior, byte order, and unaligned-access behavior require target information not supplied here.

### Preconditions

For execution to reach its ordinary return:

- Every computed call target actually invoked must be executable under the target’s calling convention.
- Every memory read and write must be valid for its width.
- Returned address components must identify accessible storage wherever used.
- Callees must return normally.
- The implicit branch input must satisfy both conditions described below.
- F-c260788e6b26 must return a high 32-bit component identifying at least 20 writable bytes.

These are execution requirements, not checks implemented by this function. No null checking, bounds checking, allocation-failure handling, or rollback is visible.

## Behavior

### 1. Obtain a derived integer and make a preparatory call

The function first invokes an unresolved computed target without explicit arguments.

It passes that result and a reference to a static data object to a second unresolved target. The second call produces a packed result.

Next, it:

1. Reads a 16-bit value at offset 12 from the address represented by the packed result’s high 32 bits.
2. Calls a third unresolved target with the packed result’s low 32 bits and that 16-bit value.
3. Retains the third call’s 32-bit integer result as the **derived integer**.
4. Forms a halfword from the derived integer shifted left by two positions, retaining the low 16 bits.
5. Calls a fourth unresolved target with the derived integer plus 80 and the integer 50.

The fourth call’s result is retained for use if the first branch permits continuation.

The 16-bit read precedes the third call. No preservation of other memory or global state across these calls is established.

### 2. Evaluate the implicit branch input

The displayed control flow derives two Boolean conditions from the incoming 32-bit branch input:

- A negativity condition for that input plus 90.
- A signed-overflow condition for adding 91 to the intermediate value obtained by subtracting 1 from that input.

If the negativity condition is true, execution goes directly to an invalid-instruction termination boundary. The writes in the following sections are not reached.

If the negativity condition is false, execution continues with the writes below. A second conditional later compares the two retained Boolean conditions. Because the negativity condition is already false on that path, ordinary continuation requires the overflow condition to be false as well.

The two conditions must not be replaced unconditionally with a single mathematical comparison. At 32-bit boundaries, the intermediate subtraction and overflow flag matter.

### 3. Perform writes before the second conditional

On the nonnegative path, the function calls another unresolved target with:

- The fourth call’s retained result.
- A reference to a second static data object.

It then performs the following operations in order:

1. Writes a 16-bit value at offset 4 from the returned address. The stored value is the low 16 bits of that address itself.
2. Invokes an unresolved target without explicit arguments.
3. Writes one byte at an address calculated as four times the derived integer, plus 9. The byte is the low 8 bits of the previously formed halfword.
4. Invokes another unresolved target without explicit arguments and retains its returned address.

Only after these operations does the function test whether the two retained Boolean conditions are equal.

If they are unequal, the function reaches the invalid-instruction boundary. The preceding calls and writes have already occurred; there is no visible rollback.

If they are equal, execution continues.

### 4. Update the returned object and a fixed location

Using the address returned by the most recent call, the function:

1. Writes the retained halfword at offset 20.
2. Writes its low byte at offset 26.
3. Writes the value 71 to a fixed global location at address `0xA2`.

The last operation is represented as a write through a global symbol whose size is not recoverable from the supplied declaration. Its exact access width is therefore **documented_only**.

The function then loads a global 32-bit value, referred to here as the **captured base**. This load occurs after the fixed-location write and before the next call.

That timing matters: subsequent calculations use the captured value. A later callee changing the global does not, by itself, replace the captured value.

### 5. Derive the byte-dependent record fields

The function calls an unresolved target with the integer 29 and receives another packed result.

It reads one unsigned byte at offset 13 from the address represented by that result’s high 32 bits. Call this byte **B**, with range 0 through 255.

It then:

1. Forms a predecessor value equal to the captured base minus 1.
2. Writes the 32-bit value 51 at the address calculated as the captured base plus B minus 2.
3. Forms a second address-like value equal to the captured base plus 108.
4. Calls an unresolved target with twice the quantity obtained by subtracting 54 from the packed result’s low 32 bits.
5. Forms a record value equal to twice B plus 99.
6. Tests whether that record value is nonnegative.

The byte-indexed write is a full 32-bit write. Its destination is not necessarily aligned, and no alignment adjustment is shown.

The record value ranges from 99 through 609 inclusive. Its calculation cannot overflow a signed 32-bit integer. Consequently, under the displayed unsigned-byte semantics, the nonnegative test always succeeds.

### 6. Populate the returned record and return

On the feasible path, the function calls F-c260788e6b26 with the integer 115.

Let **R** be that dependency’s full packed 64-bit result, and let **P** be the address represented by its high 32 bits. The function performs these stores in order:

| Destination | Width | Stored value |
|---|---:|---|
| P + 0 | 64 bits | R |
| P + 8 | 32 bits | Twice B plus 99 |
| P + 12 | 32 bits | Captured base plus 108 |
| P + 16 | 32 bits | Captured base minus 1 |

It then returns normally.

The first write is the full packed result, not merely its low word. Interpreting the bytes at offsets 0 and 4 requires the target’s byte order.

Nothing establishes that P is newly allocated, uniquely owned, or distinct from previously accessed storage. If destinations alias, later stores observe the ordinary consequences of that aliasing.

### 7. Later represented continuation

The supplied material also contains a continuation reached only if twice B plus 99 is negative.

That condition is impossible for the displayed byte width and arithmetic. Accordingly, the continuation is **documented_only**, not a required executable phase of the feasible path.

It contains additional word reads relative to the captured base, further computed calls, derived-address stores, a call to F-943b4f706744 with a data-derived argument and 67, signed-byte and unsigned-halfword reads, and more writes through addresses derived from intermediate arithmetic and auxiliary result registers. It ends with further computed calls and an unresolved indirect transfer.

Its presence does not establish that the normal path invokes F-943b4f706744. If later evidence makes this continuation reachable, its detailed executable contract must be reassessed rather than inferred from the current inconsistent control flow.

## Side effects

### Direct writes

On the feasible ordinary-return path, the visible writes occur in this order:

1. Halfword at offset 4 from an unresolved call’s returned address.
2. Byte at four times the derived integer plus 9.
3. Halfword at offset 20 from a later returned address.
4. Byte at offset 26 from that address.
5. Value 71 at the fixed global location.
6. Word value 51 at captured base plus B minus 2.
7. Four record stores through P, covering offsets 0 through 19.

Unresolved calls occur between several of these writes. The function is not a single atomic update.

### Ownership and lifetime

No ownership transfer is established for any pointer or returned storage. In particular:

- F-c260788e6b26 is not proven to allocate memory.
- No explicit release operation is identifiable on the feasible path.
- Unknown callees may alter globals, mutate shared memory, or affect storage lifetime.
- No callee is assumed to preserve unrelated memory merely because its return value is ignored.

### Failure behavior

There is no explicit status result or recoverable error branch.

The representation terminates certain paths at an invalid-instruction boundary. Whether an actual execution traps there, diverges from the recovered representation, or follows missing instructions is unresolved.

Invalid memory, invalid call targets, or callee failures may prevent completion after some writes have occurred.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| F-c260788e6b26 | Called with 115 on the feasible return path. Its high result component supplies the record destination; its full packed result is stored there. |
| F-943b4f706744 | Appears only in the inconsistent later continuation, with a data-derived first argument and second argument 67. |

Other calls use computed targets without supplied opaque identities. They remain unresolved dependencies; no identity, purpose, ownership convention, or global-preservation guarantee is assigned to them.

Static data dependencies include two objects passed by reference, the captured global base, and the fixed location receiving 71. Their broader layouts and application meanings are unknown.

## Acceptance examples

These examples are conditional static expectations, not runtime test results.

### Ordinary completion with a zero byte

Assume all calls return normally, all accesses are valid, and the branch input permits both conditions.

If B is 0:

- The byte-indexed word write stores 51 at captured base minus 2.
- The record’s word at offset 8 is 99.
- The words at offsets 12 and 16 are captured base plus 108 and captured base minus 1.
- F-c260788e6b26 receives 115.
- The function returns after the four record stores.

### Largest byte value

Under the same assumptions, if B is 255:

- The word write occurs at captured base plus 253.
- The record value is 609.
- The negative-record continuation is not selected.

A claim that this byte value makes the record calculation negative contradicts the displayed widths.

### Narrowing the derived value

If the derived integer is 16,385, its fourfold value is 65,540:

- The retained halfword is 4.
- Stores using that halfword write 4.
- Stores using its low byte also write 4.
- The separately calculated byte destination is 65,549, assuming ordinary 32-bit arithmetic without overflow.

Narrowing the stored value does not narrow the separately calculated address.

### First branch rejects continuation

For an implicit branch input of −91, the relevant additions are representable:

- The negativity condition is true because the sum with 90 is −1.
- The preparatory calls have already occurred.
- The later object, fixed-location, byte-indexed, and record writes are not reached.
- The represented path terminates at the invalid-instruction boundary.

### Second branch rejects after partial effects

If the negativity condition is false but the overflow condition is true:

- The halfword-at-offset-4 write and derived-address byte write occur.
- The following no-argument call occurs.
- The offset-20, offset-26, fixed-location, and record writes are not reached.
- Earlier effects remain; no rollback is shown.

### Global mutation after capture

If the call receiving 29 changes the global base, but returns normally, the later address and record calculations still use the value captured before that call. This does not imply that any memory at those calculated addresses remains unchanged.

## Uncertainties

The following remain **documented_only**:

- The application-level purpose and surrounding object model.
- The true public calling convention and implicit-input contract.
- Identities and side effects of computed-call dependencies.
- Exact target behavior for overflowing signed arithmetic and invalid addresses.
- Byte order and alignment requirements.
- The width of the fixed-location write of 71.
- Ownership, allocation, and lifetime of returned storage.
- Executability of paths ending at the invalid-instruction boundary.
- Whether the inconsistent later continuation reflects missing instructions, incorrect recovery, or genuinely unreachable material.
- The final unresolved indirect transfer.

None of these unknowns is resolved by assigning a plausible subsystem role to the function.

## Evidence

This contract is based solely on the supplied static function representation and dependency mapping.

The strongest evidence is the explicit access widths, store order, call arguments, low/high splitting of packed results, branch expressions, and ordinary return after record construction.

The supplied material also reports invalid instruction data, removed unreachable regions, overlapping global symbols, and an unrecovered indirect transfer. Those limitations reduce confidence in complete control-flow recovery.

The range 99 through 609 follows from the explicit unsigned-byte read and subsequent arithmetic. It establishes the inconsistency of the negative-record continuation under the displayed widths; it does not independently establish the original binary’s complete behavior.

No implementation access, runtime execution, memory trace, or test result supports this document.
