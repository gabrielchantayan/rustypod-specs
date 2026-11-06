<!-- Trusted provenance: F-a3bca5a17b7d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a3bca5a17b7d

## Purpose

F-a3bca5a17b7d conditionally invokes a preliminary dependency, routes an event through indirect handlers, and returns the first nonzero handler result. If every invoked result-bearing handler returns zero, it returns zero.

Routing depends on an event discriminator, an event-associated value, and receiver state. Importantly, routing is **not based on a single immutable snapshot**. The function rereads event fields and resolves later handlers after earlier dependencies return. Changes made by those dependencies can therefore affect subsequent routing.

The evidence establishes dispatch behavior, not application meaning. Interpretations such as “notification,” “handled,” “selection,” or “default processing” are not established.

## Interface

### Inputs

The function accepts two address-bearing arguments:

- **Receiver:** An address identifying a receiver object containing a dispatch-table reference and other state.
- **Event:** An address identifying an event record.

The supplied representation uses 32-bit integer accesses and integer-valued addresses. This contract describes that apparent 32-bit layout; it does not establish portability to a different pointer width or object model.

All offsets below are byte offsets.

### Accessed layout

| Storage | Offset | Accessed representation | Use |
|---|---:|---|---|
| Receiver | 0 | 32-bit address-bearing word | Reference to the dispatch table |
| Receiver | 80 (`0x50`) | 32-bit word | Comparison value used by the final conditional stage |
| Receiver | 268 (`0x10c`) | 32-bit word | Value narrowed to a character-sized argument for the preliminary dependency |
| Event | 4 | 32-bit integer | Event discriminator |
| Event | 8 | 32-bit word | Nested-record address for discriminator 22; direct comparison value for discriminator 15 |
| Nested record | 4 | 32-bit integer | Subtype or selector for discriminator 22 |

The meaning of event offset 8 depends on the discriminator observed at the particular stage. It is not always a nested-record address.

Indirect handlers are obtained from the receiver’s dispatch table:

| Dispatch-table byte offset | Contract-local opaque dependency ID |
|---:|---|
| 292 (`0x124`) | F-U01 |
| 268 (`0x10c`) | F-U02 |
| 272 (`0x110`) | F-U03 |
| 224 (`0xe0`) | F-U04 |

F-U01 through F-U04 are opaque identifiers for indirect call sites, not recovered identities of their targets. Different slots could identify the same function, and a slot’s target could change between calls.

The receiver field at offset 268 and the dispatch-table entry at offset 268 belong to different storage regions.

### Output

The function returns a 32-bit integer:

- The exact nonzero result of the first result-bearing handler that returns nonzero; or
- Zero if all result-bearing handlers actually invoked return zero.

There is no normalization to a Boolean. Negative results, if returned by a handler, also satisfy the nonzero return condition and are propagated unchanged.

### Preconditions

Every address actually accessed must be readable and valid for the required access. Every indirect target actually called must be callable with the supplied arguments and a compatible return convention.

Required validity is path-dependent:

- The event discriminator is read at entry.
- The nested record is dereferenced only when the discriminator observed at that stage is 22.
- Receiver offset 268 is read only when the preliminary condition matches.
- Receiver offset 80 is read only when the final stage observes discriminator 15.
- Dispatch-table storage is required for each indirect call reached.

The function performs no explicit null checks, bounds checks, object-type checks, or target validation. Exact alignment requirements and invalid-access behavior are not established by the supplied evidence.

## Behavior

### 1. Preliminary conditional dependency

The function first reads the event discriminator at offset 4.

If it is 22 (`0x16`), the function reads the nested-record address from event offset 8 and then reads the nested selector at offset 4 of that record.

A preliminary call occurs when that selector, after subtraction of 9472 (`0x2500`), compares equal to 2. For ordinary representable arithmetic, the matching selector is therefore 9474 (`0x2502`).

When this condition matches:

1. The receiver word at offset 268 is read.
2. That word is converted to a character-sized value.
3. F-79b9eb28e036 is called with the receiver and the converted value.

No result from F-79b9eb28e036 controls this function’s return or routing directly. Its mutations may nevertheless affect later reads.

The supplied representation explicitly narrows the receiver word to a character type. On the apparent 8-bit-character target, this selects the low eight bits before any argument promotion or extension required by the call convention. Whether values with bit 7 set are interpreted as negative or nonnegative is not established. The contract does not choose signed-character or unsigned-character behavior.

### 2. Optional early handler

After the preliminary stage, the function reads the event discriminator again. It does not reuse the initial discriminator.

