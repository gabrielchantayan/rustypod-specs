<!-- Trusted provenance: F-5ad1202cca6f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5ad1202cca6f

## Purpose

F-5ad1202cca6f performs an ordered, short-circuit search over an indexed collection associated with a caller-supplied object. It returns a 32-bit value of one when an examined element satisfies two dependency-defined conditions, and zero when the captured collection bound is exhausted without such an element.

The first condition uses a prepared, query-related temporary and a per-element temporary. The second condition receives only the selected element and is evaluated only when the first condition returns a nonzero result.

**Established static behavior:** preparation occurs before the collection bound is read; elements are selected in ascending index order; success stops further iteration.

**Inferred intent:** the preparation may represent a single-integer interval or query derived from the second argument and its successor. Neither that interpretation nor the meanings of the two conditions are established. The contract therefore describes their actual interfaces and ordering without assigning application-specific semantics.

## Interface

### Inputs

The function receives two arguments:

1. **Object address.** The supplied representation uses a 32-bit address-bearing integer. The function directly accesses:
   - An embedded collection location at byte offset 8 from this address.
   - A signed 32-bit iteration bound at byte offset 12 from this address.

   The collection location is passed by address to F-8966abe1b944. It is not directly loaded as a pointer by this function. Offset 12 may belong to the representation of the collection beginning at offset 8; the evidence does not establish two independent fields.

2. **Address of an integer.** This points to a readable signed 32-bit value. The function directly reads that value to form a successor and also passes the original address to F-7167855775c3.

Neither argument is explicitly checked for nullness, alignment, address validity, or accessible extent.

### Return value

On ordinary completion, the function returns exactly one of these 32-bit values:

- **0:** the captured bound permits no iteration, or every visited element fails at least one required condition.
- **1:** an element produces a nonzero result from both condition dependencies, in the required order.

Nonzero dependency results are treated as truth values. Their magnitudes are not propagated to the caller.

No separate error result is established.

### Temporary storage

The function provides:

- One signed 32-bit local value containing the successor of the integer initially read through the second argument.
- One 16-byte query-related temporary passed to F-7167855775c3.
- One separate 16-byte per-element temporary passed to F-86442406562f.

These are call-local storage regions. Their sizes are established; their internal layouts, required alignment beyond the represented local layout, and dependency-defined invariants are not.

The two 16-byte regions are not explicitly cleared by the surrounding function. Any initialization they receive comes from dependencies. The per-element region is reused across iterations rather than allocated anew for each element.

### Preconditions and defined-behavior limits

For an ordinary successful invocation, the caller and dependencies must collectively provide:

- A valid initial 32-bit read through the second argument.
- A valid 32-bit read at object offset 12 when that read occurs.
- A collection representation acceptable to F-8966abe1b944 at object offset 8.
- Selected element values acceptable to the subsequent dependencies.
- Dependency behavior compatible with the supplied temporary storage and its lifetime.

The source represents the successor calculation as signed 32-bit addition. When the initial integer is the largest signed 32-bit value, its mathematical successor is not representable. The evidence does not establish whether the actual execution wraps, traps, or has some other behavior. Ordinary acceptance examples exclude this arithmetic boundary unless an external execution model resolves it.

Address calculations likewise require a valid interpretation in the represented address space. This function does not establish safe behavior for address overflow or inaccessible object fields.

## Behavior

### Preparation precedes collection inspection

The function first reads the signed 32-bit integer addressed by the second argument and computes its value plus one into local storage.

It then calls F-7167855775c3 with three arguments, in this order:

1. The address of the 16-byte query-related temporary.
2. The original integer address.
3. The address of the local successor value.

The original integer is passed by address, not as a copy of the value used to calculate the successor. Consequently, the initially read integer and any later value observed by the dependency need not be identical if mutation occurs.

After F-7167855775c3 returns normally, the function reads the signed 32-bit bound at object offset 12. This ordering matters: changes made by the preparation dependency before that read can affect the bound used for the search.

Preparation is not skipped for an empty or negative bound. The function has not yet inspected that bound when it invokes F-7167855775c3.

