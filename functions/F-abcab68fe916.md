<!-- Trusted provenance: F-abcab68fe916; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-abcab68fe916

## Purpose

The established behavior is a state-dependent sequence of raw memory operations followed, if execution reaches it, by an unbounded indirect-call and software-interrupt cycle.

No application-level purpose is established. The supplied representation does not support identifying this operation as a parser, allocator, renderer, device driver, or other higher-level service. Its extensive dependence on incoming machine state prevents treating the explicit argument as a complete interface.

This contract distinguishes:

- **Established static evidence:** operations and control dependencies visible in the supplied representation.
- **Inferred interpretation:** the 32-bit address and arithmetic model used to describe those operations coherently.
- **documented_only unknowns:** calling conventions, executable-target validity, architectural exception behavior, and other facts not supplied.

No runtime tests have been performed.

## Interface

### Explicit input and result

The explicit input is one unsigned 32-bit word, called the **argument** in this contract.

The declared result is void. There is no visible normal-return path: the final region repeats indefinitely unless an indirect transfer, interrupt, fault, or other external event prevents continuation.

### Implicit inputs

Behavior also depends on incoming machine state:

- Two unsigned 32-bit words, designated **comparison word A** and **comparison word B**.
- An address designated the **source address**.
- An unsigned 32-bit **adjustment word**.
- An unsigned 32-bit **accumulator word**.
- An address designated the **link-derived address**.
- The address of the incoming stack position, designated the **stack address**.
- An additional machine-state slot used by a later address calculation.

These are descriptive interface roles, not additional declared parameters. The supplied material does not establish how callers initialize them or whether an ordinary language-level call can provide the required state.

### Width and address model

The visible representation uses:

- 8-bit truncation for selected shift counts and one signed-byte address calculation.
- 16-bit memory loads and stores.
- Signed 16-bit extension for two loaded values.
- 32-bit words, pointer representations, arithmetic, and several memory stores.

This contract describes ordinary unsigned 32-bit arithmetic modulo \(2^{32}\). Some expressions are represented using signed arithmetic or shifts whose language-level validity is not established. Their exact architectural behavior remains documented_only where it matters.

A pointer typed as addressing 16-bit elements advances by two bytes per element. Consequently, a displacement of `0x75b` elements is `0xeb6` bytes; a displacement of two elements is four bytes. Explicit byte-address adjustments are not scaled.

Byte order, alignment requirements, memory mapping, and instruction-set state are unspecified.

### Preconditions for continued execution

There is no explicit validation. Continued execution requires every reached read, write, indirect transfer, and interrupt to be permitted by the actual environment.

An address need not be nonzero merely because it is used as a pointer. Conversely, a nonzero address is not shown to be valid. Invalid accesses are not converted into a documented error result.

## Behavior

### Initial address selection and possible copy

The initial **scaled address** is the argument multiplied by `0x1000`, reduced to 32 bits. A separate initial flag records whether argument bit `0x40000` is set.

If the scaled address is nonzero, the preliminary conditional operations are skipped. The later call cycle uses the incoming link-derived address as its initial base, without an intervening explicit data-memory read or write.

If the scaled address is zero, the function:

1. Forms a **working address** from the incoming link-derived address, the initial flag as zero or one, and the stack address shifted right by one.
2. Forms a second scaled address by multiplying the argument by `0x2000`.
3. Computes a copy destination as source address plus comparison word B.
4. Reads one unsigned 16-bit value from the original source address.
5. Changes the working source address to copy destination minus adjustment word.
6. Writes the captured 16-bit value to the copy destination.
7. Uses the incoming link-derived address as the next zero-test value.

The read precedes the write. Overlap is not rejected. In particular, an identical source and destination still entail a visible load and store.

Under 32-bit modular multiplication, a zero initial scaled address means that the argument’s low 20 bits are zero. The second scaled address and the initial bit flag are therefore also zero on this path.

If the incoming link-derived address is nonzero, the preliminary processing ends after this copy, and the final call cycle begins with that incoming address as its base.

### Conditional zero-result progression

The remaining preliminary operations are a single forward progression, not a search or a retry loop. Each stage is enabled only by the immediately preceding stage producing zero. Once a required zero condition fails, subsequent guarded stages are disabled, and execution proceeds toward the final cycle using whatever state has already been established.

The first progression compares address-valued expressions through exclusive-or operations. Its comparisons involve, in order:

- The working address and the current zero-valued intermediate.
- The initial scaled address and the working address.
- The initial scaled address and the second scaled address, twice.
- Comparison word B and comparison word A.
- The adjustment word and the updated source address.