If this newly observed discriminator is 22, it rereads event offset 8 and the nested selector.

If the selector is zero:

1. The current receiver dispatch-table reference is read.
2. The target at dispatch-table offset 292 is obtained.
3. F-U01 is called with the receiver and event.
4. If its result is nonzero, that result is returned immediately.

If the discriminator is not 22, the selector is not zero, or F-U01 returns zero, processing continues.

Consequently, F-79b9eb28e036 can affect whether F-U01 is called by changing the event discriminator, nested-record reference, or nested selector before returning.

### 3. Main routing stage

Unless F-U01 has returned a nonzero result, the function again reads the event discriminator.

When that discriminator is 22, it reads the current nested-record address and nested selector. It then chooses exactly one main handler:

- Selector 42 (`0x2a`): F-U02, through dispatch-table offset 268.
- Every other case: F-U03, through dispatch-table offset 272.

“Every other case” includes both:

- Discriminator 22 with any selector other than 42.
- Any discriminator other than 22.

For a discriminator other than 22, this stage does not dereference the nested record.

The chosen target is obtained using the receiver’s dispatch-table reference at this stage and is called with the receiver and event.

If the chosen handler returns nonzero, the function immediately returns that exact result. If it returns zero, processing continues to the final stage.

Exactly one of F-U02 and F-U03 is called whenever the main routing stage is reached. The function does not try the other main handler after a zero result.

F-U01 may have changed the event or dispatch table before this stage. Such changes are not excluded by the contract.

### 4. Final conditional handler

This stage is reached only after the chosen main handler returns zero.

The event discriminator is read again. If it is 15 (`0x0f`):

1. The word at event offset 8 is read.
2. The receiver word at offset 80 is read.
3. The two words are compared for equality.

If they are equal, the function resolves F-U04 from the current receiver dispatch table at offset 224 and calls it with the receiver and event.

- A nonzero F-U04 result is returned unchanged.
- A zero F-U04 result leads to the function’s final zero return.

If the discriminator is not 15, or the two comparison words differ, F-U04 is not called and the function returns zero.

In this stage, event offset 8 is a direct comparison value. The function does not dereference it as a nested-record address.

### 5. Ordering and repeated observation

The dependency-call order is:

1. F-79b9eb28e036, if its condition matches.
2. F-U01, if its independently reread condition matches.
3. Exactly one of F-U02 and F-U03, unless F-U01 returned nonzero.
4. F-U04, if the main handler returned zero and the final comparison matches.

Nonzero results from F-U01, the main handler, or F-U04 end processing immediately.

No general field-preservation assumption applies across calls. In particular:

- The event discriminator is reread at each stage.
- A nested-record reference and selector are reread at each stage that observes discriminator 22.
- The receiver dispatch-table reference is obtained for each indirect call reached.
- The final comparison observes receiver and event state after the main handler returns.

These are ordered accesses shown by the supplied representation. They are not a guarantee of atomicity, inter-thread synchronization, or a hardware-level volatile-access protocol.

## Side effects

### Direct effects

The visible body performs reads, calls, comparisons, and returns. It contains no explicit writes to receiver storage, event storage, nested records, dispatch tables, or global storage.

It contains no explicit allocation, deallocation, reference-count update, or ownership-transfer operation.

### Dependency-mediated effects

Every called dependency may have effects not shown here. The contract does not assume that dependencies:

- Preserve receiver or event fields.
- Preserve the receiver’s dispatch-table reference or its entries.
- Preserve global state.
- Retain, transfer, release, or avoid retaining any supplied address.
- Return normally.

The function performs no visible cleanup after a dependency returns. A dependency that releases or invalidates storage needed by a later stage would make subsequent accesses unsafe; the function does not protect against that condition.

### Errors and termination

The function defines no separate error-reporting mechanism. A nonzero dependency result is propagated without classifying it as success, failure, or “handled.”

There is no visible recovery around invalid memory accesses or invalid indirect calls. Behavior for those cases, and for exceptions or other nonlocal exits from dependencies, is documented_only.

The visible body has no loop or retry. Termination still depends on invoked dependencies returning.

## Dependencies

| Opaque ID | Invocation condition | Arguments | Result use |
|---|---|---|---|
| F-79b9eb28e036 | Initial discriminator is 22 and nested selector satisfies the preliminary arithmetic comparison | Receiver; character-sized conversion of receiver word at offset 268 | No result used by this function |
| F-U01 | Second discriminator observation is 22 and nested selector is zero | Receiver; event | Nonzero returned immediately |
| F-U02 | Main-stage discriminator is 22 and nested selector is 42 | Receiver; event | Nonzero returned immediately |
| F-U03 | Main-stage condition for F-U02 is false | Receiver; event | Nonzero returned immediately |
| F-U04 | After a zero main-handler result, discriminator is 15 and event offset 8 equals receiver offset 80 | Receiver; event | Nonzero returned immediately |

