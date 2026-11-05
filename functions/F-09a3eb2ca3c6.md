<!-- Trusted provenance: F-09a3eb2ca3c6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-09a3eb2ca3c6

## Purpose

F-09a3eb2ca3c6 processes a compact request that selects one of three values and may enable an associated state while preserving or obtaining a 16-bit value. Successful processing updates a shared state record and invokes an operation on a dependency object.

The established behavior is narrower than an application-level interpretation:

- Only request forms designated by the signed 16-bit values 1 and 2 are processed.
- The first payload byte must be less than 3.
- A second payload byte, when applicable, controls whether an existing or newly obtained 16-bit value is retained.
- The function returns a 32-bit result of 0 for an accepted request and 4 for a rejected request, provided execution returns normally.

The meaning of the selector, the optional byte, and the retained value is not established. Describing them as a selector, an activation byte, and a retained value is terminology for this contract, not evidence of a particular device, protocol, or user-facing feature.

## Interface

### Inputs

The function has two explicit arguments.

| Argument | Representation | Established use |
|---|---|---|
| First argument | Opaque 32-bit value | Not read or forwarded by the supplied function body. |
| Second argument | Address of a request record in the supplied 32-bit address model | Used to read a payload address and a signed 16-bit request-form field. |

The request record has these relevant fields:

| Byte offset | Width | Interpretation |
|---|---:|---|
| +8 | 32-bit address field | Address of the payload bytes. Read only for request forms 1 and 2. |
| +12 | 16 bits, signed | Request form. Read before any other request field. |

No behavior is established for other record fields. The implementation does not establish a total record size, packing convention, or byte order.

For accepted request forms, the payload is interpreted as follows:

| Byte offset | Width | Interpretation |
|---|---:|---|
| +0 | 8 bits, unsigned | Selector. Values 0, 1, and 2 are accepted. |
| +1 | 8 bits, unsigned | Activation byte. Read only for form 2 when the selector is accepted. |

The request-form field is not proven to be a buffer length. Although its values determine whether one or two payload bytes can be read, callers must not infer general length validation from this behavior.

### Return value

The result is represented as a 32-bit value:

- **0:** The selector was accepted, the shared-state writes occurred, and the final dependency operation returned normally.
- **4:** The request form was neither 1 nor 2, or the selector was at least 3.

These are the only explicit normal return values. The function does not translate dependency failures, invalid memory accesses, or abnormal control transfers into an explicit result.

### Referenced state and object layouts

A shared pointer identifies a state record with these relevant fields:

| Byte offset | Width | Interpretation in this contract |
|---|---:|---|
| +8 | 8 bits | Existing activation state; replaced by the request’s activation byte, or zero. |
| +12 | 16 bits | Retained value; replaced on every accepted-selector path. |

A dependency object is obtained from F-b7af80e829ec. Its leading field identifies a dispatch table. Two dispatch entries are used:

| Dispatch-table byte offset | Arguments visible here | Result use |
|---|---|---|
| +60 | Dependency object and selector | Any return value is ignored. |
| +64 | Dependency object | Result contributes only a 16-bit value. |

These offsets describe the supplied representation. They do not establish a portable object ABI or the complete layouts of either object or table.

## Behavior

### Initial request classification

The first observable request access is a signed 16-bit read at request offset +12.

If that value is neither 1 nor 2, the function returns 4. It does not explicitly read the payload address, read payload bytes, invoke F-b7af80e829ec, or access the shared state on this path.

For forms 1 and 2, the function:

1. Reads the payload address from request offset +8.
2. Reads the selector byte at payload offset +0.
3. Calls F-b7af80e829ec to obtain a dependency object.
4. Captures the shared state-record pointer.
5. Classifies the selector.

The selector is therefore read before the dependency call, but rejected only after that call returns and the shared pointer has been captured. A selector of 3 through 255 returns 4 without explicit writes to the shared record and without either dispatch operation.

A rejected selector does **not** imply absence of side effects: F-b7af80e829ec has already run.

The request form, payload address, and selector are retained from their earlier reads. Changes to those fields during F-b7af80e829ec do not cause them to be reread by this function.

