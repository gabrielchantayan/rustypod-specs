<!-- Trusted provenance: F-7b7057ec6dca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7b7057ec6dca

## Purpose

F-7b7057ec6dca performs two direct integer-field writes to caller-supplied storage, calls an external dependency using a separately read global value, and returns the original input pointer if the dependency returns normally.

The two field values originate from **different reads of the same primary global**. The first read supplies the arithmetic input for the later field. The second read supplies the value written to the first field. Equal values require the two reads to observe equal global contents; equality is not established solely by the supplied representation.

The operation resembles partial initialization. Construction of a particular object type, registration, allocation, and ownership transfer are not established.

This contract describes static evidence and conditional consequences. No runtime tests have been performed.

## Interface

### Input and destination layout

The function accepts one pointer to storage that permits the two integer stores described below.

| Destination position | Integer-element index | Byte offset under a 32-bit integer layout | Value directly written |
|---|---:|---:|---|
| First field | 0 | 0 | Value obtained by the second primary-global read |
| Later field | 10 | 40 | Value obtained by the first primary-global read, plus 84 |

The supplied representation uses integer-element addressing. Element indices zero and ten are established by that representation. Four-byte stores, a 40-byte separation between field starts, and a 44-byte destination span are conditional on a 32-bit integer layout.

The representation alone does not independently establish the original target’s integer width or ABI. More generally, for an integer width of \(w\) bytes, the later field begins at byte offset \(10w\), and the span through its final byte is \(11w\) bytes.

A contiguous writable allocation covering that span is sufficient for the direct destination accesses, provided its alignment and object-access rules are suitable. The function does not directly access the nine intervening integer elements. The representation does not establish whether separately accessible regions could validly satisfy the destination accesses under the target’s object model.

There is no explicit destination validation. A null pointer, inaccessible storage, read-only storage, unsuitable alignment, or an invalid integer access is outside the established successful-use preconditions.

### Referenced state

The following descriptive labels distinguish values and locations:

- **Primary global:** the integer-valued global location read twice before the first destination store.
- **Initial sample:** the result of the first primary-global read, retained for the addition.
- **Later sample:** the result of the second primary-global read, used for the store to element zero.
- **Call-input global:** the separate global location read after both destination stores to obtain the external dependency’s argument.

“Initial” and “later” describe read order, not a guarantee that the observed values differ. The call-input global’s type and width are not established by the supplied material.

### Output

On normal completion, the function returns the same pointer value it received. It does not return an interior pointer or a newly allocated pointer.

Any value returned by the external dependency is unused in the shown computation.

Pointer identity does not guarantee that the destination remains allocated, accessible, or unchanged after the external call. Those properties depend on behavior not supplied for the dependency.

## Behavior

### Established operation order

The supplied representation establishes the following logical sequence:

1. Read the primary global and retain the initial sample.
2. Read the primary global again, obtaining the later sample.
3. Store the later sample into destination integer element zero.
4. Add 84 to the retained initial sample and store the result into destination integer element ten.
5. Read the call-input global.
6. Invoke the external dependency with that read value as its sole explicit argument.
7. If the dependency returns normally, return the original input pointer.

There is no destination store between the two primary-global reads. Both primary-global reads precede both destination stores. The call-input global is read only after both stores in the shown sequence.

This is source-facing logical ordering. It is not proof of a particular machine instruction sequence, two distinct hardware loads, volatile access semantics, memory barriers, or cross-thread visibility. Compiler and target details needed to establish those properties are not supplied.

### Distinct samples and field relationships

The first field receives the later sample. The later field receives the initial sample plus 84. Consequently, the difference between the two stored values is not unconditionally 84.

If the primary global is stable across the two reads, both samples are equal, and—subject to representable arithmetic—the later field is 84 greater than the first field.

If the primary global changes between the reads and the second read observes that change, the first field reflects the changed value while the later field remains based on the initial sample.

A change after the second read does not alter either sampled operand in the shown value flow. No additional primary-global read is shown before either store.

These statements distinguish the logical read results. They do not establish that unsynchronized concurrent modification is valid or that any particular external modification mechanism exists.

