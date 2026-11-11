<!-- Trusted provenance: F-ff3d9f8eb048; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff3d9f8eb048

## Purpose

F-ff3d9f8eb048 routes an event according to two byte-sized object flags, a 32-bit event classification, values obtained from dependencies, and a 32-bit object classification.

Its established behavior consists of:

- Returning early for certain event/value combinations.
- Delegating selected events and forwarding the delegate’s return value.
- Issuing object-directed notifications with values `0x14` or `0x16`.
- Computing a Boolean result from the object’s classification after selected notifications and queries.

“Event,” “classification,” and “notification” are descriptive labels, not established application-level meanings. The supplied evidence does not establish whether returning `1` means handled, accepted, consumed, or something else.

## Interface

### Inputs and layout

The function receives two address-bearing arguments in a 32-bit representation:

| Input | Required accessible fields | Interpretation within this function |
|---|---|---|
| Object | Offset `+0x04`: 32-bit integer | Object classification |
| Object | Offset `+0xCC`: one byte | Primary routing flag |
| Object | Offset `+0xCD`: one byte | Secondary routing flag |
| Event | Offset `+0x04`: 32-bit integer | Event classification |

The primary and secondary flags are tested only for zero versus nonzero. No particular nonzero value is required.

The event classification is read and compared as a signed 32-bit integer in the initial dispatch logic. The recognized classifications are `6`, `7`, `8`, `0x0D`, and `0x15`. Other values, including negative values, take rejection paths where specified below.

The object classification is compared against explicit 32-bit constants. Its broader meaning and valid domain are unknown.

When the primary flag is zero, F-fdd8589eca2d supplies an additional address-bearing result. This function accesses:

| Offset from returned address | Width | Use |
|---|---|---|
| `+0xA0` | One unsigned byte | First prefilter flag |
| `+0xA1` | One unsigned byte | Second prefilter flag, read only if the first is nonzero |

These layouts describe accessed fields only. They do not establish complete structure sizes, alignment guarantees, ownership, or additional fields.

### Output

The result is a 32-bit integer.

Most paths return exactly `0` or `1`. Two delegation paths return the result of F-cead90729166 or F-289632f4e99e without normalization. Consequently, the function’s complete return domain is not established as Boolean.

### Preconditions

Every field actually read must be accessible at the time of that read. A path that invokes a dependency also requires whatever preconditions that dependency imposes; those contracts are unavailable here.

There are no explicit null checks, bounds checks, or object-lifetime checks. Invalid addresses and dependency failures have no defined recovery behavior in the supplied evidence.

The function does not establish ownership transfer for either input.

## Behavior

### 1. Primary-flag selection

The first observable field access is a one-byte read of the object’s primary routing flag at `+0xCC`.

- Zero selects the prefilter-and-delegation route.
- Nonzero selects the notification-and-classification route.

This initial choice is not revisited, even if a dependency subsequently changes the flag.

### 2. Primary flag zero: prefilter

The function first calls F-fdd8589eca2d without an explicit argument.

It reads the unsigned byte at `+0xA0` from the returned address. Only if that byte is nonzero does it read the unsigned byte at `+0xA1` from the same returned address.

If both bytes are nonzero, it then reads the event classification:

- If that classification is not `0x0D`, no prefilter query occurs.
- If it is `0x0D`, the function calls F-11c1b7e655b3 with the event argument.
- If the low 32 bits of that query result equal `0x6E` or `0x6F`, the function returns `1` immediately.

The query result is represented as 64 bits at this call site, but the early-return decision uses only its low 32 bits. The high 32 bits do not affect a later observable result: a later notification argument is replaced by the constant `0x16` before use.

If either prefilter byte is zero, the event classification is not read for this prefilter. The normal dispatch below still reads it.

### 3. Primary flag zero: secondary routing

Unless the prefilter returned early, the function freshly reads the event classification and then reads the object’s secondary routing flag at `+0xCD`.

The classification read for this dispatch is retained for the dispatch decision. It is not a promise that dependencies will observe the same event contents.

#### Secondary flag zero

| Retained event classification | Behavior |
|---|---|
| `6`, `7`, `8`, or `0x0D` | Call F-cead90729166 with the object and event; return its result unchanged |
| `0x15` | Call F-289632f4e99e with the object and event; return its result unchanged |
| Any other value | Return `0` |

#### Secondary flag nonzero

| Retained event classification | Behavior |
|---|---|
| `6`, `7`, `8`, or `0x0D` | Call F-1c60e26b3436 with the object and `0x16`; then return `0` |
| Any other value, including `0x15` | Return `0` without that notification |

