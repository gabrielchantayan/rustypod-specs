<!-- Trusted provenance: F-a48bd86e5ff0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a48bd86e5ff0

## Purpose

F-a48bd86e5ff0 resets a 32-bit outcome field in a caller-supplied state object, invokes a primary dependency, and selects one of three continuations:

- A primary result of −1 causes a designated external 32-bit value to be copied into the outcome field.
- Any other primary result causes one of two secondary dependencies to be called, selected by the state object’s current low-order flag bit.

The function returns the outcome field’s value after the selected continuation completes. It does not directly return the primary dependency’s result.

This is an established description of the visible control flow. Interpreting the outcome field as an error code, the primary result as a resource handle, or either flag as a named operating mode would require evidence not supplied here.

## Interface

### Inputs

The function accepts three arguments:

| Argument | Visible representation | Use |
|---|---|---|
| State reference | A 32-bit address-bearing value | Locates fields, supplies addresses to dependencies, and identifies the outcome field returned to the caller. |
| Auxiliary value A | An uninterpreted 32-bit value | Passed to either secondary dependency, but not to the primary dependency. |
| Auxiliary value B | An uninterpreted 32-bit value | Passed only to F-ccbddd3ffb1a. |

The auxiliary values are not inspected or transformed locally. Their interpretation, including whether either represents an address, is unknown.

### Referenced state layout

Offsets are byte offsets from the state reference. Each listed field is accessed as a 32-bit quantity.

| Offset | Local role | Access |
|---|---|---|
| `0x0C` | Address-bearing value used to derive the primary dependency’s first argument | Read before the primary call |
| `0x1C` | Beginning of a region whose address is passed to the primary dependency | Address formed locally; no direct local read or write of its contents |
| `0x28` | Flags word | Read before the primary call; read again after a nonfailure primary result |
| `0x38` | Outcome field | Cleared on entry, conditionally overwritten on failure, and read for the return value |

The first derived address is the value loaded from offset `0x0C`, plus `0x30` bytes. It is not the address of the state field at offset `0x0C`, and it is not the state reference plus `0x30`.

The second derived address is the state reference plus `0x1C` bytes. The supplied body establishes no size for the region beginning there.

### Output

The return value is the 32-bit value read from the outcome field at offset `0x38` at the end of normal execution.

There is no locally defined narrower result, Boolean conversion, or normalization. The same field supplies the result on every normal-return path.

### Preconditions

For defined execution of the visible accesses, the state reference must permit the required 32-bit reads and writes at the listed offsets. The derived addresses must also satisfy the selected dependencies’ requirements.

The supplied material establishes no null handling, bounds validation, alignment recovery, or pointer-overflow policy. It does not establish that a state allocation ending immediately after offset `0x3B` is sufficient: dependencies may require additional storage or access other reachable objects.

## Behavior

### Entry and initial flag selection

The first visible action is a 32-bit write of zero to the outcome field at offset `0x38`. An earlier outcome value is therefore discarded before any dependency is invoked.

The function reads the flags word at offset `0x28` and tests mask `0x00000200`. That test produces a 32-bit primary-call option:

- If the masked bit is clear, the option is `0x00000000`.
- If the masked bit is set, the option is `0x00000010`.

Other flag bits do not contribute to this option. In particular, the low-order flag bit does not affect this initial selection.

The address-bearing value at offset `0x0C` is loaded and used to form the first primary-call argument by adding `0x30` bytes. The function also forms the address at state offset `0x1C`.

The visible sequence establishes that the initial outcome reset and option selection precede the primary call. It does not establish an externally meaningful order between evaluating the primary call’s two address arguments.

### Primary operation

F-3697a58448bb is invoked once with these ordered arguments:

1. The address-bearing value loaded from state offset `0x0C`, advanced by `0x30` bytes.
2. The address at state offset `0x1C`.
3. The selected 32-bit option, either zero or `0x10`.

Its result is treated as a 32-bit integer for the failure comparison. Exactly −1, represented by the all-one 32-bit pattern `0xFFFFFFFF`, selects the failure continuation.

The body does not reject zero, other negative values, or large positive values. Every result other than −1 selects a secondary call.

### Failure continuation

If the primary result is −1:

1. A designated external 32-bit datum is read.
2. Its value is written to the outcome field at offset `0x38`.
3. Neither secondary dependency is called.
4. The outcome field is read for return.

The external datum’s numeric value and meaning are not supplied. It must not be replaced in this contract by an invented error constant.

This write occurs after the primary call and therefore supersedes any primary-call modification of the outcome field, subject to the absence of additional intervening modifications.

### Nonfailure continuation

If the primary result is not −1, the function reads the flags word at offset `0x28` again and tests mask `0x00000001`.

This is a fresh read after the primary dependency returns, not reuse of the earlier flags value. Consequently, a change to that bit during the primary call can change the selected secondary dependency without changing the option already supplied to the primary call.

If the low-order bit is clear, F-ccbddd3ffb1a is called with:

1. The state reference.
2. The primary dependency’s 32-bit result.
3. Auxiliary value A.
4. Auxiliary value B.

