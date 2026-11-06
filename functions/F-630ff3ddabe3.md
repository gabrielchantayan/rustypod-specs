<!-- Trusted provenance: F-630ff3ddabe3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-630ff3ddabe3

## Purpose

F-630ff3ddabe3 adds two sequences of integer words into a destination, carrying overflow toward increasing element indices. It starts with carry zero and returns the carry remaining after the requested final element.

For valid, nonoverlapping input sequences whose values remain stable, this is unsigned multiword addition with the least-significant word at index zero. That numeric interpretation is inferred intent. The established behavior is forward processing of individual words; no external integer format or byte serialization is established.

A zero or negative count returns zero without accessing any sequence. A positive count produces exactly that many destination words. The final carry is returned separately, not appended to the destination.

## Interface

### Arguments and result

| Argument | Meaning |
|---|---|
| Destination | Address of writable unsigned-word elements. |
| First input | Address of signed-integer elements converted to the unsigned arithmetic type when used. |
| Second input | Address of signed-integer elements converted to the unsigned arithmetic type when used. |
| Count | Signed integer specifying the number of elements to process. |

The result has the unsigned word type. Under the arithmetic and valid-memory conditions below, it is zero or one.

The interface provides no initial-carry argument, input lengths, destination capacity, stride, allocation handle, or separate error channel.

### Width and layout

The supplied representation identifies signed integer inputs and an unsigned word type, but does not establish the unsigned type’s declaration or the target ABI. This contract uses \(W\) for the unsigned word width and \(M=2^W\) for its modulus. Arithmetic claims assume that the represented unsigned type is the corresponding unsigned integer type used by the displayed operations.

The signed count’s width is also unspecified. Its sign, rather than an unsigned reinterpretation, determines whether processing begins.

Each sequence is traversed in contiguous elements of its declared pointed-to type. There is no independent byte stride. On a target with 32-bit, four-byte input and destination elements, consecutive positions are four bytes apart. Neither those sizes nor byte order are independently established here.

Input conversion is numeric conversion to the unsigned arithmetic type. It is not signed multiword addition. For example, signed value −1 contributes \(M-1\), without requiring an assumption about the signed value’s underlying bit representation.

### Storage preconditions

For a positive count:

- Both inputs must permit reads of every requested element.
- The destination must permit writes of every requested element.
- Storage lifetime, alignment, object-access rules, and pointer arithmetic must permit the accesses.
- Overlapping storage must also satisfy those requirements; overlap alone does not waive them.

There is no validation of addresses or capacities. Invalid storage has no specified normal error result.

For a nonpositive count, no pointed-to storage is accessed and no pointers are advanced. Null pointer arguments therefore do not cause a dereference on that path. This statement concerns the function body, not the validity of evaluating arbitrary argument expressions before entry.

The function does not acquire or relinquish ownership of storage. Destination contents are replaced at processed positions. Previous destination values matter only when they are also observed through an input.

## Behavior

### Arithmetic contract

Carry begins at zero.

For each position, the unsigned interpretation of the first input is combined with the incoming carry, with unsigned wraparound. The unsigned interpretation of the second input is then included, again with unsigned wraparound. The resulting word is assigned to the corresponding destination position.

For values observed consistently during that position, the relationship is:

\[
a_i+b_i+c_i=d_i+M c_{i+1},
\]

where \(a_i\) and \(b_i\) are the unsigned input values, \(d_i\) is the destination value, and \(c_i\) is the carry entering the position.

Both wraparound contributions are accounted for. Nevertheless, carry remains either zero or one: the largest possible total of two words and an incoming carry is \(2M-1\). There is no reachable carry value of two under these conditions.

For disjoint, stable input sequences of positive length \(n\), define \(A\), \(B\), and \(D\) as their respective unsigned multiword values with index zero least significant. Then:

\[
A+B=D+c_nM^n.
\]

This whole-sequence equation uses the original inputs only when destination writes do not change input values before they are consumed.

