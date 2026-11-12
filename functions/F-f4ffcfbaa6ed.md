<!-- Trusted provenance: F-f4ffcfbaa6ed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f4ffcfbaa6ed

## Purpose

F-f4ffcfbaa6ed performs an ordered sequence of indirect calls and memory updates. Its visible effects include copying a retained 32-bit value into several locations, storing scaled call results, updating two byte-selected destinations, assigning a global control value, clearing a byte and a halfword, and invoking address zero with three zero arguments.

The supplied static representation does not establish a higher-level purpose. In particular, it does not justify describing this operation as initialization, allocation, serialization, a system call wrapper, or object construction.

This contract specifies the visible sequence without assigning semantics to unresolved callees. All behavior after an indirect call is conditional on that call returning normally.

## Interface

### Explicit inputs

| Input | Width | Visible use |
|---|---:|---|
| First argument | 32 bits | Passed to the first indirect call. |
| Second argument | 32 bits | No explicit subsequent use appears. |
| Third argument | Unsigned 32 bits | Used to construct a shifted call argument and a word-store address. |

The absence of an explicit use of the second argument does not prove that its incoming register or stack location is irrelevant to unresolved callees.

### Ambient inputs

The function also depends on:

- An incoming link-register value used to form the first indirect-call target.
- Additional indirect-call targets formed from fixed code-associated anchors and offsets.
- Memory read through addresses derived from call results.
- A global whose value is captured and later used as a memory base.
- A global destination assigned near the end.
- An additional register-associated pointer used for a terminal byte clear.
- The execution environment’s handling of a call to address zero.

These are not validated by the visible body.

### Output

The declared interface has no return value. There is no explicit success code, failure code, or structured result.

A normal return is visible only after the call to address zero. Whether that call can return is unknown. The contract therefore does not promise completion or a normal return.

### Width and layout conventions

The visible accesses have these widths:

- Byte: 8 bits.
- Halfword: 16 bits.
- Word: 32 bits.
- Paired call result: 64 bits, split into low and high 32-bit halves.

Offsets below are byte offsets. A byte store retains only the low eight bits of its value. Unsigned right shifts are logical.

Several address expressions scale values represented as signed 32-bit integers. The representation does not establish portable overflow behavior for those expressions. Acceptance examples involving them require representable intermediate values and valid addresses. A 32-bit modular machine-address interpretation may be intended, but is not independently established.

Endianness is not specified by the excerpt. Numeric word values and the low/high halves of paired results are specified without asserting their byte order in memory.

## Behavior

### Dependency notation and ordering

The identifiers F-indirect-001 through F-indirect-066 below are opaque **call-site dependency identifiers**, assigned in execution order. They do not establish 66 distinct target functions. Different sites may resolve to the same target.

“Zero explicit arguments” means that the supplied representation shows no arguments at that site. It does not establish that other registers, stack contents, or globals are unavailable to the callee.

Calls whose results are not described still execute; only their results are unused by the visible body.

### 1. Obtain and retain the source word

The opening calls occur in this order:

1. F-indirect-001 receives the first input argument. Its target depends on the incoming link register.
2. F-indirect-002, F-indirect-003, and F-indirect-004 are invoked with zero explicit arguments.
3. F-indirect-005 is invoked with zero explicit arguments and supplies a 32-bit result.

The result of F-indirect-005 is scaled by four. Call the scaled value the **first base**.

Before another dependency is invoked, the function:

- Reads one 32-bit word at first base + 0x74.
- Retains that word as the **source word**.
- Writes the low byte of the source word at first base + 1.

The source-word read precedes the byte write, including when their storage overlaps.

F-indirect-006 then receives the first base.

F-indirect-007 through F-indirect-009 execute with zero explicit arguments. F-indirect-010 then supplies a 32-bit result, also with zero explicit arguments. The source word is written as a 32-bit word at:

**16 × result of F-indirect-010 + 0x14**

The retained source word is not reloaded from its original address. Changes to that address by intervening callees do not change the retained value.

### 2. Process the third input and copy the source word again

F-indirect-011 receives a 32-bit value formed by logically shifting the third input right by two and then shifting that result left by sixteen.

Within the displayed unsigned 32-bit arithmetic, this is equivalent to placing input bits 2 through 17 into output bits 16 through 31 and clearing all other output bits. Input bits above bit 17 do not survive.

The following calls then execute:

- F-indirect-012 returns a 32-bit value.
- F-indirect-013 receives that value.
- F-indirect-014 returns another 32-bit value.
- F-indirect-015 receives that second value.
- F-indirect-016 through F-indirect-018 execute with zero explicit arguments.
- F-indirect-019 supplies a 32-bit result.

The source word is written at:

**4 × result of F-indirect-019 + 0x14**

F-indirect-020 subsequently receives four times that result. The write occurs before the call.

### 3. Store a scaled result through the third input

F-indirect-021 through F-indirect-023 execute with zero explicit arguments. F-indirect-024 then supplies a 32-bit result.

