<!-- Trusted provenance: F-a2cc02ba57dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a2cc02ba57dc

## Purpose

F-a2cc02ba57dc selects between two operations using the signed interpretation of a 32-bit displacement:

- A nonnegative displacement causes repeated calls to F-74745fd09174, with requests capped at 1,024 units.
- A negative displacement causes a call to F-cdf60417d657 and, if that call succeeds, arithmetic on its apparent output followed by a call to F-f7e2f4cd400d.

The apparent intent is forward movement by consuming data and backward movement by adjusting a position. That interpretation is **inferred**, not established. The dependency contracts do not establish the resource type, request units, transfer direction, or meaning of the final call’s arguments.

The established contract consists of the visible memory access, call selection, request accounting, status propagation, and arithmetic expressions. The negative-branch arithmetic requires explicit qualifications: signed right-shift behavior is unspecified by the supplied evidence, and a signed intermediate can overflow before an unsigned carry is added.

## Interface

### Inputs

| Input | Width | Established use |
|---|---:|---|
| Context address | 32 bits | Forms the address of a four-byte resource slot at byte offset 262,144. |
| Displacement | 32 bits | Supplies the branch classification and subsequent arithmetic operand. |

The context is represented as a signed integer but used for address formation. A valid invocation requires the effective resource-slot address to identify readable storage under the target execution model. The supplied evidence does not establish portable behavior if the signed address addition overflows, nor does it provide null, bounds, or alignment checks.

Under the represented 32-bit signed interpretation, displacement patterns divide into:

- `0x00000000` through `0x7fffffff`: nonnegative.
- `0x80000000` through `0xffffffff`: negative.

This classification relies on the target’s conversion from the displacement’s unsigned representation to its signed interpretation. The intended sign-bit classification is evident, but the supplied source representation is not a complete specification of target integer-conversion semantics.

No caller-supplied output buffer is accepted.

### Result

The result is a signed 32-bit status:

- Zero is returned after successful completion of all required chunk calls, including the zero-call case.
- A nonzero chunk or query status is returned unchanged.
- When the final dependency call is reached, its status is returned unchanged, whether zero or nonzero.

Positive and negative nonzero statuses are handled identically. Their meanings are unknown.

These return guarantees presume defined local execution and normally returning dependencies. In particular, the negative branch does not establish a portable result after signed intermediate overflow.

## Behavior

### Entry and resource capture

Before branch-specific processing, the function reads the four-byte resource value at context offset 262,144.

That value is retained for dependency calls. The function does not explicitly reread the context slot between calls. A dependency’s modification of the slot therefore does not replace the retained argument used later in the same invocation.

This is a description of the function’s own access pattern, not a restriction on dependency accesses.

A displacement of zero does not bypass the resource read. An invalid context therefore cannot be treated as safe merely because no movement was requested.

### Nonnegative displacement

A zero remaining amount causes an immediate zero return after the entry read.

For a positive remaining amount, the function selects the smaller of that amount and 1,024. It places the selection in a mutable four-byte request word, subtracts the selected amount from its local remaining amount, and calls F-74745fd09174 with:

1. The retained resource value.
2. The address of the request word.
3. The address of local scratch storage.

The request initialization and remaining-amount subtraction precede the call. The amount subtracted is the selected request before the dependency can modify it.

A nonzero dependency result ends processing immediately and is returned. A zero result permits another iteration; if nothing remains, that iteration returns zero without another dependency call.

There is no rollback or compensating call after an error.

#### Request mutation and progress

The request word is passed by address, but its post-call contents do not control progress.

Consequently, a successful call that changes the request from 1,024 to 7 still accounts for 1,024 units locally. A successful call that changes it to zero does not itself stop the loop or produce an error. A successful final call completes the operation regardless of a changed request word.

Whether these mutations signify short processing, end of input, or another condition is **documented_only**. The function supplies no independent short-operation handling.

Because each selected request is positive and no greater than the remaining amount, the unsigned subtraction does not underflow. Assuming normally returning, successful dependencies, the call count is the displacement divided by 1,024, rounded upward.

The maximum nonnegative displacement, 2,147,483,647, requires 2,097,152 calls: 2,097,151 requests of 1,024 and one request of 1,023.

### Negative displacement: query and failure

The function calls F-cdf60417d657 with the retained resource value and the address of a local four-byte unsigned word.

An adjacent four-byte signed word follows it in the represented local layout. Neither word is initialized by this function before the query.

If the query returns nonzero, that status is returned and F-f7e2f4cd400d is not called. The function does not consume the query words on that failure path.

