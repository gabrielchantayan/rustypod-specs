<!-- Trusted provenance: F-0c3bbc292f4e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0c3bbc292f4e

## Purpose

F-0c3bbc292f4e performs a fixed sequence of computed-address writes and indirect calls, then enters a repeating call-and-clear cycle with no represented exit.

No application-level purpose is established. The visible behavior does not identify an allocator, serializer, device protocol, or object initializer. Its contract is therefore expressed in terms of memory accesses, dependency calls, arithmetic, and retained state.

This is a **conditional static contract**. The supplied representation reports invalid instruction data and removal of unreachable regions. Those warnings leave the correctness and completeness of the represented execution path unresolved. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---|---|
| First argument | 32 bits; interpretation unspecified | No explicit use in the represented body. |
| Second argument | Signed 32-bit integer | Determines the first stored word and first destination address. |
| Third argument | Unsigned 32-bit integer | Supplies logically shifted values used in stores, call arguments, and one direct byte-write address. |

The absence of an explicit use of the first argument does not establish that dependencies cannot observe its incoming machine state. Dependency signatures and the calling convention are unresolved.

### Implicit inputs and preconditions

Execution also depends on incoming return-link state, indirect-call results, and bytes read from computed addresses. Incoming return-link state contributes to the first call target.

There is no explicit buffer, capacity, object descriptor, ownership token, or error-output parameter. Successful traversal of any part of the represented path requires:

- Each reached indirect target to be callable with the presented machine state.
- Every earlier dependency call to return.
- Every reached read and write address to permit the represented access.
- Sufficient preservation of live values for the represented dataflow to hold.

These are conditions for the described path, not locally enforced checks. Dependency preservation rules are **documented_only** unknowns.

### Output and termination

The represented interface has no return value. No normal-return path appears: the terminal loop is unconditional.

A dependency may fail, trap, terminate execution, transfer control elsewhere, or never return. A memory access may fault. The representation contains no local handling for those outcomes.

### Arithmetic and layout conventions

The following symbols describe contract-level values, not source identifiers:

- **P:** the second argument’s 32-bit bit pattern.
- **N:** the unsigned third argument.
- **A:** the low 32 bits of P × 2²⁴.
- **H:** N logically shifted right by 8.

Unless stated otherwise, scaled values and computed addresses below use a **conditional 32-bit machine-word interpretation**, retaining the low 32 bits. This convention captures the represented bit-level dataflow; it does not claim that overflowing signed arithmetic or signed left shifts have defined portable-language semantics.

A word access is 32 bits; a byte access is 8 bits. Byte reads used as address components are unsigned. Word byte order, actual pointer width, address mapping, and executable arithmetic rules are not independently established.

No aggregate layout is established. An offset of 20 in a word-store address does not by itself identify an object field or header.

## Behavior

### Ordering and dependency conventions

The order below concerns this operation’s explicit accesses and call boundaries. It does not constrain memory activity inside dependencies.

Every later event is conditional on all preceding events completing. Repeated reads of the same address are fresh reads. They must not be replaced by a previously observed byte: dependencies, overlapping stores, or external activity may change memory.

A **five-call group** means an initial call with the stated arguments followed by four calls with no explicit arguments shown. Its named result comes from the fifth call. Calls within a group are not asserted to share a target. “No explicit arguments shown” does not mean that incoming registers or other machine state are irrelevant.

The result labels below denote 32-bit values. Results used only for forwarding retain their full represented 32-bit contents; results used in scaling are interpreted through the arithmetic convention above.

### Initial word-write progression

Before the first dependency call, the operation writes A at address P × 2²⁶ + 20.

A five-call group begins with A as its single explicit argument and produces **R1**. Its first target depends on incoming return-link state.

The operation then writes A at address 4 × R1 + 20. A second five-call group begins with 4 × R1 and produces **R2**.

Next, the operation writes 4 × R2 at address 16 × R2 + 20. The stored value is retained as **B**.

These writes have no preceding bounds check, null check, or established object-pointer validation.

### Forwarded-result groups and further word writes

A nine-call group follows. Its externally relevant argument and result relationships are:

- The first call receives B.
- The second call’s result is passed unchanged to the third call.
- The fourth call’s result is passed unchanged to the fifth call.
- Four more calls follow, with no explicit arguments shown.

The ninth call supplies **R3**. The operation writes B at address 4 × R3 + 20.

A five-call group beginning with 4 × R3 produces **R4**. The operation writes 4 × R4 at address 16 × R4 + 20 and retains that stored value as **C**.

A second nine-call group has the same forwarding relationships, but its first call receives C. Its final result is **R5**. The operation writes H at address 4 × R5 + 20.

Similarity between the two groups does not establish common target identities or equivalent dependency behavior.

### Retained-address overwrite

A five-call group beginning with 4 × R5 produces **R6**.

