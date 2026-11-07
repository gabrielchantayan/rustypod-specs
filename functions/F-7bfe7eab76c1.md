<!-- Trusted provenance: F-7bfe7eab76c1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7bfe7eab76c1

## Purpose

F-7bfe7eab76c1 selects three 32-bit values and writes them to a target reached through a caller-supplied context.

Speeds of 13,500 and 108,000 select separate groups of lookup tables. Every other speed invokes a diagnostic dependency and then uses three previously captured fallback values. Unsupported speed alone does not produce a failure return.

The established behavior is initialization calls, value selection, and ordered storage. The meanings of the values, validation marker, selectors, and target fields are not established. Describing the operation as configuration programming is inferred intent, not evidence of a particular device or protocol.

This contract describes static evidence only. It does not assign portable behavior to arithmetic overflow, invalid memory access, or unknown dependency behavior.

## Interface

### Inputs

| Position | Contract name | Representation and role |
|---|---|---|
| 1 | Context | Nullable pointer to storage accessed in 32-bit words. |
| 2 | First selector | Signed 32-bit integer selecting the first output value. |
| 3 | Third selector | Signed 32-bit integer selecting the third output value. |
| 4 | Second selector | Signed 32-bit integer selecting the second output value. |
| 5 | Speed | Signed 32-bit integer compared with 13,500 and 108,000; otherwise supplied to the diagnostic dependency. |

Selector names refer to output fields, not argument order.

### Context layout

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Validation word, compared with a global marker. |
| 56 | 32 bits | Target address, captured as a signed 32-bit integer. |

The represented context layout uses four-byte word spacing. It does not establish a portable native-pointer layout or a wider target-address representation.

A nonnull context is not necessarily readable. No bounds, alignment, lifetime, or ownership check is shown.

### Target layout

| Byte offset | Width | Value |
|---|---:|---|
| 40 | 32 bits | Third selected or fallback value |
| 44 | 32 bits | First selected or fallback value |
| 48 | 32 bits | Second selected or fallback value |

The represented store order is **44, 48, 40**.

These offsets are added to a captured **signed 32-bit address value** before the resulting values are used as addresses. Access width and storage validity do not, by themselves, establish that this arithmetic has defined behavior.

### Return value

The represented result is 32 bits:

- **7:** context is null, or its validation word differs from the observed global marker.
- **0:** execution reaches and completes the three stores and the normal return.

No separate return code is shown for unsupported speed, negative selectors, arithmetic overflow, invalid storage, or dependency failure. Such conditions must not be reinterpreted as graceful failure returns.

## Behavior

### 1. Invoke initialization dependencies in order

Before validating the context, the function invokes F-U1 six times. F-U1 is a document-local opaque dependency ID; no canonical opaque ID was supplied for this helper.

Each call receives a different local destination region and a source operand obtained from a distinct global source-reference location. Each destination has a represented capacity of 20 bytes.

| Call order | Destination’s later role | Explicit extent argument |
|---|---|---|
| 1 | First-value table for 13,500 | 20 |
| 2 | Second-value table for 13,500 | 20 |
| 3 | Third-value table for 13,500 | 20 |
| 4 | First-value table for 108,000 | 20 |
| 5 | Second-value table for 108,000 | 20 |
| 6 | Third-value table for 108,000 | Not visibly supplied |

**Source-reference identity is distinct from its observed value.** The six global locations are distinct, but their contents are unavailable. Their values need not be distinct.

Each call evaluates its own global source-reference expression when that call is reached:

- The first source operand is evaluated before the first invocation.
- Each subsequent source operand is evaluated after the preceding invocation returns and before its own invocation begins.

The source does not capture all six operands before the first call. F-U1 is not established to preserve globals. Therefore an earlier invocation may affect the value observed for a later source operand. Later calls must not be specified as necessarily receiving entry-time values.

No ordering among unrelated argument evaluations within one call is established.

The pattern suggests local-table initialization from static data, but copying semantics are not proven. The sixth call does not visibly supply an extent; it must not be treated as an established 20-byte copy. Neither complete initialization nor a write extent confined to the destination’s capacity is guaranteed by the supplied evidence.

All six calls precede even the null-context rejection.

### 2. Capture fallback values

After the sixth initialization call returns, three 32-bit global values are read in this order:

1. Third fallback value.
2. Second fallback value.
3. First fallback value.

The function retains these observed values for possible later use. It does not defer their reads until the target stores.

Initialization calls may have changed these globals. The captured values are consequently not guaranteed to equal their entry-time values.

### 3. Validate the context

A null context returns 7 without directly reading its validation word.

For a nonnull context, the first 32-bit word is compared with a global validation marker. A mismatch returns 7.

Both rejection paths occur after all six initialization calls and the three fallback reads. Neither rejection path reads the target-address field, invokes the unsupported-speed diagnostic, selects table entries, or performs the three explicit target stores.

