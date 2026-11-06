<!-- Trusted provenance: F-b22d3ab67c5e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b22d3ab67c5e

## Purpose

F-b22d3ab67c5e performs an ordered update of a caller-supplied object. It invokes two preliminary dependencies, refreshes three stored 32-bit values through indirect references, applies two control values, and performs a final dependency-mediated update on a selected subordinate object.

The established behavior is the sequence of memory accesses and dependency calls described below. The meanings of the stored values and subordinate objects are not established. In particular, interpreting the operation as rendering, animation, visibility management, or synchronization would require evidence beyond the supplied source.

On normal completion, the function returns the 32-bit value zero. This is a fixed completion result, not an established report that every dependency succeeded.

## Interface

### Input

The function accepts one argument: an address identifying a mutable object, called the **root object** in this contract.

The supplied representation uses 32-bit address-bearing values and 32-bit word accesses. This contract describes that target representation; it does not imply portability to an environment with different pointer widths.

The following root-relative fields are accessed directly:

| Offset | Access width | Role established by use |
|---|---:|---|
| `0x18` | 32 bits | Value captured for a conditional dependency call |
| `0x1c` | 32 bits | Second argument to F-1d891874186d |
| `0x20` | 32 bits | First argument to F-460b2abbbbbe and F-1d891874186d |
| `0x24` | 32 bits | First argument to a second F-460b2abbbbbe call; later reread as the selected subordinate address |
| `0x48` | 8 bits | Zero/nonzero selector |
| `0x50` | 32 bits | Address used for the first indirect copy |
| `0x54` | 32 bits | Address used for the second indirect copy |
| `0x58` | 32 bits | Address used for the third indirect copy |
| `0x5c` | 32 bits | Destination of the first copy |
| `0x60` | 32 bits | Destination of the second copy |
| `0x64` | 32 bits | Destination of the third copy |

For each indirect copy, the function reads a 32-bit word at offset `0x04` from the address obtained from the corresponding root field.

The selected subordinate object has two additional relevant locations:

- An 8-bit field at offset `0x0b`, compared with the exact value `3`.
- A location at offset `0x78`, whose address is passed to two dependencies.

These offsets identify accesses, not complete object definitions. Neither the complete root layout nor the size and layout of the subordinate object is established.

### Output

The function returns a 32-bit zero if execution reaches its return.

It does not directly return any of the copied or dependency-produced values. Those values affect memory or are supplied to subsequent dependency calls.

### Preconditions

For defined normal execution:

- Each directly accessed field must be readable or writable as required when its access occurs.
- Each address used by an indirect copy must permit a 32-bit read at offset `0x04`.
- The selected subordinate address must permit the byte read at offset `0x0b`.
- All dependency-specific preconditions must hold for the arguments supplied, including the address formed at subordinate offset `0x78`.
- Address arithmetic and alignment must be valid for the target environment.

These requirements apply at the time of each access. Earlier dependencies may change memory used by later accesses. Initial validity alone is therefore insufficient if a dependency invalidates a later address.

There are no visible null checks, bounds checks, allocation checks, or checks for address-arithmetic overflow.

## Behavior

### 1. Preliminary calls

The function first calls F-cc4baf29ce3e with no explicit arguments.

After that call returns, it calls F-3fd332ba9340 with the root address.

No root fields are visibly read by this function before those two calls. Either dependency may affect the values subsequently observed; no preservation of root fields or global state is established.

### 2. Three ordered indirect copies

The function performs the following copies in order:

| Order | Address read from root | Source word location | Root destination |
|---:|---|---|---|
| 1 | `0x50` | Referenced address plus `0x04` | `0x5c` |
| 2 | `0x54` | Referenced address plus `0x04` | `0x60` |
| 3 | `0x58` | Referenced address plus `0x04` | `0x64` |

Each copy transfers one 32-bit word without visible conversion or arithmetic on its contents.

For each row, the address field and source word are read before that row’s destination is written. The first destination write precedes the second copy, and the second destination write precedes the third copy.

