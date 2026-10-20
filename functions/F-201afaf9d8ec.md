<!-- Trusted provenance: F-201afaf9d8ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-201afaf9d8ec

## Purpose

F-201afaf9d8ec returns a scalar formed from a group index and a cursor. A hidden entry condition determines whether the cursor is used unchanged or first moved downward until a mask inspection finds a set bit.

When searching occurs, it is **exclusive of the initial cursor**: the cursor is decremented before the first inspection. Subsequent inspections also follow a decrement. The first inspected set bit ends the search.

The apparent intent is to combine a position within a 32-position group with a group offset, producing a one-based position. This is inferred intent, not a complete domain contract. The visible representation does not establish that the cursor is restricted to a single group, that the group index is nonnegative, or that a successful inspection is guaranteed.

The search uses the cursor’s **low eight bits** as the shift count, while the result calculation uses the **entire final cursor**. Consequently, a shift count of zero does not establish that the cursor itself is zero. This distinction affects both exhaustion behavior and returned values.

## Interface

The visible interface has four explicit scalar arguments and one scalar return value.

| Input | Represented width and interpretation | Role |
|---|---|---|
| Group index | Signed 32-bit integer | Multiplied by 32 for the result calculation. |
| Initial cursor | Unsigned 32-bit integer | Used unchanged when searching is disabled; otherwise decremented before each inspection. |
| Unused argument | Four-byte value of unspecified interpretation | Accepted but not visibly used. |
| Bit mask | Unsigned 32-bit integer | Supplies the bits inspected during searching. |

The return is represented as a signed 32-bit integer. Executable-level confirmation of these widths and the calling convention remains documented_only.

There is also a hidden, byte-sized entry condition in the supplied representation. Its initial value is not assigned within the visible body and is not one of the four explicit arguments. Its truth value determines whether the loop begins. The explicit arguments alone therefore do not determine the function’s path.

The visible interface contains no pointer arguments, output buffers, structured records, or ownership-bearing objects. No alignment, field layout, or memory-region contract is established beyond the represented scalar widths.

The unused argument is not visibly interpreted or dereferenced. Its acceptance does not establish acquisition, transfer, retention, or release of any resource, even if a caller gives its bits an address-like or handle-like meaning.

## Behavior

### Entry selection

The hidden condition is checked before any decrement or mask inspection.

- If false, searching is skipped. The initial cursor is used directly in the result calculation.
- If true, the function enters the downward search.

The visible body does not initialize this condition from the cursor, mask, group index, or unused argument. In particular, a zero mask does not itself enable searching, and a nonzero mask does not itself disable it.

No validation or error branch precedes the selected path.

### Search ordering

For each entered iteration, the semantic order is:

1. Decrease the full unsigned 32-bit cursor by one.
2. Obtain the shift count from the updated cursor’s low eight bits.
3. Shift the unsigned mask right by that count.
4. Inspect the least significant bit of the shifted value.
5. Update the repetition condition: a clear bit continues searching; a set bit stops searching.

The cursor retained after a successful inspection is the full updated cursor, not merely its low eight bits and not the position before the decrement.

For shift counts from 0 through 31, the inspection corresponds to the mask bit with that number, with bit zero the least significant bit. Within such an execution, the first set bit encountered in descending inspection order ends the search.

These are semantic ordering requirements supported by the representation. They do not establish a particular machine instruction sequence, physical register allocation, or externally observable processor-flag behavior.

### Cursor arithmetic and shift counts

Cursor decrement is unsigned 32-bit subtraction:

- A positive cursor decreases by one without unsigned wraparound.
- A cursor of zero decreases to 4,294,967,295.

The shift count is the updated cursor modulo 256. Its represented range is 0 through 255.

Only counts from 0 through 31 have an established ordinary shift interpretation for the represented 32-bit mask. Counts from 32 through 255 have no established portable behavior here. They must not be silently reduced modulo 32, assumed to produce zero, or assigned another target-specific outcome.

The low-eight-bit calculation and full-cursor arithmetic must remain distinct:

- A shift count of zero means the full cursor is a multiple of 256.
- It does not mean that the full cursor is zero.
- Decrementing any such cursor changes the next shift count from zero to 255.
- That decrement wraps the full 32-bit cursor only when the cursor being decremented is zero.

For example, an initial cursor of 257 first decreases to 256, giving shift count zero. If that inspection is clear, the next iteration decreases the cursor to 255, giving shift count 255. Neither decrement wraps the full cursor.

