<!-- Trusted provenance: F-358dc1f4e8cf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-358dc1f4e8cf

## Purpose

F-358dc1f4e8cf performs a short sequence of indirect operations involving a supplied object and another object referenced from within it. The recovered representation shows three indirect call sites:

1. An operation on the referenced object, using a derived global value and the constant 24,576.
2. An operation on the supplied object, using two global values, a function-pointer-like argument, and an unresolved incoming machine value.
3. A further operation on the supplied object, using an earlier captured global value and another global value.

The names, meanings, and concrete implementations of these operations are unknown. Describing them as initialization, configuration, notification, rendering, or cleanup would be inferred intent rather than established behavior.

The final transfer has an explicit recovery warning: it may represent control flow that was not reconstructed correctly. Accordingly, the three-operation sequence is the **recovered call model**, not proof of the original machine-level control-flow structure.

## Interface

### Explicit input

The function has one explicit input: a pointer to a mutable, word-addressed object.

The recovered layout uses 32-bit words. Under the represented 32-bit pointer model, the following locations are accessed:

| Location | Width | Interpretation |
|---|---:|---|
| Supplied object, byte offset 0 | 32 bits | Pointer-like value identifying an indirect-dispatch table |
| Supplied object, byte offset 888 (`0x378`) | 32 bits | Pointer-like reference to another object |
| Referenced object, byte offset 0 | 32 bits | Pointer-like value identifying its indirect-dispatch table |
| Referenced object's dispatch table, byte offset 212 (`0xD4`) | Pointer-sized, modeled as 32 bits | First indirect target |
| Supplied object's dispatch table, byte offset 88 (`0x58`) | Pointer-sized, modeled as 32 bits | Target used by the second and final recovered call sites |

The offset of 888 bytes follows from a word index of 222 multiplied by a four-byte word size. The dispatch offsets are byte offsets, not word indices.

These accesses establish only the layout needed by this function. They do not establish complete object sizes, inheritance relationships, table lengths, or object types.

### Global inputs

Four distinct global storage locations participate. This contract assigns them descriptive labels solely for discussion:

| Label | Observed use |
|---|---|
| G1 | Read to form an argument for the first indirect operation |
| G2 | Captured after the first operation; also read as an argument for the second operation |
| G3 | Read as another argument for the second operation |
| G4 | Read as an argument for the final operation |

The captured G2 value is explicitly represented as a 32-bit quantity. The recovered listing does not independently establish the declared types, signedness, volatility, or semantic units of all four globals.

### Additional machine-state input

The second recovered call includes a value originating from an incoming machine register, with no initialization shown in this function.

This is an unresolved input, not an established zero, null pointer, default value, or copy of another argument. Its validity and meaning depend on calling-convention and surrounding-code information that is unavailable.

### Output

The recovered interface has no explicit return value. Return values from the indirect operations are not visibly consumed.

Normal completion is represented by a return after the final indirect operation. Because that operation follows an unresolved indirect-control-flow warning, the exact original return or tail-transfer behavior remains uncertain.

### Preconditions

For the recovered call model to execute normally:

- The supplied pointer must support all required object reads.
- Its field at byte offset 888 must yield a usable referenced-object address.
- Each dispatch-table address and target required at the time of access must be valid.
- Each indirect target must accept the actual machine-level calling sequence.
- Storage read after an earlier indirect operation must remain accessible when subsequently used.

No null checks, bounds checks, dispatch validation, or lifetime checks are visible.

These are access requirements, not evidence that the function enforces them.

## Behavior

### 1. Operation on the referenced object

The function obtains the referenced object through the supplied object's field at byte offset 888. It uses that object's leading dispatch-table pointer and selects the target at table byte offset 212.

The first recovered call receives three arguments:

1. The referenced-object pointer.
2. A value formed from G1 by subtracting 102 (`0x66`).
3. The integer constant 24,576 (`0x6000`).

The subtraction is an arithmetic adjustment in the recovered representation. There is no explicit dereference of the adjusted result in this function.

The supplied evidence does not establish whether G1 is intended to denote an address, an integer, or an encoded value. It also does not settle signed-overflow or pointer-arithmetic semantics. An implementation must not add saturation, range checks, or a special interpretation for the result without additional evidence.

