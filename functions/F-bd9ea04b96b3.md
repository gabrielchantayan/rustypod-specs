<!-- Trusted provenance: F-bd9ea04b96b3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bd9ea04b96b3

## Purpose

F-bd9ea04b96b3 updates shared numeric state and returns a 32-bit result derived from that state and the caller’s input. Before the state update, it may populate a separate shared payload through an initialization protocol.

The contract includes observable ordering, not only the returned number:

- Selected initialization writes precede state processing.
- Two state-derivation dependencies run before the new state is committed.
- Input conversion and result combination occur after that commit.
- Certain result representations select an unresolved indirect transfer.
- Shared storage is consulted at several stages without an established snapshot or synchronization guarantee.

**Inferred intent:** the state recurrence resembles a pseudorandom generator, and the result classification resembles conversion from a binary32 representation to an unsigned integer. Neither interpretation establishes the semantics of the opaque dependencies.

## Interface

### Input and output

The function accepts one 32-bit word by value. The visible body neither dereferences it nor directly classifies its bits. It passes the original word unchanged to F-00b046cf541b after committing the state update.

The input’s semantic type is unknown. It is not established to be an integer, floating-point representation, pointer, or handle.

Normal completion returns an unsigned 32-bit word. Local paths can produce a shifted conversion result, zero, or `0xffffffff`—4,294,967,295. Exceptional paths forward the word attributed to an unresolved indirect transfer if that transfer returns.

No separate status output or structured error result is visible.

### Shared storage

| Logical object | Established access |
|---|---|
| Initialization object | A 32-bit flag at byte offset 4 and a 32-bit payload at byte offset 8 |
| Initialization locator | Supplies the initialization-object address |
| State locator | Supplies an address used to access a 32-bit state cell |
| Recurrence parameter | One 32-bit argument to F-870c4fcf7295 |
| Recurrence multiplier | One 32-bit arithmetic operand read after F-870c4fcf7295 returns |
| Exception control | A 32-bit word consulted repeatedly during exceptional dispatch |
| Exceptional data | Three distinct 32-bit stored words, selected by particular exceptional paths |

The first four bytes of the initialization object have no established direct access here. The payload’s type beyond its width, lifetime, and ownership are unknown.

All word arithmetic below concerns 32-bit quantities unless explicitly identified as 64-bit. Hexadecimal notation denotes a bit pattern rather than an assumed signed type.

### Preconditions and ownership

Storage must be accessible for every read or write selected by execution, including reads whose values are later discarded. The function performs no visible address or range validation.

No explicit allocation, release, or ownership transfer appears in the visible body. That observation does not establish that dependencies allocate nothing, release nothing, or preserve memory supplied through shared state.

Dependency-specific preconditions and permissible aliasing are unresolved. There is no established protection against concurrent mutation or reentrant access.

## Behavior

### Initialization and retained destination

An initialization-object address is retained before the flag decision. Flag bit 0 controls whether initialization is attempted:

- A set bit bypasses the initialization dependencies and the direct payload write.
- A clear bit permits F-b255b26284e6 to be called with a flag address.
- A zero return from that dependency bypasses the provider, payload write, and completion dependency.
- A nonzero return selects an initialization provider. Its returned 32-bit word is written at payload offset 8 of the retained object. F-d08d3ca17b81 then receives that retained object’s flag address.

The payload write precedes F-d08d3ca17b81. Changing the initialization locator during a dependency does not redirect that write or the subsequent completion call.

The visible body does not itself set the flag. A nonzero guard result establishes branch selection only; it does not establish successful locking, exclusive initialization rights, or a particular once-only protocol.

The initial address retention, flag access, and guard argument are represented as separate locator uses. Their equality is not guaranteed under concurrent locator mutation. With stable storage, they refer to the same object.

Every initialization path whose dependencies return normally proceeds to state processing.

### Recurrence and correction

Before F-870c4fcf7295 is called, the function retains a state-cell address. That dependency receives a current state word and the recurrence parameter and returns a 64-bit value.

Both halves of that return affect the next state. The high 32-bit half is multiplied by the shared recurrence multiplier; the low half is multiplied by −2,836. The two products are added. The halves enter signed 32-bit arithmetic in the supplied representation.

The multiplier is read after F-870c4fcf7295 returns. A dependency change to that storage can therefore affect the recurrence. The mathematical meanings of the two returned halves are unknown; quotient and remainder semantics must not be assumed.

The preliminary result receives a single signed correction:

- A strictly positive result is retained.
- Zero or a negative result receives one addition of 2,147,483,647.

There is no repeated correction or range validation. Even under wrapping 32-bit arithmetic, positivity is not an unconditional postcondition: a preliminary word of `0x80000000` becomes `0xffffffff`, which remains negative when interpreted as signed.

