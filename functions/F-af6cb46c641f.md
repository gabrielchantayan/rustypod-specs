<!-- Trusted provenance: F-af6cb46c641f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-af6cb46c641f

## Purpose

F-af6cb46c641f selects a formatting path from a 32-bit selector and, when that selector is nonzero, from a comparison between two dependency-derived 32-bit values. It passes a caller-supplied destination to a formatting dependency with a size argument of 40 bytes.

One nonzero-selector path additionally replaces each byte of the resulting zero-terminated destination through a lookup table.

The result is a 64-bit value. Its meaning is path-dependent: some paths combine a local state word with a formatter result, one path explicitly supplies a zero lower half, and the zero-selector path may return an entire dependency result. It must not be treated as a uniform Boolean, byte count, pointer, or error code without additional dependency contracts.

These are established structural behaviors. The semantic identities of the formatted values, the format assets, and the lookup table are unknown.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Width or representation | Established use |
|---|---|---|
| Context | 32-bit opaque value | Forwarded to dependencies; used in both major branches. |
| Selector | Unsigned 32-bit value | Determines the initial branch and initializes the first word of a local record. |
| Destination | Pointer represented in a 32-bit pointer slot | Forwarded to formatting dependencies; directly read and written by the byte-translation path. |
| Auxiliary value | 32-bit opaque value | Initializes the third word of the local record used by the nonzero-selector branch. |

The context may represent an address, handle, or encoded value. Its meaning is not established. The auxiliary value likewise has no established independent interpretation.

### Local storage exposed to dependencies

Two distinct local storage areas matter:

- A 24-byte scratch area used only by the zero-selector branch. It is not explicitly initialized before being passed to a dependency.
- A contiguous, three-word local record initialized as follows:

| Byte offset | Width | Initial contents |
|---|---:|---|
| 0 | 4 bytes | Selector |
| 4 | 4 bytes | Destination pointer representation |
| 8 | 4 bytes | Auxiliary value |

The nonzero-selector branch passes the address of this record to dependencies. The record’s first word is subsequently used in the upper half of the function’s result.

The initial selector determines the branch before any dependency can modify the record. Later changes to the record do not cause the initial branch decision to be repeated.

### Output

The function returns 64 bits. “Upper half” and “lower half” below mean bits 63–32 and bits 31–0 respectively, independent of memory byte order.

The destination may receive formatted bytes and, on one path, further in-place byte replacements. A size argument of 40 is supplied to each formatting dependency. The available evidence does not establish whether this is an enforced capacity, whether a terminator is always produced, or how truncation is reported.

### Preconditions and ownership

For defined successful operation:

- Each dependency must accept the values and storage passed to it.
- The destination must meet the selected formatter’s access requirements.
- On the translation path, the destination must be readable and writable through its first zero byte.
- The selected lookup-table base must support every byte index encountered during translation.
- Dependencies receiving local storage must not retain it for use after the function returns unless an external contract supplies a valid mechanism for doing so.

No ownership transfer is explicitly expressed by this function. It does not directly allocate or release memory. The allocation, retention, and release behavior of dependencies is unknown.

## Behavior

### 1. Initial state and branch selection

Before selecting a path, the function initializes the three-word local record from the selector, destination, and auxiliary arguments.

It then tests the original selector:

- Zero selects the scratch-based path.
- Any nonzero 32-bit value selects the record-based path.

The function does not validate the destination or context before invoking dependencies.

### 2. Zero-selector path

The function calls F-80d53d1c9266 with the context and the address of the 24-byte scratch area.

It examines the lower 32 bits of that dependency’s 64-bit result.

**If the lower half is zero:**

- No formatting dependency is called.
- The function returns the dependency result’s upper half, with the lower half forced to zero.

Because the tested lower half was already zero, the returned 64-bit pattern equals the dependency’s returned pattern on this path.

**If the lower half is nonzero:**

- The function calls F-2280ddcb8671 with:
  - the original destination;
  - the integer size argument 40;
  - a fixed formatting asset specific to this path;
  - the address of the scratch area.
- The function returns the entire 64-bit result of F-2280ddcb8671.

The original result from F-80d53d1c9266 is not otherwise preserved in the return value after this formatter call.

Neither the record initialization dependency nor the comparison dependencies are invoked on this path.

### 3. Nonzero-selector preparation

For a nonzero selector, the function performs these calls in order:

1. F-3b3feb156f36 receives the local record’s address. Its result is ignored.
2. F-3cd57bab8f84 receives the record’s address and the context.

