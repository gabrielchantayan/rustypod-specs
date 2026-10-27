<!-- Trusted provenance: F-67e7b65f7cfe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-67e7b65f7cfe

## Purpose

F-67e7b65f7cfe performs two ordered evaluations and conditionally writes a four-byte result into caller-supplied storage.

The first evaluation has priority. A nonzero result from F-a1e68ae27737 selects the value **2** and prevents the second evaluation. If the first result is zero, a second evaluation occurs; a nonzero second result selects **6**. If both results are zero, this function performs no destination write of its own.

The two candidates are not necessarily obtained from the same base address. The supplied body reads an external base-address location twice before the first dependency invocation:

- The **initial base** is retained for a possible second-candidate read.
- The **subsequently read base** supplies the address for the first-candidate read.

The evidence does not establish that these two reads necessarily return the same address.

“Evaluation,” “candidate,” and “selection” describe the visible control structure, not established domain semantics. The dependency is not established to perform equality, string comparison, lookup, or validation. The meanings of the stored values 2 and 6 are unknown.

## Interface

### Inputs

| Input | Visible use | Established representation |
|---|---|---|
| Destination base | Supplies the address for a conditional store at byte offset 12 | An integer-valued input used in address formation. Exact address width and conversion rules are not independently established. |
| Evaluation value | Passed as the first argument to each reached dependency invocation | Four opaque bytes. Signedness, encoding, pointer interpretation, and ownership are not established. |

The destination base is not directly passed as an argument to F-a1e68ae27737. This does not establish isolation: the dependency could reach destination storage through the evaluation value, candidate values, globals, or aliases whose meanings are not supplied.

### External state and candidate layout

An external location contains a base-address value. This function visibly reads that location twice, in order.

| External-location read | Use of the value obtained |
|---|---|
| First read | Retained as the initial base; used only if the second candidate is needed |
| Second read | Used to locate the first candidate before the first dependency invocation |

Candidate accesses are:

| Candidate | Address source | Byte offset | Access width |
|---|---|---:|---:|
| First | Base obtained by the second external-location read | 100 | Four bytes |
| Second | Initial base obtained by the first external-location read | 104 | Four bytes |

Thus, the fields lie four bytes apart only when the two base values are equal. Equality of those base values is not a requirement established by the supplied body.

No containing object type, total allocation size, field meaning, or storage lifetime is supplied.

### Outputs and return

The function has no specified return value.

Its explicit output is a conditional four-byte store at destination base plus 12:

- **2** if the first dependency result is nonzero.
- **6** if the first dependency result is zero and the second is nonzero.
- No explicit destination store if both results are zero.

The store replaces four bytes; it is not established to be a one-byte flag or a particular enumeration. Byte order is not established.

### Preconditions for normal completion

Required access conditions depend on the path taken:

- The external base-address location must support both ordered reads.
- Four bytes at the base obtained by the **second** external read, plus 100, must be readable before the first dependency invocation.
- If the first invocation returns zero, four bytes at the **initial** base plus 104 must be readable when that later access occurs.
- If either invocation returns nonzero, four bytes at destination base plus 12 must be writable when the result store occurs.
- Each reached dependency invocation must satisfy its own undocumented preconditions and return normally for subsequent behavior to occur.

The initial base is captured even when the first invocation will return nonzero, but this function does not dereference that initial base on that path merely because it was captured. The first candidate’s address instead comes from the second external read.

No visible bounds, null-address, alignment, or arithmetic-overflow checks enforce these conditions. No explicit destination access occurs on the two-zero-result path, but that does not establish that an arbitrary destination input is safe for all dependency behavior.

## Behavior

### Observable ordering

The established access and call order is:

1. Read the external base-address location and retain the resulting initial base.
2. Read that external location again to obtain the base used for the first candidate.
3. Read four bytes at the second-read base plus 100.
4. Invoke F-a1e68ae27737 with the evaluation value first and that first candidate second.
5. Examine the returned integer for zero versus nonzero.
6. On a nonzero first result, write the four-byte value 2 at destination base plus 12. Do not read the second candidate or invoke the dependency again.
7. On a zero first result, read four bytes at the retained initial base plus 104, then invoke F-a1e68ae27737 with the same evaluation value first and this second candidate second.
8. On a nonzero second result, write the four-byte value 6 at destination base plus 12. On a zero second result, return without an explicit destination store.