These are sequential copies, not an atomic snapshot. If a destination overlaps a later address field or source word, the later copy observes the effect of the earlier write. The function provides no aliasing guard or temporary snapshot of all three source values.

### 3. Two control-value calls

The function reads the 32-bit field at root offset `0x20` and calls F-460b2abbbbbe with:

1. That field value.
2. The value `255`.

After this call returns, the function reads the byte at root offset `0x48`. It selects a second control value:

- Byte equal to zero: `0`.
- Any nonzero byte: `255`.

It then calls F-460b2abbbbbe again, using the 32-bit field at root offset `0x24` as the first argument and the selected value as the second argument.

The byte is tested for zero, not specifically for one. A stored byte such as `2` or `255` selects `255`.

The first call occurs before the selector byte is read. Consequently, changes made by that first call to the selector or the second target field can influence the second call.

### 4. Root-field pairing call

The function calls F-1d891874186d with:

1. The 32-bit field at root offset `0x20`.
2. The 32-bit field at root offset `0x1c`.

These are fresh reads after both preceding control-value calls. The value at offset `0x20` need not match the value previously supplied to F-460b2abbbbbe.

The source establishes the argument positions but not a portable ordering between the two field reads used to prepare this call.

### 5. Capture of the final operation’s inputs

After F-1d891874186d returns, the function:

1. Reads and retains the 32-bit value at root offset `0x18`.
2. Reads and retains the 32-bit value at root offset `0x24` as the selected subordinate address.

This is another fresh read of offset `0x24`. It is not required to equal the earlier value supplied to F-460b2abbbbbe.

The retained subordinate address is used throughout the remaining sequence. Later changes to the root’s `0x24` field do not redirect these calls.

### 6. Bracketed subordinate operation

The function calls F-338d16f9ce72 with the selected subordinate address plus `0x78`.

Only after that call returns does it read the subordinate byte at offset `0x0b`.

If that byte equals exactly `3`, the function calls F-40f10dd4a12e with:

1. The retained subordinate address.
2. The previously captured root value from offset `0x18`.

If the byte has any other value, that call is skipped.

The decision uses the subordinate byte as it exists after F-338d16f9ce72. The second argument, however, remains the earlier captured value; it is not reread from the root after that dependency runs.

### 7. Dependency-produced values and finalization

The function next calls F-9afb51fb96af with five arguments:

1. The retained subordinate address.
2. The address of a temporary region containing two 32-bit words.
3. The address of a separate temporary 32-bit word.
4. The address of another separate temporary 32-bit word.
5. The address of a final separate temporary 32-bit word.

No initialization of these temporary words is visible before this call.

After the call returns, F-b22d3ab67c5e reads the first word of the two-word region and each of the three separate words. It calls F-e2303c7cb41c with the retained subordinate address followed by those four values, preserving their corresponding argument order.

The second word of the two-word region is not directly consumed by this function afterward. Its role inside F-9afb51fb96af is unknown.

The function then calls F-1333331b2da1 with the retained subordinate address plus `0x78`.

Finally, it returns zero.

## Side effects

### Direct memory writes

The only established direct writes to caller-addressed memory are the three 32-bit writes at root offsets `0x5c`, `0x60`, and `0x64`.

Those writes occur before either call to F-460b2abbbbbe. They are not rolled back by any visible mechanism if a later operation fails.

### Dependency-mediated effects

All called dependencies may have effects not described by this function’s source. This contract does not assume that they:

- Preserve globals or root fields.
- Leave referenced objects unchanged.
- Acquire or release locks.
- Allocate or release memory.
- Retain, copy, or discard supplied addresses.
- Report failures through their return values.

In particular, the calls involving subordinate offset `0x78` surround other work, but that ordering alone does not establish lock acquisition and release semantics.

### Ownership and lifetime

The function contains no explicit allocation or deallocation.

It supplies addresses of temporary storage to F-9afb51fb96af. That storage is local to this invocation and is not available after the invocation ends. The source does not establish whether the dependency retains those addresses; any retention requiring a longer lifetime would need an external contract.