The result of F-3cd57bab8f84 is interpreted as a signed 32-bit integer.

If that result is negative, the function skips both value-extraction calls and selects the fallback formatting path.

If it is zero or positive, the function obtains two values in this order:

1. F-b8274406a146 receives the context and the integer argument 1.
2. F-b8274406a146 receives the local record’s address and the integer argument 1.

The first call completes before the second begins. Consequently, effects of the first call may be visible to the second. No assumption that either call preserves global state is justified.

### 4. Difference and range selection

The function forms a 32-bit difference from the second extracted value minus the first extracted value, and uses the resulting unsigned value for classification.

The behavioral contract is based on that 32-bit result. It does not promote the operation to a signed or unsigned 64-bit difference. Under the represented 32-bit arithmetic, a mathematically negative difference has a large unsigned representation rather than entering a special “negative interval” case.

The explicit conditions establish these contiguous ranges:

| Unsigned 32-bit difference | Selected path |
|---|---|
| 0 through 86,399 | First-range formatting |
| 86,400 through 604,799 | Second-range formatting followed by translation |
| 604,800 through 4,294,967,295 | Fallback formatting |

The comparisons internally partition the first two ranges at 81,920 and 589,824, but those internal boundaries do not change the selected path.

These numeric intervals are established. Interpreting them as elapsed seconds, calendar periods, or any other unit would be inferred intent, not an established interface property.

### 5. First-range and fallback formatting

The first-range and fallback paths share a calling pattern but use distinct fixed formatting assets.

The function calls F-eb6afeb2311e with:

- the original destination argument;
- the integer size argument 40;
- the asset selected for this path;
- the original context argument.

It then constructs its 64-bit result from:

- **Upper half:** the current first word of the local record.
- **Lower half:** the 32-bit result of F-eb6afeb2311e.

The first word is read after the formatter returns. It is not necessarily equal to the input selector: earlier dependencies received access to the record. The evidence also does not justify assuming that unrelated dependencies cannot affect that storage through aliases or retained references.

The fallback asset is selected either when the preparation result is negative or when the unsigned difference is at least 604,800.

### 6. Second-range formatting and translation

For a difference from 86,400 through 604,799, the function first calls F-eb6afeb2311e using:

- the original destination argument;
- the integer size argument 40;
- a third fixed formatting asset, distinct from the first-range and fallback assets;
- the original context argument.

The formatter’s return value is ignored.

After the formatter returns, the function reads a global value that supplies the byte-translation table’s base address. That base is obtained once for the translation pass.

Starting at the original destination address, the function then:

- Reads the current destination byte.
- Stops if that byte is zero.
- Otherwise, uses the nonzero byte’s unsigned value as an index into the translation table.
- Writes the selected table byte back to that same destination position.
- Advances one byte and repeats.

This pass has several important properties:

- The terminating zero is tested but not replaced.
- There is no explicit 40-byte bound on the translation pass.
- Lookup indices are unsigned byte values, so encountered nonzero input bytes select offsets 1 through 255.
- If a nonzero input byte translates to zero, the function still advances to the next position. The newly written zero does not itself terminate the pass.
- The table base is read after formatting, so a formatter-induced change to the global base can affect this pass.
- The table contents are read during traversal. Their stability and disjointness from the destination are not established.

After translation, the function returns a 64-bit value whose upper half is the current first word of the local record and whose lower half is zero.

## Side effects

### Direct effects

The function directly:

- Initializes its local record.
- Reads the global translation-table base on the second-range path only.
- Reads and overwrites destination bytes on that path.
- Advances its traversal pointer without modifying a caller-owned pointer object.

The local record contains a copy of the destination pointer, but later formatting and translation use the original destination argument. A dependency changing the record’s pointer word does not by itself redirect those later operations.

### Dependency-mediated effects

Dependencies can affect the scratch area, the local record, the destination, global state, or other reachable state according to their own contracts. Those effects cannot be narrowed from the supplied evidence.

In particular:

- A zero lower half from F-80d53d1c9266 suppresses the explicit formatter call, but does not prove the destination or other state remained unchanged.
- A negative preparation result selects fallback formatting; it does not establish that preparation was side-effect-free.
- Ignoring a dependency’s return value does not establish success.
- No rollback or restoration is visible.

### Errors and termination

There is no explicit null check, destination-capacity check, lookup-table validation, or exception recovery.

The function does not provide a uniform error result. The lower half may be a formatter result on some paths and an unconditional zero on another.

Translation terminates only when a zero destination byte is encountered at the current traversal position. A formatter that fails to provide a reachable terminator can therefore expose reads and writes beyond the supplied size argument.

