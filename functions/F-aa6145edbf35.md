<!-- Trusted provenance: F-aa6145edbf35; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aa6145edbf35

## Purpose

F-aa6145edbf35 performs an ordered mutation of a graph of memory objects, then transfers control through an unresolved indirect-call target.

The established behavior is low-level memory manipulation: byte, halfword, and word stores; pointer-like link traversal; propagation of previously read values; and a final call with three visible arguments. The available evidence does not establish a higher-level purpose such as initialization, serialization, allocation, or container maintenance.

Several writes are immediately superseded in the ordinary non-aliasing case. They remain part of the contract: overlapping objects, memory-mapped storage, faults, or external observation can make their order significant.

## Interface

### Explicit inputs and output

The visible interface contains:

| Input | Width | Established use |
|---|---:|---|
| First explicit input | 32 bits | No visible use |
| Second explicit input | 32 bits | No visible use |
| Third explicit input | 32 bits | Written to an ambient object, then potentially overwritten |
| Fourth explicit input | 8 bits | Written to an ambient object, then potentially overwritten |

There is no established return value. If the final indirect call returns normally, the visible routine reaches a return without producing an explicit result.

### Required ambient state

Four additional entry-state values act as memory bases. This contract names them **A**, **B**, **C**, and **D**. These are descriptive labels, not explicit interface parameters or claims about a recovered calling convention.

The routine also reads three fixed storage locations:

- **Global address value 1:** a 32-bit value used as an address component and halfword payload.
- **Global address value 2:** a 32-bit value used in address arithmetic and passed to the final call.
- **Global payload:** a 32-bit value written to a computed address.

The indirect target is another required ambient value. Its initialization and provenance are not established.

### Memory notation and widths

Offsets are byte offsets, written in hexadecimal. A word is 32 bits; a halfword is 16 bits; a byte is 8 bits.

“Low byte” and “low halfword” mean truncation to the least significant 8 or 16 bits of a value. This does not imply a particular byte order for word storage.

The labels used below for captured values refer to snapshots taken at the stated read, not continuing references to mutable fields. In particular, two reads from the same address are not interchangeable when intervening stores may alias it.

### Preconditions

For normal completion, all accessed addresses must support the relevant read or write widths, and the final indirect target must be callable under the actual machine ABI.

No null checks, bounds checks, graph validation, alignment checks, or arithmetic-overflow checks are visible. The necessary memory includes linked objects, additive combinations of addresses, doubled address-like values, and an address derived from a single byte. Ordinary valid heap objects alone are therefore not a sufficient precondition.

## Behavior

### Ordering model

The following stages preserve the order of the visible memory operations. They describe the recovered straight-line behavior, not a claim that the recovery is a complete or correct account of the original machine instructions.

Unless a stage explicitly uses a captured value, each field access is a fresh read. Aliasing is not excluded. A store through one base can change a subsequent field read through another base.

### 1. Establish initial links and capture an existing payload

The first effects are:

- Store the low byte of B at C+0x0d.
- Store the third explicit input as a word at C+0x54.
- Store A as a word at C+0x14.
- Capture the word at B+0x64 as the **initial payload**.

The routine then stores byte 0x74 at A+0x10, word 0x74 at B+0x54, and word 0x74 at A+0x10, in that order. The byte store and word store at A+0x10 are separate operations.

It reads a word from A+0x74 and retains that value as the **early linked object**. It then stores C at C+0x54 and word 0x74 at A+0x30.

Next it writes:

- Low byte of D at B+0x01.
- Low byte of the initial payload at D+0x09.
- The complete initial payload at B+0x74.
- Byte 0x74 at B+0x11.

It captures the word at A+0x60, writes the low byte of D at B+0x11, stores the initial payload at B+0x34, and writes the low byte of the captured A+0x60 value at A+0x10.

These accesses must not be collapsed into final field assignments. For example, the A+0x60 read occurs after the initial payload has been stored through B.

### 2. Mutate an existing B-linked object

The word at B+0x14 is read as a **first linked object**.

The routine stores the fourth explicit input at C+0x11, stores C at B+0x44, and writes the low byte of the initial payload at B+0x11. It then captures the current word at B+0x74 and writes the low byte of C at B+0x0d.

Through the first linked object, it stores word 0x66 at offset 0x10. It repeats the C store at B+0x44, stores byte 0x66 at the linked object’s offset 0x10, stores word 0x66 at B+0x54, and stores word 0x66 at the linked object’s offset 0x60.

It freshly reads B+0x74 as the **replacement object**, then stores C at B+0x74. At replacement-object offset 0x40 it writes word 0x66. At B+0x0d it writes the low byte of the previously captured B+0x74 value.

The replacement object is therefore determined before the B+0x74 replacement, even when the earlier B+0x74 contents resemble an integer payload rather than a valid address.

### 3. Propagate links and byte-sized address fragments

The routine captures replacement-object offset 0x44. It stores B at B+0x54 and stores the early linked object’s own address at that object’s offset 0x54.

