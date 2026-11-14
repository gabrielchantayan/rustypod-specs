<!-- Trusted provenance: F-fbae3b11d768; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fbae3b11d768

## Purpose

F-fbae3b11d768 performs an ordered series of indirect calls, calculated 32-bit memory stores, unsigned byte reads, and byte stores. It ends in an unconditional cycle of calls and selector-directed zero-byte stores.

The first call occurrence within that terminal cycle has a **different calculated target on initial entry than on subsequent iterations**. The call occurrence repeats; its initial target does not.

No application-level purpose is established. The evidence does not support classifying the operation as initialization, serialization, allocation, rendering, or device control.

This is a **documented_only** behavioral contract for the supplied static representation. Warning annotations report invalid instruction data and removal of unreachable regions. Consequently, the contract distinguishes represented operations from claims about actual executable behavior.

If execution reaches the terminal cycle, all reached calls return normally, and all memory accesses succeed, there is no normal completion.

## Interface

### Explicit arguments

The represented interface accepts three arguments:

| Argument | Represented width and interpretation | Explicit use |
|---|---|---|
| First argument | Signed 32-bit integer | Supplies initial store data and contributes to two store addresses. |
| Second argument | Opaque 32-bit value | No explicit use in the represented body. |
| Third argument | Unsigned 32-bit integer | Supplies a later call argument after two successive logical right shifts by seven bits. |

Absence of an explicit use of the second argument does not establish that dependencies cannot observe associated entry-machine state.

There are no established buffer lengths, object layouts, alignment guarantees, or ownership annotations.

### Additional machine-state inputs

The representation also depends on:

- An unsigned 32-bit **entry word** obtained from entry-register state.
- An **entry return-link value**, used to calculate the first indirect target.
- A **secondary result word** observed after F-U048, separately from its primary result.

The validity and calling-convention meaning of these values are unresolved. An interface containing only the three explicit arguments would omit represented inputs.

### Results and completion

There is no explicit return value and no visible normal-return path.

Observable behavior consists of dependency invocations, direct memory accesses, and continued terminal-cycle execution. There is no represented status result, validation failure result, rollback, or exception handler.

### Widths, arithmetic, and layout

The representation uses 32-bit integer words and address quantities, 32-bit word stores, and 8-bit reads and stores. Unsigned right shifts are logical. A byte store retains only the low eight bits of its value.

Unsigned arithmetic has a 32-bit interpretation. Several multiplications, left shifts, and address additions instead involve represented signed values. Their overflow semantics are **documented_only unknowns**. Arithmetic examples involving these operations are conditional on ordinary 32-bit machine behavior; they do not establish defined signed source-language overflow.

Every address offset below is measured in bytes. In particular, adding `0x14` means adding 20 bytes. It does not establish a field layout, allocation size, or enclosing object.

Endianness and alignment requirements are not supplied.

## Behavior

### Reading conventions

Dependencies are identified by opaque, specification-local call-occurrence IDs F-U001 through F-U132. These IDs distinguish occurrences, not necessarily distinct functions or stable targets.

A **five-call group** comprises five consecutive invocations. Unless stated otherwise:

- Only the first invocation has displayed explicit arguments.
- The remaining four have no displayed explicit arguments.
- The group result is the primary 32-bit result of its fifth invocation.

“No displayed explicit arguments” does not mean that register or stack state is semantically irrelevant.

The following ordering is the represented execution order, conditional on each preceding call returning and each preceding access succeeding. A result retained across calls is retained by the representation; this does not independently establish an ABI preservation guarantee.

### 1. Initial stores and first derived word

The first direct operation stores the first argument as a 32-bit word at four times that argument plus `0x14`.

F-U001–F-U005 then execute as a five-call group. F-U001 receives the entry word logically shifted right by eight bits. Its calculated target is the entry return-link value plus `0xffe`.

After this group, a second 32-bit store writes the first argument at the first argument plus `0x14`. This is not generally the first store’s destination.

