<!-- Trusted provenance: F-3a66d89bfec7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3a66d89bfec7

## Purpose

F-3a66d89bfec7 interprets a zero-terminated byte format using an argument cursor into four-byte words. It delegates character and value output, directly implements count-storage conversions, and finishes by requesting output of a zero value.

This is not an established standard formatting interface. Its observable parsing rules include:

- Only `f` calls the floating-output dependency. `e`, `E`, `g`, and `G` advance the argument cursor without formatting an operand.
- Integer narrowing and count-store width depend on whether the length-selection value equals 1. The selections produced by `ll` and `q` have an unresolved numerical value.
- Width and precision classification is delegated. A nonzero classification result accepts the current byte for decimal-style accumulation, even if that byte conventionally denotes a conversion.
- Unrecognized conversions produce no conversion-output call.
- `w` skips an additional format byte without inspecting it.
- Finalization can reread the limit after writing status. Aliasing can therefore change the count passed to the final output request.

These are call-site and direct-memory behaviors established by static evidence. Actual formatted output remains dependent on implementations not supplied here.

## Interface

The function has seven inputs and no explicit return value.

| Position | Input and established use |
|---|---|
| 1 | Opaque 32-bit context forwarded to output dependencies. |
| 2 | Byte-pointer destination or mode operand. Null enables the parser’s capacity check; non-null bypasses it. |
| 3 | Address of a mutable 32-bit limit field. |
| 4 | Address of a 32-bit result field written after the final output call. |
| 5 | Address of a 32-bit status field written before the final output call. |
| 6 | Address of the first format byte. |
| 7 | Initial argument cursor. Ordinary argument slots occupy four bytes; floating conversions align the cursor upward to an eight-byte boundary. |

The representation establishes a 32-bit word and address model with eight-bit bytes. Count, flags, width, precision, length selection, and cursor arithmetic are interpreted at 32-bit width. Pointer-shaped representations of numeric state do not imply that those values designate storage.

Count begins at zero. Output dependencies receive its address and the caller’s limit address. The updated argument cursor is not returned.

### Preconditions and storage

Every reached direct access requires valid storage with suitable readability, writability, and alignment. No local validation establishes these conditions.

In particular:

- The first format byte is read before the first capacity decision.
- A consuming ordinary conversion requires a readable four-byte argument slot.
- `f` requires two readable four-byte words at the aligned argument address.
- `e`, `E`, `g`, and `G` perform cursor arithmetic without directly reading the skipped argument region.
- `n` requires a writable target supplied through an argument slot.
- `s` forwards a word whose interpretation and storage requirements depend on F-29d53c9826da.
- `w` can skip a terminating zero and cause a subsequent read beyond it.

Format and argument storage are not snapshotted. Reads observe memory when reached. Earlier stores or dependency effects can change later format bytes and operands. No non-aliasing condition is enforced.

## Behavior

### Parser entry, checks, and fetching

Initial conversion state is flags zero, width zero, precision `0xffffffff`, and length selection zero. The first format byte is fetched and zero-extended to 32 bits.

Before each parser-state dispatch, parsing stops if:

1. The cached current byte is zero; or
2. The destination operand is null and the current limit is less than or equal to count, using unsigned 32-bit comparison.

The zero-byte test short-circuits the capacity test. A zero current byte therefore avoids that capacity-check limit read, although finalization still reads the limit.

These checks occur between parser steps, including steps within a conversion. They do not precede every individual memory access. A step can fetch the next byte before the next check. Consequently, a capacity stop does not establish that the pending format byte was unread.

Some state transitions retain the current byte without fetching it again. Changes to its former memory location do not alter the cached value. Changes to unread storage can affect subsequent fetches.

In ordinary-text state:

- `%` starts conversion parsing.
- Any other nonzero byte is passed, zero-extended, to F-bcb8529decd8.
- The next format byte is fetched after that call returns.

The function does not itself increment count for these requests.

### Flags

The following bytes accumulate flag bits by bitwise OR:

| Byte | Added bit |
|---|---:|
| `-` | `0x01` |
| `+` | `0x02` |
| Space | `0x04` |
| `#` | `0x08` |
| `0` | `0x10` |

Repeated flags are accepted. No local precedence or conflict resolution is applied.

