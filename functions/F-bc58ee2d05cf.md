<!-- Trusted provenance: F-bc58ee2d05cf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bc58ee2d05cf

## Purpose

F-bc58ee2d05cf modifies a state record using a configuration record, a caller-supplied word, and additional execution context absent from its explicit parameter list.

Two paths are represented:

- **Indirect dispatch:** perform initial state writes, invoke an indirectly addressed dependency, and return its result.
- **Configuration processing:** validate selected configuration fields, initialize state, conditionally process payload fields through F-6d8c85350b66, update a shared counter, invoke F-266a29fc8dc5, and return zero.

Explicit configuration rejection returns the 32-bit word `0xFFFFFFFF`. Rejection does not undo preceding writes.

An initialization or registration purpose is plausible inferred intent, not an established domain guarantee. In particular, the initialization words are not proven to be zero and one, and successful completion does not establish resource creation or ownership.

## Interface

### Explicit inputs and result

| Item | Width | Established use |
|---|---:|---|
| Primary value | 32 bits | Copied to state offset `0x64` before path selection. |
| Dispatch index | Signed 32 bits | Used in signed address arithmetic on the dispatch path. |
| Result | 32 bits | Indirect dependency result, zero on normal configuration completion, or `0xFFFFFFFF` on explicit rejection. |

### Additional execution context

The representation also requires:

- A writable state-record base.
- A readable configuration-record base.
- A 32-bit **baseline word**.
- A 32-bit **marker word**.
- Incoming carry and zero flags.

Their origins and calling convention are unresolved. A conventional two-argument interface alone does not describe all invocation requirements.

The **baseline byte** and **marker byte** mean the low eight bits of their respective words. Neither word has an established numerical value.

### Configuration record

Offsets are byte offsets from the configuration base.

| Offset | Access width | Use |
|---|---:|---|
| `0x00` | 8 bits | Mode selection. |
| `0x08` | 32 bits | Unsigned range comparisons. |
| `0x0C` | 32 bits | Signed equality comparisons with one and two; later copied to state. |
| `0x14` | 32 bits | Initial source for state `0x54`. |
| `0x28` | 32 bits | Initial source for state `0x74`. |
| `0x40` | 32 bits | Payload copy; this address is also passed to F-6d8c85350b66. |
| `0x44`, `0x48`, `0x50` | 32 bits each | Adjustable payload values. |

F-6d8c85350b66 may access more storage than these directly observed fields.

### State record

| Offset or offsets | Access width | Use |
|---|---:|---|
| `0x04`, `0x05` | 8 bits each | Baseline initialization. |
| `0x08`, `0x0C` | 32 bits each | Baseline initialization. |
| `0x14` | 32 bits | Configuration-field copy. |
| `0x28` | 8 bits | Optional-processing status. |
| `0x2C` | 32 bits | Payload copy or baseline assignment. |
| `0x30`, `0x34`, `0x38` | 32 bits each | Adjustable payload or baseline assignments. |
| `0x3C` | 32 bits | Retained, replaced, or incremented state. |
| `0x40`, `0x41` | 8 bits each | Range- and selection-stage initialization. |
| `0x45`, `0x46` | 8 bits each | Marker and baseline initialization. |
| `0x4C` | 32 bits | Baseline initialization. |
| `0x54` | 32 bits | Initial configuration copy. |
| `0x58` | 8 bits | Discriminator read; no direct write. |
| `0x64` | 32 bits | Primary-value copy. |
| `0x74` | 32 bits | Initial configuration copy. |
| `0x78` | 32 bits | Mode-dependent base word. |
| `0x7C`, `0x80` | 32 bits each | Derived values or baseline assignments. |
| `0x84`, `0x88` | 32 bits each | Initial baseline assignments. |
| `0x8C`, `0x8D` | 8 bits each | Completion baseline byte and initial marker byte. |

Unlisted state fields have no direct write established by this body.

### External storage and preconditions

The function references four unresolved external storage locations:

- A 32-bit word read for mode one.
- A separate 32-bit word read for mode seven.
- A pointer-valued location supplying the shared-counter address.
- A location supplying the first argument to F-266a29fc8dc5.