### Extent and progression

Elements are processed in increasing index order. Every requested destination assignment finishes before processing advances to the next position. There is no complete-input snapshot and no whole-group staging of output.

The internal organization groups up to four positions. This does not change the requested extent:

- Counts one through three process only the requested prefix.
- Count four processes exactly four positions.
- Larger counts continue with the next group while preserving carry.
- No input at index equal to the count, or beyond it, is needed.
- No destination at index equal to the count, or beyond it, is written.

When execution continues beyond a completed four-position group, each working address advances by four elements of its own pointed-to type.

The count checks do not require signed overflow. Entry requires a positive count, and each later position within a group is reached only when sufficient positions remain. In particular, the four-element reduction occurs only after reaching the fourth position, so its result is nonnegative. The initial negative-count case returns without count arithmetic.

### Observable access constraints

The source distinguishes positions whose indices are multiples of four from the other positions.

At an index divisible by four, the second input is captured before the first input is evaluated. The destination assignment precedes completion of that position’s carry calculation, which uses the captured values rather than rereading the overwritten destination or inputs.

At the other positions, the first input contributes to the intermediate arithmetic before the second-input expression used for carry determination. Carry is determined before the destination assignment. The second-input expression occurs again for that assignment; it is not represented as a single retained value shared by both uses.

These are source-level sequencing and value-use facts, not a promise of a physical load trace. The accesses are not marked volatile. The contract does not require a compiler to emit two machine loads for repeated ordinary reads, nor does it specify load width, instruction scheduling, atomicity, barriers, or device transactions.

For ordinary memory that remains unchanged between those evaluations, the repeated second-input expressions yield the same value. Numerical guarantees do not extend to unsupported asynchronous mutation merely because the source contains repeated expressions.

### Aliasing and overlap

There is no nonoverlap check and no protective copy.

Exact in-place operation is consistent with the access order: the destination may occupy the same element positions as either input, or both, provided the applicable storage and object-access rules permit it. Each position’s required input observations precede its destination write.

Shifted overlap is different. A destination write may replace an input element needed by a later position. That later position uses the replacement, not an implicit snapshot of the original sequence.

Forward traversal is therefore part of the behavior, not an interchangeable implementation choice for overlapping storage. The four-position grouping does not defer writes until all four positions have been read.

## Side effects

The explicit externally visible storage effects are destination assignments in increasing index order. If destination storage overlaps an input, those assignments modify that input storage as well.

The supplied body contains no function calls, explicit global accesses, allocation, deallocation, I/O, or synchronization. It does not report an error through a status object or global error field.

The function provides no rollback mechanism. If execution becomes invalid or faults after some writes, the supplied body establishes neither recovery nor restoration of earlier destination contents. It also does not define a portable partial-result guarantee for such an execution.

No ownership transfer is present. The caller remains responsible for the supplied storage. There are no callees whose ownership effects or preservation of global state need to be assumed.

Concurrent use is not independently supported by this body. Whether concurrent accesses are valid depends on the surrounding memory model and storage discipline; the function itself supplies no synchronization or coherent-snapshot mechanism.

## Dependencies

There are no dependency function IDs to list: F-630ff3ddabe3 invokes no other function in the supplied body.

Its arithmetic and memory behavior depend on:

- The actual definition and width of the represented unsigned word type.
- The signed integer types and their conversion to the unsigned arithmetic type.
- Unsigned modular addition and unsigned comparisons.
- Element-based address progression.
- Valid storage and permitted accesses.
- Consistent input observations for the stated numerical addition contract.

No external allocator, release routine, error handler, or global-state-preserving callee is evidenced.

## Acceptance examples

These are conditional acceptance criteria derived from static inspection, not executed tests. Except for the nonpositive-count cases, assume valid storage, permitted accesses, and ordinary inputs that remain stable apart from explicitly described overlap. Hexadecimal examples assume 32-bit unsigned arithmetic and compatible input element types.

### Nonpositive count