Each recognized flag is followed by a byte fetch. The first non-flag byte is retained for width processing.

### Width

F-cfad0582408b receives each prospective width byte. Any nonzero return selects accumulation; the caller does not independently require an ASCII digit.

Each accepted byte updates width to ten times its previous value, plus the zero-extended byte value, minus `0x30`, using 32-bit arithmetic. There is no range or overflow check. A following format byte is then fetched.

When classification returns zero:

- If the current byte is `*`, the next argument word replaces width. The following format byte is fetched, and the argument cursor advances by four bytes.
- Otherwise, the byte remains current for precision-introduction processing.

Thus a width argument replaces any previously accumulated width. A width whose signed interpretation is negative receives no local absolute-value conversion or automatic flag adjustment.

Classification occurs before recognition of `*`. If the classifier returns nonzero for that byte, accumulation occurs instead of argument consumption.

### Precision

A period reaching the precision-introduction state causes the following byte to be fetched into precision parsing. Without that period, precision remains `0xffffffff`.

Precision parsing calls F-cfad0582408b before deciding whether the byte belongs to a later parsing stage. For each nonzero classification result:

1. If precision has a negative signed 32-bit interpretation, it is reset to zero.
2. The current byte is accumulated using the same 32-bit decimal-style arithmetic as width.
3. The next byte is fetched.

The reset applies both to the initial sentinel and to an accumulated value that has wrapped into the signed-negative range.

When classification returns zero:

- `*` replaces precision with an argument word, fetches the following format byte, and advances the argument cursor by four bytes.
- Any other byte is retained for length processing.

A period immediately followed by a conventional conversion letter leaves precision at the sentinel **only if the classifier returns zero for that letter**. If classification returns nonzero, the letter is accumulated and consumed as precision input instead. No separate empty-precision-to-zero rule exists.

A fetched zero terminates parsing at the outer check rather than reaching classification.

### Length selection

The length stage recognizes one modifier, with a special two-byte `ll` case.

| Modifier | Established selection |
|---|---|
| None | 0 |
| `h` | 1 |
| `l` | 2 |
| `L` | 3 |
| `ll` | A symbolically represented value whose numerical value is not supplied |
| `q` | The same unresolved value as `ll` |

For the affected integer conversions and `n`, the operative condition is equality of the selected value to 1.

Therefore:

- `h` definitely selects narrow behavior.
- Default, `l`, and `L` definitely do not.
- `ll` and `q` select narrow behavior if their unresolved value equals 1; otherwise they select full-word behavior.

Neither `ll` nor `q` establishes eight-byte integer consumption. Both branches consume one four-byte slot for the affected conversions.

For `l`, the parser reads the following byte to test for a second `l`. If the test matches, that second modifier is consumed and the subsequent conversion byte is fetched. If it does not match, the byte after the first `l` is read again as the conversion byte. The static representation contains distinct reads in the latter case; they are not an immutable snapshot.

No general repeated-modifier loop exists. Apart from `ll`, a subsequent modifier-like byte becomes the conversion byte.

### Conversion dispatch

The table describes conversion-specific behavior after any width and precision arguments have already been consumed. “Narrow selection” means that the selected length value equals 1, including the unresolved conditional case for `ll` and `q`.

| Conversion | Direct behavior |
|---|---|
| `%` | Requests output of `0x25`; consumes no conversion operand. |
| `c` | Reads one word and passes its full 32-bit value to F-bcb8529decd8. |
| `d`, `i` | Reads one word and calls F-ff3dab79585a with selector 10. Narrow selection sign-extends the low 16 bits; otherwise the full word is passed. |
| `u` | Reads one word, adds flag `0x40`, and calls F-ff3dab79585a with selector 10. Narrow selection zero-extends the low 16 bits. |
| `o`, `x`, `X` | Reads one word and calls F-ff3dab79585a with the associated symbolic nondecimal selector. Narrow selection zero-extends the low 16 bits. All add `0x40`; `X` also adds `0x20`. |
| `p` | Reads one full word, adds `0x08`, and uses the hexadecimal-associated selector. It neither applies narrow selection nor itself adds `0x40`. |
| `s` | Reads one word and calls F-29d53c9826da with that operand, flags, width, and effective precision. |
| `n` | Reads one word as an address. Stores the low 16 count bits for narrow selection; otherwise stores the full 32-bit count. |
| `f` | Aligns the cursor upward to eight bytes, reads two four-byte words, and calls F-9acf98fc179b with them, width, precision, and flags. Continues after that eight-byte region. |
| `e`, `E`, `g`, `G` | Aligns the cursor upward to eight bytes and advances by eight bytes, without directly reading operand words or calling a conversion-output dependency. |
| `w` | Consumes no operand and advances past one additional format byte before the ordinary-state fetch. |
| Any other nonzero byte | Consumes no conversion operand and makes no conversion-output call. |

