<!-- Trusted provenance: F-d64faa9db7b3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d64faa9db7b3

## Purpose

F-d64faa9db7b3 performs an ordered sequence of mixed-width memory writes and reads, then invokes an unresolved indirect dependency with four 32-bit arguments.

The established behavior is memory manipulation and indirect control transfer. The supplied static representation does not establish an application-level operation, a public data structure, or a text-processing purpose. Constants are therefore specified numerically rather than interpreted as characters.

This contract distinguishes:

- **Established static evidence:** represented access widths, values, ordering, and dependencies.
- **Inferred intent:** interpretations that are not acceptance requirements.
- **documented_only:** unresolved properties for which no behavior is invented.

No runtime tests have been performed.

## Interface

### Explicit arguments and result

| Argument | Represented width | Visible use |
|---|---:|---|
| First | 32 bits | No explicit read in the visible body |
| Second | 32 bits | No explicit read in the visible body |
| Third | Signed 32 bits | Reduced by 77; used to derive addresses and stored values |

The represented result is void. There is no explicit success indicator, error result, or returned object.

The first two arguments having no visible use does not establish the complete calling convention. Additional incoming execution state supplies values needed by the body.

### Additional incoming state

The following descriptive names identify incoming values without asserting a public parameter layout:

| Name | Width or role | Use |
|---|---|---|
| Primary address | Address in the represented 32-bit address model | Initial byte destination and subsequent address calculations |
| Initial byte | 8 bits | First stored value |
| Context value | 32 bits, used with signed arithmetic | Address base and stored arithmetic value |
| Link value | 32 bits, used with signed arithmetic | Doubled to form an initial destination base and stored value |

The body also reads a 32-bit value from a fixed global storage location after its initial writes. That captured value is called the **anchor address**.

Neither the global storage location nor the memory reached through the captured anchor is established to be independent of other accesses. In particular, writes before the capture can affect the captured value if they overlap its storage.

The final indirect target’s identity, provenance, and validity are **documented_only**.

### Derived values and arithmetic

The contract uses these descriptive quantities:

- **Adjusted input:** third argument minus 77.
- **Scaled address:** four times the adjusted input, subsequently represented as an unsigned 32-bit quantity.
- **Expanded address:** eight times the adjusted input.
- **Doubled context:** twice the context value.
- **Doubled link:** twice the link value.
- **Low byte:** the least significant eight bits of a value.

These names do not imply separate storage objects.

Memory accesses have widths of 8, 16, or 32 bits. Offsets are byte offsets. A 32-bit access spans four consecutive addressed bytes; a 16-bit access spans two.

Some arithmetic is represented with signed 32-bit operands. This contract does not replace signed-overflow uncertainty with an assumed wraparound rule. Address wrapping, address conversion, endianness, and alignment behavior remain **documented_only**. Numerical examples avoid overflow in the arithmetic they evaluate.

### Preconditions for normal completion

Every reached read and write must be permitted at its represented width. The indirect target must be callable with the observed arguments and must return for the visible body to return normally.

No visible checks establish:

- Non-null or mapped addresses.
- Alignment.
- Buffer or object sizes.
- Disjointness of regions.
- Validity of memory-derived addresses.
- Validity of the indirect target.

These are execution prerequisites, not validations performed by the function.

## Behavior

### Initial writes and anchor capture

The first memory effect is an 8-bit write of the initial byte to the primary address. The third argument is then adjusted by subtracting 77.

The next writes are a 16-bit `0x003A` at twice the primary address and a 32-bit `0x0000003A` at primary address plus `0x20`.

At scaled address plus `0x20`, four successive 32-bit writes carry:

1. `0x00000020`.
2. `0x0000002F`.
3. The zero-extended byte read from scaled address plus `0x04`.
4. `0x00000020`.

The byte read occurs after the first two writes in that sequence and before the third. It is not an entry-time snapshot. Earlier writes can affect it through overlap.

