<!-- Trusted provenance: F-f5a1b900dc0c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f5a1b900dc0c

## Purpose

F-f5a1b900dc0c performs ordered memory accesses, invokes indirect dependencies, and transfers into a recurring processing path. The supplied static representation contains no normal return or explicit result.

An application-level purpose is not established. In particular, reset-associated storage does not establish that the operation implements a hardware reset.

This contract distinguishes:

- **Established static evidence:** explicit access sizes, arithmetic types, constants, data dependencies, and represented control flow.
- **Inferred intent:** application-level interpretations, none of which are required here.
- **documented_only:** behavior that the supplied representation does not resolve.

Every described effect is conditional on execution reaching it. Memory validity, returning dependencies, and successful continuation are not guaranteed.

## Interface

### Positional inputs

The represented interface has nine 32-bit positional inputs.

| Position | Visible use |
|---|---|
| 1, 3, 5, 6, 8 | No visible read. |
| 2, 4 | Combined by 32-bit exclusive-or, converted to signed 32-bit form, and shifted right by eight for F-U01. |
| 7 | Locally replaced with zero without reading its incoming value. |
| 9 | Locally replaced by a zero-extended byte read without reading its incoming value. The replacement has no subsequent visible use. |

Local replacement does not establish modification of caller-owned argument storage.

### Additional effective inputs

The operation also consumes inherited or register-derived state:

- A base address and 16-bit value for the first store.
- An initial 32-bit setup value.
- A descending cursor and a separate comparison address.
- A signed 32-bit accumulator increment.
- An inherited callable target.
- Values obtained from fixed storage.

The cursor is installed from register-derived state after F-U01 returns. It is not established as an entry-time snapshot preserved across that call.

Calling convention, register preservation, address validity, and alignment requirements are **documented_only**.

### Results and numeric interpretation

There is no visible status or normal return value. Observable results consist of reads, writes, calls, state transitions, and a conditional software breakpoint with immediate value 4.

Represented 64-bit dependency results are divided into low and high 32-bit halves. Pointer uses consume the represented 32-bit address portion. Explicit byte, halfword, word, and doubleword accesses cover 8, 16, 32, and 64 bits respectively.

Unsigned 32-bit arithmetic wraps modulo \(2^{32}\). That rule must not be applied indiscriminately:

- F-U04’s low result is converted to signed 32-bit form **before multiplication by 32**. Conversion to the unsigned address follows multiplication. Signed overflow is unresolved.
- The working word is unsigned, so multiplication by 256 **does** wrap as unsigned 32-bit arithmetic. Assignment of that result to a signed temporary is a separate, potentially unresolved conversion.
- Signed additions, signed multiplications, signed left shifts, conversions outside the signed range, and out-of-range shift counts require an established execution model.

Access widths of undeclared fixed objects cannot be established from their names or assigned constants alone.

## Behavior

### Initial accesses

The first effect is a 16-bit store of the inherited halfword at the inherited base plus two bytes.

The operation then captures a fixed-storage value as a signed 32-bit setup source and invokes F-U01 with the shifted input combination. The capture precedes the call; subsequent calculations use that captured value rather than a visible reread.

If F-U01 returns, execution proceeds in this order:

1. Read one byte at the captured value multiplied by 32.
2. Store the captured value multiplied by 16 as a 32-bit word at that same address.
3. Assign zero to the fixed object associated with `0x00000009`.
4. Assign a value explicitly converted to 16 bits, zero, to the fixed object associated with `0x00000030`.
5. Store the captured value’s low 16 bits at address `0x0000000e`.

The two multiplications use signed arithmetic; overflowing cases are unresolved. The first byte read remains an observable access even though its locally assigned result is not subsequently used. The read and following word store overlap.

The fixed-object assignments in steps 3 and 4 do not independently establish their physical access widths.

### Setup region

Two setup quantities initially equal zero. On the first traversal, the setup store value is the inherited 32-bit value.

Each traversal performs:

1. A 32-bit store of the current setup store value four bytes before the cursor.
2. A 32-bit self-address store at eight times the current setup quantity. Initially this writes zero at address zero.
3. A zero word store at a destination formed by shifting an unsigned setup quantity left by 17, converting it to signed form, shifting right by **32**, and adding 16. The destination is **documented_only**, not established as address 16.
4. A halfword store of `0xc470` at an unsigned address formed from:
   - the setup store value logically shifted right by four;
   - minus sixteen times the second setup quantity;
   - minus one if bit 3 of the setup store value is zero;
   - plus `0x26`.
