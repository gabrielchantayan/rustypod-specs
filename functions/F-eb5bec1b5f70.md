<!-- Trusted provenance: F-eb5bec1b5f70; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eb5bec1b5f70

## Purpose

F-eb5bec1b5f70 performs three initial word writes, then enters an unconditional loop of indirect calls, memory accesses, arithmetic transformations, and a software breakpoint.

The supplied static representation establishes no normal return path and no application-level purpose. Descriptive terms such as “base,” “field,” and “selector” identify relationships between values; they do not establish an object model.

All behavior is conditional on execution reaching the relevant operation. A dependency, invalid access, arithmetic condition, or breakpoint handler may prevent later operations.

## Interface

### Explicit inputs

- One 32-bit value with no visible use.
- One signed 32-bit index used in the initial address and value calculations.

No input validation, rejection, or explicit error result is visible.

### Implicit inputs

Execution additionally depends on:

- An entry global used both as an address base and as stored data.
- Two globals read during each iteration.
- Incoming link-like state used to form the first indirect-call target.
- Frame-like state used for memory accesses and decreased during each completed iteration.
- An incoming auxiliary register value used in a byte-store destination.
- Auxiliary results associated with particular indirect calls.

The two explicit arguments are therefore insufficient to define a standalone invocation environment.

### Outputs and representation

There is no visible explicit return value or normal return operation.

Direct memory accesses are 8, 16, or 32 bits wide. “Byte,” “halfword,” and “word” below mean those respective widths. Unsigned halfword reads are zero-extended; halfword writes retain the low 16 bits of their supplied value.

Three dependencies produce represented 64-bit results, each subsequently divided into lower and upper 32-bit halves. This does not establish the actual return convention.

When used as signed operands, a pair’s halves have signed 32-bit interpretations. When explicitly used as unsigned operands, they have unsigned 32-bit interpretations. This distinction is material to F-d006: its second argument has a signed intermediate addition but an unsigned final expression.

Byte order, alignment requirements, executable-address conventions, and address-space mapping are unresolved.

### Preconditions and arithmetic

Successful continuation requires valid reached memory operands and indirect targets. No bounds checks, lifetime checks, or recovery path are visible.

Explicit unsigned 32-bit operations have their represented unsigned behavior. Signed multiplication, addition, and left shifts do not establish portable wraparound. This qualification applies to index scaling, address calculations involving signed values, F-d006’s intermediate addition, scaling F-d019’s result, and arithmetic involving the third pair.

In particular, a later conversion to unsigned does not remove an earlier signed-overflow condition.

## Behavior

### Dependency notation

F-d001 through F-d037 are opaque identifiers for call sites in iteration order. Some sites have provably equal indirect targets, listed under **Dependencies**.

“No explicit arguments shown” does not mean a dependency consumes no implicit state. No dependency is assumed to preserve globals, registers, memory validity, or ownership.

### Initial accesses

Before entering the loop:

1. Read the entry global and retain its value as the captured entry base.
2. Compute the signed index multiplied by 512.
3. Perform a word store at a separately read entry-global value plus the index multiplied by 256. The stored value is another reference to the entry global.
4. At the captured entry base plus the previously computed 512-scaled offset, store that offset as a word.
5. At the same captured-base destination, store the captured entry base as a word.

The first store’s address-side and value-side global references are distinct from the initial capture. Their relative evaluation order is not established. They must not be replaced unconditionally with the captured value or treated as mutually ordered reads.

The last two stores use the captured base and already computed offset. Both writes are part of the sequence, even when the latter overwrites the former in ordinary memory.

If all entry-global reads return the same value and arithmetic is valid, index zero makes all three writes target that value. Negative indices are not rejected; valid arithmetic and destinations remain external constraints.

### Loop phase 1: initial calls and fields

Each iteration:

1. Establish a byte-valued zero.
2. Invoke F-d001 with two explicit zero arguments.
3. Establish a second byte-valued zero.
4. Invoke F-d002, then F-d003.
5. Read the first per-iteration global and retain it as the first record base.
6. Invoke F-d004.
7. Invoke F-d005, obtaining the first represented 64-bit pair.
8. Invoke F-d006 with two explicit arguments:
   - The pair’s lower half interpreted as signed 32-bit.
   - The result of the arithmetic described below.