### Arithmetic and integer boundaries

The increment is exactly hexadecimal 54, equivalent to decimal 84. It is applied only to the initial sample.

The supplied representation shows integer addition, not a pointer dereference, range check, saturation operation, or address-validity check. An address-related interpretation of the values is inferred intent at most.

The original arithmetic width, signedness, and overflow semantics are not independently established. In particular, the displayed operation does not justify promising modular wraparound.

Under a signed 32-bit interpretation:

- The greatest initial sample permitting a representable result is 2,147,483,563.
- That sample produces 2,147,483,647.
- An initial sample of 2,147,483,564 exceeds the representable addition boundary.
- Adding positive 84 to an already representable signed value introduces no lower-bound overflow.

The relevant overflow boundary concerns the **initial sample**, not the later sample. The later sample is stored without the shown addition.

Successful arithmetic results require either representability under the actual arithmetic model or independently established target semantics for the boundary case. No trapping, wrapping, saturation, or recovery behavior is specified here for overflow.

### Aliasing

The representation provides no non-aliasing guarantee between the destination and either global.

Both primary-global reads finish before the first destination store. Therefore, if a destination store aliases the primary global under a valid target layout, that write cannot retroactively change either sample used by this invocation. It may change the global’s subsequent contents, including contents later observed by the dependency.

The call-input global has different timing. Because it is read after both destination stores, an otherwise valid alias with a written destination field may cause the dependency’s argument to reflect that store rather than the global’s entry value.

These are conditional consequences, not assertions that a particular overlapping layout is legal. Type compatibility, alignment, object boundaries, and permissible aliasing remain target-dependent.

### Completion and failure

The shown path contains no conditional rejection, explicit error result, status flag, or recovery operation.

Normal pointer return occurs only after the external dependency returns normally. If that dependency does not return, terminates execution, or otherwise prevents continuation, this function has no established normal return.

No rollback is shown. If the first destination store completes and a later operation fails, the first modification is not undone by any shown action. Similarly, the external call is reached only after the preceding reads, stores, and arithmetic have proceeded sufficiently to reach it.

Exact behavior for invalid accesses, arithmetic overflow, and abnormal dependency completion remains unresolved.

## Side effects

### Direct access footprint

The function directly performs:

- Two logically distinct reads of the primary global.
- One store to destination integer element zero, using the second read.
- One store to destination integer element ten, using the first read plus 84.
- One read of the call-input global after both stores.
- One invocation of the external dependency on the normal path.

There are no shown direct stores to intervening destination elements. No explicit allocation, release, I/O, synchronization, or assignment to a named global is shown.

A destination store may nevertheless modify global storage through valid aliasing. The absence of an explicit global assignment does not exclude that effect.

The complete invocation’s footprint cannot be restricted to these accesses: the dependency’s effects are unknown.

### Ownership and lifetime

The destination is supplied by the caller. This function does not directly allocate or release it.

No ownership transfer is established. Neither acquisition of ownership nor preservation of ownership by the external dependency may be assumed.

The destination pointer is not an explicit argument to the dependency. This does not prove the dependency cannot reach the destination through globals, aliases, or other references.

The destination must permit both direct stores when they occur. Its lifetime and contents after the dependency executes are unresolved. Returning its original pointer value is not a validity guarantee.

### Synchronization

No lock, barrier, atomic operation, or other synchronization mechanism is shown.

The two destination writes are not established as an atomic pair. The representation supplies no guarantee that another execution agent can safely inspect intermediate state or modify either global concurrently.

## Dependencies

### F-unknown

**F-unknown** is a document-local opaque identifier for the sole external callable. Its canonical function ID is not supplied.

The established interaction is limited to:

- One call on the straight-line normal path.
- One explicit argument, obtained from the call-input global after both destination stores.
- No explicit passing of the destination pointer.
- No use of the dependency’s returned value.
- Dependence on normal completion of the call before returning the original pointer.

The dependency’s parameter type, argument width, calling convention, return type, side effects, failure modes, ownership behavior, and termination behavior are **documented_only** unknowns.

