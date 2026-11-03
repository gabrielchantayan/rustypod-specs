<!-- Trusted provenance: F-583a7bfa3cfe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-583a7bfa3cfe

## Purpose

F-583a7bfa3cfe conditionally transforms a four-word record in place. It uses two calls to F-74efb2674226 to derive a difference. When that difference is nonzero, it makes four calls to F-73d099a86adb and stores their results, negating two of them. It reports completion with status 0. A null record pointer or a zero difference produces status 6.

The arrangement resembles inversion of a two-by-two matrix: diagonal terms exchange roles, off-diagonal terms are negated, and a determinant-like quantity supplies a common second argument. This is **inferred intent**, not an established arithmetic contract. The supplied evidence does not establish what either dependency computes, how words encode numbers, or what rounding or exceptional arithmetic rules apply.

## Interface

### Input record

The function accepts one pointer to a record containing at least four consecutive 32-bit words. This contract labels those words by position:

| Word | Byte offset | Role in the visible transformation |
|---|---:|---|
| W0 | 0 | First input to the first preliminary call; later saved and used by the final transformation call |
| W1 | 4 | First input to the second preliminary call; first word explicitly overwritten |
| W2 | 8 | Second input to the second preliminary call; second word explicitly overwritten |
| W3 | 12 | Second input to the first preliminary call; input to the third transformation call |

These labels describe storage positions, not established matrix coordinates or numeric types. The visible record occupies 16 bytes. No record length, format tag, or separate output pointer is supplied.

A null pointer is explicitly accepted as an error case. A nonnull pointer must support the reads and writes actually reached. For a successful transformation, all four words must be readable and writable. There is no visible validation of allocation size, alignment, address validity, or memory protection.

The routine passes word **values**, not the record pointer itself, to its dependencies. This does not establish that the dependencies cannot access the record through globals or other aliases.

### Output

The visible return value is a 32-bit status:

- **0:** the nonzero-difference path reached its final store and returned normally.
- **6:** the pointer was null, or the difference between the two preliminary results was zero.

Status 0 does not independently certify mathematical invertibility, numerical accuracy, or successful arithmetic inside a dependency. No dependency status is separately checked.

The record is the in-place output. The function does not visibly allocate, replace, or return a record.

### Numeric widths

The record elements and outward result are represented as 32-bit words in the supplied evidence. Preliminary dependency results and the difference are represented as signed integer values on the apparent 32-bit target. The first two transformation results also pass through signed integer values before negation.

The numeric encoding of record words is unknown. In particular, they must not be assumed to be ordinary signed integers, floating-point values, or a particular fixed-point format.

## Behavior

### 1. Null-pointer rejection

The pointer is checked before any record access or dependency call.

For a null pointer, the function returns 6 without reading or writing the record and without invoking either named dependency.

### 2. Preliminary calls and difference

For a nonnull pointer, the function performs these dependency calls in order:

1. F-74efb2674226 receives W0 as its first argument and W3 as its second argument.
2. After that call returns, F-74efb2674226 receives W1 as its first argument and W2 as its second argument.

The first result is retained across the second call. The function then subtracts the second result from the first.

The second call’s arguments are read after the first call returns. Consequently, if the first call can modify the record through an alias, the second call observes the values present after that modification. The record is not visibly snapshotted at entry.

The evidence identifies which word supplies each argument, but does not establish the relative read order of the two arguments within an individual call.

### 3. Zero-difference rejection

If the computed difference is zero, the function returns 6.

On this path:

- Both preliminary calls have occurred.
- No transformation call occurs.
- The function itself has performed no record stores.

It is not unconditional evidence that the record or other process state remains unchanged: either preliminary call may have side effects that are not described by the supplied material.

### 4. Nonzero-difference transformation

If the difference is nonzero, that computed value is retained as the second argument for all four calls to F-73d099a86adb. The function does not recompute it after changing the record.

The visible sequence is:

1. Read W1 and call F-73d099a86adb with that value and the retained difference.
2. Negate the returned signed integer value and store the result into W1.
3. Read W2 and call F-73d099a86adb with that value and the retained difference.
4. Negate the returned signed integer value and store the result into W2.
5. Read W0 and retain that word for the final transformation call.
6. Read W3 and call F-73d099a86adb with that value and the retained difference.
7. Store that call’s returned word into W0 without an explicit negation.
8. Call F-73d099a86adb with the retained W0 word from step 5 and the retained difference.
9. Store that call’s returned word into W3 without an explicit negation.
10. Return 0.

The explicit store order is therefore **W1, W2, W0, W3**.

Two timing distinctions are important:

- The W0 value used for the final call is captured only after the first two transformation calls and their stores. It is not necessarily the entry value of W0.
- The final call uses that captured value, not the newly written W0 and not a fresh read of W0 after the third transformation call.

Likewise, W2 is read after the first transformation call and W1 store; W3 is read after the second transformation call and W2 store. Dependency side effects can therefore influence subsequent inputs.

### 5. Arithmetic boundaries

The subtraction and the two negations require care at signed 32-bit boundaries.

For preliminary results whose mathematical difference fits within signed 32-bit range, the zero/nonzero decision follows that difference directly. For differences outside that range, the supplied representation does not independently establish a portable overflow rule. A target-level wrapping interpretation must not be silently substituted for a defined source-level arithmetic contract.

