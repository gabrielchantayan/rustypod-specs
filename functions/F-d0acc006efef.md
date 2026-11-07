<!-- Trusted provenance: F-d0acc006efef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d0acc006efef

## Purpose

F-d0acc006efef conditionally performs either a bulk reset of a linked container-like structure or repeated processing of positions between two cursor-like arguments. It also writes a 32-bit result through its first argument and returns a 64-bit value assembled from two internal 32-bit words.

The established behavior is structural: comparisons, dependency calls, field reads and writes, and return-value construction. A likely interpretation is range erasure with a whole-container optimization, but that interpretation depends on undocumented dependency semantics. It is not sufficient to define this function as “erase a range” without the qualifications below.

The two completion paths have different return construction. The iterative path may make no processing calls at all, yet still returns the initially supplied container address and output address as the two return words.

## Interface

The function receives four arguments:

| Argument | Established use |
|---|---|
| Output address | Address at which the function writes a 32-bit word. |
| Container address | Address used for accesses to container fields at byte offsets 16 and 20, and passed to two dependencies. |
| Mutable cursor address | Address from which a 32-bit word is read during iteration. Passed directly to two dependencies. |
| Boundary operand | A 32-bit value passed directly as an argument to the comparison dependency. Its interpretation as an address, cursor, or other handle is dependency-defined. |

The supplied representation uses 32-bit address-bearing words and 32-bit field accesses. The return is 64 bits, formed by placing one 32-bit word in bits 63–32 and another in bits 31–0. This is word concatenation, not addition or signed extension.

The function’s own accesses establish this partial layout:

| Object | Byte offset | Width | Observed role |
|---|---:|---:|---|
| Container | 16 | 32 bits | Address-bearing link to a header-like object |
| Container | 20 | 32 bits | Nonzero gate for the bulk path; explicitly cleared on that path |
| Header-like object | 4 | 32 bits | Argument to the bulk dependency; subsequently cleared |
| Header-like object | 8 | 32 bits | Initial comparison value; subsequently set to the current header address |
| Header-like object | 12 | 32 bits | Subsequently set to the current header address |
| Mutable cursor storage | 0 | 32 bits | Position-like value captured before cursor advancement |
| Output storage | 0 | 32 bits | Result word written by this function |

No complete object size, allocation layout, element layout, or alignment contract is established.

The caller must provide memory valid for every access actually reached. In particular, the container’s header link is dereferenced before any comparison can select an alternate path. There is no null-pointer check, bounds check, object-validation check, or explicit error return.

The behavior of overlapping arguments is governed by the ordered accesses below. The function does not establish a non-aliasing guarantee or take a stable snapshot of the container fields.

## Behavior

### Entry initialization

The function first reads the 32-bit header address from container offset 16 and writes that word to the output address.

It then reads container offset 16 again, follows the resulting header address, and reads the 32-bit field at header offset 8. That field value is placed in temporary storage for the first comparison.

The second read of container offset 16 matters. If the initial output write overlaps relevant container storage, the subsequent header lookup may use a value affected by that write.

Before dependency calls begin, the function also establishes:

- A retained 32-bit word initially equal to the output address.
- A result word initially equal to the container address.

These words later supply the iterative path’s return value unless replaced through the processing sequence.

### Selecting the bulk path

The function calls F-def435bd6947 with:

1. The mutable cursor address.
2. The address of the temporary 32-bit value copied from header offset 8.

A zero result bypasses the remainder of the bulk-path selection and enters the iterative path. Any nonzero result continues selection.

On continuation, the function freshly reads container offset 16 into a separate temporary word. It then calls F-def435bd6947 with:

1. The boundary operand.
2. The address of that temporary header-address word.

Only if this second call returns nonzero does the function read container offset 20. The bulk path is taken only when that field is also nonzero.

Consequently:

- A zero first comparison result prevents both the second comparison and the bulk-path count/gate read.
- A zero second comparison result prevents the count/gate read.
- A zero count/gate field prevents bulk processing even when both comparisons return nonzero.
- The count/gate test is a zero-versus-nonzero test, not a positivity test. A word interpreted as a negative signed integer still passes if nonzero.

The comparison dependency receives addresses of temporary words during selection. Its interface does not establish that these words, the cursor, other reachable storage, or global state remain unchanged.