It reads replacement-object offset 0x14 as a destination object, then writes:

- Low byte of the captured replacement-object offset 0x44 at C+0x11.
- Low byte of a fresh word read from C+0x14 at destination-object offset 0x15.

It freshly reads C+0x14 as a **current root**, captures B+0x64 as a **payload snapshot**, and writes word 0x73 at current-root offset 0x20.

It then reads B+0x64 again as a separate **second payload snapshot** and writes its low byte at current-root offset 0x09. Those two payload snapshots are not assumed equal.

### 4. Traverse the current root and preserve a self-assignment

The word at current-root offset 0x74 selects the **working object**.

The routine writes the low byte of B at working-object offset 0x0d. It then reads the word at working-object offset 0x54 and writes that same captured word back to the same field. This is a real visible read followed by a write, not an omitted no-op.

It writes byte 0x74 at working-object offset 0x09, stores the payload snapshot at B+0x44, and stores B at working-object offset 0x34.

Working-object offset 0x04 selects a **child object**. The routine stores the complete second payload snapshot at child-object offset 0x24 and writes the low byte of the payload snapshot at B+0x19.

### 5. Rewire through fresh link reads

The routine captures child-object offset 0x74 as a **retained link**, and child-object offset 0x04 as a **replacement payload**.

A fresh read of B+0x74 selects a **branch object**. Through its offset 0x14, the routine selects another object and stores the retained link at that object’s offset 0x54.

It then reads branch-object offset 0x04 as a **branch child**. Two consecutive byte stores write the low byte of the second payload snapshot at B+0x0d. Both stores are present.

The replacement payload is stored at B+0x64, followed by word 0x74 at branch-child offset 0x30.

The routine freshly traverses branch-object offset 0x04 and then offset 0x74 to select an **active object**. This traversal is not defined to reuse the previously captured branch child.

### 6. Replace B’s links using separately captured graph state

A fresh word read at B+0x64 supplies a low byte written at B+0x1d.

The routine follows B+0x44 and then offset 0x14, capturing a **new B link**. It stores the retained link at active-object offset 0x34 and stores the new B link at B+0x14.

It captures B+0x54 as a **saved B value**, then writes the low byte of B at new-B-link offset 0x09.

Following active-object offset 0x04 and then offset 0x74 selects a **downstream object**. The routine writes the low byte of the retained link at downstream-object offset 0x15, stores B at B+0x54, writes the low byte of the active object at B+0x11, and writes the low byte of the saved B value at downstream-object offset 0x11.

It captures downstream-object offset 0x64 as a **downstream payload**, then writes at B+0x09 the low byte captured earlier from B+0x64.

### 7. Update an active child and prepare the final graph traversal

A fresh read of active-object offset 0x04 selects an **active child**.

The routine writes the low byte of B at active-child offset 0x09, captures the word at active-child offset 0x50, writes the low byte of the downstream payload at active-child offset 0x11, and repeats the byte store of B at active-child offset 0x09.

It stores the downstream payload at B+0x14, writes the low byte of the active child’s address at B+0x0d, and writes the low byte of the captured active-child offset 0x50 at active-child offset zero.

It then reads active-object offset 0x74 as a **byte-address component**, and active-object offset 0x14 as a **final parent**. It stores the active child’s own address at active-child offset 0x54.

Final-parent offset 0x04 selects the **final child**.

### 8. Write the final fixed constants

The routine performs these effects in order:

- Byte 0x64 at final-child offset 0x10.
- Word 0x64 at B+0x54.
- Word 0x64 at final-child offset 0x40.
- Low byte of the downstream payload at B+0x0d.
- Low byte of the final child’s address at final-parent offset 0x09.
- Complete downstream payload at B+0x24.
- Low byte of B at final-parent offset 0x11.

The numeric value 0x64 is both an offset used elsewhere and a stored constant here; those roles are distinct.

### 9. Perform computed-address halfword writes and derive a byte-based address

The routine reads global address value 1, then freshly reads B+0x74 as the **terminal link**.

It writes the low halfword of the terminal link to the address formed by terminal link plus B. It then writes the low halfword of global address value 1 to the address formed by twice the final child’s address.

Next it reads one byte from the address formed by byte-address component plus B. This byte is converted through a plain-character interpretation and then to a 32-bit unsigned value, producing the **derived address value**.

After that read, it writes:

- Low halfword of the terminal link at global address value 1 plus terminal link.
- Low halfword of the terminal link again at terminal link plus B.
- Word 0x45 at derived address value plus 0x45.

The two writes at terminal link plus B bracket other accesses. Neither is removable from the observable contract.

### 10. Finish the computed writes and invoke the indirect dependency

The routine reads global address value 2, then the global payload, then B+0x44 as the **outgoing payload**.

It writes byte 0x44 at derived address value plus 0x44. It stores the global payload as a word at twice global address value 2. It writes the low halfword of global address value 2 at retained link plus global address value 2.

The final indirect call has these three visible argument values, in order:

1. Derived address value bitwise-OR 0x44.
2. The captured global address value 2.
3. The outgoing payload captured from B+0x44.

