<!-- Trusted provenance: F-cdbbe059be50; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cdbbe059be50

## Purpose

F-cdbbe059be50 conditionally records a small nonzero request value in a byte-sized state field and returns a Boolean-valued, 32-bit result.

An affirmative result requires both:

- The request is one of 1, 2, or 3.
- A separately located byte, addressed relative to the supplied base, equals 2 when examined.

Once those conditions hold, the function returns 1 whether it writes the request into the state field or finds that field already nonzero. The result therefore indicates that the qualification conditions were satisfied, not necessarily that the stored state changed.

Two dependency calls bracket the function’s own conditional memory accesses. Their pairing is established; their purpose is not. Interpreting them as synchronization acquisition and release is inferred intent, not an established contract.

## Interface

### Inputs

**Base**

A 32-bit integer-valued argument used as the base of several addresses. It is not treated merely as a numeric identifier: the function derives a dependency argument and accesses byte-sized storage relative to it.

The required locations are:

| Location | Address relationship | Use |
|---|---|---|
| Dependency-associated location | Base plus 0x230 | Passed to both dependencies |
| Qualification byte | Base plus an externally supplied displacement | Read for requests 1–3 |
| State byte | Base plus 0x244 | Read after successful qualification; written only if zero |

The externally supplied displacement comes from a global value whose contents and initialization are not provided. The evidence does not establish the concrete qualification-byte offset.

**Request**

An unsigned 32-bit value.

- Zero does not qualify.
- Values 1, 2, and 3 require examination of the qualification byte.
- Values 4 through 4,294,967,295 do not qualify.

Unsigned interpretation matters. A value with its most significant bit set belongs to the last range; it is not treated as a negative value that falls below 4.

### Output

The function returns a 32-bit value restricted to:

- **1:** The request is 1, 2, or 3, and the qualification byte equals 2.
- **0:** The request is zero, at least 4, or is 1–3 with a qualification byte other than 2.

The return values of the two dependencies, if any, do not determine this result.

### Memory and execution preconditions

Normal defined operation requires:

- Base-plus-offset address calculations must produce the intended valid addresses under the execution environment.
- The dependency-associated location must satisfy both dependencies’ requirements.
- For requests 1–3, the global displacement must be available and the resulting qualification-byte address must be readable.
- If qualification succeeds, the state byte must be readable.
- If the qualifying state byte is zero, that byte must be writable.
- Both dependency calls must complete normally for an ordinary return to occur.

The visible function does not validate these requirements. It does not establish that its base designates an allocated object, that the object is large enough, or that the displacement remains within that object.

## Behavior

### Dependency bracketing

The function first forms the address at base plus 0x230 and passes it to F-338d16f9ce72.

This call occurs for every request value, including zero and values above 3. No qualification-byte or state-byte access performed by this function precedes it.

After that call returns normally, the function evaluates the request and, where applicable, accesses memory. Before each normal return it calls F-1333331b2da1 with the same previously formed address.

There is exactly one call to each dependency on every normally completing invocation. No early normal-return path bypasses the second dependency.

These statements describe calls made directly by F-cdbbe059be50. They do not restrict additional calls or memory accesses performed inside either dependency.

### Request qualification

For request zero, the function performs no qualification-byte or state-byte access of its own. It calls the second dependency and returns 0.

For requests 1, 2, and 3, the function obtains the global displacement and reads the byte at the resulting base-relative address. That byte must equal the numeric value 2 for qualification to succeed.

In particular, request 3 has no unconditional success privilege. It requires the same qualification-byte check as requests 1 and 2.

For requests at least 4, the function performs no qualification-byte or state-byte access of its own. It calls the second dependency and returns 0.

When a request in 1–3 encounters a qualification byte other than 2, the function does not inspect or update the state byte. It calls the second dependency and returns 0.

Whether plain character values are signed or unsigned does not alter this comparison: the byte representation for 2 satisfies it, and every other byte representation does not.

### Conditional state update

After a request qualifies, the function reads the byte at base plus 0x244.

- If that byte is zero, the function stores the request value there.
- If that byte is nonzero, the function leaves it unchanged.

The store is exactly one byte. Every request that can reach it is in the range 1–3, so conversion from the 32-bit request to the stored byte loses no information.

The function does not replace an existing nonzero value, even when that value differs from the request or lies outside 1–3. Such a value does not turn the result into failure.

After either state-field outcome, the function calls F-1333331b2da1 and returns 1.

### Observable order

On a qualifying invocation that finds a zero state byte, the function’s directly visible actions occur in this order:

1. Call F-338d16f9ce72 with base plus 0x230.
2. Obtain the global displacement and read the qualification byte.
3. Read the state byte at base plus 0x244.
4. Write the request into that state byte.
5. Call F-1333331b2da1 with the previously formed dependency address.
6. Return 1.

On a qualifying invocation that finds a nonzero state byte, step 4 is absent.

On an invocation with a request in 1–3 that fails qualification, the state-byte read and write are both absent, and the return is 0.

For zero and requests at least 4, the function’s qualification and state accesses are all absent, and the return is 0.