F-U006–F-U010 form a five-call group beginning with the F-U005 result shifted left by two bits. Call the group result the **first derived word**.

The operation stores four times the first derived word at sixteen times that word plus `0x14`. It then invokes:

- F-U011 with sixteen times the first derived word.
- F-U012, whose result becomes F-U013’s sole displayed argument.
- F-U014, whose result becomes F-U015’s first displayed argument; F-U015’s second is the entry word logically shifted right by eight bits.
- F-U016–F-U019 without displayed explicit arguments.

The result of F-U019 is retained. The preceding store based on the first derived word is then repeated, using the same represented address and value. Intervening dependency effects prevent treating this store as dispensable.

F-U020–F-U024 form a five-call group beginning with four times the F-U019 result. Its result is the **second derived word**.

### 2. Entry-word-related stores

Four times the second derived word is stored at sixteen times that word plus `0x14`.

The next calls are:

- F-U025 with four times the second derived word and eight times the entry word.
- F-U026, followed by F-U027 with its result.
- F-U028, followed by F-U029 with its result and the entry word logically shifted right by eight bits.
- F-U030–F-U033 without displayed explicit arguments.

Four times the second derived word is then stored at eight times the entry word plus `0x14`.

F-U034–F-U038 form a five-call group beginning with four times the F-U033 result. Four times the group result is stored at sixteen times that result plus `0x14`.

F-U039–F-U043 form another five-call group. F-U039 receives, in order:

1. Eight times the entry word.
2. Eight times the entry word again.
3. The entry word logically shifted right by eight bits.
4. The third explicit argument logically shifted right by fourteen bits.

The fourth argument reflects two successive seven-bit logical shifts, not a single seven-bit shift.

Eight times the entry word is then stored at eight times the entry word plus `0x14`.

F-U044–F-U048 form a five-call group beginning with the F-U043 result shifted left by two bits.

### 3. Secondary-result-dependent stores

The following operations use the secondary result word observed after F-U048, not its displayed primary result.

Define the **secondary-derived value** as that secondary word multiplied by `0x01000000`. The operation stores this value at the secondary word multiplied by `0x04000000`, plus `0x14`.

F-U049–F-U053 form a five-call group beginning with the secondary-derived value. That same saved value is then stored at four times the F-U053 result plus `0x14`.

F-U054–F-U058 form a five-call group beginning with four times the F-U053 result. Its result is the **third derived word**. Four times this word is stored at sixteen times it plus `0x14`.

The operation then invokes:

- F-U059 with four times the third derived word.
- F-U060, followed by F-U061 with its result.
- F-U062, followed by F-U063 with its result.
- F-U064–F-U067 without displayed explicit arguments.

Four times the third derived word is stored at four times the F-U067 result plus `0x14`.

### 4. Establishing byte-access state

F-U068–F-U072 form a five-call group beginning with four times the F-U067 result.

Define the **scaled word** as four times the F-U072 result, held as an unsigned 32-bit value. It is stored at sixteen times the F-U072 result plus `0x14`.

The next calls are:

- F-U073 with the scaled word.
- F-U074, followed by F-U075 with its result.
- F-U076, followed by F-U077 with its result.
- F-U078–F-U081 without displayed explicit arguments.

The entry word logically shifted right by sixteen bits is then stored at four times the F-U081 result plus `0x14`.

F-U082–F-U086 form a five-call group beginning with four times the F-U081 result. Define the **saved base** as sixteen times the group result. Four times that result is stored at the saved base plus `0x14`.

Two additional quantities govern subsequent byte accesses:

- **Selector address:** clear bit 31 of the scaled word, logically shift the remaining value right by seven bits, and add one.
- **Entry byte base:** the entry word logically shifted right by twenty-three bits.

F-U087–F-U091 form a five-call group. F-U087 receives the saved base twice, followed by the entry word logically shifted right by fifteen bits.

The saved base itself is then stored at the saved base plus `0x14`. This replaces the earlier represented value there, subject to intervening effects and successful accesses.

