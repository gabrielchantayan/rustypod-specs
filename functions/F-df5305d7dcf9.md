<!-- Trusted provenance: F-df5305d7dcf9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-df5305d7dcf9

## Purpose

F-df5305d7dcf9 performs an ordered sequence of indirect dependency invocations, transforms selected returned values, mutates memory through fixed and computed locations, and finally invokes an unresolved dependency with four arguments.

No application-level purpose is established. Some constants resemble encoded punctuation or letters, and some writes resemble links between records. Neither text output nor an object-management abstraction is established.

This contract distinguishes:

- **Established static evidence:** represented value widths, explicit memory-access widths, arithmetic dependencies, calls, and ordering.
- **Inferred intent:** interpretations that impose no additional requirements.
- **documented_only:** unresolved behavior or explicitly identified coverage limits.

Access history matters independently of final contents. Repeated writes and separate storage evaluations cannot be collapsed merely because their expressions resemble one another.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no declared result. There is no caller-provided buffer, length, status destination, or error object.

Implicit inputs include entry machine state, dependency results, unresolved secondary-register outputs, and memory contents evaluated during execution. Dependencies may change memory before the direct-mutation phase.

The terminal invocation receives:

| Position | Represented value |
|---|---|
| First | 0x72 |
| Second | A retained 32-bit value read at offset 0x64 from a late-derived base |
| Third | A retained 32-bit value captured earlier at offset 0x14 from the late base |
| Fourth | A subsequently captured 32-bit value read at offset 0x74 from the same late-derived base as the second argument |

These arguments have no established interpretation as characters, handles, pointers, or ownership tokens.

### Widths and storage evidence

Explicitly typed memory accesses establish the following represented widths:

| Access | Width |
|---|---:|
| Byte dereference | 8 bits |
| Halfword dereference | 16 bits |
| Word dereference | 32 bits |
| Paired result consumed in two portions | 64 bits total, divided into two 32-bit portions |

An explicit unsigned byte read yields 0–255 when widened. An explicit byte store retains the low eight bits of its supplied value. A paired result does not establish a formal calling convention.

**Named-global accesses require a different qualification.** Their declarations are absent. A conversion or receiving value’s width does not, by itself, establish the width of the underlying storage access.

| Named storage associated with address | Established value-level evidence | Unresolved storage detail |
|---|---|---|
| 0x05 | Its evaluated value is converted to unsigned 32 bits | Source width, signedness, and source-imposed range |
| 0x01, 0x15 | Assignment receives an explicitly eight-bit-converted value | Destination width and any further assignment conversion |
| 0x54 | Assignment receives a twice-scaled signed 32-bit expression | Destination width |
| 0x60 | Assignment receives 0x20 | Destination width |
| 0x087FBBBC | Evaluation supplies a signed 32-bit retained base | Source access width and conversion details |
| 0xA9 | Evaluation supplies a retained 32-bit value | Source access width and conversion details |
| 0x3A | Assignment receives character-converted arithmetic | Destination width and final conversion |
| 0x67 | Initial assignment receives an eight-bit-converted value; three later evaluations have distinct consumers | Storage width, signedness, and evaluation range |
| 0x21 | Assignment receives the first later evaluation of storage associated with 0x67 | Source and destination access widths and assignment conversion |

The addresses in this table identify the locations associated with the supplied named-storage designations; they are not substitutes for missing declarations.

In particular, the conversion associated with 0x05 does **not** establish a byte read or a 0–255 base. Its converted value is unsigned 32-bit, with no narrower range established.

Endianness, alignment, memory mapping, and volatility are **documented_only**.

### Preconditions and completion

Normal progression requires reached destinations to be callable and reached accesses to be supported. No represented validation checks those requirements.

No special handling of low addresses is shown. Introducing null checks, substitute storage, or omitted accesses would change the represented behavior.

The representation ends with a return after the terminal invocation. Whether that invocation returns normally is unresolved. There is no established success result.

## Behavior

### Ordered dependency phase

The opening phase contains many straight-line indirect calls. Calls whose results are unused remain observable boundaries.

The first destination is derived from entry machine state plus 0xFFE. Later destinations use embedded address-like anchors plus 0xFFE. This does not establish a common dispatcher or interchangeable implementations.

The first invocation receives two zeros. Subsequent invocations include calls without represented arguments and calls supplying word or paired results.

Early paired-result consumers include:

- The low portion shifted left by eight and the high portion shifted left by fourteen.
- Four arguments consisting of the low portion shifted by eight, the unchanged high portion, the high portion shifted by fourteen, and the low portion shifted by fourteen.