The representation expresses these comparisons with right shifts whose counts come from the low byte of the preceding result. On each reached stage in this progression, that preceding result is zero. The actual shift count is therefore zero; these stages do not establish a general variable-shift interface.

The flag propagated through these zero-count shifts remains false on this initial path.

If all those comparisons succeed, the accumulator word is added to the argument’s low 20 bits multiplied by `0x800`, together with the incoming flag. Both the 32-bit result and unsigned carry are retained.

Only a zero result enables a further addition of:

- The stack address.
- Twice comparison word B, with the multiplication reduced to 32 bits.
- The preceding carry as zero or one.

That addition likewise produces a 32-bit result and a carry condition.

### Special stack-derived update

A zero result from the stack-related addition enables a distinct block:

- The additional machine-state slot receives the stack address multiplied by `0x40`.
- The link-derived address is advanced by the stack address multiplied by `0x40` bytes.
- A 16-bit value is read through the old link-derived address.
- A 16-bit store uses that loaded value as its destination address and stores the value plus 15, truncated to 16 bits.
- The address used in one later comparison is advanced by two bytes from the original scaled address.
- The propagated flag is cleared.

The store destination is the zero-extended loaded halfword, not the link-derived address itself.

The representation references the old link-derived halfword more than once in this store expression. Without instruction-level evidence or memory qualifiers, the exact number and ordering of those repeated reads are documented_only. The destination address and stored value are as described if those reads observe the same value.

### Later arithmetic and record-like writes

Further zero-result comparisons follow the special update. They compare the same broad groups of values as the first progression, but incorporate the possibly advanced address.

If those comparisons all succeed, the function enters arithmetic involving the fixed word `0x085740e8`:

- It adds that word, the working address, and the current flag.
- It left-shifts the result by its own low byte.
- It subtracts that shifted value and the complement of the incoming flag from the second scaled address.

The next condition flag is formed using explicit unsigned comparisons, including whether the shifted value is less than the second scaled address and whether their modular difference is less than the incoming flag. This must not be replaced with an assumed conventional subtraction-status rule.

Shift counts of 32 or greater are possible in this expression as represented. Their behavior is documented_only; the supplied material does not establish masking, saturation, or any other target rule.

A zero subtraction result enables subtraction of the updated source address and a flag-dependent unit from the original scaled address.

Another zero result enables two changes:

- A candidate address becomes the second scaled address plus 36 bytes.
- A further subtraction compares the original scaled address with the second scaled address shifted right by one, again incorporating a flag-dependent unit.

If that result is zero:

- Comparison word B increases by 8.
- The adjustment word increases by 10.
- A destination candidate becomes the current link-derived address plus 13 bytes.
- A 16-bit value is read from the current link-derived address.
- Its low byte is interpreted as signed, and 4 is added to obtain an address.
- A 32-bit pointer value is read from that computed address.
- A new arithmetic condition is evaluated using `0x042ba0a0`, the additional machine-state slot shifted right by one, the destination candidate, and two occurrences of the complemented incoming flag.

The signed-byte interpretation matters: the intermediate address lies in the signed range −124 through 131 before conversion to the address representation. It is not an unsigned-byte table index.

If the new condition is zero, the function performs record-like stores:

1. Reads a 16-bit value from the working address and sign-extends it to 32 bits.
2. Advances a comparison address by one byte from the working address.
3. Adjusts the eventual call base by subtracting the earlier comparison address from the destination candidate.
4. Stores the sign-extended value as a 32-bit pointer representation at the destination candidate.
5. Stores the previously obtained pointer value four bytes later.

The destination may be unaligned. The representation supplies neither an alignment check nor a recovery path.

### Final conditional stores

A final zero-result comparison sequence involves the newly stored value, the retrieved pointer, the one-byte-advanced working address, both scaled addresses, the candidate advanced by 36 bytes, and the updated comparison and adjustment words.

If every condition succeeds:

- A signed 16-bit value is read 13 bytes beyond the current call base.
- The call base advances by 27 bytes.
- The loaded value is sign-extended to 32 bits.
- That value is stored as a 32-bit integer at its own value minus `0x08574200`.
- The retrieved pointer is stored at the address four bytes later.

The stores are ordered: integer first, pointer second.

### Unbounded call and interrupt cycle

Once preliminary processing completes, the repeated cycle is:

1. Transfer control indirectly to the address equal to the current call base plus `0xeb6` bytes, supplying argument words 0, 0, and `0xffffffa5`.
2. If control returns, transfer indirectly to address `0x0857a94b`, with no explicit arguments in the supplied representation.
3. If control returns, execute software interrupt `0x5b`.
4. If execution continues, replace the call base with `0x083d4cc2` and repeat.

