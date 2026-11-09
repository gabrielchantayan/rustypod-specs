<!-- Trusted provenance: F-ff4aeaa3fa12; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff4aeaa3fa12

## Purpose

F-ff4aeaa3fa12 produces a 32-bit comparison result for two indirectly referenced descriptors.

The established comparison hierarchy is:

1. Descriptor kind.
2. For matching supported kinds, collection count.
3. A reverse-indexed pass over collection elements, comparing payload-related properties and an element-level auxiliary value.
4. If that pass finds no difference, a second reverse-indexed pass comparing each element’s primary value through F-9878699bd825.

Matching descriptor kinds other than 1 and 2 return zero without examining their payloads.

The apparent intent is an ordering operation. However, the available evidence does **not** establish a total order, antisymmetry, stability under mutation, or normalized results limited to −1, 0, and 1. In particular, one payload branch obtains its result from a dependency called with only the left payload.

## Interface

### Inputs and result

There are two input arguments, each identifying a memory location containing a descriptor address. They are not direct descriptor arguments.

The represented interface uses:

- 32-bit address-bearing slots.
- 32-bit scalar fields and comparison results.
- Byte-sized elements in the specialized byte-sequence branch.
- Signed comparisons for collection-loop termination and byte-loop bounds.

The return value is a 32-bit integer. Depending on the branch, it is either:

- A difference between scalar values.
- A result returned by a dependency.
- Zero.

No explicit error result or separate error channel is present.

### Referenced layouts

These layouts describe only fields accessed by this function. They do not establish complete object sizes or meanings for intervening storage.

| Structure | Offset | Accessed content |
|---|---:|---|
| Input reference | +0 | Descriptor address |
| Descriptor | +0 | Kind, 32 bits |
| Descriptor | +4 | Address of an indirection holder |
| Indirection holder | +0 | Address of a backing object |
| Backing object for kind 1 | +20 | Address of a collection-handle cell |
| Backing object for kind 2 | +8 | Address of a collection-handle cell |
| Collection-handle cell | +0 | Handle supplied to collection dependencies |
| Collection element | +0 | Primary value |
| Collection element | +4 | Payload address |
| Collection element | +8 | Auxiliary comparison value |
| Payload | +0 | Leading scalar; also used as byte-sequence length in one branch |
| Payload | +4 | Payload tag |
| Payload | +8 | Data value; interpreted as a byte-storage address in one branch |

The terms “kind,” “tag,” “primary value,” and “auxiliary comparison value” are descriptive labels, not established domain names.

### Preconditions

For a defined, fault-free execution, every memory location actually reached by the selected branch must be readable for the width used. Dependencies must accept the handles and values passed to them and return suitably usable results.

There are no explicit checks for:

- Null or invalid references.
- Invalid descriptor or payload addresses.
- Unsupported payload tags.
- Negative or inconsistent counts.
- Out-of-range element indices.
- Byte-storage capacity.
- Integer overflow.
- Mutation during dependency calls.

The caller therefore needs stronger environmental guarantees if it requires a deterministic comparison over stable objects. Those guarantees are not enforced here.

## Behavior

### 1. Descriptor-kind comparison

The function first reads the left descriptor address from the left input reference, then the right descriptor address from the right input reference.

It reads the left kind and compares it with the right kind by subtraction. If that difference is nonzero, it returns the difference immediately. No backing object or collection is accessed on this path.

If the kinds match:

- Kind 1 selects the backing-object field at byte offset 20.
- Kind 2 selects the backing-object field at byte offset 8.
- Every other kind returns zero immediately.

For supported kinds, the right descriptor’s indirection holder, backing-object address, and selected collection-handle-cell address are resolved before the corresponding left-side chain.

The selected cell addresses are retained for subsequent use. Their handle contents are read again when collection dependencies are called; the function does not take one immutable handle snapshot for the whole operation.

### 2. Collection-count comparison

F-10b760faca36 is called first with the current left handle and then with the current right handle.

If those two results differ, the function calls F-10b760faca36 again for the left handle and then again for the right handle. It returns the difference between these **new** results.

Consequences:

- The values that trigger the unequal-count branch are not necessarily the values used for its returned difference.
- If the repeated calls return equal values, this branch returns zero.
- No element comparison occurs after entering this branch, even if the repeated count results become equal.

If the initial two results are equal, the function calls F-10b760faca36 once more for the left handle. This third result initializes the first element pass. The initial equality result is not reused as that pass’s count.

### 3. First reverse-indexed element pass

The first pass decrements its current index before each element access. A nonnegative initial count of \(N\), under stable behavior and nonoverflowing arithmetic, produces indices \(N-1\) through 0.

For each index:

1. F-6bbddce1b930 obtains the left element using the current left handle.
2. F-6bbddce1b930 obtains the right element using the current right handle.
3. The left payload tag is read.
4. The right payload tag is read.
5. Their difference is computed.

A nonzero tag difference is returned immediately. Otherwise, the matching tag selects one of the following payload comparisons.

#### Tag 19

F-4d3d7560af30 is called with the left payload address only. Its returned result becomes the payload comparison result.

The right payload is not passed to this dependency. This function performs no additional right-side content comparison for this branch before considering the element auxiliary values.

It is not established whether F-4d3d7560af30 obtains other relevant state indirectly.

#### Tag 22 with primary classification 48

For tag 22, the function calls F-2eb743f270d1 with the left element’s primary value.

If the result is 48, the payloads are treated as byte-sequence records:

- The right leading scalar is read as a candidate length.
- The left leading scalar is compared with it.
- If unequal, a left-minus-right leading-scalar difference becomes the payload comparison result.
- If equal, bytes are examined in ascending index order, starting at zero.

The byte-loop bound is the **current left leading scalar**, checked again on each iteration. It is not a cached length. The right length is not revalidated for each byte.

For each byte position:

1. A left byte is read through the current left data address.
2. F-1aef7372e8d8 is called with that byte value.
3. A right byte is read through the current right data address.
4. F-1aef7372e8d8 is called with that byte value.
5. The two dependency results are compared.

The first differing dependency results produce their left-minus-right difference. If the loop completes without a difference, the payload comparison result is zero.

The dependency’s transformation is unknown. In particular, the evidence does not establish case folding, character encoding, locale handling, or byte-identity comparison.

Equal zero lengths cause no byte reads. Equal negative lengths also cause no byte reads because the ascending loop starts at zero and uses a signed bound.

#### All other payload cases

This path applies to:

- Matching tags other than 19 and 22.
- Matching tag 22 when F-2eb743f270d1 does not return 48.

The left and right payload leading scalars are subtracted. A nonzero difference is returned immediately, without comparing element auxiliary values.

If the leading scalars match, F-7f14f1ab6851 is called with the left and right payload data values. Its result becomes the payload comparison result.

This function does not establish whether those data values are addresses, identifiers, strings, or another representation for this dependency.

#### Element auxiliary comparison

For the payload-comparison branches that reach this stage:

- A nonzero payload comparison result is returned unchanged.
- A zero payload comparison result causes comparison of the element auxiliary fields at offset 8.
- A nonzero left-minus-right auxiliary difference is returned.
- A zero auxiliary difference advances to the next lower element index.

The auxiliary comparison therefore follows the tag-19 result, specialized byte comparison, or F-7f14f1ab6851 result. It does not occur after an already nonzero tag difference or generic leading-scalar difference.

### 4. Second reverse-indexed element pass

When the first pass exhausts its indices without returning, F-10b760faca36 is called again with the current left handle.

This new result initializes a separate reverse-indexed pass. It is not required by this function to equal the first pass’s count or any earlier count.

For each nonnegative index:

1. F-6bbddce1b930 obtains the left element.
2. F-6bbddce1b930 obtains the right element.
3. F-9878699bd825 is called with their primary values.
4. A nonzero result is returned unchanged.

If all visited primary comparisons return zero, the function returns zero.

This pass does not recompare tags, payload fields, auxiliary fields, or right-side collection count. Consequently, for stable collections, every first-pass comparison has precedence over every second-pass primary comparison.

## Side effects

### Direct effects

The function contains no explicit writes to caller-provided storage, no explicit allocation, and no explicit release operation.

It directly reads the input references, descriptors, selected pointer chains, collection-handle cells, returned elements, payload fields, and—conditionally—individual bytes.

No ownership transfer is explicitly performed. Returned element addresses are used but are not explicitly released by this function.

### Dependency-mediated effects

No dependency is established to be pure, nonallocating, nonmutating, or ownership-neutral. A dependency may affect later reads or later dependency results.

Important sequencing consequences include:

- The first left count call occurs before the first right count call.
- Unequal initial counts cause two additional count calls, left then right.
- Element retrieval occurs left then right.
- Payload fields are examined after both retrieval calls.
- In the byte branch, the left transformation call occurs before the right byte is read.
- Auxiliary fields are read only after a payload comparison has produced zero.
- The second pass begins with a fresh left count call.

The function retains addresses across calls, but retention does not establish that the referenced storage remains live. Required lifetime and mutation constraints are documented_only.

## Dependencies