For count zero or any representable negative count, the result is zero and all destination contents remain unchanged. No input or destination element is accessed.

Null pointers are acceptable with respect to dereferencing on this path.

### Single word without overflow

With count one and input values 7 and 9, the destination word becomes 16 and the return value is zero.

A sentinel immediately after the destination word remains untouched. The prior destination word is irrelevant unless it also supplies an input.

### Single word with overflow

With count one, first input word `0xFFFFFFFF`, and second input word 1, the destination becomes zero and the return value is one.

No additional destination word is written to hold that carry.

### Two maximum input words

With count one and both input words `0xFFFFFFFF`, the destination becomes `0xFFFFFFFE` and the return value is one, not two.

### Carry propagation

With count two, first input `[0xFFFFFFFF, 0xFFFFFFFF]`, and second input `[1, 0]`, the destination becomes `[0, 0]` and the return value is one.

The second position consumes the carry produced by the first. Its overflow comes from combining that carry with the first input word.

### Carry across the four-position boundary

With count five, five first-input words equal to `0xFFFFFFFF`, and second input `[1, 0, 0, 0, 0]`, all five destination words become zero and the return value is one.

Carry is not reset when traversal moves into the next group.

### Short extent

With count three, inputs `[1, 2, 3]` and `[4, 5, 6]`, the destination prefix becomes `[5, 7, 9]` and the return value is zero.

A fourth destination sentinel remains unchanged. No fourth input element is required.

### Signed input conversion

With count one and signed input values −1 and 2, the destination becomes 1 and the return value is one.

This follows from converting −1 to the unsigned value `0xFFFFFFFF`; it is not the signed result of adding mathematical −1 and 2.

### Exact in-place operation

If the destination and first input occupy the same permitted storage, initially `[5, 6]`, while the second input is `[7, 8]` and count is two, the shared storage becomes `[12, 14]`. The return value is zero.

The result does not require a preliminary input copy.

### Forward shifted overlap

Let the first input start at shared storage `[1, 2, 3, 4]`, and let the destination start at its second element. With a separate second input containing three zeros and count three, final shared storage is `[1, 1, 1, 1]`, and the return value is zero.

Each write replaces the first-input value needed by the following position. An implementation producing a result from an untouched snapshot would not satisfy this overlap behavior.

## Uncertainties

**documented_only — Concrete widths and ABI.** The unsigned type definition, integer sizes, alignment requirements, pointer representation, and target ABI are not supplied. The 32-bit examples are conditional, not evidence of a confirmed target layout.

**documented_only — Byte order and external representation.** Element order is established; byte order within an element and any serialized numeric format are not.

**documented_only — Physical memory trace.** Source-level evaluations and ordering constraints are visible. Exact machine loads, stores, access granularity, and compiler transformations are not established.

**documented_only — Invalid storage.** No normal error handling is present. A particular fault, exception, recovery action, or partial-output state cannot be promised for invalid execution.

**documented_only — Asynchronously changing inputs.** The body supplies no protocol for concurrent writers, signal-driven changes, or memory-mapped devices. No numerical acceptance result is specified for unsupported mutation.

**Inferred intent — Multiword addition.** Forward propagation of a one-bit carry supports least-significant-word-first unsigned addition. It does not establish a signed whole-number interpretation or a higher-level caller protocol.

## Evidence

This contract is based only on static inspection of the supplied body for F-630ff3ddabe3.

Established observations include the nonpositive-count return, zero initial carry, unsigned intermediate arithmetic and wrap comparisons, forward destination assignments, bounded four-position grouping, carry preservation between groups, and return of the final carry. The differing treatment of the second input within each group is visible in its source-level value uses. No calls or explicit global references appear.

The carry range, multiword addition equation, and overlap outcomes are deductions from those observations under the stated type and memory preconditions. Concrete type widths and runtime access traces remain unresolved.

No runtime tests, compiler inspection, ABI verification, or external dependency inspection have been performed.
