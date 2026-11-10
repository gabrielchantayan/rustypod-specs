<!-- Trusted provenance: F-1ee2d2f3a1ea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1ee2d2f3a1ea

## Purpose

The established behavior is a sequence of memory mutations driven by one explicit input and three additional incoming machine-state values, followed by an indirect control transfer with four arguments.

No application-level purpose is established. In particular, the supplied material does not establish that this function initializes an object, updates a collection, processes text, or implements a dispatch protocol. Byte values resembling characters do not establish textual meaning.

This contract describes the supplied static evidence. It does not establish that the represented operations are reachable, valid for ordinary inputs, or faithful to every detail of the underlying executable. No runtime tests have been performed.

## Interface

### Inputs and incoming state

| Name used in this contract | Established role |
|---|---|
| **Input** | Explicit 32-bit integer argument. Used both as stored data and as the base of a four-byte memory read at displacement 116. |
| **Base A** | Additional incoming 32-bit value, represented as unsigned. Used in address calculations, as a stored word, and as the source of byte values and a shift count. |
| **Base B** | Additional incoming 32-bit value, represented as signed. Used as a memory base and as the source of a byte value. |
| **Base C** | Additional incoming 32-bit value, represented as signed. Used as a memory base and as a stored word. |
| **Indirect destination** | Unresolved control-transfer destination. Its provenance and valid targets are not established. |

The three bases are not shown as explicit formal arguments. Their calling-convention status is **documented_only**: the evidence requires incoming values but does not establish how a caller supplies them.

For compactness, this contract uses:

- **A-byte**: the low eight bits of Base A.
- **B-byte**: the low eight bits of Base B.
- **double-A**: twice Base A, reduced to 32 bits under the unsigned arithmetic shown.
- **Global 33**, **Global 58**, and **Global 103**: byte-sized locations at those decimal addresses.
- **Global word 169**: a four-byte location beginning at decimal address 169.

These names describe locations and values, not application-level objects.

### Widths and layout

The evidence distinguishes:

- One-byte reads and writes.
- One two-byte write.
- Four-byte integer or untyped-word reads and writes.
- Address calculations involving byte displacements, not array-element indices.

A four-byte word is not necessarily a pointer, even when later operations use its bits as an address.

Byte extraction retains the low eight bits. Signedness of plain byte values is not established. Consequently, the contract specifies resulting byte patterns where possible rather than assigning signed numeric interpretations.

Endianness, alignment requirements, address-space mapping, and the treatment of invalid addresses are not supplied.

### Output and preconditions

There is no explicit return value. The visible outputs are memory effects and the indirect invocation.

Successful completion of the represented path requires every accessed address to support the indicated access width, and requires a valid indirect destination. Some addresses depend on memory contents read partway through execution. No null checks, bounds checks, alignment checks, or destination validation are shown.

Address formation involving values represented as signed integers is not sufficient evidence for a portable overflow rule. Such cases remain **documented_only** unless the target machine’s arithmetic behavior is independently established.

## Behavior

### 1. Repeated write prefix

The function performs the following ordered four-write group **six times**:

| Position within each group | Destination | Width | Value |
|---|---|---:|---|
| First | double-A plus 84 | Four bytes | Input |
| Second | Base B plus 25 | One byte | A-byte |
| Third | Base A plus 17 | One byte | B-byte |
| Fourth | Base B plus 52 | Four bytes | double-A |

There are 24 represented writes in this prefix. They are not conditional on a comparison or on memory contents.

The repetition must not be collapsed in an access-sensitive interpretation, such as one involving device memory. For ordinary, stable, non-overlapping storage, it may leave the same final bytes as one group, but that does not establish equivalence for every permitted memory environment.

A-byte and B-byte are retained from the incoming base values. Writes to memory numerically related to those values do not themselves change the bases.

### 2. Initial captures and byte-level state

After the repeated prefix, the function reads, in order:

1. Global word 169, retaining all four bytes.
2. Four bytes at Input plus 116.

It next writes Global 58 with the low eight bits of twice A-byte plus 48.

Two further four-byte reads follow, in order:

- Base C plus 20.
- Base A plus 4.

These retained values are subsequently used even if their original storage is overwritten.

The following mutations then occur in order:

- Base C plus 13 receives A-byte.
- Base A plus 8 receives byte value 108.
- Base A plus 13 receives B-byte.
- Base A plus 9 receives B-byte.
- Base C plus 84 receives the four-byte Base C value.
- Global 103 receives the low byte of the captured Global word 169 value.

The function then reads four bytes from Base C plus 68. Call the captured result **Shift source**.

Next, Global 33 receives a byte read from Global 103. Base A plus 16 receives the low byte of Shift source, and Base B plus 84 receives the complete four-byte Shift source.

A one-byte write to Base C plus 1 follows. Its value is obtained from Shift source shifted right by a count derived from the low eight bits of Base A, then narrowed to one byte.

This shift is not fully specified for every input:

- Shift source is represented as a signed 32-bit value.
- The count ranges from 0 through 255.
- For counts below 32 and nonnegative Shift source values, the ordinary right-shift interpretation is unambiguous.
- Negative signed operands and counts of 32 or more require target-specific evidence. No masking rule or saturation rule may be assumed.

Base B plus 84 is then overwritten with Base A as a four-byte value. The earlier Shift source write remains an ordered intermediate effect.

### 3. Word replacement and derived address selection

The next four-byte read is from Base A plus 64. The function then:

- Writes the value previously captured from Base A plus 4 to Base B plus 116.
- Writes the just-read Base A-plus-64 word to Base A plus 16.
- Writes Base C to Base B plus 68.
- Writes a byte read from Global 103 to Base A plus 9.
- Writes A-byte to Base C plus 9.
- Writes the originally captured Global word 169 value to Base A plus 68.

The four-byte write at Base A plus 16 covers offsets 16 through 19. It therefore replaces both the earlier byte at offset 16 and the prefix’s byte at offset 17, subject to the target’s byte layout.

Next, the function reads four bytes from Base A plus 84. Call that value **Selected base**.

It then performs another byte write from Global 103 to Base A plus 9. This is a distinct write, not merely a description of the earlier one.

The low byte of the value captured from Input plus 116 is written to Selected base plus 13. After that:

- Base C plus 84 receives the previously captured four-byte value from Base C plus 20.
- Four bytes are read from Base A plus 20.
- Base C plus 17 receives the same low byte taken from the Input-plus-116 capture.
- Base A plus 9 receives the low byte of Selected base.
- Base A plus 68 receives Selected base as a four-byte value.

The read of Base A plus 20 occurs before the final two writes in this group. Its low byte is retained for a later write.

### 4. Final linked-memory effects

The function reads four bytes from Base C plus 4. Call the result **Linked base**.

The next access is a two-byte write of numeric value 101 to the address formed by doubling Linked base. This is not a write to Linked base plus two.

The function then reads four bytes from Linked base plus 100. Call the result **Linked value**.

The remaining memory operations occur in this order:

1. Selected base plus 17 receives the low byte captured from Base A plus 20.
2. Selected base plus 84 receives the four-byte numeric value 100.
3. Four bytes are read from Linked base plus 116; this word is retained for the indirect invocation.
4. Four bytes are read from Base C plus 100. Call the result **Final base**.
5. Base C plus 9 receives the low byte of Linked base.
6. Final base plus 84 receives Linked value as a four-byte value.
7. The address formed by doubling Linked value receives Final base as a four-byte value.

Doubling Linked base and Linked value is represented using signed integer values. The supplied evidence does not establish behavior when the mathematical result exceeds the signed 32-bit range.

### 5. Indirect invocation and return

After the final write, the function invokes an unresolved destination with these four arguments, in order:

| Argument | Value |
|---|---|
| First | Numeric value 114 |
| Second | Linked value, captured from Linked base plus 100 |
| Third | The word captured earlier from Base C plus 20 |
| Fourth | The later word captured from Linked base plus 116 |

No return value from this invocation is used. A return from the containing function is shown afterward, contingent on the indirect invocation returning normally.

No visible branch, loop condition, or explicit error-return path precedes this invocation. Memory faults or invalid control transfers may prevent it from being reached.

## Side effects

### Aliasing and ordering

The locations above are not established to be disjoint. Bases may coincide with each other, fixed globals, derived destinations, or portions of other accessed words.

Accordingly:

- Reads observe memory at their actual position in the sequence.
- A later read cannot generally be replaced with an entry-time snapshot.
- A captured value remains the value read at its capture point, even if the backing memory later changes.
- Earlier writes can affect later address selection.
- A byte write can alter part of a later four-byte read.
- An overwritten write remains relevant when intermediate accesses are observable.