| Opaque function ID | Use in this function | Unestablished behavior |
|---|---|---|
| F-10b760faca36 | Produces a 32-bit count-like value from a collection handle | Valid range, stability, errors, side effects |
| F-6bbddce1b930 | Produces an element address from a handle and index | Bounds behavior, lifetime, ownership, mutation |
| F-2eb743f270d1 | Classifies the left primary value for tag 22; result 48 selects byte comparison | Classification meaning, other results, side effects |
| F-1aef7372e8d8 | Transforms or classifies each byte before comparison | Mapping, result range, external-state dependence |
| F-7f14f1ab6851 | Compares or otherwise processes two payload data values | Argument interpretation, ordering semantics, errors |
| F-9878699bd825 | Produces the final-pass result from two primary values | Ordering properties, errors, side effects |
| F-4d3d7560af30 | Produces the tag-19 result from the left payload alone | Right-side relevance, external state, semantic meaning |

F-ff4aeaa3fa12 does not directly call itself in the supplied evidence.

## Acceptance examples

These examples are conditional expectations from the represented control flow, not observed runtime results. Unless stated otherwise, referenced storage remains valid and dependency calls do not mutate relevant state. Arithmetic examples avoid overflow.

1. **Different descriptor kinds.**  
   Left kind 2 and right kind 1 return 1. Neither backing-object chain is accessed.

2. **Equal unsupported kinds.**  
   Two descriptors of kind 7 return zero. Their backing-object fields need not be read on this path.

3. **Stable unequal counts.**  
   For matching supported kinds, suppose every left count call returns 3 and every right count call returns 5. Four count calls occur in left, right, left, right order; the result is −2. No element is retrieved.

4. **Unequal counts followed by equal counts.**  
   Initial results are 3 and 5; repeated results are 4 and 4. The function returns zero without inspecting elements. It does not restart comparison using count 4.

5. **Stable empty collections.**  
   Both initial counts and both later left pass counts are zero. There are four count calls total, no element retrievals, and a zero result.

6. **Reverse-index priority.**  
   Stable collections contain two elements. If the tags at index 1 are 22 on the left and 19 on the right, the function returns 3. Index 0 and the second pass are not reached.

7. **Tag-19 result precedes auxiliary comparison.**  
   At a visited element, both tags are 19 and F-4d3d7560af30 returns −6. The function returns −6 without comparing that element’s auxiliary fields or calling a right-payload counterpart.

8. **Transformed byte comparison.**  
   Both tags are 22, classification returns 48, and lengths match. At the first differing transformed byte, the left transformation returns 12 and the right returns 20. The payload result is −8; later bytes and auxiliary values are not examined.

9. **Zero-length byte sequences.**  
   Under the same specialized branch, equal zero lengths perform no byte transformation calls. Left auxiliary value 9 and right auxiliary value 4 then produce result 5.

10. **Generic leading-scalar precedence.**  
    On the generic payload path, leading scalars 10 and 7 return 3 immediately. F-7f14f1ab6851 and the auxiliary comparison are skipped.

11. **Second-pass tie breaking.**  
    All first-pass comparisons are zero. The refreshed left count is 2. F-9878699bd825 returns zero at index 1 and −4 at index 0. The function returns −4.

12. **Negative byte lengths.**  
    If specialized byte lengths are equal at −1 and storage is otherwise valid, no bytes are examined. Comparison proceeds to the auxiliary fields. This is established control flow, not evidence that negative lengths are valid domain input.

## Uncertainties

### Established static evidence

The represented accesses and branch structure establish the comparison stages, selected offsets, call order, early returns, and absence of explicit allocation, release, or storage writes.

The integer operations are represented as 32-bit operations. Loop conditions use signed comparisons. Several differences can therefore be larger in magnitude than one.

### Inferred intent

The function appears intended to compare structured collections, with payload-sensitive ordering followed by primary-value tie breaking. The count-like dependency and indexed retrieval support that interpretation.

This intent does not prove a conventional comparator contract. Tag 19’s left-only call and repeated count queries are material limitations.

### documented_only unknowns

- Overflow behavior for subtraction, decrement, and byte-index increment.
- The sign-extension or zero-extension rule used when supplying a byte to F-1aef7372e8d8.
- Required alignment and valid address ranges.
- Dependency errors, exceptions, nonlocal exits, or invalid-result behavior.
- Whether dependencies preserve globals, retain arguments, invalidate storage, or acquire or release ownership.
- Whether negative counts or lengths are legal.
- Whether collections must have equal, stable sizes throughout both passes.
- Whether any payload tag implies additional undisclosed invariants.
- Whether returned results satisfy transitivity, antisymmetry, or stable equality.

In particular, subtracting extreme 32-bit values or decrementing the most negative signed value is not assigned a portable mathematical interpretation here. Acceptance of such inputs requires additional execution-model evidence.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the supplied opaque dependency mapping.

The evidence establishes local access patterns and control flow. It does not include dependency bodies, authoritative object definitions, caller requirements, ownership documentation, or runtime observations.

No runtime tests have been performed. The acceptance examples are conditional deductions, not execution evidence.