Integer and string calls receive width, precision, flags, context, destination, count address, and limit address in their respective interfaces. Their returned values do not control subsequent parsing.

Immediately before an `s` call, signed-negative precision is replaced as follows:

- Null destination: read the current limit and use its 32-bit value.
- Non-null destination: use `0x7fffffff`.

Nonnegative precision remains unchanged. A limit-derived value is not subsequently normalized merely because its high bit is set.

Floating alignment adds seven to the 32-bit cursor and clears the low three bits. There is no wraparound check. The relative order of the two operand-word reads for `f` is not established.

After a completed conversion dispatch, flags, width, precision, and length selection return to their initial values, and the next ordinary-state byte is fetched. Unknown conversions consequently disappear rather than being emitted literally.

If zero or capacity termination occurs partway through parsing, no pending conversion dispatch is completed. Argument consumption that already occurred for width or precision is not undone.

### Finalization

Finalization has this ordered behavior:

1. Read the current 32-bit limit.
2. Subtract one at 32-bit width.
3. Compare count against that value using unsigned comparison. Status is 1 exactly when count is greater; otherwise it is 0.
4. Write status to the caller’s status field.
5. If the computed status was 1, reread the limit, subtract one at 32-bit width, and assign that value to count.
6. Call F-bcb8529decd8 with value zero and the same context, destination, count address, and limit address.
7. Write the post-call count minus one, at 32-bit width, to the result field.

The overflow branch uses the previously computed condition. Its count assignment uses the later limit read, not the earlier value. A status field overlapping the limit can therefore change the assigned count.

For limit zero, subtraction produces `0xffffffff`; this does not automatically indicate overflow. The result field depends on count after the final dependency call.

## Side effects

Direct caller-visible writes are:

- Reached `n` target stores.
- The final status store.
- The final result store.

These targets are not checked against destination, format, argument, or limit storage. Direct writes can therefore modify destination bytes, future format input, or future operands. Overlapping result and status fields can cause the result write to replace status.

Output dependencies receive mutable count and limit addresses. Their effects can alter later capacity decisions, effective string precision, finalization, and result calculation. Dependencies may also affect other accessible storage or global state; preservation is not established.

There is no explicit allocation, deallocation, ownership transfer, error result, or exception recovery in this function. This does not establish corresponding behavior inside dependencies. In particular, no claim is made that they preserve globals, retain no references, or refrain from allocating or releasing memory.

Invalid storage, mismatched arguments, arithmetic-derived invalid addresses, and dependency failures have no specified recovery path here.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-bcb8529decd8 | Receives ordinary bytes, literal percent, full-word `c` operands, and the final zero request, together with output-state references. |
| F-cfad0582408b | Classifies prospective width and precision bytes. Any nonzero result selects accumulation. |
| F-ff3dab79585a | Receives an integer operand, selector, width, precision, flags, and output-state references. |
| F-29d53c9826da | Receives a string-related operand, flags, width, effective precision, and output-state references. |
| F-9acf98fc179b | Receives two aligned argument words, width, precision, flags, and output-state references. |

The context and destination values forwarded to output dependencies remain the same. This does not imply preservation of memory reachable through them.

## Acceptance examples

These are conditional static expectations, not runtime-test results.

Unless stated otherwise, assume valid storage, no relevant overlap, stable unread format and argument storage, conventional ASCII digit classification, and dependency effects that preserve the described execution conditions. Preserving capacity alone is insufficient if a dependency changes future format bytes or operands.

1. **Ordinary text.** Format `AB` requests output of `0x41`, `0x42`, and finally zero. Actual stored bytes and resulting count require the output dependency’s contract.

