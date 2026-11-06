<!-- Trusted provenance: F-545f1faad10d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-545f1faad10d

## Purpose

F-545f1faad10d conditionally updates one of five 32-bit fields in a selected record. Selection depends on an implicit record selector and operation selector, in addition to the explicit value argument.

Two operations validate the value before writing. Three operations store the supplied 32-bit value without a range check. One validated operation invokes F-e4e2db384d33 after the write; the supplied static representation marks that dependency as non-returning.

Rejected requests may record a status in the first 32-bit word of a caller-provided context. An existing nonzero status is preserved. A rejected request can also leave the status unchanged when the configured rejection status is zero.

This describes the supplied static representation. No runtime tests have been performed. The interpretation of these fields as application-level properties is unknown.

## Interface

### Inputs

The interface has one explicit argument and three implicit inputs:

| Input | Representation | Role |
|---|---|---|
| Value | Signed 32-bit integer | Value to validate or store |
| Context address | Target address supplied through implicit machine state | Base for record addressing and location of the status word |
| Record selector | 32-bit quantity supplied through implicit machine state | Selects one of eight accepted selector values |
| Operation selector | 32-bit quantity supplied through implicit machine state | Selects validation, field offset, and any subsequent dependency call |

The record selector is accepted exactly when its 32-bit bit pattern is in the inclusive interval `0x4000` through `0x4007`. The operation selector recognizes `0x1205` through `0x1209`.

The implicit inputs are necessary parts of the effective interface. Supplying only the explicit argument is insufficient to determine behavior.

### Configuration reads

The function references three externally stored values:

- A record-address bias.
- A configured rejection status.
- An argument passed to F-e4e2db384d33.

Their numeric contents, storage ownership, initialization requirements, and mutability are not established by the supplied material.

### Address layout

Let the context address be the base address. For an accepted record selector, the record anchor is formed from:

- The context address.
- The externally stored record-address bias.
- The record selector multiplied by 168 bytes, or `0xA8`.

The multiplication uses the selector itself, not its distance from `0x4000`. A consumer must not substitute a zero-based record number unless some independently established configuration compensates for that transformation.

The fields accessed relative to that anchor are:

| Operation | Byte offset | Access width |
|---|---:|---:|
| `0x1205` | `0x64` / 100 | 32 bits |
| `0x1206` | `0x68` / 104 | 32 bits |
| `0x1207` | `0x58` / 88 | 32 bits |
| `0x1208` | `0x5C` / 92 | 32 bits |
| `0x1209` | `0x60` / 96 | 32 bits |

The status word occupies the first 32 bits at the context address. The source does not prove that this word is disjoint from every computed record field or configuration location.

### Outputs and preconditions

There is no defined return value. Observable results consist of memory changes, possible dependency invocation, and whether control returns.

Any address actually accessed must designate storage suitable for the indicated 32-bit read or write. No null-pointer, bounds, alignment, or allocation-lifetime checks are shown.

A context address need not be dereferenced on every path. For example, an invalid record selector causes status handling and therefore a context read, whereas a rejected value with a configured status of zero returns without that read.

## Behavior

### Record-selector gate

The selector gate uses an unsigned comparison after subtracting `0x4000`. Under the represented 32-bit unsigned arithmetic, acceptance requires the subtraction result to be less than eight.

Consequently:

- `0x4000` and `0x4007` are accepted.
- `0x3FFF` and `0x4008` are rejected.
- A high-bit-set selector is not accepted merely because it might represent a negative signed integer.

For a rejected selector:

1. No record-field address is used.
2. The record-address bias and configured rejection status are not read by the shown path.
3. The status candidate is fixed to `0x500`.
4. Status handling follows.

For an accepted selector, the function obtains the record-address bias and forms the record displacement. It then reads the configured rejection status before dispatching on the operation selector. This configured-status read therefore occurs even when an operation later succeeds and returns without using it.

### Operation `0x1205`

The value is accepted exactly when it lies in the signed inclusive range:

- Minimum: 0.
- Maximum: `0x800000`, or 8,388,608.

An accepted value is written as a 32-bit word at record offset `0x64`, and the function returns immediately. There is no explicit status-word read or write on that successful path.

A negative value or a value above `0x800000` causes no record-field write. The previously read configured rejection status becomes the candidate for status handling.

### Operation `0x1206`

The value is accepted when it is nonnegative and either:

- At most `0x5A0000`, or 5,898,240; or
- Exactly `0xB40000`, or 11,796,480.

The second accepted value is an isolated exception. Values strictly between `0x5A0000` and `0xB40000` are rejected, and values above `0xB40000` are rejected.

For an accepted value, the established order is:

1. Write the 32-bit value at record offset `0x68`.
2. Obtain the externally stored dependency argument.
3. Invoke F-e4e2db384d33 with that argument.

The write precedes both the dependency-argument read and the dependency invocation. If those locations alias, the argument read can observe the preceding write.

The supplied representation marks F-e4e2db384d33 as non-returning. Whether it terminates execution, transfers control, blocks indefinitely, or performs some other action is unknown. The field write has already occurred when it is invoked; no rollback is shown.

For a rejected value, the dependency is not invoked and the record field is not written. The previously read configured rejection status is used for status handling.

### Operations `0x1207`, `0x1208`, and `0x1209`

These operations write the supplied 32-bit value without a range check:

- `0x1207` writes offset `0x58`.
- `0x1208` writes offset `0x5C`.
- `0x1209` writes offset `0x60`.

Each returns immediately after its write. Negative signed values retain their 32-bit representation; they are not clamped or rejected.

These successful paths contain no explicit status-word access after dispatch. Aliasing can nevertheless make a field write affect storage also used as status.