This ordering distinguishes the two external-location reads from the two candidate reads. Both external-location reads occur before the first dependency invocation. The second candidate’s contents are obtained only after that invocation returns zero.

There is no third external-location read to choose the second candidate’s base.

### Result interpretation and priority

The dependency’s result is used solely as a zero/nonzero condition. Different nonzero results do not select different output values.

A negative result, if representable by the dependency’s integer return type, is nonzero and therefore selects the same branch as any other nonzero result. The visible body does not interpret negative values as a separate error category.

A nonzero first result has strict priority:

- Exactly one dependency invocation occurs.
- The second candidate is not read by this function.
- The explicit destination value is 2.
- No second evaluation is performed to confirm, override, or combine with the first result.

A first result of zero permits the second evaluation; it does not itself cause a destination store.

### Two base reads and delayed second-candidate access

The initial base is retained independently of the value returned by the second external-location read. If the reads produce different addresses, the first candidate comes from the later-selected storage at offset 100, while the conditional second candidate comes from the initially selected storage at offset 104.

The source does not establish a mechanism that forces the two base values to agree. Their equality must therefore remain an additional condition, not an unconditional premise of this contract.

Once both reads have occurred, a dependency-induced replacement of the external base-address value does not redirect the second candidate access. That access uses the retained initial base.

Retaining the base does not retain the second candidate’s contents. If the first dependency invocation modifies the four bytes at initial base plus 104 and then returns zero, the second invocation receives the contents read after that modification, assuming the access remains valid.

Likewise, retaining an address does not preserve the lifetime of its storage. If the first invocation invalidates the initially selected storage, the later read is not protected by any visible validity check.

### Destination behavior

The function does not explicitly read the destination field before a successful result store. It replaces the four bytes at offset 12 rather than merging a result with their previous contents.

The store follows the dependency invocation that selected the branch. If that invocation has already modified those same four bytes, this function’s later store replaces them with 2 or 6, provided the destination remains writable.

When both results are zero, “no explicit destination store” must not be strengthened into “destination unchanged.” Either dependency invocation may have modified destination storage through an alias or other access not visible here. This function performs no restoration on the two-zero-result path.

### Arithmetic and boundaries

The visible address displacements are 12, 100, and 104 bytes. Candidate values and the stored result each occupy four bytes.

The supplied evidence does not independently establish the exact width or signedness rules of the integer-valued address calculations. No checked arithmetic is visible. Behavior for overflowing address additions, invalid conversions, misalignment, or accesses crossing unavailable storage is **documented_only unknown**, not a defined fallback or error result.

## Side effects

The function’s direct visible effects are:

- Two ordered reads of the external base-address location.
- One four-byte read for the first candidate, using the second-read base.
- Conditionally, one four-byte read for the second candidate, using the retained initial base.
- One or two invocations of F-a1e68ae27737.
- At most one explicit four-byte destination store.

Both external-location reads occur even on the first-nonzero path, assuming execution reaches the first invocation normally.

There is no explicit write back to the external base-address location or either candidate field. Nevertheless, the destination store may overlap those locations if their addresses alias. No non-aliasing requirement is established.

No explicit allocation, release, ownership transfer, logging, or synchronization appears in this function’s body. These observations do not constrain the dependency’s effects. In particular, neither ownership preservation nor memory-lifetime preservation may be assumed across a dependency invocation.

The evidence establishes no atomicity or concurrent-access guarantees. Four-byte accesses must not be assumed atomic solely because of their width. The two external reads are distinct in the supplied static body; no concurrency protocol establishing consistent values is supplied.

## Dependencies

### F-a1e68ae27737

Each invocation receives two arguments:

1. The unchanged four-byte evaluation value supplied to F-67e7b65f7cfe.
2. A four-byte candidate read for that invocation.

The first invocation receives the candidate at offset 100 from the base obtained by the **second external-location read**. The conditional second invocation receives the candidate at offset 104 from the **retained initial base**.

