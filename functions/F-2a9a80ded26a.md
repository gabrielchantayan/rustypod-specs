<!-- Trusted provenance: F-2a9a80ded26a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2a9a80ded26a

## Purpose

F-2a9a80ded26a performs mode-dependent sequences of dependency calls using caller-supplied values, shared cached fields, and temporary descriptor-like storage.

Established behavior consists of reads, arithmetic, explicit writes, and ordered calls. Interpreting the calls as allocation, template copying, or object initialization is **inferred intent**, not an unconditional dependency contract. In particular, a count-bearing call does not establish that bytes are copied, and a size-like argument does not establish that storage is allocated.

The function has no explicit return value. Its observable effects include shared-state writes and whatever effects its dependencies produce. Whether these effects construct, register, display, retain, or release objects is unknown.

## Interface

### Arguments and result

The supplied representation has four 32-bit arguments:

| Argument | Established use |
|---|---|
| Context | Address-like value used for byte reads at `+4` and signed 32-bit reads at `+8`; also passed to F-ed4b1fd8953e. |
| Mode | Signed 32-bit value. Zero and nonzero select different call sequences. Exactly one additionally affects two selections. |
| Payload A | Opaque 32-bit value forwarded to the receiving dependencies listed below. |
| Payload B | Opaque 32-bit value forwarded alongside Payload A. |

Only bit 0 of the context byte at `+4` affects explicit branch and selection decisions. The context field at `+8` is interpreted as signed 32-bit arithmetic input.

There is no explicit result or error return.

### Shared storage

An address obtained from a global slot is captured at entry. Relative to that captured address:

| Offset | Width | Use |
|---|---:|---|
| `+0x10` | 32 bits | First cached arithmetic value; also associated with the initialization sentinel. |
| `+0x14` | 32 bits | Second cached arithmetic value. |
| `+0x18` | 32 bits | Cached positioning value. |

The sentinel expression separately reads the global address slot and tests the 32-bit field at `+0x10`. Subsequent cache accesses use the captured address. There is no intervening dependency call between capture and sentinel test, but concurrent mutation is not specified.

### Temporary layouts

For description, seven temporary regions are named primary, secondary, optional, zero-first, zero-second, nonzero-first, and nonzero-second. Each is passed as the destination-like argument of a call with count `0x68`.

Additional temporary regions are used for:

- A zero-mode call with count `0x5c`.
- A nonzero-mode extended descriptor with explicit fields through `+0x78`.
- Output supplied to F-59c1430f85c2, represented as two 32-bit words.

These names describe use within this function, not established application types.

### Preconditions and failures

Ordinary completion requires readable storage for all explicit dereferences, writable storage for explicit stores, and dependency results suitable for subsequent use. Required table entries are 32-bit entries at the selected offsets.

The function contains no explicit null check, bounds check, result validation, error branch, cleanup, or rollback. Requirements imposed internally by dependencies are unknown.

Signed 32-bit arithmetic is described below for inputs whose intermediate results are representable. Overflow behavior is not established.

## Behavior

### 1. Conditional cache population

If the sentinel is nonzero, this phase is skipped.

If it is zero, the following sequence occurs:

1. Call F-9d0ecc5f6adf and retain its 32-bit result.
2. Call F-ed4b1fd8953e with Context and a first selector read from global storage.
3. Capture a global 32-bit resolution argument.
4. Read a 32-bit record address from offset `+8` of the lookup result.
5. Call F-5710a8628cba with the retained result, a separate read of the global resolution argument, and the record’s first 32-bit word.
6. Call F-59c1430f85c2 with temporary output storage, the preceding result, the record’s 32-bit word at `+4` masked to its low 16 bits, and the byte at record offset `+12`.
7. Read the first output word, clear bit 0, and retain the resulting address.
8. Read signed 16-bit values at that address’s `+0x52` and `+0x54`, promote them to signed 32-bit, add them, and store the sum at captured shared address `+0x10`.
9. Perform another lookup using Context and a second globally supplied selector.
10. Read its record address at `+8`, then call F-5710a8628cba using the original retained result, the resolution argument captured in step 3, and the second record’s first word.
11. Invoke F-59c1430f85c2 with the same temporary output region and the corresponding second-record arguments.
12. Mask bit 0 from the new first output word, sum its two signed 16-bit fields at `+0x52` and `+0x54`, and store that sum at shared offset `+0x14`.
13. Calculate and store the positioning value at shared offset `+0x18`.