Only after this sequence does the function capture the anchor address from its fixed global storage location. It then writes the 32-bit value `0x00000020` at absolute address `0x60`.

The anchor remains the captured value throughout its subsequent visible uses. Later changes to the location from which it was loaded do not cause a refresh.

### Repeated relationship writes

The function first writes doubled context as 32 bits at doubled link plus `0x54`. It then performs these writes in order:

| Destination | Width | Value |
|---|---:|---|
| Anchor address plus `0x19` | 8 bits | Low byte of scaled address |
| Scaled address plus `0x11` | 8 bits | Low byte of anchor address |
| Anchor address plus `0x34` | 32 bits | Doubled link |

A fixed, explicitly repeated sequence follows. Each repeated group writes:

| Within-group order | Destination | Width | Value |
|---:|---|---:|---|
| 1 | Expanded address plus `0x54` | 32 bits | Doubled context |
| 2 | Anchor address plus `0x19` | 8 bits | Low byte of scaled address |
| 3 | Scaled address plus `0x11` | 8 bits | Low byte of anchor address |
| 4 | Anchor address plus `0x34` | 32 bits | Expanded address |

This is a finite sequence of represented repetitions, not a demonstrated conditional traversal. There is no visible per-group address progression, refreshed input, or data-dependent termination. The exact repetition count is **documented_only** in this contract.

The initial value written at anchor address plus `0x34` is doubled link; the repeated value is expanded address. Repeated writes cannot be collapsed into a single final assignment where access count, overlapping destinations, or externally observable writes matter.

### Captures and context-field updates

After the repeated phase, the function captures, in order:

- A 32-bit **global sample** from absolute address `0xA9`.
- A 32-bit **doubled-context sample** from doubled context plus `0x74`.

It next writes one byte at absolute address `0x3A`. The stored byte is congruent to eight times the adjusted input plus `0x30`, modulo 256. This describes the byte result without imposing an additional signed 32-bit multiplication at this particular store.

The following captures are a 32-bit **context sample** from context value plus `0x14`, then a 32-bit **scaled sample** from scaled address plus `0x04`.

Before the next context-field read, the writes occur in this order:

| Destination | Width | Value |
|---|---:|---|
| Context value plus `0x0D` | 8 bits | Low byte of scaled address |
| Scaled address plus `0x08` | 8 bits | `0x6C` |
| Scaled address plus `0x0D` | 8 bits | Low byte of anchor address |
| Scaled address plus `0x09` | 8 bits | Low byte of anchor address |
| Context value plus `0x54` | 32 bits | Context value |
| Absolute address `0x67` | 8 bits | Low byte of global sample |

A signed 32-bit **context field** is then read from context value plus `0x44`.

The function reads the current byte at absolute address `0x67` and writes it to absolute address `0x21`. It subsequently writes the low byte of the context field at scaled address plus `0x10`, followed by the full 32-bit context field at anchor address plus `0x54`.

At context value plus `0x01`, it writes the low byte of the context field after a signed right shift. The shift count is the low eight bits of scaled address, not a byte read from memory. Since scaled address is a multiple of four, the represented counts are multiples of four within the eight-bit range. Negative-value right shifts and counts of 32 or greater remain **documented_only**.

The function then overwrites anchor address plus `0x54` with the 32-bit scaled address.

### Word replacement and selected destination

A 32-bit **replacement word** is captured from scaled address plus `0x40`. The earlier scaled sample is then written at anchor address plus `0x74`, before the replacement word is written at scaled address plus `0x10`.

That replacement covers offsets `0x10` through `0x13`. It therefore overwrites the earlier byte at offset `0x10` and the repeated-phase byte at offset `0x11`. Their resulting individual byte values depend on byte order.

The next effects, in order, are:

- Write the context value as 32 bits at anchor address plus `0x44`.
- Read a fresh byte from absolute address `0x67` and write it at scaled address plus `0x09`.
- Write the low byte of scaled address at context value plus `0x09`.
- Write the 32-bit global sample at scaled address plus `0x44`.
- Read a 32-bit **selected address** from scaled address plus `0x54`.

