<!-- Trusted provenance: F-3eaee7c81cff; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3eaee7c81cff

## Purpose

The supplied static representation describes a procedure that performs two indirect calls, computes addresses from incoming values and call results, writes memory at those addresses, and updates global storage. The representation ends at an invalid-instruction boundary without an ordinary return.

No application-level purpose is established. The operations do not establish allocation, object construction, interrupt handling, memory management, or a particular data structure. Self-address stores and nearby offsets are observable features, not sufficient evidence of intent.

This contract describes the represented behavior and its limits. It does not establish that the complete sequence is executable. The supplied material reports invalid instruction data and overlapping global symbols. It also contains a right shift by the apparent operand width, leaving that calculation unresolved.

## Interface

### Explicit inputs

The represented interface has five positional inputs:

| Input | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use in the displayed body. |
| Second | 32 bits | No explicit use in the displayed body. |
| Third | Signed 32 bits | Addition, scaled addresses, an address offset, and a truncated store. |
| Fourth | 32 bits | No explicit use in the displayed body. |
| Fifth | Signed 32 bits | Multiplication by sixteen to obtain a read address. |

The apparent lack of use of three inputs does not prove their irrelevance to actual execution. The recovered interface may omit calling-convention state or implicit call arguments.

### Additional incoming state

Three values are consumed without a displayed definition:

- **State A:** unsigned 32-bit state used in logical shifts, signed reinterpretations, address calculations, multiplication, and truncation.
- **State B:** signed 32-bit state shifted left by two for the first indirect call.
- **State C:** signed 32-bit state added to the third explicit input.

These labels describe incoming state; they do not establish additional formal parameters, register assignments, or initialization mechanisms.

### Global-state labels

The following stable labels identify global roles without asserting distinct storage:

| Label | Role |
|---|---|
| **G-input** | Source of a value read late in the procedure and used in address arithmetic. |
| **G-derived** | Receives a scaled address, then is explicitly read to obtain write destinations and a stored value. |
| **G-half** | Receives a 16-bit extraction from G-input and is read again near the end. |
| **G-zero** | Receives zero after G-half is assigned. |
| **G-concat** | Destination of the final 32-bit concatenation of a read from G-half and the constant `0xEA70`. |

G-half and G-concat are different symbolic roles. Their labels do not establish that their storage is disjoint. The same qualification applies to all global labels and numeric global destinations.

### Outputs, ownership, and validity

No ordinary return value or successful return path is shown.

The first indirect call supplies a pointer-like result. The second supplies a represented 64-bit result whose low and high 32-bit portions are used separately. Neither result establishes ownership, allocation, lifetime, or permission to access the resulting addresses.

There are no represented null, bounds, alignment, allocation-failure, or arithmetic-error checks. Reaching an access requires its entire width to be valid under the target’s memory rules. Some addresses may be unaligned.

The terminal marker establishes where the supplied representation stops, not whether actual execution traps, continues through unrecognized instructions, or behaves otherwise.

## Behavior

### Interpretation rules

The sequence below preserves the displayed order of operations. It does not establish physical bus ordering, instruction-level atomicity, or behavior omitted by incomplete recovery.

Explicit byte, halfword, and word accesses are respectively 8, 16, and 32 bits. Truncation retains low-order bits. Endianness is not established.

Pointer conversions and word-sized pointer stores indicate an apparent 32-bit address model. This is static evidence, not an independently verified platform specification.

Unsigned right shifts are logical. Signed right shifts, signed overflow, and signed address arithmetic are not assigned portable semantics. In particular, a signed 32-bit value shifted right by thirty-two has no established ordinary result in the supplied representation.

Values already obtained by reads are distinguished from later reads of the same global or address. Possible aliasing must not be eliminated by treating global storage as cached local state.

### 1. First call and result-dependent writes

The procedure first obtains **A-high** by logically shifting State A right by eighteen bits. A-high ranges from zero through 16,383.

It then invokes **F-unresolved-1**, passing the represented result of State B shifted left by two.

Let **P** denote the pointer-like result. The following operations are displayed:

1. Store P, as an apparently 32-bit pointer value, at an address obtained by doubling P’s signed 32-bit address representation.
2. Store State A reinterpreted as signed 32-bit and shifted right by two as a word at P.
3. Calculate another destination from the address four bytes beyond P: convert that address to signed 32-bit, shift it right by thirty-two, then add sixty-four.
4. At that destination, perform these writes in order:
   - A byte containing the destination’s low eight address bits.
   - A halfword containing State A’s low sixteen bits.
   - A word containing A-high.

The destination in step 3 and operations dependent on it are **documented_only** because of the full-width shift. No processor-specific masking of the shift count is assumed.

The three writes in step 4 share a starting address. If they complete against ordinary writable memory, the final word replaces the bytes affected by the earlier two writes. Those earlier accesses remain relevant to faults, devices, volatile storage, and other observers.

### 2. Sum, second call, and initial scaled-address writes

