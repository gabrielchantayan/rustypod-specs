<!-- Trusted provenance: F-18cbf7a56e3c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-18cbf7a56e3c

## Purpose

F-18cbf7a56e3c performs a reverse-order scan of a caller-supplied sequence. The scan length is derived from the descriptor’s first field and a global count multiplier. Each visited element is multiplied by a second global multiplier.

The function returns:

- **1** if the derived scan length is less than one, or if every visited element produces a zero product.
- **0** as soon as a visited element produces a nonzero product.

This is a test of **products**, not unconditionally a test that the sequence elements themselves are zero. Its interpretation depends on the element multiplier and on the applicable arithmetic semantics.

The supplied static representation establishes the control flow. It does not establish the intended application-level meaning of the descriptor or global values.

## Interface

### Input descriptor

The function accepts one address referring to a descriptor with two consecutive integer fields:

| Relative location | Meaning |
|---|---|
| First integer field | Signed quantity used to derive the scan length |
| Second integer field | Integer-encoded base address of the sequence |

The sequence is accessed at a fixed stride of **four bytes**. Element zero is at the encoded base address; element one is four bytes later, and so on.

The representation is consistent with a 32-bit layout: two four-byte descriptor fields, signed 32-bit sequence elements, and a 32-bit integer-encoded address. The four-byte sequence stride is explicit. The exact target integer, pointer, alignment, and address-conversion rules are not independently supplied; the 32-bit descriptor interpretation is therefore a target-layout assumption, not a portable interface guarantee.

### Global inputs

Two distinct global values participate:

- **Count multiplier:** read when forming the initial scan length.
- **Element multiplier:** used afresh in each reached element-test expression.

No parameter supplies either multiplier.

### Output

The result occupies a four-byte return value. Every explicit return supplies exactly **0** or **1**.

There is no returned element index, count, pointer, or diagnostic.

### Preconditions

For a defined execution:

- The descriptor address must permit the initial first-field read.
- Initial multiplication and, when needed, negation must have defined behavior under the target arithmetic rules.
- If a scan iteration is reached, the descriptor’s second field must be readable and usable as an address under the target rules.
- Every reached sequence address must be valid for an integer read and meet any target alignment requirements.
- Index scaling, address formation, and element multiplication must have defined behavior.

The supplied representation contains no validation enforcing these preconditions.

An empty scan does not evaluate the descriptor’s second field or read sequence storage. It still requires the initial first-field read and count multiplication.

## Behavior

### Deriving the scan length

The function multiplies the count multiplier by the descriptor’s first field.

If the resulting signed value is negative, it negates that value. Otherwise, it leaves the value unchanged. The resulting value becomes the scan’s working count.

For representable multiplication and negation, this yields the mathematical absolute value of the initial product. Either sign of a nonzero first field or count multiplier can therefore produce a positive scan length.

The function does not establish an independent upper bound and does not compare the derived count with any buffer capacity.

### Empty-scan decision

Before accessing any sequence element, the function tests whether the working count is less than one.

If it is, the function returns **1** immediately.

Under ordinary representable arithmetic, this occurs when the initial product is zero. The actual branch condition is broader: it accepts any working value below one. Whether exceptional arithmetic can produce such a value is not resolved by the supplied evidence.

### Reverse scan

For a positive working count, the function reduces the count by one and uses the reduced value as the element index.

Consequently, for a derived length of five, the visited indices are four, three, two, one, and zero, unless an earlier visit terminates the scan.

For each visit, the element address is formed from the descriptor’s second field plus four times the current index. The integer at that address is multiplied by the element multiplier.

- A **nonzero** product causes an immediate return of **0**.
- A **zero** product allows the scan to continue.

After a zero product at index zero, the next count check returns **1** without another sequence read.

Each iteration therefore decreases the remaining count. With a finite positive initial count and defined execution throughout, the function either stops at the first nonzero product in reverse order or exhausts the selected range.

### What constitutes a passing element

The established predicate is that the element product equals zero.

When multiplication is representable and the element multiplier is nonzero, that predicate is equivalent to the element being zero.

When the element multiplier is zero, every reached element produces zero under ordinary arithmetic, including nonzero elements. This does **not** remove the sequence-read requirement from the supplied behavior: the element read remains part of each reached test expression.

For multiplication outside the signed representable range, no portable zero-test interpretation is established. In particular, the contract does not silently impose wrapping arithmetic or assume that two nonzero operands always produce a nonzero machine-width result.

### Changing inputs during a scan

The initial count product is computed once. Later changes to the descriptor’s first field or the count multiplier do not trigger an explicit recomputation of the working count.

The descriptor’s second field and the element multiplier occur in every reached element-test expression. The representation does not show a separate stable copy of either value spanning the scan.

Accordingly, stable descriptor and global values are a sufficient condition for the simple fixed-array interpretation. Behavior under concurrent mutation, device-backed storage, or other externally changing memory depends on target access and synchronization semantics that are not supplied.

## Side effects

### Reads and their ordering

At the logical-expression level, the function performs these reads:

1. The count multiplier and descriptor’s first field, to form the initial product.
2. If the scan is nonempty, the descriptor’s second field, to form the current sequence address.
3. The integer at that sequence address and the element multiplier, to evaluate the current product.
4. Corresponding reads for each later reached index, in descending index order.

The ordering between the two operands of either multiplication is not established. In particular, the evidence does not fix whether the element multiplier is read before or after the sequence element.

The sequence address depends on reading the descriptor’s second field. A subsequent index is reached only after the preceding product has compared equal to zero.

These are expression-level observations. The supplied representation does not establish volatile access, atomic access, exact machine-load counts, or an instruction-level memory trace.

### Writes and ownership

No explicit write to the descriptor, sequence, or either global appears.

The function does not explicitly:

- Allocate or release memory.
- Transfer ownership.
- Change the descriptor.
- Store a scan result in caller-provided storage.
- Call another function.

Its only explicit externally returned result is the four-byte value 0 or 1. Changes to its working count are internal.

The function requires readable storage for reached accesses; nothing in the supplied evidence grants it ownership of that storage.

### Failure handling

No explicit error path is present.

Invalid addresses, insufficient sequence storage, unsuitable alignment, invalid address conversion, and arithmetic exceptions are not converted into a documented return value. A return of **1** must not be interpreted as proof that the descriptor or allocation was validated.

The consequences of violating the preconditions are target-dependent and are not specified here.

## Dependencies

No outgoing function calls are visible, so there are no callee dependencies to identify.

The behavior depends on:

- The count multiplier global.
- The element multiplier global.
- The descriptor’s readable fields.
- The sequence storage selected by reached address calculations.
- Target integer, pointer, alignment, overflow, and memory-access rules.

There is no basis for assuming that an unknown callee preserves globals or releases memory: no such callee appears in the supplied representation.

## Acceptance examples

These examples are conditional contracts, not test results. Unless stated otherwise, they assume stable inputs, a compatible 32-bit layout, valid reached addresses, and representable arithmetic.

### Zero derived length

The count multiplier is zero and the descriptor’s first field is seven.

Expected result: **1**.

No sequence element or descriptor base-address field is evaluated by the scan. The initial first-field read is still required.

The same result follows when the first field is zero and the initial multiplication is defined.

### Negative initial product

The count multiplier is negative two and the first field is three.

The derived length is six. Expected visit order: indices five through zero.

If all six reached products are zero, the result is **1**. The negative initial product does not itself cause failure or an empty scan.

### Immediate rejection at the highest index

The derived length is three. The element multiplier is one. Elements at indices zero, one, and two are respectively zero, zero, and nine.

Expected result: **0**, after visiting index two.

Indices one and zero are not reached.

### Rejection after a zero product

The derived length is three. The element multiplier is two. Elements at indices zero, one, and two are respectively zero, five, and zero.

Expected visit order: index two, then index one.

Expected result: **0**. Index zero is not reached.

### Exhausting the range

The derived length is three, the element multiplier is negative one, and all three elements are zero.

Expected visit order: two, one, zero.

Expected result: **1**, with no read at a negative index.

### Zero element multiplier

The derived length is two and the element multiplier remains zero.

Expected result: **1**, regardless of the two readable element values, provided both element multiplications are defined.

Both selected element reads remain part of the supplied scan behavior. An invalid sequence address is not accepted merely because the multiplier is zero.

### Unrepresentable arithmetic

If the initial product is outside the signed representable range, or its negative value cannot be negated representably, there is no unconditional expected result under this contract.

Likewise, a nonrepresentable element product cannot be assigned an expected zero or nonzero classification without additional target evidence.

## Uncertainties

The following items are **documented_only**:

- **Exact target layout:** the representation strongly indicates 32-bit integers and addresses, but no independent ABI description is supplied.
- **Overflow semantics:** wrapping, trapping, or other behavior is not established for multiplication, negation, or address arithmetic.
- **Minimum signed value:** negating the most negative signed value is not representable in the same signed width; its effect is unresolved.
- **Memory semantics:** volatility, atomicity, concurrent mutation rules, and exact load counts are unknown.
- **Storage extent:** no sequence capacity or allocation boundary is supplied.
- **Fault behavior:** no evidence establishes how invalid accesses or arithmetic faults are reported by the surrounding system.
- **Application intent:** interpreting the result as an “all-zero sequence” predicate is only conditionally valid. The unconditional predicate concerns multiplied values.

None of these unknowns is replaced by a guessed fallback behavior.

## Evidence

**Established static evidence:** an initial product followed by conditional negation; a less-than-one termination check; descending indices; explicit four-byte element spacing; a per-element product compared with zero; returns of 0 and 1; and no visible calls or memory writes.

**Inferred intent:** a reverse-order universal zero-product check over a derived-length sequence. An ordinary all-zero check is a narrower interpretation requiring suitable multiplier and arithmetic conditions.

**Verification status:** no runtime tests have been performed. The acceptance examples describe consequences of the static control flow under their stated preconditions, not observed executions.