Later consumers use retained results after intervening invocations. Transformations include left shifts, logical right shifts of unsigned values, masked extraction of bits 8–23 or 12–23, extraction of bits 11–17 from a high result portion, and extraction of bits 4–16 through a seventeen-bit mask. Some calls receive the same transformed value in multiple positions. Others receive explicitly zero-derived arguments or unresolved secondary-register outputs.

There is no represented result-based branch that skips a call, and no retry, timeout, or recovery path.

A complete per-call argument trace and concrete destination identities remain **documented_only** coverage limits. The phase cannot be replaced by an inferred higher-level service.

### Arithmetic qualifications

Unsigned 32-bit expressions support their represented masks and logical shifts. Signed left shifts, signed multiplication overflow, address overflow, and the late variable-count signed right shift need additional machine-level evidence.

Whenever this contract gives modular numerical examples, they are conditional on 32-bit bit-vector arithmetic. They do not resolve the source representation’s signed-arithmetic uncertainties.

No dependency is assumed to preserve globals, avoid aliasing, or preserve the validity of pointer-like results.

### Initial direct mutations

The following names describe relationships, not objects:

- **Primary base:** a returned word multiplied by 2²⁵.
- **Shifted base:** a retained paired result’s low portion shifted left by twenty-five.
- **Small intermediate:** another returned word shifted left by eight, masked with 0x7FFF, then logically shifted right by six.
- **Small base:** twice the small intermediate.
- **Latest pair:** the last paired result before direct mutation.

Under 32-bit bit-vector arithmetic, the small intermediate ranges from 0 to 508 in multiples of four; the small base ranges from 0 to 1016 in multiples of eight. These bounds do not apply to the independently converted named-global value associated with 0x05.

The opening explicit accesses are ordered:

1. A word store of the shifted base at the latest pair’s high portion plus 0x14.
2. A byte store of that high portion at shifted base plus 9.
3. A word store of the primary base at 0x54 beyond the latest pair’s low portion multiplied by 0x100.
4. Another byte store at shifted base plus 9, now supplied by the shifted base.

The primary base’s offset-0x14 word is cleared. An explicit unsigned byte read at primary base plus 5 is retained. A clearing word store through the small base intervenes before that byte is published as a word at primary base plus 0x34.

The later word read at primary base plus 0x14 is not automatically zero: intervening accesses can alias it. The small-base clearing store is repeated, and two subsequent byte stores at small base plus 0x15 remain distinct.

### Dependent addressing and mixed-width copies

The named storage associated with 0x05 is next evaluated and converted to unsigned 32 bits. Call this the **converted global base**. Its source width and narrower numerical range are **documented_only**.

An assignment to named storage associated with 0x01 follows. Its supplied value has already been converted to eight bits, but the destination access width is unresolved.

The converted global base is used for an explicit byte store at offset 0x15 and an explicit word read at offset 0x54. That word read supplies the **address seed**. The offset-0x54 location is then overwritten.

The region also contains:

- Repeated word stores at offset 0x14 through the small base.
- Word reads at offsets 0x54 and 0x74 that supply further addresses.
- Assignments to named storage associated with 0x54 and 0x15.
- Multiplication of retained words by two, four, and eight.
- Repeated byte stores at offset 0x15.
- Stores that place a computed base into one of its own offset locations.
- Explicit byte reads at offset 5 whose widened values supply bases for word reads at offset 0x74.

The assignment associated with fixed address 0x54 receives twice the address seed. Its width is not established by the 32-bit expression. Similarly, the assignment associated with 0x15 receives an eight-bit-converted value without establishing an eight-bit destination.

An expression containing two reads through the same offset-0x74 expression does not establish equal results or a resolved internal evaluation order.

A later dependent word read through a freshly read unsigned byte plus 0x74 supplies the **dependent seed**. The **late base** is twice this dependent seed. The **payload base** is four times this dependent seed.

The address seed and dependent seed are distinct captures. They must not be conflated.

### Constants and publication boundary

Three explicit stores write 0x3A:

- A byte at four times the dependent seed.
- A halfword at eight times the dependent seed.
- A word at the payload base plus 0x20.

Define the **adjusted value** as four times the address seed minus 0x96. Define the **publication base** as four times the adjusted value.

At publication base plus 0x20, four explicit word stores occur in order:

1. 0x20.
2. 0x2F.
3. The widened unsigned byte read from publication base plus 4.
4. 0x20 again.

