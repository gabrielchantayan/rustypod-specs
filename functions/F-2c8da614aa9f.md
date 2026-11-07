<!-- Trusted provenance: F-2c8da614aa9f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2c8da614aa9f

## Purpose

F-2c8da614aa9f selects one of three lookup keys from a state record, obtains a byte-addressed lookup result, and dispatches an operation on a supplied receiver.

A non-null result whose first byte is nonzero follows the content-processing path. A null result or a result beginning with a zero byte follows an alternate receiver operation. The content-processing operation returns a 32-bit status: zero ends processing; any nonzero value is passed, together with the original lookup-result pointer, to an indirectly selected handler.

These are structural descriptions, not established application semantics. The supplied evidence does not identify the lookup keys, the result’s encoding, or the receiver operations’ domain meanings.

## Interface

### Inputs

The function accepts two arguments:

| Input | Representation | Required use |
|---|---|---|
| State address | A 32-bit unsigned address value | Addresses a record containing the fields described below |
| Receiver | A pointer to a dispatch-bearing object | Supplies the indirect operations used by either result path |

The visible function has no explicit return value. Results are communicated through dependency calls, receiver operations, and the final handler.

The supplied representation uses 8-bit bytes and 32-bit integers and opaque values. Its receiver layout is consistent with 32-bit pointers: the receiver’s first word identifies a dispatch table, and callable entries are obtained at byte offsets 0x08 and 0x0C. This contract describes that represented layout; it does not establish portability to a different pointer-width ABI.

### State-record fields

Offsets are measured in bytes from the state address.

| Offset | Access width | Decision role |
|---|---:|---|
| 0xA0 | 32 bits | Selects the primary branch by exact equality to 1 |
| 0xB4 | 32 bits | Read only when the field at 0xA0 equals 1 |
| 0x68 | 32 bits | Read only when the field at 0xA0 does not equal 1 |
| 0x9C | 32 bits | Read only when neither earlier branch selects the decrement key |
| 0x94 | 8 bits, unsigned | Read only when the field at 0x9C equals zero |

The 32-bit fields are compared with exactly zero or exactly one as specified. They are not generally interpreted as Boolean values: for example, a value of 2 does not satisfy a comparison with 1.

### Global selection data

Two global 32-bit integer values participate in key selection:

- **Primary selector:** read at entry to initialize the default key and separately to derive a key one less than its value.
- **Alternate selector:** read only on the alternate-key branch, after F-9d0ecc5f6adf has returned.

The initial primary-selector reads must not be collapsed into one stable snapshot without an additional stability assumption. The supplied representation contains two evaluations of that global before inspecting the state fields.

### Receiver requirements

Whenever execution reaches a receiver operation, the receiver must identify an object whose first word and selected dispatch-table entry are readable and callable with the indicated arguments.

There is no explicit receiver-null check. A null receiver is not made safe by either a null or a non-null lookup result: both result paths eventually require receiver dispatch if preceding calls return normally.

## Behavior

### 1. Capture the entry-time selection values

Before reading the state-record fields, the function obtains:

1. A primary-selector value for the default key.
2. A separate primary-selector value from which it computes the decrement key.

If the primary selector is stable during these evaluations, the decrement key is the default key minus one. If it changes between evaluations, that relationship is not guaranteed.

The decrement computation precedes branch selection, including on executions that ultimately use another key.

### 2. Select a key from the state record

The decision order is significant.

**When the 32-bit field at 0xA0 equals 1:**

- Read the 32-bit field at 0xB4.
- If it equals 1, select the decrement key.
- Otherwise, select the default key.
- Do not read the fields at 0x68, 0x9C, or 0x94 for this decision.

**When the field at 0xA0 does not equal 1:**

- Read the 32-bit field at 0x68.
- If it equals 1, select the decrement key without reading 0x9C or 0x94.
- Otherwise, read the 32-bit field at 0x9C.
  - If that field is nonzero, select the default key without reading 0x94.
  - If that field is zero, read the unsigned byte at 0x94.
    - A nonzero byte selects the default key.
    - A zero byte selects the alternate-key branch.

Every selected branch invokes F-9d0ecc5f6adf once, with no visible arguments. Its opaque 32-bit return value becomes the first argument to the lookup dependency.

For the default and decrement branches, the selected key comes from values captured before the state inspection and before F-9d0ecc5f6adf.

For the alternate branch, the alternate selector is read **after** F-9d0ecc5f6adf returns. Consequently, a change to that global made by F-9d0ecc5f6adf can affect the lookup key. The same dependency cannot retroactively change the already captured default or decrement keys.

