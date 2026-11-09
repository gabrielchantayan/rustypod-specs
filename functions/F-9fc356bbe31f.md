<!-- Trusted provenance: F-9fc356bbe31f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9fc356bbe31f

## Purpose

F-9fc356bbe31f obtains a primary object and performs a selector-dependent sequence of indirect calls. Selectors 0 through 5 select distinct paths. Every other selector still triggers primary-object acquisition and null handling, but no selector-specific calls.

The normal result is the 32-bit value 0. Selectors 3, 4, and 5 instead return 2 when their initial primary-object query returns nonzero. A null primary pointer triggers F-a817d8d58c10, whose call site is annotated as nonreturning.

Established behavior consists of acquisition, dispatch-table access, ordered calls, conditional branches, local assignments, and explicit results. Application-level meanings such as activation, shutdown, reset, or release are not established. In particular, the final secondary-object operation is not proven to destroy or release anything.

## Interface

### Inputs and result

| Item | Representation in the supplied model | Use |
|---|---|---|
| First argument | Opaque 32-bit value | Not read by the visible body. |
| Selector | 32-bit integer | Selects cases 0 through 5; all other values take the default path. |
| Result | 32-bit value | 0 on ordinary paths, except for the conditional result 2 described below. |

Negative selector values and values greater than 5 are not rejected with a distinct error. They follow the default path after acquisition and null handling.

The stated widths describe the supplied source model. Independent binary-layout and calling-convention verification has not been performed. Query results are represented as 32-bit integers and tested only for zero versus nonzero; their sign and magnitude otherwise have no effect.

A result of 0 is not a general success guarantee. Some dependency results are ignored, and several branches return 0 without performing a final action.

There is no ordinary result when a reached dependency does not return. For a null primary pointer, nonreturning behavior is supported by the annotation on F-a817d8d58c10, not by runtime observation.

### Objects and dispatch layout

This contract uses three role labels, not concrete type names:

- **Primary object:** the pointer returned by F-b7af80e829ec.
- **Helper object:** the pointer returned by a reached call to F-cdd33263ee89.
- **Secondary object:** the pointer returned by F-be45cd43f92d with argument 0.

The supplied model reads a pointer-bearing first word from each object and uses it as a dispatch-table base. Offsets below are **byte offsets from that base**, not entry indices or object-field offsets.

Each reached indirect call requires a readable object first word, a usable dispatch-table base, and an invocable entry at the specified offset. These are execution preconditions; the function does not validate them.

The primary pointer is checked for null immediately after acquisition. Helper pointers are not checked. The secondary pointer is checked only after several operations have already dereferenced and used it.

Argument lists in this contract describe explicit arguments in the supplied representation. The helper operation has no explicit arguments; this contract does not add an implicit receiver.

### Selector-1 local storage

One conditional selector-1 path uses two separate local objects:

- A four-byte local assigned the value of a 32-bit global datum.
- A one-byte local assigned zero afterward.

The address of the **four-byte local** is passed to a primary-object operation. The relative placement of the two locals is not established. In particular, the one-byte local is not proven to occupy byte offset 4 from the passed address, or to be accessible through that pointer.

## Behavior

### Common entry

Every invocation first calls F-b7af80e829ec without explicit arguments and retains its result as the primary pointer.

If that pointer is null, the next dependency call is F-a817d8d58c10. Its call site is annotated as nonreturning. No recovery, substitute object, or normal error result is specified for this condition. Behavior if that dependency unexpectedly returns remains unresolved.

Selector dispatch occurs only after this acquisition and null handling. Unsupported selectors therefore do not bypass either operation.

### Helper sequence

A reached helper sequence:

1. Calls F-cdd33263ee89 without explicit arguments.
2. Reads the helper object’s first-word dispatch-table base.
3. Reads the callable entry at table offset `0xB8`.
4. Invokes that entry with no explicit arguments.

