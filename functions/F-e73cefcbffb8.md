<!-- Trusted provenance: F-e73cefcbffb8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e73cefcbffb8

## Purpose

The supplied static representation describes an operation dominated by address-derived memory writes, low-address state changes, one indirect call, and further writes based on memory and the call’s result. It does not establish a recognizable application-level purpose.

The representation explicitly reports invalid instruction data and ends at a truncated control-flow boundary. Consequently, this contract distinguishes:

- **Established static evidence:** operations and ordering expressed by the supplied representation.
- **Inferred intent:** possible interpretations that are not acceptance requirements.
- **Unknowns:** behavior that cannot be established without a trustworthy instruction stream, execution environment, or dependency contract.

This is not a contract for a conventional initializer or memory-management routine. In particular, the evidence does not justify treating its address calculations as accesses within a single valid object.

## Interface

### Explicit inputs

The displayed interface has four arguments:

| Position | Represented width | Role established by the evidence |
|---|---:|---|
| First | 32 bits | Integer used to construct several memory addresses and address-derived values. |
| Second | 32 bits | Opaque bit pattern written both in full and after truncation to 16 bits. |
| Third | 32 bits, signed interpretation | Value subjected to a right shift by 13 before a 32-bit store. |
| Fourth | 32 bits | Integer added to a doubled address to select a 32-bit write destination. |

For the descriptions below, the arguments are called **base**, **payload**, **shift input**, and **displacement**. These are descriptive contract terms, not claims about source-level types or intended meanings.

### Additional input state

Behavior also depends on:

- An incoming register value absent from the displayed argument list.
- A 32-bit global value used to locate a two-word memory record.
- A low-address global value whose declared width and signedness are unavailable.
- A global pointer selecting a 32-bit writable location.
- Existing contents of numerous computed memory locations.
- The result and side effects of one indirectly invoked dependency.

The incoming register value is shifted left by 13. A byte derived from that shifted value is later written to address `0x00000001`.

### Output and termination

The displayed interface has no return value. It does not show a normal return. Its represented continuation ends at an invalid-instruction boundary.

The dependency call produces a represented 64-bit result, split into lower and upper 32-bit halves. That does not establish the dependency’s actual source-language return type or calling convention.

### Preconditions

No validation, bounds checking, null checking, alignment checking, or error reporting is visible.

For execution to reach the final represented operations, all accessed locations must support their requested access widths and permissions. This includes low addresses, potentially unaligned addresses, and potentially wrapped addresses. A normal user-space memory model is not sufficient by itself to justify these accesses.

Address calculations below describe the apparent 32-bit machine model. They are not a claim that signed overflow is well-defined in a source language.

## Behavior

### Address and width conventions

Unless otherwise stated:

- Address-bearing values and full-word accesses are 32 bits.
- Halfword accesses are 16 bits.
- Byte accesses are 8 bits.
- A narrow store retains the low bits of its source.
- Address arithmetic is described modulo \(2^{32}\), subject to the uncertainty about the original instruction stream.
- An unsigned right shift introduces zeros.
- A signed right shift is interpreted arithmetically where the representation gives the operand a signed type.

Define the **derived address** as base plus four, and the **doubled address** as twice that derived address. These values are calculated from the input, not read from memory.

All ordering below is the order expressed by the static representation. It does not establish bus transactions, atomicity, or preservation of every apparent access by the original machine code.

### Initial input-derived writes

The operation first derives the incoming-register shifted value and the derived address. It then:

- Writes the low 16 bits of the derived address to that address twice consecutively.
- Writes the low 16 bits of payload at doubled address plus `0x28`.
- Writes shift input, arithmetically shifted right by 13, at doubled address plus displacement.
- Writes the full payload at the address obtained by unsigned-right-shifting the derived address by one bit.

The duplicate halfword writes are both present in the representation. They cannot be collapsed when specifying potentially observable memory or device accesses.

Next, a 32-bit global is read and retained as the **record selector**. This read precedes the subsequent low-address writes.

The operation then:

- Writes the low 16 bits of doubled address plus two at doubled address plus `0x32`.
- Writes the full derived address at the derived address itself.
- Writes the 32-bit constant `0x38` at base plus eight.

The full-word self-address write overlaps the destination of the earlier duplicate halfword writes. The earlier writes remain part of the observable sequence even where the later write replaces their contents.

### Low-address and global-state changes

The 32-bit value beginning at address `0x00000005` is read, masked with `0x00FFFFFF`, and written back. This clears its most significant eight value bits. Which byte address contains those bits depends on endianness.