The dependency returns an integer that this function tests only for zero or nonzero. Its exact return width and signedness rules are not independently established by the supplied material.

The dependency’s substantive behavior is **documented_only unknown**. This contract does not assume that it:

- Compares values or reports a match.
- Interprets either argument as a pointer rather than an opaque scalar.
- Preserves the external base-address location.
- Preserves candidate storage, destination storage, or other globals.
- Preserves the lifetime of storage addressed by either captured base.
- Allocates, retains, transfers, owns, or releases memory.
- Returns normally for every input.
- Uses nonzero exclusively for success rather than another condition.

For this caller, a nonzero result means only that the corresponding result-writing branch is taken.

## Acceptance examples

These are conditional deductions from static evidence, not reported executions. Unless stated otherwise, assume valid accesses, normal dependency returns, and no unmentioned dependency effects.

| Conditions | Required behavior |
|---|---|
| First dependency result is nonzero | Read the external location twice; read the first candidate using the second-read base; invoke the dependency once; skip the second candidate; store four-byte value 2. |
| First result is zero; second is nonzero | Invoke the dependency in candidate order; obtain the second candidate from initial base plus 104 after the first invocation; store four-byte value 6. |
| Both results are zero | Perform both evaluations and no explicit destination store. |
| First result is a representable negative integer | Treat it as nonzero; store 2 and skip the second evaluation. |
| First result is zero; second is a representable negative integer | Treat the second result as nonzero; store 6. |

### Different values from the two external reads

Suppose the first external read yields address A and the second yields address B, with A and B different. The first invocation receives the candidate from B plus 100. If it returns zero, the second invocation receives the candidate from A plus 104.

The contract does not substitute B plus 104 for that second access or A plus 100 for the first access. This example is conditional on those read results and valid storage; it does not assert a particular mechanism for producing different values.

### Equal values from the two external reads

If both reads yield address A, the candidate accesses use A plus 100 and, conditionally, A plus 104. Even in this case, the second candidate is not read before the first invocation.

### External base replacement during the first invocation

Suppose the first invocation changes the external base-address location and returns zero. The second candidate still comes from initial base plus 104, not from the replacement base. The originally selected storage must remain readable.

### Second-candidate mutation

Suppose the first invocation changes the four bytes at initial base plus 104 and returns zero. The second invocation receives the later contents read from that location.

### Destination mutation

If an invocation modifies the destination field and returns nonzero, the subsequent explicit store replaces that field with 2 or 6 according to invocation order.

If dependency effects modify the destination but both invocations return zero, this function does not undo those effects.

### Unreadable storage on a skipped path

If the first invocation returns nonzero, this function does not read initial base plus 104. Readability of that candidate location is therefore not required by this function on that path. Both external-location reads and the first-candidate read are still required.

## Uncertainties

The following remain **documented_only unknowns**:

- Domain meanings of the evaluation value, candidates, destination field, and output values.
- Conditions under which the two external base-address reads return equal or different values.
- Exact address width, byte order, alignment requirements, and integer-to-address conversion rules.
- Arithmetic behavior at address-space boundaries.
- Full dependency semantics, errors, memory effects, and ownership rules.
- Behavior following an invalid access, abnormal dependency exit, or storage invalidation.
- Thread safety and outcomes of unsynchronized concurrent mutation.
- Any meaningful return-register contents beyond the void-returning interface.

An ordered two-candidate classifier is an inferred interpretation of the control structure. Domain-specific classifications or guarantees of stable shared state are not established.

## Evidence

The supplied static body establishes two ordered reads of the external base-address location. The first value is retained for the conditional candidate access at offset 104. The second value supplies the address for the first candidate access at offset 100.

It also establishes four-byte candidate values, dependency argument order, zero/nonzero branching, first-evaluation priority, conditional four-byte stores of 2 or 6 at destination offset 12, and no explicit destination store when both results are zero.

Dependency implementation, callers, complete layout definitions, and runtime traces were not supplied. No runtime tests have been performed. All acceptance examples express conditional consequences of the static evidence rather than experimentally verified behavior.