The indirect result is ignored. Each reached sequence obtains a helper through a fresh dependency call; the function does not cache a helper across invocations.

Later primary-object dispatch accesses occur after this sequence. The sequence is not assumed to preserve primary-object state, dispatch tables, or globals.

### Selector 0

The helper sequence occurs first. The function then calls the primary entry at `0x120` with the primary pointer.

- If the query returns nonzero, the function returns 0 without calling the primary entry at `0xA8`.
- If the query returns zero, the function reads and invokes the primary entry at `0xA8`, passing the primary pointer, ignores its result, and returns 0.

Thus the helper sequence occurs regardless of the primary query’s outcome.

### Selector 1

The function performs the helper sequence, then calls the primary entry at `0x110` with the primary pointer.

If that query returns nonzero, the function calls the primary entry at `0xAC` with the primary pointer, ignores the result, and returns 0. This path does not call the primary entry at `0x120` or acquire a secondary object.

If the query at `0x110` returns zero, the function calls the primary entry at `0x120`, again with the primary pointer.

- A zero result causes an immediate return of 0, without secondary acquisition.
- A nonzero result enters the secondary-object path below.

#### Secondary-object path

The function calls F-be45cd43f92d with the explicit argument 0. Using its returned secondary pointer, it performs these indirect calls in order:

| Secondary table offset | Explicit arguments | Result use |
|---|---|---|
| `0x38` | Secondary pointer | Ignored |
| `0xD8` | Secondary pointer | Ignored |
| `0x5C` | Secondary pointer, 1, 0 | Ignored |
| `0x108` | Secondary pointer | Tested for zero/nonzero |

There is no null check between acquisition and these calls.

If the query at `0x108` returns zero, the function skips the local assignments and the three primary calls described next.

If that query returns nonzero, the function performs the following ordered operations:

1. Reads the 32-bit global datum and assigns its value to the four-byte local.
2. Assigns zero to the separate one-byte local.
3. Calls the primary entry at `0x20` with these explicit arguments, in order:
   - Primary pointer.
   - Address of the four-byte local.
   - The 32-bit all-ones value, `0xFFFFFFFF`.
   - 0.
   - 3.
   - 3.
   - Secondary pointer.
4. Calls the primary entry at `0x28` with the primary pointer and 0.
5. Calls the primary entry at `0xA4` with the primary pointer, `0xFFFFFFFF`, 0, 0, and 0.

The results of all three primary calls are ignored.

The global read occurs after the secondary query returns, not at function entry. Changes made by earlier dependencies can therefore affect the value copied.

The local assignments do **not** establish a contiguous five-byte buffer or a terminated string. The four-byte local’s byte order is unspecified, its value may contain zero bytes, and the separate zero byte is not proven reachable through the address passed to `0x20`. No guarantee is made that memory immediately following the four-byte local is zero.

After either outcome of the secondary query, and after any conditional primary calls, the retained secondary pointer is tested for null:

- If null, the function returns 0.
- Otherwise, it reads the secondary entry at `0x1C`, invokes it with the secondary pointer, ignores its result, and returns 0.

This late test is not an acquisition guard: the secondary pointer has already been dereferenced and passed to four indirect operations.

### Selector 2

The function performs the helper sequence, then calls the primary entry at `0x10C` with the primary pointer.

- A zero query result causes an immediate return of 0.
- A nonzero query result causes a call to the primary entry at `0xAC`, passing the primary pointer. Its result is ignored, and the function returns 0.

The final entry is the same offset used by selector 1’s first-query branch. The preceding query and branch condition differ.

### Selectors 3, 4, and 5

These selectors first call the primary entry at `0x120` with the primary pointer.

If the query returns nonzero, the function returns 2 immediately. It does not obtain a helper or perform a final action.

If the query returns zero, the function performs the helper sequence and then one selector-specific primary call:

| Selector | Primary table offset | Explicit arguments |
|---|---|---|
| 3 | `0xB0` | Primary pointer, 0 |
| 4 | `0xB4` | Primary pointer, 0 |
| 5 | `0xB8` | Primary pointer |