No preservation guarantee is established for the primary global, call-input global, destination contents, or destination lifetime.

## Acceptance examples

These are conditional deductions from the supplied representation, not executed tests. Examples promising final destination contents assume valid storage, valid arithmetic, and a dependency that returns normally without modifying or releasing that storage.

### Stable primary global

Given a 32-bit integer layout, primary-global reads that both yield 100, a call-input read that yields 7, and no relevant aliasing:

- Element zero is directly written with 100.
- Element ten is directly written with 184.
- The dependency receives 7 after both stores.
- The function returns the original pointer.
- Intervening elements receive no direct writes from this function.

The equality of the two samples is an example precondition, not an unconditional property.

### Primary global changes between reads

Suppose a valid execution permits the first primary-global read to yield 500 and the second to yield 900.

The direct stores are:

- Element zero: 900.
- Element ten: 584.

Neither a second-field value of 984 nor a first-field value of 500 follows from these read results. This case distinguishes the two-read behavior from a single captured value reused for both fields.

### Primary global changes after the second read

Suppose both primary-global reads yield 500, followed by a permitted change of the global to 900 before the stores.

The direct stores remain 500 and 584. Neither store obtains its operand from the global’s later value.

Whether such an intervening change is possible and valid depends on execution mechanisms and synchronization not supplied here.

### Negative values

Under a signed interpretation, if the initial sample is −100 and the later sample is −20:

- Element zero receives −20.
- Element ten receives −16.

This demonstrates both the addition of positive 84 and the independence of the two sampled operands.

### Destination aliases a global

If a valid layout makes element zero overlap the primary global, the first store may change that global. Both samples have already been read, so the later-field arithmetic still uses the initial sample.

Separately, if a valid layout makes a destination store overwrite the call-input global, and nothing changes it before its subsequent read, the dependency receives the value resulting from that overwrite, interpreted according to the actual call-input type.

Neither example establishes that such aliasing is legal on the unknown target.

### Dependency modifies the destination

Suppose the direct stores write 100 and 184, after which the dependency overwrites either field and returns normally.

The function still returns the original pointer. Acceptance must distinguish the pre-call store values from the destination contents at return.

### Arithmetic boundary depends on the first read

Under signed 32-bit arithmetic, an initial sample of 2,147,483,563 produces a later-field value of 2,147,483,647, regardless of the representable value obtained by the second read.

For an initial sample of 2,147,483,564, no wrapped result is promised—even if the second read yields zero. Changing the later sample does not remove overflow from the addition based on the initial sample.

### Invalid destination or non-returning dependency

A null destination has no specified successful outcome. The representation shows the two primary-global reads before the unchecked first destination store; it does not establish a portable fault location or recovery behavior.

If the dependency does not return normally, no normal pointer return is required. The preceding destination stores are not rolled back by any shown operation.

## Uncertainties

The following remain **documented_only**:

- Original ABI, integer widths, signedness, alignment, and byte order.
- Exact overflow and invalid-access behavior.
- Whether source-level reads correspond to separate machine loads.
- Volatility, synchronization, and permitted external modification of globals.
- Global meanings and initial values.
- Concrete destination type and field semantics.
- Legality and occurrence of relevant aliasing.
- The external dependency’s canonical identity and full contract.
- Destination contents, ownership, and lifetime after the dependency executes.
- Machine-level atomicity and visibility.

Partial initialization is inferred intent. Construction, registration, address-table setup, and ownership protocols are not established.

## Evidence

The supplied static representation supports two primary-global reads in order: an initial retained read, followed by a separate read supplying destination element zero. Destination element ten receives the initial sample plus 84.

Both destination stores precede the call-input-global read and the external invocation. Normal continuation returns the original input pointer.

The representation shows no validation branch, explicit allocation, explicit release, rollback, or direct destination writes beyond elements zero and ten.

The indexed positions, distinct sampled inputs, arithmetic constant, and logical operation order are established static evidence. Four-byte widths and associated byte offsets are conditional on a 32-bit integer layout.

No runtime tests, dependency inspection, or execution observations have been performed.
