<!-- Trusted provenance: F-b64d1c3b5c3e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b64d1c3b5c3e

## Purpose

F-b64d1c3b5c3e conditionally replaces a two-word value in an implicitly addressed state block.

The replacement is selected when the current value is strictly less than a comparison value supplied through two explicit arguments. Each compared value consists of an unsigned 32-bit low word and a signed 32-bit high word. Together, these words define a signed 64-bit value.

When replacement is selected, the new words come from two other fields in the state block. The explicit arguments control the decision; they are not the values written.

Equality and greater-than cases perform no represented state writes. The function returns no explicit result.

This comparison-and-copy behavior is established by the supplied static representation. A domain interpretation such as threshold handling, bound maintenance, or reset processing is not established.

## Interface

### Explicit inputs

| Input | Width and interpretation | Role |
|---|---|---|
| Comparison low word | Unsigned 32-bit integer | Low word of the comparison value |
| Comparison high word | Signed 32-bit integer | High word of the comparison value |

The low word ranges from 0 through 4,294,967,295. The high word ranges from −2,147,483,648 through 2,147,483,647.

The mathematical comparison value is:

**comparison high word × 4,294,967,296 + comparison low word**

This definition covers the signed 64-bit range from −9,223,372,036,854,775,808 through 9,223,372,036,854,775,807. It describes the comparison semantics without requiring a native 64-bit operation.

### Implicit state input and layout

The function uses a state-base address already present in the incoming execution context. Neither explicit argument supplies this address. The function does not initialize or validate it.

The relevant layout is:

| Byte offset from state base | Access width | Meaning within this function |
|---|---|---|
| `0x38` | 4 bytes | Replacement low-word bit pattern |
| `0x3C` | 4 bytes | Replacement high-word bit pattern |
| `0x40` | 4 bytes | Current low word, compared as unsigned |
| `0x44` | 4 bytes | Current high word, interpreted as signed |

The mathematical current value is:

**current high word × 4,294,967,296 + current low word**

The replacement fields are copied as raw 32-bit words. Their interpretation does not affect whether or how they are copied.

These offsets establish the roles of the words, not the byte order within each word. The containing object’s type and total extent are unknown.

### Outputs

There is no explicit return value, success indicator, comparison result, or error status.

The represented data output is one of:

- No writes to either current-word field.
- A four-byte write to the current low-word field followed by a four-byte write to the current high-word field, using the corresponding replacement fields.

An unchanged final memory image does not necessarily imply that no writes occurred: the replacement words may equal the words already stored.

### Preconditions and ownership

Normal completion requires:

- A usable implicit state-base address.
- Accessible four-byte current-word fields for the initial reads.
- On the replacement path, accessible four-byte replacement fields and writable four-byte current fields.
- Address calculations and word accesses valid for the execution environment.

The function represents no validation of these conditions. Exact alignment requirements, address-space restrictions, and invalid-access behavior are not established.

No allocation or deallocation operation is represented. Ownership of the state and any ownership significance of its fields are not established.

## Behavior

### Initial accesses

The represented access order begins with:

1. Read the current high word at offset `0x44`.
2. Read the current low word at offset `0x40`.

The low word is compared with the explicit comparison low word using unsigned 32-bit ordering. This produces a borrow quantity:

- 1 if the current low word is less than the comparison low word.
- 0 otherwise.

The two current words are not demonstrated to form an atomic snapshot. If the environment permits an intervening modification, the decision can use words obtained from different state versions. The supplied representation does not establish whether such concurrent modification is permitted or well-defined.

### Replacement decision

Replacement is selected exactly when either:

- The signed current high word is less than the signed comparison high word; or
- The high words are equal and the unsigned current low word is less than the comparison low word.

Equivalently, the signed 64-bit current value is strictly less than the signed 64-bit comparison value.

The high-word comparison takes precedence over low-word ordering. A smaller signed high word selects replacement even if the current low word is numerically larger. A larger signed high word prevents replacement even if the current low word is smaller.

Equality of both words does not select replacement.

### Widths, borrow, and overflow

The decision is represented through fixed-width high-word arithmetic, the low-word borrow, and signed-overflow predicates.

The corresponding mathematical high-word difference is:

**current high word − comparison high word − borrow**

Its range is −4,294,967,296 through 4,294,967,295. This exceeds the signed 32-bit range.

The represented condition combines the sign of the wrapped 32-bit result with overflow information from the subtraction stages. That correction is material: the sign of the wrapped result alone does not establish the comparison.

In particular:

- Low-word comparison is unsigned.
- High-word interpretation is signed.
- Borrow is numerically 0 or 1.
- Signed high-word boundary cases retain the strict signed 64-bit ordering described above.
- Arithmetic overflow does not select a separate error or recovery path.
- No saturation or clamping is represented.

This is a contract for the fixed-width behavior expressed by the supplied representation, not a requirement to reproduce its arithmetic notation in an environment with different overflow rules.

### Selected replacement

When the condition is true, the represented memory accesses occur in this order:

1. Read the four-byte replacement low word at offset `0x38`.
2. Write that word to offset `0x40`.
3. Read the four-byte replacement high word at offset `0x3C`.
4. Write that word to offset `0x44`.

The replacement high word is not captured before the current low-word write. Describing the operation as first reading an entire replacement pair and then writing an entire current pair would lose this ordering.

The function does not:

- Write either comparison argument into the state.
- Compare the replacement pair against the old current pair.
- Compare the replacement pair against the comparison arguments.
- Recheck the condition after writing.
- Retry the operation.

Consequently, the replacement value may be less than, equal to, or greater than either value involved in the decision.

### Unselected replacement

When the current value is greater than or equal to the comparison value:

- Neither replacement field is read by the represented body.
- Neither current field is written.
- The function returns without an explicit result.