### Unrecognized operations

For an accepted record selector but any other operation selector:

- The record-address bias and configured rejection status have already been read.
- No record field is written.
- The configured rejection status is replaced as the status candidate by the fixed value `0x500`.
- Status handling follows.

Thus unrecognized operations do not use the configured status, even though they read it.

### Status handling

Status handling uses a 32-bit candidate:

- If the candidate is zero, return without reading or writing the context status word.
- Otherwise, read the status word.
- If its current value is zero, write the candidate there.
- If its current value is nonzero, leave it unchanged.
- Return.

A configured rejection status is tested only for zero versus nonzero. No positivity check is shown. A negative configured status can therefore be stored as its 32-bit representation.

The function does not clear an existing status on success or rejection. The read-then-conditional-write sequence is not shown to be atomic and does not establish a concurrency-safe “first error wins” guarantee.

### Arithmetic constraints

The accepted selector multiplied by `0xA8` ranges from `0x2A0000` through `0x2A0498`, so this multiplication alone does not overflow a signed 32-bit integer.

The record-address bias and context address are not bounded by the supplied evidence. Overflow behavior of the subsequent integer-address additions is not fully established. The contract therefore requires usable resulting addresses rather than asserting portable signed-overflow or address-wrap behavior.

## Side effects

Established direct effects are limited to:

- Reading configuration values on the paths described above.
- Writing one selected 32-bit record field on a successful operation.
- Reading and conditionally writing the 32-bit context status word during status handling.
- Invoking F-e4e2db384d33 after a successful `0x1206` write.

No allocation, release, ownership transfer, logging, or I/O is directly shown.

The caller must keep directly accessed storage valid for those accesses. No memory ownership or lifetime guarantee can be assigned to F-e4e2db384d33 from this evidence. Its effects on globals, memory, and control flow remain unknown.

Invalid addresses can fault or otherwise fail according to the execution environment. The function provides no demonstrated recovery path for such failures.

## Dependencies

### F-e4e2db384d33

- Invoked only for an accepted record selector, operation `0x1206`, and an accepted value.
- Receives one argument obtained from externally stored configuration after the record-field write.
- Is marked non-returning in the supplied static representation.
- Has no supplied behavioral contract.

Do not assume that it preserves globals, preserves the context, owns memory, releases memory, or leaves the just-written value unchanged.

No other function invocation is shown.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Assume valid storage and non-overlapping status, field, and configuration locations unless an example states otherwise.

| Conditions | Required observable result |
|---|---|
| Selector `0x4000`, operation `0x1205`, value 0 | Read the applicable configuration, write 0 at offset `0x64`, return without explicit status access |
| Selector `0x4007`, operation `0x1205`, value `0x800000` | Accept the upper boundary and write the value |
| Accepted selector, operation `0x1205`, value `0x800001`, configured status zero | No field write; return without reading the status word |
| Accepted selector, operation `0x1205`, value −1, configured status nonzero, current status zero | No field write; store the configured status in the status word |
| Accepted selector, operation `0x1206`, value `0x5A0000` | Write offset `0x68`, then read the dependency argument and invoke F-e4e2db384d33 |
| Accepted selector, operation `0x1206`, value `0x5A0001` | Reject; no field write or dependency invocation |
| Accepted selector, operation `0x1206`, value `0xB40000` | Accept the isolated exception; write before invoking the dependency |
| Accepted selector, operation `0x1206`, value `0xB40001` | Reject; apply configured-status handling |
| Accepted selector, operation `0x1207`, value −1 | Write the 32-bit representation of −1 at offset `0x58`; return |
| Accepted selector, operation `0x1208` or `0x1209` | Write offset `0x5C` or `0x60`, respectively, without value validation |
| Selector `0x3FFF` or `0x4008`, current status zero | Store `0x500`; do not read record-address or configured-status configuration |
| Accepted selector, unrecognized operation, current status zero | Read record-address and configured-status configuration, then store `0x500` |
| Any status-handling path with a nonzero candidate and an existing nonzero status | Read but preserve the existing status |

If the dependency-argument storage aliases the destination of an accepted `0x1206` write, the argument read occurs after that write. An implementation that caches that argument before writing would not preserve the established ordering.

## Uncertainties

The following remain **documented_only**:

- The external calling convention for the implicit inputs.
- The application meaning of the selectors, fields, configuration values, and statuses.
- The concrete context layout beyond the demonstrated accesses.
- Byte order, alignment requirements, memory protection, and volatile or device-memory semantics.
- Whether configuration, context, and record-field storage can alias in supported use.
- The exact address-overflow semantics of the execution environment.
- F-e4e2db384d33’s behavior and the reliability of its non-returning annotation.

The visible continuation after the dependency call leads toward configured-status handling if that call returns. Because the representation explicitly marks the call non-returning, unexpected-return behavior is not established as a supported contract.

Inferred intent, not established semantics: the record stride and eight accepted selectors suggest a record collection, while conditional status assignment suggests preservation of an earlier error. Neither inference establishes allocation size, synchronization, or an application-level error taxonomy.

## Evidence

The supplied static representation establishes:

- The unsigned eight-value record-selector gate.
- The `0xA8` stride and five field offsets.
- The two value-validation rules and three unconditional stores.
- The configured-status read before operation dispatch for accepted selectors.
- The fixed `0x500` status candidate for invalid selectors and unrecognized operations.
- Zero-candidate suppression of status access and preservation of nonzero existing status.
- The `0x1206` field-write-before-dependency sequence.

No runtime observations, dependency implementation, external layout definition, or independent calling-convention evidence were supplied. No runtime tests have been performed.
