<!-- Trusted provenance: F-9cf4366abdc8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9cf4366abdc8

## Purpose

F-9cf4366abdc8 conditionally transfers three 32-bit values and one byte from a dependency-produced temporary object into a destination obtained from another dependency.

Destination acquisition always occurs first. The transfer occurs only when an input record refers to a non-null object with at least one nonzero 32-bit field at offsets +0x38 and +0x3C. If the condition succeeds, the function initializes temporary working storage, constructs a second temporary using those two fields, copies selected fields into the destination, and invokes two cleanup dependencies in a fixed order.

This is an established structural description, not an interpretation of the data’s meaning. The supplied evidence does not establish whether the values represent text, geometry, a resource, or another application-specific entity.

## Interface

### Inputs

The function accepts two arguments:

- **Destination selector:** an opaque 32-bit value forwarded unchanged to F-1c06d0a5910b, together with two zero-valued arguments. It is not otherwise interpreted directly.
- **Input-record address:** an address used to read a 32-bit object reference at byte offset +0 and a 32-bit selector value at byte offset +4.

The input record therefore has the following directly accessed layout:

| Byte offset | Width | Observed use |
|---|---:|---|
| +0x00 | 32 bits | Object reference; zero means the conditional work is skipped |
| +0x04 | 32 bits | Read on the active path, then narrowed to a character-width value |

The referred object has two relevant fields:

| Byte offset | Width | Observed use |
|---|---:|---|
| +0x38 | 32 bits | Secondary activation field and an argument to F-a0c5cf3560c3 |
| +0x3C | 32 bits | Primary activation field and an argument to F-a0c5cf3560c3 |

“Primary” describes short-circuit evaluation order, not application significance.

The representation shown uses 32-bit object references and byte-addressed offsets. The evidence does not establish a portable host-language calling convention, address-space policy, alignment tolerance, or byte order.

### Output

The function has no explicit return value.

Its direct externally visible output, when the active path completes, consists of four writes relative to the destination address returned by F-1c06d0a5910b:

| Destination offset | Width | Source |
|---|---:|---|
| +0x04 | 32 bits | Second temporary at +0x04 |
| +0x08 | 32 bits | Second temporary at +0x08 |
| +0x0C | 32 bits | Second temporary at +0x0C |
| +0x10 | 8 bits | Second temporary at +0x10 |

The function does not directly copy the second temporary’s first four bytes. It does not directly write destination offset +0x00.

### Access preconditions

For ordinary completion, each address actually accessed must support its corresponding read or write:

- The input-record address must support a 32-bit read at +0x00, even when the referred object is null.
- A non-null referred object must support the guard reads that short-circuit evaluation reaches.
- The active path requires a readable 32-bit input field at +0x04.
- After the first temporary-initialization dependency returns, the current input-record object reference must support reads at +0x38 and +0x3C.
- The retained destination address must support the four output writes if execution reaches them.
- The dependencies must accept the supplied arguments and temporary-storage addresses.

These are access requirements, not validation performed by this function. There is no explicit check for a null input-record address or an unusable destination.

## Behavior

### Destination acquisition precedes input inspection

The first operation is a call to F-1c06d0a5910b with the destination selector and two zeros.

The returned address is retained for the later output writes. The function does not validate it and does not reacquire a destination after other dependencies run.

Only after this call returns does the function read the input record’s object reference. Consequently:

- Destination acquisition occurs even when the object reference is zero.
- Destination acquisition occurs even when both activation fields are zero.
- Effects of F-1c06d0a5910b precede the guard’s input reads.
- Input mutations performed by that dependency may affect the guard.

Skipping the transfer does not mean that the call has no side effects.

### Activation condition and read order

The function reads the object reference from input offset +0x00.

If that reference is zero, the function returns without reading the selector at input offset +0x04, without reading the referred object, and without invoking the temporary-related dependencies.

For a nonzero reference, the guard reads the 32-bit field at object offset +0x3C first:

- A nonzero value activates the transfer without a guard read of +0x38.
- A zero value causes a guard read of the 32-bit field at +0x38.
- If both observed fields are zero, the function returns without constructing temporaries or directly writing destination fields.

The test is against zero. No positivity, range, flag-mask, or pointer-validity test is established. A nonzero 32-bit representation satisfies the corresponding nonzero test regardless of its possible signed interpretation.

The guard uses the object reference already read for this decision. Its observed field values are not saved as the later construction arguments.

### Temporary preparation

On the active path, the function reads the 32-bit selector at input offset +0x04 and narrows it through a character-width conversion.

Only the low byte participates in that conversion; the upper 24 bits are not passed through unchanged. The evidence does not settle whether byte values from 0x80 through 0xFF are interpreted as signed or unsigned when presented to the dependency.

The function then invokes F-df2422ba76c2 with:

1. The address of a local working area represented as 72 bytes.
2. The narrowed selector.
3. Zero.

The caller does not explicitly initialize that working area before the call. Its contents after the call depend on F-df2422ba76c2.

### Construction uses fresh object-field reads

After F-df2422ba76c2 returns, the function invokes F-a0c5cf3560c3 with:

1. The address of a second local temporary region.
2. The address of the first working area.
3. A 32-bit value read from the current input object at +0x38.
4. A 32-bit value read from the current input object at +0x3C.

The object reference is read again from the input record for these argument expressions. The activation decision therefore does not freeze either the object identity or the field values used for construction.

In particular, changes made by F-df2422ba76c2 can affect the construction arguments. A formerly non-null object reference can become null or invalid; this function does not repeat the guard before accessing the construction fields.

The supplied representation does not establish the relative evaluation order of the two field-valued argument expressions. It establishes their argument positions and that their evaluation follows F-df2422ba76c2. A contract must not impose a +0x38-before-+0x3C argument-read order without further evidence.

The second temporary is exposed as a four-byte initial region followed by fields through byte offset +0x10. At least 17 bytes of relative layout are relevant to this function. A complete object size, alignment requirement, and internal type are not established.

### Destination publication

After F-a0c5cf3560c3 returns, the function copies the selected temporary fields into the retained destination in this order:

1. A 32-bit value to destination +0x04.
2. A 32-bit value to destination +0x08.
3. A 32-bit value to destination +0x0C.
4. An 8-bit value to destination +0x10.

Each value comes from the same relative offset in the second temporary.

These are separate field transfers, not an established atomic publication. The function provides no rollback if an access fails partway through. The evidence does not establish thread synchronization or whether an observer can safely inspect an intermediate state.

No explicit write targets destination +0x11 or later, and no initialization of destination padding is shown.

### Cleanup and completion

After all four writes, the function calls:

1. F-c1466ea05103 with the second temporary’s address.
2. F-ccf4fcd3c34d with the first working area’s address.

It then returns without an explicit value.

This order is fixed on normal fall-through. There is no visible branch based on a cleanup result and no visible error-recovery path.

## Side effects

Direct effects are limited to reads of the input structures, use of local temporary storage, and the four destination writes. Indirect effects may occur in every dependency.

The following ownership limits are important:

- Passing the temporary addresses to dependencies does not establish their complete ownership rules.
- The two final calls are cleanup-shaped, but their precise release, reference-counting, or destruction behavior is unknown.
- The copied values are transferred as three 32-bit representations and one byte. No explicit deep copy, retain, ownership transfer, or lifetime extension occurs between construction and cleanup.
- If a copied value represents a pointer or handle, its validity after cleanup depends on dependency contracts not supplied here.
- The function does not directly release the input object or destination. Dependencies might affect them; preservation cannot be assumed.

The destination address is acquired before temporary construction and retained throughout. If a dependency invalidates it, the function has no visible defensive response.

No explicit error code, diagnostic, retry, exception handler, or rollback is present. If a dependency fails to return normally or an access fails, subsequent publication or cleanup is not guaranteed by the supplied evidence.

