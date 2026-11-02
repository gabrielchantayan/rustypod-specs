<!-- Trusted provenance: F-ef14a23acfbb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ef14a23acfbb

## Purpose

F-ef14a23acfbb uses a signed numerical input to conditionally inspect a context-associated table. An eligible entry whose value is nonzero selects an ordered pair of dependency calls.

The established behavior is:

- Call F-ecd921e533b1 to obtain a context base.
- Reject negative inputs without explicitly reading context fields or table entries.
- For nonnegative inputs, read a signed bound and require the input to be strictly below it.
- Read one 32-bit table entry only after that range condition passes.
- Return normally if the selected entry is zero.
- If the entry is nonzero, call F-45554e48c02d and then, if control returns, F-feee9156a257.

The application-level meaning of the table and the conditional calls is unknown. “Eligible” denotes only satisfaction of the observed range condition. It does not establish that an address is valid or that an entry has any particular ownership or application status.

## Interface

### Input

The function accepts one pointer-shaped argument. The visible body interprets its address representation as a signed 32-bit numerical value. It does not dereference the argument as a pointer to caller-provided data.

Throughout this contract, **input** means that signed numerical interpretation.

Consequences:

- A valid object pointer is not necessarily an eligible input.
- A null argument represents zero and is not unconditionally rejected.
- Inputs with a negative signed interpretation are rejected after the initial dependency call.
- No pointee contents, object size, or caller-provided allocation are inspected through the argument.

The reason for the pointer-shaped interface is not established.

### Output and completion

The function has no explicit return value or output parameter.

Subject to dependencies returning normally and required memory accesses succeeding, normal return occurs when:

- The input is negative.
- The input is nonnegative but not strictly below the signed bound.
- The input selects an entry whose value is zero.

An eligible nonzero entry selects the dependency-call path. F-feee9156a257 is marked nonreturning in the supplied static representation; ordinary return after that call is not established.

### Widths and accessed layout

The relevant representation uses 32-bit integer quantities and 32-bit address-bearing fields.

| Location | Access width | Established use |
|---|---:|---|
| Context base plus byte offset 196 | 32 bits | Bound, compared using its signed interpretation |
| Context base plus byte offset 192 | 32 bits | Table-base address |
| Table base plus four times the input | 32 bits | Selected entry, classified as zero or nonzero |

The context base is the result of F-ecd921e533b1. The two context fields are distinct: the bound is read first, and the table-base field is read only if the range condition succeeds.

The bound has a pointer-shaped representation in the supplied material, but this function uses it numerically. It is not dereferenced as an address.

No broader context layout, table allocation extent, alignment requirement, or byte order is established.

### Preconditions

F-ecd921e533b1 is invoked for every input. Its preconditions are unknown.

The function’s explicit memory requirements are conditional:

- Negative input: no context-field or table read is required by this body.
- Nonnegative input: the four-byte bound field must be readable.
- Eligible input: the four-byte table-base field and selected four-byte entry must also be readable.

Range eligibility alone does not establish valid address arithmetic or readable storage. The function contains no explicit validation of context validity, table capacity, alignment, or arithmetic overflow, and provides no explicit recovery path for an invalid memory access.

## Behavior

### 1. Obtain the context

The first dependency invocation is F-ecd921e533b1. It occurs once, before the input can cause an ordinary return.

This includes negative inputs and null. The function is therefore not guaranteed to be a side-effect-free rejection operation. The unconditional call establishes invocation, not actual mutation: neither purity nor state changes by F-ecd921e533b1 are established.

If the call returns, its result supplies the context base for subsequent explicit accesses. The function does not call the provider again.

If the provider does not return normally, none of the later behavior is guaranteed to occur.

### 2. Classify the input

The input is interpreted as a signed 32-bit value.

For a negative input, the function returns normally after the context-provider call returns. It does not explicitly read the bound field, table-base field, or any table entry. Neither conditional-path dependency is invoked.

The supplied representation includes an initial comparison against null. This does not create a separate null-rejection rule. For every nonnegative input, including zero, the comparison used in the subsequent decision is replaced by an equality comparison against the loaded bound.

