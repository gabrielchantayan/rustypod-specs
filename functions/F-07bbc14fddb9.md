<!-- Trusted provenance: F-07bbc14fddb9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-07bbc14fddb9

## Purpose

F-07bbc14fddb9 performs a sequence of width-sensitive memory reads and writes, derives further addresses from the values read, and invokes an unresolved indirect target with three computed arguments.

The established behavior is a memory-access and control-transfer sequence. The supplied representation does not establish an application-level purpose, a data-structure interpretation, or a conventional public calling interface. In particular, several required inputs are implicit machine-state values rather than declared parameters.

This contract describes the visible sequence without assuming that the addresses are valid, that the accessed regions are disjoint, or that the indirect target preserves state.

## Interface

### Explicit inputs

| Input | Width and interpretation | Use |
|---|---|---|
| Address displacement | Signed 32-bit integer | Contributes to the address of the first 16-bit read. |
| Payload | Raw 16-bit value | Written to two implicitly supplied destinations. |

The payload has no established signed interpretation.

### Implicit inputs

The operation also requires the following entry-state values:

- **Primary address:** a pointer used as a 16-bit write destination, as part of the first read address, and as the source of two truncated pointer-derived values.
- **Secondary destination:** a pointer receiving three successive 16-bit writes.
- **First address root:** a 32-bit address used in a byte lookup and a later signed 16-bit read.
- **Second address root:** a 32-bit address used for a 16-bit write at displacement 55.
- **Indirect target:** the destination of the final indirect invocation.
- **Shared word:** a separately addressed 32-bit memory location read twice during the later part of the operation.

The explicit parameter list does not explain how these implicit values are supplied. Their validity is a precondition for any execution expected to complete normally.

### Outputs and completion

There is no established return value. The visible result consists of memory writes and the final indirect invocation.

The supplied representation shows a return after that invocation. Whether the target actually returns, transfers control elsewhere, or terminates execution is unknown.

### Width and address conventions

The representation uses:

- 8-bit unsigned byte reads and one 8-bit write;
- 16-bit signed or unsigned reads, depending on the access;
- 16-bit writes that retain the low 16 bits of their source;
- 32-bit signed and unsigned arithmetic;
- 32-bit word reads and writes;
- pointer arithmetic represented through 32-bit integers.

The descriptions below use 32-bit address bit patterns. “Signed expansion” means extending a signed value to 32 bits before using that pattern.

Calculations involving unsigned 32-bit values wrap modulo \(2^{32}\). Where the representation instead uses signed 32-bit addition, behavior on mathematical overflow is not established independently of the missing machine-level context.

## Behavior

### 1. Initial read and destination writes

The first read is a signed 16-bit value at:

**primary address + address displacement + 52**.

Its signed expansion becomes an address used shortly afterward. The read occurs before the primary destination is overwritten.

The following writes then occur in the displayed order:

1. Write the 16-bit payload at the primary address.
2. Write the low 16 bits of the primary address at the address obtained by signed expansion of the first read.
3. Write the 16-bit value 35 at the secondary destination.
4. Write the low 16 bits of **primary address − 65** at **second address root + 55**.
5. Write the 16-bit value 35 at the secondary destination again.
6. Write the 16-bit payload at the secondary destination.

The first read therefore supplies an address, not an offset from the primary address.

The two writes of 35 are separate operations in the supplied sequence. They must not be collapsed into a single event when describing observable access order. Likewise, the final payload write does not make the preceding writes irrelevant where memory is externally observable or aliases another accessed location.

There is no visible null check, bounds check, alignment check, or permission check.

### 2. Initial byte lookup and mixed address calculation

Read an unsigned byte at **first address root + 24**.

Treat that byte as an absolute address contribution: read another unsigned byte at **the first byte’s value + 24**. There is no addition of the first address root in this second lookup.

Subtract 119 from the second byte, then apply a signed arithmetic right shift by 20 positions. Under the signed-shift interpretation shown in the representation:

- A second byte from 0 through 118 produces −1, represented as unsigned 32-bit `0xFFFFFFFF`.
- A second byte from 119 through 255 produces zero.

Call this result the **shift-derived address**.

Read an unsigned byte at **shift-derived address + 16**. This becomes the **selector byte**.

Compute the **mixed address value** by subtracting 59 from the shift-derived address using unsigned 32-bit arithmetic, then applying bitwise exclusive-or with the zero-expanded selector byte.

