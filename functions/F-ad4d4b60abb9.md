<!-- Trusted provenance: F-ad4d4b60abb9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ad4d4b60abb9

## Purpose

F-ad4d4b60abb9 conditionally updates an object’s 32-bit recorded value. In its full-processing path, it also resolves an associated handle, applies a handle-derived value to the object, constructs temporary context, and conditionally submits an operation through a dependency.

A nonzero byte in the object selects a short path: record the supplied value and return without performing the other work.

These are established structural behaviors. Descriptions such as “recorded value,” “handle,” “context,” and “submission” identify roles in this contract; they do not establish application-level meanings or ownership rules.

## Interface

The function has five inputs and no specified return value.

| Input | Representation | Established use |
|---|---|---|
| Object base | Address represented by a 32-bit integer in the supplied representation | Locates object fields and an embedded lookup region. |
| Requested value | Opaque 32-bit value | Passed to lookup and eventually written to the object’s recorded-value field. |
| Operation argument | 32-bit integer | Compared with zero and forwarded unchanged to the submission dependency when submission occurs. |
| Override argument | 32-bit integer | Compared with zero to determine whether the object’s 16-bit auxiliary value is eligible for use. |
| Attribute | 8-bit value | Written to the object on the full-processing path only. |

The operation and override arguments are tested for zero versus nonzero. Their sign has no separate significance in the visible decisions. In particular, a negative operation argument follows the nonzero branch and is forwarded unchanged.

The object-relative layout used directly by this function is:

| Offset | Width or role | Direct access |
|---|---|---|
| `0xB0` | Embedded region; extent unknown | Its address is passed to lookup. |
| `0xE0` | 32 bits | Receives the requested value. |
| `0xE4` | 8 bits | Receives the attribute on the full-processing path. |
| `0xE6` | 16 bits | Read conditionally to obtain a submission argument. |
| `0xE8` | 8 bits | Read at entry to select the short or full-processing path. |

Two fields are also read through addresses returned by F-fdd8589eca2d:

| Offset | Width | Use |
|---|---|---|
| `0x2C` | 8 bits | First context-construction input. |
| `0x28` | 32 bits | Second context-construction input, obtained through a separate provider call. |

The byte order and numeric interpretation of the opaque 32-bit values are not established. The 16-bit field is passed as a two-byte value; whether a dependency interprets it as signed, unsigned, flags, or another representation is unknown.

### Preconditions

On the selected execution path, all directly accessed addresses must be valid for the stated widths. The short path requires only the initial flag read and recorded-value write from this function itself. The full path additionally requires the attribute write, valid inputs to the dependencies, and any conditional auxiliary-field read.

The supplied representation performs address calculations using a 32-bit object-base value and fixed offsets. This contract does not establish safe behavior for invalid bases, address wraparound, insufficient storage, or unsuitable alignment. There is no visible bounds or alignment validation.

No thread-safety, reentrancy, or exclusive-access guarantee is established.

## Behavior

### 1. Entry selection and short path

The first direct object access is a one-byte read at offset `0xE8`.

If that byte is nonzero:

1. The function writes the requested 32-bit value at offset `0xE0`.
2. It returns.

On this path, it does not directly write the attribute field, perform lookup, construct temporary context, query the mode, submit an operation, or invoke temporary cleanup. It makes no dependency calls.

Every nonzero byte value selects this path. The flag is not restricted to the value one.

If the entry byte is zero, processing continues through the full path. The function does not directly reread this flag later.

### 2. Attribute update and lookup

The full path first writes the supplied 8-bit attribute at offset `0xE4`.

It then calls F-1f46730b1ced with:

- the address of the embedded region at object offset `0xB0`; and
- the requested 32-bit value.

The result is used as a handle. A zero result selects the failure path; every nonzero result proceeds.

The attribute write precedes lookup and is therefore already observable if lookup fails. There is no visible rollback.

### 3. Lookup failure

If lookup returns zero, the function calls F-a817d8d58c10.

The supplied static representation marks that dependency as nonreturning. Under that designation, this invocation of F-ad4d4b60abb9 does not return normally and does not reach the later recorded-value write, context construction, submission, or cleanup.

The dependency’s actual failure mechanism is not established. It may terminate, raise a nonlocal failure, or behave in another nonreturning manner. No specific diagnostic, exception type, exit status, or recovery behavior is specified.

Behavior if this dependency unexpectedly returns is documented_only, not part of the established normal contract.

