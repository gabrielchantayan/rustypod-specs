<!-- Trusted provenance: F-7f52c3bdf9f5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7f52c3bdf9f5

## Purpose

F-7f52c3bdf9f5 selects an entry from a sentinel-terminated linked structure and requests either a whole-entry operation or a partial-entry operation. Selection uses an unsigned quantity reported by each entry and two bounds derived from a caller-supplied quantity and a requested quantity.

The function also updates a caller-owned quantity and an auxiliary output. Some conditions invoke a nonreturning dependency rather than returning an error value. Reaching the sentinel without finding a qualifying entry returns zero.

The structure resembles a resource-selection or allocation mechanism. That interpretation is **inferred intent**, not an established contract: the supplied evidence does not establish what the entries represent, whether a returned value is a pointer, or whether either final operation allocates, transfers, or releases resources.

## Interface

### Inputs and outputs

The function has five arguments:

| Argument | Established representation and use |
|---|---|
| Structure base | A 32-bit address-like value. The address at an offset of 8 bytes supplies the initial entry. The address at an offset of 4 bytes is used as the termination sentinel. |
| Quantity reference | A reference to a readable and, on selected paths, writable unsigned 32-bit word. Its initial value controls admission; a later read establishes the bounds. Subsequent updates subtract from the value present at update time. |
| Requested quantity | An unsigned 32-bit value used to establish the bounds and passed to the initial numeric dependency. |
| Context | An opaque 32-bit value forwarded to dependencies. The initial numeric call receives it twice in the visible argument list. |
| Auxiliary-output reference | A reference to a writable unsigned 32-bit word. It is initially cleared only after the admission read finds a nonzero quantity. Certain whole-entry paths overwrite it with a numeric dependency’s secondary result. |

The return value is an opaque 32-bit word:

- Zero is returned directly when traversal reaches the sentinel.
- Otherwise, the return word from the selected whole-entry or partial-entry dependency is returned unchanged.
- The evidence does not establish that zero exclusively denotes failure. A delegated operation may itself return zero.

### Representation and arithmetic

Quantities, comparisons, subtraction, and the two additions of 16 are unsigned 32-bit operations. Arithmetic wraps modulo \(2^{32}\); there is no explicit overflow or underflow check.

Address-like values and links are represented as 32-bit quantities in the supplied evidence. The following layout is established only to the extent actually accessed:

- Structure base plus 8 bytes: a 32-bit initial-entry value.
- Structure base plus 4 bytes: the sentinel address itself, not a sentinel value loaded from that location.
- Current entry plus 4 bytes: a 32-bit next-entry value, read when traversal advances.

No other structure fields, alignment guarantees, or allocation layouts are established. Address validity remains a caller/environment precondition; the evidence does not justify treating wrapped address arithmetic as access to a valid object.

### Preconditions and aliasing

On ordinary returning paths, every reference actually dereferenced must be valid for that access. The function does not visibly check pointer validity, structure integrity, link cycles, or writable capacity.

There is no visible non-aliasing check. In particular, clearing the auxiliary output can change the quantity subsequently read if both references designate the same word. Dependencies may also affect reachable memory. An implementation or consumer must not assume that the quantity remains equal to its entry value throughout the call.

## Behavior

### 1. Admission and output initialization

The function first reads the quantity word.

- If that read is zero, it invokes F-a817d8d58c10. The supplied evidence marks this invocation as nonreturning. The function does not first clear the auxiliary output or traverse the structure.
- If the read is nonzero, it writes zero to the auxiliary-output word and then reads the quantity word again.

The second read, not the admission read, determines the selection bounds. Let:

- **lower bound** be the unsigned minimum of the second quantity read and the requested quantity;
- **upper bound** be their unsigned maximum.

If the references alias, a nonzero admission value can therefore be followed by a second quantity read of zero.

### 2. Numeric admission gate

The function invokes F-d7be9cfb4000 with the following visible arguments, in order:

1. Upper bound.
2. Lower bound.
3. Requested quantity.
4. Context.
5. Context again.

A secondary result from this call controls admission:

- A zero secondary result permits traversal.
- A nonzero secondary result invokes F-a817d8d58c10, with no subsequent traversal established.

The ordinary primary return value of this numeric call is not used by the visible logic.

The evidence does not establish the dependency’s mathematical operation. Its secondary result must not be described as a remainder, status code, or alignment check without additional evidence. A zero lower bound is not locally rejected before this call; its consequences depend on the dependency.

### 3. Traversal and first qualifying entry