The same limitation applies when either of the first two transformation calls returns the signed minimum value, −2,147,483,648. Its mathematical negation is not representable as a signed 32-bit integer. The exact resulting word or exceptional behavior is **documented_only** pending an authoritative arithmetic or machine-level contract.

No wider intermediate, saturation, explicit overflow check, or arithmetic-error branch is visible.

## Side effects

### Record mutation

On a normally completing nonzero path, the function explicitly writes each record word once, in the order W1, W2, W0, W3. The operation is not visibly transactional.

If a dependency fails to return normally after earlier stores, those stores have already occurred. There is no visible rollback. Concurrent or reentrant observers may encounter a partially transformed record; no locking or atomic publication mechanism is shown.

On a null-pointer path there are no record accesses. On a zero-difference path there are reads but no direct record writes.

### Ownership and lifetime

No direct allocation, deallocation, ownership transfer, or pointer replacement is visible. The caller remains responsible for providing suitable storage.

The dependencies’ ownership effects are unknown. Their word arguments must not be assumed to exclude pointer-like encodings, global resource access, or memory release. If a dependency invalidates storage that this function later accesses, the visible routine provides no recovery mechanism.

### Other state

There are no explicit global writes or external operations in the supplied body. Any such effects performed by the dependencies remain unknown. The function does not visibly save or restore global state around calls.

## Dependencies

### F-74efb2674226

Established use:

- Called twice for every nonnull input, provided the first call returns normally.
- Receives two 32-bit record values per call.
- Its results participate in a signed integer subtraction.
- Call order is the W0/W3 pair, then the W1/W2 pair.

Unknowns include its arithmetic meaning, overflow behavior, side effects, memory access, ownership effects, and failure behavior.

A multiplication-like interpretation would support the matrix-inversion hypothesis, but is not established.

### F-73d099a86adb

Established use:

- Called only when the preliminary difference is nonzero.
- Called four times on a normally completing successful path.
- Receives a record-derived word as its first argument and the retained difference as its second argument.
- Its first two results are negated before storage; its final two results are stored without explicit negation.

Unknowns include its arithmetic meaning, numeric encoding, rounding, exceptional inputs, side effects, ownership effects, and failure behavior.

A division-like interpretation would support the matrix-inversion hypothesis. A nonzero second argument alone does not establish that every call is valid.

## Acceptance examples

These are conditional behavioral examples derived from static evidence. They are not runtime test results. Arithmetic examples avoid signed overflow.

### Null input

Given a null pointer:

- The result is 6.
- Neither dependency is called.
- No record word is accessed.

### Equal preliminary results

Given valid record storage, suppose the preliminary calls return 12 and 12 and return normally:

- The difference is zero.
- The result is 6.
- F-73d099a86adb is not called.
- There are no direct record stores.

If the preliminary calls also leave the record unchanged, its contents remain unchanged. That additional condition is necessary.

### Successful transformation without dependency mutation

Suppose the entry words are W0 = 10, W1 = 20, W2 = 30, and W3 = 40. Assume all dependencies return normally and do not independently modify the record.

If the preliminary results are 9 and 4, the retained difference is 5. The four transformation calls receive, in order:

1. 20 and 5.
2. 30 and 5.
3. 40 and 5.
4. 10 and 5.

If their respective results are 2, −3, 8, and 6, the final record is:

| Word | Final value |
|---|---:|
| W0 | 8 |
| W1 | −2 |
| W2 | 3 |
| W3 | 6 |

The result is 0. The explicit writes occur in the order W1 = −2, W2 = 3, W0 = 8, W3 = 6.

### Mutation before W0 is captured

Suppose the first transformation call changes W0 through an external alias, and the second transformation call leaves that change intact. The value captured for the final transformation call is the changed W0 value present after the second call and W2 store, not necessarily the entry value.

This remains a consequence of read timing, even if such dependency mutation is undesirable.

### Mutation after W0 is captured

Suppose W0 has been captured, then the third transformation call modifies W0 through an alias. On normal return, the function overwrites W0 with that call’s result. The final transformation call still receives the earlier captured word.

### Interruption after a partial update

If the first transformation call returns normally and its result is stored into W1, but the second transformation call does not return normally, the W1 store has already happened. The routine does not visibly restore it or return status 6 for that interruption.

## Uncertainties

The following are **documented_only**:

- The arithmetic semantics of both dependencies.
- The record’s numeric encoding and mathematical layout.
- Overflow behavior for the preliminary subtraction and signed negations.
- Argument-read ordering within individual calls.
- Dependency access to globals, aliases, resources, or record storage.
- Dependency failure, exception, or non-return behavior.
- Concrete alignment requirements and behavior for invalid nonnull pointers.
- Whether external synchronization or alias restrictions apply.
- Whether the inferred matrix operation matches the intended API.

The strongest safe contract is an ordered, dependency-driven transformation of four 32-bit words. A complete mathematical inverse contract requires dependency specifications and a numeric representation contract.

## Evidence

**Established static evidence:** the supplied function body supports the pointer check, four-word access layout, dependency call order, difference-based branch, negation placement, delayed capture of W0, explicit store order, and return values 0 and 6.

**Inferred intent:** the paired products or product-like operations, common difference argument, off-diagonal negations, and diagonal exchange resemble two-by-two matrix inversion. Dependency behavior is required to validate that interpretation.

**Verification boundary:** no runtime tests have been performed. Acceptance examples state consequences under explicit assumptions; they are not observed executions. No independent dependency implementations or authoritative arithmetic contracts were supplied.