### Bulk processing and reset

When the bulk conditions hold, the function freshly follows container offset 16 and reads header offset 4. It calls F-a358dad243c3 with the container address and that 32-bit field value.

After that call returns, the function performs these writes in order:

1. Store the current header address into offset 8 of the current header-like object.
2. Store zero into offset 4 of the current header-like object.
3. Store the current header address into offset 12 of the current header-like object.
4. Store zero into container offset 20.
5. Read container offset 16 and write that word to the output address.

The header address is read again for the reset operations rather than retained from before F-a358dad243c3. The contract therefore does not assume that the dependency leaves the header link unchanged.

Within each self-link assignment, the destination header base and the header-address value being stored are obtained from container offset 16. The source representation does not establish the relative evaluation order of those reads within the assignment. It does establish the order of the completed field writes listed above.

The function then returns a 64-bit value whose:

- High 32 bits are the header-address word read immediately before the final output write.
- Low 32 bits are the retained word initialized to the output address.

There is no iterative-path comparison, cursor advancement, or per-position processing after this reset.

The explicit reset affects only the listed fields. It does not establish that every field in either object has been initialized or cleared.

### Iterative processing

When bulk processing is not selected, the function repeatedly calls F-def435bd6947 with the mutable cursor address and the boundary operand.

Here the termination condition is stricter than the selection conditions: the loop stops only when the dependency returns exactly 1.

A result of zero continues iteration. Any other result different from 1 also continues iteration, including other nonzero values. The specification must not normalize all nonzero results to “equal” for this loop.

For each continuing iteration, the order is:

1. Read the 32-bit word at the mutable cursor address and retain it.
2. Call F-ab2ec0683d79 with the mutable cursor address.
3. Replace the retained return word with the position-like word captured before that call.
4. Call F-7f54a9348a45 with:
   - The address of the result word initially containing the container address.
   - The container address.
   - The address of the retained position-like word.
5. Write the current result word to the output address.
6. Repeat the comparison.

Thus, the cursor dependency is called before the per-position dependency, while the value supplied by address to the latter originates from the cursor before advancement.

F-7f54a9348a45 can affect both words later used to construct the return because it receives their addresses. The function does not require those words to retain their incoming values.

When the comparison finally returns exactly 1, the function returns:

- High 32 bits: the current result word, initially the container address and subsequently exposed by address to F-7f54a9348a45.
- Low 32 bits: the retained word, initially the output address and subsequently replaced once per iteration before being exposed by address to F-7f54a9348a45.

There is no final output write after the terminating comparison.

If the first iterative comparison returns exactly 1, no iteration body runs. Subject to dependency effects, the output remains the word written at entry, while the return combines the container address in its high half and the output address in its low half.

The function imposes no iteration limit and performs no independent progress check.

## Side effects

Direct writes by this function are:

- One 32-bit output write at entry.
- On the bulk path, three header-field writes, one container-field write, and a final output write.
- On the iterative path, one output write after each completed per-position dependency call.

The function directly performs no visible allocation or deallocation. However, the dependencies may do so. In particular, neither the bulk dependency nor the per-position dependency has an established ownership contract here.

The following properties are not guaranteed:

- Preservation of global state across dependency calls.
- Preservation of the container’s header link.
- Preservation of cursor storage by comparison calls.
- Continued validity of earlier header or position addresses after dependency calls.
- Atomicity, synchronization, or safety under concurrent mutation.
- Rollback if a dependency fails to return or an access is invalid.

The function contains no explicit recovery path. If execution stops during a dependency or memory access, earlier writes and dependency effects may already have occurred.

Although the bulk reset resembles an empty-container representation, the evidence does not establish who owns the removed nodes, whether their resources are released, or whether the reset alone satisfies every container invariant.

## Dependencies

| Opaque ID | Observed interface and role | Unestablished properties |
|---|---|---|
| F-def435bd6947 | Called with two operands. Selection calls pass an address of a temporary word as the second operand; iterative calls pass the boundary operand directly. Its return is tested for nonzero during selection and for exact equality to 1 during iteration. | Whether it compares equality, mutates operands, normalizes Boolean results, preserves globals, or terminates. |
| F-a358dad243c3 | Called once on the bulk path with the container address and the current header-offset-4 word, before the explicit reset writes. | Whether it traverses, destroys, detaches, or frees anything; whether it changes the header link or other state. |
| F-ab2ec0683d79 | Called once per continuing iteration with the mutable cursor address, after capturing the cursor’s current word. | Advancement semantics, ownership effects, invalid-position behavior, and whether it actually makes progress. |
| F-7f54a9348a45 | Called once per continuing iteration with addresses of two retained words and the container address between them. | Whether it erases a node, which result words it writes, resource handling, and other container mutations. |

