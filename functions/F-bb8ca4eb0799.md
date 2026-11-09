<!-- Trusted provenance: F-bb8ca4eb0799; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bb8ca4eb0799

## Purpose

F-bb8ca4eb0799 derives a signed 32-bit result from a descriptor, a selector, an adjustment, and values available after an unconditional preparation call. Depending on the descriptor, it:

- Calculates a result through F-870c4fcf7295.
- Selects one of four prepared values and adds the adjustment.
- Reads a selected word from a reference object and adds or subtracts the adjustment.
- Returns the adjustment alone.
- Returns zero for an unsupported combination.

Some descriptors also request updates to a context word. These updates can occur even when dispatch subsequently rejects the descriptor and returns zero.

A coordinate-like interpretation is plausible but not established. This contract therefore describes numeric values, word offsets, selection rules, and access ordering without assigning geometric meaning.

## Interface

### Arguments

| Position | Contract name | Observed role |
|---|---|---|
| 1 | Context | Passed to F-fdb6d4e0b2c8; initializes prepared slot 1; supplies the address for a conditional context-word update. |
| 2 | Reference | Initializes prepared slot 2; supplies the address for conditional reference-word reads. |
| 3 | Scale | Passed to F-fdb6d4e0b2c8; initializes prepared slot 3; its original numeric value is a multiplication operand on the calculation path. |
| 4 | Descriptor | Passed to dependencies; initializes prepared slot 4; controls normalization, selection, flag requests, and dispatch. |
| 5 | Adjustment | Signed 32-bit value added to or subtracted from a selected result. |
| 6 | Selector | Unsigned 32-bit value, sometimes replaced by descriptor-derived information. |

The first four arguments are pointer-shaped in the supplied representation, but their uses do not establish four ordinary object-pointer parameters. Scale and descriptor are also interpreted numerically.

The visible operations use 32-bit integer conversions and four-byte object words. The contract describes that observed 32-bit interpretation. A portable mapping to wider pointer representations is **documented_only**.

### Result

The result is a signed 32-bit integer. There is no independent success or failure indicator.

Zero may be an ordinary calculated value or the result of rejecting a descriptor or selector. Returning zero does not imply that preparation had no effects or that no context update occurred.

### Storage and access requirements

Requirements depend on the path:

- Every invocation must meet F-fdb6d4e0b2c8’s requirements, which are not supplied.
- A context flag request requires readable and writable storage for a four-byte word at context byte offset 72.
- Reference modes require the selected four-byte word at reference byte offset 0, 4, 8, or 12 to be readable.
- Prepared-value modes use selected values numerically; they do not dereference the selected value.
- The calculation path additionally requires F-870c4fcf7295’s input requirements to be met.

The function performs no visible null checks, bounds checks, alignment correction, or conversion of access failures into an error result. A path without a direct reference read still passes through the preparation dependency, whose access requirements remain unknown.

No ownership transfer is established by this interface.

## Behavior

### Preparation and observation order

The function first initializes four local prepared slots, in order, from context, reference, scale, and descriptor.

It then invokes F-fdb6d4e0b2c8 with:

1. The original context.
2. The original scale.
3. The original descriptor.
4. The address of prepared slot 1.

The return value is ignored. Descriptor interpretation occurs after this call returns.

The address of the first prepared slot is exposed to the dependency. The supplied evidence does not establish a public four-element output-array contract or the dependency’s permitted write extent. Subsequent uses of prepared slots observe their values after preparation, subject to any later effects of the calculation dependency.

The unconditional preparation call is significant: a request that ultimately returns zero does not bypass it. No assumption may be made that it preserves the context word, reference words, globals, or other reachable storage.

### Effective descriptor and selector

Dispatch uses an effective descriptor derived from the original descriptor. Dependency calls continue to receive the original descriptor.

First, the original descriptor’s low eight bits are examined:

- If any are set, the effective descriptor initially equals the original descriptor, and the supplied selector remains unchanged.
- If all are clear, the effective descriptor becomes zero. The selector becomes the original descriptor interpreted as signed 32-bit and shifted arithmetically right by eight positions, then represented in the unsigned 32-bit selector.

This shift propagates the descriptor’s sign bit. It is not an unsigned extraction of only the next byte. For example, an original descriptor of `0x80000300` produces selector `0xFF800003`, not selector 3, while its effective descriptor becomes zero.

The function next compares the effective descriptor after clearing bits 8 through 11 against `0x80000010`. On equality:

- The effective descriptor becomes exactly `0x80000010`.
- The selector becomes original-descriptor bits 8 through 11, yielding a value from 0 through 15.

Only those four bits are disregarded for this recognition. An additional bit elsewhere prevents recognition.

### Positive-descriptor calculation

If special recognition did not occur and the effective descriptor is strictly positive under signed 32-bit interpretation, the function takes the calculation path.