The resolution argument captured in step 3 and the separate read used in step 5 are not established to be immutable or identical under concurrent mutation. Likewise, selector values are globally supplied values, not proven constants.

For stable operands, define:

- \(W\): signed context field at `+8`;
- \(A\): shared field at `+0x10`, reread after the second decoding sequence;
- \(B\): the locally calculated second sum;
- \(h\): a fresh signed 16-bit read at `+0x52` of the retained first measurement address.

The positioning expression is:

\[
P=W-\operatorname{trunc}_0\!\left(\frac{W-(A+B)-2}{2}\right)-h
\]

Here division truncates toward zero. The expression contains two context-field reads; it is not unconditionally equivalent to using a single snapshot. Their relative evaluation order within the expression is not established.

The first measurement address is retained across the second dependency sequence, but its contents are not established to remain unchanged. The first shared sum is also reread rather than reused from its original calculation.

Each sum of two signed 16-bit fields lies between −65,536 and 65,534 inclusive and fits signed 32-bit storage. Later additions and subtractions may overflow.

### 2. Common temporary-region preparation

The function reads, in order, the captured shared fields at `+0x10` and `+0x14`, then the signed context field at `+8`. These three values are retained for the primary positioning calculation.

The count-bearing dependency is called **F-X1** in this document. This is a document-local opaque ID because the supplied dependency map does not assign one. Its visible arguments are destination-like address, source-like address, and count. Its actual writes and other effects are unknown.

Preparation proceeds in this order:

1. Call F-X1 for the primary region, a globally supplied source address, and count `0x68`.
2. Capture the global common-selection-table address.
3. Read context bit 0. Select index one only if that bit is set and Mode equals one; otherwise select zero.
4. Separately read the global common-selection-table address, read its 32-bit entry at index times four, and store the entry at primary `+0x48`.
5. Store at primary `+0x3c` the signed 32-bit expression

   \[
   A+\operatorname{trunc}_0\!\left(\frac{W-(A+B)-2}{2}\right)
   \]

   using the three values read before the primary F-X1 call.

6. Call F-X1 for the secondary region with its globally supplied source and count `0x68`.
7. Read context bit 0 again and calculate the same mode-qualified index.
8. Read the selected entry through the **captured** common-table address and store it at secondary `+0x48`.
9. Call F-X1 for the optional region with count `0x68`.
10. Call F-X1 for the zero-first region with count `0x68`.
11. Read the captured shared field at `+0x18`, retain it, and store its negation at zero-first `+0x3c`.
12. Call F-X1 for the zero-second region with count `0x68`.
13. Call F-X1 for the nonzero-first region with count `0x68`.
14. Store the negation of the same retained positioning value at nonzero-first `+0x3c`.
15. Call F-X1 for the nonzero-second region with count `0x68`.

Thus, the nonzero-first write occurs **before**, not after, the nonzero-second transfer-like call.

The table-address distinction is observable: changing the global common-table pointer during the secondary F-X1 call does not redirect the later secondary lookup. Changing entries at the captured address can still affect that lookup.

Only under an additional byte-copy contract for F-X1 do its calls establish copied template contents. Explicit stores are established at their stated points, but later unknown calls are not proven to preserve them.

### 3. Mode-zero sequence

When Mode is zero, the following request/receiver pairs occur in order:

| Order | Argument to F-8157a180d901 | Receiver | Temporary region |
|---|---:|---|---|
| 1 | `0x220` | F-7f6becd152e6 | Primary |
| 2 | `0xa4` | F-60d99bc349ed | Separate zero-mode region |
| 3 | `0x230` | F-60fe53f7ab39 | Zero-first |
| 4 | `0x230` | F-60fe53f7ab39 | Zero-second |

