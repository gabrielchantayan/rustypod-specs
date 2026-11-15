<!-- Trusted provenance: F-aff94cb9675c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aff94cb9675c

## Purpose

The supplied static representation establishes a sequence of indirect calls, address calculations, byte accesses, and 32-bit word accesses. It does not establish an application-level purpose.

Some values are treated both as numeric payloads and as addresses. Byte selectors direct writes into a low-address range. Separately, **three writes target two distinct low absolute addresses**: one write to 0x11 and two writes to 0x0d. These explicit absolute writes must not be confused with computed writes that may reach the same addresses.

Interpreting this sequence as allocation, initialization, serialization, or device control would require evidence not supplied here.

This contract separates:

- **Established static evidence:** represented operations, widths, operands, and ordering.
- **Conditional interpretation:** consequences if execution follows that representation using 32-bit machine arithmetic.
- **documented_only unknowns:** execution validity, dependency behavior, ABI, memory mapping, ownership, and the true endpoint.

The representation reports invalid instruction data and ends at an invalid-data marker. It does not establish a complete, normally returning execution.

## Interface

### Explicit and implicit inputs

The displayed interface has two 32-bit explicit arguments. The first has no visible direct use. The second is the second explicit operand of F-U01-A. This does not establish that omitted machine-state dependencies make the first argument irrelevant.

Additional inherited inputs are:

- A 32-bit address, called **entry base**.
- An unsigned 32-bit value, called **entry word**.
- Control-address state used to derive the first indirect target.
- Secondary machine-result words attributed to intervening calls.
- Memory reachable through inherited addresses, dependency results, scaled values, and zero-extended bytes.

The visible body neither initializes nor validates these inherited inputs.

### Outputs and ownership

No ordinary return value is shown. Some dependencies provide a represented 32-bit result; others provide a represented result with low and high 32-bit halves. That representation does not establish their actual calling convention or a language-level 64-bit return type.

Memory writes and dependency interactions are the principal visible outputs. No ownership transfer, allocation, or release is explicitly established. Dependencies may have such effects, but none may be assumed.

### Widths and addressing

A **byte** is 8 bits; a **word** is 32 bits. Byte loads used as selectors are zero-extended. Byte stores retain the low eight bits. Low and high result halves mean bits 0–31 and 32–63 respectively.

Explicitly unsigned right shifts are logical. Numerical examples and scaled addresses below use a **conditional 32-bit bit-pattern interpretation**: multiplication, addition, and left shifts retain the low 32 bits. Several represented expressions use signed types, so this convention is not a claim of portable signed-overflow or signed-left-shift semantics.

Endianness and alignment requirements are unknown. Reading a word and storing its numeric low byte is not interchangeable with reading the byte at that word’s starting address.

### Conditions for reaching an operation

Every earlier dependency must return control, every earlier access must be permitted, and each indirect target must be executable under the actual machine rules. Required implicit machine state must also exist. No visible check enforces these conditions.

## Behavior

### Trace conventions

Descriptive names below identify retained values, not object types or source identifiers.

F-U01-A and similar identifiers designate opaque **call sites**, not proven distinct functions. A stated range executes alphabetically and includes both endpoints. Calls without listed operands have no explicit operands in the representation; their actual machine interfaces remain unknown.

A **selector write using address A** consists of a fresh byte read at A, zero-extension of that byte, and a byte write at the resulting value plus 0x11. Its destination lies in 0x11–0x110. Unless explicitly described as captured, selector reads are not reused.

The following phases preserve the order of represented calls and memory effects. Retained values describe the represented dataflow, not an independently established guarantee that opaque dependencies preserve registers or globals.

### 1. Entry fields and initial result-derived base

**Doubled entry** is twice entry word. A word read at entry base + 0x74 supplies the byte written at entry base + 1.

F-U01-A receives doubled entry, the second explicit argument, entry word shifted right by two, and entry base. F-U01-B supplies a two-half result. F-U01-C receives its low half, high half, low half again, and `(entry word & 0x7fffffff) >> 1`. F-U01-D through F-U01-G follow.

Four times F-U01-G’s result becomes **first scaled base**. The word at first scaled base + 0x74 supplies the byte written at first scaled base + 1. F-U02-A receives first scaled base; F-U02-B through F-U02-E follow. A selector write using doubled entry + 1 then stores zero.