**Scaled address plus `0x54` and expanded address plus `0x54` are different expressions; neither equality nor disjointness is guaranteed.** For a third argument of 77, adjusted input, scaled address, and expanded address are all zero, so both expressions evaluate to `0x54`. Earlier repeated writes can consequently affect the later selected-address read through aliasing. Other intervening overlapping writes can also affect that read.

After capturing the selected address, the function reads absolute address `0x67` again and writes that byte at scaled address plus `0x09`. It then writes the low byte of the doubled-context sample at selected address plus `0x0D`, followed by the captured context sample as 32 bits at context value plus `0x54`.

A 32-bit **secondary scaled sample** is captured from scaled address plus `0x14`. The subsequent writes are:

| Destination | Width | Value |
|---|---:|---|
| Context value plus `0x11` | 8 bits | Low byte of doubled-context sample |
| Scaled address plus `0x09` | 8 bits | Low byte of selected address |
| Scaled address plus `0x44` | 32 bits | Selected address |

Captured values retain their sampled values even when their original memory locations are later overwritten.

### Final memory-derived chain and invocation

A 32-bit **child address** is read from context value plus `0x04`. A 16-bit `0x0065` is then written at twice the child address. Only afterward is a 32-bit **link sample** read from child address plus `0x64`.

The function writes the low byte of the secondary scaled sample at selected address plus `0x11`, then writes the 32-bit value 100 at selected address plus `0x54`.

It next captures a 32-bit **child sample** from child address plus `0x74`, followed by a 32-bit **final destination** from context value plus `0x64`.

The last writes before the indirect invocation are, in order:

| Destination | Width | Value |
|---|---:|---|
| Context value plus `0x09` | 8 bits | Low byte of child address |
| Final destination plus `0x54` | 32 bits | Link sample |
| Twice the link sample | 32 bits | Final destination |

The indirect dependency is then invoked with these four 32-bit arguments:

| Position | Value |
|---:|---|
| 1 | `0x00000072` |
| 2 | Link sample |
| 3 | Context sample |
| 4 | Child sample |

The visible body returns if the invocation returns. It does not explicitly consume a result from the dependency.

## Side effects

The directly established outputs are the ordered memory writes and the final indirect invocation.

All address expressions participate in one potentially overlapping address space. Different descriptive names or expressions do not establish different objects. Byte, 16-bit, and 32-bit accesses can overlap, and a write can affect a later value used as an address.

Consequences include:

- Captures reflect memory at their individual read points.
- Final memory contents cannot be inferred from an isolated write without considering subsequent overlaps.
- The repeated expanded-address writes may supply or alter the selected-address sample.
- Fresh reads from absolute address `0x67` must not be replaced with the earlier global sample without additional assumptions.
- Overwrites do not retroactively change values already captured.

The visible body contains no explicit allocation, release, ownership transfer, synchronization, logging, or error reporting. This does not establish those properties for the indirect dependency.

The represented order does not establish atomicity, barriers, volatile semantics, or visibility guarantees to concurrent observers.

An invalid access, arithmetic-related execution failure, or nonreturning dependency can prevent normal completion. No rollback or recovery path is visible. Earlier effects are not established to be undone.

## Dependencies

F-d64faa9db7b3 has one visible unresolved indirect dependency. No separate opaque function ID for that target is supplied; assigning it a resolved identity would exceed the evidence.

The invocation boundary is established by the four argument values and their order. The following dependency properties remain **documented_only**:

- Actual calling convention and return behavior.
- Whether arguments are interpreted as addresses.
- Reads or writes to globals and previously accessed storage.
- Allocation, retention, transfer, or release of memory.
- Changes to other execution state.

No post-call memory preservation or ownership guarantee follows from this contract.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes that the relevant operations are reached, arithmetic is defined for the values used, and all accessed memory is valid.

### Four-write destination trace

