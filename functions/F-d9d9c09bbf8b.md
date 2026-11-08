<!-- Trusted provenance: F-d9d9c09bbf8b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d9d9c09bbf8b

## Purpose

F-d9d9c09bbf8b handles selected operations on a primary object, subject to a shared setup sequence and an object-state gate. A locally completed operation writes one 32-bit result to caller-supplied storage and returns 1. A zero gate causes return 0 without a direct result write. Unrecognized operations are delegated to F-99c43bba3d43.

The gate is evaluated after a primary-object method has run. The comparison that controls auxiliary-object acquisition occurs before operation-specific handling, even for operations that do not use the auxiliary object.

“Setup,” “gate,” “comparison object,” and “auxiliary object” describe their roles in this contract. Their application meanings are not established. The behavior below is based on static evidence; no runtime tests have been performed.

## Interface

### Inputs and return value

| Input | Representation | Contractual role |
|---|---|---|
| Primary object | Pointer represented by 32 bits | Provides a dispatch table and object fields; forwarded to the fallback |
| Pass-through value | Opaque 32-bit value | Used only as an argument to the fallback |
| Operation selector | Signed 32-bit integer | Selects local handling or fallback |
| Result destination | Pointer represented by 32 bits | Identifies four bytes of writable storage for a locally produced result; forwarded to the fallback |

The function returns a 32-bit value. Its locally established return values are 0 for gate rejection and 1 for completed local handling. On fallback, it returns the dependency’s result without changing its bits.

The return value is separate from the stored result. A local operation can store zero and still return 1. Conversely, return 0 from the gate-rejection path does not represent a stored zero.

Signed selector and comparison values have the 32-bit range −2,147,483,648 through 2,147,483,647. Result-producing dependencies supply 32-bit bit patterns; this function does not impose a signed interpretation on those results.

### Accessed layouts

All offsets below are byte offsets. Pointer fields, integer fields, dispatch entries, and result storage occupy four bytes in the supplied representation.

| Storage | Offset | Required content when accessed |
|---|---:|---|
| Primary object | 0x00 | Dispatch-table pointer |
| Primary object | 0x18 | Signed comparison value |
| Primary object | 0x34 | Gate value, tested for zero |
| Primary dispatch table | 0xA0 | Entry for F-I01 |
| Comparison object | 0x104 | Signed comparison value |
| Auxiliary object | 0x00 | Dispatch-table pointer |
| Auxiliary dispatch table | 0x14 | Entry for F-I02 |
| Auxiliary dispatch table | 0x18–0x2C | Individual operation entries, as mapped below |
| Secondary object | 0x00 | Dispatch-table pointer |
| Secondary dispatch table | 0x58 | Entry for F-I09 |
| Control object | 0x04 | Flag word whose least-significant bit controls setup |

These are accessed portions of layouts, not complete object definitions. They do not establish inheritance, alignment, byte order, allocation size, or ownership.

### Global storage identities

The following opaque IDs identify separate global storage locations. Their values may coincide.

| ID | Role |
|---|---|
| G-01 | Pointer to the control object |
| G-02 | Argument source for the setup guard and final setup call |
| G-03 | Resource value used during setup and operation 0x8204 |
| G-04 | Source of argument 2 to F-804adc29bbfb |
| G-05 | Source of argument 3 to F-804adc29bbfb |

G-04 and G-05 are not interchangeable. Their identities are defined here by their distinct argument positions, not by any assumed relationship between their values.

## Behavior

### Shared setup

The initial setup decision depends only on bit zero of the control word reached through G-01. Other bits do not affect that decision.

When the bit is set, no setup dependency is called and no setup capture of G-03 occurs. When it is clear, F-b255b26284e6 receives the current G-02 value. After that dependency returns, G-03 is captured as a 32-bit value, including when the dependency returned zero.

A zero return from F-b255b26284e6 prevents the remaining setup calls. A nonzero return permits this ordered sequence:

| Dependency | Arguments and read timing |
|---|---|
| F-43799d085e81 | Receives a fresh G-03 read after the earlier capture |
| F-804adc29bbfb | Receives the captured G-03 value, then G-04 and G-05 values read after F-43799d085e81 returns |
| F-d08d3ca17b81 | Receives G-02 read after F-804adc29bbfb returns |

The relative order of the G-04 and G-05 reads is not established. Both follow F-43799d085e81; neither belongs to the earlier capture.

This distinction matters when dependencies mutate global state. F-43799d085e81 can change G-03 without changing the captured first argument to F-804adc29bbfb. Its changes to G-04 or G-05 can affect that call’s second or third argument. Changes to G-02 during the preceding setup calls can affect the argument subsequently supplied to F-d08d3ca17b81.

No direct update to the flag or these globals is established during setup. In particular, this function does not itself establish that setup runs only once.

### Primary-object call and gate

After the setup decision and any permitted setup calls, F-I01 is obtained from the primary dispatch table at offset 0xA0 and invoked with the primary-object pointer. Its return value is retained as the comparison-object pointer.

The gate at primary-object offset 0x34 is read only after F-I01 returns. Consequently, changes made by F-I01 can affect the gate decision.

A zero gate terminates local processing with return 0. There is then no comparison-object field access, auxiliary acquisition, operation-specific activity, fallback call, or direct result write. The comparison-object pointer need not be usable on that path because this function does not dereference it.

Gate rejection does not undo earlier effects. Setup dependencies and F-I01 have already run where applicable, and their effects are not constrained by this contract.

### Comparison and auxiliary acquisition

A nonzero gate requires a signed comparison between the primary field at offset 0x18 and the comparison-object field at offset 0x104. The relative order of these two reads is not established.

F-1bc3fa7742fb is called without explicit arguments exactly when the primary value is strictly less than the comparison-object value. Its return value supplies the auxiliary pointer. Equality and greater-than leave the auxiliary pointer null.

The comparison is signed, not unsigned. It is not implemented as a difference whose overflow would alter the relation. For example, a primary value of −1 is less than a comparison-object value of 0.

This stage precedes every selector outcome, including zero-result operation 0x307, resource operation 0x8204, and fallback. An operation that does not dereference the auxiliary pointer can still cause F-1bc3fa7742fb to run.

### Local operation results

For the following selectors, a single indirect dependency supplies the result:

| Selector | Auxiliary-table offset | Dependency |
|---|---:|---|
| 0x301 | 0x18 | F-I03 |
| 0x302 | 0x1C | F-I04 |
| 0x303 | 0x20 | F-I05 |
| 0x304 | 0x24 | F-I06 |
| 0x305 | 0x2C | F-I07 |
| 0x306 | 0x28 | F-I08 |

Each selected dependency has no explicit arguments in the supplied representation. Its 32-bit return value is stored at the result destination, after which F-d9d9c09bbf8b returns 1.

The mapping is intentionally nonmonotonic: 0x305 selects offset 0x2C, while 0x306 selects offset 0x28. These paths do not check the auxiliary pointer before accessing its dispatch table.

Operation 0x300 obtains its result through two dispatch roles. If the auxiliary pointer is nonnull, F-I02 receives that pointer and the 32-bit constant 0x13. Its return supplies a secondary-object pointer. F-I09, selected at secondary-table offset 0x58, is then invoked without explicit arguments. Its 32-bit return becomes the stored result, and the function returns 1.

A null auxiliary pointer skips F-I02 but leaves the secondary pointer null; the secondary dispatch access is still attempted. A null return from F-I02 likewise receives no null check. Neither situation is an established successful zero-result path.

Operation 0x307 directly stores the 32-bit value zero and returns 1. It does not dereference the auxiliary pointer, but all preceding shared behavior still applies.

