<!-- Trusted provenance: F-e7e203ce4d1a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7e203ce4d1a

## Purpose

F-e7e203ce4d1a performs a fixed sequence of byte, halfword, and word writes through explicit and implicit address-bearing inputs, reads additional addresses and values from that memory, updates three fixed-address global locations, and invokes an unresolved indirect destination with four arguments.

The supplied static evidence does not establish an application-level purpose. In particular, it does not establish that the addressed regions are valid objects, that the repeated writes are redundant, or that the final indirect destination is a conventional callback. This contract therefore describes the observable memory behavior without assigning a higher-level role.

The sequence is unconditional in the supplied representation. Completion depends on the validity of the accessed memory, the interpretation of certain arithmetic operations, and the behavior of the indirect destination.

## Interface

### Explicit inputs

| Contract name | Width | Role |
|---|---:|---|
| Primary input | 32 bits | Initially written as a word; also used as the base address for a word read at offset `0x74`. |
| Secondary input | 32 bits | Written to an implicit-state address before that location is repeatedly overwritten. No later explicit use is shown. |

The primary input is represented as a signed integer in the supplied evidence, but is also used as an address. Its valid range as an address is not established.

### Implicit inputs

Three incoming machine-state values are used without initialization inside the function:

| Contract name | Represented width | Role |
|---|---:|---|
| Address A | 32 bits, unsigned | Base address, arithmetic operand, source of low-byte values, and contributor to the shift count. |
| Address B | 32 bits | Base address and source of a low-byte value. |
| Address C | 32 bits | Base address and value stored into memory. |

These are required entry-state inputs, not values derived from the explicit arguments. Their calling convention is **documented_only**: the supplied evidence does not establish how a caller supplies them.

All addresses subsequently read from memory are also required to be usable for the accesses that follow. The function does not establish their validity before use.

### Widths and notation

This contract uses:

- **Byte:** 8 bits.
- **Halfword:** 16 bits.
- **Word:** 32 bits.
- **Low byte:** the least significant eight bits of a value.
- **Doubled A:** the unsigned 32-bit result of multiplying Address A by two, reduced modulo \(2^{32}\).

Three fixed-address locations are distinguished:

- **Global source:** a word read beginning at address `0xA9`.
- **Global byte X:** the byte at address `0x3A`.
- **Global byte Y:** the byte at address `0x67`.
- **Global byte Z:** the byte at address `0x21`.

These names identify accesses, not separate storage ownership domains. Addressed regions may overlap.

Word and halfword byte order, alignment requirements, address-space mapping, and fault behavior are not established.

### Output and completion

No explicit return value is provided.

After its memory operations, the function invokes an unresolved indirect destination. If that invocation returns normally, the supplied representation then returns without further explicit work. Whether the destination actually returns, transfers control elsewhere, or terminates execution is unknown.

## Behavior

### Initial repeated stores

The initial memory activity consists entirely of writes. First, the secondary input is written as a word at Address B plus `0x34`.

The function then performs **eight rounds** of the same ordered set of writes:

1. The primary input is written as a word at Doubled A plus `0x54`.
2. The low byte of Address A is written at Address B plus `0x19`.
3. The low byte of Address B is written at Address A plus `0x11`.
4. Doubled A is written as a word at Address B plus `0x34`.

There are therefore 33 explicit stores in this initial portion: the first secondary-input store and 32 stores across the eight rounds.

The repetition is part of the established access sequence. It must not be reduced to final memory values when repeated accesses can be observed, including through memory-mapped devices or externally monitored memory.

No non-aliasing condition is established. Within a round, later writes can overwrite bytes written earlier. Across rounds, those same effects recur. The address and value operands remain the incoming values; they are not reloaded from the modified memory.

### Initial captures and global-byte calculation

Following the eighth round, the function reads and retains the word at the global source, then reads and retains the word at the primary input plus `0x74`.

It next writes Global byte X. The written byte is:

\[
(2 \times \operatorname{low8}(\text{Address A}) + 0x30) \bmod 256.
\]

The supplied representation performs the intermediate calculation through a character-sized conversion. Signedness of that character type does not change the resulting low eight bits under the ordinary truncating byte-store interpretation.

The function then captures, in order:

- The word at Address C plus `0x14`.
- The word at Address A plus `0x04`.