Forming a context-relative address in the representation does not itself establish a memory read. The negative-input path contains no explicit dereference of that address.

### 3. Read and apply the signed bound

For a nonnegative input, the function reads the 32-bit field at context offset 196.

The subsequent decision requires both inequality with the bound and a signed less-than-or-equal comparison. Together, these establish an exclusive upper bound:

**The input must be nonnegative and strictly less than the signed bound.**

Accordingly:

- A zero bound admits no input.
- A negative bound admits no input.
- A positive bound admits the numerical range from zero through one less than the bound.
- Equality is excluded.
- Inputs above the bound are excluded.
- A bound with a negative signed interpretation is not treated as a large unsigned limit.

If the condition fails, the function returns without explicitly reading the table-base field or any table entry.

The bound is read once in the visible body. It is not reloaded before table access or before the conditional dependency calls.

### 4. Read one selected entry

When the range condition succeeds, the function reads the table-base address from context offset 192. It then uses the input scaled by four to address one 32-bit entry relative to that base.

The explicit read order is:

1. Bound field.
2. Table-base field.
3. Selected table entry.

The stride is four bytes. There is no visible search, neighboring-entry scan, table-length check, or second read of the selected entry.

Only the distinction between zero and nonzero affects the subsequent behavior. The entry’s sign and magnitude do not select different branches. Its possible interpretation as an address, handle, flag, or ownership marker is unknown.

The multiplication and address additions have no visible overflow guard. The range condition does not independently make those operations safe.

### 5. Return or invoke the conditional dependencies

If the selected entry is zero, the function returns normally.

If the selected entry is nonzero:

- F-45554e48c02d is invoked.
- If that call returns normally, F-feee9156a257 is invoked.

Neither call has an explicit argument in the supplied representation. No result from F-45554e48c02d is used by this function.

There is no intervening reload or revalidation of the bound, table base, or selected entry. In particular, a state change by F-45554e48c02d does not cause this function to reconsider its decision before calling F-feee9156a257.

The static representation classifies F-feee9156a257 as nonreturning. It does not establish whether that means termination, an exception, a nonlocal transfer, or another control-flow effect.

## Side effects

### Explicit reads and calls

The following sequences assume normal continuation through preceding calls and successful memory accesses:

| Condition | Explicit sequence |
|---|---|
| Negative input | Call F-ecd921e533b1; return |
| Nonnegative input failing the bound condition | Call F-ecd921e533b1; read bound; return |
| Eligible input with zero entry | Call F-ecd921e533b1; read bound; read table base; read entry; return |
| Eligible input with nonzero entry | Call F-ecd921e533b1; read bound; read table base; read entry; call F-45554e48c02d; call F-feee9156a257 if reached |

These are the operations explicit in this function. They do not enumerate operations performed inside dependencies.

### Writes and state preservation

The visible body contains no explicit stores to caller-provided data, context fields, table entries, or globals.

This does not establish that an invocation leaves state unchanged. Any invoked dependency may have effects not described here. Conversely, the existence of a dependency call does not prove that any state changes.

No dependency is established to preserve globals, context contents, table storage, allocations, or other process state.

### Ownership and lifetime

The visible body contains no explicit allocation, deallocation, reference-count adjustment, ownership transfer, or lifetime extension.

The argument is not used as an owned pointee. Context and table storage are read, but their ownership and lifetime rules are unknown.

Dependency call order does not establish that a callee acquires, retains, releases, or invalidates memory. Such effects remain unresolved.

### Errors and concurrency

The function provides no explicit status result, error output, fallback, retry, or memory-fault handler.

No synchronization operation is visible. Ordered field reads do not establish a coherent snapshot under concurrent modification. The contract therefore makes no guarantee about concurrent changes to the context, bound, table base, or selected entry.

## Dependencies