### 4. Handle-derived object update

For a nonzero handle, the function performs these calls in order:

1. F-d2f99870cc43 receives the handle and supplies a 32-bit value.
2. F-bf557f07179f receives the object base and that supplied value.
3. F-fa8898fa75ae receives the original handle.

The same handle is retained for possible later submission.

The role of F-fa8898fa75ae is not established. Its position must not be taken as proof that it destroys or releases the handle. Conversely, this contract does not assume that the handle remains valid because that dependency preserves it. The established fact is that the original handle value is used again if submission occurs.

Dependency effects on the object, global state, or other memory remain possible and are not bounded by the direct-access list above.

### 5. Temporary-context construction

The function reserves 92 bytes of automatic temporary storage and uses that storage as a context object.

Construction inputs are obtained in this order:

1. Call F-fdd8589eca2d.
2. Read one byte at offset `0x2C` from the returned address.
3. Call F-fdd8589eca2d again.
4. Read 32 bits at offset `0x28` from this second returned address.
5. Call F-2c5a9f9c1eb3 with the temporary-storage address, the 32-bit value from the second result, and the byte from the first result.

The first byte is retained across the second provider call. The two provider results need not be equal. Context construction can therefore combine values originating from different returned objects or different provider states.

There is no explicit initialization of the 92-byte region before the construction call. Which bytes the constructor initializes, and whether they include owned resources or internal pointers, is unknown.

There is no visible branch on the constructor’s result or explicit construction-success check.

### 6. Mode decision

After context construction, the function calls F-5c5958cfd5fe.

If the result is exactly eight, submission is skipped. Processing proceeds to the final recorded-value write and temporary cleanup.

Every other result, including zero and negative values under the represented integer interpretation, takes the submission path. No other mode values receive special treatment.

A mode of eight does not bypass the earlier attribute write, lookup, handle-derived update, post-update handle call, provider reads, or context construction.

### 7. Submission argument selection

When submission is not skipped, the function calls F-fdd8589eca2d once more. This third result supplies the first submission argument.

The provider call occurs before any direct read of the object’s 16-bit auxiliary field. Effects of that call can therefore affect a subsequent auxiliary-field read.

The 16-bit submission value is selected as follows:

| Override argument | Operation argument | Selected value |
|---|---|---|
| Nonzero | Any | Zero |
| Zero | Zero | Zero |
| Zero | Nonzero | Current 16-bit contents at object offset `0xE6` |

The auxiliary field is directly read only in the final row. It is not read merely to discard its value in either zero-selected case.

The function then calls F-818c773e8c8a with, in order:

1. The result of the third provider call.
2. The temporary-context address.
3. The original lookup handle.
4. The original 32-bit operation argument.
5. The selected 16-bit value.

The override argument is not itself forwarded to this dependency.

No explicit branch inspects the submission result. A returned failure indication, if that dependency has one, does not produce a visible recovery path in this function.

### 8. Final write and cleanup

Both the mode-eight path and the normally returning submission path converge on:

1. A 32-bit write of the requested value to object offset `0xE0`.
2. A call to F-920630e53283 with the temporary-context address.
3. Normal return.

The recorded-value write is after submission, when submission occurs, but before cleanup. Cleanup can therefore observe the newly written value if it can access the object through dependency-managed state.

The function does not write the recorded value again after cleanup. A dependency effect that subsequently changes that field is not reversed.

This sequence applies only when preceding dependencies return normally. No visible local mechanism ensures cleanup after an earlier nonlocal failure or nonreturning call.

## Side effects

Direct effects are limited to:

- Reading the entry flag.
- Writing the attribute on the full path.
- Reading the two provider-supplied fields during context construction.
- Conditionally reading the object’s 16-bit auxiliary field.
- Writing the requested 32-bit recorded value.
- Providing temporary storage to construction, optional submission, and cleanup dependencies.

These are not a complete inventory of all memory or external effects. Called dependencies may mutate the object, provider state, globals, resources, or memory reachable through their arguments. Their implementations are not supplied.

The object is modified in place. The function does not visibly allocate, free, or transfer ownership of the object.

The temporary storage’s automatic lifetime ends when the function exits. Whether construction acquires resources and cleanup releases them is unknown. The temporary address is passed to submission and cleanup; safe retention beyond its lifetime cannot be assumed.

The function has no normal error return. Its visible explicit failure handling is the nonreturning dependency call after a zero lookup result. No rollback is visible for any prior mutation.

