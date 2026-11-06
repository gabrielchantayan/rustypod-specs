<!-- Trusted provenance: F-9afb51fb96af; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9afb51fb96af

## Purpose

F-9afb51fb96af conditionally fills four caller-supplied 32-bit output locations using a dependency associated with a context record. A one-byte context field selects the dependency requests and determines which outputs receive their results.

The function returns its original context-address argument on every visible normal-return path. That return value is not a success indicator: an absent associated value, an unsupported mode, and completed output population all return the same argument.

The established behavior is a sequence of memory accesses, dependency calls, and output writes. The semantic meanings of the context, mode, associated value, selectors, and returned output words are not established.

## Interface

### Inputs and return value

The visible interface has five arguments:

| Argument | Role |
|---|---|
| Context address | A 32-bit value used as the base address for context-field accesses and returned unchanged on normal completion. |
| Output A | Address of a writable 32-bit output location. |
| Output B | Address of a writable 32-bit output location. |
| Output C | Address of a writable 32-bit output location. |
| Output D | Address of a writable 32-bit output location. |

The context-address argument is represented as a 32-bit integer in the supplied material, although it is used to address memory. Its pointer provenance, signed-address interpretation, and platform-level validity rules are not established. The returned value preserves the original argument; it is not reread from the context.

Output values are opaque 32-bit words. The function does not interpret their sign, perform arithmetic on them, or narrow them before storage.

### Context layout

Only two context fields are directly accessed:

| Byte offset from context base | Access width | Observed use |
|---|---:|---|
| 10 | 1 byte | Mode selection. |
| 80 | 4 bytes | Initial presence test and first argument for dependency calls. |

The offset-80 field is initially tested for zero. A nonzero value permits output initialization and mode processing. When a dependency call is made, that field is read again to obtain the call argument.

The mode comparisons distinguish byte values 0, 1, 2, and 3. Every other byte value follows the unsupported-mode path. Whether the underlying one-byte type is signed does not affect those four comparisons or the disposition of other byte values.

No complete record layout, total object size, or meaning for intervening bytes is established.

### Preconditions and validity

For defined execution of the represented memory accesses:

- The context must permit a 4-byte read at offset 80.
- If that initial read is nonzero, all four output locations must permit 4-byte writes, and the context must permit a one-byte read at offset 10.
- Each selected dependency call must accept the actual value passed to it.
- Additional memory accesses performed by the dependency require whatever validity conditions that dependency imposes.

The visible function performs no null checks, bounds checks, or explicit alignment checks.

If the initial offset-80 value is zero, this function does not access the outputs or the mode field. Consequently, their validity is not needed for this function’s visible accesses on that path.

The source does not establish valid address arithmetic for a context base near an address-space boundary. No wraparound-address guarantee is part of this contract.

## Behavior

### Initial gate and output initialization

The first visible memory operation is a 4-byte read of the context field at offset 80.

If the value is zero:

- No output is written.
- The mode field is not read.
- No dependency is called.
- The original context-address argument is returned.

If the value is nonzero, the function writes 32-bit zero to every output, in this order:

1. Output D.
2. Output C.
3. Output B.
4. Output A.

Only after all four writes does it read the mode byte at context offset 10.

This order matters when output locations overlap one another or overlap context fields. In particular, the mode used for dispatch is the byte present after output initialization, not necessarily the byte present on entry.

The mode is read once for dispatch. Later dependency behavior does not cause the function to select a different branch.

### Mode-dependent processing

F-6d4e5f4e1b57 is the sole visible dependency. Each call takes two arguments: a freshly read 32-bit value from context offset 80 and a selector. Its returned 32-bit word is then written to one or more outputs.

For nonoverlapping outputs and context storage, and assuming no other actor changes output memory, the direct results are:

| Mode | Selector sequence | Output A | Output B | Output C | Output D |
|---:|---|---|---|---|---|
| 0 | 0, 2, 3 | Result for 0 | Result for 2 | Result for 3 | Zero |
| 1 | 0, 1 | Result for 0 | Result for 1 | Result for 0 | Result for 1 |
| 2 | 4 | Result for 4 | Zero | Zero | Zero |
| 3 | 5 | Result for 5 | Zero | Zero | Zero |
| Other | None | Zero | Zero | Zero | Zero |

Each “result” means the value returned by that particular call. It does not imply that a selector always produces the same value.

#### Mode 0

The observable call-and-write sequence is:

1. Read context offset 80 and call F-6d4e5f4e1b57 with selector 0.
2. Write the returned word to Output A.
3. Read context offset 80 again and call the dependency with selector 2.
4. Write that returned word to Output B.
5. Read context offset 80 again and call the dependency with selector 3.
6. Write that returned word to Output C.

There is no further direct write to Output D after initialization.

#### Mode 1

The observable sequence is:

1. Read context offset 80 and call F-6d4e5f4e1b57 with selector 0.
2. Write the returned word to Output C.
3. Write that same returned word to Output A.
4. Read context offset 80 again and call the dependency with selector 1.
5. Write that returned word to Output D.
6. Write that same returned word to Output B.

Only two dependency calls occur. The paired output assignments use a single returned word per pair; they do not request duplicate results from the dependency.

#### Modes 2 and 3

Mode 2 makes one dependency call with selector 4. Mode 3 makes one dependency call with selector 5.

In either case, context offset 80 is read for the call’s first argument, and the returned word is written only to Output A after initialization.

#### Other modes

Every other mode value causes an immediate normal return after output initialization. There is no dependency call and no additional direct output write.

### Repeated reads and intervening changes

The initial nonzero test does not cache the associated value for later calls. The field at context offset 80 is reread before each dependency invocation.