5. Assignment of `0x8000` to the fixed object associated with `0xfe238028`.
6. Replacement of the first setup quantity with `0x0897f228`.
7. Assignment of the self-address value associated with `0x225fc470` to that fixed object.
8. Assignment of `0x225f` to the fixed object associated with `0x0000229f`.
9. A read of the fixed object associated with `0x00000022`, a right shift by 16, and a 32-bit store of the result 84 bytes beyond the cursor.

The undeclared fixed objects retain unresolved access widths and, where relevant, signedness.

The comparison address is then compared with the cursor plus 20 bytes. On inequality, F-U02 is invoked through the current setup call-base plus `0x2e`. That base initially equals `0x0897f0c3`; after selecting the target, but before invoking it, the operation replaces the base with `0x0897f193`.

### Transition into recurring processing

The next region establishes:

- A 64-bit pair with low half zero and high half `0x0897f1f8`.
- A signed accumulator initially zero.
- Displacement `0x0897f22d`.
- A state value of `0x0897f1f8`.

It then:

1. Stores the displacement as a word at the pair’s low-half address plus 12.
2. Interprets that low half as signed and shifts it right by a count taken from its low eight bits.
3. Records whether subtracting one from the shifted signed value would overflow.
4. Reads a word at twice the shifted value.
5. Stores the read word’s low 16 bits into storage associated with F-422afe36d401.
6. Stores word `0x0897f1f8` at the read word’s address value plus four.
7. Replaces the cursor with its previously calculated predecessor, four bytes earlier.

F-422afe36d401 is associated with a store, not an invocation. The setup store value is also replaced with `0x0897f1ec` before any represented repeat of the setup region.

An overflow indication transfers back to setup. On the first literal traversal, however, the shifted value is zero and subtracting one does not overflow. Executability of the repeat edge is not established.

On the non-overflow path, the operation writes byte `0xd8` through the pair’s high half, installs working word 8, and selects F-U03 through the current setup call-base plus `0xe56`. It replaces that base with `0x0897f1d7` before calling F-U03 with zero.

After F-U03 returns:

1. Add the inherited increment to the signed accumulator.
2. Read the fixed object associated with `0x0897f216`, convert its value to unsigned 32-bit form, then use that value as the replacement cursor address.
3. Read a byte at F-U03’s low result plus `0x0897f1f8`; retain it as the initial byte-source value.
4. Read an explicit 16-bit address at F-U03’s low result plus the displacement.
5. Zero-extend that address and store the accumulator through it as a word.
6. Enter recurring processing with argument zero, state value `0x0897f1f8`, and working word 8.

Step 2 contains **no explicit 16-bit load or 16-bit conversion**. The fixed object’s access width and any extension before conversion to unsigned 32-bit form remain unresolved.

Steps 4–6 are guarded by equality between zero and the recorded overflow indicator. That equality holds on the represented path reaching F-U03. This is a dataflow observation, not independent evidence about physical flag preservation across a call.

### Recurring entry and smaller-address branch

Each traversal first retains the low byte of the current byte-source value. The byte-source value and working word are distinct state roles.

The operation sets its recurring call-base to `0x0897edf1` and calls F-U04 with the current argument. It saves F-U04’s high 32-bit result.

F-U04’s low result is converted to signed 32-bit form and multiplied by 32 using signed arithmetic. The product is then assigned to an unsigned 32-bit address value. Out-of-range signed conversion and signed multiplication overflow are **documented_only**; unsigned assignment afterward does not make the multiplication unsigned.

The operation writes that address into itself as a word, then compares the unsigned address with 20.

If the address is less than 20:

- The larger transformation is skipped.
- F-U07’s argument is zero.
- The call-base remains `0x0897edf1`.
- No represented local update changes the state value, accumulator, or working word on this branch.

The self-address store occurs even when the address is zero.

### Larger-address transformation

Otherwise, the operation:

1. Stores the state value as a word at the computed address plus 88.
2. Stores the cursor’s low halfword at the state value plus 14.
3. Replaces the unsigned working word with the state value shifted left by two.
4. Reads reset-associated high-byte storage and assigns a 16-bit combined value: the freshly read byte is the high byte; the retained byte is the low byte.
5. Calls F-U05 with the cursor logically shifted right by two.
6. Installs call-base `0x0897ee1d`.
7. Calls F-U06 through a previously selected code-associated anchor plus `0xffe`.

The reset-associated read occurs after F-U04 and before F-U05. It is not an entry snapshot. Neither the read object nor the combined-value destination has a separately supplied declaration establishing every physical access detail.

After F-U06 returns, the ordered memory writes are:

1. Store the retained byte at twice F-U06’s low-half address.
2. Store the low byte of F-U06’s high half at its low-half address plus eight.
3. Store F-U04’s saved high half as a word at twice the incoming accumulator.
4. Store the complete 64-bit F-U06 result at its low-half address.
5. Store three consecutive words through F-U04’s saved high-half address:
   - the working word;
   - the cursor shifted left by eight;
   - the cursor itself.

Between steps 4 and 5, the accumulator becomes four times the signed interpretation of F-U04’s high half, and the state value becomes the cursor shifted left by eight through signed arithmetic. These signed operations do not have an unconditional unsigned-wrap guarantee.

A branch quantity is derived by shifting the signed cursor left by 31, shifting the result right by **32**, converting to unsigned, and logically shifting right by 20. Its value is unresolved.

Separately, the unsigned working word is multiplied by 256 with defined 32-bit wraparound. The resulting unsigned value is assigned to a signed 32-bit temporary. Only that conversion is unresolved when the value is outside the signed range; the unsigned multiplication itself is not.

If the branch quantity is zero:

- Assign the signed temporary’s low 16 bits as a replacement reset-associated value, without preserving the earlier high byte.
- Add the unsigned working word to the accumulator’s unsigned-converted value, with unsigned wraparound, then assign the result back to the signed accumulator.
- Store that accumulator as a word through F-U06’s high-half address.
- Select argument zero and retain call-base `0x0897ee1d`.

The final assignment to signed form remains unresolved for an out-of-range result.

If the branch quantity is nonzero:

- Store it as a word at the signed temporary plus 32.
- Select call-base `0x08790d52`.
- Derive F-U07’s argument by shifting the signed temporary right by 29.
- Execute software breakpoint 4.
- If execution resumes, replace the state value with F-U06’s high half interpreted as signed and shifted right by 18.

This branch leaves the accumulator at its earlier four-times value.

### Final call and recurring-state transition

F-U07 identifies an invocation site, not necessarily one underlying function.

| Path | Target | Argument |
|---|---:|---|
| Smaller address | `0x0897f80f` | Zero |
| Larger, zero branch quantity | `0x0897f83b` | Zero |
| Larger, nonzero branch quantity | `0x08791770` | Signed temporary shifted right by 29 |

The final row requires execution to resume after the breakpoint.

After F-U07 returns, the operation:

1. Stores its low result’s low 16 bits through that low-result address.
2. Performs a word self-address store at the unsigned state value multiplied by 64.
3. Captures the word at the current cursor.
4. Stores the low 16 bits of cursor plus four at state value plus four.
5. Loads a replacement cursor from the word at F-U07’s high-half address plus 80.
6. Calls F-U08, then F-U09.
7. Copies the existing working word into the byte-source value.
8. Calls F-U10 with the captured cursor word.
9. Replaces the working word with that captured word and sets the next F-U04 argument to the captured word logically shifted right by three.
10. Repeats.

Thus the next retained byte comes from the working word copied **before F-U10**, not generally from the newly captured cursor word.

## Side effects

Memory effects include low-address accesses, fixed-object assignments, computed byte/halfword/word stores, a 64-bit store, and a halfword store associated with F-422afe36d401.

No disjointness is established. Overlapping stores may replace earlier bytes. In particular, the 64-bit F-U06 result store follows both byte stores and the accumulator-addressed word store.

No allocation, release, ownership transfer, bounds check, null check, rollback, cleanup, or status-return convention is visible. Address provenance and lifetime are unknown.

Opaque dependencies may modify globals, memory, devices, or allocation state. Their return does not establish global preservation or ownership behavior.