### Accepted selector with form 1

For form 1 and selector 0, 1, or 2:

- The activation byte is treated as zero.
- Payload offset +1 is not read.
- The retained value is set to zero.
- No operation at dispatch offset +64 is invoked.

The function then performs the shared-state writes and final dispatch described below.

### Accepted selector with form 2

For form 2 and selector 0, 1, or 2, the function reads the activation byte at payload offset +1. This read occurs after F-b7af80e829ec returns and after the shared state pointer has been captured.

If the activation byte is zero:

- The retained value is set to zero.
- The existing shared activation state is not read.
- The existing shared retained value is not read.
- The operation at dispatch offset +64 is not invoked.

If the activation byte is nonzero, the existing shared activation state at offset +8 is examined.

- **Existing activation state is zero:** Invoke the operation at dispatch offset +64 with the dependency object. Use its result as a 16-bit retained value.
- **Existing activation state is nonzero:** Read the existing 16-bit value at shared-state offset +12 and use that value. Do not invoke the operation at dispatch offset +64.

All nonzero activation bytes take the same branch, but the byte is not normalized to 1. For example, an activation byte of 255 is ultimately stored as 255.

The existing activation state is likewise tested only for zero versus nonzero.

### Shared-state update and final dispatch

Every accepted-selector path reaches the same ordered update:

1. Write the selected 16-bit retained value at offset +12 of the previously captured shared state record.
2. Write the activation byte at offset +8 of that record.
3. Invoke the operation at dispatch offset +60 with the dependency object and the original selector.
4. Return 0 if that operation returns normally.

The retained-value write precedes the activation-byte write. Both precede the final dispatch.

The function does not return the result of the final dispatch and does not explicitly test it. Consequently, return value 0 does not establish that the dependency’s final operation succeeded according to any separate application-level convention.

### State-pointer and dispatch timing

The shared state-record pointer used for the writes is captured after F-b7af80e829ec returns and before any optional dispatch at offset +64.

If that optional dispatch changes the shared global pointer, the subsequent explicit writes still target the previously captured record. The function does not reload the write target after the callback.

The source separately expresses the existing-state reads through the shared pointer. In ordinary sequential execution without asynchronous mutation, these refer to the record captured immediately beforehand: there is no intervening explicit call before those reads. The supplied evidence does not establish volatile semantics, synchronization, or exact physical reload behavior.

Each dispatch obtains its operation through the dependency object’s table at the time of that dispatch. The final operation is not shown as a target cached before the optional operation. An optional operation that changes the object’s table can therefore affect the subsequent dispatch, subject to the object remaining valid.

## Side effects

### Explicit memory writes

The only explicit data writes shown are:

- A 16-bit write at shared-state offset +12.
- An 8-bit write at shared-state offset +8.

They occur only for selectors 0, 1, or 2 under request forms 1 or 2. No payload or request-record writes are explicit.

These are two ordered writes, not an established atomic transaction. A callback, concurrent observer, or failure may expose consequences not representable as one indivisible state transition.

### Dependency effects

F-b7af80e829ec runs for every request of form 1 or 2, including requests later rejected for an invalid selector.

The optional dispatch at offset +64 runs only when all of the following hold:

- The request form is 2.
- The selector is less than 3.
- The activation byte is nonzero.
- The existing shared activation state is zero.

The final dispatch at offset +60 runs for every accepted selector, after the two writes.

No dependency is proven to preserve shared globals, request contents, payload contents, the object’s table, or other memory.

### Ownership and lifetime

The function does not explicitly allocate, free, retain, or release memory. That observation does not establish the ownership behavior of its dependencies.

Callers must provide valid storage for every access reached on the chosen path. In particular:

- The request-form field must be readable even for an unsupported form.
- Forms 1 and 2 require a readable payload address field and first payload byte.
- Form 2 with an accepted selector requires a readable second payload byte.
- Accepted selectors require a writable captured shared record.
- Invoked dependency operations require valid object and dispatch-table storage.

If an invoked operation invalidates the captured state record or dependency object before a later use, no protective check is shown.

