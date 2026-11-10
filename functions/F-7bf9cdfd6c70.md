<!-- Trusted provenance: F-7bf9cdfd6c70; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7bf9cdfd6c70

## Purpose

F-7bf9cdfd6c70 exposes a condition-dependent fragment of machine-state processing: an optional byte read and coprocessor load, repeated coprocessor operations, and an optional 48-byte state-record write.

The supplied static representation ends at an invalid-instruction boundary. It does not establish a complete callable routine, a successful return, or a higher-level purpose. In particular, the state record must not be described as a stack frame, exception context, serialization format, or resumable checkpoint without additional evidence.

This contract distinguishes behavior explicitly represented in the supplied material from unknown machine-level effects. It is a source-facing specification, not a claim that the fragment has been executed or that the recovered control flow is correct.

## Interface

### Explicit inputs

The represented interface has four ordered inputs, each carrying 32 bits of unspecified data.

| Input | Established use |
|---|---|
| First | No explicit use in the visible fragment. |
| Second | No explicit use in the visible fragment. |
| Third | Written into the lowest-addressed word of the conditional state record. |
| Fourth | Written into the next word of that record. |

No signedness, pointer interpretation, ownership transfer, or validation requirement is established for these inputs. The first two inputs must not be described as meaningful controls merely because they appear in the interface.

### Implicit execution-state inputs

The fragment additionally depends on state not supplied through those four inputs:

- Entry negative, overflow, and zero condition indicators.
- An incoming byte-address value.
- An incoming base-address value used for both a coprocessor-load address and the conditional state record.
- An incoming 32-bit value that may be replaced by the zero-extended byte read.
- Five other incoming general-state words.
- An incoming scratch-state word.
- An incoming link-state word.
- Two coprocessor-related operands.

These are descriptive roles, not additional formal parameters. The representation does not establish a conventional calling convention or a self-contained way for an ordinary caller to initialize all required state.

For the following sections:

- **Condition match** means that the entry negative and overflow indicators are equal.
- **Zero condition** means that the entry zero indicator is asserted.
- **Byte address** means the incoming byte-address value, or its adjusted value where specified.
- **Base address** means the incoming address used to locate the load operand and state record.
- **Working word** means the incoming 32-bit value that the optional byte read replaces.

The visible predicates use entry condition indicators. The representation does not show them being refreshed after a memory access or coprocessor operation.

### Outputs and completion

No ordinary return value is established. The represented interface is void-like, and every visible path reaches the invalid-instruction boundary rather than an explicit normal return.

Observable outputs may include memory writes, coprocessor effects, memory-access effects, and abnormal termination or control transfer. Their exact machine-level realization is only partly established.

## Behavior

### Conditional byte access and coprocessor load

When condition match holds, the fragment performs the following ordered effects:

1. It decreases the byte address by 0x15DC, or 5,596 bytes.
2. It reads one byte at the resulting address.
3. It replaces the working word with that byte interpreted as an unsigned value and extended to 32 bits.
4. It requests a coprocessor load using selector 5, the first coprocessor-related operand, and an address 0x1DC, or 476 bytes, above the base address.

The one-byte read therefore produces a working-word value in the inclusive range 0 through 255. There is no sign extension.

The address adjustment precedes the byte read. Both precede the coprocessor load. If the byte read fails, the representation does not establish that the later load or any subsequent operation occurs.

When condition match does not hold, this entire group is omitted. In that case, the byte address and working word retain their incoming represented values.

The coprocessor-load operand address is explicit, but the transfer size, access mode, destination-state layout, and actual memory transaction sequence are not established. It must not be specified as an ordinary four-byte read.

### Initial coprocessor group

When either the zero condition holds or condition match does not hold, the fragment requests four consecutive coprocessor operations.

Each request has the same displayed operands: selector 0xE, two fields with value 7, and the second coprocessor-related operand supplied in both remaining positions.

The four occurrences are distinct operations. Their repeated appearance does not justify collapsing them into one operation or treating them as no-ops.

If the earlier conditional byte access and coprocessor load execute, they precede this group. This ordering matters when the zero condition and condition match both hold.

