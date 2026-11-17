<!-- Trusted provenance: F-bba13a37cbf3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bba13a37cbf3

## Purpose

F-bba13a37cbf3 performs an ordered sequence of indirect calls and fixed-width transformations, followed by memory mutations through retained, absolute, and read-derived addresses. It ends with an unresolved indirect transfer carrying four displayed operands.

No application-level purpose is established. Character-valued constants do not establish text production; recurring offsets do not establish a particular record layout, linked structure, or ownership model.

This contract distinguishes:

- **Established static evidence:** explicit conversions, typed pointer accesses, value dependencies, and ordering visible in the supplied representation.
- **[INFERENCE]:** interpretations requiring additional assumptions.
- **documented_only:** unresolved behavior or trace detail not specified as an executable acceptance requirement.

No runtime tests have been performed.

## Interface

### Effective inputs and outputs

The displayed interface declares neither arguments nor a return value. Execution nevertheless consumes incoming machine state:

- A 32-bit incoming quantity supplies both displayed operands of the first call after a seven-position left shift.
- Another 32-bit incoming quantity is consumed after a nine-position left shift.
- Incoming continuation-related state determines the first call destination by addition of 0xffe.
- Later operands depend on primary and secondary call results. Some results occupy 64-bit containers whose low and high 32-bit portions are consumed separately.
- Memory reads supply data, addresses, and subsequent indirect-transfer operands.

The representation does not establish a safe standalone calling convention.

There is no declared status result, result buffer, or conventional success indication. Observable outputs are calls, memory accesses, and the final transfer. Its displayed operands are 0x72, a later captured word, an earlier captured word, and another later captured word. Whether this transfer returns is **documented_only**.

### Widths and layout

An explicitly typed pointer access establishes its represented access width: 8 bits for a byte, 16 bits for a halfword, and 32 bits for a word. Offsets are byte offsets. Explicit one-byte conversions retain the low eight bits. Conversion of an explicitly unsigned byte to an unsigned 32-bit quantity zero-extends.

**These rules do not establish the access widths of bare absolute-address globals whose declarations are absent.** In particular:

| Absolute storage | Established value operation | Unresolved access property |
|---|---|---|
| 0x05 | Its value is converted to an unsigned 32-bit quantity used as a base | Read width, signedness, and whether conversion zero-extends |
| 0x01 and 0x15 | Assignment sources are explicitly reduced to one byte | Destination store widths and destination conversions |
| 0x3a | An operand is character-converted, multiplied by eight, and increased by 0x30 | Character signedness, destination width, and final conversion |
| 0x67 | The initial assignment source is explicitly reduced to one byte; three later value reads occur | Store and read widths, declared type, and read conversions |
| 0x21 | Receives the first subsequent value read from 0x67 | Destination width and assignment conversion |
| 0x54, 0x60, and 0xa9 | Assignments or captures involving represented 32-bit values | Underlying absolute-global access widths |

The separate global whose value is retained as an address base also lacks a supplied declaration. Its captured value is represented as 32 bits; its underlying read width is **documented_only**.

A conversion to a 32-bit destination does not prove a 32-bit source-memory read. Likewise, assigning a one-byte value does not prove a one-byte destination-memory store.

No endianness, alignment guarantees, object boundaries, permissions, or valid address ranges are established. Accesses may overlap, including accesses of different widths.

### Preconditions and failures

Completion requires meaningful incoming execution state, executable reached call destinations, returning dependencies where continuation is needed, and accessible storage for every reached access.

No null checks, bounds checks, alignment repairs, rollback, or recoverable error result are displayed. Fault signaling, partial-write behavior, exception propagation, and nonreturning dependencies are **documented_only**.

Signed overflow, signed left shifts, address wrapping, and oversized variable shifts require target semantics. Unsigned modular behavior must not be assigned indiscriminately to signed expressions.

## Behavior

### Indirect calls and arithmetic

The opening region is a finite, straight-line sequence of indirect calls. No displayed caller-side conditional branch or loop controls this region.

The first call receives two equal seven-position-shifted operands. Calls without displayed operands and calls consuming transformed results follow. An early primary result is shifted left by eight for a subsequent call. Later calls consume retained earlier results, secondary results, explicit zeros, and transformed portions of 64-bit results.

A call without displayed operands is not established as pure or free of effective machine-state inputs. Explicit zero operands do not authorize call elimination.

Transformations include masks of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, and one bit, together with shifts through 31 positions. Width-sensitive relationships include:

- Extracting the low 32 bits before shifting is distinct from shifting the full 64-bit container.
- Logical 64-bit right shifts by 34, 36, and 44 followed by 32-bit extraction yield the high word logically shifted by two, four, and twelve.
- A 20-bit masked quantity logically shifted right by ten is at most 0x3ff.
- A 19-bit masked quantity shifted left by two is at most 0x1ffffc; logical right shifts of that bounded result by 23 or 24 yield zero.
- Under truncating 32-bit semantics, successive left shifts can discard all bits. Wider intermediate arithmetic cannot be substituted without establishing equivalence.

[INFERENCE] Packed-state manipulation is plausible, but no parser, protocol, serializer, or arithmetic algorithm is established.

The complete call-site enumeration is **documented_only** in this contract. Unresolved dependency identities do not make calls interchangeable or authorize reordering.

### Initial typed memory effects

The final result-producing call supplies low and high 32-bit portions used as address material and data.

Retained values include a result shifted left by 25, a separate result multiplied by 2²⁵, and an earlier eight-position-shifted result. A bounded index comes from an eight-position left shift, a low-15-bit mask, and a logical right shift by six. Under the represented truncating interpretation, the index is a multiple of four from zero through 508; twice the index ranges from zero through 1016.

Before subsequent direct reads, four explicitly typed stores occur in order:

1. A word at high-result offset 0x14 receives the retained 25-position-shifted value.
2. A byte at shifted-base offset 0x09 receives the high result’s low byte.
3. A word at offset 0x54 from 256 times the low result receives the separate 2²⁵-scaled value.
4. The byte at shifted-base offset 0x09 is overwritten with the shifted base’s low byte.

These are distinct accesses even when an earlier value is subsequently overwritten.

A word at scaled-base offset 0x14 is cleared, then an explicitly unsigned byte is read at that base’s offset 0x05. A word at twice-index offset 0x14 is cleared before the captured byte is zero-extended into a word store at scaled-base offset 0x34.

The first cleared word is read only afterward. Intervening stores may overlap it; the read is not unconditionally zero. Its captured value subsequently influences both data and addressing.

### Absolute-derived and explicitly byte-derived bases

Further effects include another clear at twice-index offset 0x14 and duplicate byte stores at twice-index offset 0x15.

The value of absolute storage 0x05 is then read and converted to an unsigned 32-bit base, called the **absolute-derived base** here. Neither a byte-wide read nor a range of zero through 255 is established. A narrow signed declaration, for example, could produce a different conversion from an unsigned-byte declaration; neither declaration is supplied.

An explicitly byte-reduced value is assigned to absolute storage 0x01. A typed byte store at absolute-derived-base offset 0x15 follows, as do cross-base typed stores.

A word captured at absolute-derived-base offset 0x54 before that location is overwritten is called the **original link**. This name describes only a retained dependency, not a validated pointer.

The ensuing region uses the original link, four and eight times that value, and words read at offsets 0x04, 0x64, and 0x74. Effects include self-valued word stores, duplicate stores, byte stores at offsets 0x09, 0x0d, 0x11, and 0x15, and word stores at offsets 0x14, 0x24, 0x34, 0x44, and 0x54. Assignments also target absolute storage 0x54 and 0x15; their store widths remain **documented_only**.

Repeated offset-0x74 reads are separate accesses. One compound store derives both its destination and its data from such reads; their relative evaluation order is not established.

Unlike the read of absolute storage 0x05, later lookups explicitly read unsigned bytes through typed pointers. One such byte, obtained through a word-derived base, selects an offset-0x74 word lookup. The resulting word is called the **loaded link**. That byte selector is established as zero through 255.

The broader intermediate trace is summarized rather than exhaustively enumerated. Omitted access-by-access detail remains **documented_only**; non-aliasing must not be presumed.

### Constants and captured global base

For the remaining regions:

- **Adjusted unit** means four times the original link minus 0x96.
- **Work base** means four times the adjusted unit.
- **Doubled-link base** means twice the loaded link.
- **Retained payload** means four times the loaded link.

These relationships retain the arithmetic qualifications above; they are not guarantees of valid addresses.

Explicitly typed stores write 0x3a as a byte at four times the loaded link, as a halfword at eight times that link, and as a word at retained-payload offset 0x20.

At work-base offset 0x20, successive word stores write 0x20 and 0x2f. An explicitly unsigned byte read at work-base offset 0x04 then supplies a zero-extended word store to the same destination, followed by another word store of 0x20.

**After that last store, the separate global’s value is read and captured as the global-derived base. Only afterward is 0x20 assigned to absolute storage 0x60.** The absolute assignment’s store width is unresolved; its position in the trace is not.

Later repeated effects use the captured base rather than reloading the global after the 0x60 assignment.

