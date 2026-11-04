<!-- Trusted provenance: F-c7edaefed1e5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c7edaefed1e5

## Purpose

F-c7edaefed1e5 conditionally transfers values from two memory locations and ambient machine state into additional memory locations and global state. On every path that reaches its normal return, it returns its fifth explicit argument unchanged.

The branch decision uses the second explicit argument. Importantly, the function reads memory relative to that argument **before** checking whether it is zero. The zero case therefore is not established as safe handling of a missing pointer.

The available static evidence establishes the accesses and branch structure described below. It does not establish an application-level purpose, a valid object model, or the intended meanings of the global destinations.

## Interface

### Explicit inputs and output

The supplied representation exposes five explicit arguments:

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No visible use. |
| Second | 32 bits | Used as an address-like value for an initial halfword read and then compared with zero. |
| Third | 32 bits | No visible use. |
| Fourth | 32 bits | No visible use. |
| Fifth | 64 bits | Returned unchanged if execution reaches the return. |

The return is an opaque 64-bit value. No interpretation as an integer, pointer, aggregate, or ownership-bearing handle is established.

The width descriptions follow the conventional 16-bit halfword and 32-bit integer interpretation of the supplied representation. The actual calling convention and argument-to-register mapping are not established independently.

### Ambient inputs

Two additional values are consumed without visible initialization inside the function:

- **Ambient base:** a represented 32-bit value used for address calculations, arithmetic, and a global-state assignment.
- **Ambient auxiliary word:** a represented 32-bit value assigned to a global destination on the zero branch.

These are effective inputs even though they are not included among the five explicit arguments. Their origin, validity, and preservation requirements are unknown.

The ambient base is not merely an address. Depending on the branch, it also becomes a stored numeric value or supplies bits for a derived address.

### Memory and state inputs

The function accesses:

- A 16-bit value at the second argument plus 4.
- A 16-bit value at the ambient base plus hexadecimal 28.
- A 32-bit destination at the ambient base plus hexadecimal 30 on the nonzero branch.
- A 32-bit destination at the ambient base plus hexadecimal 3B on the zero branch.
- A final 32-bit destination derived from the ambient base on the zero branch.
- Five named global destinations on the zero branch, one of which is also read back.

The supplied source warns that some global symbols overlap smaller symbols at the same address. The backing storage sizes, addresses, and complete alias relationships are not provided.

### Preconditions for normal completion

Normal completion requires every executed access to be valid in the actual execution environment. This includes:

- Read access at both initial halfword locations, including the first location when the second argument is zero.
- Write access at the branch-specific word destinations.
- Valid access to the global storage used by the zero branch.
- Any alignment requirements imposed by the target.
- Defined or otherwise understood target behavior for the address and shift arithmetic.

No length, permission, alignment, or object-boundary validation is visible.

## Behavior

### Unconditional reads

The represented operation order begins with two reads:

1. A 16-bit read at the second argument plus 4.
2. A 16-bit read at the ambient base plus hexadecimal 28.

The second sample is converted to an unsigned 32-bit value. Consequently, its stored numeric value is between 0 and 65,535, with its upper 16 bits clear.

Both reads precede the zero comparison. The first read remains part of the represented behavior even on the nonzero branch, where its sampled value is not subsequently used.

When the second argument is zero, the first effective address is numerically 4 under the represented address arithmetic. Whether that access succeeds is environment-dependent. The function does not visibly detect or recover from its failure.

These are source-visible ordering facts, not an independently established hardware memory-ordering guarantee.

### Nonzero second argument

If the second argument is nonzero:

1. The function writes the zero-extended second halfword sample as a 32-bit value at the ambient base plus hexadecimal 30.
2. It returns the fifth explicit argument unchanged.

No access to the five named global destinations is visible on this branch.

The first halfword sample does not affect the destination, stored value, or return. Its read can nevertheless matter through faults, memory-mapped I/O, or other externally observable read behavior.

### Zero second argument

If the second argument is zero and both initial reads complete, the represented sequence is:

1. Assign the ambient base minus 5 to the first global destination.
2. Read that first global destination and assign its value multiplied by hexadecimal 100 to the second global destination.
3. Write the previously sampled, zero-extended halfword from the ambient-base-relative read as a 32-bit value at the ambient base plus hexadecimal 3B.
4. Select hexadecimal 8D as the value for the final word write.
5. Form the final destination base by shifting the ambient base left by 8 bits.
6. Assign the first halfword sample, converted to an unsigned 32-bit value, to the third global destination.
7. Assign the original ambient base to the fourth global destination.
8. Assign the ambient auxiliary word to the fifth global destination.
9. Write the selected value at the derived base plus hexadecimal 30.
10. Return the fifth explicit argument unchanged.

Steps that only select a value or calculate an address are not themselves memory writes. Their placement clarifies which values the later writes use.

The word write at the ambient base plus hexadecimal 3B uses the halfword sampled **before** the branch. It does not reread that location after the global assignments.

Likewise, the third global assignment uses the first saved halfword sample. Earlier writes do not cause that sample to be refreshed.

The second global assignment differs: it uses a readback of the first global destination after the first assignment. It must not be reduced to a calculation solely from the ambient base unless the first destination’s storage type and readback behavior are independently established.

### Arithmetic and widths

The explicit pointer-cast word stores are represented as 32-bit writes. The explicit halfword reads are represented as 16-bit reads. Endianness is not supplied, so these facts specify access widths and numeric values, not byte order.

The zero-branch final stored value is hexadecimal 0000008D, or decimal 141.

Under a 32-bit wrapping machine model, the zero-branch final destination is obtained by retaining the low 32 bits of the ambient base shifted left by eight positions, then adding hexadecimal 30 with the same address-width treatment. Under that model, high bits shifted out of the ambient base do not contribute to the address.