F-U092–F-U096 form a five-call group beginning with the F-U091 result shifted left by two bits. The final unsigned result is the **first byte-source word**.

### 5. Selector-directed byte writes

One unsigned byte is read from the selector address. The operation adds `0x11` to that byte’s numeric value and stores the low byte of the first byte-source word shifted left by two bits at the resulting address.

This is a byte-to-address calculation, not a pointer-sized load through the selector. Its destination ranges from `0x11` through `0x110`, inclusive.

The next invocations are:

- F-U097 with three copies of the first byte-source word shifted left by two bits.
- F-U098, followed by F-U099 with its result.
- F-U100–F-U103 without displayed explicit arguments.

The selector address is read again. At this newly selected byte value plus `0x11`, the operation stores the low byte obtained by clearing the first byte-source word’s top two bits and logically shifting right by twenty bits.

F-U104–F-U108 form a five-call group beginning with the F-U103 result shifted left by two bits. Its final unsigned result is the **second byte-source word**.

The operation then, in order:

1. Stores the low byte of the second byte-source word shifted left by two bits at the entry byte base plus `0x1c`.
2. Reads the selector address again.
3. Stores zero at the newly read byte’s numeric value plus `0x11`.

The entry byte base ranges from zero through 511, so the first destination ranges from `0x1c` through `0x21b`.

### 6. Final straight-line operations

F-U109 receives the entry byte base. F-U110 has no displayed explicit arguments.

F-U111–F-U115 form a five-call group beginning with the second byte-source word after clearing its top two bits and logically shifting right by six bits.

The selector address is read again, and zero is stored at the read byte’s numeric value plus `0x11`.

F-U116–F-U120 form a five-call group beginning with the F-U115 result shifted left by two bits.

The **initial cycle field** is obtained from the unsigned F-U120 result by clearing its top two bits and logically shifting right by twenty-five bits. Its range is zero through 31.

The **cycle selector address** is calculated from the second byte-source word shifted left by two bits, multiplied by `0x400`, then increased by one. Its arithmetic remains subject to the signed-operation qualification above. A byte is read from this address, and zero is stored at its numeric value plus `0x11`.

F-U121 receives twice the initial cycle field. F-U122 has no displayed explicit arguments; its primary result becomes the **initial cycle argument**.

### 7. Unconditional terminal cycle and target transition

Each iteration begins with the five-call group F-U123–F-U127. F-U123 receives:

1. The current cycle argument.
2. The current cycle field logically shifted right by eight bits.

**F-U123’s calculated target depends on whether this is the first iteration:**

| Iteration | Represented target base | Added displacement | Calculated target |
|---|---:|---:|---:|
| Initial entry | `0x08832789`, established before the cycle | `0x0ffe` | `0x08833787` |
| Every subsequent iteration | `0x088327c7`, established at the preceding loop tail | `0x0ffe` | `0x088337c5` |

This is a one-time transition from the initial target to the subsequent-iteration target. It is not alternation between two targets. Repetition of the F-U123 call occurrence must not be interpreted as repetition of its initial target.

The target arithmetic and transition are established by the representation. Whether either address is executable, and which callable implementation it denotes, remain unresolved.

After F-U127 returns:

1. A byte is read again from the cycle selector address.
2. Zero is stored at that byte’s numeric value plus `0x11`.
3. F-U128–F-U132 execute as a five-call group, beginning with the F-U127 result shifted left by two bits.

The F-U132 result determines the next iteration’s state:

- The next cycle argument is that result shifted left by two bits.
- The next cycle field is that result shifted left by twenty-six bits.

The loop-tail assignment also supplies the target base used by F-U123 on the next iteration. The other call occurrences repeat at their represented calculated targets; the initial-entry distinction above concerns F-U123.

There is no represented exit comparison, iteration limit, success condition, or normal return.

## Side effects

Direct effects include ordered 32-bit stores, selector-byte reads, and byte stores. No explicit read of a word-store destination precedes its write.

