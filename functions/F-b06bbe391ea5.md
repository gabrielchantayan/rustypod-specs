<!-- Trusted provenance: F-b06bbe391ea5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b06bbe391ea5

## Purpose

F-b06bbe391ea5 performs a context-dependent sequence of indirect dependency invocations, followed by ordered memory mutations and a final four-argument indirect transfer.

The supplied static representation establishes no application-level purpose. Constants resembling characters, repeated offsets, and address propagation do not establish formatting, serialization, initialization, allocation, or container maintenance.

This contract distinguishes:

- **Established static evidence:** represented invocation relationships, access widths, arithmetic transformations, and temporal relationships.
- **[INFERENCE]:** conclusions conditional on an additional machine or memory model.
- **documented_only:** behavior or detail not resolved by this contract.

All downstream behavior is conditional on execution reaching it. Invalid accesses, dependency failure, non-returning dependencies, or an incompatible indirect destination can prevent completion.

## Interface

### Entry context

There are no represented explicit parameters. Implicit inputs include:

- An incoming 32-bit register value used twice in the first invocation.
- Another incoming register value used in shifts by nine bits.
- Incoming link context used to derive the first indirect destination.
- Primary and secondary result words supplied by dependencies.
- Global storage and memory reached through computed addresses.

The absence of explicit parameters does not establish a context-free entry point. A caller must supply the required execution context through an external convention; that convention is documented_only.

### Outputs and completion

No explicit return value is represented. Observable outputs comprise dependency invocations, memory reads and writes, and the final indirect invocation.

The final invocation receives four represented argument words. If it returns normally, the function has no further explicit work before returning. Neither the identity of that destination nor its return behavior is established.

### Widths and layout

| Category | Represented width |
|---|---:|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Word access | 32 bits |
| Combined dependency result | 64 bits |
| Address-bearing arithmetic | 32 bits |

Combined results are commonly divided into their low and high 32-bit words. Some extractions instead shift the 64-bit result right by 34, 36, or 44 bits before narrowing. These discard an additional two, four, or twelve bits relative to a high-word extraction.

Byte stores retain the low eight bits. Explicit widening of a byte to a word is zero-extension. A halfword store affects sixteen bits, not one byte.

Repeated offsets describe access locations, not a proven object layout. The representation uses bytes at offsets including 1, 4, 5, 8, 9, 0x0D, 0x10, 0x11, 0x15, and 0x19, and words at offsets including 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. Different accesses need not share an object or a stable base.

### Preconditions and arithmetic limits

Completion requires every reached destination to be executable and compatible with the live calling context. Every reached memory access must be valid for its width. Low absolute addresses must be accessible where used. No visible validation establishes these conditions.

Unsigned 32-bit transformations must preserve intermediate truncation. Signed shifts, signed multiplication overflow, pointer conversion, alignment tolerance, and address overflow require machine-model information not supplied here.

**[INFERENCE]** Under a wrapping 32-bit interpretation, shifting left by twenty-five bits and then seven bits yields zero. Performing those transformations in an unbounded intermediate width would not preserve the represented word behavior.

## Behavior

### Dependency-driven prefix

The prefix is a long serial chain of indirect invocations. The first destination is derived from incoming link context plus 0xFFE, or 4,094 bytes. Subsequent destinations are formed from embedded continuation-like values or address-bearing references using that displacement.

The first invocation receives equal first and second argument words, each formed by shifting the incoming value left seven bits. Later invocations consume transformed primary results, secondary result words, explicit zeros, and values retained across intervening calls.

Established properties include:

- Invocation order remains significant even when no explicit arguments appear.
- A combined result can contribute separate low-word and high-word arguments.
- Some argument positions receive identical transformed values.
- Some results remain live across several invocations.
- There is no visible status-based retry, recovery, or early-return branch around the chain.

A call with no represented arguments is not established to consume no register state. An invocation with no retained visible result is not established to be redundant.

Transformations include left shifts, logical right shifts, and mask-and-shift extraction. Masks include widths of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits. Masking before shifting matters: a 20-bit quantity shifted right ten bits has at most ten significant bits; a 24-bit quantity shifted right twenty bits has at most four.