The marker is observed during validation, not captured before initialization. The relative machine-level read order of the context word and marker within their comparison is not established.

An unreadable nonnull context has no specified graceful-failure outcome.

### 4. Normalize selectors

After successful validation, signed comparisons and replacements occur in this order:

1. Third selector: values greater than 2 become 0.
2. Second selector: values greater than 4 become 0.
3. First selector: values greater than 4 become 0.

Negative values are unchanged.

Thus ordinary in-table selector ranges are 0 through 4 for the first and second tables, and 0 through 2 for the third table. Large positive signed values select entry 0 after normalization.

These checks do not validate negative indices or establish safe arithmetic for every retained value.

### 5. Capture the target address

After normalization and before speed dispatch, the function reads the signed 32-bit target-address value from context offset 56.

It does not check this value for nullness, alignment, writable extent, or arithmetic suitability. All three later stores use this captured value. The function does not reload it after the diagnostic dependency returns.

### 6. Select values

#### Speed 13,500

The represented selection order is:

1. First-value table entry selected by the normalized first selector.
2. Second-value table entry selected by the normalized second selector.
3. Third-value table entry selected by the normalized third selector.

All selected values are 32 bits.

The first two tables are represented as five four-byte entries. The third table is addressed by a byte displacement computed as the third selector multiplied by four.

#### Speed 108,000

The same selection order and widths apply to the other three tables.

The third-value table is the destination of the sixth initialization call. Its readable, initialized contents are not established solely by that invocation.

#### Every other speed

The function invokes F-1b7fa166ef25 with the diagnostic-format reference associated with unsupported speed and the original signed speed value.

No dependency return value controls subsequent behavior. If the dependency returns normally, the function proceeds with the three fallback values captured earlier.

No speed-selection table reads occur on this branch. In particular, negative selectors do not cause table indexing or third-selector multiplication here.

The call establishes a diagnostic request, not actual output, its destination, or exact formatting.

### 7. Perform address arithmetic and stores

For either supported speed, all three selected values are obtained before target stores begin. For unsupported speed, the diagnostic call precedes the stores.

The represented sequence is:

1. Add 44 to the captured signed 32-bit address value and store the first value.
2. Add 48 to that same captured value and store the second value.
3. Add 40 to that same captured value and store the third value.
4. Return 0.

No rollback, completion flag, or checked write result is shown.

### Arithmetic limits

The third-table displacement uses **signed 32-bit multiplication by four**. Positive selectors have already been reduced to 0, 1, or 2, but negative selectors remain unrestricted.

For example:

- −1 gives the mathematically representable displacement −4, which points before the established table region.
- −536,870,912 gives the representable displacement −2,147,483,648, still outside that region.
- −536,870,913 would require −2,147,483,652, outside the signed 32-bit range.

The target-address additions likewise operate on a signed 32-bit value. To keep all three mathematical sums within that range, the captured value must be no greater than **2,147,483,599**. This bound alone establishes neither a usable address nor writable storage. A negative represented address is not independently rejected.

The supplied representation does **not** establish portable overflow behavior, widening before arithmetic, or wraparound. It also does not establish a portable interpretation of every signed integer converted to an address.

Negative first and second selectors select before their established array regions. Their array indexing must not be converted into an asserted portable overflow or wraparound rule.

## Side effects

### Direct and dependency-mediated effects

The function supplies six local destination addresses to F-U1. Their represented capacities do not prove the helper’s actual write extents.

On a normally completed valid-context path, the function explicitly performs three four-byte stores in the stated order. No direct update to the context validation word or target-address field is otherwise shown. Target aliasing with context or other storage is not ruled out.

Dependencies are not established to preserve globals, caller-owned storage, local storage whose address they receive, or other external state. In particular:

- Earlier initialization calls may change later source-reference operands, fallback globals, or the validation marker.
- The diagnostic may change the context, target storage, or fallback globals before returning.
- Changes to the context’s target field or fallback globals during the diagnostic do not replace the target address and fallback values already captured by this invocation.

The stores are not represented as a transaction. An access failure may leave earlier effects without any shown rollback. Where signed overflow occurs, however, no portable prefix of execution is guaranteed by the supplied evidence.

### Ownership and lifetime

No explicit allocation, release, or ownership-transfer operation appears in the function body. This does not establish that dependencies allocate nothing, release nothing, or retain no addresses.

Local destination storage is not returned. Any local address retained by F-U1 must not be assumed valid after the function returns.

### Preconditions for normal completion

Normal completion requires more than initialized tables and writable target storage:

- Every dependency invoked on the chosen path returns normally.
- Required context and global reads are valid.
- Context validation succeeds as observed at comparison time.
- Supported-speed table reads use valid, initialized storage.
- Relevant index, displacement, address-addition, and address-conversion operations have defined behavior in the execution environment.
- The three effective target addresses support the required four-byte stores.
- No dependency or concurrent action invalidates these conditions before their use.

