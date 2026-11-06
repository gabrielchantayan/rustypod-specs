<!-- Trusted provenance: F-9ad23bf5b97e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9ad23bf5b97e

## Purpose

F-9ad23bf5b97e conditionally updates accounting associated with a supplied memory address, then delegates further handling of that address’s preceding metadata to another function.

The established behavior is narrower than a general memory-release contract. For a nonzero address whose metadata passes a minimal validity check, the function subtracts a masked quantity from several accounting fields, decrements two counters, and invokes additional dependencies. Whether those dependencies release memory, merge regions, change ownership, or provide synchronization is not established.

An initial dependency call occurs even when the supplied address is zero. Consequently, the zero-address case is not an unconditional no-op.

## Interface

### Inputs and result

| Input | Representation | Established use |
|---|---|---|
| First argument | 32-bit value | Not read or forwarded by the visible function body. |
| Second argument | 32-bit address-bearing value | Compared with zero; when nonzero, used to locate metadata immediately before the supplied address. |

The function has no specified return value. Ordinary completion is possible for both zero and nonzero second arguments. A failed metadata check transfers control to F-a817d8d58c10, which is represented as nonreturning.

No error code or success indicator is produced.

### Metadata layout

Let **P** denote the supplied nonzero address. The visible accesses use this preceding layout:

| Relative location | Width | Interpretation in this function |
|---|---:|---|
| P − 8 | 32 bits | Header checked for validity and masked to obtain an accounting quantity. |
| P − 4 | 16 bits, unsigned | First accounting selector. |
| P − 2 | 16 bits, unsigned | Second accounting selector. |
| P and above | Not accessed directly | No payload access is visible in this function. |

The header check requires a nonzero value with bit 0 clear. It does not require the masked accounting quantity to be nonzero.

The accounting quantity is obtained by retaining header bits selected by `0x3ffffffc`. Thus:

- Bits 0 and 1 are excluded.
- Bits 30 and 31 are excluded.
- The result is a multiple of four.
- Its range is zero through `0x3ffffffc`, inclusive.

Bit 1 is not rejected by validation. Neither are bits 30 or 31.

### Context layout

After an initial nonzero-address dependency call, the function reads a context address from a global location. The address captured by that read is used for the remaining visible context accesses and context arguments.

The following offsets are relative to that captured context address:

| Offset or indexed location | Width | Visible operation |
|---|---:|---|
| `0x224 + 4 × first selector` | 32 bits | Subtract the initially calculated accounting quantity. |
| `0x30c` | 32 bits | Subtract that same quantity. |
| `0x0e0 + 4 × second selector` | 32 bits | Subtract that same quantity. |
| `0x220` | 32 bits | Subtract that same quantity. |
| `0x310 + 4 × dependency result` | 32 bits | Decrement by one. |
| `0x390` | 32 bits | Decrement by one. |
| `0x008` | 32 bits | Subtract a newly read and masked header quantity. |

These are access formulas, not evidence of array sizes or nonoverlapping fields. The function performs no visible selector bounds checks.

## Behavior

### Entry and zero-address handling

The first observable action is a call to F-e70ee8234ed9 with the value 1.

Only after that call returns does the function test the supplied address. A zero address causes an immediate ordinary return. This path does not visibly read metadata, read the context global, update accounting fields, or invoke the remaining dependencies.

The effects of F-e70ee8234ed9 are still part of the zero-address path. Its name, argument, and position do not establish whether it initializes, enables, records, or synchronizes anything.

### Context selection and metadata validation

For a nonzero address, F-2fd8826557a4 is called without arguments. The context address is read from the global location after that call returns. Therefore, any change that dependency makes to the global can affect which context is selected.

The header at P − 8 is then checked:

- A zero header causes transfer to F-a817d8d58c10.
- Otherwise, a header with bit 0 set causes the same transfer.
- A nonzero header with bit 0 clear permits continuation.

The supplied representation contains a header read for the zero comparison and, if that comparison does not terminate the check, another read for testing bit 0. This is not an atomic snapshot contract. Concurrently changing metadata is not shown to be supported.

No accounting field is visibly updated before validation succeeds. The earlier dependency calls may nevertheless have side effects.

The failure dependency is represented as nonreturning. No normal return, local rollback, or later dependency call is established for that path.