Evaluation order among the target lookup and individual argument expressions is not established. The referenced-object field occurs in both target construction and argument construction; the listing does not justify promising a single atomic field read or a cached pointer value under concurrent mutation.

### 2. Capture a global value

After the first indirect operation returns, the function captures G2 as a 32-bit value.

This ordering is significant. If the first dependency changes G2 before returning, the capture observes the post-operation value under ordinary sequential execution.

The captured value is later supplied to the final recovered operation. It is not represented as being refreshed after the second operation.

### 3. First operation on the supplied object

The function dispatches through the supplied object's leading table pointer, selecting the entry at byte offset 88.

The recovered call has five arguments:

1. The supplied-object pointer.
2. A read of G2.
3. A read of G3.
4. A function-pointer-like value obtained from the supplied object's dispatch-table entry at byte offset 88.
5. The unresolved incoming machine value.

The second argument is represented as a global read, whereas the later final operation uses the previously captured G2 value. In an ordinary sequential execution with no intervening mutation, these values agree. Their provenance nevertheless differs.

The fourth argument comes from the same object-and-offset lookup used to select this operation's target. It is therefore consistent with passing the operation's own target address as data. That interpretation is not guaranteed under concurrent mutation, and the purpose of the argument is unknown.

The representation does not establish evaluation order among the dispatch lookup and these arguments. It also does not prove the callee's true prototype. In particular, the apparent fifth argument and the unresolved incoming machine value require ABI evidence before they can be treated as a confirmed high-level parameter contract.

### 4. Final operation on the supplied object

Following the second operation, the recovered representation performs another lookup through the supplied object's dispatch-table entry at byte offset 88.

The final recovered call receives three arguments:

1. The supplied-object pointer.
2. The 32-bit G2 value captured before the second operation.
3. A read of G4.

The final target is not explicitly cached across the preceding operation. Consequently, if that operation changes the supplied object's dispatch-table pointer or the selected table entry, the final lookup can select a different target.

Likewise, G4 is used after the second operation. There is no earlier capture of G4 shown. Changes to G4 made by that operation can therefore affect the final call.

By contrast, changes to G2 made by the second operation do not replace the already captured value supplied to the final operation.

### 5. Completion and abnormal paths

If all recovered operations return normally, execution reaches the represented return.

There is no visible:

- Conditional branch based on an indirect operation's return value.
- Retry.
- Error translation.
- Rollback.
- Explicit recovery from invalid object or table access.
- Explicit release of resources after a dependency fails.

Faults, exceptions, nonlocal transfers, process termination, or non-returning dependency behavior can prevent subsequent operations. Their mechanics are not established here.

The final indirect transfer remains subject to the recovery limitation described under Purpose.

## Side effects

### Direct effects

The recovered body contains no explicit stores to the supplied object, the referenced object, or the global locations.

Its direct observable activity consists of memory reads, arithmetic used to prepare an argument, and transfers to indirect targets. The absence of explicit stores does not make the function pure: the dependencies may have arbitrary externally visible effects.

### Dependency effects and sequencing

No dependency is known to preserve:

- The supplied object's fields.
- The referenced object's fields.
- Dispatch tables or their entries.
- Any global value.
- Object lifetimes.
- External resources.

The ordering of completed calls matters. Effects of the first operation can influence the G2 capture and subsequent object dispatch. Effects of the second operation can influence the final dispatch and G4 read.

If a dependency invalidates storage that a later stage accesses, this function contains no visible protection against that invalidation.

### Ownership

The function performs no explicit allocation, reference-count adjustment, ownership transfer, or deallocation.

Passing a pointer to a dependency does not establish that the dependency borrows, retains, consumes, or releases it. The caller's ownership obligations and the dependencies' ownership behavior remain unknown.

### Concurrency

No synchronization, atomic access, or coherent snapshot mechanism is shown.

The sequential ordering above describes the recovered execution model. It does not establish safe behavior when another execution context mutates the objects, tables, or globals concurrently.

## Dependencies

The provided mapping identifies only F-358dc1f4e8cf itself. None of the indirect targets has a supplied resolved function ID.

The following are **document-local opaque dependency IDs**, not claims of resolved function identity:

| Opaque ID | Dispatch location | Recovered argument count | Identity status |
|---|---|---:|---|
| D-01 | Referenced object's table, byte offset 212 | 3 | Unresolved |
| D-02 | Supplied object's table, byte offset 88, second operation | 5 | Unresolved |
| D-03 | Supplied object's table, byte offset 88, final operation | 3 | Unresolved; control-flow recovery warning |

D-02 and D-03 use the same dispatch offset but are separate observations. They may resolve to the same implementation or to different implementations if intervening state changes alter dispatch.

The apparent difference in argument counts does not establish that one concrete implementation safely supports both call shapes. Possible explanations require additional evidence; this contract does not choose one.

All dependency semantics, return conventions, ownership effects, and error behavior are `documented_only`.

## Acceptance examples

These are conditional acceptance scenarios derived from the recovered representation. They are not reports of executed tests.

### Stable objects and globals

Given valid objects, unchanged dispatch tables, normal-returning dependencies, G1 equal to 1,000, and G2 equal to 17:

- D-01 receives the referenced object, the numeric adjustment result 898, and 24,576, provided G1 is an integer value for which subtraction has that meaning.
- G2 is captured as 17 after D-01 returns.
- D-02 receives the supplied object, 17, the then-read G3 value, the selected dispatch-entry value, and the unresolved incoming machine value.
- Under the recovered call model, D-03 receives the supplied object, 17, and the then-read G4 value.

No assertion is made about what these dependencies do with those arguments.

### First dependency changes G2

Given G2 initially equals 17 and D-01 changes it to 29 before returning:

- The subsequent capture is 29.
- With no further intervening change before D-02's argument read, D-02 receives 29 as its second argument.
- D-03 receives the captured value 29 even if D-02 later changes G2.

Capturing G2 before D-01 would violate the observed ordering.

### Second dependency changes global inputs

Given the captured G2 value is 29, and D-02 changes G2 to 41 and G4 to 73:

- D-03's second argument remains 29.
- Its third argument is obtained from G4 after D-02 and can therefore be 73.

Refreshing the captured G2 value for the final operation would change the represented behavior.

### Second dependency changes dispatch

Given D-02 replaces the supplied object's dispatch table with another valid table before returning:

- The final operation consults the current table at byte offset 88.
- The final target need not equal D-02's target.

Reusing a cached D-02 target for the final operation is not supported by the recovered representation.

### Invalid referenced object

Given the supplied object's field at byte offset 888 contains an unusable address:

- No explicit guard or defined fallback is visible.
- Successful normal completion is not an acceptance requirement for that input.
- A particular exception type, diagnostic, or return behavior cannot be specified from this evidence.

### Dependency does not return

If D-01 does not return, the later G2 capture and remaining operations are not reached in the recovered sequential model.

If D-02 does not return, the final operation is not reached.

## Uncertainties

The following remain `documented_only`:

- Actual ABI, pointer width, alignment rules, and source-level type declarations beyond the represented word layout.
- Whether G1's adjustment is integer arithmetic, address arithmetic, or a reconstruction artifact.
- Signedness and overflow behavior of that adjustment.
- Concrete dependency identities and prototypes.
- Meaning and validity of the unresolved incoming machine value.
- Whether the apparent five-argument second call accurately reflects the original calling convention.
- Whether the final indirect transfer is truly an ordinary call followed by return.
- Exact ordering and number of machine reads within individual call-expression preparation.
- Global volatility, asynchronous mutation, and synchronization requirements.
- Ownership, lifetime, error, and state-preservation guarantees of every indirect dependency.
- Semantic purpose of the constants and global values.

No unknown above is resolved by assuming conventional object-oriented behavior or benign dependency effects.

## Evidence

Established static evidence consists of the supplied recovered function representation:

- One explicit object-pointer input and no explicit return value.
- A referenced-object access at word index 222.
- Dispatch offsets of 212 and 88 bytes.
- A first-call argument adjustment of minus 102 and a constant argument of 24,576.
- A 32-bit global capture after the first indirect operation.
- Recovered call shapes of three, five, and three arguments.
- A fresh final dispatch expression using the supplied object.
- An unresolved incoming machine value used by the second call.
- No explicit body-level stores, validation branches, or error handling.
- A warning that the final indirect control transfer was not reliably reconstructed.

The stated sequential data-flow consequences follow from that representation. Semantic labels such as initialization or cleanup are not established.

No runtime tests, execution traces, binary-level verification, or dependency implementations were available or exercised.