Dependency names and the structural pattern support the range-processing interpretation only as an inference. The conditional behavior above does not require that interpretation to be correct.

## Acceptance examples

These examples are conditional contracts, not reports of executed tests. Unless stated otherwise, they assume valid, non-overlapping storage, ordinary memory, returning dependencies, and no incidental dependency mutations beyond those described.

### Bulk selection succeeds

Given:

- Both selection calls return nonzero.
- Container offset 20 is nonzero.
- F-a358dad243c3 returns without changing the header link.

Required observations:

- F-a358dad243c3 receives the container address and the header-offset-4 value read for that call.
- Header offsets 8 and 12 become the header address.
- Header offset 4 and container offset 20 become zero.
- The output contains the header address.
- The return’s high word is that header address; its low word is the output address.
- No iterative comparison or per-position dependency call occurs.

### Bulk dependency replaces the header

If bulk selection succeeds and F-a358dad243c3 replaces container offset 16 with another valid header address, subsequent reset accesses use fresh header-link reads. They must not be specified as resetting only the header observed before the dependency.

### The gate field is zero

If both selection calls return nonzero but container offset 20 is zero, F-a358dad243c3 is not called. Execution enters the iterative comparison path. The zero field does not itself cause an immediate return.

### Immediate iterative termination

If bulk processing is not selected and the first iterative comparison returns exactly 1:

- Neither F-ab2ec0683d79 nor F-7f54a9348a45 is called.
- No output write follows the entry write.
- The return’s high word is the container address.
- The return’s low word is the output address.

### A noncanonical nonzero result

If an iterative comparison returns 2, the function performs an iteration rather than terminating. In contrast, a selection comparison returning 2 satisfies that selection’s nonzero condition.

### One processing iteration

Suppose the iterative comparison first returns zero and then 1. The cursor initially contains position word P. F-ab2ec0683d79 changes the cursor to Q. F-7f54a9348a45 writes R into its first addressed word and S into its third addressed word.

Required observations:

- P is captured before cursor advancement.
- The third argument’s storage contains P when F-7f54a9348a45 is called.
- The output is written with R after that call.
- The return’s high word is R and low word is S.
- Q is not substituted for P merely because advancement occurred first.

### Failure to reach the exact termination result

If every iterative comparison returns a value other than 1 and all dependencies return, the function continues iterating. No built-in maximum, cursor-stagnation detection, or alternate completion condition is established.

## Uncertainties

The following remain **documented_only**:

- The logical meaning of each cursor or boundary operand.
- Whether the comparison dependency implements equality and whether it always returns canonical Boolean values.
- The actual container type and the meaning of its offset-20 field.
- Whether the header’s self-links represent a valid empty state for all consumers.
- Allocation, destruction, and ownership behavior of dependencies.
- The semantic meaning of the packed 64-bit return, especially on the zero-iteration path.
- The full memory-aliasing and alignment requirements of the surrounding ABI.
- Dependency failure conventions and exception or nonlocal-exit behavior.
- Whether concurrent or externally observable memory changes are supported.

A likely intent is whole-container clearing when the supplied bounds cover the container, otherwise advancing before removing each selected position. That intent does not override the observed exact-1 termination condition, repeated header reads, or unusual return-word initialization.

## Evidence

Established static evidence consists of the supplied function representation and opaque dependency mapping. It supports the listed 32-bit accesses, field offsets, branch conditions, dependency argument ordering, explicit writes, and 64-bit word concatenation.

The representation does not include dependency implementations, caller contracts, a complete type definition, or a confirmed interpretation of the return value. Semantic descriptions such as “header,” “cursor,” “bulk reset,” and “range erasure” are structural labels or explicitly identified inferences, not independently verified type information.

No runtime tests have been performed. The acceptance examples specify conditional expected behavior derived from static evidence; they are not execution results.