### 2. Retained pointer and linked-word selection

F-U03-A receives an unresolved secondary machine-result word from the preceding calls. F-U03-B supplies **retained pointer**. F-U03-C receives retained pointer; F-U03-D through F-U03-G follow, producing **first index**.

A fresh selector read at doubled entry + 1 directs a byte write containing bits 24–31 of doubled entry.

F-U04-A receives four times first index; F-U04-B through F-U04-E follow. Four times the final result becomes **second scaled base**. A word read at retained pointer + 4 supplies **linked word**. A selector write using linked word + 1 stores zero.

F-U05-A receives second scaled base three times. After F-U05-B through F-U05-E produce **second index**, a byte read at `(linked word >> 2) + 1` supplies **small base**. Zero is written at small base + 0x11.

### 3. Small-base field and packed-word reduction

F-U06-A receives four times second index. F-U06-B through F-U06-E produce **third index**.

A word read at second scaled base + 4 supplies **packed word**. Next, a word read at small base + 0x74 supplies **field word**, whose low byte is written at small base + 1.

F-U07-A receives field word, second scaled base shifted left by 19 twice, and four times third index. F-U07-B supplies a word result. Packed word is logically shifted right by 22 to form **reduced packed word**. F-U07-C receives the F-U07-B result and field word shifted right by two. F-U07-D through F-U07-G follow.

### 4. Copied word and packed payload

Four times F-U07-G’s result becomes **third scaled base**. A word read at third scaled base + 0x74 supplies **copied word**, whose low byte is written at third scaled base + 1.

F-U08-A receives third scaled base; F-U08-B through F-U08-E follow. Copied word is then written as a word at four times an unresolved secondary result plus 0x14. F-U09-A receives that same secondary result.

F-U09-B supplies a word. F-U09-C receives that word and two copies of `(reduced packed word & 0x1ff) << 21`. F-U09-D through F-U09-G produce **packed destination index**. A word containing reduced packed word shifted left by 23 is written at four times packed destination index + 0x14.

### 5. Masked index and destination construction

F-U10-A receives four times packed destination index. F-U10-B through F-U10-E produce **raw index**.

**Saved scaled word** is four times raw index. It is written as a word at sixteen times raw index + 0x14. **Masked index** is raw index AND 0x3fffffff; **quarter index** is masked index shifted right by two.

F-U11-A receives masked index. F-U11-B through F-U11-E produce **destination index**. Masked index is written as a word at four times destination index + 0x14.

F-U12-A receives four times destination index. F-U12-B through F-U12-E produce **raw link**.

### 6. Two writes to the +0x34 field

**Eightfold destination** is eight times destination index. Its low byte is written at field word + 1. Four times raw link is then written as a word at eightfold destination + 0x34.

F-U13-A receives four times raw link. F-U13-B supplies a word result. **Masked link** is raw link AND 0x3fffffff. F-U13-C receives the F-U13-B result; F-U13-D through F-U13-G produce a two-half result.

The low byte of eightfold destination is written at masked link + 1. The high half of the two-half result is written as a word at eightfold destination + 0x34. This is a second write to the same calculated address, not proof that no intervening effect changed that location.

F-U14-A receives four times the low half. F-U14-B through F-U14-E produce **later index**.

### 7. Sixteenfold destination and the explicit 0x11 write

**Sixteenfold destination** is sixteen times destination index. Its low byte is written at `(masked link >> 20) + 0x11`. The low byte of saved scaled word is then written at sixteenfold destination + 0x0d.

F-U15-A receives four times later index twice. F-U15-B supplies **shift source**. F-U15-C receives shift source and its logical right shift by three. F-U15-D through F-U15-G follow.

The low byte of sixteenfold destination is explicitly written at absolute address **0x11**. The low byte of saved scaled word is again written at sixteenfold destination + 0x0d.

F-U16-A receives four times F-U15-G’s result. F-U16-B through F-U16-E produce **layout seed**. Saved scaled word is written as a word at masked link + 0x14. The low byte of twice masked link is written at masked link + 0x0d.

### 8. Layout fields and selector interaction

**Layout base** is layout seed multiplied by 32. **Layout payload** is layout seed shifted left by seven. **Small selector** is `(layout seed & 0x07ffffff) >> 17`.