These are storage reads, not established immutable constants. Distinct mode-word locations do not imply distinct values. The counter pointee is accessed as a signed 32-bit word. The external pointer-slot width and completion-argument width are not independently specified by the supplied representation.

Invocation requires accessible storage for every reached access and a valid target for any reached indirect invocation. No null, bounds, alignment, overlap, or synchronization checks are visible. Endianness and supported aliasing arrangements are not established.

## Behavior

### Initial writes

Before either execution path is selected, the following accesses occur in order:

1. Write the primary value to state `0x64`.
2. Read configuration `0x14`; write that word to state `0x54`.
3. Read configuration `0x28`; write that word to state `0x74`.
4. Write the baseline word to state `0x88`.
5. Write the baseline word to state `0x84`.
6. Write the marker byte to state `0x8D`.

These writes remain observable if later validation rejects.

### Entry-condition dispatch

Dispatch occurs when incoming carry is clear **or** incoming zero is set. Only carry-set, zero-clear execution enters configuration processing.

The represented dispatch target is an address computed from the fixed address base `0x0800665C` plus four times the signed 32-bit dispatch index. The multiplication and addition are represented as signed 32-bit operations.

The representation invokes the computed address directly and returns its 32-bit result. It does **not** establish an intervening read of a target pointer from a table.

No explicit arguments are shown for this invocation, but its effective execution-context contract is unknown. Index bounds, executable target validity, signed-overflow behavior, and the fidelity of the represented call to the original control transfer remain unresolved. The address spacing alone does not establish a pointer table or ordinary callable-function entries.

### Range-dependent initialization

On the configuration-processing path:

1. Read configuration `0x08` as an unsigned 32-bit word.
2. Write the baseline byte to state `0x40`.
3. If the captured word exceeds eight, read configuration `0x08` again.
   - If the second read exceeds sixteen, return `0xFFFFFFFF`.
   - Otherwise, write the marker word to state `0x3C`.
4. If the first read is at most eight, leave state `0x3C` unchanged at this stage.

With stable storage, zero through eight retain the existing state word, nine through sixteen replace it, and larger values reject.

The two reads are distinct. The write to state `0x40` occurs between them, so aliasing can affect the second read. A first value above sixteen does not by itself establish rejection if the second read differs.

### One-or-two selection

Read configuration `0x0C` as a signed 32-bit value.

- If it equals one, write the marker byte to state `0x41`.
- Otherwise, write the baseline byte to state `0x41`, then read configuration `0x0C` again.
  - If this second read is not two, return `0xFFFFFFFF`.
  - If it is two, read state `0x3C`, add one using the represented signed 32-bit arithmetic, and write the result back.

With stable configuration storage, only one and two pass. The increment operates on either the retained state word or the marker word installed by the range stage.

Signed overflow is not assigned a guaranteed wrapping meaning by this contract.

### Common initialization

After selection succeeds, perform these accesses in order:

1. Read configuration `0x0C` again and copy it to state `0x14`.
2. Write the baseline word to state `0x08`.
3. Write the baseline word to state `0x0C`.
4. Write the baseline byte to state `0x05`.
5. Write the baseline byte to state `0x04`.
6. Write the baseline word to state `0x4C`.
7. Write the marker byte to state `0x45`.
8. Write the baseline byte to state `0x46`.

The value copied to state `0x14` is a fresh read. It need not equal a previously validated value when storage changes between accesses.

### Mode selection and external word reads

Next, read the configuration byte at `0x00`.

| Mode | Ordered action |
|---:|---|
| 0, 3, 5 | Write the baseline word to state `0x78`; continue. |
| 1 | Read the mode-one external 32-bit word, then write the captured word to state `0x78`; continue. |
| 7 | Read the mode-seven external 32-bit word, then write the captured word to state `0x78`; continue. |
| Every other byte value | Return `0xFFFFFFFF`. |

The external word reads occur after common initialization and mode selection, immediately before their corresponding state assignment. Earlier writes may therefore affect the value read if storage aliases.

Modes one and seven read distinct external locations. Their values may be equal, may differ, and are not established as immutable.

Mode rejection does not directly write state `0x78`, invoke either identified dependency, or increment the shared counter.