| Selector | Difference used for calculation | Base used after the calculation call |
|---|---|---|
| 1 or 3 | Prepared slot 3 minus prepared slot 1 | Then-current prepared slot 1 |
| 2 or 4 | Prepared slot 4 minus prepared slot 2 | Then-current prepared slot 2 |
| Any other value | None | Return zero |

For a supported selector, the selected difference is multiplied by the original scale’s numeric value. The resulting 32-bit expression and the original descriptor are passed to F-870c4fcf7295.

The difference and product are evaluated before the dependency call. The return base is obtained afterward:

- For selector 1 or 3, the function copies the then-current prepared slot 1 into prepared slot 2.
- For selector 2 or 4, it uses the then-current prepared slot 2 without that copy.

The result expression adds the dependency result to the adjustment, then adds prepared slot 2.

The distinction between pre-call operands and post-call base is part of the behavior. Preservation of accessible prepared storage by the dependency is not established.

All positive-descriptor paths return before this function’s context-flag handling. In particular, a positive descriptor containing bit `0x20` or `0x40` does not cause the direct flag updates described below.

### Context flag requests

All paths not handled by the positive-descriptor calculation inspect two independent bits of the effective descriptor.

The direct memory accesses occur in this order:

1. If bit `0x20` is set, read the context word at byte offset 72, set bit `0x10` in the value read, and write the result to the same word.
2. If bit `0x40` is set, read that context word and write back the value read with bit `0x100` set.

When both requests are present, the second read follows the first write. The function does not replace these with a single combined read and write.

Each update is based on the word observed at that update’s actual read. Neither an entry-time value nor a value observed before preparation determines the exact write unless intervening preservation is separately established.

These accesses are ordinary read/modify/write sequences. Atomicity, synchronization, and protection against intervening modifications are not established.

For dispatch, the function clears bit `0x20` from the effective descriptor. It does not clear bit `0x40`. Consequently:

- A `0x20` request can accompany a supported mode.
- A `0x40` request can update context and nevertheless leave an unsupported dispatch value.
- Dispatch rejection does not undo either update.

### Mode dispatch

After flag handling, dispatch recognizes these values:

| Dispatch value | Result behavior |
|---|---|
| `0x80000001` | Return the adjustment; ignore the selector. |
| `0x80000002` | Select a prepared value and add the adjustment. |
| `0x80000004` | Use the reference selection described below. |
| `0x80000008` | Use the reference selection described below. |
| `0x80000010` | Select a prepared value and add the adjustment. |
| `0x00000000` | Select a prepared value and add the adjustment. |
| Any other value | Return zero. |

For prepared-value selection, selectors 1, 2, 3, and 4 select the correspondingly numbered prepared slots. Any other selector returns zero without adding the adjustment. Selected values contribute their numeric 32-bit representation, not the contents of memory they might address.

Reference selection is:

| Dispatch value | Selector | Direct reference read | Result |
|---|---|---|---|
| `0x80000008` | 1 | Four-byte word at offset 8 | Word minus adjustment |
| `0x80000008` | 3 | Four-byte word at offset 0 | Word plus adjustment |
| `0x80000004` | 2 | Four-byte word at offset 12 | Word minus adjustment |
| `0x80000004` | 4 | Four-byte word at offset 4 | Word plus adjustment |

Other selectors in either reference mode return zero without a direct reference-word read.

Reference access uses the original reference argument, not the potentially modified value in prepared slot 2. Context updates likewise use the original context argument, not prepared slot 1.

A selected reference word is read after preparation and after any applicable direct context writes. If context and reference storage overlap, the reference read may observe a preceding context update. Dependency effects can also change the word before its read.

### Arithmetic boundaries

Descriptor masks, selectors, and accessed words are 32-bit. Descriptor positivity and the descriptor-derived right shift use signed 32-bit interpretation.

The visible arithmetic includes signed 32-bit differences, multiplication, and additions or subtraction. There is no visible saturation, widened intermediate, overflow check, or overflow status.

Low-32-bit wraparound is a plausible machine-level interpretation, but a portable signed-overflow rule is not established by the supplied representation alone. Overflow-sensitive outcomes are **documented_only**. The acceptance examples avoid signed overflow.

## Side effects

Direct effects are limited to local prepared-storage changes, the conditional context-word reads and writes, selected reference-word reads, and dependency calls.

No direct allocation, deallocation, reference-count operation, persistent-storage operation, or ownership transfer is visible.

Dependency-mediated effects are not bounded by the visible body. In particular:

- F-fdb6d4e0b2c8 may affect storage before this function reads context or reference words.
- Neither dependency is established to preserve globals or other reachable state.
- Neither dependency’s resource acquisition, release, callbacks, or exceptional behavior is known.
- Passing local storage does not establish a valid lifetime for retaining that address beyond this invocation.

There is no visible rollback. A returned zero does not reverse prior dependency effects or context writes.

## Dependencies

### F-fdb6d4e0b2c8

Invoked once unconditionally before descriptor dispatch, assuming execution reaches and returns normally from the call.