By contrast, an initial cursor of 1 first decreases to zero. If that inspection is clear, the next decrement wraps the full cursor to 4,294,967,295 and also gives shift count 255.

### Established search domain

A simple sufficient domain for an established search result is:

- The hidden condition is true.
- The initial cursor is between 1 and 32.
- At least one mask bit strictly below that cursor is set.
- The result arithmetic remains within the established arithmetic domain described below.

Within that domain, the search finds the highest-numbered set mask bit strictly below the initial cursor. Bits at or above the initial cursor do not affect the selection.

This is a sufficient domain, not an input restriction enforced by the function.

Some larger cursors also permit established inspections and successful exits. For example, an initial cursor of 257 produces an initial shift count of zero. If mask bit zero is set, searching stops with full cursor 256. The final calculation uses 256, not zero.

More generally, an initial cursor whose remainder modulo 256 is from 1 through 32 begins with a shift count from 0 through 31. If a set bit is reached before descending past shift count zero, the visible search remains within the ordinary shift domain. A successful inspection retains the corresponding full cursor, including its higher bits.

An initial cursor outside those residue classes begins with an unsupported shift count. A set bit elsewhere in the mask does not establish how that execution behaves.

### Exhaustion and termination

There is no explicit exhausted-search result, iteration bound, sentinel, or error report.

If every inspected bit in a valid descending sequence is clear, the function continues decrementing. After a clear inspection at shift count zero, the next iteration has shift count 255.

The full cursor at the zero-count inspection determines whether that next decrement wraps:

- If the full cursor is zero, the next decrement wraps to 4,294,967,295.
- If the full cursor is a positive multiple of 256, the next decrement does not wrap.

In either case, the next shift count is outside the established 0-through-31 shift domain. Thus, a clear bit-zero inspection can lead to unsupported shift behavior without any full-cursor wraparound.

An all-zero mask with searching enabled has no successful exit while its inspections remain within the established shift domain. It must not simply be specified as an infinite loop: continued execution reaches an unsupported shift count, either immediately or after the valid descending sequence.

### Result calculation

After searching stops, or immediately when searching is skipped, the function forms:

**32 × group index + final cursor + 1**

The multiplication is represented as signed arithmetic. For a signed 32-bit group index, multiplication by 32 is within range exactly when the group index is between −67,108,864 and 67,108,863, inclusive. Outside that interval, a portable signed multiplication result is not established by the supplied representation.

Where that multiplication is defined, its result is combined with the unsigned 32-bit cursor. Under the represented arithmetic rules, the additions are unsigned and wrap modulo 2³². The final addition of one participates in that unsigned arithmetic.

The resulting unsigned value is then returned through a signed 32-bit return type:

- Values from zero through 2,147,483,647 have an unambiguous signed return value.
- Conversion of larger unsigned values to the signed return type is not established for the original execution environment.

Accordingly, the mathematical formula is an exact signed-return specification only when the multiplication and final conversion are within their established ranges. Unsigned addition can wrap even when the eventual return value is unambiguous.

No argument check rejects arithmetic boundary cases.

## Side effects

The visible body contains no explicit memory access, function call, allocation, deallocation, I/O, synchronization operation, or persistent-state update.

Its visible state transitions are limited to:

- Decrementing the working cursor before each inspection.
- Replacing the loop condition after each inspection.
- Producing the scalar return value on a completed execution.

The group index, unused argument, and mask have no visible updates.

There is no visible ownership transfer or resource release. No unknown callee’s memory or global-state behavior needs to be assumed because no calls appear.

These observations do not establish an ABI-level preservation guarantee. Register preservation, processor-flag effects, argument transport, and return transport remain documented_only. Likewise, the lack of explicit memory operations is not a general safety guarantee for executions that encounter unsupported shifts or arithmetic behavior.

## Dependencies

No called-function dependencies are visible. F-201afaf9d8ec invokes no other opaque function ID in the supplied body.

Its behavioral dependencies are:

- The hidden condition’s value at entry.
- The represented integer widths and signedness.
- Unsigned 32-bit decrement and addition.
- Extraction of the cursor’s low eight bits.
- Shift semantics for the resulting count.
- Signed multiplication by 32.
- Conversion of the unsigned calculated value to the signed return type.

