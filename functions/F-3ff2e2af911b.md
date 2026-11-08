<!-- Trusted provenance: F-3ff2e2af911b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3ff2e2af911b

## Purpose

F-3ff2e2af911b performs a synchronous enumeration over a built-in location, accumulates identifiers from each returned record into a shared bitmap, and conditionally delivers records to a caller-supplied callback.

Bitmap accumulation precedes callback eligibility checking. Consequently, records not delivered to the callback can still change shared state. A callback result of zero stops further enumeration but does not undo updates already made.

The routine has no explicit return value. Its observable results are dependency calls, callback invocations, shared-state updates, and cleanup.

The interpretation of the built-in location as a directory and of returned records as filesystem entries is inferred intent, not an established interface requirement.

## Interface

### Inputs

The routine accepts two inputs:

| Input | Established use |
|---|---|
| Callback address | Invoked synchronously for eligible records. Its result is tested for zero. |
| Opaque 32-bit value | Retained and passed unchanged to F-41e251a24ab6 after each callback that returns nonzero. |

No null check is performed on the callback address. A valid callable address is required if enumeration can produce an eligible record.

The opaque value is not directly dereferenced or released by this routine. Its meaning and validity requirements belong to F-41e251a24ab6.

### Callback interface

The callback receives seven arguments, in this order:

1. A 32-bit token obtained once from F-33c2b4a03348 before enumeration initialization.
2. The unsigned first byte of the current record’s flag output, with a value from 0 through 255.
3. A 32-bit auxiliary output from the enumerator, reset to zero before each enumeration attempt.
4. A second 32-bit auxiliary output, also reset to zero before each enumeration attempt.
5. A constant zero.
6. A 32-bit value obtained from F-44362605ab79 after the current record has been transformed by F-f0b79b581bd5.
7. Another 32-bit enumerator output, which this routine does not initialize before enumeration.

A zero callback result requests termination. Every nonzero result permits enumeration to continue; positive and negative nonzero results are not distinguished.

The lifetime, representation, and ownership of the sixth argument are not established. In particular, the routine does not establish that a callback may retain it after returning.

### Internal layouts relevant to the contract

The supplied representation exposes:

- A 68-byte local enumeration context passed to initialization, enumeration, and finalization dependencies.
- A record descriptor comprising two consecutive 32-bit words. The first receives a captured default value; the second is reset to zero before each enumeration attempt.
- Two separate four-byte output areas. Only the first byte of each is inspected locally.
- Three additional 32-bit enumeration outputs.
- A 32-bit cursor-like value initialized through F-44362605ab79 and passed by address to F-d3922c7fe0ec.
- A shared bitmap object reached through a global pointer slot.

The shared bitmap object is accessed using these byte offsets:

| Offset | Access |
|---|---|
| +4 | A 32-bit count, incremented when a membership query returns zero. |
| +8 | A 32-bit address of an array of 32-bit bitmap words. |

No local access establishes the meaning of the object’s first word.

### Preconditions

Successful execution requires valid storage and compatible behavior from the dependencies. In particular:

- Enumeration must provide every output subsequently read on a successful result.
- The shared bitmap object and its backing array must be valid for every identifier processed.
- An eligible callback must be callable with the seven arguments above.
- Dependency-owned values must remain usable for the duration of their observed uses.

The routine performs no local bounds, capacity, allocation, or callback-validity checks.

## Behavior

### Initialization and retained values

The routine establishes a local mask of zero and retains the callback, opaque input, and built-in location reference.

It then performs these calls in order:

1. F-bfade2068003 with arguments 8 and 0.
2. F-33c2b4a03348, retaining its 32-bit result as the callback’s first argument.
3. F-ddeb195e4df1 with the enumeration context, built-in location reference, zero mask, and another zero argument.

After initialization, it reads a global 32-bit default value. That value is captured once and reused to initialize the first descriptor word on every enumeration attempt. The global is not reread for each record.

The local mask has no explicit update in the supplied body.

### Enumeration attempts

Before each call to F-b84ca98fab94, the routine:

- Resets the descriptor’s second word to zero.
- Sets its first word to the captured default.
- Resets both auxiliary callback outputs to zero.

It does not explicitly initialize the two four-byte flag/status output areas or the remaining 32-bit callback output before each call.

F-b84ca98fab94 receives the enumeration context and addresses of the descriptor, flag output, status output, and three auxiliary outputs.

A zero result ends enumeration immediately and proceeds to common cleanup. No identifiers or callback eligibility fields are processed for that attempt.

Any nonzero result is treated as a returned record. The routine does not distinguish positive success values from negative values. If the dependency uses negative values for errors, this caller still processes them as records.

### Identifier accumulation

For each returned record, F-d8cbf9aa79a9 is called with the descriptor’s second word. Its result supplies an unsigned 32-bit iteration count.

