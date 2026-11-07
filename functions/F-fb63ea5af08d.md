<!-- Trusted provenance: F-fb63ea5af08d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fb63ea5af08d

## Purpose

F-fb63ea5af08d selects behavior using a one-byte discriminator in a caller-supplied context:

- **Mode zero:** if a signed length comparison succeeds, write two zero bytes, set the supplied length to two, and return a context-designated address.
- **Mode six:** resolve an object, invoke an indirect operation, pass its result to an external operation, conditionally reduce the supplied length, and return a context field.
- **Mode ten:** if the supplied length equals one, write a byte obtained through lookup, or zero when lookup fails, and return a context-designated address.
- **Other modes:** return zero without accessing the supplied length.

These are established caller-side behaviors. The application meanings of the discriminator, selectors, output bytes, and dependency results are not established. In particular, the external operation’s argument shape does not establish that it copies memory.

## Interface

### Inputs and return value

The function accepts:

1. A context address.
2. An address identifying a 32-bit length location.

It returns a 32-bit value. Depending on the branch, this is zero or a value loaded from a context field. The representation does not establish a separate status type, success flag, or ownership-bearing return type.

Addresses and pointer-sized context fields are represented as 32-bit quantities. The length is compared as signed 32-bit in mode zero and as unsigned 32-bit in mode six. Mode ten tests its 32-bit value for equality with one.

### Context layout

Offsets are byte offsets from the context address.

| Offset | Access width | Established use |
|---|---:|---|
| `0x70` | 32 bits | First argument to the external operation in mode six; reloaded afterward for the return value. |
| `0x91` | 8 bits | Initial mode discriminator. |
| `0x92` | 16 bits | Read in mode six; the numeric upper eight bits become an indirect-call argument. |
| `0x94` | 16 bits | Lookup selector: masked to eight bits in mode six, used at full width in mode ten. |
| `0xB4` | 32 bits | Output address for direct byte stores in modes zero and ten; also supplies their return values. |

Access widths are established; byte order is not. References to upper and lower bits concern the numeric result of a 16-bit read, not an assumed memory byte order.

### Referenced storage

A successful mode-six lookup supplies an object address used to read:

- A 32-bit dispatch-table address at object offset zero.
- An indirect callable entry at dispatch-table offset `0x0C`.

A successful mode-ten lookup supplies an address from which one byte is read at offset four. The shared lookup dependency does not establish that both branches receive objects of the same logical type.

Mode six also supplies an indirect dependency with the address of a local region containing two 32-bit words. The wrapper initializes only the first word.

### Path-dependent access requirements

Valid storage is required for every access actually reached. There are no explicit general null, bounds, alignment, or callable-target checks.

- **Every invocation:** the context must support the initial discriminator-byte read.
- **Mode zero:** the length location must be readable. If its signed value exceeds one, the output-address field must be readable, the two reached output addresses must be writable, and the length location must also be writable.
- **Mode six before lookup:** the two selector-related context fields must be readable. The wrapper has not yet accessed the caller’s length.
- **Mode six, lookup failure:** the length location must be writable for a 32-bit zero store. This path does not first read the length.
- **Mode six, lookup success:** the object and dispatch information must support the reached accesses and call. After the external operation returns, the length location must be readable. It must additionally be writable only if the then-current first local word is smaller than the then-current unsigned length. If that comparison is false, the wrapper only reads the length.
- **Mode ten:** the length location must be readable. If its value equals one, the lookup-selector and output-address fields must be readable, the output byte must be writable, and a nonzero lookup result must support the byte read at offset four. The wrapper does not explicitly write the length on this branch.
- **Other modes:** the wrapper does not access the length location.

These requirements describe the wrapper’s own accesses. Dependencies may impose additional requirements, including requirements on storage reachable through the context. Those requirements are unknown. In particular, read-only length storage on a successful, non-reducing mode-six path is sufficient for the wrapper’s own length access, not an unconditional guarantee for the entire call.

## Behavior

### Initial selection

The function first reads the byte at context offset `0x91`. The captured value determines the branch. The discriminator is not reread after dependency calls or direct stores.

### Mode zero: two-byte zero result

The function reads the supplied 32-bit length and interprets its bit pattern as signed 32-bit.

If the signed value is greater than one, the established sequence is:

1. Load the output address from context offset `0xB4` and store a zero byte there.
2. Reload the output address from the same field and store a zero byte at that address plus one.
3. Reload that context field and retain its 32-bit value for the return.
4. Store the 32-bit value two at the length location.
5. Return the retained value.

The separate output-address loads are observable when output storage overlaps the context. The first byte store can affect the address used by the second store. The return value is captured before the length store, so overlap between the length location and the output-address field can make the returned value differ from the field’s final contents.

If the signed comparison fails, the function returns zero without explicitly writing the output or length.

