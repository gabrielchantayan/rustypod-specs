<!-- Trusted provenance: F-8f75ffdb841a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8f75ffdb841a

## Purpose

F-8f75ffdb841a conditionally classifies an input and dispatches one of two recognized results to operations on a receiver. Its Boolean result indicates recognition of classification 6 or 7, not independently verified success of the invoked operations.

On every normally completing path, a non-null input receives a final virtual invocation. This invocation is consistent with releasing a reference, but its ownership semantics are not established.

The apparent intent is classification-driven processing with conditional receiver-state capture. Those interpretations do not establish the meanings of the classifications, fields, or dependency operations.

## Interface

### Inputs and result

The function receives two pointers:

- **Receiver:** used by the classification-specific operations and, for classification 7, by direct memory accesses.
- **Input:** passed to inspection and lookup dependencies, then used for final virtual dispatch if non-null.

The visible representation uses 32-bit integer words and 32-bit object addresses. Offsets below are byte offsets. They describe storage accessed by this function, not complete object layouts.

On normal completion, the result is:

- **True** if inspection returns exactly 1 and classification returns exactly 6 or 7.
- **False** otherwise.

This result does not incorporate a separate success indication from the classification-specific operations or the final input operation. No exception handler, fault recovery, or alternative error result is visible.

### Receiver storage

| Byte offset | Access | Role established by the function |
|---|---|---|
| `0x00` | 32-bit read | Dispatch-table address for the conditional receiver virtual invocation |
| `0x40` | 32-bit reads | Sentinel-tested value; later read again for copying to `0x4C` |
| `0x48` | 32-bit read | Value copied to `0x54` |
| `0x4C` | 32-bit write | Receives the later value read from `0x40` |
| `0x50` | 32-bit write | Receives the receiver virtual invocation’s result |
| `0x54` | 32-bit write | Receives the value read from `0x48` |
| `0x58` | 8-bit write | Receives 1 |
| `0x80` | Address formation | Receiver-relative address passed to F-80a6c6456406 |

The receiver virtual target is obtained from byte offset `0x9C` in the table addressed by the receiver’s first word. It receives the receiver pointer and supplies a 32-bit result.

For final input processing, the input’s first word supplies a dispatch-table address. The target at table byte offset `0x04` receives the input pointer.

### Global inputs

Two distinct global locations provide the lookup’s second and third arguments:

- **Global A** supplies the second argument.
- **Global B** supplies the third argument.

These labels identify distinct locations, not semantic names. Their contents, meanings, declared types, and mutability are not supplied. The exact load widths cannot be independently established without their declarations.

Both values are read on the authorized lookup path, after F-de6064227850 returns 1 and before F-b3bed2dfc5ec is invoked. F-de6064227850 may therefore affect the values subsequently supplied. No relative evaluation order between the two global reads is established.

### Validity and lifetime requirements

The input is passed to F-de6064227850 before any null check. The final null guard therefore does not establish general null-input safety.

Receiver requirements depend on the selected path:

- When inspection fails or classification is unrecognized, this function makes no direct receiver access.
- Classification 6 passes the receiver to F-d98827850840.
- Classification 7 directly reads receiver storage and invokes additional receiver-related operations.

Any storage accessed later must remain valid across preceding calls. In particular, a non-null input must still support final virtual dispatch after classification-specific processing. No visible alias check, lifetime guard, or synchronization enforces these requirements.

## Behavior

### Inspection gate

F-de6064227850 receives the input pointer.

Only an inspection result exactly equal to the 32-bit integer value 1 authorizes lookup and classification. All other values, including zero, negative values, and positive values other than 1, bypass those operations.

When inspection does not authorize processing, the function performs no direct receiver access and proceeds to final input processing with a false result.

### Lookup and zero-result handling

After authorized inspection, the function obtains the values from Global A and Global B and invokes F-b3bed2dfc5ec with these argument positions:

1. Input pointer.
2. Value read from Global A.
3. Value read from Global B.
4. The 32-bit all-ones value, `0xFFFFFFFF`.

The globals are read after inspection, rather than captured before it. Their argument positions are fixed even though their relative read order is not established.

The fourth argument’s bit pattern is established. Whether its semantic meaning is signed −1, an unsigned maximum, or another sentinel is unknown.

The lookup returns a 32-bit value:

- A nonzero result is passed to F-118693b436a8.
- A zero result causes an additional helper invocation with no visible arguments. If that helper returns normally, F-118693b436a8 receives zero.

The helper’s return value, if any, is not used. Its purpose and normal-return behavior are unknown. A zero lookup result does not itself require a false result or an immediate return.