Next, the procedure forms the represented signed sum of State C and the third explicit input. Call this value **S**.

It writes S as a word at the address obtained by reinterpreting State A as signed 32-bit and shifting it right by two.

It then invokes **F-unresolved-2**, with no displayed explicit arguments. Let **L** and **H** be the low and high 32-bit portions of its represented 64-bit result.

The subsequent operations are:

1. Write L’s low sixteen bits at the address represented by L.
2. Form **B** by multiplying State A by sixty-four in the represented unsigned 32-bit domain.
3. Store B as a word at address B.
4. Read an unsigned word at the address obtained by multiplying the fifth explicit input by sixteen. Call this read value **W**.
5. Store a pointer-like word representing four bytes beyond that read address at B plus sixteen.
6. Store A-high’s low byte at B plus twenty.
7. Store the low sixteen bits of the pointer-like value from step 5 at State A plus four.
8. Store B as a word at A-high plus sixty.

The read producing W occurs after both calls and the preceding writes. It is not an entry-time memory snapshot. Earlier accesses or either dependency may have changed its source.

### 3. Writes and a read derived from W

The procedure logically shifts W right by three to obtain **Q**. Q ranges from zero through 536,870,911 and is used as an address.

The sequence is:

1. Store W as a word at Q.
2. Store W as a word at Q plus twenty.
3. Store Q as a pointer-like word at Q plus the bitwise conjunction of H and Q.
4. Read a word at the address obtained by doubling Q’s signed address representation. Call the result **V**.
5. Store V’s low sixteen bits at Q.
6. Store V as a word at the address represented by V.
7. Store a zero word at thirty-two times the third explicit input, plus twenty-four.
8. Store the third explicit input’s low sixteen bits at the address four bytes beyond that input’s numerical value.
9. Assign five to the global identified by address `0x00000005`.
10. Store Q as a pointer-like word at four times Q’s signed address representation, plus twenty-four.

The bitwise conjunction in step 3 is not evidence of bounds checking, alignment, or containment within an object.

V is read after the three initial W/Q-dependent writes. Any overlap with those writes affects the value obtained. It must not be replaced by a hypothetical value from before the sequence.

### 4. Global reads and dependency-address-derived writes

After the preceding operations, the procedure reads G-input. Let **G** denote the resulting represented signed 32-bit value.

Let **E** denote the numeric entry address associated with **F-8afdaab91693**. That dependency is used as address-valued data here; it is not invoked.

The displayed operations are:

1. Assign four times Q’s signed address representation to G-derived.
2. Perform an indirect word store that reads G-derived for its destination and reads G-derived for its stored value.
3. Read G-derived again to obtain a destination base, then store S as a word four bytes beyond that newly read base.
4. Store four times G as a word at the address represented by four times G.
5. Store four times E as a word at the address obtained by adding G to the signed right-shift-by-seventeen result of G shifted left by six.
6. Assign G’s upper sixteen bits to G-half, using an unsigned right shift for extraction.
7. Assign zero to G-zero.
8. Form an address from eight times E.
9. Store a zero word at that address.
10. Store another zero word four bytes beyond it.
11. Store one byte sixteen bytes beyond that base. The stored value is the low byte of the base address plus eight, truncated to eight bits.

Step 2 preserves both represented reads of G-derived: one supplies the destination and one supplies the data. Their relative evaluation order within that assignment is not established.

Step 3 contains a later read of G-derived. It does not reuse the destination captured for step 2. If the indirect store in step 2 changes storage associated with G-derived, that change can affect the destination used for S.

The two zero-word writes precede the byte write. The byte destination is base plus sixteen; the addition of eight contributes to the byte value, not the destination offset.

Signed multiplication, left shifting, right shifting, and addition in this phase retain their represented arithmetic uncertainties.

### 5. Final global assignments

The last displayed assignments occur in this order:

1. Assign `0x897EA700` to the global identified by address `0x00000086`.
2. Assign `0x66` to the global identified by address `0xEA700000`.
3. Assign `0xEAEC` to the global identified by address `0x0897EB26`.
4. Assign `0x1A` to the global identified by address `0x12FD4C34`.
5. Assign an address-valued reference to location `0x897EA65C` to the global identified by that same address.
6. Read G-half and assign a 32-bit concatenation to **G-concat**: the read halfword occupies bits 31–16, and `0xEA70` occupies bits 15–0.
7. Assign `0x5C` to the global identified by address `0x12FD4CB8`.

The source of the high halfword in step 6 is the read of G-half at that point. It is not necessarily the halfword previously calculated from G: intervening writes may overlap its storage.

G-concat is the destination of this full 32-bit assignment, not the label for the halfword source. Their symbolic distinction provides no non-aliasing guarantee.

The representation then ends at the invalid-instruction boundary.

## Side effects

The represented side effects comprise two indirect calls, computed-address memory writes, explicit memory reads, and global-state accesses.

There is no represented conditional branch that skips an access because an input is zero, negative, unaligned, or outside an expected range. There is also no displayed recovery, rollback, transaction, or error return.