Only F-79b9eb28e036 has a supplied resolved dependency identity. The identities, implementations, ownership behavior, and global effects of the indirect targets are unknown.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence. They are not executed tests. Except where a mutation is specified, each scenario assumes valid storage, compatible call targets, normally returning dependencies, and stable relevant fields across calls.

### Selector zero with early termination

Given discriminator 22 and nested selector zero:

- F-79b9eb28e036 is not called.
- F-U01 is called.
- If F-U01 returns 7, the function returns 7.
- Neither main handler nor F-U04 is called.

This demonstrates exact result propagation and early termination.

### Selector zero with continuation

Given discriminator 22 and nested selector zero:

- F-U01 returns zero.
- The main-stage reread still observes selector zero.
- F-U03 is called, not F-U02.
- If F-U03 returns −3, the function returns −3.

The negative result is not converted to a Boolean or replaced by zero.

### Selector 42

Given discriminator 22 and nested selector 42:

- The preliminary dependency and F-U01 are not called.
- F-U02 is called.
- If F-U02 returns zero and the discriminator remains 22, F-U04 is not called.
- The final result is zero.

### Preliminary selector and narrowing

Given discriminator 22, nested selector 9474, and receiver word `0x1234567f` at offset 268:

- F-79b9eb28e036 is called before any result-bearing handler.
- Under the apparent 8-bit-character interpretation, its second argument represents 127.
- If fields remain unchanged, F-U01 is skipped and F-U03 is called.

For a receiver word whose low byte is `0xff`, the character-sized bit pattern is established under that interpretation, but signed versus unsigned extension is documented_only.

### Mutation by the preliminary dependency

Given an initial discriminator of 22 and selector 9474:

- F-79b9eb28e036 changes the nested selector to zero and returns.
- The second stage rereads the selector and calls F-U01.
- If F-U01 returns 11, the function returns 11 without a main-handler call.

An implementation that cached the initial selector for later routing would not satisfy this behavior.

### Mutation by the early handler

Given discriminator 22 and selector zero:

- F-U01 changes the selector to 42 and returns zero.
- The main stage rereads the selector and calls F-U02.
- It does not select F-U03 based on the earlier zero selector.

### Final equality check

Given discriminator 15 and equal words at event offset 8 and receiver offset 80:

- F-U03 is called first.
- If F-U03 returns zero and the comparison still matches, F-U04 is called.
- If F-U04 returns 5, the function returns 5.

For this path, the value at event offset 8 need not identify a nested record; this function uses it only as a comparison word.

### Main-handler mutation enabling the final stage

Given an initial discriminator other than 22:

- F-U03 is selected.
- F-U03 changes the discriminator to 15, makes the final comparison words equal, and returns zero.
- The final stage observes those changes and calls F-U04.

Thus, initial discriminator 15 is not required for F-U04 to be reached.

### Nonmatching final comparison

Given discriminator 15, a zero result from F-U03, and unequal final comparison words:

- F-U04 is not called.
- The function returns zero.

## Uncertainties

The following remain documented_only:

- Application-level meanings of the discriminator and selector constants.
- Full receiver, event, and nested-record layouts.
- Exact ABI, pointer encoding, endianness, and alignment requirements.
- Character signedness and the precise argument extension convention.
- Overflow semantics of the preliminary subtraction outside its ordinary representable domain.
- Identities and behavior of indirect call targets.
- Dependency effects on globals, ownership, lifetime, and dispatch state.
- Behavior under concurrent modification or nonlocal exits.
- Any semantic interpretation of zero versus nonzero beyond the observed control flow.

The apparent 32-bit widths and byte offsets are supported by the supplied representation. A language-level signed-overflow policy or a portable replacement ABI is not established by that evidence.

## Evidence

Established static evidence consists of the supplied body for F-a3bca5a17b7d and the supplied mapping identifying F-79b9eb28e036.

That evidence supports the accessed offsets, repeated reads, conditional call order, indirect dispatch offsets, character narrowing, absence of explicit stores or memory-management operations, and exact nonzero-result propagation described above.

Calling the inputs a receiver and an event, the nested value a selector, and the first receiver word a dispatch-table reference is descriptive inference from their use, not evidence of application semantics.

No runtime tests have been performed. The acceptance examples are static consequences subject to their stated conditions.