The first argument is not the address of the preceding word store: bitwise OR with 0x44 differs from addition of 0x45.

## Side effects

The routine modifies caller-reachable and ambient memory without a rollback mechanism. A fault or non-returning dependency can leave any already executed prefix of writes visible.

Important overlapping fields include:

| Location | Visible kinds of mutation |
|---|---|
| B+0x0d | Repeated byte fragments from addresses and payloads |
| B+0x11 | Constant, address fragments, and payload fragments |
| B+0x14 | Word replacement from different graph-derived values |
| B+0x44 | C, later a captured payload; finally read for the outgoing call |
| B+0x54 | Constants and self-address stores |
| B+0x64 | Read repeatedly, then replaced |
| B+0x74 | Initial payload, later C, subsequently reread |
| C+0x11 | Explicit byte input, later a linked-field fragment |
| C+0x54 | Third explicit input, later C |

These are access descriptions, not guaranteed final values. Aliasing and the final dependency can change the eventual contents.

No allocation, deallocation, reference-count operation, ownership transfer, or synchronization operation is explicitly established. Storing an address does not establish ownership of the referenced storage.

## Dependencies

There is one unresolved indirect dependency, invoked after the visible memory mutations.

No opaque dependency ID was supplied for that target. Assigning it a known function identity would be unsupported.

Established dependency behavior is limited to the three visible argument expressions and its position after the stores. The following remain unknown:

- Calling convention and any additional implicit inputs.
- Whether it returns.
- Whether it reads, modifies, retains, or releases referenced memory.
- Whether it changes globals or ambient state.
- Whether it reports failure through an unobserved channel.

The contract does not assume preservation of globals, memory, or ownership across that call.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Overwritten explicit word input.** If C+0x54 does not alias intervening reads or stores, and memory operations complete normally, that field receives the third explicit input before receiving C. An implementation that omits the first store does not preserve the full access behavior.

2. **Separate payload reads.** If the store of 0x73 through the current root changes B+0x64 by aliasing, the payload snapshot and second payload snapshot can differ. Later uses must retain their respective captured values.

3. **Repeated stores remain observable.** With storage that observes individual writes, B+0x0d receives two consecutive identical byte stores during the rewiring stage. Combining them into one is not equivalent under that observation model.

4. **Width-sensitive overlap.** A word store of 0x74 at A+0x10 affects four bytes; the later low-byte store at that offset affects only one. Exact resulting bytes depend on byte order and any other aliasing writes.

5. **Byte conversion below 0x80.** If the computed byte read yields 0x20, the derived address value is 0x20 under either signed or unsigned plain-character interpretation. The word-store address is then 0x65, the byte-store address is 0x64, and the first outgoing argument is 0x64.

6. **Byte conversion at or above 0x80.** If that byte is 0x80, a signed 8-bit interpretation produces unsigned value 0xffffff80; an unsigned interpretation produces 0x00000080. Acceptance of either concrete address set requires external evidence resolving character signedness.

7. **Failure partway through traversal.** If a required link is inaccessible, normal completion is not required. Earlier successful writes are not specified to be undone, and no particular error code is established.

8. **Returning indirect target.** If every access succeeds and the indirect target returns, the routine reaches its visible return. No explicit result value or preservation of the mutated graph is promised.

## Uncertainties

### Established static evidence

The supplied representation establishes the visible access widths, offsets, constants, read/write ordering, captured-value reuse, and final three-argument indirect invocation described above.

It presents a straight-line path with no explicit validation or conditional branch before the indirect call.

### Inferred intent

Repeated self-address stores and linked-field traversals suggest graph-oriented state mutation. That observation is not sufficient to assign object types, field meanings, invariants, or ownership rules.

The unusual address calculations and repeated constant stores also make a trustworthy high-level interpretation uncertain.

### documented_only unknowns

- The true entry convention supplying A, B, C, D, and the indirect target.
- The layout, extent, alignment, and lifetime of every referenced object.
- Byte order.
- Plain-character signedness and the exact byte-extension instruction.
- Whether address addition and doubling wrap modulo \(2^{32}\), trap, or reflect an imperfect recovered expression. The visible operands are 32-bit; portable signed-overflow semantics must not be assumed.
- Actual fault behavior, memory-mapped effects, concurrency, and external observers.
- The initial values and mutability of the three fixed storage locations.
- Whether the indirect transfer is semantically a returning call or a continuation transfer.
- Any behavior of that dependency beyond the visible invocation.

These uncertainties are not resolved by choosing convenient heap layouts or assuming disjoint objects.

## Evidence

This contract is based solely on the supplied static representation for F-aa6145edbf35. It distinguishes explicit 8-, 16-, and 32-bit accesses and preserves the ordering relevant to aliasing, repeated writes, and captured values.

The representation itself reports uncertainty in recovering the final indirect control transfer. Accordingly, the contract does not claim a resolved target or complete calling convention.

No implementation was accessed, no runtime tests were performed, and no acceptance example was executed.
