<!-- Trusted provenance: F-dd5dd0cb2803; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dd5dd0cb2803

## Purpose

F-dd5dd0cb2803 conditionally processes a record whose 16-bit discriminator equals 10. Processing is further gated by a status operation on an externally obtained object. When both gates permit processing, the function decodes four values from a payload, performs an object lookup, and may forward a two-part result together with the decoded values to another dependency.

The function has two materially different completion modes:

- Paths that do not enter payload processing return the 32-bit value 4.
- Paths that enter payload processing reach a call marked non-returning, unless an earlier operation itself prevents further execution.

The non-returning call is reached even when lookup produces a null result or the looked-up object declines further processing. It must not be described merely as optional cleanup followed by a return.

The business meaning of discriminator 10, the payload values, and the object operations is not established.

## Interface

### Inputs

The function accepts two machine-level arguments:

| Argument | Established representation | Use |
|---|---|---|
| First argument | Opaque 32-bit value | Forwarded unchanged to F-b6884dd99886 if the forwarding condition succeeds. Otherwise not used directly. |
| Record address | Address represented by a 32-bit value in the supplied representation | Used to read the discriminator and, conditionally, a payload address. Also forwarded unchanged to F-b6884dd99886. |

The first argument may represent an address, handle, identifier, or another value. No interpretation is established.

The record has the following accessed layout:

| Byte offset from record address | Access | Meaning established by this function |
|---|---|---|
| +8 | 32-bit read | Payload address, read only after the discriminator and status gates succeed |
| +12 | 16-bit read | Discriminator, compared with 10 |

The comparison is expressed through a signed 16-bit read. Because the comparison value is positive and exactly representable, only the 16-bit value corresponding to 10 passes; signedness does not introduce additional accepted values.

No record length, payload length, ownership flag, or alignment field is examined.

### Payload access points

After loading the payload address, the function passes these addresses to decoding dependencies, in order:

| Payload-relative address | Dependency | Captured result |
|---|---|---|
| +0 | F-c25a2fcc75b5 | One 32-bit value used as a lookup argument |
| +4 | F-64e2dfdd4a5d | One 32-bit value later forwarded |
| +6 | F-64e2dfdd4a5d | One 32-bit value later forwarded |
| +8 | F-64e2dfdd4a5d | One 32-bit value later forwarded |

These are dependency call addresses, not proof of the number of bytes each dependency reads. In particular, the two-byte spacing of the last three addresses does not establish 16-bit return widths or 16-bit memory accesses.

All four decoded results are represented as 32-bit values. There is no visible masking, range check, or narrowing before their subsequent use.

### Outputs and completion

The only established ordinary return value is the 32-bit value 4.

An ordinary return occurs when either:

1. The discriminator is not 10; or
2. The discriminator is 10 and the external status result is nonzero.

If the discriminator is 10 and the status result is zero, the function proceeds toward F-feee9156a257, whose call is marked non-returning in the supplied static evidence.

No output buffer or direct caller-visible memory write is established.

## Behavior

### 1. Discriminator gate

The first observable record access is a 16-bit read at record offset +12.

If its value is not 10, the function returns 4. On this path it does not obtain an external object, read the payload address, invoke any decoding dependency, perform lookup, or call the terminal dependency.

There is no preliminary null or bounds check for the record address.

### 2. Obtain an object and query its status

For discriminator 10, the function calls F-b7af80e829ec with no explicit arguments.

Its result is treated as an object address. The function reads that object's dispatch-table pointer and selects the operation at byte offset +288, equivalent to hexadecimal +120. That operation is invoked with the object address as its explicit argument.

The result is interpreted as a 32-bit value:

- Any nonzero value causes an ordinary return of 4.
- Zero permits payload processing.

The function does not check whether the object returned by F-b7af80e829ec is null before accessing its dispatch table. Validity of that object and the selected operation is therefore a precondition for this path, rather than a condition enforced by the function.

No assumption is made that either dependency preserves global state or leaves the record unchanged.

### 3. Capture and decode the payload

Only after the status operation returns zero does the function read the 32-bit payload address from record offset +8.

This timing matters: changes made to that record field by earlier dependencies can affect the selected payload. The payload address is then retained locally for all four decoding calls; it is not reloaded from the record between them.

The function makes the four decoding calls in the order listed under Interface:

1. F-c25a2fcc75b5 at the captured payload address.
2. F-64e2dfdd4a5d at the captured address plus 4.
3. F-64e2dfdd4a5d at the captured address plus 6.
4. F-64e2dfdd4a5d at the captured address plus 8.

Each result is captured before the next call. These calls are not established to be pure. Later calls can observe memory or global changes made by earlier calls.

The first result becomes the lookup argument. The other three remain separate values, preserving their order for forwarding.

There is no check for a null payload address, insufficient accessible storage, arithmetic overflow, or an invalid decoded value. No decoder result is treated as a failure indicator by this function.

The address calculations use the 32-bit address representation shown in the supplied evidence; no widening or explicit overflow protection is visible. Behavior for addresses near the address-space boundary depends on the target's address arithmetic and validity rules and is not safely defined by this contract.

### 4. Look up a second object

After all four decoding calls, the function selects the operation at byte offset +244, hexadecimal +F4, from the dispatch table of the originally obtained object.

The dispatch-table pointer is read again for this operation. It is not established that the status query and lookup use an unchanged dispatch table: the intervening dependencies may have modified reachable state.

The lookup receives two explicit arguments:

1. The originally obtained object address.
2. The 32-bit result from F-c25a2fcc75b5.

Its result is treated as another object address.

A null lookup result skips the subsequent object operations and forwarding, but does not skip F-feee9156a257.

### 5. Conditionally obtain and forward a 64-bit result

For a nonnull lookup result, the function selects the operation at byte offset +8 from that object's dispatch table. The supplied representation shows no explicit arguments for this invocation.

Its result is interpreted as a 32-bit value:

- Zero suppresses forwarding.
- Any nonzero value permits the next operation.

If permitted, the function reads the object's dispatch-table pointer again and invokes the operation at byte offset +12. This invocation also has no explicit arguments in the supplied representation. Its result is captured as a 64-bit quantity.

The function separates this result into two 32-bit halves:

- The least-significant 32 bits.
- The most-significant 32 bits.

It then calls F-b6884dd99886 with seven arguments, in this order:

1. The original first argument.
2. The original record address.
3. The low 32-bit half.
4. The high 32-bit half.
5. The result decoded from payload offset +4.
6. The result decoded from payload offset +6.
7. The result decoded from payload offset +8.

The split is by numerical bit position, not by an established byte order in memory. Both halves retain their respective 32-bit patterns. Their semantic signedness is unknown.

No return value from F-b6884dd99886 is inspected. If that dependency returns, execution proceeds to the terminal call.

### 6. Terminal call

Once the zero-status payload-processing path has been entered, F-feee9156a257 is called with the lookup result, provided all preceding operations complete.

This includes all three cases:

- Lookup returned null.
- Lookup returned a nonnull object whose offset +8 operation returned zero.
- Lookup returned a nonnull object whose offset +8 operation returned nonzero, and forwarding subsequently returned.

The argument is the lookup result, not the originally obtained object.

The supplied evidence marks F-feee9156a257 as non-returning. Consequently, no ordinary return of 4 is established after this call. Whether it terminates execution, transfers control, unwinds, or performs another nonlocal action is unknown.

## Side effects

### Direct effects

The visible body performs reads and dependency calls. It does not directly write record fields, payload bytes, object fields, global storage, or an output buffer.

Its direct reads include:

- The record discriminator.
- The record's payload address, conditionally.
- Object dispatch-table pointers and selected dispatch entries.

Payload reads are delegated to the decoding dependencies; their exact widths and internal access order are not established.

### Dependency-mediated effects

Every invoked dependency may have effects not shown here. In particular:

- Global state must not be assumed stable across calls.
- The record and payload must not be assumed immutable across calls.
- Dispatch tables must not be assumed unchanged between separate selections.
- No dependency is established to be pure, idempotent, or thread-safe.

The body does not reread the discriminator after external calls. A later change to the discriminator therefore does not cause it to reconsider the initial gate.

The payload address is captured once, but its pointed-to contents are not copied into a local aggregate before decoding. Decoder-visible contents can therefore change between calls.

### Ownership and lifetime

No ownership transfer, retention, release, or allocation policy is established for either object or either input.

The final call's placement and argument do not prove that it destroys or releases the lookup result. Its receipt of null on one path also does not establish a general null-handling contract beyond this observed call site.

The function performs no visible balancing action on the originally obtained object when the status result is nonzero.

