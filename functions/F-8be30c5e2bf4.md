<!-- Trusted provenance: F-8be30c5e2bf4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8be30c5e2bf4

## Purpose

F-8be30c5e2bf4 scans an input source under the control of a format stream. It matches literals, processes classified format runs, dispatches selected numeric conversions, and reports successful nonsuppressed conversions. A count-reporting directive writes the accumulated input-consumption count through a destination pointer.

Its established behavior is narrower than a conventional formatted-input interface. Character, string, and bracket-set conversions terminate without conversion success. Some length-qualified integer conversions also terminate without invoking a conversion dependency. Familiar directive spelling does not establish conventional library semantics.

This contract describes static evidence from the supplied representation. Scanner and whitespace terminology expresses inferred intent where noted. No runtime tests have been performed.

## Interface

### Arguments and result

The function receives:

1. An opaque **32-bit input context**, passed to input callbacks and numeric dependencies.
2. An opaque **32-bit format handle**, initially stored in scanner state.
3. A writable **scanner-state address**.

It returns a signed 32-bit value:

- Ordinarily, the number of successful, nonsuppressed dispatched conversions.
- Under specified input-failure conditions, −1.
- No distinct error result for unsupported directives, rejected widths, malformed bracket syntax, or internal status −2.

There are no established null, bounds, callback-validity, or destination-validity checks.

### Scanner-state layout

Offsets are bytes from the supplied state address.

| Offset | Width | Established role |
|---|---:|---|
| 0 | 32 bits | Cursor into four-byte destination-address entries. |
| 4 | 32 bits | Current conversion flags. |
| 8 | 32 bits | Current field width. |
| 12 | 32 bits | Format handle/cursor storage. |
| 16 | 32 bits | Auxiliary state, initialized to zero. |
| 20 | 32 bits | Format-reader address. |
| 24 | 32 bits | Externally supplied input-reader address. |
| 28 | 32 bits | Externally supplied pushback address. |
| 32 | 32 bits | Classifier address. |

Direct accesses require at least 36 accessible bytes. Dependencies may require more.

State-provided callback addresses have their lowest two bits cleared before invocation. Callback slots are read at their call sites; entry initialization does not guarantee that later calls use unchanged addresses.

### Preconditions and ownership

Every reached state access, callback invocation, destination-entry read, and destination write requires valid storage or an executable target. The format reader must accept the movement operands described below.

No allocation or release is directly visible. Ownership, retention, release, global preservation, and state preservation inside dependencies are unknown. Callback side effects must not be assumed absent.

## Behavior

### Entry initialization and global address computation

Entry operations occur in this order:

1. Write the format handle to state offset 12.
2. Write the address of F-0fba57bfda18 to offset 20.
3. Read a 32-bit global value, designated **G-1**.
4. Add the fixed code-relative base **0x080332BC** to that value as a 32-bit address computation, and store the result at offset 32.
5. Write zero to offset 16.

G-1’s value and the resulting classifier identity are unknown. The stored address is subsequently subject to the callback-address mask when invoked; the computation alone does not establish a valid target.

The function starts with assignment count zero, input-consumption count zero, and a marker indicating that no dispatched conversion has yet succeeded. It retains the input context separately.

### Format reading and literal matching

Format-reader calls receive the address of state offset 12 and a 32-bit movement operand:

- 1 for forward reading;
- 0 for inspection;
- −1, represented as 0xFFFFFFFF, for backward movement.

These movement descriptions express apparent intent. Exact cursor semantics belong to the reader dependency.

A forward format read returning zero ends scanning with the current assignment count.

A nonzero format value other than percent is passed explicitly to the classifier. If classification returns zero:

1. Invoke the input reader with the input context.
2. Compare its result with the format value.
3. On equality, increment input consumption by one and continue.
4. On inequality, invoke pushback before returning.

After a mismatch, return the assignment count unless the input result is −1 and the assignment count is zero, in which case return −1. Pushback is invoked even when the mismatching result is −1. No explicit character argument is visible in that invocation.

### Classified format runs

A nonzero classification result starts two runs:

1. Repeatedly perform a forward format read followed by classification until classification returns zero, then issue one backward format-reader call.
2. Repeatedly perform an input read followed by classification. Increment input consumption once for each nonzero classification result. Push back the first input value ending the run.

The classifier calls inside these runs have no explicit value argument in the supplied representation. Classification of the immediately preceding read is inferred intent, not an established argument contract.

Whitespace collapsing is the apparent purpose. This path has no direct EOF-return branch; termination at input exhaustion depends on classifier and pushback behavior.

### Directive parsing and threshold snapshot

After percent, the function inspects the next format value using movement zero. If it is an asterisk, it issues a forward format-reader call to consume that marker and records assignment suppression.

**After this inspection and optional consumption, but before the first forward read of a possible width digit, the function reads one signed 32-bit global threshold, G-2.** That value is retained for the entire directive. Later format-reader calls may change G-2 without changing the threshold used by this directive. A new directive takes a new snapshot.

Width begins at zero. Each forward-read value is a decimal digit only when its unsigned 32-bit difference from the value for `0` is below ten. For each accepted digit:

1. Compare the previous accumulated width with the retained threshold, using signed comparison.
2. If the previous width exceeds the threshold, return the current assignment count without incorporating that digit.
3. Otherwise form ten times the previous width plus the digit value.
4. If the resulting signed width is negative, return the assignment count.
5. Mark the width explicit.

Thus the threshold constrains the **previous** width, not directly the new width. Its value is unknown. A negative threshold rejects the first digit because the previous width is initially zero. Without any digits, no threshold comparison occurs and width becomes 2,147,483,647. Explicit zero remains zero.

The recognized flags are:

| Syntax | Flag |
|---|---:|
| Assignment suppression | 0x001 |
| Explicit width | 0x010 |
| Single `l` | 0x004 |
| Double `l`, or `j` | 0x002 |
| Single `h` | 0x008 |
| Double `h` | 0x800 |
| `L` | 0x020 |

`t` and `z` consume a modifier position without adding a flag. The parser recognizes these individual forms, not arbitrary combinations of modifiers.

After parsing, it writes flags to offset 4, then width to offset 8. Unsupported conversion characters still receive these writes. Width rejection occurs before them.

### Literal percent

A percent conversion reads one input value and requires a literal percent. Equality increments input consumption but not assignment count. Inequality invokes pushback and uses the ordinary literal-mismatch return rules.

Parsed suppression, width, and modifiers do not alter this visible matching behavior.

### Integer conversions

Integer conversion dispatch uses F-8882bf052070 with four arguments:

1. The 32-bit sentinel 0xFFFFFFFE.
2. The retained input context.
3. A radix selector.
4. The scanner-state address.

| Conversion | Radix | State-flag change before dispatch or rejection |
|---|---:|---|
| `d`, `u` | 10 | Set 0x040. |
| `i` | 0 | Set 0x040. |
| `o` | 8 | Set 0x040. |
| `x`, `X` | 16 | Set 0x040. |
| `p` | 16 | Clear bits selected by 0x80E. |

For all listed conversions except `p`, parsed flag 0x002 causes status −2 without a dependency call. The additional flag write still occurs.

Pointer conversion instead masks the parsed flags with 0xFFFFF7F1, clearing 0x002, 0x004, 0x008, and 0x800. It therefore reaches the integer dependency despite those parsed length flags. Suppression and explicit-width flags survive.

Accepted syntax, signedness, overflow handling, destination selection and width, and destination-cursor movement are dependency-defined. In particular, identical dispatch of `d` and `u` does not establish conventional signed-versus-unsigned behavior.

### Floating conversions

`a`, `A`, `e`, `E`, `f`, `F`, `g`, and `G` call **F-U01** with:

1. 0xFFFFFFFE.
2. The retained input context.
3. A temporary-storage address.
4. The scanner-state address.

The supplied representation identifies a four-byte temporary object, but recovered neighboring storage boundaries are uncertain. No floating format, destination size, accepted syntax, rounding behavior, or retention rule is established.

### Character, string, and bracket-set conversions

For `c`, the function changes state width to one unless an explicit width was parsed. `c` and `s` then reach status −2 without a visible conversion call or input-reader call for that conversion.