That wrapping model is conditional, not an established portable-language guarantee. The ambient base is represented as signed, and the supplied material does not settle signed overflow, out-of-range shifts, or integer-to-address conversion.

The global declarations are absent. Therefore, the contract does not assert that every global assignment performs a 32-bit physical store, even when its assigned expression is a 32-bit value. Truncation, extension, and readback behavior depend on those missing declarations and the actual target.

## Side effects

### Visible writes

On the nonzero branch, the visible write set consists of one 32-bit store at the ambient base plus hexadecimal 30.

On the zero branch, the visible write sequence consists of:

- An assignment to the first global destination.
- An assignment to the second global destination.
- A 32-bit store at the ambient base plus hexadecimal 3B.
- Assignments to the third, fourth, and fifth global destinations, in that order.
- A final 32-bit store at the shifted-base-relative destination.

These are logical destinations, not a guarantee of distinct storage. Global overlap is explicitly warned about, and no evidence excludes aliasing between the global storage and address-derived locations.

Consequently, the final state cannot generally be described as six independent zero-branch results. A later write may overwrite bytes affected by an earlier write. The readback used for the second global assignment must also be interpreted in light of the first global destination’s actual storage.

### Alignment and access failures

The store at the ambient base plus hexadecimal 3B is not inherently word-aligned. For a word-aligned ambient base, it is three bytes past a word boundary. The target may support that access, reject it, or impose target-specific behavior.

The function has no visible error return, exception handler, rollback, or retry path. If an access prevents further execution, a normal return is not established. Earlier completed writes may remain observable.

### Ownership

No allocation, release, ownership transfer, or lifetime extension is visible. The function uses supplied and ambient values to access existing storage.

The fifth argument is returned unchanged without a visible dereference. That does not independently establish an ownership policy for values a caller might place in that argument.

## Dependencies

No direct or indirect function call is visible in F-c7edaefed1e5. No additional opaque function dependency can be established from the supplied material.

Execution nevertheless depends on:

- The calling environment supplying the ambient inputs.
- The target’s integer, addressing, alignment, and endianness rules.
- The validity and behavior of the accessed memory.
- The declarations and overlap relationships of the global storage.
- Any fault or device semantics attached to those locations.

These are environmental dependencies, not evidence of unseen callees.

## Acceptance examples

The following are conditional acceptance scenarios, not executed tests.

### Nonzero branch with readable storage

Assume:

- The second argument is nonzero.
- Both initial halfword locations are readable.
- The halfword at the ambient base plus hexadecimal 28 has numeric value hexadecimal BEEF.
- The word destination at the ambient base plus hexadecimal 30 is writable.
- Ordinary memory semantics apply.

The function first reads the second-argument-relative halfword, then the ambient-base-relative halfword. It writes hexadecimal 0000BEEF to the word destination and returns the fifth argument unchanged.

No named global assignment occurs on this path.

### Zero argument with an inaccessible initial read

Assume the second argument is zero and the effective address 4 is unreadable under the target’s addressing rules.

The first represented read cannot complete normally. No successful global update, final word write, or normal return is required by the evidence. A contract that promises safe null handling would contradict the visible access order.

### Zero branch under explicit 32-bit assumptions

Assume:

- The ambient base is hexadecimal 00001000.
- The halfword at effective address 4 is hexadecimal 1234.
- The halfword at hexadecimal 00001028 is hexadecimal ABCD.
- All accesses succeed.
- Relevant arithmetic uses 32-bit wrapping semantics.
- All global destinations provide ordinary, nonoverlapping 32-bit storage.

The observable write sequence is:

1. First global destination receives hexadecimal 00000FFB.
2. Second global destination receives hexadecimal 000FFB00 after reading back the first.
3. Address hexadecimal 0000103B receives the 32-bit value hexadecimal 0000ABCD.
4. Third global destination receives hexadecimal 00001234.
5. Fourth global destination receives hexadecimal 00001000.
6. Fifth global destination receives the ambient auxiliary word.
7. Address hexadecimal 00100030 receives hexadecimal 0000008D.

The fifth explicit argument is returned unchanged. The nonoverlap and 32-bit-global assumptions are necessary for this simplified final-state description.

### Later failure without rollback

Assume the zero branch completes its global assignments and earlier word store, but its final destination is not writable.

The function has already attempted the preceding writes in the stated order. No rollback is visible, and no normal return is guaranteed after the failing final store.

## Uncertainties

The following are **documented_only**:

- The intended application-level purpose.
- The actual ABI and source of the ambient inputs.
- Whether the unusual zero-branch addressing is deliberate.
- Complete global storage widths and alias relationships.
- Target endianness and unaligned-access behavior.
- Exact signed arithmetic and address-conversion semantics.
- Whether any accessed location is volatile, device-backed, or concurrently modified.
- The mechanism and externally visible result of an invalid access.

A plausible interpretation is that the ambient base identifies structured state because several fixed offsets are used. That is inferred intent only. No object type, allocation size, or field meaning follows from it.

## Evidence

Established static evidence consists of the supplied representation of F-c7edaefed1e5:

- Five explicit arguments and an unchanged fifth-argument return.
- Two ambient inputs consumed without visible local initialization.
- Two halfword reads before the branch.
- A branch based solely on whether the second argument is zero.
- One final word store on either normally completing branch.
- Additional global assignments and a word store on the zero branch.
- A readback of the first global destination before the second global assignment.
- A warning that global symbols overlap smaller symbols.
- No visible function calls, allocation, release, explicit validation, or recovery path.

No runtime tests have been performed. All examples above are conditional consequences of the static evidence, with additional assumptions stated explicitly.