This is not an unsigned capacity check. Bit patterns from `0x80000000` through `0xFFFFFFFF` represent negative signed values and fail the comparison. The passing range is `0x00000002` through `0x7FFFFFFF`.

The second byte store uses an address increment of one. Valid address arithmetic and writable reached storage are required; the representation does not establish portable behavior at an address-space boundary.

### Mode six: lookup and dependent operations

The function first reads the 16-bit field at context offset `0x92`. It then reads the 16-bit field at offset `0x94`, masks that value to its low eight bits, and initializes the first word of the local two-word region with the resulting value.

F-7c98cdf9320e is called with:

- The context address.
- The masked selector, numerically between zero and 255.

The wrapper does not read the caller’s length before this lookup.

#### Lookup returns zero

The function stores zero in the caller’s 32-bit length location and returns zero.

It does not invoke the subsequent dependencies or explicitly store through either context-designated output address. The length location needs to support a write, but this branch does not require a preceding length read.

The lookup may itself have side effects; returning zero does not establish that it left memory unchanged.

#### Lookup returns a nonzero object address

The function proceeds in this order:

1. Call F-f1edefb7c207 with no explicit arguments.
2. Resolve the indirect callable using the lookup object’s dispatch-table field and the entry at offset `0x0C`.
3. Invoke that callable with four arguments:
   - The lookup object address.
   - The upper eight bits of the earlier 16-bit read from context offset `0x92`.
   - The address of the local two-word region.
   - The 32-bit result of F-f1edefb7c207.
4. Call F-unknown-transfer with:
   - A value loaded from context offset `0x70`.
   - The indirect operation’s return value.
   - The current first local word.
5. After that external call returns, compare the current first local word with the current caller-supplied length as unsigned 32-bit values.
6. If the first local word is smaller, store its value at the length location. Otherwise make no explicit length store.
7. Reload context offset `0x70` and return that value.

The indirect argument derived from offset `0x92` lies between zero and 255. Its lower eight bits do not contribute.

Dispatch resolution occurs after F-f1edefb7c207 returns. Nothing supplied establishes that this dependency preserves the lookup object, dispatch information, context, or other shared state. Normal completion requires the later accesses to remain valid.

The indirect operation receives writable local storage and can change the first word. Consequently, the third argument to F-unknown-transfer need not equal the original masked selector.

The length comparison occurs only after the external operation. The wrapper does not first limit that operation’s third argument to the supplied length. Even if the dependency were independently established to copy bytes, this wrapper would not establish the caller’s length as a pre-call transfer bound.

The first local word is examined again after the external operation. No supplied dependency contract guarantees that this value is preserved across the call, including through an address retained by an earlier dependency. The comparison value therefore must not unconditionally be equated with the previously passed third argument.

On the successful path, a length read is unconditional once execution reaches the comparison; a length write is conditional. Equality does not cause a write. Both comparison operands use their full unsigned 32-bit values.

The external operation’s return value is ignored. The function returns a fresh load from context offset `0x70`, not the indirect operation’s result and not necessarily the value passed as the external operation’s first argument.

### Mode ten: one-byte lookup result

The function reads the supplied 32-bit length and tests whether its value equals one. The representation temporarily treats that value as an address, but it does not dereference that address.

If the value is not one, the function returns zero without calling lookup or explicitly writing an output byte.

If the value is one:

1. Read the full 16-bit selector at context offset `0x94`.
2. Call F-7c98cdf9320e with the context and that selector.
3. If lookup returns zero, store one zero byte through the address then obtained from context offset `0xB4`.
4. Otherwise, obtain the byte at lookup-result offset four and store it through the context-designated output address.
5. Reload context offset `0xB4` and return its 32-bit value.

No explicit length store occurs. Dependencies or overlapping output storage could nevertheless change the length location.

Mode ten preserves all 16 selector bits. Values differing only in the upper byte are therefore distinct lookup arguments in this branch, unlike mode six.

The representation establishes the source byte and destination field used for the successful store, but does not establish an independent ordering guarantee between their operand reads.

### Other modes

All discriminator values other than zero, six, and ten return zero after the initial mode read. The wrapper does not read or write the supplied length or access either output-address field on these paths.

## Side effects

The wrapper’s explicit caller-visible stores are:

- Two zero-byte stores and a length store of two on the accepted mode-zero path.
- A length store of zero on mode-six lookup failure.
- A conditional length store on mode-six lookup success, strictly when the post-external-call local value is smaller than the current unsigned length.
- One output-byte store on the accepted mode-ten path.

The local first-word initialization is separate from these caller-visible stores. Dependencies receive access to local storage and may have additional effects.

No explicit allocation, release, reference-count adjustment, or ownership transfer appears in the wrapper. This does not establish that dependencies perform none of those actions. Returned values are not established as newly allocated storage or as transferring ownership.