## Dependencies

| Opaque function ID | Established role in this function | Unknown contract details |
|---|---|---|
| F-80d53d1c9266 | Receives context and 24-byte scratch storage; returns 64 bits used for the zero-selector decision. | Scratch layout, result meaning, initialization requirements, side effects. |
| F-2280ddcb8671 | Receives destination, size 40, a fixed asset, and scratch storage; supplies the complete zero-selector formatted-path result. | Formatting syntax, termination, capacity enforcement, errors, ownership behavior. |
| F-3b3feb156f36 | Receives the initialized local record before preparation. | Whether it normalizes, replaces, or otherwise mutates record fields; other effects. |
| F-3cd57bab8f84 | Receives record and context; its signed 32-bit result gates extraction. | Meaning of negative and nonnegative results; record and global mutations. |
| F-b8274406a146 | Called first with context and 1, then with record and 1; supplies values for the difference. | Units, interpretation of argument 1, side effects, valid input representations. |
| F-eb6afeb2311e | Receives destination, size 40, a selected asset, and context; its 32-bit result is retained on first-range and fallback paths. | Output bytes, truncation, terminator guarantees, return semantics, errors. |

Four fixed formatting assets are referenced across the function: one for the zero-selector formatter, plus first-range, second-range, and fallback assets. Their contents and semantic meanings are documented_only unknowns.

## Acceptance examples

These examples are conditional contracts, not records of executed tests. Dependency results and side effects are stipulated only to make each path observable.

1. **Zero selector with a zero lower result.**  
   If F-80d53d1c9266 returns `0x1234567800000000`, the function returns that same pattern. Neither formatting dependency is called by this function.

2. **Zero selector requiring formatting.**  
   If F-80d53d1c9266 returns a value with any nonzero lower half, F-2280ddcb8671 receives the destination, size 40, the zero-selector asset, and the scratch address. If it returns `0xABCDEF0123456789`, that entire value is returned.

3. **Negative preparation result.**  
   With a nonzero selector, if F-3cd57bab8f84 returns a negative signed 32-bit value, neither extraction call occurs. Fallback formatting is selected regardless of what the extraction results might otherwise have been.

4. **First-range boundary.**  
   Following nonnegative preparation, differences of 0, 81,920, and 86,399 all select first-range formatting. A difference of 86,400 selects second-range formatting and translation.

5. **Second-range boundary.**  
   Differences of 589,824 and 604,799 select second-range formatting. A difference of 604,800 selects fallback formatting.

6. **Wrapped negative difference.**  
   If the first extracted value is 10 and the second is 9, the 32-bit difference is `0xFFFFFFFF`. It selects fallback formatting, not first-range formatting.

7. **Record mutation affects the return.**  
   If the record’s first word is 7 when first-range formatting finishes, and that formatter returns `0xFFFFFFFE`, the function returns `0x00000007FFFFFFFE`. The formatter result occupies the lower half without sign extension into the upper half.

8. **Translation can introduce an embedded zero.**  
   Suppose the formatted destination begins with bytes 65, 66, and 0, and the selected table maps 65 to 0 and 66 to 90. The resulting first three bytes are 0, 90, and 0. Translation still processes the second byte after writing zero at the first position.

9. **Empty translated output.**  
   If the destination’s first byte is zero after second-range formatting, the translation pass performs no byte replacement. The return still has a zero lower half, regardless of the ignored formatter result.

## Uncertainties

The following remain documented_only:

- Exact formatting text, argument interpretation, output encoding, and truncation behavior.
- The semantic meaning of context, selector, auxiliary value, and the record as a whole.
- Scratch contents and the dependency’s required scratch extent.
- Whether extracted values represent time or any other measurement.
- Lookup-table contents, lifetime, and alias relationships.
- Dependency access to globals and reachable memory.
- Dependency allocation, ownership, retention, and release behavior.
- Error propagation and caller interpretation of the packed return.
- Behavior when required pointers, storage extents, or dependency-specific representations are invalid.

The interface suggests formatting and byte normalization, but more specific domain intent is not established.

## Evidence

This contract is based solely on static inspection of the supplied function representation and its dependency-ID mapping. Established evidence covers argument widths, local storage layout, branch conditions, call ordering, explicit size arguments, return composition, and the direct translation loop.

Descriptive roles such as “formatting asset,” “preparation,” and “extraction” identify observed usage, not verified dependency implementations.

No runtime tests have been performed. All acceptance examples are conditional predictions from the supplied static evidence, not observed executions.