### 3. Perform the lookup

The function calls F-c0b020cd3882 with:

1. The opaque value returned by F-9d0ecc5f6adf.
2. The selected 32-bit key.

Its return value is interpreted as a byte pointer.

No retry, key substitution, range validation, or lookup-error translation is visible.

### 4. Distinguish absent or zero-leading results from nonzero-leading results

If the lookup result is null, the function does not dereference it and proceeds to the alternate receiver operation.

If the lookup result is non-null, the function reads its first byte once for the branch decision:

- Zero selects the alternate receiver operation.
- Any value from 1 through 255 selects content processing.

The byte is interpreted as unsigned. Values with the high bit set are therefore nonzero content indicators, not negative status values.

This decision does not establish that the result is a valid string, that a terminator exists, or that any particular length is available. Only the first-byte test is visible here.

### 5. Process a nonzero-leading result

For a non-null result whose first byte is nonzero:

1. Call F-8810f53aaa0b with the original lookup-result pointer.
2. Obtain the receiver operation from dispatch-table byte offset 0x08.
3. Invoke that operation with:
   - the receiver;
   - the opaque 32-bit value returned by F-8810f53aaa0b;
   - a literal zero as the third argument.
4. Inspect the operation’s 32-bit integer return value.

A return value of exactly zero causes F-2c8da614aa9f to return immediately.

A nonzero return value causes invocation of a handler selected through a global function pointer. The handler receives:

1. The nonzero receiver-operation result.
2. The original lookup-result pointer.

The function then returns if the handler returns normally. It does not invoke the alternate receiver operation in response to a nonzero status.

The first byte is not visibly checked again after F-8810f53aaa0b or the receiver operation. The pointer passed to the final handler remains the original lookup pointer, regardless of possible changes to its contents.

### 6. Process a null or zero-leading result

For either a null lookup result or a zero first byte:

1. Obtain the receiver operation from dispatch-table byte offset 0x0C.
2. Invoke it with the receiver as its sole visible argument.
3. Return if that operation returns normally.

F-8810f53aaa0b and the receiver operation at offset 0x08 are not invoked on this path.

## Side effects

### Directly visible effects

The function performs conditional state-record reads, global reads, lookup-result inspection, and indirect calls. It contains no explicit write to:

- the state record;
- either selector global;
- the receiver;
- the lookup-result buffer.

That absence does not imply that the complete operation is read-only. Every called dependency may have effects not shown in the supplied evidence.

### Call ordering and mutable state

The following ordering constraints are established by the supplied representation:

- Both entry-time primary-selector evaluations precede state-record inspection.
- State-based branch selection precedes F-9d0ecc5f6adf.
- The alternate-selector read, when required, follows F-9d0ecc5f6adf.
- F-c0b020cd3882 follows the completed key selection.
- A non-null lookup result’s first byte is inspected before either receiver path.
- On the content-processing path, F-8810f53aaa0b precedes receiver dispatch.
- The global handler is invoked only after a nonzero result from the receiver operation at offset 0x08.

Receiver dispatch data is consulted after the preceding dependencies have run. It must not be treated as necessarily equal to its entry-time state. Likewise, the global handler target must not be assumed fixed across earlier calls.

### Ownership and lifetime

No explicit allocation, release, reference-count adjustment, or ownership transfer is visible in this function.

The lookup result must be readable for its first-byte test when non-null. On the content-processing path, its original pointer is also passed to F-8810f53aaa0b and, conditionally, to the final handler. The lifetime and validity required by those callees depend on their contracts.

The supplied evidence does not establish whether:

- lookup storage is borrowed, static, owned, or transient;
- F-8810f53aaa0b copies, consumes, mutates, or releases that storage;
- the receiver retains its opaque argument;
- the final handler releases or retains the lookup result.

An implementation must not invent cleanup or retention rules from the absence of explicit cleanup here.

### Failures and exceptional control flow

There is no visible validation for an invalid state address, unreadable required field, invalid receiver, invalid dispatch table, or invalid handler target.

There is also no visible local exception handling, retry, recovery, or cleanup around dependency calls. Normal-path descriptions are conditional on those calls returning normally.

The meaning of a nonzero receiver result is unknown. Structurally, it triggers the final handler; calling it an application error would be an inference.

## Dependencies