The byte read occurs after the 0x2F store.

The next boundary is:

| Earlier event | Next event |
|---|---|
| Final explicit word store of 0x20 at publication base plus 0x20 | Evaluate named storage associated with 0x087FBBBC |
| That evaluation | Assign 0x20 to named storage associated with 0x60 |
| That assignment | Begin publication groups |

The evaluation supplies a retained 32-bit **global-derived base**. It is not established as a 32-bit physical read. Likewise, the assignment at 0x60 is not established as a word store.

Their ordering remains established despite unresolved widths. The global evaluation must not move after the 0x60 assignment. Subsequent groups use the captured base rather than reevaluating that global.

### Repeated publication groups

Each group contains four explicit accesses:

| Order | Access |
|---:|---|
| 1 | Word store of the payload base at a scaled target plus 0x54 |
| 2 | Byte store of the publication base’s low byte at global-derived base plus 0x19 |
| 3 | Byte store of the global-derived base’s low byte at publication base plus 0x11 |
| 4 | Word store of the scaled target at global-derived base plus 0x34 |

The first scaled target is eight times the address seed. Subsequent groups use eight times the adjusted value.

The destinations do not advance through successive elements. The groups remain separate accesses even when their expressions and values repeat.

The exact repetition count is not enumerated here and remains a **documented_only** trace-coverage limit. This does not mean that the supplied straight-line sequence has an indeterminate count or permits arbitrary repetition.

### Final captures and mutations

After the publication groups, the initial capture order is:

1. Evaluate named storage associated with 0xA9 into a retained 32-bit value.
2. Explicitly read a word at payload base plus 0x74.
3. Assign character-derived arithmetic to named storage associated with 0x3A.
4. Explicitly read a word at late base plus 0x14.
5. Explicitly read a word at publication base plus 4.

The first capture is not established as a physical word read. The third event is not established as a byte store. Its expression first character-converts the adjusted value, multiplies by eight, and adds 0x30; character signedness and the destination conversion remain unresolved. It establishes no digit encoding.

The captured late-base-plus-0x14 word survives subsequent mutations and becomes the terminal third argument.

Subsequent explicit mutations include byte writes at late base plus 0x0D and publication base plus 8, 0x0D, and 9. The offset-8 byte receives 0x6C. A word store puts the late base into late base plus 0x54.

### Three separate evaluations associated with 0x67

Named storage associated with 0x67 first receives the eight-bit-converted retained value captured from 0xA9. That conversion does not establish the destination width.

Three later evaluations must remain distinct:

| Evaluation | Timing and consumer |
|---|---|
| First | After the word read at late base plus 0x44; supplies the assignment associated with 0x21 |
| Second | After intervening computed accesses, including the word store at global-derived base plus 0x44; supplies an explicit byte store at publication base plus 9 |
| Third | After a byte store at late base plus 9, a word store of the retained 0xA9-derived value at publication base plus 0x44, and a word read at publication base plus 0x54; supplies another explicit byte store at publication base plus 9 |

These are **not established as three byte reads**. Their source widths and ranges remain unresolved. The second and third consumers are explicitly byte-sized, so those stores retain the low eight bits after any required value conversion. The first consumer’s destination width is also unresolved.

Between the first and second evaluations, explicit accesses include a byte store at publication base plus 0x10, two word stores at global-derived base plus 0x54, and a byte store at late base plus 1 using a variable-count signed right shift. Its count is the publication base’s low eight bits; counts outside the word width are unresolved.

A word read at publication base plus 0x40 supplies a word store at publication base plus 0x10. This later word store is distinct from the earlier byte store at that displacement.

### Remaining address chain and dispatch

After the third 0x67-associated evaluation:

- The low byte of the earlier payload-base-plus-0x74 capture is written through later computed locations.
- The saved late-base-plus-0x14 word is stored into late base plus 0x54.
- A fresh word read at publication base plus 0x14 supplies a later byte write.
- Publication base plus 9 is overwritten with the low byte of the earlier publication-base-plus-0x54 read.

A word read at late base plus 4 supplies the **terminal-derived base**. A halfword store writes 0x65 at twice that base.

The terminal second argument is then captured by an explicit word read at terminal-derived base plus 0x64. An intervening byte store and a word store of decimal 100 through another retained base occur before the explicit word read at terminal-derived base plus 0x74 captures the fourth argument.

A further word read at late base plus 0x64 supplies another address. Final mutations include a byte store at late base plus 9, a word store through that newly read address plus 0x54, and a word store at twice the retained terminal second argument.