### Accounting updates

After validation succeeds, F-c0fc6d033caa is called with the captured context address. Only after it returns are the selectors and the accounting quantity used for the updates read.

This timing matters: the function does not preserve a pre-call copy of either selector or of the header quantity. Any modifications made by F-c0fc6d033caa can affect subsequent accounting. There is no visible repeat of the validity check after that dependency returns.

The visible accounting proceeds in this order:

1. The first unsigned 16-bit selector is read from P − 4.
2. The header is read and masked with `0x3ffffffc`, producing the quantity used by the next four subtractions.
3. The selected field at context offset `0x224 + 4 × first selector` is read, reduced by that quantity, and written.
4. The field at context offset `0x30c` is read, reduced by the same quantity, and written.
5. The second unsigned 16-bit selector is read from P − 2.
6. The selected field at context offset `0x0e0 + 4 × second selector` is read, reduced by the same quantity, and written.
7. The field at context offset `0x220` is read, reduced by the same quantity, and written.

The second selector is therefore not necessarily an entry-time value: its read follows two context writes. Aliasing between the metadata and context storage could make this ordering significant. The function supplies no visible alias checks.

Each selector ranges from 0 through 65,535. Its four-byte displacement ranges from zero through `0x3fffc`; that multiplication itself fits within 32 bits. Whether the resulting address lies within valid context storage is a caller or environment prerequisite, not a checked condition.

### Count updates and final quantity subtraction

After the four initial subtractions, F-09dcb55e8e0a is called without arguments. Its returned integer selects the 32-bit field at context offset `0x310 + 4 × result`.

That selected field is decremented first. The field at context offset `0x390` is then decremented.

The function subsequently reads the header at P − 8 again, masks it with `0x3ffffffc`, and subtracts that newly obtained quantity from the field at context offset `0x008`.

The last subtraction need not use the quantity used by the earlier four subtractions. In particular, F-09dcb55e8e0a executes between those operations, and its effects on metadata are unknown. There is no visible header validation after this dependency call.

No range or sign check is visible for the dependency’s returned selector. A negative or excessively large result is not converted into a defined validation error by this function.

### Final delegation

After all visible accounting writes, F-cc6c9b4a0f68 is called with:

- The captured context address.
- The metadata address P − 8, not P.

If that call returns, F-e586bff64fb7 is called with the same captured context address. Ordinary return follows if that final dependency returns.

The function does not reread the context global to choose arguments for these calls. A dependency may change that global, but such a change does not replace the already captured address in the visible continuation.

## Side effects

### Memory and arithmetic

The successful nonzero path performs seven visible 32-bit read-modify-write operations: five quantity subtractions and two decrements.

There are no visible guards against:

- Subtracting a quantity larger than the stored value.
- Decrementing a zero count.
- Selecting overlapping accounting locations.
- Address arithmetic extending beyond valid storage.

The stored results have 32-bit width. As a 32-bit bit-pattern accounting model, subtraction and decrement retain the low 32 bits; for example, decrementing zero produces `0xffffffff`. The supplied representation does not establish a portable signed-overflow contract for other execution models.

Updates are sequential, not a simultaneous transformation of seven independent entry values. If two formulas resolve to the same storage, the later operation reads the value left by the earlier operation. For example, if the dependency selector is 32, its selected count field coincides with offset `0x390`, so that word is decremented twice.

No rollback mechanism is visible. If execution fails after some writes, this body does not restore earlier values.

### Ownership and synchronization

The function directly passes the metadata address to F-cc6c9b4a0f68. It does not visibly access the payload, clear the header, or independently perform a recognizable ownership transfer.

The following are not established:

- That P must originate from a particular allocator.
- That ordinary return makes P unusable.
- That metadata or payload memory is released.
- That dependency calls acquire or release a lock.
- That any dependency preserves globals or context contents.
- That the operation is atomic or safe under concurrent access.

For defined ordinary completion, all addresses actually accessed must be valid for their respective widths and access types, and dependencies must satisfy whatever undisclosed preconditions they impose. Alignment and byte order are not specified by the supplied material.

## Dependencies