The repeated reads of Global 103 are especially important. The evidence shows separate byte reads for Global 33 and for each later Global-103-derived write to Base A plus 9. Aliasing or externally changing memory can make their values differ.

No atomicity, synchronization, volatile-access guarantee, or concurrency protocol is established. The order described is the represented static order, not proof of a particular compiler or hardware memory-ordering guarantee.

### Ownership and lifetime

No explicit allocation or release operation is visible before the indirect invocation. This does not establish ownership of any accessed region.

The function does not demonstrate that it:

- Owns the incoming bases.
- Retains or relinquishes ownership of the four invocation arguments.
- Creates valid object lifetimes through its stores.
- Preserves global state across the indirect invocation.

The unresolved dependency may allocate, release, mutate, fail to return, or perform other effects. None of those behaviors can be assumed absent.

### Errors

There is no shown validation, status result, rollback, or cleanup path.

If an access fails after earlier writes have occurred, the evidence provides no recovery behavior and no guarantee that previous mutations are undone. Whether failure traps, signals, aborts, or behaves otherwise is environment-dependent.

## Dependencies

**F-unknown-1** is an opaque reference assigned here to the unresolved indirect dependency; it is not a recovered identity.

Its visible boundary consists of the four ordered arguments described above. The supplied material establishes no implementation, stable destination, calling convention details beyond those represented arguments, preservation guarantees, memory ownership rules, or error behavior.

There are no other explicit calls in the supplied body.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

### Six prefix groups remain distinguishable

Given an environment that records each represented write, the prefix must expose six repetitions of the four destinations in the stated order. Reporting only the final four destination values would not satisfy that observation model.

### Byte arithmetic wraps at the destination

If A-byte is 200, and the execution reaches the Global 58 write, that write carries byte pattern 192: twice 200 plus 48 is 448, whose low eight bits are 192.

This example does not require treating A-byte as either a positive or negative signed byte.

### Intermediate and replacement writes both matter

If the read from Base C plus 68 returns 4660, Base B plus 84 first receives the four-byte value 4660 and later receives Base A.

A final-memory observation may see only the latter. An ordered-write observation must retain both.

### Defined shift subset

If the low eight bits of Base A equal 4 and Shift source is 128, the byte written to Base C plus 1 is 8 under the represented right-shift operation.

No corresponding numeric acceptance result is established here for a shift count of 32 or greater.

### Captured invocation argument survives memory replacement

If the read from Base C plus 20 captures value 287454020, later writes to that storage do not by themselves change the third indirect argument. That argument remains the captured word.

This condition assumes execution reaches the invocation and that the incoming machine state itself is not altered by an unspecified mechanism.

### Linked word affects the final destination

If Linked value is 4096 and Final base is 12288, with valid arithmetic and writable storage, the final pre-invocation write stores the four-byte value 12288 at address 8192.

The immediately preceding write stores Linked value at Final base plus 84.

### Aliased global reads cannot be treated as constants

If an intervening store changes Global 103 through aliasing, a subsequent Global-103-derived byte write uses the byte present at that later read. It need not match the low byte initially copied from Global word 169.

## Uncertainties

The following remain **documented_only**:

- Application-level intent and object types.
- How the three implicit incoming bases are supplied.
- Whether all represented accesses are reachable in the actual execution environment.
- Target endianness, alignment policy, and memory map.
- Overflow behavior for signed address calculations.
- Signed-right-shift behavior and out-of-range shift-count handling.
- The identity and provenance of the indirect destination.
- Whether the final transfer is fully characterized by its representation as an invocation.
- External memory changes, access-sensitive memory behavior, and concurrency.
- Ownership, lifetime, and indirect-dependency effects.

No inference of object validity, global preservation, or memory release is part of this contract.

## Evidence

**Established static evidence:** the supplied body represents six identical four-write prefix groups, the subsequent ordered byte/halfword/word accesses, the stated captured values, and a final indirect invocation with four arguments. The widths and displacements above follow those represented accesses.

**Inferred intent:** none adopted. The mixture of low-address globals, doubled address expressions, overlapping stores, and unresolved incoming state does not support a reliable higher-level purpose.

**Verification limit:** no implementation access, executable inspection, or runtime tests were used. The acceptance examples are conditional consequences of the supplied static representation, not observed execution results.