That result is shifted left by two. Call the resulting value the **input-linked value**.

The function clears the low two bits of the third input, adds 0x14, and writes the input-linked value there as a 32-bit word. Thus, all third-input values differing only in their low two bits select the same destination for this particular store, subject to address arithmetic.

F-indirect-025 receives the input-linked value twice.

The next forwarding sequence is:

- F-indirect-026 returns a 32-bit value.
- F-indirect-027 receives that value.
- F-indirect-028 returns a 32-bit value.
- F-indirect-029 receives that value.

F-indirect-030 through F-indirect-032 then execute with zero explicit arguments. F-indirect-033 supplies a 32-bit result.

The function writes a zero word at:

**4 × result of F-indirect-033 + 0x14**

F-indirect-034 receives four times that result after the zero write.

### 4. Distribute a paired result and capture a global base

F-indirect-035 through F-indirect-037 execute with zero explicit arguments. F-indirect-038 returns a 64-bit value.

The low 32-bit half is shifted left by two to form the **distributed value**. The high 32-bit half is used as an index.

The function writes the distributed value as a word at:

**4 × high half of F-indirect-038 + 0x14**

It then reads a global value and retains it as the **saved global base**. This global read occurs after the word store and before F-indirect-039.

F-indirect-039 receives four explicit arguments, in order:

1. Distributed value.
2. High half of F-indirect-038.
3. Distributed value.
4. Distributed value.

F-indirect-040 through F-indirect-042 execute with zero explicit arguments. F-indirect-043 then supplies a 32-bit result.

The distributed value is written again, at:

**4 × result of F-indirect-043 + 0x14**

After that write, the function reads one word at saved global base + 0x14. Call this retained word the **payload word**.

This ordering matters:

- A change to the global variable during F-indirect-039 through F-indirect-043 does not replace the already saved base.
- Changes to memory at saved global base + 0x14 before the read can affect the payload word.
- If the preceding distributed-value store aliases that location, the payload read observes the resulting memory state.

F-indirect-044 then receives four times the result of F-indirect-043.

### 5. Perform two byte-indirected updates

F-indirect-045 through F-indirect-047 execute with zero explicit arguments. F-indirect-048 returns a 64-bit value. Its low and high 32-bit halves are retained separately.

The first byte-indirected update uses the retained source word:

1. Read an unsigned byte at **(source word logically shifted right by 23) + 1**.
2. Treat that byte as an unsigned numeric address component.
3. Add 0x11.
4. Write the low byte of the payload word to that destination.

The byte-table read occurs after F-indirect-048 and before F-indirect-049. Its destination is derived from the byte read, not directly from the source word.

F-indirect-049 receives:

1. High half of F-indirect-048.
2. The same high half.
3. The same high half.
4. Low half of F-indirect-048 shifted left by two.

F-indirect-050 returns a 32-bit value, which F-indirect-051 receives. F-indirect-052 through F-indirect-054 execute with zero explicit arguments. F-indirect-055 supplies a 32-bit result.

The second byte-indirected update then occurs:

1. Mask the retained low half of F-indirect-048 with 0x3fffffff.
2. Logically shift the masked value right by 21.
3. Add one and read an unsigned byte there.
4. Add 0x11 to that byte’s numeric value.
5. Write the low byte of the payload word shifted right by 22.

Consequently, the second written byte contains payload bits 22 through 29. Payload bits 30 and 31 do not survive the byte truncation.

F-indirect-056 receives the result of F-indirect-055 shifted left by two.

Both byte-indirected reads use indices from 1 through 512 inclusive. Both resulting write addresses range from 0x11 through 0x110 inclusive. The excerpt provides no bounds checks or proof that these low numeric addresses are valid in the execution environment.

The two byte destinations can coincide. In that case, the second write occurs later, with intervening dependency calls.

### 6. Terminal calls, global assignment, and clears

F-indirect-057 through F-indirect-059 execute with zero explicit arguments. F-indirect-060 supplies a 32-bit result.

F-indirect-061 receives that result shifted left by two in each of its two explicit argument positions.

F-indirect-062 executes with zero explicit arguments. F-indirect-063 returns a 32-bit value, which F-indirect-064 receives.

F-indirect-065 then executes with zero explicit arguments. Its result is assigned to the global control destination.

After that assignment:

- A zero byte is written through the additional register-associated pointer.
- A zero halfword is written at **(retained low half of F-indirect-048 masked with 0x3fffffff) logically shifted right by 7**.
- F-indirect-066 invokes address zero with three explicit zero arguments.
- The function returns only if that invocation returns.

The halfword address lies between zero and 0x007fffff inclusive. It is not explicitly aligned. No alignment repair or error branch is visible.

## Side effects

The body explicitly performs these data-memory updates in order:

1. Source word’s low byte to first base + 1.
2. Source word to the destination selected by F-indirect-010.
3. Source word to the destination selected by F-indirect-019.
4. Input-linked value to the third-input-derived destination.
5. Zero word to the destination selected by F-indirect-033.
6. Distributed value to the destination selected by the high half of F-indirect-038.
7. Distributed value to the destination selected by F-indirect-043.
8. Payload word’s low byte to the first byte-indirected destination.
9. Payload bits 22–29 to the second byte-indirected destination.
10. Result of F-indirect-065 to a global.
11. Zero byte through the additional pointer.
12. Zero halfword to the retained-low-half-derived address.

These writes are interleaved with calls and reads as specified above. They are not an atomic transaction. There is no visible rollback.

No allocation, ownership transfer, release, or cleanup operation is identifiable from the body alone. That does not establish their absence inside dependencies. The caller cannot assume unresolved callees preserve globals, preserve referenced memory, or retain or release any resource.

Aliasing is not excluded. Later writes may overwrite earlier writes, and calls may alter locations before later explicit reads. Numeric address expressions and execution order take precedence over any inferred object or field interpretation.

No explicit checks cover null pointers, address validity, index bounds, alignment, overflow, or callee failures. Invalid accesses may fault or have environment-specific effects. The excerpt does not establish exception handling or recovery.

## Dependencies

| Opaque dependency sites | Visible contract |
|---|---|
| F-indirect-001 | Target depends on incoming link state; receives first input. |
| F-indirect-002–F-indirect-065 | Indirect targets derived from code-associated anchors and offsets; arguments, consumed results, and order are specified in Behavior. |
| F-indirect-038, F-indirect-048 | Each produces a paired 64-bit result whose halves are consumed separately. |
| F-indirect-065 | Produces the value assigned to the global control destination; relationship to the additional pointer is unresolved. |
| F-indirect-066 | Fixed zero target; receives three zeros. |

No dependency implementations or calling contracts are supplied. A site returning a displayed 32-bit or 64-bit value establishes only how this body consumes that result.

The global base input, global output, byte tables, and register-associated pointer are ambient dependencies. Their ownership, lifetime, validity, and alias relationships are unknown.

## Acceptance examples

These are conditional static acceptance examples, not runtime observations. All relevant calls must return normally, and all accessed memory must be valid.

### Source-word retention

If F-indirect-005 returns 0x100, the first base is 0x400. If the word read at 0x474 is 0x12345678:

- The immediate byte write at 0x401 is 0x78.
- The later two source-word stores use 0x12345678.
- A dependency changing memory at 0x474 does not cause an explicit reload.

### Third-input transformations

For a third input of 0x00000007:

- F-indirect-011 receives 0x00010000.
- The input-linked word destination is 0x18.

For a third input of 0x00040007, F-indirect-011 still receives 0x00010000 because the additional high input bit is discarded by the unsigned 32-bit shift sequence. The word destination, however, is 0x00040018.

These transformations are distinct; acceptance must not conflate their input-bit dependencies.

### Saved base versus later global changes

If the captured global base is 0x2000, a dependency later replacing that global with 0x3000 does not redirect the payload read. The explicit payload read remains at 0x2014. A dependency changing the word at 0x2014 can change the payload.

### Byte selection and truncation

Suppose the source word is 0x01800000 and the byte at address 4 is 0x20. The first byte destination is 0x31.

If the payload word is 0xABCDEF12, that destination receives 0x12.

Suppose the retained low half of F-indirect-048 is 0x00600080 and the byte at address 4 is still 0x20 when the second byte-table read occurs:

- The second byte destination is also 0x31.
- Its written value is 0xAF, from payload bits 22–29.
- The terminal halfword-clear address is 0xC001.

The unaligned halfword address is not corrected. Whether that store succeeds is environment-dependent.

### Non-returning terminal behavior

If F-indirect-066 traps or does not return, there is no normal function return. The preceding global assignment and clears have already been attempted in the specified order; no rollback is shown.

## Uncertainties

The following remain **documented_only**:

- The semantic purpose of the operation and its data structures.
- Resolved dependency identities and whether sites share targets.
- Completeness of recovered argument and return conventions.
- How the additional pointer is produced, and whether it relates to F-indirect-065.
- Global widths beyond their displayed uses, storage overlap, and external visibility.
- Signed overflow, signed left-shift behavior, and address wraparound outside representable cases.
- Endianness, alignment requirements, memory protection, and low-address mappings.
- Volatile, device-memory, concurrency, and memory-ordering semantics.
- Whether dependencies allocate, release resources, mutate globals, or fail nonlocally.
- Whether address zero is callable and whether normal completion is possible.

A plausible intent is to distribute indexed values and update packed metadata. That interpretation is not established and is not an acceptance requirement.

## Evidence

Established static evidence consists solely of the supplied representation of F-f4ffcfbaa6ed: explicit call ordering, displayed operand widths, arithmetic and masks, memory access widths, retained results, global accesses, and the final zero-address call.

The source representation warns of overlapping global symbols. No independent global layout or dependency implementation is available, so distinct logical descriptions here do not imply disjoint storage.

No runtime tests have been performed. The acceptance examples are arithmetic and ordering consequences under stated preconditions, not evidence of successful execution on a target system.