Read and retain an unsigned byte at **selector byte + 17**. This retained value is used later as an address displacement.

### 3. First computed word write

Multiply the selector byte by 4096. Because the selector is at most 255, this multiplication fits within 32 bits.

Using that product:

1. Read an unsigned byte at **product + 16**.
2. Read another unsigned byte at **the preceding byte’s value + 16**.
3. Write the 32-bit mixed address value at **the second byte’s value + mixed address value**.

Both intermediate reads produce absolute, low-valued address contributions. They are not established as indices into an array rooted at the product.

This word write precedes the next signed 16-bit read. If its destination overlaps a later source, the later read observes memory after this write, subject to the platform’s memory semantics.

### 4. Signed displacement and byte-derived address

Read a signed 16-bit displacement at:

**first address root + mixed address value**.

Retain this displacement for a later 32-bit read.

Separately:

1. Read an unsigned 16-bit value at **selector byte × 4096 + mixed address value**.
2. Read an unsigned byte at **that unsigned 16-bit value + 17**.

Call the resulting byte the **byte-derived address**. Its numeric range is 0 through 255.

Read a signed 16-bit value at:

**byte-derived address + retained byte from selector byte + 17**.

Expand this signed value to a 32-bit unsigned bit pattern. Call that pattern the **expanded halfword**. Its possible forms are:

- `0x00000000` through `0x00007FFF` for nonnegative reads;
- `0xFFFF8000` through `0xFFFFFFFF` for negative reads.

Write this complete 32-bit pattern at **byte-derived address + 4**.

The signed read is significant: a negative 16-bit value does not become a zero-expanded positive integer.

### 5. Shared-word snapshot and adjusted address

After the preceding word write, read and retain the 32-bit shared word. This is the **shared-word snapshot**.

Next, read a signed 32-bit word at:

**expanded halfword + signed expansion of the retained displacement**.

Derive the **adjusted address** by adding:

- that signed word;
- that same word arithmetically shifted right by 12;
- 53;
- one if the byte-derived address is greater than 60, otherwise zero.

The threshold comparison is strictly greater than 60. Equality does not add the extra one.

The representation performs this calculation in 32-bit quantities. Exact behavior where signed additions overflow remains dependent on the underlying instruction semantics.

### 6. Final lookup values

Read an unsigned 16-bit value at:

**adjusted address + logical right shift of expanded halfword by 16**.

Call the result the **final halfword**.

Because the expanded halfword originated from a signed 16-bit read, its logical right shift by 16 is either:

- zero, for an originally nonnegative value; or
- 65,535, for an originally negative value.

It is not an arbitrary 16-bit table index.

Then perform these reads in order:

1. Read and retain an unsigned byte at **adjusted address + 9**. This is the **argument byte**.
2. Read an unsigned byte at **adjusted address + final halfword**. This is the **writeback byte**.
3. Read a signed 16-bit value at **byte-derived address − 61 + final halfword**. This is the **final signed halfword**.

The address expression for the last read includes an unsigned 32-bit subtraction. Values below 61 therefore participate in unsigned wrapping arithmetic before the final address is formed.

### 7. Final writes and indirect invocation

For the next word-write destination, perform a fresh lookup:

1. Read an unsigned byte at **adjusted address + 9** again.
2. Read an unsigned byte at **that newly read byte’s value + 9**.
3. Use **the second byte’s value + 4** as the 32-bit write destination.

Read the shared word again and write that value to the computed destination.

This is not specified as a write of the earlier shared-word snapshot. The source location is read again. The representation does not establish the relative evaluation order between this shared-word read and the independent reads used to calculate the destination; it does establish that the completed word write precedes the following byte write.

Write the writeback byte as an 8-bit value at:

**writeback byte + 12**.

Finally invoke the unresolved target with three arguments, in this order:

1. The final signed halfword, signed-expanded to 32 bits.
2. The earlier shared-word snapshot plus 51.
3. The argument byte minus 61, evaluated as a signed integer.

The third argument ranges from −61 through 194. It does not wrap to an 8-bit result.

The second argument uses the earlier snapshot, even if the shared word was changed by an intervening aliased write. Its overflow behavior is subject to the signed-arithmetic uncertainty described above.

## Side effects

The supplied sequence contains ten explicit memory writes before the indirect invocation:

- six 16-bit writes;
- three 32-bit writes;
- one 8-bit write.

These counts describe the supplied operations, not verified bus transactions or guaranteed externally visible events on a particular platform.