The selected call’s result is ignored, and the function returns 0.

Each path performs exactly one final primary call. Selectors 3 and 4 do not invoke the selected target again with a shorter argument list.

For these selectors, the helper sequence follows the primary query. For selectors 0, 1, and 2, it precedes their first primary query.

### Other selectors

Any selector outside 0 through 5 returns 0 after common acquisition and null handling. It performs no helper acquisition or indirect dispatch.

## Side effects

### Direct memory effects

The visible body contains no explicit object-field or global-data writes.

On the qualifying selector-1 path, it explicitly performs two local assignments, in order:

- A four-byte local receives the value read from a 32-bit global datum.
- A separate one-byte local receives zero.

These assignments establish neither adjacency nor a five-byte memory region beginning at the four-byte local. The zero byte’s accessibility through the passed pointer is unverified.

The address of the four-byte local is supplied to the primary operation at `0x20`. Whether that dependency reads, modifies, copies, or retains the addressed storage is unknown. The storage is local to the invocation; no validity after return is established. No ownership transfer is demonstrated.

### Observable ordering and state changes

Dispatch-table bases and entries are read at the corresponding call sites rather than cached once for the entire invocation. Earlier calls can therefore affect later dispatch if they change reachable state.

In particular:

- Selector-1 secondary calls occur in the order `0x38`, `0xD8`, `0x5C`, `0x108`.
- A nonzero result from `0x108` precedes the global read and both local assignments.
- The conditional primary calls occur in the order `0x20`, `0x28`, `0xA4`.
- The final secondary call at `0x1C`, when reached, occurs afterward.

No dependency result is directly propagated as the function’s result. Query results only control branches.

All additional state changes are dependency-dependent. No callee is assumed to preserve globals, object contents, dispatch tables, or memory lifetime.

### Ownership and failures

The visible body establishes no allocation, deallocation, reference-count adjustment, or ownership transfer. Returned pointers are used without a visible ownership declaration.

The secondary operation at `0x1C` may have a lifecycle role, but that is inferred intent, not guaranteed cleanup or release.

There is no visible retry, rollback, exception handler, timeout, or failure translation. If a dependency does not return normally, subsequent operations are not guaranteed. If a dependency invalidates an object needed later, the function supplies no general validity recheck.

Invalid object or dispatch-table accesses have no specified recovery behavior. The late secondary null test must not be used to infer that a null secondary acquisition is safely handled.

## Dependencies

### Direct dependencies

| Opaque function ID | Visible interaction | Unresolved contract |
|---|---|---|
| F-b7af80e829ec | Called once at entry without explicit arguments; supplies the primary pointer. | Identity, lifetime, ownership, side effects, failure behavior. |
| F-a817d8d58c10 | Called for a null primary pointer; annotated nonreturning. | Actual control-transfer mechanism and other effects. |
| F-cdd33263ee89 | Called without explicit arguments for a reached helper sequence. | Nullability, identity, lifetime, ownership, state changes. |
| F-be45cd43f92d | Called with 0 on the qualifying selector-1 path; supplies the secondary pointer. | Meaning of 0, nullability, lifetime, ownership, state changes. |

### Indirect dependencies

Opaque identities for indirect targets are unavailable. Their observable call sites are distinguished by object role and table byte offset:

- **Primary queries:** `0x10C`, `0x110`, `0x120`.
- **Primary actions:** `0x20`, `0x28`, `0xA4`, `0xA8`, `0xAC`, `0xB0`, `0xB4`, `0xB8`.
- **Helper action:** `0xB8`, with no explicit arguments.
- **Secondary operations:** `0x1C`, `0x38`, `0x5C`, `0xD8`, `0x108`.

Equal offsets on different object roles do not establish identical targets or semantics. Argument lists and ordering are specified above; concrete callee contracts are not available.