### Bound capture and traversal

The bound is captured once. The function begins with index zero and compares the current index against this captured signed value.

If the bound is zero or negative, the function returns zero without selecting an element or invoking either condition dependency.

For a positive bound, the candidate indices are zero through one less than the captured bound. Selection occurs in ascending order. The function does not reread the bound after each iteration.

This is a snapshot of the iteration limit, not necessarily a snapshot of the collection. Dependencies may change collection state, and subsequent indexed selections may observe that changed state. No synchronization, immutability, or consistency guarantee is established.

For an ordinary signed 32-bit bound, the index progression itself does not require signed overflow. Even at the largest positive bound, the last candidate index is one less than that bound; an unsuccessful final iteration advances the index to the bound, after which the function exits.

### Per-element processing

Each visited index follows this observable call order:

1. F-8966abe1b944 receives the address at object offset 8 and the current index. Its result is retained as an opaque 32-bit element value.
2. F-86442406562f receives the address of the per-element 16-byte temporary.
3. F-50b504e2c17e receives the selected element value, the query-related temporary address, and the per-element temporary address.
4. Only if F-50b504e2c17e returns nonzero, F-86f4ca7480e8 receives the selected element value.

The per-element preparation therefore occurs **after** element selection but **before** the first condition.

If F-50b504e2c17e returns zero, F-86f4ca7480e8 is not called for that element. The next index is examined.

If F-50b504e2c17e returns nonzero but F-86f4ca7480e8 returns zero, the next index is examined.

If both return nonzero, the function immediately returns one. It does not select any later element.

The same selected 32-bit element value is supplied to both condition dependencies. The function does not select it again between those calls. This does not imply that the state identified by that value is unchanged.

### Exhaustion

If every permitted index has been examined without a successful pair of condition results, the function returns zero.

For a positive captured bound and no early success, the selection dependency, per-element preparation dependency, and first condition dependency are each called once per permitted index. The second condition dependency is called only for elements whose first condition result is nonzero.

These call counts describe normal returns from all invoked dependencies. They do not apply after a fault, nonlocal exit, or nonreturning call.

## Side effects

### Direct reads and writes

The surrounding function directly performs two caller-memory data reads relevant to its control flow:

- The initial signed 32-bit integer read through the second argument.
- The signed 32-bit bound read at object offset 12, after query preparation.

Other collection and element reads occur inside dependencies, not as established direct accesses by this function.

Its own explicit data writes initialize or update local values. No direct assignment to caller-owned object fields or to the original integer is shown.

This is **not** a purity guarantee. Dependencies receive addresses or element values through which additional state may be reachable.

### Dependency-mediated mutation

The function does not enforce read-only access to:

- The original integer address.
- The embedded collection representation.
- Either 16-byte temporary.
- State reachable through the selected element.
- Global or external state accessible to dependencies.

It also does not restore such state after dependency calls. No assumption that a dependency preserves globals is justified.

The query-related temporary is prepared once and reused for all examined elements. The first condition dependency receives its address repeatedly. The function does not refresh or restore that temporary between calls. If a dependency mutates it, later behavior may depend on that mutation.

The per-element temporary receives a preparation call on every visited index. The extent to which this replaces prior contents is unknown.

### Ownership and lifetime

No explicit allocation, release, destructor, or cleanup dependency is shown.

Passing an element value onward does not establish ownership transfer. Passing a temporary address does not establish that a dependency may retain it after the invocation. The temporary storage has call-local lifetime; any longer-lived use would require a separate dependency contract not provided here.

If preparation or condition evaluation acquires resources, this function shows no separate release operation on success, exhaustion, or an exceptional path. Whether resources exist, are embedded, are transferred, or are released internally remains unknown.

### Errors and abnormal completion

There is no explicit validation, retry, exception translation, or error recovery. A nonzero condition result is treated as success for that condition, not classified into separate success and error categories.

No behavior is established for invalid memory, dependency faults, invalid element values, or a dependency that does not return normally. Such events must not be silently reinterpreted as a normal zero result.

## Dependencies