These captured words are retained for later stores or for the indirect invocation. Later writes to their source locations do not, by themselves, replace the captured values.

### First state-update group

The next writes occur in this order:

- Address C plus `0x0D` receives the low byte of Address A.
- Address A plus `0x08` receives byte `0x6C`.
- Address A plus `0x0D` receives the low byte of Address B.
- Address A plus `0x09` receives the low byte of Address B.
- Address C plus `0x54` receives Address C as a word.
- Global byte Y receives the low byte of the captured global-source word.

Only after these writes does the function read the word at Address C plus `0x44`. Call this the **shift-source word**.

Global byte Y is then read and its byte value written to Global byte Z. This is a read of the global byte at that point, not an unconditional reuse of the earlier captured global-source word.

Next:

- The low byte of the shift-source word is written at Address A plus `0x10`.
- The complete shift-source word is written at Address B plus `0x54`.
- A byte derived by right-shifting the shift-source word is written at Address C plus `0x01`.
- Address B plus `0x54` is overwritten with Address A as a word.

The shift count is the low byte of Address A. The shift-source word is represented as signed. The exact result for negative values or counts outside the ordinary 32-bit shift range is not fully established; see **Uncertainties**.

### Second state-update group

The function now reads the word at Address A plus `0x40`.

It then:

- Writes the previously captured word from Address A plus `0x04` to Address B plus `0x74`.
- Writes the newly read word from Address A plus `0x40` to Address A plus `0x10`.
- Writes Address C as a word at Address B plus `0x44`.
- Reads Global byte Y and writes that byte at Address A plus `0x09`.
- Writes the low byte of Address A at Address C plus `0x09`.
- Writes the originally captured global-source word at Address A plus `0x44`.

The word store at Address A plus `0x10` spans offsets `0x10` through `0x13`. In ordinary byte-addressed memory it therefore overlaps both the earlier byte store at offset `0x10` and the repeated initial byte stores at offset `0x11`. Final byte values within that span depend on byte order and any further aliasing.

### First loaded destination

After the preceding writes, the function reads a word from Address A plus `0x54` and treats it as **destination D**.

It reads Global byte Y again and writes that byte at Address A plus `0x09`. It then:

- Writes the low byte of the captured word from the primary input plus `0x74` at destination D plus `0x0D`.
- Writes the captured word from Address C plus `0x14` back to Address C plus `0x54`.
- Reads and retains the word at Address A plus `0x14`.
- Writes the same primary-input-derived low byte at Address C plus `0x11`.
- Writes the low byte of destination D at Address A plus `0x09`.
- Writes destination D as a word at Address A plus `0x44`.

The store to Address C plus `0x54` uses the earlier capture. It is not a fresh read from Address C plus `0x14`.

The reads of Global byte Y remain distinct observable reads. If another write aliases that global byte, a later read may differ from an earlier one.

### Final address chain and invocation

The function next reads a word at Address C plus `0x04`, treating it as **address E**.

It writes halfword `0x0065` at twice address E. It then reads the word at address E plus `0x64`, retaining it as **value H**.

Before obtaining the final destination, the function writes:

- The low byte of the captured word from Address A plus `0x14` at destination D plus `0x11`.
- Word value `100`, or `0x00000064`, at destination D plus `0x54`.

It next captures the word at address E plus `0x74`, then reads the word at Address C plus `0x64`, treating the latter as **destination J**.

The final three explicit stores are:

- The low byte of address E at Address C plus `0x09`.
- Value H as a word at destination J plus `0x54`.
- Destination J as a word at twice value H.

The final invocation receives four arguments, in this order:

| Position | Value |
|---|---|
| First | `0x72`, decimal 114 |
| Second | Value H |
| Third | The word captured from Address C plus `0x14` before the first state-update group |
| Fourth | The word captured from address E plus `0x74` |

There is no explicit validation between these memory operations and the invocation. The source of the indirect destination itself is unresolved.

## Side effects

### Memory and ordering

The function writes through all three implicit addresses, two subsequently loaded destinations, two doubled loaded values, and three fixed-address global bytes.

The ordering described above is material:

- Reads can observe earlier writes when addresses alias.
- A captured value remains distinct from a later reload of its source.
- Repeated writes remain distinct events even when their values match.
- Word and halfword writes may overlap earlier byte writes.