F-U17-A through F-U17-C produce **layout index**. Layout payload is written as a word at layout base + 0x14. The low byte of four times layout index is written at layout base + 0x0d.

F-U18-A receives four times layout index; F-U18-B through F-U18-E follow. The low byte of twice quarter index is written at small selector + 7. A selector write using layout base + 1 then stores the low byte of small selector.

F-U19-A receives layout index shifted left by 21 three times. F-U19-B supplies **saved word**. F-U19-C receives saved word. F-U19-D through F-U19-G produce **next index**.

A fresh byte read at layout base + 1 supplies **captured byte**. Zero is written at captured byte + 0x11.

### 9. The two explicit 0x0d writes

F-U20-A receives four times next index; F-U20-B through F-U20-E follow.

**Byte-derived base** is twice captured byte. **Saved byte** is the low eight bits of saved word logically shifted right by three. Saved byte is written at layout base + 0x0d. Captured byte is then explicitly written at absolute address **0x0d**.

F-U21-A receives byte-derived base, 0x98, and 0x260. F-U21-B follows. F-U21-C receives an unresolved unsigned secondary machine-result word logically shifted right by three. F-U21-D through F-U21-G follow.

F-U22-A receives four times F-U21-G’s result and supplies another word result. Saved byte is written at byte-derived base + 0x0d. The byte **0xc0** is then explicitly written at absolute address **0x0d**.

F-U22-B receives four times F-U22-A’s result. F-U22-C through F-U22-F produce **payload index**.

### 10. Nested payload and saved-word propagation

Four times payload index is written as a word at byte-derived base + 0x24. A word read at byte-derived base + 0x74 supplies **nested base**. **Expanded payload** is payload index shifted left by five.

F-U23-A receives nested base, twice nested base, expanded payload, and `(nested base & 0x7fffffff) >> 2`. F-U23-B through F-U23-E follow. Expanded payload is written as a word at nested base + 0x24.

A word read at nested base + 0x74 is passed to F-U24-A. F-U24-B through F-U24-E produce a two-half result. **Propagated word** is four times its low half. Saved word logically shifted right by three is written as a word at four times its high half + 0x14.

F-U25-A receives propagated word three times and that high half logically shifted right by three.

### 11. Further propagation and independently selected byte writes

F-U25-B supplies another two-half result. F-U25-C receives its low half, high half, low half again, and high half logically shifted right by three. F-U25-D through F-U25-G produce **propagation index**.

Propagated word is written as a word at four times propagation index + 0x14. F-U26-A receives four times propagation index. F-U26-B through F-U26-E produce **final seed**.

**Final scaled word** is four times final seed. A selector write using nested base + 1 stores its low byte.

F-U27-A through F-U27-C follow. A fresh selector read at nested base + 1 directs a byte write containing the low eight bits of `(final seed & 0x3fffffff) >> 20`. The two selector writes need not have the same destination.

### 12. Repeated zeroing and represented endpoint

F-U28-A receives four times F-U27-C’s result; F-U28-B through F-U28-E follow.

A selector write using final scaled word multiplied by 0x400, plus 1, stores zero. F-U29-A receives final scaled word shifted left by three twice, final scaled word, and four times F-U28-E’s result. F-U29-B through F-U29-E follow.

The same calculated selector address is read again, and another selector-directed zero write occurs. Its destination may differ from the earlier destination.

F-U30-A receives four times F-U29-E’s result. F-U30-B through F-U30-E follow. The representation then terminates at its invalid-data marker; no normal return is shown.

## Side effects

The established effects include byte and word reads, byte and word writes, and opaque indirect calls interleaved with those accesses.

The recurring offsets do not establish a structure layout. Observable access widths include:

- Word reads at +4 and +0x74.
- Byte reads at +1 and at other selector addresses.
- Byte writes at +1, +7, +0x0d, and selector-derived addresses.
- Word writes at +0x14, +0x24, and +0x34.

The three explicit absolute-address writes occur in this order:

1. At 0x11: low byte of sixteenfold destination.
2. At 0x0d: captured byte.
3. At 0x0d: 0xc0.

Other computed writes may alias these locations. The list is not a claim that these are the only effects at those addresses.