The exact invocation inventory is not enumerated here. Dependency identities and their external contracts remain documented_only.

### Opening memory phase

Several retained dependency results become addresses, indices, or payloads. The following descriptive terms identify their roles without asserting object types:

- **Seed base:** an earlier low result shifted left twenty-five bits.
- **Wide base:** another retained result multiplied by 2²⁵.
- **Shifted payload:** an earlier result shifted left eight bits.
- **Small index:** a dependency result shifted left eight bits, masked with 0x7FFF, then logically shifted right six bits.
- **Small base:** twice the small index.

The small index ranges from 0 through 508 in multiples of four. The small base ranges from 0 through 1,016 in multiples of eight.

The opening effects have an established temporal boundary: a word containing the seed base is written at the final combined result’s high word plus 0x14; the low byte of that high word is written at seed base plus 9; the wide base is written at the final low word scaled by 256 plus 0x54; and seed base plus 9 is overwritten with the low byte of the seed base.

These two byte writes remain distinct effects.

Next, the word at wide base plus 0x14 is cleared. A byte is read from wide base plus 5 before the first clear at small base plus 0x14. The captured byte is then zero-extended into the word at wide base plus 0x34.

The function reads wide base plus 0x14, clears small base plus 0x14 again, and twice writes the low byte of the captured word at small base plus 0x15. The captured word is subsequently used as an address-bearing value.

**[INFERENCE]** In ordinary memory without an intervening overlapping effect, reading the cleared wide-base word yields zero. Aliasing or external memory behavior can invalidate that conclusion.

The next absolute read obtains a byte from address 5 and widens it to a base value. A byte store to absolute address 1 receives the low byte of the small base. Subsequent effects include a small-index byte store through the byte-derived base, two different byte writes at wide base plus 0x11, and a word write through the captured cleared-word value.

### Saved address and intermediate propagation

A word is read from the byte-derived base plus 0x54 before that location is replaced. This captured word is the **saved address** used in later scaling and self-address stores. It is a value snapshot, not a requirement to reload the overwritten field.

The intermediate region follows offset-0x74 words, writes through the resulting addresses, repeats self-address stores at offset 0x24, and writes low address bytes at offset 9. Some offset-0x74 accesses are repeated rather than universally replaced by one retained value. No general disjointness assumption is justified.

Two absolute updates are part of this region:

- **Absolute word address 0x54** receives the saved address shifted left one bit. This occurs after a preceding write at small base plus 0x14 and before two writes there containing the saved address.
- **Absolute byte address 0x15** receives the low byte of the small base. It occurs after two byte writes at small base plus 0x15 containing the low byte of four times the saved address.

Thus, the derived stores at small base plus 0x15 and the absolute store at 0x15 have different data sources and distinct temporal positions. They can coincide when the small base is zero.

Further effects use four and eight times the saved address, double loaded words, follow byte-derived bases through offset 0x74, and update offsets 0x0D, 0x14, 0x44, and 0x54. These include self-address word stores and overwrites of previously stored words.

A later word obtained through a byte-derived offset-0x74 access supplies a **constant-region source**. Twice that word becomes the **terminal base**. The surrounding stores can affect later reads through aliases; they do not establish a linked-data-structure invariant.

### Constant stores and captured global base

The constant region contains three distinct stores of 0x3A:

| Width | Target |
|---|---|
| 8 bits | Four times the constant-region source |
| 16 bits | Eight times that source |
| 32 bits | Four times that source, plus 0x20 |

The byte store precedes the halfword store, which precedes the word store.

An **adjusted value** is formed by subtracting 0x96, or 150, from four times the saved address. Four times this adjusted value becomes the **adjusted base**.

At adjusted base plus 0x20, the function performs four ordered word stores:

1. 0x20.
2. 0x2F.
3. A zero-extended byte newly read from adjusted base plus 4.
4. 0x20.

The byte read belongs between the second and third stores. Its value is not an entry snapshot and can reflect preceding overlapping writes.