### Conditional state-record write

When the zero condition holds, the fragment issues twelve 32-bit writes below the base address.

The record occupies offsets −48 through −1 relative to that base. The table gives both the layout and the represented write order; writes proceed from the highest-addressed word toward the lowest-addressed word.

| Write order | Offset from base | Width | Represented value |
|---:|---:|---:|---|
| 1 | −4 | 32 bits | Fixed literal 0x08463FCC |
| 2 | −8 | 32 bits | Incoming link-state word |
| 3 | −12 | 32 bits | Incoming scratch-state word |
| 4 | −16 | 32 bits | First auxiliary state word |
| 5 | −20 | 32 bits | Working word |
| 6 | −24 | 32 bits | Second auxiliary state word |
| 7 | −28 | 32 bits | Byte-address value |
| 8 | −32 | 32 bits | Third auxiliary state word |
| 9 | −36 | 32 bits | Fourth auxiliary state word |
| 10 | −40 | 32 bits | Fifth auxiliary state word |
| 11 | −44 | 32 bits | Fourth explicit input |
| 12 | −48 | 32 bits | Third explicit input |

The auxiliary-word labels distinguish otherwise unspecified incoming values. They do not imply a public field schema beyond this observed layout.

When condition match also holds, the stored working word is the zero-extended byte obtained earlier, and the stored byte-address value is the adjusted address. When condition match does not hold, these two slots instead receive their incoming represented values.

The record is not shown as an atomic transaction. If a store faults, earlier stores may already be visible. No rollback, fault recovery, or all-or-nothing guarantee is present.

The fixed literal is established only as a stored 32-bit value. Although it has an address-like form, its meaning is unknown.

### Final coprocessor group and terminal boundary

Under the same predicate used for the initial coprocessor group—zero condition asserted or condition match absent—the fragment requests another 27 consecutive operations with the same displayed operands.

When the state record is written, this final group follows all twelve stores. When the zero condition is not asserted, no state-record writes intervene between the four-operation group and the 27-operation group.

Thus, any path selecting these groups contains 31 repeated coprocessor-operation requests in total, assuming execution continues through all of them.

After the conditional groups, every represented path reaches an invalid-instruction termination marker. That marker establishes the end of the available representation, not a portable guarantee of a particular exception, process exit, trap code, or permanent halt.

## Side effects

### Memory

The directly represented ordinary-memory effects are:

- One one-byte read when condition match holds.
- Twelve four-byte writes when the zero condition holds.

For nonwrapping, ordinary address calculations, the writes cover a contiguous 48-byte region immediately below the base address. No explicit write updates the base address itself.

No alignment check, bounds check, null check, buffer-size check, or address-validity check is shown. Execution without a memory fault therefore requires suitable readable and writable mappings for the selected operations. The target’s alignment rules remain unknown.

If the byte-read address overlaps the state-record region, the explicit byte read occurs before the state-record writes. Consequently, absent intervening changes from opaque operations, the working word records the byte observed before those stores.

No byte order is established for the 32-bit stores. The table specifies word values and offsets, not their individual byte encodings.

### Coprocessor and hidden state

The fragment explicitly requests a coprocessor load and repeated coprocessor operations. Their effects on coprocessor state, general state, memory, condition indicators, privilege state, or external devices are unknown.

The representation supplies no basis for assuming these operations preserve globals or all other state. The described layout reflects the visible operand relationships; a stronger machine-level guarantee requires the instruction semantics and execution environment.

### Ownership and resources

No allocation, deallocation, reference-count change, or explicit ownership transfer is visible. The fragment accesses caller- or environment-supplied addresses without an established ownership contract.

Absence of an explicit release is not proof that opaque operations cannot affect resource ownership. No such behavior is established either way.

## Dependencies

No ordinary callee with a separate opaque function ID is identified in the supplied material. The only mapped function identity is F-7bf9cdfd6c70 itself.

The fragment depends on three classes of environmental primitive:

1. **Coprocessor load:** requires target-specific interpretation of its selector, operand, address, transfer width, and fault behavior.
2. **Repeated coprocessor operation:** requires interpretation of the displayed operand tuple and of each operation’s side effects.
3. **Invalid-instruction boundary:** requires the actual instruction bytes and execution environment to determine whether execution traps, terminates, transfers elsewhere, or has been represented incorrectly.

These are unresolved primitives, not documented application APIs. No dependency contract is available that establishes preservation of globals, memory ownership, successful completion, or exception handling.

## Acceptance examples

These are conditional acceptance examples derived from static evidence. They are not runtime test results. Each assumes that preceding operations complete and that the supplied control-flow representation is accurate.

### Condition match, zero condition absent

With equal negative and overflow indicators and the zero condition clear:

- The byte address is decreased by 5,596.
- One byte is read and zero-extended into the working word.
- The coprocessor load is requested at base plus 476.
- Neither repeated coprocessor group is selected.
- No state-record stores occur.
- Execution reaches the terminal boundary in the representation.

For example, reading a byte with value 0xFF produces a working word of 0x000000FF, not 0xFFFFFFFF.

### Condition mismatch, zero condition absent

With unequal negative and overflow indicators and the zero condition clear:

- No byte-address adjustment, byte read, or coprocessor load occurs.
- Four repeated coprocessor operations are followed by 27 more.
- No state-record stores occur.
- The terminal boundary follows.

The incoming byte address must not be dereferenced merely because it is present in execution state.

### Condition match, zero condition asserted

With equal negative and overflow indicators and the zero condition asserted:

- The adjusted-address byte read and coprocessor load occur first.
- Four repeated coprocessor operations follow.
- Twelve record stores occur in the specified descending-address order.
- The working-word slot receives the zero-extended byte.
- The byte-address slot receives the adjusted address.
- The final 27 repeated operations follow the stores.

This example selects all visible groups.

### Condition mismatch, zero condition asserted

With unequal negative and overflow indicators and the zero condition asserted:

- The byte-read/load group is omitted.
- Four repeated coprocessor operations precede the record stores.
- The working-word and byte-address slots receive their incoming represented values.
- The final 27 repeated operations follow the stores.

This distinguishes the zero condition’s control over record creation from condition match’s control over byte acquisition.

### Memory failure and partial effects

If the selected byte read encounters an invalid address, successful completion of the later coprocessor load is not required by the visible ordering.

If the fifth record store fails, the representation allows the first four stores to have occurred already. It provides no basis for requiring rollback or completion of the remaining stores.

## Uncertainties

The following are **documented_only**:

- Whether the supplied entry point is a genuine callable-function boundary.
- Whether the displayed predicates and instruction sequence accurately represent executable code.
- The semantics and privilege requirements of the coprocessor operations.
- The coprocessor load’s transfer size and destination format.
- The actual terminal behavior at the invalid-instruction boundary.
- The target’s endianness and alignment requirements.
- Architectural address width and overflow behavior.
- Preservation of globals, hidden registers, memory, and resources across opaque operations.
- The intended meaning of the fixed literal and state-record fields.

The visible data-word width is 32 bits. The byte-address slot is also represented as a four-byte store, supporting a 32-bit address representation for this fragment. That does not establish a complete machine ABI.

Address expressions use fixed byte offsets. Their behavior near address-space boundaries must not be inferred from ordinary integer arithmetic: neither trapping nor modular wraparound is established. Conditional examples therefore assume valid, nonwrapping address calculations.

A context-saving or low-level state-management purpose is plausible inferred intent, but it is not an established behavioral guarantee.

## Evidence

Established static evidence consists of the supplied interface, condition expressions, byte access, coprocessor-operation requests, twelve explicitly ordered word stores, and terminal invalid-instruction marker.

The source material itself warns that invalid instruction data affected control-flow recovery. That warning limits confidence in treating the representation as a complete machine-level contract; it does not justify replacing visible behavior with a guessed implementation.

The operation counts, record size, offsets, and zero-extension behavior above are derived from that static representation. No runtime tests, instruction decoding, memory observations, or independent implementation checks have been performed.