The operation retains **D**, the low 32 bits of 16 × R6, and writes 4 × R6 at D + 20. It also retains:

**E = (C AND 0x7fffffff), logically shifted right by 7.**

Under the stated interpretation, E lies between 0 and 0x00ffffff.

Another five-call group begins with three arguments: D, D, and N logically shifted right by 7. Its final result is **R7**.

After that group returns, the operation writes D at D + 20. This is a second explicit write to the retained address used immediately before the group. Neither its destination nor its stored value is derived from R7. Intervening dependencies may nevertheless have modified that location.

A further five-call group begins with R7 shifted left by two bits and supplies an unsigned result **U**.

### Byte-derived destinations

The next store depends on an unsigned byte read at E + 1. If the read yields **b**, the destination is b + 17. The stored byte is the low eight bits of U shifted left by two.

The read supplies an integer address component, not a word-sized pointer. Consequently, every destination obtained this way lies between addresses 17 and 272 inclusive.

A seven-call group follows:

- The first call receives three equal arguments, each U shifted left by two.
- The second call’s result is forwarded unchanged to the third call.
- Four calls with no explicit arguments shown follow.

The seventh call supplies **R8**.

The operation then reads E + 1 again and writes at that newly read byte plus 17. This time the stored byte is the low eight bits of:

**(U AND 0x3fffffff), logically shifted right by 20.**

The first byte value depends on U’s low six bits, positioned in output bits 2 through 7. The second depends on U’s bits 20 through 27. The two destinations need not be equal.

A five-call group beginning with R8 shifted left by two produces an unsigned result **V**.

### Direct byte store and clears

The operation retains **J**, the low 32 bits of V shifted left by two.

It writes the low byte of J directly at:

**(H logically shifted right by 7) + 28.**

Because H is N logically shifted right by 8, this address is equivalently:

**(N logically shifted right by 15) + 28.**

The address ranges from 28 through 0x0002001b.

Immediately after that direct write, the operation reads E + 1 again and writes zero at the returned byte plus 17. There is no explicit dependency call between the direct write and this read. Thus, if the direct destination aliases E + 1, the read observes memory after the direct write.

A seven-call group then occurs:

- Its first call receives H logically shifted right by 7.
- Its second call has no explicit arguments shown.
- Its third call receives (V AND 0x3fffffff) logically shifted right by 6.
- Four further calls have no explicit arguments shown.

The final result is **R9**.

The operation again reads E + 1 and clears the address obtained by adding 17 to that fresh byte. This is the fourth explicit read at E + 1.

A five-call group beginning with R9 shifted left by two supplies an unsigned result **W**.

### Entry to the repeating phase

The operation forms the initial state value:

**Q = (W AND 0x3fffffff), logically shifted right by 25.**

Initially Q is between 0 and 31.

The byte-read source changes to a different retained address:

**K = the low 32 bits of (1024 × J + 1).**

The operation reads an unsigned byte at K and writes zero at that byte plus 17. K is derived from J, not from E, W, or the current byte contents.

A call receiving Q shifted left by one is followed by a call with no explicit arguments shown. The latter supplies the initial loop-carried value **L**.

### Repeating phase

Each iteration has the following observable structure:

1. A five-call group begins with two explicit arguments: L and Q logically shifted right by 8. Its final result is **T**.
2. The operation freshly reads one unsigned byte at K and writes zero at that byte plus 17.
3. A five-call group begins with T shifted left by two and produces **S**.
4. For the next iteration, L becomes the low 32 bits of S shifted left by two, and Q becomes the low 32 bits of S shifted left by 26.

K remains retained across iterations; its byte contents are not cached.

The first iteration’s Q-derived argument is necessarily zero because initial Q is at most 31. In later iterations, that argument equals the preceding S’s low six bits shifted left by 18.

The first call’s target on initial loop entry is constructed differently from the target used after the backedge. The repeated argument pattern therefore does not establish an identical target sequence from the first iteration onward.

No represented condition exits the loop.

## Side effects

### Explicit memory activity

Before loop entry, the represented path performs **eight 32-bit writes and eleven byte accesses**.

The eleven byte accesses comprise:

- Four reads at E + 1.
- One read at K.
- Five byte-indirected writes corresponding to those five reads.
- One separate direct byte write.

Thus the totals are **five byte reads plus six byte writes**, not eight byte accesses.

The six byte writes occur in this order:

| Position | Destination | Stored byte |
|---|---|---|
| 1 | Fresh byte at E + 1, plus 17 | Low byte of U shifted left by two |
| 2 | Fresh byte at E + 1, plus 17 | Low byte of (U AND 0x3fffffff) shifted right by 20 |
| 3 | (N shifted right by 15) + 28 | Low byte of J |
| 4 | Fresh byte at E + 1, plus 17 | Zero |
| 5 | Fresh byte at E + 1, plus 17 | Zero |
| 6 | Fresh byte at K, plus 17 | Zero |