### Repeated cross-base stores

Each complete group has four ordered, explicitly typed effects:

| Position | Destination | Data |
|---|---|---|
| First | Word at an eight-times-scaled base plus 0x54 | Retained payload |
| Second | Byte at global-derived base plus 0x19 | Work base’s low byte |
| Third | Byte at work base plus 0x11 | Global-derived base’s low byte |
| Fourth | Word at global-derived base plus 0x34 | Eight-times-scaled value |

The initial group’s first store targets eight times the **original link** plus 0x54. Subsequent first stores target eight times the **adjusted unit** plus 0x54. Their stored payload is unchanged.

The initial fourth store writes eight times the original link. Subsequent fourth stores write eight times the adjusted unit to the same global-derived destination.

Thus the first-store distinction concerns destination addresses; the fourth-store distinction concerns data. Both byte values and the global-derived base remain captured throughout.

The region is finite and explicitly repeated, not an unbounded loop. Its exact group count remains **documented_only** in this contract. Aliasing can affect final contents within a group; replacing the trace with final assignments is not equivalent.

### Concluding captures and three separate uses of 0x67

After the repeated region, the function captures a value represented as 32 bits from absolute storage 0xa9, then reads a typed word at retained-payload offset 0x74.

An assignment to absolute storage 0x3a uses a character-converted adjusted unit, multiplied by eight and increased by 0x30. The destination declaration is absent. Consequently, neither a byte store nor a final modulo-256 result is established. Character signedness and destination conversion remain unresolved.

Before related writes, typed words are captured from doubled-link-base offset 0x14 and work-base offset 0x04. Early concluding effects include typed byte stores at doubled-link-base offset 0x0d and work-base offsets 0x08, 0x0d, and 0x09. The offset-0x08 value is 0x6c. A word store at doubled-link-base offset 0x54 writes that base itself.

The absolute-storage flow preserves this ordering:

- The low eight bits of the captured 0xa9 value are assigned to storage 0x67.
- A typed word is read at doubled-link-base offset 0x44.
- The first subsequent value read from 0x67 supplies an assignment to storage 0x21.

Neither absolute store width nor the width of this 0x67 read is established. The intervening offset-0x44 word read remains significant for access-sensitive storage.

Before the second 0x67 read, mutations include a byte store at work-base offset 0x10, two word stores at global-derived offset 0x54 separated by a variable-shift byte store, a word read at work-base offset 0x40, and word stores at global-derived offset 0x74, work-base offset 0x10, and global-derived offset 0x44.

The second value read from 0x67 supplies an **explicitly one-byte destination** at work-base offset 0x09. The stored byte is the low eight bits of that read value; this does not prove that the source read was byte-wide.

Before the third read, the function writes a byte at doubled-link-base offset 0x09, writes the captured 0xa9 value as a typed word at work-base offset 0x44, and reads a word at work-base offset 0x54. The third value read from 0x67 supplies another one-byte store at work-base offset 0x09.

All three reads are separate. Intervening writes may alias 0x67. Its second and third values need not equal its first value or the initially assigned low-byte value.

### Remaining mutations and final transfer

The variable-shift byte store targets doubled-link-base offset 0x01. Its source uses a signed right shift of the word captured at doubled-link-base offset 0x44, with a count equal to the work base’s low eight bits. Negative-value right shifts and counts of at least 32 are **documented_only** without target semantics.

The work-base offset-0x40 word supplies the later word store at offset 0x10, after the earlier byte store there. The captured work-base offset-0x04 word is stored at global-derived offset 0x74.

After the third 0x67 read, later effects consume the captured payload-offset-0x74 word and the word obtained at work-base offset 0x54. The earlier captured doubled-link-base offset-0x14 word is written at doubled-link-base offset 0x54, replacing the earlier self-valued store.

Further typed accesses include a work-base offset-0x14 word read, byte stores at offsets 0x09, 0x0d, and 0x11, and a halfword store of 0x65 through twice a word-derived base.

A word read at doubled-link-base offset 0x04 supplies a new base. Words at its offsets 0x64 and 0x74 become final-transfer operands. Other concluding effects include a word store of decimal 100 at a derived offset-0x54 destination, a word read at doubled-link-base offset 0x64, and a final word store through twice the captured new-base offset-0x64 value.

The final transfer follows that last direct store. Its operands, in order, are:

1. 0x72.
2. The word captured at new-base offset 0x64.
3. The earlier captured doubled-link-base offset 0x14 word.
4. The word captured at new-base offset 0x74.

## Side effects