9. Store one byte from an auxiliary result associated with F-d006 at the retained record base plus 17.
10. Invoke F-d007.
11. Store the first established zero byte at that same base plus 21.
12. Invoke F-d008, obtaining a 16-bit result and an auxiliary address-bearing result.
13. Store the 16-bit result at the auxiliary address plus 12.
14. Invoke F-d009, obtaining the second represented 64-bit pair.

**F-d006’s second argument has two distinct arithmetic stages.** First, add the first pair’s upper and lower halves as signed 32-bit values. That intermediate addition requires the signed-overflow qualification. Next, add the second byte-valued zero, explicitly converted to unsigned 32-bit. This converts the signed intermediate to unsigned 32-bit before the final addition. The final argument expression is therefore unsigned 32-bit.

For a representable signed intermediate, adding unsigned zero leaves its unsigned conversion unchanged. A negative intermediate consequently becomes its corresponding unsigned value modulo \(2^{32}\). This is an unsigned conversion, not a guarantee of signed wraparound. If the initial signed addition overflows, the final unsigned operation does not establish a portable result.

The record base is captured before F-d004. The two subsequent byte stores use that retained value rather than visibly reloading the global.

### Loop phase 2: second pair and breakpoint preparation

Using the second pair:

1. Store a zero halfword at its signed upper-half address plus 44.
2. Store its unsigned lower half shifted right by 26 as a word at the frame-like base plus 620.
3. Invoke F-d010.
4. Read a word at the frame-like base plus 800 and retain it as a destination address.
5. Invoke F-d011, then F-d012.
6. Store the second pair’s unsigned lower half shifted left by two at the retained destination plus 84.
7. Invoke F-d013, F-d014, F-d015, F-d016, and F-d017, in order.
8. Read an unsigned halfword at fixed address 0x21C48994.
9. Invoke F-d018.
10. Read a word at the frame-like base plus 640 and supply it to F-d019.

The right-shifted value ranges from 0 through 63. The left-shifted value is the low 32 bits of four times the unsigned lower half.

The fixed-address halfword is subsequently used directly as a zero-extended address base. No larger address base is visibly combined with it.

### Breakpoint boundary

Immediately after F-d019, execute a software breakpoint with immediate 0x92.

Later operations are represented, but their execution depends on breakpoint handling. The environment may stop, trap, resume, or transfer control elsewhere. No particular outcome is established.

### Loop phase 3: captured bases and halfword stores

If execution continues after the breakpoint:

1. Shift F-d019’s signed 32-bit result left by two.
2. Derive an offset by unsigned-shifting 0x21C48960 right by 18 and multiplying by four, producing 0x21C4.
3. Read a word at the earlier fixed-address halfword’s zero-extended value plus 68. Retain it as the later destination base.
4. Invoke F-d020.
5. Invoke F-d021 and retain its 32-bit result.
6. Read a word at the same zero-extended halfword value plus 4.
7. Invoke F-d022 with F-d021’s result and that loaded word.
8. Read the second per-iteration global and retain its value.
9. Invoke F-d023.
10. At that captured global value plus the scaled F-d019 result, store the low 16 bits of the captured global value.
11. Invoke F-d024.
12. Read a word at fixed address 0x2218 and retain it as an address.
13. Invoke F-d025.
14. Store halfword 0x21C4 at F-d025’s returned address plus 4.
15. Invoke F-d026, then F-d027.

The address 0x2218 is the derived offset 0x21C4 plus 84. The word loaded there supplies a later address; it is not the halfword stored after F-d025.

The store after F-d023 uses the captured global value for both destination formation and stored-value truncation, even if F-d023 changes that global. Storage validity is a separate requirement.

### Loop phase 4: third pair and frame-like adjustment

Next:

1. Read an unsigned halfword at offset 52 from the address loaded at 0x2218. Retain it as the selector.
2. Invoke F-d028, obtaining the third represented 64-bit pair.
3. Store the pair’s lower half as a word at the current frame-like base plus 600.
4. Read an unsigned halfword at the pair’s signed upper-half address plus 20. Retain it as the field value.
5. Form a displacement by shifting the pair’s signed lower half left by six.
6. Form a scaled address by multiplying its signed upper half by four.
7. Decrease the frame-like base by 244.
8. Form four times the zero-extended selector.
9. Store the third pair’s lower half as a word at address 116 plus four times the selector.
10. Invoke F-d029.
11. Store the zero-extended field value as a word at the later destination base plus 84.
12. Invoke F-d030.
13. Store the same field value as a word at that base plus 100.
14. Invoke F-d031.
15. Store the low 16 bits of the later destination base at the six-bit-shifted displacement plus an auxiliary result associated with F-d031.