## Dependencies

| Opaque ID | Call condition and role | Established arguments |
|---|---|---|
| F-1c06d0a5910b | Always first; returns the retained destination address | Destination selector, zero, zero |
| F-df2422ba76c2 | Active path only; prepares the first temporary | Working-area address, narrowed selector, zero |
| F-a0c5cf3560c3 | After first temporary preparation; produces the second temporary’s relevant fields | Second temporary address, first temporary address, current object field +0x38, current object field +0x3C |
| F-c1466ea05103 | After all destination writes | Second temporary address |
| F-ccf4fcd3c34d | After F-c1466ea05103 returns | First working-area address |

Only F-1c06d0a5910b has a return value directly consumed by this function. No result from the other calls is explicitly tested.

None of these dependencies is established to preserve globals, preserve input structures, avoid destination writes, retain memory, or release memory.

## Acceptance examples

These are conditional contract examples, not executed tests. Unless stated otherwise, accesses are valid, dependencies return normally, and no interfering mutation occurs.

### Null object reference

If the input object reference is zero after destination acquisition:

- F-1c06d0a5910b has already been called once.
- The selector and referred-object fields are not read.
- No temporary-related dependency is called.
- This function performs none of its four direct destination writes.

The destination may nevertheless have changed through F-1c06d0a5910b.

### Both activation fields zero

If the object reference is nonzero and both activation fields are zero, the guard reads +0x3C and then +0x38. The function skips temporary preparation, construction, publication, and cleanup.

### Only the primary activation field is nonzero

If +0x3C is nonzero, the guard does not read +0x38. The active path still later reads both fields for F-a0c5cf3560c3.

An unreadable +0x38 field is therefore not made safe by short-circuit activation: construction still requires it.

### Only the secondary activation field is nonzero

If +0x3C is zero and +0x38 is nonzero, the active path runs. With stable input, the construction arguments include the nonzero +0x38 value followed by the zero +0x3C value.

### Selector narrowing

Selector values 0x0000007F and 0x1234567F supply the same character-width value, 0x7F, to F-df2422ba76c2.

For a low byte of 0xFF, preservation of that byte is established, but its signed numerical interpretation at the dependency boundary is documented_only.

### Input changes during preparation

Suppose the guard activates using one valid object, and F-df2422ba76c2 replaces the input record’s object reference with another valid object. With no further mutation, F-a0c5cf3560c3 receives fields from the replacement object, not saved fields from the original.

If preparation instead replaces the reference with zero, there is no second null check.

### Field publication

Suppose F-a0c5cf3560c3 establishes three particular 32-bit representations and one particular byte at temporary offsets +0x04, +0x08, +0x0C, and +0x10. Those representations are written to the corresponding destination offsets, in that order, before either cleanup call.

This example establishes representation copying only, not the semantic validity or lifetime of copied resources.

## Uncertainties

The following are documented_only:

- Application-level meanings of the inputs, destination, and copied fields.
- Complete layouts and sizes of the input object, destination, and temporaries.
- Character signedness and exact argument-extension conventions.
- Byte order and alignment requirements.
- Relative evaluation order of the construction field arguments.
- Dependency failure behavior and nonlocal control flow.
- Dependency effects on globals, inputs, destination storage, and resource ownership.
- Whether construction initializes every temporary field subsequently read.
- Whether copied representations remain valid after cleanup.
- Concurrent-access rules, synchronization, and atomicity guarantees.

Temporary initialization, construction, and cleanup are inferred roles based on call placement and data flow. They are not established semantic names.

## Evidence

The supplied static body establishes call order, conditional control flow, short-circuit guard reads, field offsets, access widths, selector narrowing, destination-write order, and cleanup-call order.

Dependency implementations and independent type definitions were not supplied. Their semantics cannot be derived solely from their use here.

No runtime tests have been performed. The acceptance examples are conditional consequences of the supplied static evidence, not observed execution results.