For dependency identification within this contract, that unresolved helper is labeled **F-U01**. This is a contract-local opaque label, not an established implementation identity.

### Classification 6

If F-118693b436a8 returns exactly 6, F-d98827850840 receives the receiver pointer.

There are no explicit receiver-field writes in this branch. Any changes made by F-d98827850840 remain dependency effects.

The function then performs final input processing and returns true if execution completes normally. F-d98827850840 has no result used to determine that Boolean.

### Classification 7

If F-118693b436a8 returns exactly 7, the function first reads the 32-bit receiver field at `0x40`.

If that value is not the all-ones 32-bit representation of −1, the following operations occur in order:

1. Read the receiver’s dispatch-table address and obtain its target at table byte offset `0x9C`.
2. Invoke that target with the receiver pointer.
3. Write its 32-bit result to receiver offset `0x50`.
4. Read the 32-bit value at receiver offset `0x48` and write it to `0x54`.
5. Read the 32-bit value at receiver offset `0x40` again and write it to `0x4C`.
6. Write the one-byte value 1 to receiver offset `0x58`.

The receiver virtual target is labeled **F-U02**, another contract-local opaque label.

The field copies occur after F-U02 returns. Neither copied value is necessarily the value present before that call. In particular, the value written to `0x4C` may differ from the value that passed the sentinel comparison.

The write at `0x58` affects one byte only. It does not explicitly overwrite the next three bytes.

If the initial field value at `0x40` is −1, the function skips F-U02 and all four conditional writes. It does not explicitly reset the flag or refresh the destination fields on that subpath.

Both sentinel outcomes then converge on this sequence:

1. Invoke F-8dd08d4a2463 with the receiver pointer.
2. Invoke F-80a6c6456406 with the address formed by adding `0x80` bytes to the receiver.
3. Invoke F-ad00a5c3b7dd with the preceding dependency’s 32-bit result as its first argument and integer 7 as its second argument.

The address at receiver offset `0x80` is supplied directly. The function does not load a pointer from that location before the call.

After these operations, final input processing occurs. Normal completion returns true. The results of the class-7 operations are not checked for success; the result of F-80a6c6456406 is used only as the next call’s argument.

### Other classifications

Any classification other than exactly 6 or 7 causes no class-specific receiver operation. Final input processing follows, and normal completion returns false.

### Final input processing

After the selected processing path completes, the function checks the original input pointer.

For a non-null input, it:

1. Reads the input’s dispatch-table address.
2. Obtains the target at table byte offset `0x04`.
3. Invokes that target with the input pointer.

This target is labeled **F-U03**, a contract-local opaque label. Its result, if any, does not determine the function’s Boolean result.

For a null input, this invocation is skipped.

Final input processing is reached after every visible normally completing branch, including failed inspection, unrecognized classification, classification 6, either classification-7 sentinel outcome, and a zero lookup whose subsequent calls return normally.

There is no visible guarantee that final input processing occurs after a fault, exception, termination, or other non-normal departure from a preceding operation.

## Side effects

The function’s explicit receiver writes occur only for classification 7 when the initial field at `0x40` is not −1:

1. A 32-bit write to `0x50`.
2. A 32-bit write to `0x54`.
3. A 32-bit write to `0x4C`.
4. An 8-bit write to `0x58`.

These writes are ordered but are not an atomic transaction. No rollback is visible if a later access or call fails.

The function directly reads Global A and Global B only on the authorized lookup path. No direct write to either global is shown. Dependencies may nevertheless modify those locations or other global state; no preservation guarantee is available.

Every dependency and virtual target may have additional effects on receiver storage, input storage, resources, globals, or object lifetimes. Consequently, the explicit stores do not establish final receiver contents after later calls.

F-U03 is release-like only by inference. Its invocation does not establish destruction, consumption of exactly one reference, or any other ownership transfer. No assumption is made that earlier dependencies retain or release memory.

No direct allocation, explicit retain, loop, or retry is visible. The direct state operations are comparisons and fixed-width copies. Receiver-relative addresses use fixed byte displacements; valid, suitably addressable storage is required, and no address-overflow check is visible.

## Dependencies

