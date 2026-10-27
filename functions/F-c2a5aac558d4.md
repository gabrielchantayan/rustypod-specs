<!-- Trusted provenance: F-c2a5aac558d4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c2a5aac558d4

## Purpose

F-c2a5aac558d4 selects an address-like result through one of two paths and publishes that result through a caller-provided output location.

A nonzero leading value in the input object selects direct address calculation. A zero leading value selects delegation to F-5e2345d8f7b1. The direct path returns a zero status word; the delegated path returns the dependency’s status word unchanged.

The function does not read the contents at the selected address. Its established role is address selection and publication, not element retrieval, validation, allocation, or release.

Interpreting the leading value as a count and the calculated address as the last entry of a collection is plausible intent, not an established semantic guarantee.

## Interface

The function accepts four arguments and returns one 32-bit word.

| Argument | Contract |
|---|---|
| Input object address | Identifies storage whose first signed integer word is read unconditionally. On the nonzero path, a pointer-bearing word at byte offset 8 is also read under the represented four-byte-word layout. |
| Opaque auxiliary word | A 32-bit value forwarded unchanged to F-5e2345d8f7b1 on the zero path. It is otherwise unused by this function. |
| Output location | Identifies storage receiving one 32-bit address representation before normal return. This write occurs on both paths. |
| Fallback address | Supplies the initial address value for the delegated path. This function does not dereference it. It has no effect on the direct path. |

The input’s leading value is treated as a signed 32-bit integer in the supplied representation. The return value and auxiliary argument are four-byte quantities. The output write stores an integer representation of the selected pointer, rather than writing through that pointer.

The supplied representation is consistent with four-byte integer words and a 32-bit address model. The pointer width is not independently established by an ABI description. A lossless interpretation of the output as a complete address therefore requires that address model, or another documented guarantee that the selected pointer fits in the output word.

### Input layout

Only these input locations are explicitly accessed:

| Byte offset | Meaning established by use |
|---|---|
| 0 | Signed discriminator used both for path selection and direct-path address calculation. |
| 8 | Pointer-bearing value used as the direct-path base, read only when the discriminator is nonzero. |

No read at byte offset 4 is established. The representation does not establish the complete object size, the meanings of other fields, or a public structure definition.

### Preconditions

For defined operation:

- The input address must support the initial integer-word read.
- If the discriminator is nonzero, the pointer-bearing word at offset 8 must be readable.
- The output location must support the final four-byte write when that write occurs.
- The direct-path arithmetic and pointer representation must be valid under the actual target’s execution rules.
- On the zero path, the dependency’s own requirements must also hold.

There is no explicit null check, bounds check, alignment repair, or output-pointer check. Invalid access is not converted into a returned error by any visible logic.

## Behavior

### Initial observation and path selection

The function first reads the input object’s leading integer word. That single observed value determines the path.

A value of zero selects delegation. Every nonzero value selects direct address calculation, including negative values. There is no separate positive-count condition.

The visible logic does not reread the discriminator. Subsequent changes to the input storage do not cause path selection to be reconsidered.

### Direct address calculation

For a nonzero discriminator, the function reads the pointer-bearing word at input offset 8 and uses it as the base address.

Let:

- \(n\) be the signed discriminator observed initially.
- \(B\) be the base address read from offset 8.

With four-byte integer words, the mathematical byte displacement is:

\[
16n - 16
\]

Thus, where the arithmetic is representable and the address operation is valid, the selected address is:

\[
B + 16(n - 1)
\]

The factor of 16 matters. The represented calculation advances in four-byte units after forming an integer displacement that itself includes multiplication by four. It is not merely a four-byte stride per discriminator increment.

The function then:

1. Writes the selected address’s integer representation to the output location.
2. Returns the 32-bit value zero.

F-5e2345d8f7b1 is not called on this path. The auxiliary word and fallback address do not affect the result.

The selected address is not dereferenced. No evidence establishes that it identifies a valid element, that any element occupies 16 bytes, or that the base has enough associated storage for the displacement.

The formula above is a mathematical description for ordinary, valid cases. It is not a guarantee of wraparound for extreme values. Signed intermediate arithmetic, pointer arithmetic, and pointer-to-integer conversion require target-specific rules not supplied here.

### Delegated selection

For a zero discriminator, the function initializes a temporary pointer-sized result slot with the fallback address. It then calls F-5e2345d8f7b1 with three arguments, in this order:

1. The original input object address.
2. The opaque auxiliary word, unchanged.
3. The address of the temporary result slot.

The dependency therefore receives access to the slot holding the fallback address, not the caller’s output location directly.

If the dependency returns normally, this function takes the slot’s resulting pointer value and writes its integer representation to the caller’s output location. It then returns the dependency’s 32-bit return value unchanged.

There is no status-dependent branch after the call. In particular, the function does not suppress the output write when the returned word is nonzero.

If the dependency leaves the temporary slot unchanged, the published value is the fallback address’s representation. That outcome is conditional on dependency behavior; it is not a preservation guarantee supplied by this function.

### Observable ordering and aliasing

The visible access order is:

- Read the leading input word.
- On the nonzero path, read the base word at offset 8, then write the output.
- On the zero path, initialize the temporary slot, call the dependency, then write the output after normal return.

There is no explicit read from the output location before the write.

Aliasing is not prohibited by the visible logic. On the direct path, if the output location overlaps the discriminator or base storage, the necessary input reads precede the final write. The write can consequently modify the input object without changing the address already calculated.

On the delegated path, the final output write occurs after the dependency returns. If the output aliases storage changed by the dependency, this write may overwrite that changed storage. The dependency’s internal access order is unknown.

## Side effects

The function’s explicit externally visible write is the final four-byte store through the output argument. The output’s prior contents are not preserved.

The function also maintains a temporary result slot. On the delegated path, its address is exposed to F-5e2345d8f7b1 for the duration of the call. That exposure does not establish permission to retain the slot’s address after this invocation returns.

No explicit allocation, deallocation, input-field update, logging, synchronization, or global-variable access appears in the supplied body. The final output write can nevertheless modify the input object when arguments alias.

The dependency may have additional side effects. Its implementation and contract are unavailable, so this function cannot be specified as globally side-effect-free, allocation-free, or ownership-neutral on the delegated path.

No ownership transfer is established for the input object, fallback address, calculated address, or published address. Returning or publishing an address does not prove that it is newly allocated, borrowed, retained, or safe to release.

## Dependencies

### F-5e2345d8f7b1

This dependency is called exactly once on the zero-discriminator path and is not called on the nonzero path.

Its visible interface at this call site consists of:

- The original input object address.
- An unchanged opaque 32-bit word.
- A pointer to a temporary slot initialized with the fallback address.
- A 32-bit return word.

The caller uses two results independently:

- The return word becomes F-c2a5aac558d4’s return value.
- The temporary slot’s post-call value becomes the published output.

The supplied evidence does not establish whether the dependency:

- Interprets zero or nonzero return words as success.
- Always writes the temporary slot.
- Validates or changes the input object.
- Allocates or releases storage.
- Modifies global state.
- Retains any passed address.
- Returns normally for all valid inputs.

None of those properties should be inferred from its use here.

## Acceptance examples

These examples are conditional on valid storage, representable arithmetic, and the applicable address model. They describe expected behavior, not executed tests.

| Condition | Required observable result |
|---|---|
| Discriminator is 1 and the base word represents \(B\). | Publish \(B\); return zero; do not call the dependency. |
| Discriminator is 2 and the base word represents \(B\). | Publish \(B + 16\) bytes; return zero; do not call the dependency. |
| Discriminator is 3 and the base word represents \(B\). | Publish \(B + 32\) bytes; return zero; do not call the dependency. |
| Discriminator is −1 and the resulting backward address calculation is valid. | Take the direct path and publish \(B - 32\) bytes; return zero. Negative values do not select delegation. |
| Discriminator is zero; fallback represents \(P\); the dependency leaves its result slot unchanged and returns \(R\). | Publish \(P\)’s integer representation and return \(R\). |
| Discriminator is zero; the dependency replaces the result slot with \(Q\) and returns \(R\). | Publish \(Q\)’s integer representation and return \(R\). |
| The dependency returns a nonzero word normally. | Still perform the output write; propagate that return word unchanged. |
| The direct path’s output location aliases the input discriminator. | Read the discriminator and base before overwriting the aliased storage with the selected address representation. |

A zero discriminator does not require this function itself to read the base word at offset 8. This does not establish that the dependency avoids that location.

An unreadable selected address does not itself trigger a dereference in this function. Whether forming that address is valid remains a separate pointer-arithmetic question.

## Uncertainties

The following are **documented_only** because the supplied evidence does not resolve them:

- **Application meaning:** The discriminator may represent a count, but only its numerical role is established.
- **Storage organization:** The 16-byte stride suggests a layout relationship; it does not establish a record type or allocation extent.
- **ABI details:** Pointer width, alignment requirements, and exact pointer-to-integer conversion rules lack independent documentation.
- **Overflow behavior:** Extreme discriminator values may exceed signed intermediate ranges. Neither defined wrapping nor a checked error response is established.
- **Pointer bounds:** The validity of calculated addresses, including backward calculations, depends on storage and execution rules not shown.
- **Status semantics:** The meaning of the dependency’s return word is unknown.
- **Ownership and lifetime:** No release obligations or usable lifetime for published addresses are established.
- **Dependency side effects:** Global state preservation, input preservation, allocation behavior, and exceptional control flow are unknown.
- **Concurrency:** No synchronization or atomicity contract is established.

These unknowns do not justify adding validation, normalization, fallback behavior, or ownership guarantees to the contract.

## Evidence

The static evidence consists solely of the supplied function representation and the dependency identity mapping.

It establishes the initial discriminator read, the conditional base read, the two result-selection paths, the dependency’s argument order, unconditional output publication after normal path completion, and return-value selection.

The four-byte-word interpretation yields the 16-byte displacement formula. The collection interpretation is inferred intent only. Exact ABI behavior and dependency internals remain unresolved.

No runtime tests have been performed.