| Opaque ID | Call conditions and arguments | Established continuation |
|---|---|---|
| F-e70ee8234ed9 | Always called first, with 1. | Address test occurs after return. |
| F-2fd8826557a4 | Called only for a nonzero address; no arguments. | Context global is read after return. |
| F-a817d8d58c10 | Called when the checked header is zero or has bit 0 set; no arguments. | Represented as nonreturning. |
| F-c0fc6d033caa | Called after successful validation, with the captured context. | Selector and quantity reads follow its return. |
| F-09dcb55e8e0a | Called after the first four accounting writes; no arguments. | Return value selects a count field. |
| F-cc6c9b4a0f68 | Called after all visible accounting writes, with the captured context and P − 8. | Final dependency follows only if it returns. |
| F-e586bff64fb7 | Called after F-cc6c9b4a0f68 returns, with the captured context. | Ordinary return follows if it returns. |

Except for the stated nonreturning failure path, dependency implementations and their side effects are unknown.

## Acceptance examples

These are conditional contract examples, not runtime test results.

### Zero address

Given a zero second argument, if F-e70ee8234ed9 returns:

- It is called once with 1.
- The function then returns.
- No other listed dependency or direct metadata/context access occurs.

Any side effect of that initial dependency remains possible.

### Invalid header

Given a nonzero address and stable header value `0x00000000`, the zero check invokes F-a817d8d58c10.

Given a stable header value `0x00000005`, the low-bit check invokes the same dependency.

In both cases, the initial two dependencies and context capture have already occurred. No visible accounting update precedes the failure call.

### Ordinary accounting with stable metadata

Assume:

- Valid, accessible, nonoverlapping selected fields.
- A stable header of `0x00000022`.
- Dependencies do not alter the relevant metadata or accounting values.
- F-09dcb55e8e0a returns an in-range selector.

The quantity is 32: bit 1 does not contribute. Each of the five quantity-accounting fields decreases by 32, and each of the two distinct count fields decreases by one. The final two dependencies receive the captured context; F-cc6c9b4a0f68 additionally receives P − 8.

### Valid header with zero accounting quantity

A stable header of `0x40000000` is nonzero and has bit 0 clear, so it passes validation. Its masked quantity is zero.

With otherwise valid storage and noninterfering dependencies, all five quantity subtractions retain their prior bit patterns, while both count decrements still occur. Passing validation therefore does not imply a positive quantity.

### Header change between accounting phases

Assume the quantity used by the first four subtractions is 64. If F-09dcb55e8e0a changes the accessible header so that its masked quantity becomes 16, and nothing else interferes, the offset-`0x008` field is reduced by 16 rather than 64.

A contract requiring all five subtractions to use one immutable quantity would not match the visible behavior.

### Overlapping count selection

If F-09dcb55e8e0a returns 32, both count operations target offset `0x390`. With no interfering effects, a starting value of 7 becomes 5.

This example demonstrates access ordering; it does not establish that 32 is an intended selector.

## Uncertainties

### Inferred intent

The metadata layout, masked quantity, accounting reductions, and final delegation suggest a resource-removal or memory-release accounting operation. That interpretation is not sufficient to assign ownership or allocator semantics to any dependency.

Likewise, the two selectors appear to classify the accounted quantity, while the dependency result appears to classify a count. Their domains and meanings remain unknown.

### documented_only

The following require documentation or implementation evidence beyond the supplied body:

- Context-global initialization and lifetime.
- Valid selector ranges and context allocation size.
- Header flag meanings and quantity units.
- Dependency behavior, including synchronization and memory ownership.
- Whether metadata changes across dependency calls are permitted.
- Concurrency, reentrancy, and atomicity guarantees.
- Alignment, byte order, and address-space constraints.
- Failure behavior for inaccessible storage or invalid calculated addresses.
- Effects on external state, including logging or process termination.

These unknowns must not be replaced with assumptions that dependencies preserve state, release storage, or complete successfully.

## Evidence

The established observations above come solely from the supplied static representation: access widths and offsets, constants, conditional branches, call arguments, data dependencies, and ordering of visible reads and writes.

Comments and apparent operation names are not treated as independent evidence of behavior. No dependency implementation, external API contract, or runtime observation was supplied.

No runtime tests have been performed. Acceptance examples describe conditional consequences of the visible operations; they do not claim observed execution.