Operation 0x8204 calls F-43799d085e81 with the current G-03 value. After that call returns, G-03 is read again, and this later value is stored as the result. The function then returns 1. The stored value need not match the dependency’s argument.

### Fallback

All selectors other than 0x300 through 0x307 and 0x8204 use F-99c43bba3d43. This includes negative selectors.

The fallback receives the four original input values in their original positions. Original pointer values are forwarded; the objects and storage they designate are not restored to their entry state.

F-d9d9c09bbf8b propagates the fallback’s 32-bit return value unchanged and performs no additional direct result write. The fallback’s own writes, errors, and other effects are not established here.

## Side effects

### Writes and ordering

Every normally completed local operation performs exactly one direct four-byte result write. For result-producing dependencies, that write follows their return. Gate rejection and fallback perform no direct result write by this function.

The established ordering is:

- The control flag and any setup activity precede F-I01.
- F-I01 precedes the gate read.
- A nonzero gate precedes the comparison and any auxiliary acquisition.
- Comparison and acquisition precede operation-specific handling or fallback.
- A local result write follows the calls required for that result.

This ordering does not promise that earlier values remain current. Dependencies may change globals, object fields, or dispatch tables. Each later access uses the state available at that access, except for values explicitly retained, such as the setup capture of G-03.

### Ownership and aliasing

No explicit allocation, deallocation, reference-count adjustment, or ownership transfer is established. The function does not directly assign to object fields or global storage except where the result destination aliases such storage.

No non-aliasing requirement is visible. If the result destination overlaps an object field or global, the final local write changes that storage. There are no subsequent object-field or global reads by this function after that write.

Aliasing does not excuse invalid accesses. All storage needed before the write must remain usable. Dependencies may themselves access or invalidate overlapping storage; their effects are unknown.

A returned pointer does not establish allocation, retained ownership, or permission for the caller to release it. No dependency is assumed to preserve globals or object lifetimes.

### Preconditions and failures

Normal completion requires valid storage and callable entries at every access actually taken:

- The control pointer and its flag word must be readable.
- The primary object and F-I01 dispatch entry are needed even if the gate will reject processing.
- A nonzero gate requires a readable comparison-object field for every selector.
- Operations 0x301–0x306 require a usable auxiliary dispatch path.
- Operation 0x300 requires usable auxiliary and secondary dispatch paths for normal completion.
- Every completed local operation requires writable four-byte result storage.

These conditions are not validated or converted into defined error returns. Invalid pointers, invalid indirect targets, dependency failures, and nonreturning dependencies have no established recovery behavior.

The constants, captures, result writes, and forwarded values fit within 32 bits. No result arithmetic, explicit widening, saturation, or truncation beyond the established 32-bit representation is shown. Offset-based addressing does not establish behavior for invalid or wrapping addresses.

## Dependencies

Direct dependencies have the following established interactions:

| Dependency | Interaction |
|---|---|
| F-b255b26284e6 | Receives G-02; zero versus nonzero controls the remaining setup calls |
| F-43799d085e81 | Receives G-03 during permitted setup and operation 0x8204; return value unused |
| F-804adc29bbfb | Receives captured G-03, later G-04, and later G-05, in that argument order; return value unused |
| F-d08d3ca17b81 | Receives refreshed G-02 during permitted setup; return value unused |
| F-1bc3fa7742fb | Has no explicit arguments; supplies the auxiliary pointer after a true comparison |
| F-99c43bba3d43 | Receives all four original inputs; supplies the fallback return |

F-I01 through F-I09 are opaque identifiers for indirect call roles, not claims that the underlying functions are distinct. F-I01 supplies the comparison pointer; F-I02 supplies the secondary pointer; F-I03 through F-I09 supply operation results.

“No explicit arguments” describes only the supplied representation. Complete machine-level calling conventions, implicit state, and additional register dependencies remain unresolved.