The function is not established as pure, atomic, reentrant, or thread-safe. Effects include indirect calls, mixed-width typed accesses, absolute-global accesses of unresolved widths, repeated stores, and read-derived addressing.

No explicit allocation or release operation is identified. Unknown callees may allocate, release, retain, or mutate storage. No ownership transfer, borrowing lifetime, or preservation guarantee is established.

Earlier effects may remain if a dependency does not return or an access fails. Access-trace equivalence is stronger than final-memory equivalence, particularly for shared, instrumented, or device-backed storage. Missing global declarations also prevent exact overlap analysis for those accesses.

## Dependencies

Only F-bba13a37cbf3 has an authoritative identity in the supplied dependency map. These opaque references are document-local:

| Dependency | Established interaction | Unresolved contract |
|---|---|---|
| F-opaque-01 | Entry-derived destination receiving two equal shifted operands | Identity, ABI, results, effects |
| F-opaque-02 | Subsequent call-site family consuming transformed primary and secondary results | Target distinctness, hidden inputs, preservation, ownership |
| F-opaque-03 | Final transfer receiving four displayed operands beginning with 0x72 | Destination, return behavior, effects, failures |

No unknown callee is assumed to preserve globals, own passed storage, or release resources.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Truncating entry shift

If the incoming quantity is 0x02000001 and the shift has truncating 32-bit semantics, both first-call operands are 0x00000080.

### Explicit byte read versus absolute-global conversion

If the explicitly typed early scaled-base offset-0x05 byte read yields 0xe7, its associated word store writes 0x000000e7.

This does not establish the same behavior for absolute storage 0x05. If its value converts to 0x1234, the absolute-derived base is 0x1234; imposing a low-byte mask is unsupported. Whether that source value is representable depends on the missing declaration.

### Clear followed by read

If no intervening write overlaps the cleared word and no external mutation occurs, its later read yields zero. The ensuing result-based offset-0x54 store reaches absolute address 0x54 through an explicitly typed word access.

### Captured global base

If the captured global-derived base later changes in storage, the repeated region still uses its retained value. Moving the capture after the assignment to absolute storage 0x60 changes the specified ordering.

### Initial repeated group

For original-link value 0x100 and defined arithmetic, the adjusted unit is 0x36a. Initial and subsequent offset-0x54 destinations are 0x854 and 0x1ba4, with unchanged payload. The global-derived offset-0x34 data instead changes from 0x800 to 0x1b50.

### Separate 0x67 values and destination widths

Suppose the second and third reads from 0x67 yield values whose low bytes are 0xcd and 0xef. The corresponding typed work-base offset-0x09 stores write those bytes respectively.

Their source read widths remain unresolved. The first read’s assignment to 0x21 does not establish a byte store there. Assigning an initial low-byte value of 0xab to 0x67 likewise does not establish that only one memory byte changes.

### Character arithmetic and variable shift

If the adjusted unit’s character conversion yields 0x40, the arithmetic value supplied to storage 0x3a is 0x230 before any destination conversion. A final value of 0x30 requires an additional narrowing assumption.

A variable shift count of at least 32 has no specified result here. Reaching F-opaque-03 establishes neither successful completion nor a success status.

## Uncertainties

- **documented_only — purpose and ABI:** application meaning, entry convention, and complete call conventions.
- **documented_only — arithmetic:** signed overflow, character signedness, signed shifts, address wrapping, and oversized shift counts.
- **documented_only — absolute globals:** absent declarations leave access widths, signedness, assignment conversions, and precise overlap unresolved.
- **documented_only — layout:** endianness, alignment, object boundaries, permissions, and address validity.
- **documented_only — trace coverage:** complete call enumeration, summarized intermediate accesses, and exact repeated-group count.
- **documented_only — expression ordering:** relative ordering of constituent reads in some compound expressions.
- **documented_only — resources and completion:** dependency effects, preservation, ownership, faults, concurrency, reachability, and final-transfer return behavior.

These uncertainties do not erase explicit typed access widths, the capture-before-0x60 ordering, the initial-group destination/data distinction, or the three separate reads of 0x67.

## Evidence

The supplied source representation and dependency map are the sole behavioral evidence. Prior prose, comments, and reviewer feedback are not independent authority.

Explicit pointer types support the stated byte, halfword, and word accesses. Explicit conversions support value narrowing or extension only where the operand type establishes it. Bare absolute-global expressions lack declarations and therefore do not independently establish underlying memory-access widths.

Static evidence supports the retained dependencies, ordered global capture, differing initial repeated group, and three separate concluding reads of 0x67.

No runtime trace, dependency implementation, ABI specification, or memory map was supplied. No runtime tests have been performed.