Aliasing between request, payload, shared state, or dependency storage is not prohibited by an explicit check. Such aliasing can make the stated access order observable; non-aliasing guarantees are unknown.

## Dependencies

### F-b7af80e829ec

Called with no explicit arguments. Its result is used as the dependency-object address.

Established timing:

- After the request form, payload address, and selector have been read.
- Before selector validation.
- Before capture of the shared state-record pointer.
- Before the possible second-payload-byte read.

Its ownership contract, failure representation, and side effects are unknown. No null-result check is shown. For a rejected selector, the returned object is not explicitly dereferenced.

### Indirect operation at offset +64

No opaque function ID is supplied for the dynamic target.

It receives the dependency object and supplies the retained value when activation is requested from an existing zero state. The caller retains only 16 bits of its result; a wider result, if present, has its higher bits discarded.

Its full return type, failure convention, and effects are unknown.

### Indirect operation at offset +60

No opaque function ID is supplied for the dynamic target.

It receives the dependency object and selector after the shared-state writes. Its return value, if any, is ignored. Argument widening and the full calling convention are not established beyond the displayed arguments and the selector’s range.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Normal completion, valid reached storage, and dependency behavior consistent with each scenario are assumed.

| Scenario | Required behavior |
|---|---|
| Request form is 0, 3, or −1 | Return 4 after reading the form; do not explicitly inspect the payload or call a dependency. |
| Form 1, selector 3 | Call F-b7af80e829ec, capture the shared pointer, then return 4; no explicit shared-record write or indirect dispatch. |
| Form 2, selector 255 | Reject as above; do not read payload offset +1. |
| Form 1, selector 2, previous state arbitrary | Write retained value 0, then activation byte 0; invoke the final operation with selector 2; return 0. |
| Form 2, selector 0, activation byte 0 | Do not inspect the old activation or retained fields; write zeros in the specified order, then perform the final dispatch. |
| Form 2, selector 1, activation byte 7, existing activation 0 | Invoke the optional operation; store its 16-bit result, then byte 7; perform the final dispatch with selector 1. |
| Form 2, selector 2, activation byte 255, existing activation 9, retained value 4660 | Do not invoke the optional operation; read and rewrite retained value 4660, store byte 255, then dispatch with selector 2. |

Additional ordering criteria:

- If F-b7af80e829ec changes the first payload byte, the selector already read remains authoritative.
- If it changes the second payload byte before a form-2 accepted-selector path reads it, that later read observes the byte available at that time.
- If it changes the shared pointer before returning, the subsequent capture uses the pointer then available.
- If the optional operation changes the shared pointer, the two writes still target the record captured before that operation.
- If the optional operation yields a wider integer value of 65,537, the retained 16-bit value is 1.
- If the final operation returns an application-level error value normally, this function still returns 0 because that result is not inspected.

## Uncertainties

The following are **documented_only**:

- Application meaning of either request form, selector values, activation state, and retained value.
- Complete structure sizes, byte order, required alignment, and exact platform ABI.
- Whether addresses or storage can be modified asynchronously.
- Dependency ownership, allocation, release, synchronization, and failure behavior.
- Identities and complete signatures of the indirect operations.
- Whether dependencies preserve or replace global state and dispatch tables.
- Behavior after an invalid address, unavailable operation, or nonreturning dependency.
- Whether external conventions prohibit aliasing or concurrent invocation.

The field widths and offsets support the supplied 32-bit representation; they do not justify assuming identical layout on a different target.

The apparent intent to obtain a value only on a transition from inactive to active is an inference. The established behavior is the zero/nonzero branch and ordered memory operations, not a validated lifecycle protocol.

## Evidence

This contract is based solely on the supplied static function body and the supplied mapping of the direct dependency to F-b7af80e829ec.

Established evidence includes the 32-bit result constants, signed 16-bit form comparison, unsigned byte selector comparison, conditional payload reads, direct dependency-call placement, shared-pointer capture, conditional 16-bit value selection, ordered writes, and two indirect dispatch offsets.

No runtime tests have been performed. Acceptance examples express consequences of the supplied static evidence under their stated preconditions; they are not observed execution results.