All precede the terminal indirect invocation. Its destination-selection mechanism is unresolved.

## Side effects

### Layout and aliasing

Explicit computed accesses use these offsets:

| Width | Observed offsets |
|---|---|
| Byte | 1, 5, 8, 9, 0x0D, 0x10, 0x11, 0x15, 0x19 |
| Halfword | Scaled addresses without a common field offset |
| Word | 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74 |

These are access locations, not a structure declaration. Decimal 100 and hexadecimal 0x64 denote the same displacement.

The named fixed-location accesses are separately width-qualified in the interface. They must not be inserted into this explicit-width layout table based solely on their names or assigned expressions.

There is no non-aliasing guarantee. Computed writes may affect named storage, byte writes may affect later word reads, and captured values may differ from later evaluations.

### Ownership and errors

No direct allocation or release operation is established. Dependencies may allocate, release, retain, or invalidate memory; their contracts are unknown.

No locking, atomicity, rollback, explicit error report, or cleanup path is represented. Partial mutations may be visible before terminal dispatch.

Invalid call destinations, inaccessible storage, arithmetic-derived invalid addresses, and dependency failures have no established recovery behavior.

## Dependencies

The following are document-local opaque boundary labels, not recovered concrete identities:

| Opaque ID | Boundary |
|---|---|
| F-00000001 | Entry-derived invocation receiving two zeros |
| F-00000002 | Ordered anchor-derived invocations, sometimes supplying word or paired results |
| F-00000003 | Terminal invocation receiving four arguments beginning with 0x72 |

F-00000002 groups multiple unresolved destinations; it does not identify one implementation. None of these labels implies purity, termination, ownership behavior, global preservation, or a standard calling convention.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Explicit unsigned extraction:** Masking 0x12345678 to twenty-four bits and logically shifting right by eight yields 0x3456.

2. **Conditional truncation:** Under 32-bit bit-vector arithmetic, shifting 0x12345678 left by eight yields 0x34567800.

3. **Small-base boundary:** Under that interpretation, a relevant returned value of 0x7F produces small intermediate 508 and small base 1016; 0x80 produces zero for both.

4. **Named-global conversion:** If the evaluation associated with 0x05 converts to 0x1234, the converted global base is 0x1234, not 0x34. This remains conditional on the missing declaration permitting that evaluated value.

5. **Global boundary:** If the 0x087FBBBC-associated evaluation supplies different values before and after the 0x60 assignment, publication uses the value captured before that assignment. Neither access width is thereby resolved.

6. **Separate 0x67 evaluations:** If the second and third evaluations convert to values ending in 0x34 and 0x56, the respective explicit publication-base-plus-9 byte stores receive 0x34 and 0x56. Reusing the first evaluation violates the represented dependencies. The assignment at 0x21 cannot be given a byte-width expectation without its declaration.

7. **Write history:** The four word stores at publication base plus 0x20 retain the history 0x20, 0x2F, widened-byte value, 0x20. One final store is insufficient.

8. **Retained argument:** Later changes to late base plus 0x14 do not change the already captured terminal third argument. Fresh evaluations elsewhere remain fresh.

9. **Conditional character assignment:** Only if storage associated with 0x3A is eight-bit and ordinary low-bit truncation applies does its final stored pattern equal the low eight bits of eight times the adjusted value plus 0x30. That condition is not established.

## Uncertainties

The following remain **documented_only**:

- Application purpose, object identities, and encoding.
- Concrete dependency identities, complete call trace, and publication repetition count.
- Calling conventions and secondary-register provenance.
- Named-global declarations, physical access widths, and assignment conversions.
- The source-imposed range of the value associated with 0x05.
- Character signedness, endianness, alignment, mapping, and volatility.
- Signed overflow, address overflow, signed shifts, and excessive shift counts.
- Ownership, lifetimes, concurrency, faults, and abnormal termination.
- Terminal destination selection and normal return.

These uncertainties authorize neither invented validation nor substituted storage, cleanup, reordered accesses, or guessed byte-width globals.

## Evidence

This contract uses the supplied source representation as static evidence. Prior prose, comments, and reviewer assertions are not independent behavioral authority.

Explicit dereferences support access-width claims. Conversions, retained-value declarations, constants, and expression dependencies support value-level claims. Missing named-global declarations prevent promoting those value widths into categorical physical-access widths.

The representation establishes the ordered global-evaluation boundary and three distinct evaluations associated with 0x67. It does not establish their missing storage declarations or runtime memory behavior.

No tools were used and no runtime tests were performed.
