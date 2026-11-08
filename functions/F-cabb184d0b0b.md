<!-- Trusted provenance: F-cabb184d0b0b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cabb184d0b0b

## Purpose

F-cabb184d0b0b operates on an object containing an embedded synchronization-like structure. It selects between two behaviors:

- When an object state field is nonzero and the second argument is exactly `1`, it obtains a status, conditionally clears that state field, invokes an exit dependency, and either returns that dependency’s result or dispatches an indirect callback.
- Otherwise, it checks and potentially releases the embedded structure according to a marker, an owner-like value, mode bits, and a 16-bit nesting field.

The synchronization, ownership, and nesting terminology describes inferred intent. The established behavior is the sequence of field accesses, dependency calls, comparisons, writes, and returns below. The supplied material does not establish a complete locking protocol.

## Interface

### Arguments and result

The function accepts two machine-level arguments:

1. **Object address:** represented as a 32-bit integer in the supplied representation and used as the base address for field access.
2. **Selector:** a 32-bit integer. Only equality to `1` is significant; other values select the same alternative path when the special path is not otherwise eligible.

The result is a 32-bit integer. Depending on the path, it is:

- A result returned by a dependency.
- The result returned by an indirect callback.
- The literal value `0`.
- The literal value `5`.
- The literal value `26`.

No general success/error interpretation is established for these values.

### Object layout

Offsets are byte offsets from the object address. Widths below describe accesses visible in the supplied representation.

| Offset | Width | Use |
|---|---:|---|
| `+0x18` | 32 bits | Address of a callback receiver object |
| `+0x20` | 32 bits | Argument passed to F-cc972bc08727 |
| `+0x24` | 32 bits | State field controlling special-path eligibility |
| `+0x28` | 32 bits | Leading marker of the embedded structure |
| `+0x2c` | 32 bits | Owner-like field |
| `+0x36` | 16 bits, read unsigned | Flags used to select nesting behavior |
| `+0x3a` | 16 bits | Nesting-like field, read with signed interpretation |
| `+0x3c` | Address of embedded storage | Argument passed to F-259d2fe281b0 |

The embedded structure begins at `+0x28`. Its total size and the extent of storage needed by dependencies are unknown.

The callback receiver has a first 32-bit word interpreted as a dispatch-table address. A callable address is obtained from byte offset `+8` within that table. The callback receives the receiver address and a 32-bit argument equal to either `0` or `1`.

### Preconditions and validity

The caller must provide storage valid for all accesses taken on the selected path. The callback path additionally requires a valid receiver, a readable dispatch table, and a callable entry at the described slot.

Dependencies can impose additional preconditions not recoverable here.

There is no general null-object check. The function calls its entry dependency before testing the embedded-structure address or reading the path-selection field. Consequently, the later address check does not make an invalid object safe.

## Behavior

### Entry and path selection

The first observable operation is a call to F-UNMAPPED-01 with the address of the embedded structure, object address plus `0x28`. Its return value is not used.

Only after that call does the function read the 32-bit state field at `+0x24`.

The special path is selected if both conditions hold:

- The state field is nonzero.
- The selector equals `1`.

Otherwise, execution follows the embedded-structure path.

The state field used for this decision is its value after F-UNMAPPED-01 returns. No preservation of object fields or global state by that dependency is assumed.

### Special path: status processing and callback dispatch

The function reads the 32-bit value at `+0x20` and passes it to F-cc972bc08727. It retains the returned status for subsequent decisions.

If that status is anything other than decimal `33`, the function writes a 32-bit zero to the state field at `+0x24`.

If the status is `33`, this function does not perform that clear. This does not guarantee that the field remains unchanged: the invoked dependencies may modify it.

Next, the function calls F-UNMAPPED-02 with the embedded-structure address. This call occurs for every status on the special path and precedes any callback receiver or dispatch-table lookup.

Subsequent behavior depends on the retained status:

- **Status `33`:** return F-UNMAPPED-02’s result. Do not resolve or invoke the callback.
- **Status `0`:** resolve the callback and invoke it with the receiver address and `1`. Return the callback’s result.
- **Any other status:** resolve the callback and invoke it with the receiver address and `0`. Return the callback’s result.

On either callback path, F-UNMAPPED-02’s result is discarded.

The callback receiver is read from `+0x18` after F-UNMAPPED-02 returns. Its dispatch-table address and callable slot are then read. Thus, changes to the receiver field or dispatch structures by earlier dependencies can affect which callback executes. The callback argument nevertheless remains determined by the retained status, not by any later reread of object state.