| Opaque ID | Established call-site behavior |
|---|---|
| F-de6064227850 | Receives the input. Exactly 1 authorizes subsequent global reads, lookup, and classification. |
| F-b3bed2dfc5ec | Receives the input, Global A’s value, Global B’s value, and `0xFFFFFFFF`, in that argument order. Produces the 32-bit lookup result. |
| F-U01 | Invoked without visible arguments when lookup returns zero. If it returns normally, classification receives zero. No return value is used. |
| F-118693b436a8 | Receives the lookup value, or zero after F-U01. Produces the classification compared with 6 and 7. |
| F-d98827850840 | Receives the receiver for classification 6. No result is used. |
| F-U02 | Receiver virtual target at table byte offset `0x9C`; conditionally invoked for classification 7. Its 32-bit result is stored at `0x50`. |
| F-8dd08d4a2463 | Receives the receiver after any conditional class-7 writes. No result is used. |
| F-80a6c6456406 | Receives the receiver-relative address at `0x80`. Its 32-bit result supplies the next call’s first argument. |
| F-ad00a5c3b7dd | Receives that result and integer 7. No result is used. |
| F-U03 | Input virtual target at table byte offset `0x04`; invoked at final processing only for a non-null input. No result is used. |

The three contract-local labels identify call roles only. No additional implementation identity or dependency guarantee is implied.

## Acceptance examples

These are conditional deductions from static evidence, not executed tests. Each assumes normal completion and validity of every storage access actually required.

1. **Inspection rejects processing.**  
   With a non-null input and inspection result 2, neither global is directly read for lookup, and neither lookup nor classification is invoked. F-U03 runs, and the result is false. Inspection results 0 and −1 select the same path.

2. **Global values are obtained after inspection.**  
   Suppose inspection returns 1 after changing Global A and Global B. If those values remain unchanged until their respective reads, the lookup receives the changed Global A value as argument two and the changed Global B value as argument three. This example imposes no relative order on the two reads.

3. **Recognized classification 6 with non-null input.**  
   Given a non-null input, inspection result 1, nonzero lookup result, and classification 6, F-d98827850840 receives the receiver. Class-7 operations do not occur. F-U03 follows, and the result is true.

4. **Recognized classification 6 with accepted null input.**  
   If the input is null, preceding dependencies accept it, inspection returns 1, and classification returns 6, F-d98827850840 still runs. Final virtual invocation is skipped, and normal completion returns true. This does not establish general null safety.

5. **Classification 7 captures post-invocation values.**  
   Suppose the initial field at `0x40` is 12. F-U02 returns 90 and leaves `0x48` equal to 23 and `0x40` equal to 14. Immediately after the conditional stores, `0x50` contains 90, `0x54` contains 23, `0x4C` contains 14, and the byte at `0x58` contains 1. Later calls may change those values. The common class-7 sequence follows; F-U03 occurs only if the input is non-null. Normal completion returns true.

6. **Classification 7 skips conditional capture.**  
   With an initial all-ones value at `0x40`, F-U02 and the four conditional stores are skipped. F-8dd08d4a2463, F-80a6c6456406, and F-ad00a5c3b7dd still run in that order. Final processing remains conditional on input non-nullness. Normal completion returns true.

7. **Zero lookup can still produce a recognized result.**  
   A zero lookup invokes F-U01. If it returns and classification of zero yields 7, class-7 processing occurs. Zero lookup alone is not sufficient to require false.

8. **Unrecognized classification.**  
   Classification 8 invokes neither recognized branch. A non-null input still receives F-U03. Normal completion returns false.

9. **Null input rejected by inspection without further processing.**  
   If inspection accepts a null pointer and returns a value other than 1, lookup and classification are skipped. No final virtual invocation occurs, and the result is false.

## Uncertainties

**Established static behavior:** exact branch comparisons, lookup argument positions, the two distinct global inputs and their placement after inspection, receiver access widths and offsets, conditional write order, post-virtual-invocation field reads, and final non-null input dispatch.

**Inferred intent:** classification-based processing, saved-state capture, and release-like final input handling. These interpretations add no dependency obligations.

**documented_only unknowns:**

- Global A and Global B’s values, meanings, declared types, exact load widths, and mutability.
- Relative evaluation order of the two global reads.
- Lookup-result type and ownership beyond its observed 32-bit use.
- F-U01’s identity, effects, and normal-return behavior.
- Semantic meanings of receiver fields and classifications 6 and 7.
- Dependency error signaling, exceptions, reentrancy, and global preservation.
- Ownership effects of all calls, including whether either virtual target retains or releases objects.
- Safe receiver/input aliasing conditions.
- Thread-safety and external synchronization requirements.
- Boolean ABI representation beyond logical true and false.
- Additional validity requirements imposed by dependencies.

## Evidence

This contract is based on the supplied static function representation and dependency-ID mapping. It distinguishes visible accesses and control flow from interpretations of purpose and unresolved dependency behavior.

The static evidence establishes that the lookup’s second and third arguments come from distinct global locations on the path authorized by inspection. It does not establish their relative read order or preservation by inspection.

No runtime tests have been performed. Acceptance examples describe conditional outcomes, not observed executions.