The supplied representation contains signed arithmetic that can overflow for unrestricted inputs. It does not establish the executable target’s overflow behavior. Statements that depend on low-word wrapping are conditional, not portable signed-arithmetic guarantees.

### Derivation and state commit

The corrected recurrence word is supplied to F-1c7710595b35. Its returned word is passed to F-f2b8ff34d9f0 together with `0xffffffe1`. That constant represents −31 under signed 32-bit interpretation; its meaning to the dependency is unknown.

Only after both calls return does the function write the corrected recurrence word to the retained state-cell address. The committed value is neither dependency’s result.

This timing has several consequences:

- A derivation dependency’s write to the retained state cell can be overwritten by the commit.
- A change to the shared state locator does not redirect the retained commit destination.
- No fresh state read reconciles intervening changes before that commit.
- Failure to return from either derivation dependency prevents the visible commit from being reached.

The retained destination and the address used to obtain the dependency’s state argument are represented separately. Without a stability guarantee, the contract does not require them to remain identical under concurrent locator changes.

### Input combination

After the state commit, F-00b046cf541b receives the original caller word. F-fdb31af1de99 then receives two words in this order:

1. The result of F-00b046cf541b.
2. The earlier result of F-f2b8ff34d9f0.

The returned word is the **result representation** classified below. No further direct state-cell write is visible.

### Local result classification

Three bit-level quantities describe the classification:

- The **classification index** is the result of a signed, sign-extending right shift of the result representation by 23 positions, subsequently held in an unsigned 32-bit word.
- The **conversion candidate** is the low 32 bits of the representation shifted left by eight positions, with its top bit additionally set when the classification index is nonzero.
- The **doubled representation** is the low 32 bits of twice the result representation.

The classification index is not an unsigned eight-bit exponent extraction. A negative representation produces an index whose signed interpretation is negative.

| Applicable condition | Established outcome |
|---|---|
| Negative classification index and doubled representation unsigned-less-than `0x7f000000` | Return zero |
| Nonnegative classification index below 159 | Return the conversion candidate after a logical right shift |
| Neither condition above | Apply the later saturation or exceptional classification |

For the direct shift return, the count is the low eight bits of the difference between 158 and the classification index. The selected index range makes the possible counts zero through 158.

Counts of 32 or more remain **documented_only**. The eight-bit count restriction does not establish that the executable reduces counts modulo 32, nor does it make these shifts valid portable operations on 32-bit words.

For paths reaching the later classification:

- A doubled representation no greater than `0xff000000` produces `0xffffffff` for a nonnegative original representation and zero for a negative original representation.
- A doubled representation greater than `0xff000000` selects exceptional dispatch.

Equality at `0xff000000` has the same sign-dependent output as the rest of that saturation range. It does not introduce a distinct numeric outcome.

These rules specify representation-level behavior. They do not establish a floating-point rounding mode or the numeric operation performed by F-fdb31af1de99.

### Exceptional dispatch

Exceptional dispatch first selects a mode using the low four bits of the exception-control word. Subsequent flag tests and argument construction consult that storage again. A single consistent control snapshot is not established.

The transfer destination is derived from incoming return-context bits by clearing the lowest two bits. The available evidence does not establish its identity, complete interface, or whether it behaves as an ordinary returning call.

The following table specifies only explicitly prepared argument words:

| Initially selected mode | Later condition | Prepared words, in order |
|---|---|---|
| 9 | Control bit 20 clear | Zero |
| 9 | Control bit 20 set | 8; a control-derived word described below |
| 8 | Control bit 6 set | Zero; zero |
| 8 | Bit 6 clear, then bit 4 clear | First exceptional stored word; doubled representation |
| 8 | Bit 6 clear, then bit 4 set | Zero; second exceptional stored word |
| 10 | Control bit 6 clear | Third exceptional stored word |
| 10 | Control bit 6 set | `0x80000000` |
| Any other mode | No further visible selection | No explicit argument words established |

For mode 9 with bit 20 set, the second word is formed from a later control read: shift left by 15 positions, complement the resulting 32-bit word, then shift left once more, retaining 32 bits. The complement occurs between the shifts.

Two reads remain observable even when their values are replaced:

- In mode 8 with bit 6 clear, the first exceptional stored word is read before bit 4 is tested. If bit 4 is set, that value is discarded and the second exceptional stored word supplies the second argument.
- In mode 10, the third exceptional stored word is read before bit 6 is tested. A set bit replaces its value with `0x80000000`.

If the transfer returns a word, that word is forwarded without further local normalization. The table does not establish that unspecified argument positions are absent, zero, or preserved.

## Side effects

The visible body has two explicit write destinations:

1. The initialization payload, only when the initialization guard returns nonzero.
2. The retained state cell, once execution reaches the commit after both state-derivation calls.

The state commit precedes input conversion, result combination, and exceptional dispatch. No rollback is visible if later processing fails, transfers control elsewhere, or never returns.