No ownership transfer of the root, subordinate object, referenced sources, or temporary storage is established.

### Failure and interruption

No dependency result is inspected for an error condition. There is no visible retry, rollback, or exceptional cleanup path.

If a dependency does not return normally, subsequent steps are not guaranteed to occur. In particular, the call to F-1333331b2da1 is not protected by a visible cleanup mechanism.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-cc4baf29ce3e | First call; no explicit arguments |
| F-3fd332ba9340 | Second call; receives the root address |
| F-460b2abbbbbe | Called twice with a root-derived target and a control value |
| F-1d891874186d | Receives root fields `0x20` and `0x1c` |
| F-338d16f9ce72 | Receives subordinate address plus `0x78` before the conditional byte read |
| F-40f10dd4a12e | Called only when subordinate byte `0x0b` equals `3` |
| F-9afb51fb96af | Receives the subordinate address and four temporary-storage addresses |
| F-e2303c7cb41c | Receives the subordinate address and four values read after F-9afb51fb96af |
| F-1333331b2da1 | Receives subordinate address plus `0x78` after F-e2303c7cb41c |

The names and argument shapes establish call relationships only. Semantic roles beyond those listed remain unverified.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reports of executed tests.

### Ordinary distinct-source copies

Assume all relevant addresses remain valid, the preliminary dependencies return normally, and the three source words contain `0x11223344`, `0x55667788`, and `0x99aabbcc`.

Immediately after the copy sequence, root offsets `0x5c`, `0x60`, and `0x64` must contain those respective words. Later dependencies may subsequently change them.

### Zero and nonzero selector

Assume the byte at root offset `0x48` is zero when read. The second F-460b2abbbbbe call must receive `0` as its second argument.

If the byte is instead `7`, that argument must be `255`. The first F-460b2abbbbbe call receives `255` in either case.

### Selector changed by the first control call

Assume the selector starts at zero but the first F-460b2abbbbbe call changes it to a nonzero byte before returning.

The second call must receive `255`, because the selector read occurs afterward.

### Exact subordinate-type condition

Assume the subordinate byte is `3` after F-338d16f9ce72 returns. F-40f10dd4a12e must be called once at that point.

If the byte is `2` or `4`, that call must be skipped. F-9afb51fb96af and the remaining normal-return sequence still follow.

### Captured value remains stable

Assume root offset `0x18` contains value A when captured, and F-338d16f9ce72 later changes that field to B.

If the subordinate byte selects the conditional call, F-40f10dd4a12e must receive A, not B.

### Sequential aliasing

Assume the second copy’s source address plus `0x04` equals root offset `0x5c`, with all other accesses valid.

The second copy must read the value already written by the first copy, rather than the old contents of `0x5c`.

### Temporary-value propagation

Assume F-9afb51fb96af supplies defined values A, B, C, and D in the four temporary words subsequently consumed.

F-e2303c7cb41c must receive the retained subordinate address followed by A, B, C, and D in that order. No visible transformation is applied.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of every root field and subordinate field.
- The full layouts, alignment requirements, and valid address ranges of the involved objects.
- Dependency side effects, failure behavior, ownership rules, and global-state effects.
- Whether the two calls using subordinate offset `0x78` implement synchronization or another protocol.
- Which temporary locations F-9afb51fb96af initializes and why its first output region contains two words.
- Behavior when a subsequently consumed temporary word has not been initialized by the dependency.
- Behavior for invalid addresses, arithmetic wraparound, or concurrent mutation.
- Any externally enforced restrictions on aliasing.

An inferred intent is that F-9afb51fb96af produces values for F-e2303c7cb41c. The established fact is only that temporary addresses are passed to the former and corresponding words are read for the latter.

## Evidence

This contract is based solely on static inspection of the supplied function representation and its opaque dependency mapping.

Established evidence includes the explicit access widths and offsets, direct writes, call order, constants, branch comparisons, captured values, and fixed return value.

Dependency implementations and external type definitions were not supplied. No runtime tests have been performed. The acceptance examples specify conditional expected behavior; they do not claim observed execution results.