Each completed loop iteration adds one explicit byte read and one explicit byte write. All counts exclude dependency activity.

### Aliasing and visibility

No distinctness between computed addresses is established. Word writes may overlap each other or byte-access locations. A byte-derived destination may equal its own read source. Direct writes may alter later read sources. Dependency calls may change any subsequently accessed memory.

Under the conditional 32-bit address interpretation, the computed word-store addresses are multiples of four. This does not establish accessibility, sufficient alignment for the actual machine, or valid object membership.

No volatile, atomic, synchronization, or memory-barrier guarantee is established. The ordering described here is the order represented by the supplied static body, not an independently verified hardware or concurrency model.

### Ownership and errors

No direct allocation, release, ownership transfer, or resource cleanup is identified. Nothing establishes whether dependencies allocate, release, retain, or invalidate memory.

There is no local error-status interpretation, rollback, retry, or recovery path. Results used as values or address components are not visibly checked for failure indicators.

## Dependencies

Only F-0c3bbc292f4e has a supplied stable function identity.

**F-INDIRECT-UNKNOWN** is an opaque contract label for the unresolved dependency set. It does not assert that every call reaches the same function.

The represented target constructions include:

- A target dependent on incoming return-link state.
- Targets formed from fixed location-like values and offsets.
- A target formed using a code-location value.

The visible argument patterns contain zero, one, two, or three explicit words. Some call results are forwarded unchanged; others control subsequent writes or loop state.

The following dependency properties remain **documented_only**: identities, signatures, calling convention, return guarantees, side effects, register and memory preservation, ownership behavior, and error conventions. No dependency is assumed to preserve globals or to own or release any particular allocation.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Each requires the relevant calls to return, accesses to succeed, and the stated arithmetic interpretation to hold.

### Initial scaling and ordering

For P = 1, the first explicit write stores 0x01000000 at address 0x04000014, before the first indirect call.

For P = 0x100, both scaled expressions wrap so that the stored word is zero and its destination is address 20.

### Unsigned extraction

For N = 0x12345678, H is 0x00123456, and the direct byte-write address is 0x00002484.

For N = 0xffffffff, the direct address is 0x0002001b. Arithmetic right shifts would not satisfy these criteria.

### Retained-address overwrite

If R6 = 3, D is 48. The first write at address 68 stores 12. After the intervening five-call group returns, the second explicit write at address 68 stores 48, regardless of R7. Dependency writes between those events remain unconstrained.

### Fresh reads and truncation

If the first read at E + 1 yields 5, the first byte-derived destination is 22. If a dependency changes that source byte to 9 before the next read, the next destination is 26.

For U = 0xffffffff, those first two byte-store values are respectively 0xfc and 0xff. Equal values of U do not imply equal destinations.

### Pre-loop access accounting

For a path that reaches the loop, excluding all dependency memory activity, the operation must have performed four reads at E + 1 and one at K, with five corresponding byte-indirected stores and one additional direct store: eleven byte accesses in total.

### Loop state and nontermination

For W = 0xffffffff, initial Q is 31. The pre-loop call receiving Q shifted left by one receives 62, while the first iteration’s Q-derived argument is zero.

If an iteration produces S = 1, the next iteration receives L = 4 and a Q-derived second argument of 0x00040000.

If every dependency returns and every access succeeds indefinitely, execution continues indefinitely. A normal return after a fixed number of iterations does not match the represented path.

## Uncertainties

The following are **documented_only**, not resolved assumptions:

- Whether the represented control flow faithfully describes executable behavior.
- Whether the invalid instruction data or omitted regions affect reachability, arguments, or termination.
- Actual instruction-set, arithmetic, pointer-width, alignment, and byte-order rules.
- Whether computed addresses, especially the low byte-derived destinations, are mapped and writable.
- Dependency identities, live-state preservation, side effects, ownership, and failure behavior.
- Any implicit role of the otherwise unused first argument.
- Whether the unconditional loop is intentional.

The retained values and addresses describe represented dataflow. They do not independently prove the preservation guarantees required by the underlying calling convention.

Repeated call structures suggest structural regularity only. Their semantic purpose and commonality remain unknown.

## Evidence

Established static evidence consists of the supplied function representation: three explicit arguments, computed word stores, unsigned byte reads, byte stores, indirect-call argument and result relationships, masks and shifts, retained address expressions, and an unconditional terminal loop.

The pre-loop byte-access count follows directly from four byte reads using E + 1, one using K, five associated indirect byte stores, and one direct byte store.

The representation also reports invalid instruction data and removed unreachable regions. Those warnings limit confidence in the recovered path without establishing an alternative behavior.

No runtime tests, executable validation, dependency inspection, or independent memory-layout verification have been performed.