For `[`, bracket parsing occurs first:

1. Read the first candidate member.
2. If it is a caret, record inversion and read another candidate member.
3. Read auxiliary state at offset 16. If zero, clear **eight consecutive 32-bit bitmap words**, in increasing word order.
4. Only after this conditional initialization, test the candidate member for zero. Zero returns the current assignment count.
5. For each nonzero member, separately reread auxiliary state. If zero, insert its bit into the bitmap.
6. Read the next format value. A closing bracket ends the set; otherwise process that value as another member.

Consequently, a closing bracket in the initial member position is a member, not a terminator. A subsequent closing bracket is required. Hyphen has no visible range-expansion behavior.

For member values from 0 through 255, the bitmap location corresponds to the value divided by 32, with the bit position given by its remainder modulo 32. Zero nevertheless terminates parsing before insertion. Values outside the byte domain have no established safe bitmap interpretation.

If inversion was requested and a closing bracket is reached, the function complements **all eight bitmap words**, in increasing word order, **without checking auxiliary state**.

Initialization and insertion do not share a cached auxiliary-state decision. A callback can leave auxiliary state nonzero at initialization and later change it to zero, causing insertion into words not initialized by this directive. Inversion can likewise read and complement such words even if insertion remained disabled. This initialization uncertainty is separate from the bitmap-storage extent discrepancy described below.

After `c`, `s`, or completed bracket parsing, the common tail reads auxiliary state and then tests flag 0x004 from state offset 4. All represented alternatives yield −2. No input matching against a completed bracket set is visible.

### Count reporting

For `n`, the ordered accesses are:

1. Read the destination cursor from offset 0.
2. Advance it by four bytes and write the advanced cursor to offset 0.
3. Read a 32-bit destination address from the old cursor location.
4. Store the current input-consumption count through that address.

Cursor advancement precedes destination-entry reading; aliasing must not be treated as an atomic fetch-and-advance operation.

Store selection uses parsed flags:

- 0x800: one 8-bit store.
- Otherwise 0x008: one 16-bit store.
- Otherwise: one 32-bit count store.
- In the last case, 0x002 additionally causes a second 32-bit store, at destination plus four, containing the count’s sign extension.

The narrow stores retain the count’s low 8 or 16 bits. The two-word form stores the count first, then either zero or all-one bits according to its sign. This establishes word-address order, not native byte order.

Single `l`, `L`, `t`, and `z` do not independently change this selection. Suppression does not prevent cursor advancement or writing. `n` neither increments assignment count nor marks a dispatched conversion successful.

### Conversion-result accounting

For integer and floating dependency results:

- Any negative status other than −1 returns the current assignment count.
- Status −1 returns −1 if no dispatched conversion has succeeded; otherwise it returns the assignment count.
- A nonnegative status is added to input consumption.
- A successful nonsuppressed conversion increments assignment count.
- Every successful dispatched conversion clears the no-success marker, including suppressed conversions and status zero.

Suppression accounting uses the retained parsed flag, not a reread of potentially modified state flags.

Literal EOF mismatch instead tests whether assignment count is zero. Literal matches and `n` do not clear the no-success marker.

Unsupported conversion characters return the current assignment count without a conversion call.

## Side effects

Direct effects include state initialization, two global reads at their specified points, per-directive state writes, format reads and repositioning, input reads, pushback, dependency calls, count-reporting stores, and temporary bitmap accesses.

There is no rollback. Early termination retains preceding cursor changes, state writes, destination writes, input effects, and dependency effects.

Callbacks may modify globals, scanner state, destination storage, or other memory. G-2 changes cannot affect the already retained threshold for the current directive, but state changes can affect subsequent callback selection and auxiliary-state decisions. Unknown dependencies are not assumed to preserve or own any storage.

## Dependencies