The embedded-structure path is not entered after this processing.

### Embedded-structure path: address and marker checks

The function forms the address at object offset `+0x28`.

If that computed address is zero, it returns `26` without performing the later marker, flags, owner, or nesting accesses on this path. The initial F-UNMAPPED-01 call has already occurred.

This is a check of the computed embedded address, not of the original object address. On a 32-bit wrapping-address interpretation, a nonzero base could produce a zero result. The supplied material does not establish portable pointer-overflow semantics or that such an input is valid.

For a nonzero embedded address, the function reads its leading 32-bit marker and compares it with a global marker value.

- If the values differ, it skips F-621dae46a146.
- If the values match, it calls F-621dae46a146 with the embedded address and a second argument of zero.
  - A nonzero result is returned immediately.
  - A zero result permits processing to continue.

There is no visible marker reread after that dependency returns.

### Owner comparison and mode selection

After the marker stage succeeds or is skipped, the function reads the unsigned 16-bit flags at `+0x36`. This flags value is retained.

It then calls F-196812b4b122 with no arguments. After the call returns, it reads the 32-bit owner-like field at `+0x2c` and compares it with that result.

If they differ, the function returns `5`. It performs no direct nesting or owner write on this branch and does not invoke F-259d2fe281b0.

This ordering matters:

- Flags are captured before F-196812b4b122.
- The owner field is read after F-196812b4b122.
- Changes to flags during that call do not replace the captured flags.
- Changes to the owner field during that call do affect the comparison.

If the owner matches, the function evaluates the captured flags. The nesting mode is selected exactly when masking the flags with `0x003f` and shifting right by four produces `2`. Equivalently, bit 5 is set and bit 4 is clear; the other bits do not affect this decision.

### Matching owner without nesting mode

When the owner matches but nesting mode is not selected, the function:

1. Writes 16-bit zero to `+0x3a`.
2. Writes 32-bit zero to `+0x2c`.
3. Calls F-259d2fe281b0 with the address at `+0x3c`.
4. Returns that dependency’s result.

This path does not need to read the previous nesting value.

The two clears occur before the dependency call. No restoration of those values is visible if the dependency returns an error-like result.

### Matching owner with nesting mode

When the owner matches and nesting mode is selected, the function reads the 16-bit nesting field at `+0x3a`, decrements it, and stores the 16-bit result back to that field.

The stored result is then tested against zero:

- **Nonzero:** return `0`. There is no direct owner clear and no call to F-259d2fe281b0.
- **Zero:** write 16-bit zero to the nesting field again, then write 32-bit zero to the owner field, call F-259d2fe281b0 with the address at `+0x3c`, and return its result.

The decrement uses a signed 16-bit read followed by assignment to a 16-bit value. The supplied representation therefore supports a narrowed 16-bit result, not an unbounded integer counter. Exact behavior at the signed conversion boundary remains an implementation-model uncertainty.

Under the ordinary low-16-bit narrowing interpretation:

- `1` becomes `0` and triggers the owner clear and dependency call.
- `2` becomes `1` and returns `0` without that call.
- `0` becomes `0xffff`, interpreted as `-1`, and does not trigger release.
- `0x8000` becomes `0x7fff`.

There is no visible check for an invalid initial nesting count, overflow, or underflow.

## Side effects

Direct writes by this function are limited to:

- Clearing the state field at `+0x24` on the special path when the retained status is not `33`.
- Updating or clearing the 16-bit nesting field at `+0x3a` on matching-owner paths.
- Clearing the owner-like field at `+0x2c` when proceeding to F-259d2fe281b0.

Calls may introduce additional effects. No dependency is established to preserve globals, object fields, memory ownership, or previously supplied addresses.

The function visibly passes addresses into existing storage. It performs no explicit allocation, deallocation, reference-count operation, or ownership transfer. That observation does not constrain what its dependencies or callback may do.

Not every entry-dependency call has a visible matching exit-dependency call. F-UNMAPPED-02 is invoked only on the special path; the other path uses separate marker and owner processing. The complete synchronization balance cannot be established from this function alone.

## Dependencies

F-UNMAPPED-01 and F-UNMAPPED-02 are document-local opaque identifiers for dependencies lacking supplied public IDs. Their internal behavior is documented_only.