**The globally stored base is captured before the absolute word store of 0x20 at address 0x60.** The subsequent repeated region uses that captured global base. If the absolute store overlaps the storage supplying the global base, the memory containing the base may change, but the already captured base is not replaced by a reload.

### Repeated-store region

The unrolled region repeatedly performs this ordered four-effect pattern:

- A word store at a scaled address plus 0x54, containing four times the constant-region source.
- A byte store at captured global base plus 0x19, containing the low byte of the adjusted base.
- A byte store at adjusted base plus 0x11, containing the low byte of the captured global base.
- A word store at captured global base plus 0x34, containing the scaled address.

The first group uses eight times the saved address as its scaled address. Subsequent groups use eight times the adjusted value.

These are finite, explicit repetitions, not a visible condition-controlled loop. Collapsing the region to final memory values loses represented ordering and multiplicity. The exact repetition count is documented_only in this contract; this is an inventory limitation, not evidence of a dynamically unknown bound.

### Terminal captures and absolute-byte update

After the repeated region, four reads and writes establish important snapshots:

- A 32-bit word is captured from absolute address 0xA9.
- A word is captured from four times the constant-region source plus 0x74.
- A byte is written to absolute address 0x3A.
- A word is captured from terminal base plus 0x14, followed by a word captured from adjusted base plus 4.

The **absolute 0x3A byte store** therefore occurs after both the absolute 0xA9 word read and the following offset-0x74 read, but before the retained offset-0x14 and offset-4 reads.

Its value is obtained by narrowing the adjusted value to a byte, multiplying that byte-sized contribution by eight, adding 0x30, and retaining the low eight bits for storage. It is not another store of constant 0x3A.

The retained word from terminal base plus 0x14 supplies the final invocation’s third argument and is also written back to terminal base plus 0x54 later.

### Absolute 0x67 and position-sensitive reads

The low byte of the captured absolute 0xA9 word is written to absolute address 0x67. That write does not establish one retained byte for all later uses.

There are three subsequent byte reads of absolute 0x67 at separate positions:

- The copy to absolute address 0x21 reads it after a word read at terminal base plus 0x44.
- The first later store to adjusted base plus 9 reads it after intervening word and byte mutations, including a word copy into adjusted base plus 0x10.
- The second later store to adjusted base plus 9 reads it after another derived byte store, a word store at adjusted base plus 0x44, and a word read at adjusted base plus 0x54.

Each read must reflect storage at its own position. Intervening derived stores may overlap absolute 0x67. Replacing these reads with the byte originally written there is not justified without additional non-aliasing and memory-behavior assumptions.

Other terminal mutations include byte 0x6C at adjusted base plus 8, address-byte copies, a self-address word at terminal base plus 0x54, and a word copied from adjusted base plus 0x40 to adjusted base plus 0x10.

A byte written at terminal base plus 1 comes from a signed right-shift expression whose count is the low eight bits of the adjusted base. Rules for negative operands and counts of 32 or more remain documented_only.

### Final argument capture and transfer

A word loaded from terminal base plus 4 becomes the **handoff base**. A halfword containing 0x65 is stored at twice that value. A word is then captured from handoff base plus 0x64; this supplies the final invocation’s second argument.

After further byte and word stores, including a word value of decimal 100 at a derived offset 0x54, the fourth argument is captured from handoff base plus 0x74.

The fourth-argument capture precedes:

- A word read from terminal base plus 0x64.
- A byte store at terminal base plus 9 containing the low byte of the handoff base.
- A word store at the newly loaded base plus 0x54 containing the retained second argument.
- A word store at twice the retained second argument containing the newly loaded base.

The final invocation receives, in order:

| Position | Value |
|---|---|
| First | 0x72 |
| Second | Captured word from handoff base plus 0x64 |
| Third | Earlier captured word from terminal base plus 0x14 |
| Fourth | Captured word from handoff base plus 0x74 |

Neither the second nor fourth argument is reloaded at invocation. Their captured values survive the three final stores even if those stores overwrite their original source locations. The later read from terminal base plus 0x64 does not replace the second argument.

## Side effects