After the numeric gate succeeds, the function reads the initial entry from the structure base plus 8 bytes.

For each current entry:

1. It compares the entry address with the sentinel address, structure base plus 4 bytes.
2. If they are equal, it returns zero.
3. Otherwise, it invokes F-970e8f46c913 with the current entry and context.
4. It interprets that dependency’s 32-bit return as an unsigned candidate quantity.
5. If the candidate quantity is at least the lower bound, traversal stops and that entry is selected.
6. Otherwise, it reads the next-entry word from the current entry plus 4 bytes and continues.

Selection is therefore **first qualifying entry in traversal order**, not largest, smallest, or closest fit. No later entry is inspected after a qualifying entry is found.

The next link is read after the quantity-query dependency returns. If that dependency changes the link, the changed link is the one subsequently observed.

There is no explicit traversal limit. A cycle that never reaches the sentinel or a qualifying entry can prevent termination.

### 4. Interior candidate with sufficient lower-bound separation

For the selected candidate, the function first checks both of these conditions:

- Candidate quantity is strictly less than the upper bound.
- Candidate quantity is strictly greater than the unsigned 32-bit result of lower bound plus 16.

If both hold, F-d7be9cfb4000 is invoked again, this time with the candidate quantity and lower bound as the two visible arguments. Let **secondary amount** denote the unsigned 32-bit secondary result observed from this call.

There are two outcomes.

**Secondary amount at most 16**

The function:

1. Reads the current quantity word.
2. Subtracts the full candidate quantity, modulo \(2^{32}\), and writes the result back.
3. Writes the secondary amount to the auxiliary-output word.
4. Invokes F-d1b79a076570 with the selected entry and context.
5. Returns that dependency’s return word.

**Secondary amount greater than 16**

The function forms a partial amount by subtracting the secondary amount from the candidate quantity, modulo \(2^{32}\). It then:

1. Subtracts that partial amount from the current quantity word and writes the result back.
2. Invokes F-a6c4f7234593 with the selected entry, partial amount, and context.
3. Returns that dependency’s return word.

This second outcome contains no explicit auxiliary-output write beyond the earlier initialization. Its final value is not unconditionally guaranteed to remain zero, because aliases and dependencies may modify it.

### 5. Remaining candidates

If the interior-candidate condition does not hold, the function compares the candidate quantity with the unsigned 32-bit result of upper bound plus 16.

**Candidate quantity at most upper bound plus 16**

The function:

1. Invokes F-d7be9cfb4000 with the candidate quantity and lower bound as the two visible arguments.
2. Writes its unsigned secondary result to the auxiliary-output word.
3. Reads the current quantity word, subtracts the full candidate quantity modulo \(2^{32}\), and writes the result back.
4. Invokes F-d1b79a076570 with the selected entry and context.
5. Returns that dependency’s return word.

Unlike the interior whole-entry case, this path writes the auxiliary output **before** updating the quantity. That difference is observable when the references alias.

This path does not check whether the secondary result exceeds 16.

**Candidate quantity greater than upper bound plus 16**

The function:

1. Subtracts the upper bound from the current quantity word, modulo \(2^{32}\), and writes the result back.
2. Invokes F-a6c4f7234593 with the selected entry, upper bound, and context.
3. Returns that dependency’s return word.

No second numeric call occurs on this path.

## Side effects

### Explicit memory effects

The locally established order is:

- Admission quantity read.
- On nonzero admission, auxiliary-output clear.
- Quantity reread for bounds.
- Initial numeric call.
- On gate success, initial-entry read.
- Sentinel comparisons and entry-query calls.
- Next-link reads only for candidates below the lower bound.
- Selected-path quantity/output writes.
- Final delegated operation.

Quantity subtraction always uses a fresh read of the quantity word at the update point. It does not necessarily subtract from the bound-setting snapshot.

No local rollback is visible. Quantity and auxiliary-output changes precede the final delegated operation. If that operation reports failure through its return word, these changes have already occurred.

### Ownership and dependency effects

The function does not visibly allocate or release memory directly. Ownership consequences of F-d1b79a076570 and F-a6c4f7234593 are unknown.

No dependency is established as pure. Calls may affect globals, referenced memory, structure links, or ownership according to contracts not supplied here. Consequently:

- An exhausted search has no explicit quantity update, but dependency effects remain possible.
- A locally cleared output need not remain zero.
- A selected entry need not remain valid after a delegated operation unless that operation’s contract guarantees it.
- No preservation guarantee for global state follows from this evidence.

There is no visible synchronization or transactional protection.

## Dependencies

