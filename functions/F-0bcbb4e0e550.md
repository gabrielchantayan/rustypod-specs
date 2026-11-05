<!-- Trusted provenance: F-0bcbb4e0e550; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0bcbb4e0e550

## Purpose

F-0bcbb4e0e550 conditionally forwards a selector and a stored 32-bit value to F-7c96094455b3, then writes a one-byte marker through a freshly read value from the same storage location.

Selectors 0 through 6 are supported. Selectors 0 through 5 produce a marker of 0; selector 6 produces a marker of 1. Every other selector causes an immediate return without a dependency call or an explicit data-memory read or write by this function.

The important sequencing property is that the value used for the marker destination is read **after** the dependency returns. It is not necessarily the value passed to the dependency.

These are established structural behaviors. Interpreting the selector as an operating mode or the marker as an enabled flag is plausible intent, not an established semantic contract.

## Interface

### Inputs

The function takes two inputs:

| Input | Width and interpretation | Use |
|---|---|---|
| Container address | Represented as a 32-bit integer in the supplied material | Establishes the address of a four-byte slot at byte offset `0x30` |
| Selector | Four-byte value | Selects cases 0 through 6 or the unsupported-selector return path |

The selector is matched as supplied. The function does not visibly narrow it to one byte, mask it, clamp it, or convert an unsupported value to a supported one. Signedness is not established, but does not change the supported set: only the full-width values 0, 1, 2, 3, 4, 5, and 6 match.

### Relevant layout

Only two storage locations are directly evidenced:

| Location | Access width | Meaning established here |
|---|---|---|
| Container address plus `0x30` | Four bytes | Read before the dependency call as its first argument; read again after the call as the base for the marker destination |
| Post-call base plus `0xE3C` | One byte | Written with 0 or 1 according to the original selector |

Offsets are byte offsets. They are not scaled by the width of the accessed value.

No complete container or target-object layout is established. In particular, the evidence does not establish field names, allocation sizes, object types, or the contents of neighboring storage.

### Output

There is no specified return value. Completion is observable through the dependency invocation and final byte write, or through the absence of those actions for an unsupported selector.

No success code, failure code, previous marker value, or dependency result is returned.

### Preconditions

For a supported selector, defined completion requires:

- The four-byte slot at container address plus `0x30` to be readable when initially accessed.
- The dependency invocation to be valid under F-7c96094455b3’s own contract.
- The same slot to remain readable after that dependency returns.
- The post-call slot value to identify a base for which the byte at offset `0xE3C` is writable.
- The address calculations and accesses to be valid in the execution environment.

The function does not check these conditions.

For an unsupported selector, this function does not dereference the container address. Its supported-path memory requirements therefore do not apply to that path.

## Behavior

### Selector dispatch

The function first distinguishes the seven supported selector values from all other four-byte values.

An unsupported selector terminates processing immediately. There is no fallback dependency invocation, marker normalization, or object inspection.

For every supported selector, exactly one invocation of F-7c96094455b3 is explicitly present on the normal path.

### Pre-call read and dependency invocation

For selectors 0 through 6, the function reads four bytes from container address plus `0x30`.

That value becomes the first argument to F-7c96094455b3. The second argument is a four-byte value equal to the selected case:

| Original selector | Dependency second argument |
|---|---|
| 0 | 0 |
| 1 | 1 |
| 2 | 2 |
| 3 | 3 |
| 4 | 4 |
| 5 | 5 |
| 6 | 6 |

There is no evidence of a transformation of the first argument beyond reading and passing its four-byte value.

No marker write by F-0bcbb4e0e550 occurs before the dependency call. The dependency may perform its own writes, including writes to overlapping storage; those effects are not specified by this function.

### Post-call read

After F-7c96094455b3 returns normally, the function reads the four-byte slot at container address plus `0x30` again.

This second access is behaviorally significant. The dependency may have changed the slot, directly or indirectly. The final write uses the value obtained by this second read, whether or not it equals the value supplied as the dependency’s first argument.

Thus:

- An unchanged slot directs the final write relative to the same base value passed to the dependency.
- A changed slot redirects the final write relative to the replacement base.
- A post-call invalid slot or invalid replacement base is not handled by a fallback to the earlier value.

The evidence does not establish that the slot is stable during the call or that any related global state is preserved.

### Final marker write

After the second read, the function writes exactly one byte at post-call base plus `0xE3C`.

The byte value is:

- 0 for selectors 0, 1, 2, 3, 4, and 5.
- 1 for selector 6.

The marker value depends on the original selector, not on a returned status from the dependency. No dependency result is inspected.

The function does not explicitly read the old marker value. The assignment is not a bit update: it replaces the entire addressed byte with 0 or 1. Neighboring bytes are not explicitly written by this function.