The supplied evidence does not provide synchronization, atomicity guarantees, volatile-memory semantics, or protection against concurrent modification. The ordered accesses describe the supplied static representation; exact machine-level bus transactions are **documented_only**.

### Ownership

No explicit allocation, release, reference-count operation, or ownership transfer is shown before the indirect invocation.

The function writes address-bearing words into memory, but that alone does not establish ownership transfer. Memory must remain accessible for every reached access. The indirect destination may have additional ownership or lifetime effects; none are established here.

### Errors and partial effects

No explicit error result, validation branch, rollback, or recovery path is shown.

An invalid address, unsupported alignment, prohibited memory access, or failing indirect transfer may interrupt execution after earlier writes have already occurred. The specific exception or platform response is unknown. Successful completion cannot be promised solely from the two explicit input values.

## Dependencies

The supplied dependency map identifies only F-e7e203ce4d1a itself. No other resolved opaque function ID is available.

There is one unresolved indirect dependency at the end of the function. Its four argument values are specified above; its target-selection mechanism, calling convention, return behavior, global-state effects, and memory-ownership effects are **documented_only**.

There are no other explicit calls in the supplied representation. This does not establish that the indirect dependency preserves any memory or global state.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

### Repeated writes are retained

Given valid writable locations and an observer capable of distinguishing stores, the initial portion must expose:

- One word store of the secondary input at Address B plus `0x34`.
- Eight subsequent word stores of Doubled A at that same address.
- Eight stores at each of the other three repeated destinations, in the specified per-round order.

Merely reaching the same final memory contents with fewer stores does not satisfy this access-sequence contract.

### Global-byte arithmetic

If the low byte of Address A is `0x05`, Global byte X receives `0x3A`.

If that low byte is `0xF0`, Global byte X receives `0x10`, because the result is truncated to eight bits.

These cases assume execution reaches the global-byte write.

### Defined-range shift case

If the shift-source word is `0x12345678` and the low byte of Address A is four, the byte written at Address C plus `0x01` is `0x67`.

This case avoids both negative signed operands and out-of-range shift counts.

### Snapshot versus later memory contents

Suppose the word captured from Address C plus `0x14` is `0x11223344`. If intervening writes later change that source location without changing the retained value, both the later word store at Address C plus `0x54` and the third invocation argument use `0x11223344`.

### Final invocation arguments

If the relevant reads produce value H equal to `0x2000`, the captured Address C plus `0x14` word equal to `0x11223344`, and the captured address E plus `0x74` word equal to `0x55667788`, the invocation arguments are, in order:

`0x72`, `0x2000`, `0x11223344`, and `0x55667788`.

This assumes all intervening accesses and arithmetic are valid and execution reaches the invocation.

## Uncertainties

The following remain **documented_only**:

- **Entry convention:** how the three implicit inputs are supplied.
- **Address model:** whether all represented 32-bit values correspond directly to usable addresses.
- **Alignment and byte order:** especially for the fixed-address word read at `0xA9`, overlapping stores, and doubled-address writes.
- **Address addition:** behavior when signed address expressions exceed their representable range.
- **Signed doubling:** twice address E and twice value H are expressed using signed integer values. Behavior on signed overflow is not established. This differs from the explicitly unsigned doubling of Address A.
- **Right shift:** handling of negative shift-source values and counts from 32 through 255.
- **Aliasing and concurrency:** no disjointness or exclusive-access guarantee is supplied.
- **Indirect control transfer:** destination identity, destination selection, return behavior, and effects.
- **Application meaning:** no reliable object layout, protocol, device interface, or business purpose is established.

A successful-path precondition must include valid memory for every reached access and arithmetic behavior supported by the actual execution environment. Restricting signed calculations to their representable ranges avoids some ambiguities, but the evidence does not establish that actual callers obey such restrictions.

## Evidence

**Established static evidence:** the supplied representation shows two explicit 32-bit inputs, three uninitialized incoming state values, the ordered memory accesses described above, eight repeated store rounds, one unresolved indirect invocation, and no explicit result value or error path.

**Inferred intent:** the repeated cross-address stores and loaded address chain may represent state mutation or low-level control preparation. No application-level interpretation is accepted as a requirement.

**Unknowns:** machine execution semantics, complete calling convention, object boundaries, address validity, ownership rules, and indirect-destination behavior are not supplied.

No runtime tests have been performed. The acceptance examples are deductions under stated conditions, not observations of execution.