After the first receiver returns, but before the second F-8157a180d901 call, F-X1 is called for the separate zero-mode region using a globally supplied source and count `0x5c`.

Each receiver is passed these five arguments:

1. The immediately preceding F-8157a180d901 result.
2. Literal zero.
3. Payload A.
4. Payload B.
5. The indicated temporary-region address.

The function does not use a receiver’s return value.

The size-like arguments suggest storage requests, and the receivers suggest initialization routines. Neither interpretation is established by the supplied caller alone.

### 4. Nonzero-mode sequence

Every nonzero Mode selects this sequence.

Before its first F-8157a180d901 call, the function explicitly writes these 32-bit fields in the extended region:

| Offset | Value |
|---|---|
| `+0x68` | `0x1e` |
| `+0x6c` | Opaque value read from global storage |
| `+0x70` | `2` |
| `+0x78` | `0` |
| `+0x74` | `0x19` |

The table lists their actual assignment order, including `+0x78` before `+0x74`.

It then calls F-X1 with the extended region, the primary region, and count `0x68`. After that call, it:

- Writes `0x19` at extended `+0x30`.
- Reads context bit 0 and uses it directly as index zero or one into a separately supplied global table.
- Writes that table’s selected 32-bit entry at extended `+0x48`.

This selection does not depend on Mode equaling one.

Under an exact byte-copy contract, the first `0x68` bytes inherit the already-modified primary contents, the trailing fields lie outside the copied range, and the two later writes replace inherited fields. Without that contract, neither inheritance nor survival of the trailing assignments is established.

The call sequence is:

| Order | Argument to F-8157a180d901 | Receiver | Temporary region |
|---|---|---|---|
| 1 | Globally supplied 32-bit value; numeric value unavailable | F-4a73b9be0f3a | Extended |
| 2 | `0x230` | F-60fe53f7ab39 | Nonzero-first |
| 3 | `0x230` | F-60fe53f7ab39 | Nonzero-second |

Receiver arguments follow the same five-argument arrangement as in zero mode.

### 5. Common completion

After either mode-specific sequence:

1. Call F-8157a180d901 with `0x220`.
2. Pass its result, zero, both payloads, and the secondary-region address to F-7f6becd152e6.
3. Read context bit 0 again.

If that final bit read is set:

1. Read a 32-bit value at byte offset `+0x0c` from a globally supplied table address.
2. Store it at optional-region offset `+0x64`.
3. Call F-8157a180d901 with `0x230`.
4. Pass its result, zero, both payloads, and the optional-region address to F-60fe53f7ab39.

There is no explicit return value.

## Side effects

The explicit shared writes during cache population occur at `+0x10`, then `+0x14`, then `+0x18`. They are not transactional. An interruption or nonreturning dependency can leave partial state.

A first sum of zero leaves the sentinel zero if no other agent changes it. A later invocation then attempts cache population again. Conversely, a nonzero sentinel skips population without validating the other cached fields or their suitability for the current Context.

The function explicitly reads, but does not explicitly write, context storage. Dependencies may have additional effects through passed addresses, globals, or other reachable state. No preservation guarantee is supplied for context, shared storage, tables, source regions, or temporary regions.

Results from F-8157a180d901 are forwarded without explicit checking. There is no explicit release operation. Whether those results represent owned storage, handles, failures, or another resource is unknown. Ownership transfer and cleanup cannot be inferred.

Temporary addresses are supplied to dependencies. Their validity as local storage ends with this invocation. If a dependency retains such an address for later use, safety requires an additional lifetime arrangement not established here.

## Dependencies