| Opaque ID | Role established by this function | Arguments and result use |
|---|---|---|
| F-7167855775c3 | One-time query-related preparation | Receives the first 16-byte region, the original integer address, and the successor address. Its return value is not used. |
| F-8966abe1b944 | Indexed element selection | Receives the collection address at object offset 8 and a signed index. Returns an opaque 32-bit value used as the selected element. |
| F-86442406562f | Per-element temporary preparation | Receives the second 16-byte region. Its return value is not used. |
| F-50b504e2c17e | First required condition | Receives the selected element and both temporary addresses. A zero result rejects the element; a nonzero result enables the second condition. |
| F-86f4ca7480e8 | Second required condition | Receives the selected element only. A nonzero result, following a nonzero first result, causes the function to return one. |

These roles describe call-site behavior, not complete dependency contracts. In particular, “preparation” does not establish construction semantics, and “condition” does not establish freedom from side effects.

## Acceptance examples

These are conditional static acceptance scenarios, not reported runtime tests. They assume valid accesses, representable successor arithmetic, and normal dependency returns.

1. **Zero bound after preparation.**  
   F-7167855775c3 returns, and the subsequent bound read yields zero. The function returns zero. No element selection, per-element preparation, or condition evaluation occurs.

2. **Negative bound.**  
   The captured bound is negative. The function returns zero after query preparation. The bound is not interpreted as a large unsigned element count.

3. **Preparation changes the bound.**  
   The object initially contains a zero bound, but F-7167855775c3 changes it to two before returning. The function captures two and may examine indices zero and one. An implementation that captures the bound before preparation would differ observably.

4. **First condition rejects the only element.**  
   The captured bound is one. Selection at index zero and per-element preparation occur. F-50b504e2c17e returns zero. F-86f4ca7480e8 is not called, and the function returns zero.

5. **Second condition rejects the only element.**  
   The captured bound is one. The first condition returns nonzero and the second returns zero. The function returns zero after completing that iteration.

6. **Later element succeeds.**  
   The captured bound is three. Index zero fails at least one condition; index one produces nonzero results from both. The function returns one without selecting index two.

7. **Negative condition results remain truthy.**  
   Both condition dependencies return negative signed 32-bit values for an element. Because both are nonzero, the function returns one rather than requiring either result to equal positive one.

8. **A later bound mutation does not shorten the captured limit.**  
   The captured bound is three. A subsequent dependency changes the stored bound to zero. If collection access remains valid and no element succeeds, the function still attempts indices zero, one, and two. It does not reread the changed bound.

9. **Original integer changes during preparation.**  
   The initial read yields seven, so the local successor initially contains eight. F-7167855775c3 receives the original integer address and the separate successor address. The contract does not require the original integer to remain seven, nor require the dependency to observe an immutable pair.

10. **Maximum initial integer.**  
    The integer initially read is the largest signed 32-bit value. No portable successor result is established by this evidence. Accepting a wrapped minimum value requires an independently established execution model.

## Uncertainties

The following remain **documented_only**:

- The semantic meanings of the integer, collection, element values, and conditions.
- The full object layout and collection representation.
- Whether the opaque element value is a pointer, handle, integer, or another representation.
- The contents and invariants of both 16-byte regions.
- Dependency reads, writes, allocation, retention, release, and global-state effects.
- Behavior under concurrent mutation or collection invalidation.
- The actual execution semantics of signed successor overflow.
- Fault, exception, cancellation, and nonlocal-exit behavior.
- Whether any dependency uses a return value as an error indication that this caller nevertheless interprets as truth.

No interval interpretation, memory-ownership convention, or dependency purity assumption is required by this contract.

## Evidence

The established behavior comes solely from static inspection of the supplied function representation: argument widths, fixed byte offsets, temporary extents, call arguments, call order, signed comparisons, short-circuit evaluation, and literal return values.

Names used here for query preparation, element selection, and conditions are descriptive roles. They do not establish application semantics beyond the observed call structure.

No dependency implementation or independent interface specification was supplied. No runtime tests have been performed. The acceptance examples state consequences of the visible control flow under their explicit assumptions; they are not execution results.