Arguments are original context, original scale, original descriptor, and the address of prepared slot 1. Its result is ignored.

Its requirements, write extent, context/reference preservation, ownership rules, and failure behavior are **documented_only**. Calling it a preparation dependency describes its position and subsequent slot use, not an independently established semantic guarantee.

### F-870c4fcf7295

Invoked once only on the positive-effective-descriptor path with selector 1, 2, 3, or 4.

Arguments are the selected difference multiplied by the original scale, and the original descriptor. Its returned integer contributes directly to the result.

Its mathematical meaning and side effects are **documented_only**. Division, interpolation, fixed-point conversion, and state preservation must not be assumed.

## Acceptance examples

These are conditional contract examples, not executed tests. All require valid accesses, normal dependency returns, and arithmetic within signed 32-bit range.

1. **Zero descriptor replaces the supplied selector.**  
   Descriptor is zero, supplied selector is 4, and adjustment is 7. Even if preparation leaves slot 4 equal to 40, the derived selector is zero. The function returns zero, not 47.

2. **Selector encoded above a zero low byte.**  
   Descriptor is `0x00000300`. Preparation leaves slot 3 equal to 30, and adjustment is 7. The effective descriptor is zero and derived selector is 3. The result is 37.

3. **Signed selector derivation is not byte extraction.**  
   Descriptor is `0x80000300`. The effective descriptor becomes zero, and the selector becomes `0xFF800003`. Prepared-value selection rejects it and returns zero.

4. **Special four-bit selector encoding.**  
   Descriptor is `0x80000310`. Preparation leaves slot 3 equal to 30, and adjustment is 7. Special recognition selects mode `0x80000010` and selector 3. The result is 37.

5. **Context update followed by supported dispatch.**  
   Descriptor is `0x80000021`. The context word contains `0x02` immediately before this function’s flag read, after preparation has returned. The function writes `0x12`, removes `0x20` for dispatch, and returns the adjustment. No assumption about the context word at entry is required.

6. **Context update followed by rejection.**  
   Descriptor is `0x80000041`. The context word contains `0x02` immediately before this function’s flag read, after preparation has returned. The function writes `0x102`, retains `0x40` in the dispatch value, and returns zero.

7. **Reference subtraction uses the value at the actual read.**  
   Descriptor is `0x80000008`, selector is 1, and adjustment is 7. The reference word at offset 8 contains 100 when this function reads it after preparation. The result is 93, irrespective of any different value that word held before preparation.

8. **Reference addition uses the value at the actual read.**  
   Descriptor is `0x80000004`, selector is 4, and adjustment is 7. The reference word at offset 4 contains 25 when this function reads it after preparation. The result is 32.

9. **Adjustment-only mode ignores selector.**  
   Descriptor is `0x80000001`, selector is 99, and adjustment is −12. After preparation, the function returns −12 without a direct reference-word read.

10. **Calculation operands precede the post-call base.**  
    Descriptor is 1, selector is 1, and original scale is 2. Prepared slots 1 and 3 contain 10 and 30 when the calculation arguments are evaluated. F-870c4fcf7295 receives 40 and descriptor 1. If it returns 5, slot 1 contains 10 when subsequently read, and adjustment is 7, the result is 22. A different post-call slot 1 value changes the base without changing the already-passed product.

11. **Unsupported positive selector bypasses direct flag handling.**  
    Descriptor is `0x21` and selector is 5. Preparation still occurs, but the function returns zero without calling F-870c4fcf7295 or directly accessing the context flags word.

12. **Two flag requests remain two updates.**  
    Descriptor is `0x80000061`. If the first flag read observes `0x02`, its write is `0x12`. If the second read observes `0x12`, its write is `0x112`. Dispatch retains `0x40` and returns zero. The second exact write is conditional on the value observed by the second read.

## Uncertainties

The following remain **documented_only**:

- Semantic object types and the intended meaning of modes and selectors.
- The preparation dependency’s write extent and which prepared slots it changes.
- Either dependency’s preservation of context, reference, globals, or accessible local storage.
- Dependency ownership, retained addresses, resource operations, callbacks, and failures.
- Overflow-sensitive arithmetic results and portability to wider pointers.
- Alignment requirements, byte order, concurrent-access behavior, and memory-mapped-I/O guarantees.
- Behavior following invalid memory access or abnormal dependency completion.

“Prepared,” “scale,” and “context word” are descriptive contract names, not evidence of a broader API model.

## Evidence

Established static evidence includes the preparation order and call, descriptor transformations, signed-positive branch, calculation arguments, post-call base selection, context access order, dispatch values, reference offsets, and result expressions.

The coordinate-like purpose and a low-32-bit wraparound interpretation are inferences, not acceptance requirements.

The acceptance examples condition context and reference values on their actual observation points. Entry-time values alone cannot determine those observations because the unconditional preparation dependency runs first and has no established preservation contract.

No runtime tests have been performed. No dependency implementation or execution trace was supplied.