The function does not comprehensively check these requirements.

## Dependencies

### F-U1

**Identity:** document-local opaque ID for the initialization helper; canonical mapping unavailable.

**Established use:** six ordered calls, each with a local destination and a separately evaluated global source-reference operand. The first five visibly include extent 20; the sixth does not.

**Inferred intent:** initialization of local lookup tables.

**Unknowns:** full interface, effective sixth-call arguments, source interpretation, copy behavior, write bounds, initialization guarantees, return behavior, retained addresses, ownership effects, and other mutations.

### F-1b7fa166ef25

**Established use:** one call for an unsupported speed after successful context validation and target-address capture.

**Arguments:** diagnostic-format reference and original signed speed value.

**Result handling:** no returned value is used.

**Unknowns:** exact output, destination, formatting, failure behavior, normal-return guarantee, mutations, and ownership effects.

### Data dependencies

The function uses six global source-reference locations, three fallback words, a validation marker, and a diagnostic-format reference. Their contents are unavailable. Numeric table and fallback outputs remain symbolic.

## Acceptance examples

These are conditional static criteria, not runtime test results.

### Null context

Given a null context and normally returning initialization calls:

- All six initialization calls occur, with each source operand evaluated at its own call boundary.
- The fallback globals are read in third, second, first order.
- No context word is directly read.
- No diagnostic or explicit target store occurs.
- The function returns 7.

### Initialization changes a later source operand

Suppose the first initialization call changes the global location used as the second call’s source-reference operand and returns normally.

The second call receives the value observed when its own source expression is evaluated, after the first call returns. Capturing all six operands before the first call would not preserve this ordering.

This example does not assert that F-U1 actually performs such a mutation.

### Validation mismatch

Given a readable nonnull context whose validation word differs from the observed marker:

- Initialization and fallback capture have already occurred.
- The target address is not read.
- No table selection, diagnostic, or explicit target store occurs.
- The function returns 7.

### Supported speed with ordinary selectors

Given speed 13,500, first selector 1, third selector 2, and second selector 4:

- Selection uses entries 1, 4, and 2 of the corresponding tables.
- Stores occur at offsets 44, 48, and 40.
- Return is 0, conditional on valid initialization, accesses, arithmetic, address interpretation, and normal dependency returns.

Actual numeric values are unknown.

### Positive out-of-range selectors

Given speed 108,000 and first, third, and second selectors of 5, 3, and 5, all become 0.

Selection uses entry 0 of each corresponding table, subject to the same completion conditions. The sixth initialization call alone does not prove that the third-table entry is initialized.

### Unsupported speed and diagnostic mutation

Given successful validation and an unsupported speed:

- The target address and fallback values are captured before the diagnostic.
- No speed-selection table entry is read.
- After normal diagnostic return, the captured values are stored using the captured address.
- Later changes to the fallback globals or context target field do not replace those captured values.

A return of 0 additionally requires defined target arithmetic and successful stores.

### Negative selector and multiplication overflow

For a supported speed, third selector −1 remains negative and produces a displacement before the table. No valid result or graceful error is established.

Third selector −536,870,913 additionally exceeds the representable range when multiplied by four. No wrapping displacement is specified.

For an unsupported speed, neither selector causes a table read or that multiplication.

### Target-address overflow boundary

A captured target-address value of 2,147,483,600 permits a mathematical addition of 44 within the signed 32-bit range, but addition of 48 exceeds that range.

Even if the corresponding storage is otherwise writable, the source does not establish portable normal completion, wraparound, or a guaranteed partial-write result. Storage validity alone is insufficient.

## Uncertainties

The following remain **documented_only**:

- Table contents, fallback values, marker value, and source-reference values.
- Whether source-reference values coincide or change during initialization.
- F-U1’s complete interface and effective sixth-call behavior.
- Dependency mutations, ownership effects, retained addresses, and return guarantees.
- Negative supported-speed indexing behavior outside established table regions.
- Overflow handling, widening, wraparound, and integer-to-address interpretation.
- Alignment, byte order, invalid-access behavior, and partially writable storage behavior.
- Concurrency, synchronization, atomicity, and external visibility guarantees.
- Whether target fields are ordinary memory, device registers, or another interface.
- Intended meanings and units of speed constants and selected values.

No hardware effect, persistence guarantee, or ownership transfer is inferred from the stores.

## Evidence

Established static evidence consists of the supplied representation’s call order, per-call global source expressions, fallback-read order, validation, signed selector comparisons, 32-bit widths, table-selection order, signed displacement and target-address arithmetic, store order, and return values.

Table initialization and configuration programming are inferred intent. They do not establish dependency semantics, table contents, or portable behavior for operations outside their represented arithmetic and storage domains.

No dependency implementation, static-data contents, runtime trace, or test result was supplied. **No runtime tests have been performed.**
