<!-- Trusted provenance: F-3cbde1b148f3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3cbde1b148f3

## Purpose

F-3cbde1b148f3 conditionally raises a caller-supplied two-word unsigned value to a computed candidate. Two preliminary checks and a preparation operation gate the update. When those gates permit comparison, the candidate combines a working low word, a supplied high word, and a 32-bit contribution obtained from another dependency.

The comparison is a strict unsigned increase: the destination is written only when the candidate is greater than its current value. Equality does not cause a write.

The returned value is a dependency status, not an indication that the destination changed. In particular, a zero return can accompany either a write or several different no-write outcomes.

The apparent intent is to maintain a monotonic maximum of a position-like quantity. That interpretation is inferred; the units, application meaning, and dependency semantics are not established.

## Interface

The interface has four inputs and a 32-bit unsigned return value:

| Input | Representation | Role |
|---|---|---|
| Context | Address represented by a 32-bit integer in the supplied interface | Provides a byte at offset 5 and a four-byte dependency argument at offset 8. |
| Destination | Pointer to two consecutive 32-bit unsigned words | Holds the value that may be increased. The first word is the low word; the second is the high word. |
| Initial low word | 32-bit unsigned integer | Initializes a working low word whose address is passed to a dependency. |
| Initial high word | 32-bit signed integer | Initializes a saved high word. Candidate arithmetic subsequently uses its 32-bit unsigned representation. |

The destination’s logical unsigned value is:

$D = D_{\mathrm{low}} + 2^{32}D_{\mathrm{high}}$.

This specifies word significance, not the byte order within each word. The supplied material does not independently establish byte endianness or alignment requirements.

The context has the following relevant layout:

| Offset from context | Width | Access |
|---|---:|---|
| 5 bytes | 8 bits | Conditionally read as an unsigned byte. |
| 8 bytes | 32 bits | Read before the first dependency call and read again before the preparation call, if that call is reached. |

Other context fields are not directly accessed by this function, though dependencies may access them.

The destination must contain two valid readable and writable words whenever the execution reaches the comparison and update path. The function performs no explicit pointer, bounds, alignment, or access-permission validation.

A working copy of the initial low word is passed by address to a dependency. The original scalar argument is not itself caller-owned output storage. No allocation, release, or ownership transfer is explicit in this function.

## Behavior

### Preliminary gating

The function initializes its working low word and saved high word from the two scalar inputs.

It reads the four-byte field at context offset 8 and passes that value to F-dc7894d9c188.

If F-dc7894d9c188 returns a nonzero 32-bit value:

- That value is returned.
- The context byte at offset 5 is not directly read.
- Neither remaining dependency is called.
- The function performs no direct destination access.

If F-dc7894d9c188 returns zero, the function reads the unsigned byte at context offset 5.

If that byte is nonzero:

- The function returns zero.
- Neither remaining dependency is called.
- The function performs no direct destination access.

Thus, the context byte is a gate, not the returned status. A nonzero byte suppresses candidate processing without producing a nonzero return.

These accesses are ordered around the first dependency call. If that dependency changes the context byte, the subsequent byte read observes the state available after the call, subject to the execution environment’s memory rules.

### Preparation

When both the first dependency result and the context byte are zero, the function reads the context field at offset 8 again. It passes that freshly read value and the address of the working low word to F-a042d20fe1ce.

The second context-field read is distinct from the first. The function does not require the two field values to match and does not explicitly retain the first value for reuse.

If F-a042d20fe1ce returns a nonzero 32-bit value:

- That value is returned.
- F-0d05d1544f78 is not called.
- The function performs no direct destination access.

If F-a042d20fe1ce returns zero, candidate processing proceeds. The working low word used afterward is its value following that call, not necessarily the initial low-word argument.

The supplied material does not define the preparation operation’s output range, the meaning of its statuses, or the full extent of memory it may modify.

### Candidate formation

