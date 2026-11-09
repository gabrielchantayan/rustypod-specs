<!-- Trusted provenance: F-0ed4a2dbfdc0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ed4a2dbfdc0

## Purpose

F-0ed4a2dbfdc0 processes at least one record from an opaque input source, populates fixed-size records in a destination table, and continues while a flag in the most recently populated record requests another record.

The operation is incremental, not transactional. Destination preparation, input-reader calls, and record writes can occur before an error is returned. The destination’s published record count and the context’s remaining-budget field are updated only after normal termination.

The apparent intent is to decode linked records containing an identifier, flags, two scalar values, and four transformation-like values. That interpretation is inferred. The established contract is the field access, dependency-call, arithmetic, and publication behavior described below; the underlying data format and reader semantics remain unspecified.

## Interface

### Context and reachable objects

The supplied argument is represented as a 32-bit address of a context object. The function accesses these context-relative fields:

| Offset | Access width | Role |
|---|---:|---|
| `0x0C` | 32 bits | Address of a destination-owning object |
| `0x18` | 32 bits | Opaque input-source value passed to readers |
| `0x1C` | 32 bits | Signed remaining budget; replaced only on normal completion |

The destination-owning object has these relevant fields:

| Offset | Access width | Role |
|---|---:|---|
| `0x4C` | 32 bits | Published record count |
| `0x50` | 32 bits | Address of the current record table |

Each table record occupies `0x20` bytes:

| Record offset | Width | Established use |
|---|---:|---|
| `0x00` | 32 bits | Identifier, stored after masking a reader result to its low 16 bits |
| `0x04` | 16-bit write; later 32-bit reads | Flags; the upper 16 bits are not explicitly replaced |
| `0x08` | 32 bits | First scalar value |
| `0x0C` | 32 bits | Second scalar value |
| `0x10` | 32 bits | First transformation component |
| `0x14` | 32 bits | Second transformation component |
| `0x18` | 32 bits | Third transformation component |
| `0x1C` | 32 bits | Fourth transformation component |

These are in-memory layouts, not a declaration of input byte order or serialized field widths.

### Preconditions

For ordinary completion, the context and reachable objects must support the accesses above. Each selected record must remain writable through its population, including across dependency calls.

The supplied evidence establishes no null checks, alignment checks, pointer validation, or protection against address-arithmetic overflow. An invalid address can therefore prevent a normal return rather than produce a documented status.

The input source must satisfy the requirements of the reader dependencies. Those requirements are unknown.

### Return value

The supplied representation constructs a 64-bit result:

- Low 32 bits: status.
- High 32 bits: an unresolved stack-derived value with no established initialization or meaning.

The low status is:

- Zero on normal completion.
- `0x15` when a local budget check fails.
- A nonzero status returned by F-d0804a542462 when destination preparation fails.

Consumers must not assume the complete 64-bit result is zero on success. The intended external return ABI, including whether consumers observe only the low word, is documented_only.

## Behavior

### Entry state and iteration

The function captures the context’s input-source value, destination-owning object address, and signed budget before the first iteration. It starts its local record index at zero; an existing published count does not determine the starting index.

The loop executes at least once. There is no empty-input success path based solely on an initial zero budget.

For each record, the following ordering is established.

### Destination preparation and initial budget check

First, F-d0804a542462 is called with:

1. The captured destination-owning object address.
2. The current index plus one.

A nonzero result terminates processing immediately and becomes the low return status. This failure takes precedence over any budget failure that would otherwise occur during that iteration.

After successful preparation, the function checks whether the signed budget minus four is negative. If so, it returns low status `0x15`.

This first subtraction is a check only. It does not replace the running budget with “budget minus four.”

Only after that check passes does the function read the table address from destination offset `0x50` and select the record at the current index using a 32-byte stride. Reloading the table address after preparation permits preparation to relocate the table. The evidence does not establish that relocation actually occurs.

### Initial record writes and header reads

Before reading the header, the function writes zero to:

1. Record offset `0x0C`.
2. Record offset `0x08`.

It then calls F-7efb5338f0fa with the captured input-source value. The low 16 bits of that result are written to record offset `0x04`. The adjacent upper 16 bits are left unchanged by this write.

A second call to F-7efb5338f0fa receives the same captured input-source value. Its result is masked to 16 bits and stored as a complete 32-bit value at record offset `0x00`.

The function then reads the complete 32-bit word at record offset `0x04` to select the payload branches and compute the next budget.