## Dependencies

| Opaque ID | Established role in this function |
|---|---|
| F-1f46730b1ced | Looks up a handle using the embedded-region address and requested value. Zero selects explicit failure handling. |
| F-a817d8d58c10 | Called after zero lookup; marked nonreturning in the supplied representation. |
| F-d2f99870cc43 | Obtains a 32-bit value from the handle. |
| F-bf557f07179f | Receives the object and handle-derived value. |
| F-fa8898fa75ae | Receives the handle after the object-update call and before context construction. |
| F-fdd8589eca2d | Supplies addresses for two construction-field reads and, conditionally, a submission argument. |
| F-2c5a9f9c1eb3 | Constructs context in the 92-byte temporary region. |
| F-5c5958cfd5fe | Supplies the mode; exactly eight suppresses submission. |
| F-818c773e8c8a | Receives the provider result, temporary context, original handle, operation argument, and selected 16-bit value. |
| F-920630e53283 | Receives the temporary context after the final recorded-value write. |

No dependency is assumed to preserve globals, preserve other reachable state, own memory, or release memory unless independently documented.

## Acceptance examples

These examples are conditional behavioral checks derived from static evidence. They have not been executed. Examples involving continued execution assume each preceding dependency returns normally.

### Nonzero entry flag

Given any nonzero entry byte, including a value other than one:

- The requested value is written as a 32-bit quantity at offset `0xE0`.
- No dependency is called.
- This function does not write the supplied attribute to offset `0xE4`.

### Failed lookup

Given a zero entry flag and a zero lookup result:

- The attribute write occurs before lookup.
- F-a817d8d58c10 is invoked.
- Under its nonreturning designation, no context is constructed and the final recorded-value write is not reached.

No claim is made that the recorded-value field remains unchanged by lookup or the failure dependency.

### Mode eight

Given successful lookup and a mode result of eight:

- Handle-derived processing and context construction occur.
- F-fdd8589eca2d is called twice, not three times.
- The auxiliary field is not directly read.
- Submission is omitted.
- The requested value is written before temporary cleanup.

### Nonzero override

Given a mode other than eight and a nonzero override argument:

- The provider is called a third time.
- Submission receives zero as its 16-bit argument.
- The auxiliary field is not directly read, regardless of the operation argument.

### Zero operation argument

Given a mode other than eight, zero override, and zero operation argument:

- Submission still occurs.
- Its operation argument is zero and its 16-bit argument is zero.
- The auxiliary field is not directly read.

### Auxiliary-field selection

Given a mode other than eight, zero override, and nonzero operation argument:

- The third provider call occurs before the auxiliary-field read.
- Submission receives the 16-bit value present at the time of that read.
- A negative operation argument is forwarded unchanged and still selects this field-reading case.

### Different provider results

If the first two provider calls return distinct addresses, context construction uses the byte from the first address and the 32-bit field from the second. Equality of those addresses is not required by the visible control flow.

### Cleanup observation

If cleanup can inspect the object’s recorded-value field, it runs after this function has written the requested value there. If cleanup subsequently modifies that field, this function does not restore it before returning.

## Uncertainties

The following remain documented_only:

- Application-level meanings of the object fields, mode, attribute, operation argument, and override argument.
- Complete object and provider layouts, alignment requirements, and byte order.
- Whether the lookup result is a pointer, reference-counted resource, numeric identifier, or another handle.
- Ownership and lifetime consequences of all handle-related calls.
- Whether provider calls return stable objects or modify shared state.
- Temporary-context representation, initialization coverage, and resource ownership.
- Whether submission retains either the handle or temporary-context address.
- Failure behavior within construction, object update, submission, and cleanup.
- Exception, nonlocal-exit, concurrency, and reentrancy behavior.
- Actual final object contents when dependencies can mutate the same storage.

Intent is inferred only at the level of the role names used here. No ownership, synchronization, or error-handling guarantees are inferred from those names.

## Evidence

The contract is based solely on the supplied static function representation and opaque dependency mapping.

Established static evidence includes access widths, object-relative offsets, comparisons with zero and eight, explicit argument flow, dependency-call order, conditional auxiliary-field access, and the ordering of the final write before cleanup.

The supplied representation identifies the lookup-failure dependency as nonreturning; its underlying behavior is not independently established.

No dependency implementations or runtime observations were supplied. No runtime tests have been performed.