| Opaque function ID | Established role | Unresolved properties |
|---|---|---|
| F-ecd921e533b1 | Called once before input-dependent return; its result supplies the context base | Preconditions, provenance, purity, side effects, failure behavior, ownership, lifetime, global preservation |
| F-45554e48c02d | Called after an eligible nonzero entry is observed | Purpose, side effects, ownership effects, failure behavior, whether it returns normally |
| F-feee9156a257 | Called after F-45554e48c02d returns on the selected path; statically marked nonreturning | Exact control transfer, termination mechanism, state effects, ownership effects |

No internal behavior of these dependencies is established by the supplied function body.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Unless an example states otherwise, required memory is readable, relevant address calculations are valid, and preceding dependencies return normally.

### Negative input avoids explicit field reads

For input −1, F-ecd921e533b1 is invoked. After it returns, this function returns without explicitly reading either context field or any table entry.

The same branch behavior applies to −2,147,483,648, the minimum signed 32-bit value.

These examples do not require the provider to be pure. They establish only the absence of subsequent explicit field reads by this function.

### Null can select the first entry

For a null argument, bound 3, and a valid table:

- The bound is read.
- The table base is read.
- The entry at byte displacement zero is read.

A zero entry produces ordinary return. A nonzero entry selects F-45554e48c02d followed, if reached, by F-feee9156a257.

Unconditional rejection of null would not satisfy this behavior.

### Equality excludes table access

For input 3 and bound 3, the function reads the bound and returns. It does not explicitly read the table-base field or entry 3.

For input 4 and bound 3, the same read-and-return pattern applies.

### Nonpositive bounds admit nothing

For input zero and bound zero, the function returns after reading the bound.

For input zero and bound −1, it also returns after reading the bound. Neither case reads the table-base field.

### Entry contents control the call path

For input 2 and bound 4, the selected entry lies eight bytes after the table base.

If that 32-bit entry is zero, the function returns normally. If it is any nonzero value, including a value with a negative signed interpretation, the conditional dependency path is selected.

### Earlier control transfer prevents later calls

For an eligible nonzero entry, if F-45554e48c02d does not return normally, F-feee9156a257 is not reached.

If F-45554e48c02d returns after changing the selected entry to zero, this function still proceeds to F-feee9156a257: it performs no second entry read.

### Range success does not prove arithmetic safety

Input 536,870,912 is below a signed bound of 536,870,913. It therefore passes the numerical range condition.

Its four-byte displacement is mathematically 2,147,483,648, outside the positive signed 32-bit range. Successful address formation and a successful entry read are not established for this case. The contract supplies no overflow rejection, saturation, wider arithmetic, or safe-wrap guarantee.

## Uncertainties

The following remain **documented_only**, not acceptance guarantees:

- **Address arithmetic:** Machine-level behavior for context-offset addition, input scaling, and table-address addition is not independently established. No portable signed-overflow rule is inferred.
- **Memory representation:** Byte order, alignment, mappings, and allocation extents are unspecified.
- **Interface origin:** The numerical index-like use is established; the reason for a pointer-shaped argument is unknown.
- **Bound meaning:** An exclusive signed upper bound is established. Capacity, population, and other application meanings are not.
- **Entry meaning:** Only zero versus nonzero is established.
- **Dependency purity:** Neither purity nor actual mutation is established for the context provider or the other dependencies.
- **Ownership:** No callee is known to preserve, retain, acquire, or release memory.
- **Terminal behavior:** The nonreturning classification is static evidence, not runtime confirmation of a particular outcome.
- **Exceptional outcomes:** Faults, exceptions, invalid-memory behavior, and nonlocal transfers are not specified by the visible body.
- **Concurrency:** No synchronized or coherent multi-field observation is established.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence comprises the unconditional provider call; signed input classification; field offsets and access widths; exclusive-bound condition; four-byte entry stride; zero/nonzero decision; conditional call order; absence of explicit data stores; and the nonreturning annotation on F-feee9156a257.

“Index-like,” “context,” and “table” describe observed structural roles. They do not assign application-level semantics.

No runtime tests, implementation execution, dependency inspection, or dynamic memory observations have been performed.