## Acceptance examples

These are conditional acceptance criteria from static evidence, not executed tests. Except where stated otherwise, assume valid object/table accesses and normally returning dependencies.

1. **Unsupported selector:** With selector −1 or 6 and a nonnull primary pointer, F-b7af80e829ec is called, no helper or indirect call occurs, and the result is 0.

2. **Null primary:** If F-b7af80e829ec returns null, F-a817d8d58c10 is called regardless of selector. If its nonreturning annotation holds, no ordinary result follows.

3. **Selector 0:** The helper sequence precedes the query at primary `0x120`. A nonzero query suppresses `0xA8`; a zero query permits it. Both ordinary paths return 0.

4. **Selector 1, first query nonzero:** Following the helper sequence, a nonzero result from primary `0x110` leads to primary `0xAC`. Primary `0x120` and secondary acquisition are omitted.

5. **Selector 1, both primary queries zero:** After zero results from `0x110` and `0x120`, the function returns 0 without calling F-be45cd43f92d.

6. **Selector 1, secondary query zero:** After qualifying secondary acquisition, the four secondary calls occur in the specified order. A zero result from `0x108` suppresses both local assignments and primary `0x20`, `0x28`, and `0xA4`. With a nonnull secondary pointer, `0x1C` still occurs.

7. **Selector 1, secondary query nonzero:** The global value is read only after `0x108` returns nonzero. The four-byte assignment precedes the separate one-byte zero assignment. Primary `0x20`, `0x28`, and `0xA4` follow in order, then secondary `0x1C` for a nonnull pointer.

8. **Local-storage boundary:** On that nonzero secondary-query path, acceptance requires the address passed to primary `0x20` to identify the initialized four-byte local. It does not require a zero at address-plus-four or permit assuming that the passed object is a five-byte terminated string.

9. **Selector 2:** A zero query at primary `0x10C` suppresses `0xAC`; a nonzero query permits it. The helper sequence precedes the query, and both ordinary paths return 0.

10. **Selectors 3 through 5, early result:** A nonzero initial query at primary `0x120` returns 2 and suppresses helper acquisition and the selector-specific final action.

11. **Selectors 3 through 5, final calls:** After a zero query and the helper sequence, selector 3 calls `0xB0` with primary pointer and 0; selector 4 calls `0xB4` with those arguments; selector 5 calls `0xB8` with only the primary pointer. Each final call occurs once, and the result is 0.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of selectors, queries, actions, and result 2.
- Concrete object types and indirect target identities.
- Independently verified pointer widths, calling conventions, and implicit arguments.
- The global datum’s value, byte order, and interpretation.
- Relative placement of the four-byte and one-byte locals, including whether the zero byte is accessible through the passed pointer.
- Any string interpretation or readable extent beyond the four-byte local.
- Signed interpretation of the all-ones arguments; the established 32-bit pattern is `0xFFFFFFFF`.
- Callee access to, modification of, or retention of local storage.
- Ownership, reference counting, object invalidation, and release behavior.
- Whether F-a817d8d58c10 actually never returns, and behavior if it does.
- Behavior after invalid accesses or violated dependency preconditions.
- Dependency mutation of objects, globals, dispatch tables, or other reachable state.
- Thread safety, reentrancy, and synchronization requirements.

A guarded state-transition dispatcher is a plausible interpretation. It adds no required behavior or ownership guarantees.

## Evidence

This contract is based solely on static inspection of the supplied function representation and opaque dependency mapping.

Established evidence includes selector comparisons, zero/nonzero branches, explicit argument lists, dispatch-table byte offsets, call order, the two separate local assignments, and explicit return values.

The representation does not establish the locals’ relative layout. Their names are not layout evidence, and no contiguous five-byte temporary is asserted.

Dependency implementations, runtime object layouts beyond the accessed model, and the global datum’s contents were not supplied. No runtime tests, binary inspection, or independent calling-convention checks have been performed.