No non-aliasing guarantee exists. A write may alter a later selector, pointer, payload, or dependency-visible location. Dependencies may also change memory between accesses. Repeated stores and selector reads therefore cannot be collapsed on the evidence supplied.

There is no visible bounds check, null check, error-status interpretation, allocation, release, rollback, or ownership handoff. If an access faults or a dependency does not return, the representation provides no recovery path and no guarantee that earlier effects are undone. It does not specify the exact fault mechanism or partial-access behavior.

## Dependencies

The F-U identifiers used above are opaque call-site IDs. Their grouping is organizational only. Different IDs may resolve to the same target; target validity is not established.

The first target depends on inherited control state. Later targets use represented fixed control-location arithmetic. No trustworthy application-level dependency identity follows from that arithmetic.

For every dependency, the following remain unresolved unless a particular operand or result is explicitly described:

- Full argument list and calling convention.
- Implicit register inputs, secondary outputs, and clobbers.
- Memory or global-state effects.
- Ownership acquisition, transfer, or release.
- Faulting, exception, termination, and non-return behavior.
- Agreement between the represented result width and the actual ABI.

Calls with unused results still occur. Calls with no displayed operands must not be treated as pure or genuinely argument-free.

## Acceptance examples

These are conditional static acceptance criteria, not observed executions. Each assumes the required preceding calls return and accesses succeed.

1. **Numeric low-byte copy.** If the entry word read at entry base + 0x74 yields 0x123456ab, the following byte write at entry base + 1 stores 0xab, independent of memory endianness.

2. **Entry arithmetic boundary.** Under 32-bit bit-pattern arithmetic, entry word 0x80000000 produces doubled entry zero. F-U01-A’s shifted entry operand is 0x20000000; F-U01-C’s masked-and-shifted entry operand is zero.

3. **Selector destination.** A selector byte of 0xfe directs its write to 0x10f, not to the selector’s storage address plus 0x11.

4. **Fresh selector observation.** If doubled entry + 1 initially contains 3, the first selector-directed zero targets 0x14. If dependencies change that byte to 9 before the later read, the later entry-derived byte write targets 0x1a.

5. **Packed payload truncation.** Packed word 0xffffffff reduces to 0x3ff. The masked, shifted F-U09-C operands are 0x3fe00000. The later word payload is 0xff800000 under 32-bit truncation.

6. **Three explicit absolute writes.** If sixteenfold destination has low byte 0x70 and captured byte is 0x2a, the explicit absolute-write sequence is 0x70 at 0x11, then 0x2a at 0x0d, then 0xc0 at 0x0d. Dependencies and computed aliases may add other effects between or after these writes.

7. **Captured value survives a memory change in the represented dataflow.** Once captured byte is read, byte-derived base uses twice that captured value. A later change to layout base + 1 does not itself constitute another read for that calculation.

8. **No success-return criterion.** Completing the final represented call sequence does not establish an ordinary successful return. The supplied endpoint remains the invalid-data marker.

## Uncertainties

- **documented_only — execution validity:** invalid-instruction warnings leave unresolved whether the represented sequence executes as shown.
- **documented_only — endpoint:** the final marker may reflect a representation boundary rather than an intentional runtime trap.
- **documented_only — ABI and retained state:** inherited machine inputs, secondary results, register preservation, and actual result conventions are incomplete.
- **documented_only — arithmetic:** conditional wrapping arithmetic does not settle signed-expression behavior or the actual address model.
- **documented_only — memory:** endianness, alignment, permissions, mappings, concurrency, and memory-mapped I/O semantics are unknown.
- **documented_only — dependencies and ownership:** no purity, global-preservation, ownership, release, or return guarantee is established.
- **Inferred intent only:** repeated offsets suggest field-like access, but no object schema or application purpose is established.

## Evidence

This contract uses only the supplied static representation: explicit input widths, inherited-state references, dependency operands and results, memory-access widths, address offsets, masks, shifts, repeated reads, and endpoint warnings.

The absolute-write count follows from three separately represented byte stores: one at 0x11 and two at 0x0d. Selector-derived writes are accounted for separately.

Descriptive names and opaque dependency IDs organize the contract; they do not recover application interfaces or dependency identities.

No runtime tests have been performed. Executability, successful completion, actual dependency behavior, and application-level intent have not been independently verified.