| Dependency | Visible contract at this call site |
|---|---|
| F-9d0ecc5f6adf | Called once on each normal path before lookup; takes no visible arguments and returns an opaque 32-bit value |
| F-c0b020cd3882 | Receives that opaque value and the selected 32-bit key; returns a value interpreted as a byte pointer |
| F-8810f53aaa0b | Called only for a non-null, nonzero-leading lookup result; receives the original pointer and returns an opaque 32-bit value |
| Receiver operation at dispatch offset 0x08 | Receives the receiver, the transformed opaque value, and zero; returns a 32-bit integer status |
| Receiver operation at dispatch offset 0x0C | Receives the receiver on null or zero-leading lookup results; its return value is not used |
| Globally selected handler | Receives a nonzero status and the original lookup pointer; its return value is not used |

F-151ab4bfe553 is associated in the supplied mapping with a shared continuation label. The visible body does not establish a distinct call to it. It must not be added as another runtime dependency invocation merely because execution branches to that label.

## Acceptance examples

These examples are conditional expectations derived from static evidence, not executed tests. They assume readable required memory, valid callable targets, and normal returns from dependencies.

1. **Primary branch selects decrement.**  
   If both entry-time primary-selector reads produce 20, the field at 0xA0 is 1, and the field at 0xB4 is 1, the lookup key is 19. Fields at 0x68, 0x9C, and 0x94 are not read for selection.

2. **Exact equality matters.**  
   With a stable primary selector of 20, fields at 0xA0 and 0xB4 equal to 1 and 2 respectively select key 20, not 19.

3. **Secondary branch bypasses later fields.**  
   If 0xA0 does not equal 1 and 0x68 equals 1, the decrement key is selected regardless of the values at 0x9C and 0x94. Those later fields are not read.

4. **Alternate selector is observed after the accessor.**  
   Suppose 0xA0 and 0x68 do not equal 1, 0x9C equals zero, and the byte at 0x94 equals zero. If F-9d0ecc5f6adf changes the alternate selector from 30 to 40 before returning, the subsequent alternate-selector read supplies 40 to the lookup.

5. **Nonzero word suppresses the byte read.**  
   Under the same initial branch conditions, a nonzero value at 0x9C selects the cached default key. The byte at 0x94 is not consulted.

6. **Null lookup result.**  
   A null result causes only the receiver operation at offset 0x0C after lookup. There is no lookup-buffer dereference and no call to F-8810f53aaa0b.

7. **Zero-leading lookup result.**  
   A readable result beginning with byte zero takes the same receiver path as a null result, even if later bytes are nonzero.

8. **High-bit content and zero status.**  
   A result beginning with 0x80 takes the content-processing path. If the receiver operation at offset 0x08 returns zero, the function returns without calling the global handler.

9. **Nonzero status forwarding.**  
   If that receiver operation returns −1, the handler receives that nonzero 32-bit result and the original lookup pointer. The alternate receiver operation is not called.

10. **Separate primary-selector observations.**  
    If the first primary-selector evaluation yields 20 and the second yields 25, default selection uses 20 and decrement selection uses 24, provided the subtraction is representable under the relevant arithmetic interpretation.

## Uncertainties

The following are **documented_only**:

- The selectors’ concrete values and application meanings.
- The state fields’ domain names and valid value ranges.
- Lookup-result encoding, length, mutability, and ownership.
- All dependencies’ internal effects, including changes to globals or receiver state.
- The meaning of the zero argument passed to the receiver operation.
- Whether nonzero receiver statuses denote errors, handles, or another category.
- The final handler’s behavior, including whether it returns.
- Concurrency, synchronization, atomicity, and memory-order guarantees.
- Exact calling conventions beyond the visible argument and result shapes.
- Arithmetic at the minimum signed 32-bit selector value.

The decrement is represented as signed 32-bit subtraction by one. Ordinary representable cases are clear. For an input of −2,147,483,648, the supplied material alone does not justify promising either a wrapped result of 2,147,483,647 or a particular language-level overflow behavior.

Application interpretations such as “lookup text,” “display content,” “clear the receiver,” or “handle an error” are inferred intent, not established behavior.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes the field offsets and access widths, exact comparisons, conditional read order, selector timing relative to calls, first-byte test, dependency argument shapes, receiver dispatch offsets, and zero-versus-nonzero status handling.

The supplied representation marks the terminal indirect transfers as imperfectly recovered. Their displayed targets and arguments support the call relationships described above, but do not independently establish whether the machine-level transfers are ordinary calls, tail transfers, or another equivalent dispatch form.

No runtime tests have been performed. Acceptance examples specify expected behavior under their stated conditions; they are not reports of observed execution.