Therefore:

- Different calls within one invocation may receive different first arguments.
- A dependency-induced change to that field can affect a later call.
- An output write that overlaps that field can affect a later call.
- A later reread may produce zero even though the initial test was nonzero.

There is no second presence test. A zero obtained by a later reread is passed to the dependency rather than causing the call to be skipped.

### Aliasing

The visible sequence does not impose or check a distinct-output requirement. Where overlapping accesses are valid in the execution environment, their consequences follow the stated write order.

Output names are destinations, not guarantees of independently retained values. A write through one output may overwrite storage also named by another output.

Likewise, output initialization may alter the context before the mode read or before dependency arguments are obtained. The simple result table therefore applies only under its stated nonoverlap and noninterference assumptions.

No atomicity, synchronization, or concurrent-access guarantee is established.

## Side effects

The direct side effects are:

- A 4-byte read at context offset 80.
- On the nonzero path, four ordered 4-byte zero writes.
- On that same path, a one-byte mode read.
- Depending on the mode, one, two, or three additional reads at context offset 80.
- Dependency calls and the ordered output writes described above.

The visible body contains no direct allocation, deallocation, ownership transfer, or explicit update to the context through a dedicated context-field assignment. However, output aliasing can modify context storage, and dependency calls may have additional effects.

The function does not establish ownership semantics for either the context or the outputs. Its returned context-address value does not itself imply a transfer, retention, or release.

There is no explicit error channel. Dependency return values are stored as opaque words without checking for failure markers. Unsupported modes are not reported through the return value. Invalid memory access, dependency failure, nonlocal control transfer, or other abnormal execution is outside the established normal-return behavior.

The sequence is not transactional. Output initialization occurs before any dependency call, and earlier result writes precede later calls. No rollback mechanism is visible.

## Dependencies

### F-6d4e5f4e1b57

Established call contract at this boundary:

- First argument: the 32-bit word read from context offset 80 immediately for that call.
- Second argument: one of the selectors 0, 1, 2, 3, 4, or 5.
- Return: a 32-bit word copied to the selected output locations.
- Call order: fixed by the selected mode.

The dependency’s semantic purpose, accepted first-argument domain, error conventions, memory ownership, and side effects are unknown.

In particular, this contract does not assume that the dependency preserves globals, context storage, output storage, or external resources. It also does not assume that the dependency acquires, retains, or releases memory. Any acceptance example requiring preservation states that condition explicitly.

If a dependency call does not return normally, the subsequent operations described for that branch are not established to occur.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They are not runtime observations.

### Absent associated value

Given a readable offset-80 field containing zero and valid execution of that read, the function returns the original context-address argument. Existing output contents remain unchanged by this function, and no dependency call occurs.

The mode value is irrelevant because it is not read.

### Mode 0 with stable, separate storage

Given an initially nonzero associated value, mode 0 after initialization, distinct output storage, and a dependency that preserves the relevant context and outputs:

- The selectors are requested in order 0, 2, 3.
- If those calls return 11, 22, and 33 respectively, the final outputs are A = 11, B = 22, C = 33, D = 0.
- The original context-address argument is returned.

### Mode 1 with paired results

Under the same separation and preservation conditions, mode 1 requests selectors 0 and 1, in that order.

If the returns are 17 and 29, the direct write sequence after initialization is C = 17, A = 17, D = 29, B = 29. The final outputs are A = C = 17 and B = D = 29.

### Unsupported mode

Given an initially nonzero associated value, separate output storage, and a mode byte of 7 after initialization, all four outputs are set to zero. No dependency call occurs, and the original context address is returned.

### All outputs share one word

Given that all four outputs refer to one valid 32-bit word that does not overlap the context, and the dependency leaves that word untouched except through this function’s result stores:

- In mode 0, the final shared word is the result of the selector-3 call.
- In mode 1, the final shared word is the result of the selector-1 call.

These outcomes follow from store order, not from four independent output values.

### Associated value changes between calls

Given mode 0 and otherwise separate storage, suppose the first dependency call changes context offset 80 from value H to value J and returns normally.

The selector-2 call receives J if the field remains J until its argument read. It does not receive a cached H. If J is zero, selector 2 is still called with zero.

### Initialization changes the mode

Given an initially nonzero offset-80 field, suppose a valid output write overlaps the mode byte and causes it to become zero, without altering the associated-value field.

Dispatch uses mode 0, regardless of the mode byte’s entry value, because initialization precedes the mode read.

## Uncertainties

The following are documented_only:

- The semantic meanings of the context fields, selectors, and output words.
- The dependency’s implementation, failure behavior, ownership rules, and global or memory side effects.
- The complete context layout and required object extent beyond the accessed fields.
- Endianness, alignment requirements, memory-protection behavior, and treatment of otherwise invalid overlapping accesses.
- Whether external callers impose stronger validity, lifetime, or nonaliasing preconditions.
- Behavior under concurrent mutation or abnormal dependency termination.
- Whether the apparent modes correspond to named domain concepts.

An inferred intent is to populate mode-specific groups of related values from an associated object. That interpretation is not needed for conformance and does not establish any domain-specific meaning.

## Evidence

The supplied static function body establishes the field offsets and access widths, initial zero gate, reverse output-initialization order, post-initialization mode read, selector sequences, repeated associated-value reads, result-store order, and unchanged context-address return.

The supplied dependency mapping identifies the external call as F-6d4e5f4e1b57. No dependency body or caller contract was supplied.

No runtime tests have been performed. Acceptance examples are conditional predictions from the supplied static evidence, not claims about exercised behavior.