2. **Live format input.** If the dependency handling `A` in `AB` changes the unread `B` to `C` while preserving termination and continued parsing, requests become `A`, `C`, and zero.

3. **Live arguments.** For `A%d`, if the dependency handling `A` changes the unread integer word from 7 to 9, the integer dependency receives 9.

4. **Narrow integers.** With operand `0x0000ffff`, `%hd` forwards `0xffffffff` with selector 10. `%hu` forwards `0x0000ffff` with selector 10 and added flag `0x40`.

5. **Unresolved length selection.** For `%lld` or `%qd` with operand `0x0000ffff`, the forwarded value is `0xffffffff` if the unresolved selection equals 1, and `0x0000ffff` otherwise. Either case consumes one four-byte conversion operand.

6. **Empty precision under conventional classification.** `%.s` leaves precision negative until string dispatch. Effective precision becomes the current limit for a null destination, or `0x7fffffff` otherwise.

7. **Nonconventional precision classification.** If the classifier accepts `s` during precision parsing in `%.sA`, precision becomes `0x43` after that byte, and `s` is consumed rather than dispatched as a string conversion. Subsequent behavior depends on classification of `A` and later parsing.

8. **Width replacement.** `%12*d` consumes one word as width and the next as the integer operand. The width word replaces 12.

9. **Unknown conversion and skip.** `%zA` makes no conversion-output call for `z`, then processes ordinary `A`. `%wAB` skips `A` and processes `B`. If zero immediately follows `w`, that zero is skipped without inspection by the dispatch.

10. **Floating cursor movement.** `%e%d` skips an aligned eight-byte region and obtains the decimal operand at its end. `%f%d` first calls the floating dependency with the two aligned words, then obtains the decimal operand there.

11. **Count-store widths.** `%hn` stores 16 bits. `%ln` and `%Ln` store 32 bits. `%lln` and `%qn` store 16 bits if their unresolved selection equals 1, otherwise 32 bits.

12. **Capacity interruption.** With null destination and count equal to limit at the check before `n` dispatch, no target store occurs. The conversion byte may already have been fetched.

13. **Aliased finalization.** Entering finalization with count 5, limit 5, and shared status/limit storage first determines status 1, then writes 1 to that storage. The subsequent limit read obtains 1, so count becomes 0 before the final zero request—not 4.

14. **Zero limit.** With count and limit both zero, null-destination parsing stops at its first check. Finalization writes status 0 because the comparison threshold is `0xffffffff`. The result still depends on the final dependency’s count update.

## Uncertainties

**Established static evidence:** Parser transitions, live memory reads, direct stores, 32-bit arithmetic, argument-slot consumption, explicit flag bits, equality-to-1 length checks, decimal selector 10, and finalization’s status-write/conditional-reread ordering.

**Inferred intent:** A formatted-output engine; octal and hexadecimal interpretations of the associated nondecimal selectors; terminator intent for the final zero request. These interpretations do not establish dependency output.

**documented_only unknowns:**

- The numerical selection assigned by `ll` and `q`, including whether it equals 1. Affected integer narrowing and count-store width remain conditional.
- Numerical values of the symbolically represented nondecimal selectors.
- The classifier’s accepted-byte set and side effects.
- Dependency padding, signs, prefixes, precision interpretation, string traversal, and character narrowing.
- Whether count measures attempted output, stored output, or another quantity.
- Whether the final zero request writes a terminator.
- Dependency errors, allocation, ownership, retention, release, and global-state effects.
- Operand-read ordering not established by the supplied expression, including the two floating operand reads.
- Machine byte order and execution-environment alignment requirements.
- Whether unusual parsing behavior is intentional.

## Evidence

This contract derives solely from the supplied static function representation and opaque dependency mapping. The prior document and reviewer feedback are not independent behavioral evidence.

The representation explicitly assigns 1 for `h`, 2 for `l`, and 3 for `L`, but supplies only a symbolic value for `ll` and `q`. The affected conversion branches test equality to 1, so those modifiers cannot be classified unconditionally as non-narrow.

Precision classification precedes transition to length and conversion handling. Therefore, a conversion-looking byte ends precision parsing only when its classification result is zero.

The representation also establishes live future-input reads, unchecked direct-store targets, and separate finalization limit reads surrounding the status write. Dependency implementations were not supplied or inspected. No runtime tests have been performed.