This route does not inspect the object classification.

### 4. Primary flag nonzero: initial dispatch

The function reads the event classification once for initial dispatch.

| Initial event classification | Behavior |
|---|---|
| `6`, `7`, `8`, or `0x0D` | Continue to event preparation |
| `0x15` | Follow the special path below |
| Any other value | Return `0` |

For initial classification `0x15`, the function reads the object classification:

- If it is `0x24`, return `1` without a notification.
- Otherwise, call F-1c60e26b3436 with the object and `0x14`, then return `1`.

The `0x15` path does not perform event preparation or the later query/classification stages.

### 5. Event preparation and first query

For an initially selected event classification of `6`, `7`, `8`, or `0x0D`, the function:

1. Calls F-682adfd5a561 with the event and the integer `0x0A00`.
2. Calls F-11c1b7e655b3.
3. Interprets the latter call’s result as a 32-bit value.

The supplied representation shows no explicit argument for this particular query call. Other calls to the same dependency explicitly pass the event. The actual argument convention at this call site is therefore unresolved and must not be silently replaced with an assumed event argument.

If the first query result is one of:

`0x10`, `0x59`, `0x5A`, `0x6E`, `0x6F`

the function calls F-1c60e26b3436 with the object and `0x14`, then returns `1`.

All other 32-bit results, including negative values, continue below.

The classification selected before event preparation is not reused to control the following checks. Event classifications are reread after dependency calls.

### 6. First conditional `0x16` notification

The function checks whether the event currently has classification `0x0D` or `8`, using short-circuit evaluation:

- Read the classification and compare it with `0x0D`.
- Only if that comparison fails, read the classification again and compare it with `8`.

If neither comparison succeeds, this notification stage is skipped.

If either succeeds, the function queries F-11c1b7e655b3 with the event:

- A result of `0x45` suppresses this notification stage.
- Otherwise, it queries the dependency again. A result of `0x4D` suppresses this stage.
- Otherwise, it queries a third time.

For that third query:

- A result other than `0x1C` causes a notification with `0x16`.
- A result of `0x1C` causes a fresh event-classification read. Classification `8` suppresses the notification; any other classification causes it.

Each query result belongs to a distinct call. They must not be collapsed into a single cached value.

### 7. Second conditional `0x16` notification

After the preceding stage, including any notification it issued, the function freshly reads the event classification.

If it is `6`, the function queries F-11c1b7e655b3 with the event. A result of `0x3B` causes a call to F-1c60e26b3436 with the object and `0x16`.

If the classification is not `6`, or that query does not return `0x3B`, this stage issues no notification.

This stage is independent of the preceding one. Without assumptions about dependency effects, both stages can notify during one invocation.

### 8. Final selection and object-classification result

The function next chooses between a broad and a narrow object-classification set.

First it freshly reads the event classification:

- If it is `6`, it queries F-11c1b7e655b3 with the event.
- If that query returns `0x3B`, the narrow set is selected immediately. The second event condition below is not evaluated.

Otherwise, it freshly reads the event classification again:

- If it is `0x0D`, it queries F-11c1b7e655b3 with the event.
- A result of `0x1C` selects the narrow set.
- Any other outcome selects the broad set.

Only after this selection does the function read the object classification.

| Selected set | Object classifications returning `1` |
|---|---|
| Broad | `0x0A`, `0x0D`, `0x0E`, `0x0F`, `0x18`, `0x19`, `0x23` |
| Narrow | `0x0D`, `0x18`, `0x19` |

All other object classifications return `0`.

A notification issued earlier does not itself determine this final result. In particular, this route can notify and subsequently return `0`.

## Side effects

### Direct effects

The supplied body contains no direct writes to object fields, event fields, or the additional state returned by F-fdd8589eca2d.

It performs conditional reads, invokes dependencies, and returns an integer. There is no explicit allocation, release, reference-count adjustment, or ownership transfer.

### Dependency-mediated effects

The following effects are observable as calls, but their internal consequences are unknown:

- Event preparation through F-682adfd5a561.
- Object-directed calls through F-1c60e26b3436.
- Delegated processing through F-cead90729166 and F-289632f4e99e.
- State acquisition and value queries through F-fdd8589eca2d and F-11c1b7e655b3.

No dependency is assumed to preserve the object, event, globals, or pointed-to storage. No dependency is assumed to acquire, retain, or release ownership.

Repeated reads and calls are behaviorally significant. For example, the query that triggers the second `0x16` notification is distinct from the query used to select the final result set. Their results can differ.

Likewise, a notification may change the object classification before the final classification read. The final result then follows the later value, provided all accesses remain valid.