If the byte at scaled address plus `0x04` is `0xAB` when sampled, the four 32-bit writes at scaled address plus `0x20` carry, in order:

`0x20`, `0x2F`, `0xAB`, `0x20`.

A check of only the final value does not establish the required intermediate effects.

### Byte truncation

For a third argument of 141:

- Adjusted input is 64.
- Scaled address is 256.
- Expanded address is 512.

The repeated byte writes at anchor address plus `0x19` carry zero. The byte written at absolute address `0x3A` is `0x30`, because the corresponding mathematical expression is `0x230` and only its low byte is stored.

These are write values, not guarantees that later writes preserve those bytes.

### Coincident scaled and expanded destinations

For a third argument of 77, adjusted input, scaled address, and expanded address are zero.

Every repeated write at expanded address plus `0x54` consequently writes doubled context at absolute address `0x54`. The later selected-address read also accesses `0x54`.

If no write after the last such repeated write overlaps bytes `0x54` through `0x57`, and no external modification occurs, the selected address is doubled context. A specification requiring these two expressions to identify distinct locations would reject behavior established by this input.

For a third argument of 78, their numerical addresses instead evaluate to `0x58` and `0x5C`, respectively. Neither example establishes a general disjointness rule.

### Replacement of byte fields

With known byte order and no other overlapping writes, the replacement word stored at scaled address plus `0x10` determines the bytes at offsets `0x10` and `0x11`.

Preserving the earlier individual byte values instead of performing the 32-bit replacement would not match the represented behavior.

### A write affecting the subsequent link sample

For a child address of `0x64`, twice the child address and child address plus `0x64` both equal `0xC8`.

The 16-bit `0x0065` write therefore modifies the first two addressed bytes of the subsequent 32-bit link-sample read. The resulting link sample depends on byte order and the other two bytes; it is not the value from before that write.

### Indirect invocation boundary

Suppose the final chain reaches the invocation with:

- Link sample `0x1000`.
- Context sample `0x11223344`.
- Child sample `0x55667788`.
- Final destination `0x2000`.

Before invocation, the function writes `0x1000` at `0x2054`, then `0x2000` at `0x2000`. The dependency receives `0x72`, `0x1000`, `0x11223344`, and `0x55667788`, in that order.

These are pre-call observations, not promises that the dependency preserves those locations.

### Invalid access

If a reached destination is inaccessible, the evidence establishes no returned error, validation branch, or transactional recovery. An implementation contract promising such behavior would exceed the supplied evidence.

## Uncertainties

The following remain **documented_only**:

- Complete calling convention and provenance of additional incoming state.
- Accuracy and completeness of the represented function boundary.
- Application-level meaning of fields and constants.
- Exact repeated-group count as an enumerated acceptance requirement.
- Endianness, alignment, address conversion, and address-wrap behavior.
- Signed-overflow behavior.
- Signed right-shift behavior for negative operands and excessive counts.
- Memory permissions, mapping, and device-backed behavior.
- Ownership and lifetime of referenced storage.
- Indirect target identity, ABI, side effects, and return behavior.

An **inferred intent** is that the function updates interconnected memory fields before transferring control. This does not establish a list, tree, formatter, ownership protocol, or other application abstraction.

Aliasing between scaled and expanded destinations is not an unresolved possibility requiring a guessed answer: equality is established for the third argument 77. What remains input- and state-dependent is which accesses overlap in a particular execution and what values those overlaps produce.

## Evidence

The supplied static representation supports the access widths, constants, relative offsets, explicit absolute addresses, capture order, repeated write pattern, and final four-argument invocation described above.

The equality example follows directly from the arithmetic: subtracting 77 from 77 yields zero, and multiplication by either four or eight preserves zero. The repeated write destination and later selected-address source then both evaluate to `0x54`.

The supplied material also indicates overlapping global storage and unresolved indirect control flow. It does not establish independent global objects, a complete public interface, or downstream behavior.

No executable, resolved dependency body, calling context, memory map, runtime trace, or test result was supplied. No runtime tests have been performed.