The first transfer is to the computed address itself, not to a function pointer loaded from memory there.

After a completed first cycle, the arithmetic target for the first transfer is `0x083d5b78`. Whether the machine treats these addresses as executable entries, encoded instruction-state addresses, or faults is unspecified.

## Side effects

Visible possible effects include:

- The initial 16-bit copy.
- A special 16-bit store through an address obtained from a halfword.
- Two ordered 32-bit record-like stores.
- Two further ordered 32-bit stores through a sign-extended halfword-derived address.
- Changes to working machine-state values and one explicitly represented additional state slot.
- Arbitrary effects of the indirect targets and software interrupt.

There is no visible allocation, release, ownership transfer, or cleanup operation. That does not establish that indirect dependencies lack such effects.

No error code, diagnostic, rollback, or normal completion indication is shown. A later fault does not undo earlier writes.

Memory may alias across all these roles. A prior store can therefore affect a later read. No separation or exclusivity guarantee is established.

## Dependencies

No independently identified dependency function is supplied. The only available opaque function ID identifies this function itself.

The unresolved external dependencies are:

| Dependency | Established interface | Unknowns |
|---|---|---|
| Computed-address indirect transfer | Three explicit argument words: 0, 0, `0xffffffa5` | Calling convention, return behavior, effects, ownership |
| Fixed-address indirect transfer | No explicit argument expressions | Actual consumed machine state, effects, return behavior |
| Software interrupt `0x5b` | Interrupt identifier | Privilege requirements, handler, effects, resumption behavior |

Their return values are not explicitly consumed. They must not be assumed to preserve globals, memory, registers, or ownership merely because those effects are not described locally.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

### Nonzero scaled address

For argument 1, the initial scaled address is `0x1000`.

Expected local behavior:

- No preliminary explicit data-memory access.
- First indirect target equals the incoming link-derived address plus `0xeb6`.
- Explicit argument words are 0, 0, and `0xffffffa5`.

This example does not require either indirect target to return.

### Zero scaled address with nonzero link-derived address

For argument 0 and a nonzero incoming link-derived address, assuming the source and destination accesses are valid:

- One 16-bit value is read from the original source.
- That value is stored at source plus comparison word B.
- The deeper zero-result progression is skipped.
- The first indirect target remains incoming link-derived address plus `0xeb6`.

If comparison word B is zero, this is a same-address read followed by write, not an omitted operation.

### Wrapped multiplication

Argument `0x00100000` produces a zero initial scaled address under 32-bit modular arithmetic. It therefore follows the copy-enabled path, not the nonzero-scaled-address path, despite the argument being nonzero.

### Null link-derived address and nonzero stack-derived working address

Assume the initial scaled address is zero, the initial copy succeeds, the incoming link-derived address is zero, and the stack address shifted right by one is nonzero.

The first zero-result comparison fails. The deeper conditional stores do not occur. The first computed transfer target is `0xeb6`; no validity guarantee follows.

### Completed first cycle

If both indirect transfers return and interrupt `0x5b` resumes execution, the next cycle uses arithmetic target `0x083d5b78` for its first transfer. The cycle is repeated, not followed by a normal return.

## Uncertainties

The following remain documented_only:

- The intended high-level operation and supported caller states.
- Exact ABI, register preservation, stack layout, and executable-address interpretation.
- Architectural treatment of oversized shifts and signed-overflow expressions.
- Exact repeated-read behavior in the halfword-derived store.
- Alignment, byte order, memory permissions, and fault semantics.
- Practical reachability of the deepest guarded blocks on the actual machine.
- Effects of the indirect dependencies and interrupt on subsequent execution.
- Whether the supplied representation accurately captures all machine-state dependencies.

The extensive zero-result gates permit strong static simplifications on ordinary paths, but do not justify deleting exceptional paths from the documented contract without target-level evidence.

## Evidence

This contract is based exclusively on the supplied static representation.

Established evidence includes the declared unsigned-word input, void result, halfword accesses, pointer-width stores, explicit arithmetic constants, conditionally ordered writes, indirect transfers, software interrupt, and unconditional final repetition.

The 32-bit modular address model is an interpretation supported by the represented word widths and carry operations. Where signed expressions, unusual shifts, or hidden machine-state effects exceed that evidence, this contract leaves the behavior documented_only.

No runtime execution, external dependency inspection, architecture validation, or dynamic reachability check has been performed.