Writes may overlap one another, later read sources, the shared word, or other state. No non-aliasing guarantee is present. Access width is part of the behavior: the final 8-bit write must not be described as a word replacement, and the 32-bit write of the expanded halfword is not merely a 16-bit store.

No allocation, deallocation, ownership transfer, lock operation, or explicit error report is visible before the indirect invocation.

Access failures have no visible local recovery path. The source does not establish whether an invalid or misaligned access faults, is accepted by the target platform, or has device-specific effects. No rollback is shown; writes already completed before a failure are not explicitly undone.

## Dependencies

### F-UNKNOWN-01

This is a documentary opaque identifier for the unresolved indirect target, not a recovered identity.

Established interface evidence is limited to an invocation with the three arguments described above. Unknown properties include:

- calling convention and true parameter types;
- target-selection mechanism;
- return value, if any;
- whether control returns;
- additional implicit inputs;
- memory reads and writes;
- preservation of shared state;
- allocation, release, or ownership behavior;
- error or exception behavior.

No preservation or ownership assumption about F-UNKNOWN-01 is part of this contract.

The shared word and address roots are memory or machine-state dependencies, not additional identified functions.

## Acceptance examples

These examples are conditional consequences of the supplied representation. They are not runtime test results. They assume valid accesses and the stated arithmetic interpretation.

### Initial write order with distinct destinations

If the initial read returns signed value 256, the primary address has low 16 bits `0x1234`, and the payload is `0xABCD`, then:

- the primary destination receives `0xABCD`;
- address 256 receives the 16-bit value `0x1234`;
- the secondary destination receives 35, then 35 again, then `0xABCD`.

If those destinations overlap, this example does not establish their final contents without accounting for the intervening writes.

### Initial signed-address interpretation

If the first 16-bit read has bit pattern `0xFFFF`, its signed value is −1. The pointer-derived halfword write uses the 32-bit address pattern `0xFFFFFFFF`, not address 65,535.

### Shift boundary

If the second lookup byte is 118, the shift-derived address is `0xFFFFFFFF`, and adding 16 produces address 15 under unsigned 32-bit arithmetic.

If that byte is 119, the shift-derived address is zero, so the next byte is read at address 16.

### Negative expanded halfword

If the signed read used to form the expanded halfword returns −2:

- the following 32-bit write stores `0xFFFFFFFE`;
- the later logical shift by 16 produces 65,535;
- the final-halfword lookup uses **adjusted address + 65,535**.

Zero-expanding the original `0xFFFE` would produce different behavior.

### Threshold behavior

Holding the other inputs to the adjusted-address calculation equal:

- byte-derived address 60 contributes no extra increment;
- byte-derived address 61 contributes one.

### Repeated shared-word read

Suppose the earlier shared-word snapshot is 100, and an intervening aliased write changes the shared word to 200 before its second read. If no further change occurs:

- the later computed word destination receives 200;
- the second indirect-call argument is 151.

The two uses are not interchangeable.

### Final argument and byte write

If the argument byte is 10, the third invocation argument is −51.

If the writeback byte is 7, the final 8-bit write stores 7 at address 19. It does not store at an address relative to the adjusted address.

## Uncertainties

The following items are **documented_only**:

- Actual calling convention and origins of the implicit inputs.
- Correctness and reachability of the supplied control-flow reconstruction.
- Identity and complete behavior of F-UNKNOWN-01.
- Address-space mapping, memory permissions, byte order, and alignment rules.
- Whether low addresses denote ordinary memory, device registers, or inaccessible locations.
- Exact machine behavior for signed arithmetic overflow.
- External mutation, concurrency, and volatile-access semantics.
- Whether the repeated reads return equal values.
- Whether any accessed regions overlap.
- Application-level meaning of the constants, lookups, and writes.

An intent to update linked state, process a particular object format, or dispatch through a conventional table would be inference rather than established behavior. No such interpretation is required here.

## Evidence

The static basis is the single supplied function representation and its dependency mapping to F-07bbc14fddb9. It establishes the displayed access widths, arithmetic operations, write sequence, repeated shared-word read, and three-argument indirect invocation.

The supplied annotation reports that the indirect control transfer was not fully recovered. It does not establish a target set or prove that the transfer is an ordinary returning call.

No implementation was executed, no runtime tests were performed, and no external implementation or documentation was consulted. The acceptance examples are derived conditions, not observed outcomes.
