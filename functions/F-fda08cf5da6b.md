<!-- Trusted provenance: F-fda08cf5da6b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fda08cf5da6b

## Purpose

F-fda08cf5da6b coordinates several mutually exclusive response paths for an object: immediate navigation, selection notification, or ordinary activation. Ordinary activation includes two service-related check/operation sequences, pending-action dispatch, and a conditional secondary notification.

These role descriptions are inferred intent. Established static behavior consists of the accesses, branch conditions, argument distinctions, and call ordering below.

Every visible path that completes normally clears the object’s byte at offset 198 and returns the 32-bit value 1. That value does not distinguish paths or report whether any dependency accomplished its apparent purpose.

## Interface

### Inputs and output

- **Object pointer:** a 32-bit address identifying storage that supports all accesses and indirect calls reached on the selected path.
- **Activation mode:** a 32-bit integer. The function distinguishes zero from nonzero; negative values follow the nonzero branch.
- **Result:** the 32-bit value 1 on normal completion.

No explicit input validation, error return, rollback, or recovery is visible.

### Directly accessed layouts

All offsets below are in bytes.

| Storage | Offset | Width | Use |
|---|---:|---:|---|
| Input object | 0 | 32 bits | Method-table pointer |
| Input object | 176 | 32-bit read; 8-bit write | Pending action; only its first addressed byte is explicitly cleared |
| Input object | 198 | 8 bits | Secondary-notification gate and final clear |
| Input object | 199 | 8 bits | Immediate-navigation flag and ordinary-activation gate |
| Initial context | 236 | Character-sized read | Conditional entry processing |
| Selection-service storage | 128 | 16-bit write | Set to 1 on a selection match |
| Saved root object | 1016 | 32-bit reads | Secondary-notification branching and context argument |

The input object’s method table supplies methods at offsets 284 and 288. Both receive the input object and a second value; their results are ignored.

A separate gate object supplies a method at method-table offset 420. The representation shows no explicit arguments to that invocation. Its implicit calling convention is unresolved.

The function also uses:

- Four-byte construction buffers.
- Additional four-byte temporary-storage areas for selection constructions.
- A four-word, 16-byte selection argument area, with 32-bit words numbered 0 through 3.
- Global storage supplying a root pointer, lock-enable byte, comparison target, saved context word, and three opaque notification values.

The object and all reached storage must remain usable after calls that precede further accesses. Passing an address to a dependency does not establish ownership transfer, non-retention, or lifetime extension.

## Behavior

### 1. Entry processing

The first call is F-dafcc25e1374, with no visible arguments. F-753f887b468d is then called without visible arguments to obtain an initial context.

The function calls F-06e8aeec34c6 with that context, then F-5eac81954d4b with the same context and zero, if either:

- Activation mode is nonzero; or
- The character-sized value at context offset 236 compares equal to −1.

Nonzero mode short-circuits the context-character test. The condition is not equivalent to testing whether that byte is nonzero. Character interpretation remains subject to the uncertainty described below.

Next, the function reads the global root pointer and passes it to F-c49a155bf927. That dependency supplies a result represented as two 32-bit halves.

### 2. Immediate-navigation precedence

A nonzero lower half selects the first-boot navigation path immediately.

Otherwise, the function calls F-eea3e31f6354 with zero and the upper result half, then calls F-92b5ab85f830. A nonzero result selects notes navigation.

If that result is zero, the function reads the lock-enable global byte. Only when it is nonzero does the function call F-6ac70711bbb5 followed by F-b7a2a0b657d4. A zero result from the latter selects lock navigation.

The priority is therefore:

1. First boot.
2. Notes.
3. Enabled lock check returning zero.
4. Selection detection and ordinary activation.

Each immediate-navigation path performs the following observable sequence:

1. Write 1 to object byte 199.
2. Call F-9b9c1c7483f4 with a construction-buffer address and the selected text address: `GotoFirstBoot`, `GotoNotesApp`, or `GotoLockApp`.
3. Load the object’s method at table offset 284 and invoke it with the object and the **constructor’s returned value**.
4. Call F-8a5aadac202f with the **construction-buffer address**, not necessarily the returned value.
5. Clear object byte 198 and return 1.

These constructions have two visible arguments. Unlike selection constructions, they have no additional temporary-storage pointer.