The routine then calls F-44362605ab79 on the descriptor to obtain the initial cursor-like value. This call occurs even when the count is zero.

For each of the reported identifiers:

1. The current shared bitmap object pointer is read from its global slot.
2. F-d3922c7fe0ec is called with the cursor’s address, producing a 32-bit identifier.
3. The identifier is divided into a bitmap word index and bit position:
   - Word index: identifier divided by 32, discarding the remainder.
   - Bit position: identifier modulo 32.
4. F-88658862e573 receives the previously captured object pointer, word index, and bit position.
5. If its result is zero, the routine increments the object’s 32-bit count at offset +4, then sets the selected bit in the backing array reached through offset +8.
6. If its result is nonzero, neither of those direct writes occurs.

The count write precedes the bitmap-word write.

The global object pointer is reread for every identifier, not merely once per record. However, for a particular identifier, it is captured before F-d3922c7fe0ec runs. Changes to the global pointer during that call do not redirect that identifier’s membership query or direct updates.

The routine also does not reread the global pointer after F-88658862e573. Its direct writes still target the captured object. The object’s count and backing-array pointer are read after that dependency returns, so mutations to those fields by the dependency can affect the subsequent writes.

### Integer widths and boundaries

Identifiers, the iteration count, the word index, and the loop index are represented as unsigned 32-bit quantities.

For an identifier of `0xffffffff`:

- The word index is `0x07ffffff`.
- The bit position is 31.
- The byte displacement within the backing array is `0x1ffffffc`.
- The selected bit is the high bit of a 32-bit word.

No signed-negative identifier rejection exists. A high-bit-set identifier is processed according to its unsigned bit pattern.

The bit operation targets a 32-bit word and uses a position from 0 through 31. The supplied representation does not establish portable source-language shift semantics independently of the represented machine-width operation.

No overflow guard is present for the count increment. The body shows a 32-bit increment and store; it does not establish a wider count or an overflow error policy.

The unsigned loop covers exactly the reported number of identifiers, assuming all calls return normally. A maximum count of `0xffffffff` does not require the loop index to wrap: the final processed index is `0xfffffffe`, followed by the terminating comparison at `0xffffffff`.

### Callback eligibility and delivery

After all identifiers for a record have been processed, the routine checks:

- Whether the bitwise intersection of the local mask and the first flag byte is zero.
- Whether the first status byte is zero.

Both conditions must hold.

Because the local mask is initialized to zero and has no explicit update, the flag-mask condition does not exclude any flag-byte value in the represented flow. The first status byte is therefore the effective local eligibility discriminator.

The other three bytes of each four-byte output area are not inspected locally.

For an eligible record, the routine:

1. Calls F-f0b79b581bd5 with the descriptor’s address, zero, and the built-in location reference.
2. Calls F-44362605ab79 on the resulting descriptor.
3. Invokes the callback with the seven arguments specified above.

The sixth callback argument comes from this second F-44362605ab79 call. It is not taken from the cursor used for identifier accumulation.

If the callback returns zero, the routine exits the loop without calling F-41e251a24ab6.

If the callback returns nonzero, F-41e251a24ab6 is called with the original opaque 32-bit input. Only then does execution reach per-record cleanup.

A nonzero status byte suppresses descriptor transformation, the callback-value extraction, the callback itself, and F-41e251a24ab6. It does not suppress identifier accumulation.

### Cleanup and exit

An unmapped descriptor-cleanup dependency is called:

- At the end of every iteration that continues.
- Once after leaving the loop.

Thus:

- A skipped record receives per-record cleanup.
- A callback-accepted record whose callback returns nonzero receives per-record cleanup.
- A record whose callback returns zero receives common cleanup rather than both cleanup calls.
- An enumeration attempt returning zero still leads to common descriptor cleanup.

After common descriptor cleanup, the routine calls F-eecd2e1474fa on the enumeration context, then F-85722f62e20a with arguments 8 and 1000, then returns without an explicit value.

These cleanup guarantees describe normal returns through the visible control flow. They do not establish behavior if a dependency faults, throws, terminates execution, or performs a nonlocal transfer.

## Side effects

Established direct effects are:

- Repeated updates to a shared object’s 32-bit count.
- Read-modify-write updates to selected 32-bit bitmap words.
- Writes to local descriptor, context, cursor, and output storage through dependencies.
- Synchronous invocation of the caller’s callback.
- Conditional invocation of F-41e251a24ab6.
- Initialization, cleanup, finalization, and fixed-argument service calls.

Bitmap changes are not transactional. Callback termination does not roll them back, and records withheld from the callback may still contribute identifiers.

No locking, atomic operation, or reentrancy guard is visible. Safe concurrent mutation of the shared object is not established. An unsynchronized bitmap read-modify-write could lose another writer’s update.