The selector ranges from 0 through 65,535. Its fourfold scaling ranges from 0 through 262,140; the direct word-store address therefore ranges from 116 through 262,256.

The field value also ranges from 0 through 65,535. Both word stores preserve that zero-extended value.

The frame-relative store precedes the decrement. No balancing restoration or loop-entry reset is visible. A subsequent iteration uses the represented adjusted state; the adjustment’s architectural meaning remains unresolved.

### Loop phase 5: byte transfer and repetition

The iteration finishes with:

1. Invoke F-d032, then F-d033.
2. Read one byte at five times the selector.
3. Invoke F-d034.
4. Store that byte at the field value plus the incoming auxiliary register value plus 25.
5. Invoke F-d035.
6. Store the scaled address derived from the third pair’s upper half as a word at the later destination base plus 68.
7. Invoke F-d036.
8. Set the link-like target-forming value to 0x08712031.
9. Read a word at the scaled address plus 68 and supply it to F-d037.
10. Begin another iteration.

The byte-read address ranges from 0 through 327,675. No hidden base is visible.

On the first iteration, F-d001’s target is the incoming link-like value plus 0xFFE. The tail assignment makes the represented next-iteration target 0x0871302F. Reaching that call remains conditional on F-d037 returning and the target-forming state being available under the actual calling convention.

## Side effects

Direct effects include:

- Three initial word writes.
- Per-iteration byte, halfword, and word reads and writes.
- Fixed-address and computed-address accesses.
- Thirty-seven indirect calls per fully completed iteration.
- A software breakpoint.
- A frame-like base decrement.
- Replacement of the first-call target-forming state.

No direct allocation, release, ownership transfer, synchronization, transaction, rollback, or explicit error-reporting mechanism is identifiable.

Dependencies may nevertheless allocate, release, mutate globals, invalidate retained storage, or fail to return. Capturing an address establishes neither ownership nor continued validity.

Writes are not established as an atomic group. Aliasing, overlapping global storage, concurrency, and device memory may expose intermediate effects. No global-stability guarantee is implied.

## Dependencies

### Established equal-target groups

| Call sites | Established relationship |
|---|---|
| F-d005 and F-d006 | Equal indirect target |
| F-d009 and F-d013 | Equal indirect target |
| F-d017 and F-d024 | Equal indirect target |
| F-d019 and F-d031 | Equal indirect target |
| F-d025 and F-d032 | Equal indirect target |

These relationships follow from fixed target arithmetic. They do not imply equal arguments, results, effects, or invocation state.

### Visible interfaces

| Call sites | Explicitly represented interface |
|---|---|
| F-d001 | Two zeros; target formed from link-like state plus 0xFFE |
| F-d002–F-d004 | No explicit arguments shown |
| F-d005 | First 64-bit pair |
| F-d006 | Signed lower-half argument; unsigned 32-bit second argument formed by signed addition of both halves, followed by unsigned conversion and addition of unsigned zero; auxiliary byte result |
| F-d007 | No explicit arguments shown |
| F-d008 | Halfword result and auxiliary address |
| F-d009 | Second 64-bit pair |
| F-d010–F-d018 | No explicit arguments shown |
| F-d019 | Frame-relative word argument; signed 32-bit result subsequently scaled |
| F-d020 | No explicit arguments shown |
| F-d021 | Word result used by F-d022 |
| F-d022 | F-d021’s result and a memory-loaded word |
| F-d023–F-d024 | No explicit arguments shown |
| F-d025 | Address result used for a halfword store |
| F-d026–F-d027 | No explicit arguments shown |
| F-d028 | Third 64-bit pair |
| F-d029–F-d030 | No explicit arguments shown |
| F-d031 | Auxiliary address component |
| F-d032–F-d036 | No explicit arguments shown |
| F-d037 | Word read from the scaled address plus 68 |

F-d006’s signed intermediate remains subject to overflow qualification despite the final expression’s unsigned type. Concrete dependency identities and semantics remain unknown.