No selection comparison, service sequence, action dispatch, or secondary-notification phase follows immediate navigation. There is no later direct clear of byte 199 on these paths, although dependencies may modify it.

### 3. Selection detection and target reads

If immediate navigation is not selected, the function considers these candidates in order:

1. `SelectArtist`
2. `SelectAlbum`
3. `SelectCompilation`
4. `StartGenius`

A nonzero F-8094c3d72052 result means a match for this function. Evaluation stops at the first such result.

Each candidate construction calls F-9b9c1c7483f4 with three visible arguments:

- Its construction-buffer address.
- Its text address.
- A separate temporary-storage address.

Constructor results and construction-buffer addresses are not interchangeable in this contract:

- The first comparison receives the first constructor’s **returned value**.
- Each later comparison receives that candidate’s **construction-buffer address**. Those constructor return values are not used for comparison.

After constructing the first candidate, the function accesses the comparison-target global twice:

1. Read it into a saved target value.
2. Read it again as the first comparison’s target argument.

The first comparison therefore uses the second read. Every later comparison uses the saved value from the first read. There is no visible reload for those later comparisons. Equality of the two reads is not guaranteed by this function.

After the match decision, F-8a5aadac202f receives every constructed candidate’s construction-buffer address, in reverse construction order. It does not receive the constructor result merely because that result was used by the first comparison. The first candidate is always included in cleanup.

Only after all reached candidate-cleanup calls does the function read a separate 32-bit global into the **saved context word**. That saved word supplies subsequent selection or ordinary-notification arguments.

### 4. Selection-match path

When any candidate matched, the function:

1. Calls F-fdd8589eca2d.
2. Writes the 16-bit value 1 at offset 128 of the returned storage.
3. Calls F-78801ca166f2 with the address of a four-word temporary area.
4. Writes that area in this exact order:
   - Word 1 receives the saved context word bitwise-ORed with its signed 32-bit interpretation arithmetically shifted right by 23 bits.
   - Word 0 receives zero.
   - Word 2 receives zero.
   - Word 3 receives zero.
5. Calls F-fdd8589eca2d again.
6. Calls F-8fa83bbc2366 with the second accessor result, the saved context word, and words 0, 1, 2, and 3 in that argument order.
7. Constructs a selection-notification value through F-9b9c1c7483f4.
8. Calls the object’s method at table offset 284 with the object and the constructor’s returned value.
9. Calls F-8a5aadac202f with the notification’s construction-buffer address.
10. Clears object byte 198 and returns 1.

The word-write order is distinct from the argument order. F-78801ca166f2 has already received the area’s address; its retention or exposure of that storage is unknown, so the writes must not be treated as an unordered replacement.

The bitwise expression uses the original saved 32-bit word for both operands. It is not addition. Representative results are:

| Saved context word | Word 1 |
|---|---|
| `0x00000000` | `0x00000000` |
| `0x00800000` | `0x00800001` |
| `0x80000000` | `0xFFFFFF00` |

The selection-notification construction has three visible arguments: construction buffer, text address, and an additional temporary-storage pointer. Its text address is one byte beyond the address identified by `MusicSelected`. The undisplaced text must not be substituted; the actual text at the displaced address is unresolved.

The two F-fdd8589eca2d results are not assumed equal. This path does not enter ordinary activation or secondary notification.

### 5. Ordinary activation: service sequences

If every candidate comparison returned zero, the first service sequence is:

1. Call F-fdd8589eca2d.
2. Pass its result and the object to F-c7ba4c901230.
3. If the check returns zero, call F-fdd8589eca2d again, then pass this new result and the object to F-8f9bd59e3ec5.

The second accessor call must not be replaced by reuse of the first result.

The second service sequence is:

1. Call F-ac356e5603c7.
2. Pass its result and the object to F-c7ba4c901230.
3. If the check returns zero, pass the **same saved service result** and the object to F-8f9bd59e3ec5.

“Service,” “membership,” and “registration” describe inferred roles. Only these conditional call sequences are established.

### 6. Ordinary activation: pending action or forced action

The function calls F-187151a4b5ba with the object. Its returned value does not determine the subsequent branch:

- With zero activation mode, it is replaced by a read of object byte 199.
- With nonzero activation mode, forced activation is selected regardless of the result.

