<!-- Trusted provenance: F-9b939bfbd36e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9b939bfbd36e

## Purpose

F-9b939bfbd36e validates a requested 32-bit selection value, records an accepted value in a caller-supplied state object, performs one of two branch-specific operations, and then invokes four dependencies in a fixed order.

The distinguished value `0xFFFFFFFF` is accepted without a range check against the state object's count. Every other accepted value must have its high bit clear and be strictly less than that count.

**Inferred intent:** the distinguished value may represent “no selection,” while ordinary values identify an entry. Neither that interpretation nor the meanings of the subsequent dependency calls are established by the supplied evidence. The contract therefore uses “selection” only as a descriptive label.

The routine has two explicit return outcomes:

- `0xFFFFFFCE` for a rejected selection.
- `0x00000000` after an accepted selection and normal return from all invoked dependencies.

The success result does not independently establish that any dependency completed its intended work.

## Interface

### Inputs

The routine receives two arguments:

| Argument | Established representation | Role |
|---|---|---|
| State address | Address represented by a 32-bit integer in the supplied representation | Base address for state fields and dependency arguments. |
| Requested selection | Unsigned 32-bit value | Value to validate and, if accepted, store. |

No explicit null, alignment, lifetime, or mapped-memory checks are visible. The caller must supply memory suitable for the accesses taken by the chosen path and for any accesses performed by dependencies.

### State layout directly referenced

Offsets below are byte offsets from the supplied state address.

| Location | Width | Direct use |
|---|---:|---|
| Offset `0x14` | 32 bits | Receives the accepted selection value. |
| Offset `0x42C` | No direct access established | Its address is passed to F-f376023180bc on the ordinary-selection path. |
| Offset `0xE78` | 32 bits | Read as an unsigned count during ordinary-selection validation. |
| Externally specified offset | 16 bits | Receives zero on the distinguished-value path. |

The externally specified offset comes from a global value. Its numeric value, width, signedness, initialization, and relationship to the other fields are not established by the supplied fragment.

Passing the address at offset `0x42C` does not establish the size or layout of an object there. It also does not, by itself, establish whether the dependency reads, writes, or retains that address.

### Return value

The result is a 32-bit value:

- `0xFFFFFFCE`: explicit validation rejection. This bit pattern is −50 when interpreted as a signed 32-bit integer.
- `0x00000000`: all visible accepted-path operations reached normal completion.

There is no explicit propagation of dependency return values. The evidence does not establish a broader error-code convention or an external name for the rejection code.

## Behavior

### Validation

Validation treats `0xFFFFFFFF` as a special case before considering the ordinary range rules.

For that distinguished value:

- The value is accepted.
- The count at offset `0xE78` is not read by the visible validation logic.
- The signed-negative rejection rule does not apply.

For every other value, validation proceeds in this order:

1. Examine whether interpreting the selection as a signed 32-bit integer makes it negative.
2. If so, reject it without reading the count.
3. Otherwise, read the unsigned 32-bit count at offset `0xE78`.
4. Reject if the count is less than or equal to the selection; accept otherwise.

Consequently:

- Values from `0x80000000` through `0xFFFFFFFE` are rejected without a count read.
- Values from zero through `0x7FFFFFFF` require a count read and are accepted only when strictly below that count.
- `0xFFFFFFFF` is always accepted by this validation.

The ordinary-selection acceptance condition is therefore both a signed-range restriction and an unsigned upper-bound restriction. A large count does not make high-bit-set ordinary values valid.

### Rejection path

Upon validation rejection, the routine immediately returns `0xFFFFFFCE`.

Before that return, the visible routine performs:

- No write to the selection field.
- No distinguished-path zero write.
- No call to any listed dependency.

For rejection due to a high-bit-set, non-distinguished value, no state-object memory access is visible. For rejection due to the count comparison, the count read is the only visible state-object access.

These statements describe direct behavior of the supplied routine, not behavior of instrumentation or an external execution environment.

### Accepted-path selection write

For every accepted input, the first visible state write stores the full 32-bit requested selection at offset `0x14`.

This write precedes either branch-specific operation and all four common dependency calls.

The routine does not compare the requested selection with the previously stored selection. An accepted value is processed even if it is already present. There is no visible unchanged-selection shortcut.

### Distinguished-value branch

After storing `0xFFFFFFFF` at offset `0x14`, the routine obtains an offset from an external global value and writes a 16-bit zero at the state address plus that offset.

This establishes a two-byte zero write, not a whole-object reset or a wider-field clear.

The global offset is consulted after the selection-field write. That ordering matters if the relevant storage aliases or is externally mutable. The fragment supplies no aliasing or synchronization guarantees.

F-f376023180bc is not called on this branch.

### Ordinary-selection branch

After storing an accepted ordinary selection, the routine calls F-f376023180bc with three arguments, in order:

1. The supplied state address.
2. The requested selection.
3. The address at state offset `0x42C`.

The visible routine does not directly initialize the memory at offset `0x42C` before this call. It does not inspect a return value from the dependency.

No further purpose or effect of this dependency is established.

### Common dependency sequence

Once the branch-specific operation completes normally, both accepted paths invoke:

1. F-41899b1592e0 with the state address.
2. F-3ea7a339f5ec with the state address.
3. F-a96742fec78e with the state address.
4. F-087700124fe9 with the state address.

The calls occur in that order. Their return values do not participate in any visible decision.

After the final dependency returns normally, the routine returns zero. If any dependency does not return normally, the supplied evidence establishes neither the remaining calls nor a final return value.

The selection write occurs before these calls, but its persistence through them is unknown. A dependency may alter state or globals; no preservation guarantee is supplied.

## Side effects

### Direct memory effects

On a normally completed accepted ordinary-selection path, the routine directly:

- Reads the 32-bit count at offset `0xE78`.
- Writes the 32-bit selection at offset `0x14`.

On a normally completed distinguished-value path, it directly:

- Writes `0xFFFFFFFF` at offset `0x14`.
- Reads the externally supplied offset.
- Writes a 16-bit zero at the computed location.

The distinguished path does not visibly read the count.

These are access widths and ordering guarantees from the supplied representation. They do not establish atomicity, physical bus transactions, byte order, or inter-thread synchronization.

### Indirect effects

An accepted ordinary selection invokes five dependencies; the distinguished value invokes four.

Dependency effects are unknown. They may include additional state accesses, global changes, resource operations, or other externally observable behavior. None may be assumed absent merely because its return value is unused.

### Ownership and lifetime

No allocation, deallocation, copying of an entire state object, or explicit ownership transfer is visible in this routine.

That observation does not establish that dependencies preserve ownership or retain no pointers. In particular, both the state address and the derived address at offset `0x42C` cross dependency boundaries.

The caller must keep the necessary storage valid for the accesses actually performed. Whether that requirement extends beyond this routine's return depends on undocumented dependency behavior.

### Failure and rollback

There is no visible rollback after acceptance. The selection field is written before branch-specific processing and before the common dependency sequence.

If a later access fails or a dependency exits abnormally, the fragment provides no recovery path and no guarantee of restoration. It also does not define a returned error for invalid memory.

## Dependencies

| Opaque function ID | Invocation condition | Visible arguments | Established role |
|---|---|---|---|
| F-f376023180bc | Accepted ordinary selection only | State address; selection; state address plus `0x42C` | Branch-specific operation before the common sequence. |
| F-41899b1592e0 | Every accepted path after its branch-specific operation | State address | First common call. |
| F-3ea7a339f5ec | After F-41899b1592e0 returns normally | State address | Second common call. |
| F-a96742fec78e | After F-3ea7a339f5ec returns normally | State address | Third common call. |
| F-087700124fe9 | After F-a96742fec78e returns normally | State address | Fourth and final common call. |

The routine also depends on a global offset for the distinguished-path zero write.

No dependency contract was supplied. Their argument interpretation, memory ownership, global effects, error behavior, and normal-return guarantees are **documented_only** unknowns.

## Acceptance examples

The following are conditional contract examples derived from static evidence, not observed executions. Successful completion assumes valid accessed memory and normal return from every invoked dependency.

| Selection | Count at `0xE78` | Required visible outcome |
|---|---:|---|
| `0` | `0` | Reject with `0xFFFFFFCE`; no direct write or dependency call. |
| `0` | `1` | Accept; store zero; call F-f376023180bc; perform the four common calls; return zero. |
| `4` | `5` | Accept because the selection is strictly below the count. |
| `5` | `5` | Reject because equality is outside the accepted range. |
| `0x7FFFFFFF` | `0x80000000` | Accept; the selection is nonnegative under signed 32-bit interpretation and below the unsigned count. |
| `0x7FFFFFFF` | `0x7FFFFFFF` | Reject on equality. |
| `0x80000000` | Any value | Reject before reading the count. |
| `0xFFFFFFFE` | Any value | Reject before reading the count. |
| `0xFFFFFFFF` | Any value | Accept without reading the count; store the distinguished value; perform the two-byte zero write; make only the four common calls; return zero. |

Additional ordering and boundary examples:

- **Repeated accepted selection:** if the state already contains the requested accepted value at offset `0x14`, the routine still stores it and performs the applicable calls.
- **Unavailable count memory:** with selection `0xFFFFFFFF`, the visible routine does not require a count read. Dependencies may still access that location; their behavior is unknown.
- **Overlapping zero-write destination:** if the external offset directs the distinguished-path zero write into bytes of the selection field, that zero write follows the full-width selection write and may overwrite part of it. The fragment does not establish whether such an offset is permitted in the intended environment.
- **Dependency mutation:** if F-41899b1592e0 changes state before returning, F-3ea7a339f5ec is still invoked next with the same supplied state address. The routine performs no intervening visible restoration or revalidation.

## Uncertainties

### Established static evidence

The fragment establishes the 32-bit selection rules, explicit result patterns, direct access widths, fixed field offsets, branch distinction, dependency arguments, and call order.

It also establishes that the routine does not visibly use dependency results or revalidate the selection after writing it.

### Inferred intent

The distinguished value may mean “no selection,” and the zeroed field may be associated with clearing selection-dependent state. The common calls may refresh related state. These interpretations are not acceptance requirements.

### Documented-only unknowns

The following remain unresolved:

- The complete state-object layout and required allocation size.
- The numeric global offset and its representation.
- Alignment, byte order, and concurrency requirements.
- Aliasing among the state object, global-offset storage, and dependency-managed memory.
- The valid extent and layout of storage beginning at offset `0x42C`.
- Dependency mutations, ownership rules, retained references, and failure mechanisms.
- Whether application-level invariants impose additional restrictions on the count.
- Address-arithmetic overflow and invalid-address behavior.
- Whether rejection code −50 has a named meaning elsewhere.

No guarantee of unchanged globals, released memory, retained memory, or final state preservation is inferred from an unknown callee.

## Evidence

This contract is based solely on the supplied static function fragment and opaque dependency mapping.

The evidence directly shows a short-circuit validation condition, a 32-bit accepted-selection write, a distinguished-path 16-bit zero write through an externally supplied offset, one ordinary-path dependency call, four ordered common calls, and two explicit return constants.

No dependency bodies, complete object definition, global initializer, external API documentation, or execution traces were supplied. No runtime tests have been performed. The acceptance examples express conditional behavior supported by the fragment; they are not test results.