The origin of the hidden entry condition is documented_only. No supplied evidence identifies a caller or another function that establishes it.

Target-specific behavior for oversized shifts and out-of-range signed return conversions is also documented_only. No such behavior is assumed in the acceptance examples.

## Acceptance examples

These are conditional static acceptance examples, not runtime observations. “Only bit” means all other mask bits are clear.

| Conditions | Required outcome or established boundary |
|---|---|
| Entry condition false; group index 2; cursor 7; any mask | Return 72 without inspecting any mask bit. |
| Entry condition false; group index 0; cursor 0; zero mask | Return 1. The zero mask does not enable searching. |
| Entry condition true; group index 0; cursor 6; bit 5 set | Inspect bit 5 first and return 6 immediately. |
| Entry condition true; group index 2; cursor 6; only bit 3 set | Inspect bits 5, 4, then 3; return 68. |
| Entry condition true; group index 0; cursor 6; only bits 6 and 2 set | Do not inspect bit 6; inspect down to bit 2 and return 3. |
| Entry condition true; group index 0; cursor 32; only bit 31 set | Inspect bit 31 first and return 32. |
| Entry condition true; group index 0; cursor 1; bit 0 set | Decrease the cursor to zero, inspect bit zero, and return 1. |
| Entry condition true; group index 0; cursor 257; bit 0 set | Decrease the full cursor to 256, inspect bit zero, and return 257. |
| Entry condition true; cursor 257; zero mask | Inspect bit zero with full cursor 256. Then decrease to 255 without full-cursor wraparound; the next shift count is 255. No portable return is established. |
| Entry condition true; cursor 1; zero mask | Inspect bit zero with full cursor zero. Then wrap the cursor to 4,294,967,295; the next shift count is 255. No portable return is established. |
| Entry condition true; cursor 0 | The first decrement wraps to 4,294,967,295 and the first shift count is 255. No portable return is established. |
| Entry condition true; cursor 33 | The first decrement gives cursor 32 and shift count 32. No portable shift outcome or return is established. |
| Entry condition false; group index 0; cursor 4,294,967,295 | Unsigned addition of one wraps to zero; return 0. |

Changing only the unused four-byte argument must not change the specified calculation or inspection sequence for an otherwise established execution.

The same four explicit arguments may select different paths depending on the hidden condition. With group index zero, cursor six, and only mask bit three set, a false entry condition returns seven; a true entry condition returns four.

These examples do not assign outcomes to executions after an unsupported shift or signed arithmetic boundary.

## Uncertainties

The following remain documented_only:

- **Entry-state origin:** what supplies the initial condition and whether callers can reliably control it.
- **Executable integer model:** whether the represented widths and arithmetic precisely match the original execution environment.
- **Oversized shifts:** actual target behavior for counts from 32 through 255.
- **Signed multiplication overflow:** executable behavior outside the stated multiplication range.
- **Signed return conversion:** interpretation of calculated unsigned values above 2,147,483,647.
- **Calling convention:** argument placement, return transport, register preservation, and processor-flag effects.
- **Caller guarantees:** whether callers constrain cursor values, masks, group indices, or hidden entry state.
- **Application meaning:** whether the result denotes a previous set-bit position, an index, or another domain quantity.

The inferred grouped-position interpretation must not replace the observed full-cursor calculation. In particular, the use of an eight-bit shift count does not normalize the cursor to a 32-position group or to an eight-bit value.

## Evidence

Established static evidence is limited to the supplied representation: four explicit scalar arguments, an initially unassigned condition, a condition-controlled decrement-and-inspect loop, and a final scalar calculation. No calls or explicit memory accesses appear.

The representation directly supports:

- Entry-condition-dependent searching.
- Decrement before every inspection.
- Unsigned full-cursor wraparound specifically when decrementing zero.
- Shift-count formation from the updated cursor’s low eight bits.
- Continuation after a clear inspected bit and termination after a set inspected bit.
- Use of the full final cursor in the result.
- Absence of an explicit exhaustion check.

The distinction between cursor wraparound and a low-eight-bit transition follows from those separate operations. A shift count of zero permits full cursors such as zero and 256; decrementing both produces shift count 255, but only decrementing zero wraps the full cursor.

The grouped, one-based-position interpretation is inferred from multiplication by 32 and addition of one. Hidden-condition provenance and target-specific boundary behavior are not established.

No implementation access, runtime execution, or runtime tests have been performed.