| Opaque identifier | Established boundary |
|---|---|
| F-0fba57bfda18 | Initially installed format reader; receives the format-state address and movement operand. Cursor and error semantics are unknown. |
| F-8882bf052070 | Integer conversion dependency; receives sentinel, input context, radix, and state; returns a signed status. |
| F-U01 | Unmapped floating dependency; receives sentinel, input context, temporary-storage address, and state. |
| F-U02 | Input-reader role reached through offset 24; receives the input context. |
| F-U03 | Pushback role reached through offset 28; receives the input context, with no explicit character argument visible. |
| F-U04 | Classifier role initially installed through G-1 plus the fixed base; some calls have an explicit value argument and others do not. |

F-U01 through F-U04 are document-local opaque function identifiers, not verified published identities. State changes may replace callback targets.

G-1 is the 32-bit classifier-address contribution. G-2 is the signed 32-bit width threshold. Neither global’s value is supplied.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Syntax examples assume the format reader provides the apparent forward, inspection, and backward behavior.

1. **Empty format:** A first forward read of zero returns zero after entry initialization, including the G-1 read and classifier-address store.

2. **Literal match and count:** Two ordinary matching literals followed by `hhn` consume one destination entry and store the 8-bit value two. Assignment count remains zero.

3. **Literal EOF:** A first ordinary literal receiving input −1 invokes pushback and returns −1.

4. **Threshold snapshot:** Suppose G-2 is read as 10 and the following width is `99`. Both checks pass: previous widths are zero and nine. The width becomes 99. A format-reader callback changing G-2 to zero between digits does not change this outcome.

5. **Threshold rejection:** With retained threshold 10, width `999` rejects at the third digit because the previous width is 99. No per-directive flags or width are stored. With a negative threshold, even explicit width `0` rejects at its first digit.

6. **Unsupported directive:** Reaching `q` stores parsed flags and width, then returns the assignment count without a conversion call.

7. **Suppressed success and EOF:** A suppressed decimal dependency result of three adds three to input consumption and records conversion success without an assignment. A following dependency result −1 returns zero if no nonsuppressed conversion succeeded. A following literal EOF mismatch instead returns −1.

8. **Length dispatch:** `lld` sets flag 0x040 and terminates without calling F-8882bf052070. `llp` clears the relevant length flags and calls it with radix 16.

9. **Suppressed count reporting:** `*n` still advances the destination cursor and writes the count.

10. **Character conversion:** An unqualified `c` stores width one, then terminates without a visible character-conversion input read.

11. **Zero first bracket member:** If the first candidate is zero and auxiliary state is zero, eight bitmap words are cleared before returning. With an initial caret, this applies to the candidate read after the caret.

12. **Skipped initialization:** If auxiliary state is nonzero at initialization, clearing is skipped. If a later callback changes it to zero before a member insertion, that insertion reads and modifies a word not initialized by this directive. For an inverted set reaching its closing bracket, all eight words are complemented regardless of auxiliary state.

## Uncertainties

The following remain **documented_only**:

- G-1, G-2, and the resulting classifier identity are unknown.
- Exact maximum accepted widths cannot be stated without G-2. Threshold snapshot timing is established.
- Width and counters use 32-bit representations. Overflow guarantees for width accumulation, count addition, and address validity are not established; no general saturation or safe-overflow promise is made.
- Bitmap storage is represented as seven 32-bit words, while clearing and inversion access eight. Full-byte indexing also requires eight. Whether this reflects recovered storage boundaries or an actual out-of-bounds access is unresolved.
- Independently, auxiliary-state changes can cause bitmap reads before initialization by this directive.
- Bracket indexing has no visible range check. Negative and greater-than-byte member values are not established as safe.
- Classifier arguments inside classified runs are incompletely represented.
- Numeric destinations, conversion syntax, dependency ownership, temporary retention, native byte order, and callback preservation are unknown.

## Evidence

Established claims derive from static control flow, data flow, access widths, and the supplied dependency mapping. No implementation execution, runtime tests, or smoke runs were performed.

The scanner interpretation, whitespace interpretation, and movement terminology are inferred intent. Initialization order, global-read timing, retained threshold use, dispatch decisions, count-store widths, bracket initialization and inversion order, and return distinctions are directly supported by the supplied representation. Unavailable dependency behavior and unresolved storage boundaries remain explicit rather than being replaced with conventional assumptions.