### Optional payload processing

For each accepted mode, invoke F-6d8c85350b66 once with the address of configuration `0x40`. Its represented result is a signed 32-bit value, tested only for equality to zero.

No preservation guarantee is assumed for configuration, state, or external storage. Subsequent memory reads use the contents present after the call.

**Zero result:** write, in order:

1. Baseline byte to state `0x28`.
2. Baseline word to state `0x2C`.
3. Baseline word to state `0x30`.
4. Baseline word to state `0x34`.
5. Baseline word to state `0x38`.
6. Baseline word to state `0x7C`.
7. Baseline word to state `0x80`.

These are not necessarily zero assignments.

**Nonzero result:** perform, in order:

1. Write the marker byte to state `0x28`.
2. Read configuration `0x40`; copy it to state `0x2C`.
3. Read configuration `0x44`; copy it to state `0x30`, retaining the read value.
4. Read configuration `0x48`; copy it to state `0x34`, retaining the read value.
5. Read configuration `0x50`; copy it to state `0x38`, retaining the read value.
6. Read the byte at state `0x58`.

If the discriminator equals seven, shift each of the three retained values left by one bit and write the results to state `0x30`, `0x34`, and `0x38`, in that order. These operations use the retained configuration reads, not fresh reads of the destination fields.

The shifts are represented as signed 32-bit operations. Negative operands and results outside the signed representable range have no portable outcome established here.

Then:

1. Read state `0x78` as an unsigned 32-bit base and retain it.
2. Read state `0x30`, add it to the retained base using unsigned 32-bit arithmetic, and write the sum to state `0x7C`. This addition wraps modulo \(2^{32}\).
3. Read state `0x34` as unsigned and compare it with the retained base.
4. If the base is smaller, read state `0x34` again and subtract the base using unsigned 32-bit arithmetic. Otherwise select zero.
5. Write the selected result to state `0x80`.

The subtraction result passes through a represented signed 32-bit temporary. Portable conversion behavior for unsigned results above the signed maximum is not established. For stable storage and representable differences, state `0x80` receives the positive excess of state `0x34` over the base, or zero.

There is no corresponding derived calculation using state `0x38`.

### Completion and external-read ordering

After either optional-processing branch:

1. Read the counter-pointer external storage location and retain the pointer value obtained.
2. Write the baseline byte to state `0x8C`.
3. Read the signed 32-bit counter through the retained pointer.
4. Add one and write the result through that same retained pointer.
5. Read the external storage supplying the completion-call first argument.
6. Invoke F-266a29fc8dc5 with that captured argument and a second argument of zero.
7. Return the 32-bit word zero if the invocation returns normally.

The counter pointer is captured **before** the state `0x8C` write. That write can affect the subsequent counter read if it aliases the counter, but a change to the external pointer slot does not cause a pointer reload.

The completion argument is read **after** the counter update. It therefore reflects any applicable aliasing effect of that update rather than a value captured earlier.

The counter increment is signed 32-bit arithmetic. Overflow handling, atomicity, and synchronization are unresolved. F-266a29fc8dc5’s return value does not determine this function’s result.

## Side effects

Direct effects comprise the ordered state writes, external storage reads, counter read-modify-write, and dependency invocations described above.

There is no rollback:

- Range rejection preserves the initial writes and state `0x40` assignment.
- Selection rejection additionally preserves any state `0x3C` replacement and the baseline assignment to state `0x41`.
- Mode rejection preserves common initialization.
- Completion effects occur only after optional processing finishes.

No direct configuration write, allocation, release, or ownership transfer is visible. This does not prove that dependencies neither retain pointers nor release memory.

There is no established concurrency or reentrancy guarantee. Access ordering matters for aliased storage, but the representation does not establish a language-level volatile or atomic access contract.

Dependencies may change previously written state or external storage. Consequently, direct assignments describe effects at their execution points, not unconditional final postconditions after arbitrary dependency behavior.

## Dependencies