Each selector read is a separate access. Dependencies, earlier stores, aliasing, or external activity may change the selected byte between reads. A replacement must not cache a selector value across these accesses merely because the selector address is unchanged.

Similarly, repeated writes cannot be eliminated solely because their represented values and addresses match. Intervening dependencies may alter memory or make accesses externally observable.

No allocation, release, retention, or ownership transfer is established. Numeric arguments must not be classified as owned pointers, borrowed buffers, or handles without further evidence. Dependencies may affect memory, globals, registers, or ownership beyond the direct operations described here.

Continued execution requires reached targets to support the calls and reached addresses to support the access widths, subject to actual platform rules. There is no represented validation or recovery. Faults, traps, nonreturning calls, and other abnormal transfers have unresolved behavior.

## Dependencies

F-U001–F-U132 identify call occurrences only. The supplied mapping identifies F-fbae3b11d768 itself, not its dependencies.

Established dependency information includes displayed argument order, selected result uses, represented call order, and calculated target expressions. In particular, unresolved function identities do **not** make F-U123’s known target transition unknown.

Unresolved properties include:

- Semantic function identities and possible aliases.
- Executability and interpretation of calculated target addresses.
- Calling conventions and undisplayed arguments.
- Primary-result meaning and secondary-result validity.
- Memory, global-state, allocation, and release effects.
- Whether a call returns normally or transfers control elsewhere.

No dependency is assumed to preserve global state or own or release memory.

## Acceptance examples

These are conditional static obligations, not executed tests.

1. **Initial store addresses remain distinct.** With first argument 3 and valid accesses, the first word store writes 3 at `0x20`. After F-U001–F-U005, the second writes 3 at `0x17`.

2. **Two seven-bit shifts accumulate.** With third argument `0xffffffff`, F-U039’s fourth displayed argument is `0x0003ffff`.

3. **Selector reads remain independent.** If two successive selector reads yield 2 and 9, their corresponding destinations are `0x13` and `0x1a`. The second destination must not reuse the first selector value.

4. **Byte stores truncate.** If the first byte-source word is `0x00000041`, its shifted low-byte store writes `0x04`, not the wider value `0x104`.

5. **First cycle argument pair.** Regardless of the F-U120 result, the initial cycle field is at most 31. Therefore F-U123’s second displayed argument on initial entry is zero.

6. **Later cycle fields differ.** Under ordinary 32-bit shift behavior, an F-U132 result of 1 produces next cycle argument 4 and next cycle field `0x04000000`. The next F-U123 second argument is `0x00040000`.

7. **Target changes once.** If three terminal iterations are reached, F-U123’s calculated targets are, in order, `0x08833787`, `0x088337c5`, and `0x088337c5`. Reusing the initial target or alternating targets would contradict the representation.

8. **No successful completion.** If calls return and direct accesses succeed indefinitely, terminal-cycle entry leads to continued execution, not a normal return.

## Uncertainties

**Established static evidence:** represented access order and widths; displayed argument lists; unsigned masks and logical shifts; the unconditional cycle; and the initial-to-subsequent target transition at F-U123.

**Inferred intent:** repeated scaling and call patterns may indicate systematic transformation. They do not establish an application domain or object model.

**documented_only unknowns:** fidelity of instruction recovery, actual reachability, signed-overflow behavior, endianness, alignment, memory-map validity, dependency semantics, ABI details, concurrency, and abnormal-execution behavior.

Low numeric byte-store destinations must not be silently replaced by object-relative addressing. The terminal cycle must not be made finite because indefinite execution appears unusual. Conversely, represented addresses and control flow do not prove that an actual execution can reach or sustain the cycle.

## Evidence

This contract uses only the supplied static representation and its warning annotations. Those annotations report invalid instruction data and removal of unreachable regions.

The terminal target distinction follows from separate represented base assignments before initial cycle entry and at the loop tail, each combined with the same displacement.

No dependency implementations or runtime traces were available. **No runtime tests have been performed.** Acceptance examples express arithmetic and ordering consequences subject to the stated qualifications.