## Acceptance examples

These are conditional static consequences, not executed tests. Each assumes valid reached accesses, applicable arithmetic conditions, and continuation through relevant dependencies.

1. **Stable entry global, zero index.** If every entry-global reference yields the same valid base, the three word writes target that base and contain, in order, the base, zero, and the base. Final contents alone do not demonstrate the sequence.

2. **Stable entry global, index one.** The first store writes the base at base plus 256. The next two write 512 and then the captured base at base plus 512.

3. **Different entry-global observations.** If the initial capture yields 0x1000, the first store’s address-side reference yields 0x2000, its value-side reference yields 0x3000, and the index is one, the first store writes 0x3000 at 0x2100. The subsequent stores write 0x200 and then 0x1000 at 0x1200. This does not prescribe the relative order of the first store’s two references.

4. **F-d006 with a negative, representable intermediate.** If the first pair’s signed upper half is −2 and signed lower half is 1, the intermediate signed sum is −1. F-d006’s first argument is 1; its second argument expression is unsigned 4,294,967,295 after conversion and addition of unsigned zero.

5. **F-d006 with signed overflow.** If the signed upper half is 2,147,483,647 and the signed lower half is 1, the intermediate signed addition overflows. This contract does not require a particular final argument value. The later unsigned zero does not justify treating the intermediate as wrapping.

6. **Unsigned second-pair shifts.** A lower half of 0xFC000003 produces 63 at the frame-like base plus 620 and 0xF000000C at the retained destination plus 84.

7. **Halfword widening.** A field value of 0xFFFF produces word stores containing 65,535 at the later destination base plus 84 and plus 100, not a sign-extended negative value.

8. **Selector addressing.** A selector of seven produces direct word-store address 144 and byte-read address 35.

9. **Truncating a captured base.** A later destination base of 0x12345678 supplies halfword 0x5678 to the store after F-d031, provided its separately formed destination is valid.

10. **Equal target, separate calls.** F-d005 and F-d006 resolve to the same target. Their different argument presentations do not establish identical results or effects.

11. **Next-iteration target.** If execution reaches the tail assignment and continues through F-d037 with the represented target-forming state, the next F-d001 target is 0x0871302F, regardless of the incoming first-iteration value.

12. **No breakpoint resumption or invalidated storage.** If breakpoint handling does not resume, no subsequent write from that iteration is required. If a dependency invalidates a retained destination, no visible recovery guarantees the later access. Earlier completed effects are not visibly rolled back.

## Uncertainties

### Established static evidence

The representation establishes access widths, operation ordering, constants, unsigned extensions and shifts, truncations, fixed-target equalities, the explicit tail assignment, the unconditional loop, and absence of a visible normal return.

For F-d006, it establishes signed addition of the two halves before the unsigned-zero operand participates. The intermediate is signed 32-bit; the final argument expression is unsigned 32-bit.

### Inferred intent

The accesses may concern records, tables, or a specialized calling convention. These interpretations are not required behavior. They do not authorize adding hidden bases, validation, ownership rules, or recovery.

### documented_only

Unresolved matters include:

- Concrete callee identities and semantics.
- Implicit arguments, auxiliary-result conventions, and state preservation across calls.
- Breakpoint handling and resumability.
- Outcomes of overflowing signed arithmetic and signed shifts outside valid conditions.
- Relative evaluation order of the first store’s address-side and value-side global references.
- Stability of entry-global observations.
- Byte order, alignment, executable-address rules, and memory mapping.
- Architectural meaning of the frame-like adjustment.
- Aliasing, concurrency, overlapping globals, and device-memory behavior.
- Dependency ownership and lifetime effects.
- Whether unusual accesses reflect actual machine behavior or limitations of the supplied representation.

These unknowns remain unresolved rather than being replaced with fallback behavior or preservation assumptions.

## Evidence

This contract is based on the supplied static representation of F-eb5bec1b5f70. Prior prose and reviewer feedback are not independent evidence.

The representation independently supports the F-d006 distinction: its two pair halves enter a signed 32-bit addition first; an explicitly unsigned 32-bit zero participates afterward, making the final expression unsigned without defining an overflowing signed intermediate.

Dependency identifiers are opaque call-site references. No implementation access, runtime inspection, or runtime tests were performed. Acceptance examples describe conditional consequences, not observed execution.