### Budget accounting

The running budget is reduced by a base amount selected by flag bit `0x01`:

| Flag condition | Base decrement |
|---|---:|
| `0x01` clear | 6 |
| `0x01` set | 8 |

An additional decrement is selected with this precedence:

| Condition | Additional decrement |
|---|---:|
| `0x08` set | 2 |
| Otherwise, `0x40` set | 4 |
| Otherwise, `0x80` set | 8 |
| None of those bits set | 0 |

Consequently, `0x08` overrides both `0x40` and `0x80`; when `0x08` is clear, `0x40` overrides `0x80`.

If the resulting signed budget is negative, processing returns low status `0x15`. The record’s two zero writes and both header-reader calls have already occurred. No explicit context-budget or published-count update is performed on this path.

The decrements are established accounting behavior. Their equivalence to bytes consumed is not established: the reader implementations and serialized widths are unavailable.

### Two scalar values

After the budget check passes, flag bit `0x01` from the previously read flag word selects the scalar-reader family.

When `0x01` is clear:

1. F-0b247502190b is called with the captured input-source value.
2. Its result is stored at record offset `0x08`.
3. F-0b247502190b is called again with that value.
4. Its result is stored at record offset `0x0C`.

When `0x01` is set:

1. F-7efb5338f0fa is called.
2. Its result is stored at record offset `0x08`.
3. F-7efb5338f0fa is called with the captured input-source value.
4. Its result is stored at record offset `0x0C`.

The supplied representation does not recover an explicit argument for the first reader call in the set-bit branch. It must not be silently specified as receiving the same argument as the other calls.

These scalar stores are 32 bits wide. Unlike the identifier store, they have no explicit 16-bit mask. The available evidence does not establish whether narrow reader results are sign-extended, zero-extended, or governed by another calling convention.

### Four transformation components

After the scalar stores, the function rereads the complete flag word from record offset `0x04`. Optional-field decoding therefore uses this later value, not necessarily the value used for budget accounting.

Initially, the four logical components have these defaults:

| Record offset | Default |
|---|---:|
| `0x10` | `0x00010000` |
| `0x14` | 0 |
| `0x18` | 0 |
| `0x1C` | `0x00010000` |

The reread flags select one branch:

- **`0x08` set:** one F-7efb5338f0fa call supplies a value shifted left by two bits. That shifted value supplies both offsets `0x10` and `0x1C`; the other components retain zero.
- **Otherwise, `0x40` set:** two reader calls supply, in order, the values for offsets `0x10` and `0x1C`, each shifted left by two bits. The other components retain zero.
- **Otherwise, `0x80` set:** four reader calls supply values for offsets `0x10`, `0x18`, `0x14`, and `0x1C`, in that input order. Each is shifted left by two bits.
- **None set:** no optional reader calls occur; all four defaults remain.

These calls explicitly receive the captured input-source value.

After all selected reads, the function writes the components in this order:

1. Offset `0x1C`.
2. Offset `0x10`.
3. Offset `0x14`.
4. Offset `0x18`.

The shifts and stores have 32-bit destinations. For values whose represented signed left shift is well-defined, the numerical effect is multiplication by four. Sign extension, overflow behavior, and out-of-range reader values require dependency or machine-level evidence not supplied here.

### Continuation and publication

The local completed-record count is incremented after the component writes.

The function then rereads the complete flag word at record offset `0x04`. If bit `0x20` is set, it begins another iteration, including another destination-preparation call. Otherwise, it terminates normally.

Normal termination publishes state in this order:

1. Write the completed-record count to destination offset `0x4C`.
2. Write the current remaining budget to context offset `0x1C`.
3. Return a low status of zero.

This successful result follows a successful preparation call in the final iteration. No separate success constant is assigned after publication.

## Side effects

### Partial state on failure

There is no rollback.

An initial budget failure can follow a successful preparation call but precedes this function’s explicit record writes for that iteration.

A later budget failure occurs after the two scalar-zeroing stores and two header reads. Earlier completed records remain written even if a later iteration fails. Nevertheless, the function does not explicitly publish a new count or remaining budget on an error path.

Those statements describe this function’s own accesses. They do not establish that dependencies leave the count, context budget, table, globals, or other memory unchanged.

### Input and aliasing

Reader calls may advance or otherwise modify the opaque input source; their exact effects are unknown. No reader error status is separately checked by this function. Returned values are used as data.