| Opaque dependency | Established interaction | Unresolved behavior |
|---|---|---|
| F-6d8c85350b66 | Receives configuration base plus `0x40`; zero/nonzero result selects payload processing. | Access extent, mutations, pointer retention, ownership, blocking, and failure behavior. |
| F-266a29fc8dc5 | Receives a freshly read external argument and zero after the counter update; result ignored. | Argument meaning and width, mutations, ownership, blocking, and failure behavior. |
| Unresolved indirect dependency | Invoked at the computed dispatch address; returned word propagated. | Target validity, effective arguments, context requirements, and original transfer semantics. |

External data dependencies are the two mode-word storage locations, counter-pointer storage, completion-argument storage, and the fixed dispatch-address base. Only the dispatch-address base is represented as an immediate fixed address; the other dependencies require memory reads.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless stated otherwise, assume valid, stable, non-overlapping storage, normal dependency returns, no dependency mutations, and a counter increment within signed range.

1. **Dispatch precedence.** Carry clear selects dispatch after the six initial writes, even if the configuration mode would reject during configuration processing. For representable address arithmetic and a valid target, the returned word is the indirect dependency’s result.

2. **Stable upper-range rejection.** Carry set, zero clear, and range seventeen cause both range reads to exceed their respective thresholds. The function writes state `0x40`, then returns `0xFFFFFFFF` without reaching selection.

3. **Changing range reads.** If the first range read is seventeen and the second is sixteen, range processing replaces state `0x3C` with the marker word and continues. The first read alone does not cause rejection.

4. **Retained state increment.** Range eight, selection two, and existing state `0x3C` equal to twelve produce thirteen before mode processing.

5. **Replacement and rejection.** Range nine and selection three replace state `0x3C` with the marker word, assign the baseline byte to state `0x41`, and reject before common initialization.

6. **Equal external mode words.** If both mode-word locations contain forty, separate otherwise identical invocations with mode one and mode seven each assign forty to state `0x78`. Their distinct locations impose no inequality requirement.

7. **Invalid mode.** Valid range and selection followed by mode two reject after common initialization. Neither identified dependency is invoked, and the counter is not incremented.

8. **Zero optional result.** Mode zero and a zero result from F-6d8c85350b66 assign the baseline word to the optional payload and derived fields. Completion still captures the counter pointer, writes state `0x8C`, updates the counter, reads the completion argument, invokes F-266a29fc8dc5, and returns zero.

9. **Adjusted payload.** Mode zero, baseline word four, a nonzero optional result, discriminator seven, and adjustable payloads ten, six, and three produce twenty, twelve, and six. State `0x7C` becomes twenty-four; state `0x80` becomes eight.

10. **Unsigned sum boundary.** Mode zero, baseline word `0xFFFFFFFF`, nonzero optional result, discriminator other than seven, first adjustable payload two, and second adjustable payload zero produce state `0x7C` equal to one and state `0x80` equal to zero.

11. **Captured counter pointer.** If the state `0x8C` write aliases the external pointer slot, subsequent counter accesses still use the pointer captured before that write, assuming those accesses remain valid.

12. **Late completion-argument read.** If the counter update validly changes storage supplying the completion argument, the later argument read observes the resulting contents. No earlier argument snapshot is established.

## Uncertainties

The following remain **documented_only**:

- Calling convention and origins of implicit inputs and flags.
- Fidelity of the represented indirect invocation to the original control transfer.
- Valid dispatch indices, targets, and signed address-arithmetic overflow behavior.
- External storage contents, pointer-slot width, and completion-argument width.
- Signed increment overflow, signed-shift edge behavior, and out-of-range unsigned-to-signed conversion.
- Dependency preservation, ownership, retention, failures, and nonreturning behavior.
- Supported aliasing, alignment, concurrency, reentrancy, and memory-access conditions.
- Domain meanings, physical units, and resource lifecycle semantics.

## Evidence

Established evidence consists solely of the supplied static representation: access widths and offsets, represented arithmetic types, branch conditions, ordered reads and writes, dependency invocations, and explicit return values.

The representation establishes direct computed-address dispatch, distinct external mode-word reads, counter-pointer capture before the state `0x8C` write, and completion-argument loading after the counter update. It does not establish a dispatch pointer-table load, unequal mode-word values, or immutable external storage.

No runtime tests have been performed. Acceptance examples are conditional static deductions, not observed executions.