At address `0x00000009`, the operation writes the byte formed by:

1. Taking the low 16 bits of the derived address.
2. Multiplying by `0x400`.
3. Retaining the low 16 bits of that product.
4. Shifting the retained value right by eight.

Equivalently, for this expression’s widths, the stored byte contains bits 0–5 of the derived address shifted left by two. Its low two bits are zero.

A binary selector is then formed from bit 22 of the derived address:

| Derived-address bit 22 | Selector |
|---|---:|
| Clear | 1 |
| Set | 0 |

That selector controls three writes:

| Selector | 16-bit destination and value | 32-bit destination and value | Additional 16-bit destination and value |
|---:|---|---|---|
| 0 | `0x0000001C` receives `0x0000` | `0x00000000` receives `0x00000038` | `0x00000000` receives `0x0038` |
| 1 | `0x0000001A` receives `0xFFFE` | `0xFFFFFFFC` receives `0x00000038` | `0x00001FFF` receives `0x0038` |

Between the full-word selector-dependent write and the final halfword selector-dependent write:

- A reset-designated global is set to zero.
- Address `0x00000001` receives bits 8–15 of the previously shifted incoming-register value.

Under ordinary 32-bit left-shift bit-vector semantics, that byte depends only on the incoming value’s low three bits, shifted into byte positions 5–7.

After these operations, a global beginning at address `0x00000062` is assigned `0x22`. Its access width is not recoverable from the supplied declaration.

### Record-derived addressing and publication

The retained record selector is doubled to locate a record with two adjacent 32-bit words. The first word is read and retained as **record payload**. The second word is read and doubled to form the **record-derived address**.

A value associated with address `0x00000E2E` is then read and converted to 32 bits. Its declaration is missing, so its load width and any preceding sign extension remain unknown.

The operation:

- Writes `0x71` as a 32-bit value at record-derived address plus eight.
- Writes record payload as a 32-bit value at an address equal to the low-address value multiplied by `0x800`, minus `0x71`.

It next evaluates the sign of record-derived address plus four by a represented signed right shift of 31 bits. Under that interpretation, the result is either zero or all ones.

This produces two possible **small addresses**:

| Signed interpretation of record-derived address plus four | Small address |
|---|---:|
| Nonnegative | `0x00000002` |
| Negative | `0x000000E2` |

A 32-bit zero is written at record-derived address for the nonnegative case, or at record-derived address minus one for the negative case.

The global pointer is then loaded. The small address is written as a 32-bit value through that pointer.

From the global pointer’s value, the operation constructs a **scaled structure address** by adding four and multiplying by `0x100`. At that address:

1. A 32-bit self-address value is written.
2. The small address is written at offset four.
3. A zero byte is written at offset zero.

The final byte store may modify the self-address value just written. Its effect on that value’s numeric interpretation depends on endianness.

A vector-designated global is assigned an address obtained by taking bits 16–23 of global-pointer-plus-four. A halfword containing the low 16 bits of the record-derived address is then written through that newly assigned address. The destination is therefore between `0x00` and `0xFF`, inclusive, and need not be aligned.

### Preparation for the dependency call

The **working address** is the small address multiplied by `0x20`:

- Small address `0x02` gives working address `0x40`.
- Small address `0xE2` gives working address `0x1C40`.

Before calling the dependency, the operation:

- Writes the small address as a halfword at `0x22` or `0x102`, respectively.
- Assigns `0xF760` to another global whose declaration and access width are unavailable.
- Writes the working address as a 32-bit self-pointer at working address plus `0x20`.

No explicit arguments are shown at the call site. This does not prove that the dependency consumes no register, stack, or global inputs.

### Dependency result and final mutations

After the indirect call:

- The lower 32 result bits are written to address `0x5FDD8024`.
- The upper 32 result bits are retained as a later address offset.
- A 32-bit value is read from the working address and treated as a **halfword target address**.
- Another 32-bit value is read from working address plus eight and treated as a **word target address**.

These reads occur after the dependency. No pre-call contents at those locations may be assumed to survive the call.

The remaining represented operations are ordered as follows:

1. Multiply the halfword target address by eight and write that resulting 32-bit address to itself.
2. Write the small address as a 32-bit value at the word target address.
3. Arithmetically right-shift the multiplied address using the low byte of the halfword target address as the represented shift count.
4. Write the low 16 result bits at the small address.
5. Read one byte at small address multiplied by `0x400`, plus `0x0F`.
6. Write the low byte of the small address at twice the shifted result.
7. Write a zero halfword at the halfword target address.
8. Write a zero halfword at word target address plus six.
9. Write the low 16 bits of the halfword target address at that target address.
10. Write small address shifted left by four as a 32-bit value at word target address plus `0x20`.
11. Write the low 16 bits of the halfword target address at upper-result-half plus halfword target address.
12. Write the low 16 bits of word target address plus four at halfword target address plus the previously read byte.
13. Write the upper result half as a 32-bit value at small address plus upper-result-half.