Dependencies and the unresolved transfer may produce additional effects. Their memory ownership, allocation, release, global preservation, error behavior, and synchronization are unknown.

The state operation is not established to be atomic. A possible initialization protocol does not establish protection for the later state read, recurrence, and commit.

Zero and `0xffffffff` are ordinary results on established local paths. Neither is a universal error indicator.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-b255b26284e6 | Receives a flag address; a zero or nonzero result controls initialization |
| F-d08d3ca17b81 | Receives the retained object’s flag address after the selected payload write; no returned value is consumed |
| F-870c4fcf7295 | Receives two 32-bit words; returns 64 bits consumed as two halves |
| F-1c7710595b35 | Receives the corrected recurrence word; returns a 32-bit word |
| F-f2b8ff34d9f0 | Receives that word and `0xffffffe1`; returns a 32-bit word |
| F-00b046cf541b | Receives the original caller word; returns a 32-bit word |
| F-fdb31af1de99 | Receives the input-derived word followed by the state-derived word; returns the result representation |

The initialization provider has no supplied opaque function ID. Its identity and semantics beyond the consumed return word remain documented_only.

The exceptional transfer likewise lacks an established opaque dependency identity and complete calling contract. No dependency is assumed to preserve shared storage or own or release memory.

## Acceptance examples

These examples are conditional consequences of static evidence, not runtime tests. Referenced storage is assumed accessible and dependencies normally returning unless stated otherwise.

1. **Initialization bypass:** With flag bit 0 set at the flag read, no initialization dependency or direct payload write occurs. State processing still proceeds.

2. **Initialization declined:** With flag bit 0 clear and F-b255b26284e6 returning zero, the provider and F-d08d3ca17b81 are not invoked.

3. **Retained initialization destination:** After a nonzero guard result, a provider change to the initialization locator does not redirect the payload write or completion call. Both use the retained object.

4. **Post-call multiplier:** If F-870c4fcf7295 changes the multiplier, recurrence arithmetic uses the subsequently read multiplier rather than an assumed entry value.

5. **Zero correction:** A preliminary recurrence result of zero becomes 2,147,483,647. This corrected word is passed to F-1c7710595b35 and later committed.

6. **Wrapping limitation:** Under wrapping 32-bit arithmetic, preliminary `0x80000000` becomes `0xffffffff`. No second correction follows.

7. **Commit ordering:** If a derivation dependency writes a different value to the retained state cell and returns normally, the later commit overwrites it with the corrected recurrence word.

8. **Defined conversion:** Representation `0x3fc00000` produces classification index 127, conversion candidate `0xc0000000`, and shift count 31. The local return is 1.

9. **Upper direct boundary:** Representation `0x4f000000` produces index 158 and shift count zero, returning `0x80000000`. Representation `0x4f800000` instead reaches saturation and returns `0xffffffff`.

10. **Oversized shift unresolved:** Representation zero selects count 158. No returned value is assigned here without executable shift semantics.

11. **Negative early result:** Representation `0xbf000000` doubles to `0x7e000000` and returns zero through the negative early-return condition.

12. **Saturation equality:** Representations `0x7f800000` and `0xff800000` both double to `0xff000000`. Their returns are respectively `0xffffffff` and zero.

13. **Discarded read still required:** With representation `0x7fc00000`, mode 10 selected, and bit 6 subsequently set, the third exceptional stored word is read before `0x80000000` replaces it. The transfer outcome remains unresolved.

14. **No later rollback:** If input conversion does not return, the earlier state commit has already occurred. The visible body provides no compensating write.

## Uncertainties

The following remain **documented_only**:

- Dependency semantics, failures, ownership, and global preservation.
- Meanings of the two halves returned by F-870c4fcf7295.
- Runtime recurrence parameters and supported ranges.
- Executable signed-overflow, signed-shift, and oversized-shift behavior.
- Intended numeric interpretation of the result representation.
- Initialization-provider identity and payload lifetime.
- Aliasing, concurrent mutation, reentrancy, and synchronization guarantees.
- Exceptional-transfer identity, complete arguments, calling convention, and return behavior.
- Behavior not reliably represented around unresolved indirect control flow.

These limits prevent unconditional promises about randomness, portable floating-point conversion, thread safety, resource ownership, or exceptional recovery.

## Evidence

Established static evidence consists of the supplied function representation and opaque dependency mapping: access widths and offsets, retained destinations, explicit calls, argument order, constants, branch conditions, write timing, and local return expressions.

The pseudorandom interpretation and binary32 interpretation are inferred intent only. Exceptional behavior is supported only to the extent of visible mode selection, reads, argument preparation, transfer-target formation, and return forwarding.

No runtime tests have been performed. Dependency implementations, executable instructions, and observed runtime state were not supplied for independent validation.