| Opaque ID | Established role at this call site |
|---|---|
| F-970e8f46c913 | Receives an entry and context. Its 32-bit return supplies the unsigned candidate quantity. |
| F-d7be9cfb4000 | Supplies a secondary result used first as a zero/nonzero admission gate and later as an unsigned amount. The initial invocation has five visible arguments; later invocations have two. |
| F-d1b79a076570 | Final operation on whole-entry paths. Receives the selected entry and context; its return word is propagated. |
| F-a6c4f7234593 | Final operation on partial-entry paths. Receives the selected entry, selected amount, and context; its return word is propagated. |
| F-a817d8d58c10 | Invoked for zero admission quantity or a nonzero initial numeric secondary result. Marked nonreturning in the supplied evidence. |

The secondary-result channel of F-d7be9cfb4000 is ABI-dependent and not fully specified by the supplied function-level material. The different visible argument lists do not establish a complete callable interface for that dependency.

## Acceptance examples

These examples are conditional contracts, not runtime observations. Unless stated otherwise, references do not alias, dependencies preserve the relevant caller memory, arithmetic does not wrap, the initial numeric secondary result is zero, and final dependencies return normally.

### Zero admission quantity

Given an initial quantity of zero, F-a817d8d58c10 is invoked. No explicit auxiliary-output clear, structure read, or entry-query call precedes it.

### Nonzero numeric gate

Given a nonzero admission quantity and an initial numeric secondary result of 1, the auxiliary output has already been cleared before F-a817d8d58c10 is invoked. No traversal occurs.

### Exhausted traversal

Given quantity 120, requested quantity 40, and the initial entry equal to the sentinel, the function clears the auxiliary output, passes the numeric gate, and returns zero without an entry-query call or explicit quantity subtraction.

### First qualifying entry

Given bounds 40 and 120 and successive candidate quantities 20, 39, and 60, the first two entries cause next-link reads. The entry reporting 60 is selected. Any later entry is not queried.

### Interior whole-entry operation

Given quantity 120, request 40, selected candidate 60, and a later numeric secondary result of 4:

- The interior condition holds because \(60 < 120\) and \(40 + 16 < 60\).
- Quantity becomes 60.
- Auxiliary output becomes 4.
- F-d1b79a076570 is invoked, and its return word is propagated.

### Interior partial-entry operation

Given quantity 120, request 40, selected candidate 75, and secondary result 19:

- The partial amount is 56.
- Quantity becomes 64.
- No explicit auxiliary-output overwrite follows initialization.
- F-a6c4f7234593 receives the selected entry, 56, and context.

### Upper threshold and boundary strictness

Given quantity 120 and request 40:

- A candidate of 56 does not satisfy the strict interior separation test. It takes the remaining-candidate whole-entry path.
- A candidate of 136 takes that same whole-entry path because the upper comparison is inclusive.
- A candidate of 137 takes the partial-entry path with amount 120, without a second numeric call.

### Wrapped upper threshold

Given quantity 4,294,967,288, request 16, candidate 32, and a successful initial gate:

- Lower bound plus 16 is 32, so the strict interior separation test fails.
- Upper bound plus 16 wraps to 8.
- Candidate 32 exceeds 8, selecting a partial operation with amount 4,294,967,288.

The function does not add a safeguard requiring that partial amount to fit within the reported candidate quantity.

### Aliased output and quantity

If both references designate a word initially containing 120, admission succeeds, then clearing the auxiliary output makes the second quantity read zero. With request 40, the bounds become zero and 40. Further behavior depends on the numeric dependency’s handling of those arguments; rejection is not locally guaranteed.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning and units of quantities, entries, context, and return values.
- The mathematical operation and complete ABI of F-d7be9cfb4000.
- Dependency effects on globals, memory, links, and ownership.
- Whether normal callers guarantee non-aliasing, acyclic lists, arithmetic bounds, or a nonzero lower bound.
- Whether the final operations can fail, and what their return words mean.
- The failure mechanism of F-a817d8d58c10.
- Whether the apparent whole-entry and partial-entry distinction corresponds to removal, splitting, allocation, or another operation.

No unstated invariant is used to eliminate unsigned wraparound, underflow, aliasing effects, or nontermination.

## Evidence

The established behavior above comes from static inspection of the supplied representation: explicit word accesses, branch conditions, arithmetic widths, dependency arguments, secondary-result uses, write ordering, and return propagation.

Resource-management terminology is identified as inferred intent rather than established semantics. Dependency contracts, full ABI details, and ownership guarantees were not supplied.

No runtime tests have been performed.