If the low-order bit is set, F-a202fb1cc878 is called with:

1. The state reference.
2. Auxiliary value A.

The latter dependency receives neither the primary result nor auxiliary value B through the visible argument list.

The secondary dependencies’ return values are not used by this function. After the selected secondary call returns, the function reads the outcome field and returns that value.

### Final outcome

A nonfailure primary result does not itself imply a zero return. Zero is only the initial value written to the outcome field. Dependencies may change that field.

Conversely, the failure path does not establish that the returned value is nonzero: the designated external datum could contain zero.

The stated normal-return behavior assumes that the invoked dependencies return normally and that the required memory accesses complete. No visible recovery path handles a dependency that does not return or an invalid access.

## Side effects

The local memory operations are:

- An unconditional 32-bit zero write at state offset `0x38`.
- Reads of the flags and address-bearing field needed for the primary call.
- On primary failure, a read of the designated external datum and a 32-bit overwrite at state offset `0x38`.
- On primary nonfailure, another read of the flags word.
- A final 32-bit read at state offset `0x38`.

The function locally performs no allocation, deallocation, explicit ownership transfer, or copy of the region beginning at offset `0x1C`.

These local observations do not constrain dependency behavior. In particular, the dependencies may modify state, access globals, or operate on memory reachable through their arguments. Passing an address does not establish whether its recipient borrows, retains, or releases the referenced storage.

For normal completion, the outcome field must remain readable through the final load. No synchronization or atomicity guarantee is visible.

## Dependencies

| Opaque function ID | Invocation condition | Visible contract boundary |
|---|---|---|
| F-3697a58448bb | Every invocation reaching the primary call | Receives two derived addresses and a 32-bit option; returns a 32-bit result tested against −1. |
| F-ccbddd3ffb1a | Primary result is not −1 and the post-call low-order flag bit is clear | Receives the state reference, primary result, and both auxiliary values. Its return value is ignored. |
| F-a202fb1cc878 | Primary result is not −1 and the post-call low-order flag bit is set | Receives the state reference and auxiliary value A. Its return value is ignored. |

A separate external 32-bit datum supplies the failure-path outcome. Its identity beyond that role, mutability, initialization, and relationship to dependency state are unknown.

No dependency is assumed to preserve flags, preserve the outcome field, preserve globals, or refrain from releasing memory.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not reports of executed tests.

1. **Primary failure.** Suppose the primary dependency returns −1, the designated external datum contains `0x12345678`, and no other modification intervenes. The outcome field is first cleared, then overwritten with `0x12345678`. Neither secondary dependency is called, and the function returns `0x12345678`.

2. **Both relevant flag bits clear.** Suppose the initial flags word has mask `0x200` clear, the primary result is 7, and the low-order bit remains clear. The primary option is zero. F-ccbddd3ffb1a receives the state reference, 7, auxiliary value A, and auxiliary value B. If all dependencies leave the outcome field unchanged, the return value is zero.

3. **Both relevant flag bits set.** Suppose the initial flags word contains `0x201`, the primary result is 7, and the low-order bit remains set. The primary option is `0x10`. F-a202fb1cc878 receives the state reference and auxiliary value A. The primary result is not passed to that secondary dependency.

4. **Flag change during the primary call.** Suppose the initial flags word is `0x200`, but the primary dependency sets the low-order bit before returning a nonfailure result. The primary option remains `0x10`, while the subsequent call is to F-a202fb1cc878. Selection based solely on the initial low-order bit would violate the visible behavior.

5. **Negative result other than −1.** Suppose the primary dependency returns −2. That result follows the nonfailure continuation. If the post-call low-order bit is clear, F-ccbddd3ffb1a receives the same 32-bit result.

6. **Dependency-written outcome.** Suppose the selected secondary dependency writes `0xA5A5A5A5` to the outcome field and no later modification occurs. The function returns that value regardless of the secondary dependency’s own return value.

7. **Zero-valued failure datum.** Suppose the primary result is −1 and the external datum contains zero. The function returns zero after taking the failure continuation. A zero result alone therefore does not identify which continuation ran.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of the state object, flags, primary result, and outcome field.
- The extent and format of storage beginning at offset `0x1C`.
- The validity requirements and arithmetic behavior for overflowing derived addresses.
- Dependency side effects, ownership rules, retained references, and global accesses.
- The external failure datum’s value and whether dependencies can modify it.
- Behavior under concurrent modification, aliasing that affects dependency-visible storage, invalid addresses, or nonreturning dependencies.

An inferred intent is that the outcome field communicates operation status while the primary result guides further processing. That interpretation is not necessary to the contract and is not established by the supplied body.

## Evidence

The supplied static body supports the 32-bit field accesses, byte offsets, masks, argument lists, exact −1 comparison, conditional call selection, and final outcome-field return described above.

The dependency mapping supplies the opaque function identities used here. No dependency implementation or external-datum definition was supplied.

No runtime tests have been performed. The acceptance examples specify conditional behavior to check; they are not execution evidence.