Consequences include:

- Earlier effects may remain if a later access fails.
- A computed write may overlap a later read source, another destination, global storage, or executable memory.
- Global overlap can change later reads, including the repeated reads of G-derived and the final read of G-half.
- Repeated or overlapping writes cannot be collapsed solely because their eventual ordinary-memory contents appear equivalent.
- No locking, atomicity, memory barriers, or concurrency guarantees are established.
- Neither indirect call is known to preserve globals, memory, or implicit machine state.
- No ownership transfer or release operation is established.

Explicit access widths are preserved where supplied. Assignment constants alone do not establish the widths of incompletely declared numeric globals. G-half is used as a 16-bit value, and the G-concat assignment is explicitly constructed as 32 bits; other uncertain global widths remain **documented_only**.

## Dependencies

| Opaque ID | Established interaction | Unresolved properties |
|---|---|---|
| **F-unresolved-1** | First indirect call; receives the represented State B shift result and supplies P. | Identity, actual signature, implicit arguments, failure representation, ownership, side effects, preservation rules. |
| **F-unresolved-2** | Second indirect call; no explicit arguments displayed; supplies a represented 64-bit result used as L and H. | Identity, true return convention, implicit inputs, side effects, ownership, preservation rules. |
| **F-8afdaab91693** | Entry address used as numeric data for scaling, stored values, and write destinations. No invocation shown. | Whether these recovered address operations correspond to valid execution and writable memory. |

The unresolved IDs are local opaque dependency labels, not identifications of known functions.

## Acceptance examples

These are conditional consequences of the static representation, not runtime tests. Each requires execution to reach the relevant operation and all necessary preceding accesses and calls to permit progress.

1. **Unsigned extraction.** With State A equal to `0xFFFFFFFF`, A-high is `0x00003FFF`. The byte store derived from A-high writes `0xFF`.

2. **Signed and unsigned operations differ.** With State A equal to `0x80000000`, A-high is `0x00002000`. If signed right shift is arithmetic on the target, the separate shift by two yields `0xE0000000`. These calculations are not interchangeable.

3. **Full-width shift remains unresolved.** The first derived destination cannot be accepted as sixty-four merely by assuming a right shift by thirty-two produces zero. Its outcome remains documented_only.

4. **Overlapping writes retain order.** If that uncertain destination resolves to ordinary valid memory, the byte write precedes the halfword write, which precedes the word write. Final ordinary memory contains the word representation of A-high; an observer may still distinguish all three accesses.

5. **Zero is not guarded.** If W is zero, Q is zero. The sequence attempts word writes at zero and twenty. This does not establish that either access succeeds.

6. **The second call’s high portion affects addressing.** If H is zero, the conjunction-derived store targets Q. If H is `0xFFFFFFFF`, it targets twice Q. No bounds or validity guarantee follows.

7. **V reflects prior writes.** If the read producing V overlaps a preceding write, its result is determined by memory at that later read, including applicable effects of those writes.

8. **G-derived is read again.** If the indirect store using G-derived alters the value subsequently read from G-derived, the S store uses that later value plus four. Reusing the earlier destination would not preserve the represented behavior.

9. **Final byte location and value differ.** If the eight-times-E base has low byte `0xFC`, the final byte value is `0x04`. Its destination is base plus sixteen, not base plus eight.

10. **Concatenation has a named destination and a late source.** If the final read of G-half yields `0x1234`, G-concat receives `0x1234EA70`, irrespective of the earlier extracted halfword. This does not imply that G-half and G-concat occupy disjoint storage.

11. **No guaranteed completion.** If an earlier required access fails, acceptance must not require later globals to have been assigned.

## Uncertainties

### Established static evidence

The supplied representation supports the displayed access sequence, explicit access widths, arithmetic expressions, constants, two indirect-call sites, additional incoming state, repeated global reads, address-valued dependency use, and terminal recovery boundary.

### Inferred intent

Self-address stores and field-like offsets may suggest initialization of runtime structures. This is insufficient to establish object layouts, invariants, allocation, ownership, or lifecycle.

### Documented_only

Unresolved matters include:

- Correct decoding, function boundaries, and reachability.
- Completeness of the represented interface and calling convention.
- The full-width right shift and dependent accesses.
- Signed overflow, signed-shift semantics, and address wrapping.
- Exact widths and overlap relationships of incompletely declared globals.
- Endianness, alignment requirements, permissions, and memory mapping.
- Relative evaluation order of the two G-derived reads within one assignment.
- Dependency side effects, implicit inputs, and ownership.
- Actual terminal behavior or continuation beyond the supplied boundary.

## Evidence

This contract is based solely on the supplied static representation. Its warnings report invalid instruction data and overlapping globals; neither warning independently establishes the underlying machine behavior.

The supplied mapping identifies F-8afdaab91693 as the dependency whose address is used as data. Neither indirect call is resolved.

No implementation access, runtime tests, or execution traces were used. The acceptance examples are conditional static deductions, not observed executions.