After this write, the function returns.

### Widths and address arithmetic

The slot reads are four bytes each; the final store is one byte. The selector values and both dependency arguments are four-byte quantities in the supplied representation.

The container address and post-call base appear in a 32-bit address-like representation. The evidence does not establish a wider-pointer interface or authorize widening the slot to a native pointer width on another platform.

The additions of `0x30` and `0xE3C` are address calculations. Their behavior on overflow, nonrepresentable addresses, or invalid address mappings is not independently established. Valid, representable accesses are preconditions rather than checked cases. No portable wraparound behavior is promised.

## Side effects

On a supported path that completes normally, the directly established sequence is:

1. Read the four-byte container slot.
2. Invoke F-7c96094455b3 with that value and the supported selector.
3. Read the four-byte container slot again.
4. Write one byte through the newly read base.

This ordering must be preserved wherever dependency-visible mutations could distinguish the two reads.

For unsupported selectors, there is no explicit data-memory access or dependency invocation by this function.

The function contains no explicit allocation, deallocation, ownership transfer, or reference-count operation. Passing the stored value to the dependency does not establish whether the dependency borrows, retains, replaces, or releases any associated resource.

There is no explicit error handling, rollback, retry, or cleanup. If the dependency does not return normally, the subsequent read and marker write are not reached through the shown normal flow. If a later access fails, no compensating action is shown for effects the dependency has already performed.

No synchronization or atomic-access guarantee is established. The presence of separate reads does not define behavior under unsynchronized concurrent mutation.

## Dependencies

### F-7c96094455b3

For each supported selector, F-0bcbb4e0e550 calls this dependency once with:

1. The four-byte value initially read from container offset `0x30`.
2. The four-byte supported selector.

The caller does not use a dependency return value.

The dependency’s implementation and independent contract are unavailable. The following remain unknown:

- Whether it changes the container slot or the eventual marker.
- Whether it modifies globals or other reachable state.
- Whether it allocates, retains, transfers, or releases memory.
- Whether every supplied value is valid for it.
- Whether it always returns normally.

None of these properties may be inferred from the caller’s lack of checks. In particular, the caller’s second slot read must not be replaced by an assumption that the dependency preserves the first value.

## Acceptance examples

These are conditional behavioral examples derived from static evidence, not executed tests.

### Supported selector with stable storage

Given selector 3, a readable container slot containing a valid target base, and a dependency that returns without changing that slot:

- The dependency receives the original slot value and selector 3.
- The slot is read again after the call.
- The byte at that base plus `0xE3C` becomes 0.

An earlier nonzero marker does not prevent or alter the write.

### Selector 6

Given selector 6 and valid accesses throughout:

- The dependency receives the initially stored value and selector 6.
- The final marker write is one byte with value 1.
- The dependency call precedes both the second slot read and that write.

Selector 6 does not bypass the dependency.

### Dependency replaces the stored base

Given selector 2, an initial slot value identifying target A, and a dependency that replaces the slot with a valid base for target B:

- The dependency receives A’s base.
- The final marker write sets B’s byte at offset `0xE3C` to 0.
- This function does not explicitly write A’s marker unless A and B’s marker locations coincide.

### Dependency changes the eventual marker

Given selector 6 and a dependency that writes another value to the byte eventually targeted, then returns normally:

- The subsequent explicit write replaces that byte with 1.
- The dependency’s earlier value does not determine the final marker.

This assumes no intervening external mutation.

### Unsupported full-width value

Given selector 7, or selector `0x00000100`:

- The function returns without invoking the dependency.
- It does not read the container slot.
- It does not write a marker.

The latter value is not accepted merely because its low byte is zero.

### Invalid post-call target

Given a supported selector and a dependency that leaves an unusable base in the slot, successful completion is not established. The function neither validates the replacement nor falls back to the original base. The precise resulting failure is environment-dependent.

## Uncertainties

The following are **documented_only**:

- Semantic names and purposes of the container, referenced target, selector, and marker.
- Complete object layouts, allocation bounds, alignment requirements, and byte order.
- Exact source-level signedness and pointer types.
- Address-overflow and invalid-access behavior in the intended runtime.
- Dependency effects, ownership rules, failure modes, and global-state preservation.
- Concurrency guarantees and behavior under external mutation.

Inferred intent is limited to a selector-dependent operation followed by recording a binary marker. More specific interpretations are not supported.

## Evidence

Established behavior comes from the supplied static function representation: seven supported cases, an immediate unsupported-case return, one dependency call per supported path, two four-byte reads of the same container-relative slot separated by that call, and a final one-byte store selected by whether the original selector is 6.

No dependency body, surrounding caller contract, runtime trace, or complete data-type definition was supplied. No runtime tests have been performed.