No dependency implementation is available to establish mutation, memory-management, error, or preservation guarantees.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless an example states otherwise, all required accesses are valid, dependencies return normally, and no unmentioned mutation changes the conditions.

1. **Gate rejection occurs after the primary call.** If F-I01 returns and the gate subsequently reads zero, the function returns 0 without a direct result write, comparison-field read, auxiliary acquisition, or fallback.

2. **Other flag bits do not suppress setup.** A control word of 2 has bit zero clear, so F-b255b26284e6 is called. A control word of 3 has bit zero set, so the setup calls are skipped.

3. **A rejected setup guard still permits the capture.** With the initial flag clear, a zero return from F-b255b26284e6 is followed by the G-03 capture. The other three setup dependencies are not called for that setup sequence.

4. **Setup retains its first resource value.** Suppose the setup capture reads G-03 as A, and F-43799d085e81 changes G-03 to B, G-04 to C, and G-05 to D. F-804adc29bbfb receives A, C, and D—not B, C, and D. No relative order between the reads of C and D is required.

5. **The final setup argument is refreshed.** If F-804adc29bbfb changes G-02 before returning, F-d08d3ca17b81 receives the later G-02 value.

6. **Signed ordering controls acquisition.** With a nonzero gate, primary −1 and comparison-object 0 cause F-1bc3fa7742fb to run. Equal values do not. These rules also apply to selector 0x307.

7. **The final two single-call mappings differ.** For 0x305, a return of 0xA1B2C3D4 from F-I07 at offset 0x2C is stored unchanged, and the function returns 1. Selector 0x306 instead selects F-I08 at offset 0x28.

8. **The two-stage result uses the secondary object.** For 0x300, F-I02 receives the auxiliary pointer and 0x13. If it returns a valid secondary object and F-I09 returns 7, the destination receives 7 and the function returns 1.

9. **Zero output does not require an auxiliary dereference.** For 0x307, a false comparison leaves the auxiliary pointer null, yet the destination receives zero and the function returns 1.

10. **The resource result is read after the call.** For 0x8204, if F-43799d085e81 receives A and changes G-03 to B, the destination receives B.

11. **Fallback preserves return bits.** For 0x308, F-99c43bba3d43 receives the original four inputs. A return of 0x80000002 is propagated unchanged, without an additional direct result write.

12. **Missing auxiliary storage is not a handled error.** For 0x304, a false comparison leads to an attempted null auxiliary dispatch access. For 0x300, it leads to an attempted null secondary dispatch access. Neither establishes return 0 or a stored zero.

## Uncertainties

The following remain **documented_only**:

- Application meanings of selectors, fields, globals, and results.
- Whether setup implements initialization, synchronization, registration, or another protocol.
- Dependency mutations, callbacks, errors, allocation, release, and ownership effects.
- Object lifetimes across calls and whether a dependency invalidates later accesses.
- Complete calling conventions and identities behind indirect call roles.
- Alignment, byte order, complete layouts, and platform-level address behavior.
- Concurrency, reentrancy, synchronization, and atomicity guarantees.
- Runtime consequences of invalid memory accesses or indirect calls.
- Fallback behavior beyond argument forwarding and return propagation.

Setup terminology is inferred intent, not evidence of a one-time protocol. No unresolved dependency behavior is replaced with an assumption of preservation or successful recovery.

## Evidence

The supplied static representation establishes the selector set, signed comparison, four-byte values and accesses, field and dispatch offsets, explicit arguments, conditional calls, direct result writes, and return propagation.

It also establishes distinct setup read times: G-03 is captured after F-b255b26284e6 even on a zero return; G-04 and G-05 are read after F-43799d085e81 for arguments 2 and 3; and the final setup argument is a later G-02 read. The relative evaluation order of G-04 and G-05 is unresolved.

No dependency implementations, independent layout definitions, or runtime traces were supplied. No runtime tests have been performed.