If the query returns zero, the subsequent expressions use both words. Defined execution therefore requires suitable initialized contents for both, not merely a successful status. The available evidence does not establish that the query supplies them.

### Negative displacement: arithmetic and final call

For this description:

- \(L\) is the first query word, interpreted as unsigned 32-bit.
- \(H\) is the adjacent query word, interpreted as signed 32-bit.
- \(D\) is the displacement’s unsigned 32-bit pattern.
- \(C\) is the unsigned carry from adding \(L\) and \(D\), either zero or one.
- \(S\) is the result of shifting the negative signed displacement right by 31.

The adjusted low word is:

\[
L'=(L+D)\bmod 2^{32}.
\]

This low-word addition is unsigned. Its modular result and unsigned carry do not require an assumption about signed overflow.

The other argument is formed in two semantically distinct stages:

- First, the signed 32-bit addition \(H+S\).
- Then, addition of the carry after unsigned conversion.

The signed intermediate must not be collapsed into an unconditional modular expression.

#### Signed-shift qualification

An arithmetic signed right shift would make \(S=-1\) for every displacement on this branch.

The supplied evidence does not independently establish arithmetic right-shift semantics. Shifting a negative signed value is implementation-defined in the relevant source-language model. Consequently, neither an all-ones shift result nor the simplified subtraction-by-one formula is unconditional.

If the target uses another defined shift interpretation, its actual \(S\) governs both the signed intermediate and the final argument.

#### Signed-overflow qualification

Assuming an arithmetic right shift, the signed intermediate is \(H-1\).

For \(H\) greater than −2,147,483,648, this intermediate is representable. The final unsigned word then satisfies:

\[
H'=(H-1+C)\bmod 2^{32}.
\]

For \(H=-2,147,483,648\), the intermediate is not representable as signed 32-bit. This remains true when \(C=1\): the later carry does not retroactively make the earlier signed operation defined.

The supplied source representation alone does not establish wrapping semantics for that overflowing intermediate. At this boundary, the formula above is valid only under a separately established target execution model that wraps the signed intermediate. Without that prerequisite, no particular final argument, final call, or returned status is guaranteed.

#### Final argument positions

When evaluation is defined, F-f7e2f4cd400d receives:

1. The retained resource value.
2. The original second query word, \(H\).
3. The adjusted unsigned low word, \(L'\).
4. The result of the signed intermediate followed by unsigned carry addition.

The second argument remains the original \(H\); it is not replaced by the adjusted high word or a constant.

The evidence does not establish a relative evaluation order among these argument expressions. It does establish that they are evaluated after a successful query and before entry into the final dependency.

The final dependency’s status is returned directly. No subsequent validation, retry, or range check is shown.

#### Conditional two-word interpretation

If the query words represent the high and low halves of a 64-bit quantity, then, under arithmetic signed right shift and either a representable signed intermediate or established target wrapping, the adjusted pair represents addition of the negative displacement modulo \(2^{64}\).

This mathematical relationship does not establish that the pair is a resource position.

There is no explicit range rejection for a negative mathematical result, a wrapped high word, or displacement −2,147,483,648. Absence of a range check does not establish defined execution through signed intermediate overflow.

## Side effects

### Direct accesses and state changes

The function itself:

- Reads the context resource slot before branch-specific work.
- Initializes the mutable request word before every chunk call.
- Decreases its local remaining amount before that call.
- Uses both query words after a successful query.
- Does not explicitly write the context slot.
- Does not inspect scratch contents after a dependency call.

The scratch storage is not initialized by this function before use.

### Local storage layout

The represented local layout is:

| Region | Size and placement |
|---|---|
| Mutable request | Four bytes immediately before scratch storage. |
| Declared scratch storage | 1,016 bytes. |
| First query word | Four bytes immediately after scratch storage. |
| Second query word | Four bytes immediately after the first query word. |

The maximum request is eight units larger than the scratch region’s declared byte size.

If requests are byte counts and F-74745fd09174 writes the full requested count consecutively from the scratch address, a 1,024-byte transfer reaches the two adjacent query words. This is a conditional layout consequence, not proof of an executed memory fault.

The actual transfer semantics and authoritative allocation boundaries remain unknown. Neither a guaranteed 1,024-byte buffer nor a guaranteed overflow may be inferred solely from this representation.

### Ownership and lifetime

No explicit allocation, release, or resource close appears in this function.

The resource is forwarded by value. Pointers passed to dependencies refer to invocation-local storage and have no lifetime supplied by this function after return.

There is no established guarantee that dependencies preserve globals, retain resource validity, avoid resource release, avoid retaining pointers, or restrict writes to apparent output regions. Those requirements need separate dependency contracts.

Calls are issued serially. No locking or asynchronous-completion handling is shown.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-cdf60417d657 | One negative-branch call with the resource value and first query-word address. Nonzero status is returned; zero enables arithmetic. | Output extent, initialization of both words, data meaning, ownership and global effects. |
| F-74745fd09174 | Nonnegative-branch calls with the resource value, mutable request address, and scratch address. Initial requests range from 1 to 1,024. | Units, transfer direction, writable extent, request mutation, short-operation semantics, ownership and statuses. |
| F-f7e2f4cd400d | Called after a successful query and defined argument evaluation, with the four arguments specified above. Its status is returned. | Argument meanings, resource mutation, range enforcement, ownership and statuses. |

## Acceptance examples

These are conditional static expectations, not executed tests. Assume valid memory, defined address formation, the stated branch interpretation, and normally returning dependencies. Negative arithmetic examples additionally state their shift and overflow conditions.

### Zero displacement

The resource slot is read, no dependency is called, and zero is returned. An unreadable resource slot is not covered by this success expectation.

### Chunk sequence and retained resource

For displacement 2,050 and successful chunk calls, initial request values are 1,024, 1,024, and 2. Every call receives the resource captured at entry, even if an earlier dependency changes the context slot. The result is zero.

If the second call instead returns nonzero status \(E\), only two calls occur, the two-unit request is omitted, and \(E\) is returned without rollback.

### Request changed by dependency

For displacement 1,024, if the dependency changes the request word to 7 and returns zero, the function returns zero after one call. It does not issue another request for 1,017 units. This does not assert that such dependency behavior satisfies its own contract.

### Query failure

For a negative displacement and query status \(E\ne0\), the function returns \(E\) without calling the final dependency. No arithmetic-shift assumption is needed for this path because the shift is not evaluated.

### Negative one, representable intermediate

Assume arithmetic signed right shift, displacement −1, successful query, \(L=5\), and \(H=2\).

The unsigned low addition carries; the adjusted low word is 4. The signed intermediate is 1, and adding the carry produces adjusted high word 2. Final arguments are the retained resource, 2, 4, and 2.

With \(L=0\) and the same \(H\), there is no carry. The final arguments are the retained resource, 2, `0xffffffff`, and 1.

### Two-word wrap without signed intermediate overflow

Assume arithmetic signed right shift, displacement −1, and successful query words \(L=0\), \(H=0\).

The signed intermediate is −1 and is representable. Both adjusted words are `0xffffffff`; the original second argument remains zero. The function calls the final dependency without its own range rejection.

### Minimum signed high word

Assume arithmetic signed right shift, displacement −1, and \(H=-2,147,483,648\).

With \(L=5\), the low addition carries and produces 4. The mathematical adjusted high result would equal the original high word, but the earlier signed subtraction still overflows.

Without established target wrapping, no final-call expectation is specified. With established wrapping, the adjusted high word is `0x80000000`.

With \(L=0\), there is no carry. Under the same wrapping condition, the adjusted low word is `0xffffffff` and the adjusted high word is `0x7fffffff`.

## Uncertainties

The following remain **documented_only**:

- Target signed conversion, negative right-shift semantics, and signed-overflow execution behavior.
- Resource type and request units.
- Query output meaning and whether success initializes both words.
- Meaning of the final dependency’s original-high-word argument.
- Authoritative scratch allocation boundaries and actual transfer extent.
- Dependency effects on globals, ownership, resource validity, and pointer retention.
- Status meanings beyond zero versus nonzero.
- Byte order and additional alignment requirements.
- Behavior after invalid accesses, insufficient output initialization, or dependency contract violations.

The forward-consumption/backward-repositioning interpretation remains inferred intent. It does not resolve these uncertainties.

## Evidence

This contract derives from static inspection of the supplied function representation: its fixed-offset resource read, signed branch test, capped request loop, pre-call subtraction, status checks, adjacent query-word layout, unsigned low addition, carry calculation, signed shift and intermediate addition, and final four-argument call.

The arithmetic distinction is material: the signed high-word intermediate precedes unsigned carry addition. Neither the later unsigned operation nor a mathematically in-range final result establishes defined behavior for an overflowing earlier intermediate.

No dependency implementations, authoritative target arithmetic specification, or runtime traces were supplied. No runtime tests have been performed. Acceptance examples are conditional arithmetic and control-flow consequences, not measured outcomes.