The descriptor-cleanup call suggests resource management, but it does not prove ownership transfer, deallocation, reference counting, or invalidation of callback-visible data. No dependency is assumed to preserve globals or to own or release memory merely because of its position in this routine.

## Dependencies

| Opaque ID | Observed role |
|---|---|
| F-bfade2068003 | Initial call with 8 and 0; result is not inspected. |
| F-33c2b4a03348 | Produces the retained 32-bit callback token. |
| F-ddeb195e4df1 | Initializes the 68-byte enumeration context using the built-in location and two zeros. |
| F-b84ca98fab94 | Produces successive records and associated outputs; zero terminates enumeration. |
| F-d8cbf9aa79a9 | Converts the descriptor’s second word into the identifier iteration count. |
| F-44362605ab79 | Produces the identifier cursor initially and the callback’s sixth argument after transformation. |
| F-d3922c7fe0ec | Produces the next identifier while receiving a mutable cursor address. |
| F-88658862e573 | Queries membership using an object pointer, word index, and bit position. Zero enables direct updates. |
| F-f0b79b581bd5 | Transforms an eligible record’s descriptor using zero and the built-in location. |
| F-41e251a24ab6 | Receives the opaque input after a nonzero callback result. |
| F-eecd2e1474fa | Finalizes the enumeration context. |
| F-85722f62e20a | Final call with 8 and 1000. |

A separate descriptor-cleanup dependency is visible, but no corresponding opaque ID was supplied. Its identity and detailed contract remain documented_only.

Only the return values explicitly described above affect local control flow. Uninspected results do not provide an established error-reporting channel for this routine.

## Acceptance examples

These are conditional acceptance criteria derived from the supplied static representation, not executed tests.

1. **Immediate exhaustion.** If the first enumeration call returns zero, no identifier is requested and no callback runs. Common descriptor cleanup, context finalization, and the final call with 8 and 1000 occur in that order.

2. **Status-based exclusion still accumulates identifiers.** If a returned record reports two identifiers and has a nonzero first status byte, both identifiers are processed before cleanup. No descriptor transformation or callback occurs.

3. **Flag bytes do not independently exclude records.** With a zero first status byte, flag values 0 and 255 both satisfy the local eligibility check because the mask is zero. The corresponding value is delivered as the callback’s second argument.

4. **New identifier at a word boundary.** If an identifier is 32 and the membership dependency returns zero, the count is incremented, then bit 0 of bitmap word 1 is set. Existing bits in that word are preserved.

5. **Existing identifier.** If the membership dependency returns nonzero, this routine performs no direct count or bitmap write for that identifier. Dependency side effects remain outside this assertion.

6. **Repeated identifier.** If the same identifier appears twice, the membership dependency is called twice. If it returns zero first and nonzero second, the direct count increment occurs once. The routine itself does not deduplicate before querying.

7. **Callback stop.** If an eligible record’s callback returns zero, its identifier updates have already occurred. F-41e251a24ab6 is omitted, no subsequent enumeration attempt occurs, and common cleanup follows.

8. **Callback continuation.** If the callback returns any nonzero value, including a negative value, F-41e251a24ab6 receives the original opaque input before per-record cleanup and the next enumeration attempt.

9. **Global-pointer replacement during extraction.** If F-d3922c7fe0ec replaces the global bitmap pointer, the current identifier still uses the object captured before that call. A subsequent identifier reads the replacement pointer.

10. **Zero identifiers.** If the reported count is zero, F-44362605ab79 still initializes the cursor-like value. No identifier extraction or membership query occurs. Eligibility checking and possible callback delivery still proceed.

## Uncertainties

The following remain documented_only:

- The exact built-in location value and its encoding.
- Whether the descriptor represents a string, collection, path, or another resource.
- The meanings of the auxiliary callback arguments.
- Whether the initial callback token represents time, context, identity, or something else.
- Whether enumeration distinguishes exhaustion from failure through outputs not interpreted here.
- Valid bitmap capacity and maximum supported identifier.
- Intended count-overflow behavior.
- Dependency mutation of globals, descriptor storage, or shared-object fields.
- Callback-value ownership and permitted retention.
- The cleanup dependency’s identity and resource semantics.
- Thread-safety and reentrancy requirements.
- The semantic meanings of the initial and final fixed-argument calls.

The apparent count of populated bitmap bits is inferred intent. Equality between that count and actual population is not guaranteed by this body alone; it depends on initial consistency and dependency behavior.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. It establishes visible argument order, local initialization, zero/nonzero branches, 32-bit arithmetic structure, byte-level eligibility checks, bitmap field offsets, direct write ordering, and normal cleanup paths.

Filesystem terminology, resource-management interpretations, and bitmap-population intent are explicitly inferential. Dependency internals and environmental validity conditions are not established.

No runtime tests, dependency executions, or independent implementation inspection have been performed.