Faults, nonreturning calls, or breakpoint handling may prevent later effects. Represented access order does not establish hardware ordering, atomicity, synchronization, or transactional execution.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-422afe36d401 | Associated storage receives a halfword; no visible call. |
| F-U01 | Inherited target; receives the shifted input combination. |
| F-U02 | Conditional setup call through the selected base plus `0x2e`. |
| F-U03 | Setup-base-relative call with zero; low result supplies addresses. |
| F-U04 | Recurring call; both result halves are consumed. |
| F-U05 | Receives cursor logically shifted right by two. |
| F-U06 | Anchor-relative call returning a represented 64-bit pair. |
| F-U07 | Branch-selected target and argument listed above. |
| F-U08 | Called after cursor replacement; no explicit argument shown. |
| F-U09 | Called after F-U08; no explicit argument shown. |
| F-U10 | Receives the captured cursor word. |

Invocation-site labels do not establish distinct implementations. Dependency internals, failure conventions, ownership, and preservation rules remain unknown.

## Acceptance examples

These are conditional static criteria, not executed tests. They assume preceding accesses succeed, required calls return, and relevant arithmetic has established semantics.

1. **Defined recurring multiplication.** If F-U04’s low result is 1, signed multiplication produces 32, followed by unsigned conversion. A word containing 32 is stored at address 32 before the larger branch begins.

2. **Signed-overflow boundary.** If that low result is `0x04000000`, signed multiplication by 32 exceeds the signed 32-bit range. The contract does not require unsigned address `0x80000000`; that would assume unresolved signed-overflow behavior.

3. **Unsigned scaling and conversion.** If the working word is `0x01000000`, multiplication by 256 wraps to zero, and signed conversion yields zero. If it is `0x00800000`, multiplication yields unsigned `0x80000000`; conversion to the signed temporary remains unresolved.

4. **Fixed-object cursor conversion.** The post-F-U03 cursor comes from conversion of the fixed object’s value to unsigned 32-bit form. A 16-bit load, zero-extension from 16 bits, or truncation to 16 bits is not an acceptance requirement.

5. **Fresh reset-associated byte.** If the retained byte is `0x34` and the high-byte read yields `0xab`, the combined assigned value is `0xab34`, regardless of an earlier high-byte value.

6. **Replacement rather than preservation.** On the zero-quantity branch, a scaled temporary with low halfword `0x1200` supplies replacement value `0x1200`, not a combination preserving the earlier high byte.

7. **Initial working-word lag.** If the first recurring traversal takes the smaller branch and captures `0x11223344`, the next retained byte is `0x08`. The next F-U04 argument is `0x02244668`.

8. **Larger-branch lag.** If a larger traversal installs working word `0x48` and later captures `0xabcdef01`, the next retained byte is `0x48`, not `0x01`.

9. **Zero address remains observable.** A zero F-U04 low result produces a word self-address store at zero before smaller-branch selection.

10. **Captured word survives later memory mutation.** Changes by F-U08 or F-U09 to the old cursor’s storage do not alter the already captured value supplied to F-U10 in the represented dataflow.

## Uncertainties

The following remain **documented_only**:

- Function-boundary validity and recovered control flow.
- Calling convention and register-derived state.
- Undeclared fixed-object widths, signedness, and extension behavior, especially the post-F-U03 cursor source.
- Out-of-range signed conversions and signed arithmetic.
- Both shifts by 32, variable shift counts, and signed-shift execution semantics.
- The precise F-U06 anchor address.
- Endianness, alignment, atomicity, concurrency, and memory ordering.
- Whether addressed storage is ordinary memory, device storage, executable storage, or invalid.
- Alternative-path executability, breakpoint resumption, faults, and termination.
- Address ownership, provenance, aliasing, and lifetime.

## Evidence

This contract uses only the supplied static representation and opaque dependency mapping. Established facts derive from explicit casts, access types, constants, operation ordering, result-half dataflow, selected targets, and control transfers.

The arithmetic distinctions are expression-specific: signed multiplication precedes unsigned assignment at recurring entry; unsigned multiplication precedes signed assignment during scaling. The cursor-source expression supplies an unsigned conversion but no explicit halfword load.

Prior prose and feedback are not independent behavioral evidence. No runtime tests, execution traces, or external dependency inspection have been performed.