| Opaque function ID | Visible interaction | Unknowns |
|---|---|---|
| F-UNMAPPED-01 | Called first with the embedded address; result ignored | Meaning, success conditions, synchronization effects, mutations |
| F-cc972bc08727 | Receives the value read at `+0x20`; returns the status controlling the special path | Status meanings beyond the visible comparisons |
| F-UNMAPPED-02 | Receives the embedded address after status handling; result returned only for status `33` | Meaning, mutations, ownership effects |
| F-621dae46a146 | Called with embedded address and zero when the marker matches; nonzero result returned immediately | Marker protocol and result meanings |
| F-196812b4b122 | Called without arguments; result compared with the subsequently read owner field | Whether the result denotes a thread, task, token, or another identity |
| F-259d2fe281b0 | Receives address at `+0x3c` after nesting and owner are cleared; result returned unchanged | Release semantics, failure effects, memory ownership |
| F-UNMAPPED-03 | Indirect callback resolved through the receiver’s dispatch table; receives receiver and `0` or `1` | Concrete target, calling convention details, side effects, ownership |

The global marker’s value, initialization, and mutability are not supplied. The comparison is established; its semantic meaning is not.

## Acceptance examples

These are conditional behavioral examples, not executed tests. Dependency effects must be held consistent with each example’s stated conditions.

1. **Special status defers callback dispatch.**  
   After F-UNMAPPED-01 returns, the state field is nonzero and the selector is `1`. F-cc972bc08727 returns `33`. The function performs no direct state-field clear, calls F-UNMAPPED-02, returns its result, and performs no callback lookup.

2. **Special status reports the positive callback argument.**  
   The special path is selected and the status is `0`. The function clears the state field before F-UNMAPPED-02. After that dependency returns, it resolves the callback, supplies argument `1`, and returns the callback result.

3. **Other special statuses use the negative callback argument.**  
   With the special path selected and status `7`, the function clears the state field, calls F-UNMAPPED-02, then invokes the callback with argument `0`. The callback result, not `7` and not F-UNMAPPED-02’s result, is returned.

4. **Selector equality is exact.**  
   A nonzero state field with selector `2` does not select the special path. It proceeds through the embedded-structure path.

5. **Marker dependency short-circuits later processing.**  
   The embedded address is nonzero, its marker matches, and F-621dae46a146 returns a nonzero value. That value is returned without a subsequent flags read or F-196812b4b122 call.

6. **Owner mismatch prevents direct release writes.**  
   Marker processing permits continuation, but the owner field read after F-196812b4b122 differs from its result. The function returns `5` without directly writing nesting or owner fields and without calling F-259d2fe281b0.

7. **A nested operation remains held.**  
   The owner matches, captured flags select nesting mode, and the nesting field is `2` when read. The field becomes `1`; the function returns `0` without clearing the owner or calling F-259d2fe281b0.

8. **The final nested level proceeds to release.**  
   With matching owner, nesting mode, and initial nesting value `1`, the decrement stores zero. The function writes zero again to nesting, clears the owner, calls F-259d2fe281b0, and returns its result.

9. **Non-nesting mode disregards the old nesting value.**  
   With matching owner and captured flags outside nesting mode, the function clears nesting and owner and calls F-259d2fe281b0 regardless of the previous nesting contents.

10. **Release failure does not cause visible rollback.**  
    If F-259d2fe281b0 returns a nonzero result, that result is returned. The function performs no subsequent restoration of the fields it cleared before the call.

## Uncertainties

The following remain documented_only:

- Exact entry, exit, marker-validation, identity, and release semantics.
- Valid object states and required external synchronization.
- Whether any dependency mutates or invalidates object storage.
- The global marker’s identity and lifetime.
- The callback’s concrete target and ownership contract.
- Platform behavior for address wraparound and out-of-range signed narrowing.
- Whether the apparent indirect call has additional ABI requirements not visible in the supplied representation.
- Semantic names for results `5`, `26`, and status `33`.

Locking, recursive ownership, and final release are inferred intent, not independently established API guarantees.

## Evidence

Established static evidence consists of the supplied function representation and dependency-ID mapping. It supports the field widths and offsets, branch conditions, dependency arguments, direct-write order, callback lookup structure, and result selection described above.

The specification does not rely on embedded comments as authority for semantic intent. It does not assume that unknown callees preserve globals, preserve object fields, or acquire or release memory.

No runtime tests have been performed. Acceptance examples are predictions from the visible control flow under their stated conditions, not observations of execution.