Replacement-field access is therefore conditional. This does not establish that an otherwise invalid containing object is acceptable under every possible execution environment.

## Side effects

The only represented memory writes are the conditional four-byte writes at offsets `0x40` and `0x44`.

The fields at offsets `0x38` and `0x3C` are read but not written by this body. No accesses to other state fields are represented.

The writes are ordered low word first, high word second. No lock, transaction, memory barrier, or atomic two-word operation is shown. If the execution environment permits observation between these writes, an observer may see the new current low word paired with the previous current high word.

Similarly, the replacement fields are read separately. The body does not guarantee that their contents remain stable between those reads.

No rollback mechanism is represented. If execution is interrupted by an access failure after the first write, the body supplies no compensating write. Whether such a failure occurs, how it is reported, and what effects remain observable are environment-dependent and **documented_only**.

The access sequence describes the supplied representation. It does not establish physical bus transactions, machine-level atomicity, compiler concurrency guarantees, or synchronization with other actors.

No allocation, deallocation, I/O, callback, or explicit external function call is represented. Ownership consequences cannot be inferred from this absence. Register and processor-flag preservation outside the represented memory behavior is also not established.

## Dependencies

No separate callable dependency is identified in the supplied body.

The behavior depends on:

- An externally established implicit state-base address.
- Four-byte memory accesses at the specified offsets.
- Unsigned 32-bit low-word comparison.
- Signed interpretation of 32-bit high words.
- Fixed-width subtraction and signed-overflow information sufficient to implement the stated comparison.

The overflow predicates express arithmetic semantics in the supplied representation; they do not establish calls to independently implemented routines.

No unknown callee is assumed to preserve globals, retain state, or allocate or release memory. No dependency identifier beyond F-b64d1c3b5c3e is needed for the supplied body.

## Acceptance examples

These examples are conditional expectations derived from static analysis, not observed executions. Assume valid, accessible, stable storage and no intervening modifications unless an example states otherwise.

For the following table, let the replacement low word be `0x89ABCDEF` and the replacement high-word bit pattern be `0x01234567`.

| Current high | Current low | Comparison high | Comparison low | Required behavior |
|---|---|---|---|---|
| 0 | `0x00000005` | 0 | `0x00000006` | Replace both current words |
| 0 | `0x00000006` | 0 | `0x00000006` | No writes |
| 0 | `0x00000007` | 0 | `0x00000006` | No writes |
| −1 | `0xFFFFFFFF` | 0 | `0x00000000` | Replace; the current value is −1 |
| 0 | `0x00000000` | −1 | `0xFFFFFFFF` | No writes; 0 is greater than −1 |
| 5 | `0x00000000` | 4 | `0xFFFFFFFF` | No writes; the greater high word controls |
| −2 | `0xFFFFFFFF` | −1 | `0x00000000` | Replace; the smaller signed high word controls |
| 3 | `0x7FFFFFFF` | 3 | `0x80000000` | Replace; low-word ordering is unsigned |

For each selected replacement, normal completion leaves current low `0x89ABCDEF` and current high bits `0x01234567`.

### Signed boundaries

With current high −2,147,483,648 and current low `0x00000000`, comparison high 2,147,483,647 and comparison low `0xFFFFFFFF` must select replacement.

Reversing those pairs must produce no writes.

These expectations distinguish overflow-aware signed ordering from a decision based only on the sign of a wrapped high-word subtraction.

### Replacement is independent of the comparison value

With current value 5, comparison value 6, and replacement value 100, the function must write the replacement pair representing 100.

It must not write 6, reject 100 for exceeding the comparison value, or perform a second comparison.

### Selected writes can preserve the memory image

With current value 5, comparison value 6, and replacement value 5, replacement is selected. Both represented writes still occur, although the final current-word bit patterns equal their initial values.

### Conditional and ordered accesses

When the current pair equals the comparison pair, neither replacement field is read by the represented body.

When replacement is selected, the replacement-low read and current-low write precede the replacement-high read and current-high write. Reading both replacement words before either write does not match the represented sequence.

## Uncertainties

The following are **documented_only** because the supplied static representation does not establish them:

- How the implicit state base is supplied, initialized, or maintained.
- The containing object’s type, total size, owner, and ownership protocol.
- Whether any field has ownership significance.
- The domain meaning of the current, comparison, or replacement values.
- The origin, validity requirements, or intended invariant of the replacement pair.
- Byte order within each four-byte word.
- Exact alignment requirements and address-space constraints.
- Invalid-access, fault-reporting, interruption, and recovery behavior.
- Whether surrounding execution guarantees stable state during the operation.
- Whether fields have memory-mapped or access-triggered effects.
- Machine-level access granularity, atomicity, and synchronization guarantees.
- Register and processor-flag preservation beyond the represented behavior.

No domain meaning or ownership model is required to state the comparison-and-copy behavior.

## Evidence

**Established static evidence:** The supplied body presents two explicit 32-bit scalar arguments and an implicit state base. It reads the current high word at offset `0x44`, then the current low word at offset `0x40`. It uses unsigned low-word comparison and overflow-aware high-word arithmetic to choose a branch. The selected branch copies four-byte words from `0x38` to `0x40`, then from `0x3C` to `0x44`. No explicit return value, allocation, deallocation, or external call is represented.

**Derived behavioral conclusion:** The condition implements strict signed 64-bit ordering of the current and comparison pairs. The selected update obtains its contents from the replacement fields rather than the comparison arguments and performs the low-word copy before the high-word copy.

**Inferred intent:** The function may participate in maintaining a larger state invariant. Neither that invariant nor a specific domain purpose is established.

**Verification status:** Static analysis only. No runtime tests have been performed. All acceptance examples are conditional requirements derived from the supplied representation, not measured results.