### Errors and invalid inputs

There are no explicit error codes other than the ordinary return value 4, and that value is used for both nonmatching records and nonzero status results.

There are no visible checks for invalid record addresses, invalid object dispatch tables, invalid payload storage, or malformed decoded contents. Failure modes for such inputs depend on the execution environment and dependencies; they are not defined as graceful rejection.

## Dependencies

| Dependency | Established role and contract boundary |
|---|---|
| F-b7af80e829ec | Called without explicit arguments when the discriminator is 10. Returns the address used for status and lookup dispatch. |
| F-c25a2fcc75b5 | Called once with the captured payload address. Produces the 32-bit lookup argument. Read width and decoding format are unknown. |
| F-64e2dfdd4a5d | Called three times, at captured payload offsets +4, +6, and +8. Each result is retained as 32 bits and forwarded in that order if forwarding occurs. |
| F-b6884dd99886 | Receives the two original arguments, two halves of a 64-bit result, and three decoded values. Its return value is unused. |
| F-feee9156a257 | Receives the lookup result, including null. Marked non-returning; its actual terminal mechanism and ownership effects are unknown. |

Four additional dependencies are selected indirectly through object dispatch tables: the status operation, lookup operation, readiness operation, and 64-bit result operation. No opaque function identities are supplied for them. Their byte offsets identify dispatch positions, not stable semantic names.

The two operations on the lookup result show no explicit receiver argument. Whether an implicit receiver or other ABI-specific input exists is documented_only.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Each assumes valid required memory and that dependencies return where the path requires them to return.

### Nonmatching record

Given a discriminator of 9, 11, or any other 16-bit value except 10:

- Return 4.
- Do not read the record's payload-address field.
- Do not call any dependency.

### Matching record with nonzero status

Given discriminator 10 and a status result of 1 or any other nonzero 32-bit value:

- Obtain the first object and invoke its status operation.
- Return 4.
- Do not decode the payload or perform lookup.
- Do not call F-feee9156a257.

### Lookup misses

Given discriminator 10, status zero, and a null lookup result:

- Complete all four decoder calls before lookup.
- Do not invoke operations on the null result.
- Do not call F-b6884dd99886.
- Call F-feee9156a257 with null.
- Do not expect an ordinary return.

### Looked-up object declines forwarding

Given a nonnull lookup result whose offset +8 operation returns zero:

- Do not invoke its offset +12 operation.
- Do not call F-b6884dd99886.
- Call F-feee9156a257 with the lookup result.

### Successful forwarding and half ordering

Suppose the offset +8 operation returns nonzero, and the offset +12 operation returns the 64-bit value 0x1122334455667788.

F-b6884dd99886 receives 0x55667788 as its third argument and 0x11223344 as its fourth argument. Its final three arguments are the results decoded from offsets +4, +6, and +8, respectively.

If forwarding returns, F-feee9156a257 is then called with the lookup result.

### Record mutation before payload capture

If an earlier dependency changes the payload-address field before the status operation returns zero, the later field read observes the value present at that read.

If a decoder changes that field after capture, the remaining decoder addresses still derive from the already captured address. This does not imply that the payload contents remain unchanged.

## Uncertainties

The following are documented_only:

- Business meanings of the discriminator, decoded fields, dispatch operations, and return value 4.
- Decoder access widths, byte order, sign extension, validation, and error behavior.
- Object layouts beyond the accessed dispatch mechanism.
- Calling-convention details for operations shown without explicit arguments.
- Whether the 64-bit result is logically a single integer, packed fields, or another representation.
- Allocation, ownership, lifetime, and release policies.
- Side effects and global-state interactions of every dependency.
- The mechanism by which F-feee9156a257 does not return.
- Exact target behavior for invalid addresses, boundary-crossing address arithmetic, and alignment violations.
- Concurrency and reentrancy guarantees.

Names such as “status,” “lookup,” and “forwarding” describe control-flow roles. They are not established domain semantics.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

Established evidence includes the two initial gates, accessed record offsets, decoder call order and address offsets, dispatch byte offsets, null check, nonzero readiness condition, 64-bit split, forwarding argument order, terminal-call placement, and ordinary return value.

The non-returning property is supplied as a static annotation; the terminal dependency's implementation was not provided. Semantic interpretation beyond the visible operations remains explicitly uncertain.

No runtime tests have been performed.