After successful preparation, the function calls F-0d05d1544f78 with the context argument. Its 32-bit unsigned return is used as an arithmetic contribution, not as a success or failure status.

Let:

- $L$ be the working low word available after the dependencies;
- $A$ be the contribution returned by F-0d05d1544f78;
- $H$ be the 32-bit unsigned interpretation of the saved high word.

The candidate low word is:

$C_{\mathrm{low}} = (L + A) \bmod 2^{32}$.

The carry is one when the mathematical sum of the two unsigned 32-bit operands is at least $2^{32}$, and zero otherwise.

The candidate high word is:

$C_{\mathrm{high}} = (H + \mathrm{carry}) \bmod 2^{32}$.

The signed declaration of the high-word input does not make the comparison signed. A negative input contributes its corresponding 32-bit unsigned representation. For example, an initial high word of −1 contributes `0xFFFFFFFF` before any carry.

A carry out of the high word is discarded. There is no saturation, overflow error, or additional output word.

There is no explicit reassignment of the saved high word after initialization. The ordinary intended calculation therefore uses the supplied high word. This observation does not establish that unknown dependencies preserve all accessible storage.

### Comparison and update

After F-0d05d1544f78 returns, the function reads the destination high word. It compares the candidate and destination as unsigned two-word values, with the high word taking precedence.

The decision is:

| Relationship | Direct result |
|---|---|
| Candidate high word greater than destination high word | Write the candidate. |
| Candidate high word less than destination high word | Leave the destination unchanged. |
| High words equal and candidate low word greater than destination low word | Write the candidate. |
| High words equal and candidate low word less than or equal to destination low word | Leave the destination unchanged. |

When the high words differ, no direct destination-low read is needed by the shown decision path.

When the high words are equal, the source-level expression first reads the destination low word for a non-strict comparison. If that comparison permits continuation, it reads the low word again to exclude equality. Consequently, the supplied expression does not establish a single low-word snapshot under concurrent or reentrant modification. This is a source-level observation; exact physical load counts require lower-level execution evidence.

When an update is selected, the direct writes occur in this order:

1. Candidate low word to the first destination word.
2. Candidate high word to the second destination word.

There is no atomic two-word commit or synchronization operation in the supplied behavior. The write pair must not be treated as an indivisible 64-bit update.

Under stable inputs and destination storage, the resulting destination is the unsigned maximum of its previous value and the computed candidate. This monotonicity statement concerns the candidate after 64-bit modular wrap, not an unbounded mathematical sum.

### Return value

The return behavior is:

| Reached outcome | Returned value |
|---|---|
| F-dc7894d9c188 returns nonzero | That nonzero value |
| First dependency returns zero; context byte is nonzero | Zero |
| F-a042d20fe1ce returns nonzero | That nonzero value |
| Preparation succeeds; comparison is completed | Zero, whether or not a write occurs |

The contribution returned by F-0d05d1544f78 never becomes the function’s return status.

## Side effects

The function’s explicit effects are limited to:

- Initializing local working storage.
- Reading the described context fields.
- Calling dependencies in the gated order.
- Passing writable working-low storage to F-a042d20fe1ce.
- Conditionally reading and writing the destination.

“No direct destination access” on an early-return path does not mean “no possible destination effect.” A dependency could reach the destination through aliases, context state, or globals. Its behavior is not supplied.

Likewise, the function does not establish that dependencies preserve the context field at offset 8, the gate byte, global state, or memory reachable from their arguments. The repeated context-field read makes intervening changes relevant.

The function explicitly allocates no heap storage and explicitly frees no storage. It does not visibly transfer ownership of the context or destination. Dependency ownership behavior remains unknown. Any retention of the working-low address beyond its valid lifetime would require an external lifetime contract not present here.

No rollback is shown for dependency side effects. No recovery path is shown for invalid memory access or abnormal dependency termination.

## Dependencies