These are function-local ordering guarantees. They are not a claim of atomicity, cross-thread visibility, or hardware memory-barrier semantics.

## Side effects

### Direct effects

The only explicit data-memory write visible in this function is the conditional one-byte store at base plus 0x244.

The function visibly performs no allocation, deallocation, bulk copy, or explicit ownership transfer. Passing the base-derived address to dependencies does not establish whether those dependencies retain it, mutate associated storage, or release resources.

The global displacement is read when needed; the function contains no explicit write to it.

### Dependency-mediated effects

Both dependencies may have effects that are not described by the available evidence. In particular, the contract cannot assume that they:

- Preserve the qualification byte or state byte.
- Preserve the global displacement.
- Preserve the lifetime or ownership of the addressed storage.
- Act only on the address supplied to them.
- Acquire or release any particular synchronization primitive.

Consequently, the function’s memory decisions concern values observed after F-338d16f9ce72 returns, not necessarily values present at function entry.

Similarly, a direct store by this function does not by itself guarantee the same byte value is present when the function returns: F-1333331b2da1 runs afterward and may have undocumented effects.

The second dependency does not change the function’s selected return value through any explicit result-handling mechanism. However, a dependency that does not return normally prevents the corresponding ordinary return.

### Aliasing and concurrency

The evidence does not establish that the qualification byte, state byte, global storage, and dependency-associated storage are disjoint.

If the qualification-byte address coincides with the state-byte address, qualification observes 2. A later read of that same byte ordinarily observes a nonzero value and causes no direct write, provided nothing changes it between the reads.

Without documented dependency and execution-environment guarantees, the function cannot be characterized as a thread-safe check-and-set operation.

## Dependencies

| Opaque function ID | Direct role established here | Contract not established |
|---|---|---|
| F-338d16f9ce72 | Called once before request-dependent data accesses, with base plus 0x230 | Synchronization, memory effects, ownership effects, failure behavior |
| F-1333331b2da1 | Called once before every normal return, with the same derived address | Synchronization, memory effects, ownership effects, failure behavior |

The paired calls suggest a protected region, but their actual semantics require independent dependency specifications.

The global displacement is an additional data dependency. Its value, width-specific interpretation, initialization, and mutability are not sufficiently described to assign a concrete layout offset.

## Acceptance examples

These examples are conditional on valid addresses, normally returning dependencies, and the stated values being observed at the relevant reads. Claims about final memory additionally require that the second dependency and concurrent actors do not change it afterward.

| Request | Observed qualification byte | Observed state byte | Direct state effect | Return |
|---|---:|---:|---|---:|
| 0 | Not read | Not read | No write | 0 |
| 1 | 2 | 0 | Store 1 | 1 |
| 2 | 2 | 0 | Store 2 | 1 |
| 3 | 2 | 0 | Store 3 | 1 |
| 1 | 2 | 3 | Preserve 3 | 1 |
| 3 | 2 | 255 | Preserve 255 | 1 |
| 3 | 0 | Not read | No write | 0 |
| 2 | 255 | Not read | No write | 0 |
| 4 | Not read | Not read | No write | 0 |
| 4,294,967,295 | Not read | Not read | No write | 0 |

Additional conditional cases:

- **Dependency changes qualification:** If F-338d16f9ce72 changes the relevant qualification byte from 0 to 2 before returning, a request of 1 qualifies based on the subsequently observed 2. The entry value is not decisive.
- **Dependency changes stored state afterward:** If the function stores 2 and F-1333331b2da1 subsequently changes that byte, return 1 remains consistent with the function’s decision. Persistent storage of 2 is not established.
- **Repeated qualifying requests:** With qualification held at 2 and dependencies preserving the state byte, a first request of 1 changes state from 0 to 1. A subsequent request of 3 returns 1 but preserves state 1.
- **State inaccessible on a nonqualifying path:** The function itself does not access the state byte for request 4. This does not guarantee safe execution, because both dependencies still run and have separate preconditions.

## Uncertainties

The following items are **documented_only**:

- The concrete qualification-byte displacement and enclosing object layout.
- The semantic meaning of qualification value 2 and stored request values 1–3.
- Whether the paired dependencies implement locking or another protocol.
- Dependency effects on globals, memory ownership, object lifetime, and stored state.
- Concurrency guarantees and whether either byte access has special memory semantics.
- Behavior after an invalid address, arithmetic overflow, dependency failure, or nonlocal exit.
- Exact machine-level address wrapping and overflow behavior.

The represented interface uses 32-bit integer quantities, and the explicit memory accesses are byte-sized. That does not justify promising portable wrapping behavior for base-plus-offset calculations. Valid address derivation is a precondition; no overflow handling is visible.

The write-on-zero rule suggests retention of an initial qualifying request. That is inferred intent, not a stronger persistence or exclusivity guarantee.

## Evidence

This contract is based solely on static inspection of the supplied function representation and opaque dependency mapping.

Established static evidence includes the request ranges, byte comparisons, state-field offset, conditional single-byte store, dependency-call ordering, and two possible 32-bit return values.

No dependency implementations, complete object layout, global initialization, calling-environment guarantees, or runtime observations were supplied. No runtime tests have been performed.