Direct effects include byte, halfword, and word stores to derived and absolute locations. Absolute accesses include byte reads at 5 and 0x67, word reads at 0xA9, byte writes at 1, 0x15, 0x21, 0x3A, and 0x67, and word writes at 0x54 and 0x60.

Different storage labels do not establish disjointness. Overlap must be considered from access addresses and widths, independently of comments. A word store at offset 0x10 can overwrite byte locations at offsets 0x10 and 0x11; the resulting bytes depend on endianness.

No allocation, release, ownership transfer, synchronization, atomicity, rollback, or reentrancy guarantee is established. Dependencies may retain, release, invalidate, or mutate referenced storage. No assumption that they preserve globals or memory lifetime is permitted.

There is no represented error result, bounds check, null check, diagnostic path, or rollback. If execution fails after prior writes, those effects have already occurred. The platform’s trap or exception behavior is documented_only.

## Dependencies

Only F-b06bbe391ea5 has a resolved opaque identity in the supplied map.

| Opaque ID | Interaction | Unresolved contract |
|---|---|---|
| F-unknown-01 | Ordered prefix dependency set; transformed arguments and primary/secondary result propagation | Individual identities, arities, calling convention, memory effects, ownership, failure, return behavior |
| F-unknown-02 | Terminal indirect destination receiving four represented arguments | Destination selection, application meaning, ownership, memory effects, return behavior |

F-unknown-01 denotes a set of unresolved destinations, not one proven shared function. Neither dependency ID implies purity or preservation.

## Acceptance examples

These are conditional criteria, not executed tests.

### Entry duplication and index boundary

Under wrapping 32-bit shifts, incoming value 1 produces first-invocation argument words of 128 and 128.

An all-ones result used for the small-index transformation produces index 508 and small base 1,016, not an unrestricted 15-bit index.

### Absolute updates

If the saved address is 0x1234 and the relevant arithmetic is valid, absolute word address 0x54 receives 0x2468 at its specified position.

For small index 508, absolute byte address 0x15 receives 0xF8, the low byte of 1,016.

For an adjusted value whose low byte is 0x25, the terminal absolute 0x3A byte store receives 0x58: eight times 0x25 plus 0x30, narrowed to eight bits.

### Global-base snapshot

If the global-base read yields address P and the subsequent absolute 0x60 word store changes the storage that supplied P, repeated-region accesses still use captured P. Reloading the changed storage would violate the capture timing.

### Independent reads at 0x67

If an intervening aliased store changes absolute 0x67 after its initial write but before the copy to 0x21, that copy uses the changed byte. The two later reads supplying adjusted-base offset-9 stores independently use their then-current bytes.

### Final snapshots

If the second and fourth argument captures yield 0x11223344 and 0x55667788, those argument values remain unchanged even when the final stores overlap their source fields. The final invocation uses those snapshots, the retained third argument, and first argument 0x72.

### Width and ordering

A widened byte 0xE1 produces word value 225. The mixed-width 0x3A stores retain their respective widths. Observable storage must distinguish the four offset-0x20 stores rather than only their final value.

## Uncertainties

The following remain documented_only:

- Application purpose and complete entry convention.
- Dependency identities, preservation rules, ownership, and lifetime effects.
- Exact signed-overflow, large-shift, pointer-conversion, and address-wrap behavior.
- Endianness, alignment, permissions, object boundaries, and alias relationships.
- Complete prefix invocation inventory and exact repeated-region multiplicity within this contract.
- Terminal destination selection and whether it returns.
- Platform behavior after invalid execution.

The arithmetic and low-address accesses do not independently prove corruption, deliberate obfuscation, or unreachable execution.

## Evidence

This contract is based solely on the supplied static representation. Prior prose, reviewer feedback, and embedded comments are not independent behavioral authority.

Established claims derive from represented widths, transformations, value captures, and ordering. In particular, the representation places the global-base capture before the absolute 0x60 store, distinguishes the three later reads of absolute 0x67, and captures the final second and fourth arguments before the final stores.

No runtime execution, dependency inspection, smoke checks, or tests have been performed. No successful completion, memory-safety guarantee, application interpretation, or provenance hash is asserted.