| Opaque ID | Established caller-visible use |
|---|---|
| F-9d0ecc5f6adf | Called without arguments; its 32-bit result is reused in two later calls. |
| F-ed4b1fd8953e | Receives Context and a selector; its result is dereferenced at `+8`. |
| F-5710a8628cba | Receives three values; its result is forwarded to F-59c1430f85c2. |
| F-59c1430f85c2 | Receives writable temporary storage and three other arguments; the caller subsequently reads its first output word. |
| F-8157a180d901 | Receives one size-like value; its result becomes a receiver’s first argument. |
| F-7f6becd152e6 | Receives five arguments with primary or secondary temporary storage last. |
| F-60d99bc349ed | Receives five arguments with the separate zero-mode region last. |
| F-4a73b9be0f3a | Receives five arguments with the extended region last. |
| F-60fe53f7ab39 | Receives five arguments with branch-specific or optional storage last. |
| F-X1 | Document-local ID for the unmapped three-argument, count-bearing dependency; writes and other effects unknown. |

## Acceptance examples

These are conditional static criteria, not executed tests. Unless stated otherwise, dependencies return normally, required storage is valid, relevant state is stable, and signed arithmetic does not overflow.

### Mode and request order

With a nonzero sentinel and context bit 0 clear:

- Mode zero produces F-8157a180d901 arguments `0x220`, `0xa4`, `0x230`, `0x230`, `0x220`.
- Any nonzero Mode produces the globally supplied request value, `0x230`, `0x230`, `0x220`.
- The cache-population dependencies are not called.
- The optional receiver call is omitted.

If the final context-bit read is set, append a request with `0x230` and its optional receiver call.

### Selection distinctions

With stable context bit 0 set:

- Mode one selects common-table index one for primary and secondary.
- Mode zero, two, or any negative Mode selects common-table index zero.
- Any nonzero Mode selects index one from the separate extended-region table.

These criteria establish selected reads and explicit stores, not that unknown intervening dependencies preserve every stored value.

### Captured table address

Suppose the common-table global points to table T when captured after the primary F-X1 call. If the secondary F-X1 call changes that global pointer to U, the secondary selection still reads from T. Its entry value depends on T’s contents at the subsequent read.

### Positioning arithmetic

Suppose cache population observes first components 10 and 4, second components 6 and 2, and stable context value 100. With no relevant mutations:

- The cached sums are 14 and 8.
- The division term is 38.
- Cached positioning is \(100-38-10=52\).
- Primary `+0x3c` is explicitly assigned 52.
- Zero-first and nonzero-first `+0x3c` are explicitly assigned −52.

A numerator of −1 divided by two contributes zero, not −1.

### Conditional transfer interpretation

If F-X1 copies exactly its count of bytes without other writes, the extended region inherits the primary region’s first `0x68` bytes and retains the five trailing assignments. Without that additional contract, this content criterion is **documented_only**.

### Changing context

If earlier selection reads observe bit 0 clear but the final read observes it set, the optional receiver call occurs despite the earlier index-zero selections. The reverse transition omits that call.

## Uncertainties

The following remain **documented_only**:

- Concrete source-region contents, selector values, table entries, and the globally supplied nonzero-mode request value.
- F-X1’s memory effects, count interpretation, preservation guarantees, and return behavior.
- Allocation, initialization, registration, ownership, retention, release, and failure semantics of all dependencies.
- Application meanings of the cached values, descriptor-like regions, and payloads.
- Behavior following invalid addresses, unusable dependency results, or failed resource requests.
- Signed 32-bit overflow behavior, including negation of the minimum signed value.
- Operand-read ordering within arithmetic expressions where not independently sequenced.
- Synchronization, atomicity, reentrancy, and concurrent mutation semantics.
- Failure mechanisms not visible in this function.

No explicit recovery mechanism is established.

## Evidence

This contract derives from static examination of the supplied function representation and dependency mapping. The prior draft and feedback are not independent evidence.

Established claims concern argument widths, access offsets, signed promotions, masks, branch predicates, explicit assignments, call arguments, address captures, and sequencing. Copying, allocation, object initialization, and application purpose remain inferred interpretations unless expressly conditioned on an additional dependency contract.

No runtime tests have been performed.