The variable shift’s behavior for counts outside 0–31 is not established. No masking to five bits, saturation, or instruction-specific rule should be assumed.

The represented flow then terminates at the invalid-instruction boundary.

## Side effects

The operation can mutate caller-associated memory, globals, low memory, and arbitrary computed destinations. The evidence does not establish disjoint regions.

Aliasing is significant:

- Input-derived writes can change globals before they are read.
- Low-address writes can overlap one another.
- Published structures can overlap later read locations.
- The dependency can modify memory used after the call.
- Final writes can overlap each other and previously initialized locations.

Consequently, final memory contents alone do not describe the contract. Ordered reads and writes matter, especially for mapped devices or faulting addresses.

No allocation, release, ownership transfer, or resource acquisition is explicitly established. Writing an address into a global or memory record does not prove ownership transfer.

There is no visible rollback. If an access faults or the dependency does not return, earlier side effects may remain. The representation supplies no recovery behavior.

## Dependencies

### F-000000000001

This is a document-local opaque identifier for the single indirect call target. It is not an externally verified function identity.

Established characteristics:

- One indirect call is represented.
- No explicit arguments appear.
- Its represented result is consumed as two 32-bit halves.
- Several memory reads occur only after it returns.

Unknown characteristics:

- Actual calling convention and implicit inputs.
- Whether it returns normally.
- Global, memory, register, device, or synchronization effects.
- Allocation, retention, or release of resources.
- Whether the represented 64-bit result corresponds to a genuine return value.

The invalid-instruction endpoint is not established as a second callable dependency.

## Acceptance examples

These examples are conditional consequences of the representation, not executed tests.

### Duplicate writes remain observable

If the derived address is writable and execution reaches the initial stores, two consecutive 16-bit writes of its low half must be represented before the later full-word self-address write. Matching only the eventual contents is insufficient for an access-trace comparison.

### Payload truncation

For payload `0xA1B2C3D4`, the input-derived halfword destination receives `0xC3D4`, while the payload’s full-word destination receives `0xA1B2C3D4`, assuming no earlier failure.

### Signed shift input

With shift input interpreted as signed `-8192`, its right shift by 13 produces `-1` under arithmetic-shift semantics. The corresponding 32-bit store is therefore `0xFFFFFFFF`, not a zero-filled positive result.

### Selector-dependent low-address writes

If derived-address bit 22 is clear, the selector-dependent full-word write targets `0xFFFFFFFC`, and the later selector-dependent halfword write targets `0x1FFF`. A model that redirects either write to a safer address is not equivalent.

### Negative record-derived branch

If record-derived address plus four is negative in the represented signed 32-bit interpretation, the small address is `0xE2`, the working address is `0x1C40`, and the record-associated zero write is at record-derived address minus one.

### Dependency-modified targets

If F-000000000001 changes the words at working address and working address plus eight, the final phase uses those post-call words. Reusing previously observed values would contradict the represented ordering.

### Replaced zero halfword

If both stores reach the halfword target without intervening alias effects, that target first receives zero and later receives its own low 16 address bits. The intermediate zero store remains part of the trace.

## Uncertainties

The following are **documented_only**:

- Whether the supplied representation corresponds to executable instructions throughout.
- The real behavior at the invalid-instruction endpoint.
- Architecture, endianness, alignment rules, memory permissions, and device mappings.
- Exact declarations and access widths for globals lacking type information.
- The identity and role of the incoming nonargument register.
- Actual overflow and oversized-shift behavior.
- The indirect dependency’s complete interface and side effects.
- Whether the apparent address-based operations reflect real data processing, privileged behavior, or erroneous decoding.

Possible initialization or device-control intent is inference only. Neither interpretation supplies missing safety guarantees or permits simplifying the observed sequence.

## Evidence

The only evidence is the supplied static representation and its warnings. It directly supports the described expressions, apparent access widths, ordering, indirect call, result-half usage, and lack of a normal return in the displayed flow.

It does not establish runtime reachability, successful memory accesses, valid object boundaries, ownership, or a correct reconstruction of the underlying instructions.

No runtime tests have been performed. No implementation, dependency body, executable, or external provenance was inspected.