The call itself remains observable.

If mode is zero and the subsequently read byte 199 is zero, the function reads the full 32-bit word at object offset 176, then clears only the byte at offset 176. Dispatch uses a character-sized conversion of the saved word:

| Converted action | Dependency, called with object and zero |
|---|---|
| 0 | None |
| 1 | F-6f3e0e96361a |
| 2 | F-16f0e29b138e |
| 3 | F-6b95b88ebbda |
| 4 | F-f13086bc6f2a |
| 5 | F-4bb453fa7166 |
| Other | None |

The clear precedes any action dependency. The remaining three bytes of the pending-action word are not explicitly cleared.

Otherwise, the function clears byte 176, clears byte 199, and then calls F-6f3e0e96361a with the object and zero. This forced branch does not read the pending-action word.

Pending action 1 reaches the same dependency but has no explicit byte-199 clear. These write sequences are not identical.

### 7. Optional secondary notification

After dispatch and any action dependency, the function reads object byte 198. Consequently, action-side mutations can affect this gate.

If the byte is nonzero, the function calls F-87be4aa684d1, reads the returned object’s method table, and invokes its method at offset 420. Only a nonzero method result permits further secondary processing.

On continuation:

1. Read the global root pointer again and save this root value.
2. Call F-fdd8589eca2d.
3. Call F-6cf78263cbb0 with that accessor result and the saved context word.
4. Read the 32-bit field at offset 1016 of the saved root for a zero test.

The root is saved before the two calls, but the field test occurs afterward.

If the field test is zero:

1. Load the input object’s method pointer at table offset 288.
2. Read the first opaque notification global.
3. Invoke the loaded method with the object and that value.

If the field test is nonzero, the function **reads the same field again** as an argument to F-753f887b468d. It does not necessarily reuse the tested value. An additional register-carried argument has unresolved provenance.

The function then calls F-6904540d62b2 with that result and zero. Its two-part result selects the remaining branches:

- **Lower half zero:** load the object’s method pointer at table offset 288, then read the second opaque notification global, then invoke the loaded method with the object and that value.
- **Lower half nonzero:** call F-753f887b468d with both result halves, then F-5eac81954d4b with its result and zero. Next load the object’s method pointer at table offset 288, read the third opaque notification global, and invoke the loaded method.

In each branch, method-pointer loading precedes the corresponding notification-global read. The function does not first select a notification value and then perform one shared method load.

Whether this phase is skipped, rejected by the gate method, or completed, normal completion clears object byte 198 and returns 1.

## Side effects

Established direct writes are:

- Object byte 199: set to 1 during immediate navigation; cleared during forced ordinary activation.
- Object byte 176: cleared during pending-action consumption or forced activation.
- Object byte 198: cleared at every visible normal exit.
- A 16-bit field at offset 128 of the first selection-path accessor result: set to 1.
- The four-word selection area: written in order 1, 0, 2, 3 after F-78801ca166f2.

Construction and auxiliary temporary areas are exposed to dependencies. Explicit cleanup calls target construction-buffer addresses; no corresponding cleanup of the auxiliary areas is visible.

Dependencies and indirect methods may change globals, object fields, retained state, or ownership. No preservation or release contract is established for them. In particular, constructor return values are not assumed to alias construction buffers.

No explicit release is shown for context pointers, service values, root objects, or the secondary gate object. A failure to return or an invalid memory access can prevent final clearing and return. Exception and unwinding behavior are unresolved.

## Dependencies