No non-overlap guarantee is shown. Output stores and length stores may affect later context or object reads when storage aliases. Likewise, “no explicit length store” does not mean that the length cannot change through overlapping output storage or dependency effects.

There is no explicit recovery mechanism for invalid memory, an invalid callable target, or dependency failure outside the checked zero lookup result. Zero is not a unique error diagnosis: it can result from a rejected branch, a failed lookup in mode six, or a zero value loaded for an otherwise completed branch.

## Dependencies

### F-7c98cdf9320e

Called with the context and a selector:

- Mode six supplies an eight-bit-masked numeric value.
- Mode ten supplies the full 16-bit value.

A zero result selects the missing-object behavior. A nonzero result is used as an address. Lookup semantics, mutation effects, ownership, lifetime, and failure causes are unknown.

### F-f1edefb7c207

Called only after successful mode-six lookup, with no explicit arguments. Its 32-bit result becomes the fourth indirect-call argument. It is not established to be pure or to preserve shared state.

### F-unknown-dispatch

Document-local opaque ID for the callable selected through dispatch-table offset `0x0C`.

It receives the object, an eight-bit-derived numeric argument, the local-region address, and the result of F-f1edefb7c207. Its result becomes the external operation’s second argument.

The second local word is not initialized by the wrapper. Whether the callable reads it, writes it, retains the region’s address, or imposes additional storage requirements is unknown.

### F-unknown-transfer

Document-local opaque ID for the external operation.

It receives a context-loaded value, the indirect result, and the first local word. A destination/source/count interpretation is plausible intent, not established behavior. Copying, overlap handling, zero-count behavior, errors, resource ownership, and preservation guarantees remain unknown.

## Acceptance examples

These are conditional static consequences, not executed tests. Except where stated otherwise, assume valid non-overlapping storage, normal dependency returns, and no dependency mutation of the values described as remaining unchanged.

1. **Mode zero, length two:** two output bytes become zero, the length becomes two, and the captured output-address value is returned.

2. **Mode zero, length one:** return zero; no explicit output or length store occurs.

3. **Mode zero, length `0xFFFFFFFF`:** the signed value is negative, so the branch rejects it without output or length stores.

4. **Mode six, selector `0xABCD`, failed lookup:** lookup receives `0xCD`; the wrapper writes zero to the length and returns zero. The length location must be writable, but the wrapper does not read its previous contents.

5. **Mode six, argument field `0xA512`:** the indirect operation receives `0xA5` as its second argument, regardless of the lower byte.

6. **Mode six, successful lookup and reduction:** if the post-external-call local word is 12 and the current length is 100, the wrapper stores 12 at the length location. That location must be readable and writable.

7. **Mode six, successful lookup without reduction:** if the post-external-call local word is 12 and the current length is five, the wrapper reads the length but does not write it. Read-only length storage satisfies this wrapper-side access, provided unknown dependency requirements are also met.

8. **Mode six, equal values:** if both comparison operands are 12, no length store occurs. Equality does not enter the reduction branch.

9. **Mode six, unsigned comparison:** with local value `0xFFFFFFFF` and length one, no length store occurs. The local value is not treated as negative.

10. **Mode ten, successful lookup:** with length one and selector `0xABCD`, lookup receives `0xABCD`. If the object byte at offset four is `0x7E`, that byte is stored in the output.

11. **Mode ten, failed lookup:** with length one, lookup failure produces a zero output byte and returns the reloaded output-address field. No explicit length store occurs.

12. **Unsupported mode seven:** return zero without accessing the supplied length.

## Uncertainties

The following remain **documented_only**:

- Application semantics and valid domains of the context fields.
- Byte order, alignment requirements, and boundary address-arithmetic behavior.
- Full dependency contracts, including storage validity requirements and state preservation.
- The external operation’s actual memory and error behavior.
- The indirect operation’s use of the uninitialized second local word.
- Retention or later use of the local-region address.
- Ownership and lifetime of lookup results, indirect results, and returned addresses.
- Intended support for aliasing or concurrent mutation.
- Behavior when reached memory or callable targets are invalid.

These gaps do not justify assuming copying, global preservation, allocation, release, or unconditional writable-length requirements.

## Evidence

The supplied static representation establishes branch selection, access widths and offsets, selector transformations, explicit store sequences, dependency-call order, comparison signedness, and return-value loads.

In particular, it establishes distinct mode-six length requirements: lookup failure performs a store without a preceding length read; lookup success reads the length after the external call and stores only when the unsigned comparison is strictly less.

The supplied dependency map identifies F-7c98cdf9320e and F-f1edefb7c207. The indirect callable and external operation lack resolved identities and implementations.

Application intent and dependency semantics remain separate from established wrapper behavior. No runtime tests have been performed.