Under a stable-event, stable-query assumption, the two conditional `0x16` notification stages cannot both match: one requires classification `8` or `0x0D`, while the other requires `6`. That mutual exclusion is not established when dependencies can mutate state.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-fdd8589eca2d | Called only on the primary-zero route; its result supplies the address for two byte flags |
| F-11c1b7e655b3 | Supplies comparison values; most calls explicitly receive the event, while one call’s argument convention is unresolved |
| F-682adfd5a561 | Called with the event and `0x0A00` before the first query on the selected primary-nonzero route |
| F-1c60e26b3436 | Called with the object and either `0x14` or `0x16`; its return value is ignored |
| F-cead90729166 | Receives object and event for selected primary-zero, secondary-zero events; its return value is forwarded |
| F-289632f4e99e | Receives object and event for primary-zero, secondary-zero classification `0x15`; its return value is forwarded |

No dependency’s detailed semantics, failure policy, or memory-management contract is established.

## Acceptance examples

These are conditional static acceptance cases, not runtime test results. Unless a case explicitly varies state, assume dependencies return normally, preserve relevant fields, and provide the stated query results.

1. **Prefilter early return.**  
   The primary flag is zero, both external flag bytes are nonzero, and the prefilter sees event classification `0x0D`. If its query returns low 32 bits `0x6E`, the result is `1`. The secondary flag is not read, and neither event delegate is called.

2. **First external flag suppresses its second read.**  
   The primary flag is zero and the external byte at `+0xA0` is zero. The external byte at `+0xA1` is not read. Dispatch proceeds using a fresh event-classification read.

3. **Delegate result is not Boolean-normalized.**  
   The primary and secondary flags are zero, the prefilter does not return early, and dispatch sees classification `7`. If F-cead90729166 returns `-3`, this function returns `-3`.

4. **Secondary flag rejects classification `0x15`.**  
   The primary flag is zero, the secondary flag is nonzero, and dispatch sees `0x15`. The result is `0`, without either delegate or an object notification.

5. **Special object classification avoids notification.**  
   The primary flag is nonzero, initial event classification is `0x15`, and object classification is `0x24`. The result is `1`, with no event preparation, value query, or notification.

6. **First-query notification shortcut.**  
   The primary flag is nonzero, initial event classification is `6`, and the first query after preparation returns `0x59`. The function issues `0x14` and returns `1`, without the later conditional stages.

7. **Classification `8` suppresses one notification but still uses the broad set.**  
   The primary flag is nonzero and the event remains classification `8`. The first query returns `0`; the next three queries return `0`, `0`, and `0x1C`. The first conditional notification is suppressed. No second notification occurs. Object classification `0x0A` returns `1` through the broad set.

8. **Notification does not imply success.**  
   The primary flag is nonzero and the event remains classification `8`. The first query and the next three queries all return `0`. The function issues `0x16`. If the final object classification is `0x24`, the result is nevertheless `0`.

9. **Narrow-set classification.**  
   The primary flag is nonzero and the event remains classification `6`. The first query returns `0`; both later queries return `0x3B`. The function issues one `0x16` notification and selects the narrow set. Object classification `0x0A` returns `0`; classification `0x18` returns `1`.

10. **Repeated queries can change the final set.**  
    Under the preceding classification-`6` conditions, let the notification-stage query return `0x3B`, but the final-selection query return `0`. A `0x16` notification still occurs, but the broad set is selected. Object classification `0x0A` then returns `1`.

## Uncertainties

The following are `documented_only`:

- The application meaning of classifications, query values, flag bytes, and notification values.
- The precise calling convention and supplied arguments for the first query after event preparation.
- Whether the query dependency truly exposes a 64-bit return contract, or whether that representation reflects recovery uncertainty. Only the observed low-32-bit comparison is required here.
- Dependency mutations, global-state effects, ownership effects, and failure behavior.
- Concurrency, reentrancy, atomicity, and synchronization guarantees.
- Complete object layouts, address validity rules, and field alignment.
- Whether delegated results are restricted to `0` and `1`.
- Whether any dependency can exit nonlocally or invalidate storage subsequently read.

Inferred intent is limited to event routing and notification. Names such as “preparation” and “query” organize observable calls; they do not establish hidden semantics.

## Evidence

Established static evidence consists solely of the supplied function representation and the supplied mapping from dependencies to opaque IDs.

That evidence supports the field offsets and widths, branch-specific call order, explicit comparison constants, repeated-read behavior, short-circuit evaluation, and return propagation described above.

No dependency implementations, external interface declarations, or runtime observations were supplied. No runtime tests have been performed.