The function retains the initially captured input-source value and destination-object address even if a dependency changes the corresponding context fields.

Within an iteration, it also retains the selected record address across reader calls. No table-address reload occurs before each subsequent record store. Safe execution therefore requires that address to remain usable, or equivalent guarantees outside the supplied evidence.

Aliasing or dependency mutation can affect behavior because flags are read at three distinct stages:

- Budget and scalar-reader selection.
- Optional-component selection.
- Continuation selection.

No snapshot-consistency guarantee is established across these reads.

### Ownership

The function contains no explicit release operation. It does not establish ownership transfer for the context, destination table, or input source.

F-d0804a542462 may manage storage, but allocation, initialization, relocation, ownership, and failure guarantees are unknown. Reader dependencies likewise have no established ownership contract.

## Dependencies

| Opaque ID | Established interaction | Unknown contract |
|---|---|---|
| F-d0804a542462 | Called before each record with the captured destination address and required record count; nonzero status is propagated | Capacity semantics, allocation, initialization, ownership, mutation, and failure side effects |
| F-7efb5338f0fa | Supplies header values, one scalar branch, and optional components | Serialized width, byte order, signedness, extension rules, input exhaustion, error behavior, and the unrecovered argument at one call site |
| F-0b247502190b | Supplies both scalar values when flag `0x01` is clear; receives the captured input-source value | Serialized width, signedness, input advancement, error behavior, and other side effects |

Neither reader’s name nor its use establishes a complete primitive-reading contract.

## Acceptance examples

These examples are conditional static acceptance cases, not executed tests. Unless stated otherwise, preparation succeeds, addresses are valid, arithmetic remains in range, dependencies do not mutate relevant state unexpectedly, and reader values have the stated ordinary integer interpretation.

### One record with no optional components

With an initial budget of 6 and a stored flag word with bits `0x01`, `0x08`, `0x20`, `0x40`, and `0x80` clear:

- The budget reaches zero.
- The two scalar values come from F-0b247502190b.
- Components are `0x00010000`, zero, zero, `0x00010000` at ascending component offsets.
- The published count becomes one.
- The context budget becomes zero.
- The low status is zero.

### Budget failure after header reads

With initial budget 5 and the same relevant flags clear:

- Preparation succeeds and the initial budget check passes.
- The scalar slots are zeroed and both header-reader calls occur.
- The base decrement makes the budget negative.
- Low status is `0x15`.
- No scalar payload readers or component writes occur.
- This function does not publish count or budget.

### Preparation failure takes precedence

If preparation returns a nonzero status while the initial budget is below four, that preparation status is returned. The function does not reach its local budget check or perform explicit writes to the selected record.

### Optional-flag precedence

With initial budget 8, scalar-selector bit `0x01` clear, and all of `0x08`, `0x40`, and `0x80` set consistently:

- The total decrement is 8.
- Only the `0x08` optional branch runs.
- An optional reader value of 3 produces 12 at both diagonal component positions and zero at the other positions.

### Four-component ordering

With initial budget 14, only `0x80` selected among the optional flags, and `0x01` clear, optional reader values 1, 2, 3, and 4 produce:

| Offset | Value |
|---|---:|
| `0x10` | 4 |
| `0x14` | 12 |
| `0x18` | 8 |
| `0x1C` | 16 |

This distinguishes input-read order from ascending destination-field order.

### Failure after an earlier record

If the first record requests continuation and the second preparation call fails, the first record’s writes remain. The second preparation status is returned. This function does not publish a count of one or its locally reduced budget on that error path.

## Uncertainties

The following are documented_only:

- The actual serialized format and whether the budget measures bytes.
- Reader widths, signedness, extension rules, exhaustion handling, and error signaling.
- The missing explicit argument at one scalar-reader call site.
- The meaning and ABI visibility of the high return word.
- Dependency preservation or mutation of globals, context fields, table contents, and storage ownership.
- Behavior at signed-arithmetic extremes, record-index overflow, address wraparound, and problematic left-shift values.
- Concurrency, reentrancy, synchronization, and aliasing guarantees.

The transformation-like interpretation, possible fixed-point scaling, and destination-capacity interpretation are inferred intent, not established semantic guarantees.

## Evidence

Established static evidence consists of the supplied representation’s field accesses, access widths, dependency calls, branch precedence, arithmetic, write order, error exits, and normal-completion publication order.

No dependency implementations, runtime traces, external format specification, or independently verified calling convention were supplied. No runtime tests have been performed.