| Opaque ID | Visible use |
|---|---|
| F-dafcc25e1374 | Initial unconditional call |
| F-753f887b468d | Initial context access and later transformations with differing visible argument forms |
| F-06e8aeec34c6 | Conditional entry-context operation |
| F-5eac81954d4b | Context operation with zero |
| F-c49a155bf927 | Root-dependent, two-part navigation result |
| F-eea3e31f6354 | Receives zero and the upper navigation-result half |
| F-92b5ab85f830 | Nonzero selects notes navigation |
| F-6ac70711bbb5 | Preliminary enabled-lock operation |
| F-b7a2a0b657d4 | Zero selects lock navigation |
| F-9b9c1c7483f4 | Construction with two or three visible arguments |
| F-8094c3d72052 | Candidate comparison; nonzero means match |
| F-8a5aadac202f | Explicit construction-buffer cleanup |
| F-fdd8589eca2d | Repeated service/storage accessor |
| F-ac356e5603c7 | Second service accessor |
| F-c7ba4c901230 | Service/object check |
| F-8f9bd59e3ec5 | Operation following a zero check result |
| F-78801ca166f2 | Receives the four-word area before explicit writes |
| F-8fa83bbc2366 | Receives service result, saved context, and four words |
| F-187151a4b5ba | Object operation with branch-irrelevant result |
| F-6f3e0e96361a | Forced action or action 1 |
| F-16f0e29b138e | Action 2 |
| F-6b95b88ebbda | Action 3 |
| F-f13086bc6f2a | Action 4 |
| F-4bb453fa7166 | Action 5 |
| F-87be4aa684d1 | Secondary gate-object accessor |
| F-6cf78263cbb0 | Receives service result and saved context |
| F-6904540d62b2 | Two-part secondary-selection result |

No direct recursive call is visible.

## Acceptance examples

These are conditional static scenarios, not executed tests. Each assumes valid reached storage and returning dependencies. Preservation assumptions apply only where explicitly stated.

1. **First-boot priority.** A nonzero lower result half from F-c49a155bf927 causes byte 199 to be set before construction. The offset-284 method receives the constructor result; cleanup receives the buffer address. No later navigation checks or selection comparisons occur.

2. **Distinct target reads.** If the saved comparison-target read yields one value and the immediately following read yields another, the first comparison uses the latter. If it fails, the second comparison uses the former.

3. **Distinct construction results.** If a constructor returns an address different from its buffer, the first candidate comparison and notification methods receive the returned value. Later candidate comparisons and all explicit construction cleanup calls receive the relevant buffer address.

4. **Third candidate match.** Zero artist and album results followed by a nonzero compilation result construct three candidates. Cleanup order is compilation, album, artist. `StartGenius` is not constructed.

5. **Selection-word order.** With saved context `0x00800000`, the post-F-78801ca166f2 writes are word 1 = `0x00800001`, then words 0, 2, and 3 = zero. The later call receives the words in order 0, 1, 2, 3.

6. **Pending action 4.** With ordinary activation reached, zero mode, byte 199 still zero after F-187151a4b5ba, and converted action 4, byte 176 is cleared before F-f13086bc6f2a runs. This branch does not explicitly clear byte 199.

7. **Forced activation.** With ordinary activation reached and nonzero mode, the pending-action word is not read. Bytes 176 and 199 are cleared, in that order, before F-6f3e0e96361a.

8. **Late secondary gate.** An action dependency may set byte 198 nonzero. If the subsequent offset-420 method returns zero, no offset-288 notification occurs; normal completion still clears byte 198.

9. **Repeated root-field read.** If the first offset-1016 read is nonzero but the second yields zero, F-753f887b468d receives zero as that field argument. The earlier nonzero test does not force reuse of its value.

10. **Notification load order.** In each selected secondary branch, the offset-288 method pointer is loaded before its corresponding notification global is read.

## Uncertainties

The following remain **documented_only**:

- Exact dependency prototypes, calling convention, and the extra register-carried argument’s origin.
- Constructor-result aliasing, allocation, auxiliary-storage use, retention, and cleanup semantics.
- Actual text at the one-byte-displaced selection-notification address.
- Contents and meanings of the three opaque notification globals.
- Character signedness and exact machine-level pending-action conversion; byte order is not independently established.
- Possible overlapping global storage, as indicated by the supplied representation.
- Dependency mutation, ownership, failure, exception, and unwinding behavior.
- Concurrency, synchronization, reentrancy, and observability of exposed temporary storage.
- Whether inferred activation, navigation, selection, and service roles match the wider application.

Repeated reads and repeated accessor calls remain distinct operations; unknowns do not justify collapsing them.

## Evidence

The supplied static function representation and opaque dependency mapping are the sole evidence. They establish the visible widths, offsets, conditions, argument forms, reads, writes, and call ordering described above. Role names are inferred intent, not verified dependency contracts.

No implementation access, dependency inspection, runtime execution, or tests have been performed. Acceptance examples are conditional consequences of the static representation. No provenance hash is asserted.