| Opaque ID | Inputs at this call site | Result used here | Established role |
|---|---|---|---|
| F-dc7894d9c188 | Four-byte context field read at offset 8 | 32-bit status | First gate; nonzero terminates processing. |
| F-a042d20fe1ce | Newly read four-byte context field and working-low address | 32-bit status | Preparation; zero permits candidate processing. |
| F-0d05d1544f78 | Context argument | 32-bit unsigned contribution | Supplies the amount added to the working low word. |

All three calls are conditional except the first, assuming normal entry and valid access to its argument field.

No dependency implementation is available. Their status meanings, effects on globals, memory ownership rules, pointer-retention behavior, and abnormal termination behavior are documented_only unknowns.

## Acceptance examples

These examples are conditional acceptance criteria derived from static behavior. They have not been executed. Unless stated otherwise, assume valid storage, normal dependency returns, no concurrent modification, and no dependency effects beyond the stated working-low update.

1. **First gate returns a status.**  
   F-dc7894d9c188 returns 7. The function returns 7 without directly reading the context gate byte or destination and without calling either later dependency.

2. **Context byte suppresses processing.**  
   The first dependency returns zero and the subsequently read byte at offset 5 is 1. The function returns zero without calling preparation or contribution dependencies and without directly accessing the destination. Any other nonzero byte has the same gating effect.

3. **Preparation returns a status.**  
   The first result and gate byte are zero; preparation returns 9. The function returns 9, does not call F-0d05d1544f78, and performs no direct destination access.

4. **Preparation changes the low word.**  
   The initial low word is 100. Preparation changes it to 20 and returns zero. With saved high word 2 and contribution 3, the candidate is high word 2, low word 23—not low word 103.

5. **Strict increase within one high word.**  
   Candidate high word is 2 and candidate low word is 23. A destination containing high word 2, low word 22 is updated to 2:23. A destination already containing 2:23 is not written. A destination containing 2:24 remains unchanged. Each path returns zero.

6. **High word dominates.**  
   Candidate 3:0 exceeds destination 2:`0xFFFFFFFF` and is written. Candidate 1:`0xFFFFFFFF` does not exceed destination 2:0 and is not written.

7. **Low-word carry.**  
   The working low word is `0xFFFFFFFE`, contribution is 5, and saved high word is 7. The candidate is high word 8, low word 3. Destination 8:2 is updated to 8:3.

8. **Full-width wrap is not saturation.**  
   The saved high word is −1, working low word is `0xFFFFFFFF`, and contribution is 1. The candidate wraps to 0:0. A destination containing 0:1 remains unchanged. A destination containing 0:0 also receives no write because equality is excluded.

9. **Context-field changes are observable.**  
   The first dependency receives a field value of 11, changes that context field to 12, and returns zero. If the gate byte then reads zero, preparation receives 12. The function does not require reuse of 11.

## Uncertainties

- **Inferred intent:** The operation resembles a guarded maximum update for an accumulated position. No semantic unit or domain meaning is established.
- **documented_only — dependency contracts:** Status interpretations, mutation scope, ownership, retained pointers, and termination behavior are unavailable.
- **documented_only — memory environment:** Byte endianness, alignment requirements, permitted aliasing, and invalid-access behavior are not established.
- **documented_only — working-storage extent:** Passing the working-low address establishes an address argument, not permission to access neighboring storage. The dependency’s permitted access extent is unknown.
- **documented_only — concurrency:** No synchronization is visible. A coherent destination snapshot and atomic update cannot be promised.
- **documented_only — physical access details:** The supplied expressions establish logical access ordering, but not exact machine-load counts, bus transactions, or compiler transformations.

## Evidence

The established behavior comes solely from the supplied static function representation: its four-argument interface, explicit field accesses, dependency calls, conditional branches, 32-bit carry operation, unsigned comparisons, ordered destination assignments, and returned status.

The dependency mapping supplies opaque identities only; it supplies no dependency behavior.

No runtime tests, dependency executions, memory traces, or machine-level access observations have been performed. The acceptance examples are derived criteria, not reported test results.
