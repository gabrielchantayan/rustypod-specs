<!-- Trusted provenance: F-ea5e93c24ef5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ea5e93c24ef5

## Purpose

F-ea5e93c24ef5 performs an ordered sequence of dependency calls and memory writes, then enters an unconditional infinite loop. The supplied static representation contains no normal return path.

Its observable behavior depends on four explicit arguments, additional machine-entry state, dependency results, and a global value evaluated near the end. Several values are used both as data and as address components. Some derived destinations can be very small addresses.

No application-level purpose is established. Object-like offsets, pointer-shaped values, and self-referential writes do not establish allocation, initialization, serialization, ownership transfer, or any particular data structure.

This contract describes the supplied static representation. Whether that representation faithfully describes an executable on a particular platform remains **documented_only**.

## Interface

### Explicit arguments

| Argument | Represented width and interpretation | Visible use |
|---|---|---|
| First | 32-bit uninterpreted value | No explicit use in the body |
| Second | 32-bit uninterpreted value | No explicit use in the body |
| Third, designated **P** | Signed 32-bit integer | Address component |
| Fourth, designated **W** | Unsigned 32-bit integer | Stored directly and shifted logically right |

The absence of explicit uses for the first two arguments does not establish that the initial indirect dependency cannot observe them through the machine-level calling convention.

### Additional entry state

The representation also uses:

- **B**: a signed 32-bit entry value used in address arithmetic and stored as data.
- **C**: a signed 32-bit entry value used in address arithmetic and stored as data.
- **Q**: an entry pointer used both as a destination and as stored pointer data.
- **L**: an entry control-link value used to form the initial indirect-call target.
- **S**: a 32-bit stack-associated value supplied to F-c512cbdbd236. The representation associates it with displacement `0x168`; the applicable frame origin and calling convention are unknown.
- A global scalar associated with address `0x0000003c`, referenced in the final assignment.

These names describe roles in this contract; they are not additional declared parameters. The representation does not establish how a caller must initialize all this state.

### Derived values

- **R**: unsigned 32-bit result of the initial indirect call.
- **W₂**: W shifted logically right by two bits.
- **R₂**: R shifted logically right by two bits.
- **R₄**: R shifted logically right by four bits.
- **T**: uninterpreted 32-bit result of a later indirect call.
- **U**: signed 32-bit result of a later indirect call.
- **V**: signed 32-bit result of the final indirect call.
- **K**: the code-symbol address used as the base of one indirect target. Its numeric address is not independently supplied.

Integer stores write 32-bit words in the represented model. Stores of Q are pointer-typed stores; treating those stores as 32 bits follows the apparent machine model, not a portable guarantee about pointer width. Byte order and alignment requirements are unspecified.

### Completion and preconditions

There is no returned output. If every dependency returns and every access succeeds, execution reaches an infinite loop.

Reaching that loop requires valid call targets, sufficient machine-entry state, and permitted access to every derived address. The function does not check these conditions. A fault, nonreturning dependency, or external interruption can prevent later events; the body provides no recovery mechanism.

## Behavior

### Arithmetic rules and limits

The right shifts of W and R are unsigned logical shifts:

- W₂ and R₂ are in `0x00000000` through `0x3fffffff`.
- R₄ is in `0x00000000` through `0x0fffffff`.
- Shifting R₂ right by another two bits produces R₄ exactly.

Low-order bits are discarded; no rounding adjustment occurs.

Other arithmetic includes signed address additions, unsigned address calculations, doubling B, multiplying R by four, and shifting U left by two bits. A uniform overflow rule is not established. In particular, the left shift of signed U must not be described as universally defined multiplication by four.

Under a **conditional 32-bit modular machine interpretation**, arithmetic retains the low 32 bits and the final call’s second argument is the low 32 bits of four times U. That interpretation is useful for conditional examples but is not an established language-level guarantee for every expression.

No null, alignment, bounds, overflow, or address-mapping checks are visible.

### Ordered calls and writes

The following sequence specifies the explicit calls and stores in represented program order. Dependency-internal operations are additional and unknown. Repeated destinations remain separate events.

| Event | Established action |
|---:|---|
| 1 | Invoke F-000000000001 at the target formed from L + `0xffe`; obtain R. No explicit arguments are displayed. |
| 2 | Store W at P + B. |
| 3 | Store R at W₂ + B. |
| 4 | Store W₂ at P + `0x40`. |
| 5 | Store W at C + B. |
| 6 | Store W₂ at W₂ + `0x4c`. |
| 7 | Store W₂ at B + `0x4c`. |
| 8 | Store C at twice B. This stores C itself; it does not dereference C. |
| 9 | Store R₂ at four times R + `0x14`. |
| 10 | Store R at R + C. |
| 11 | Store R₂ again at four times R + `0x14`. |
| 12 | Invoke F-c512cbdbd236 with the 32-bit word read at C + `0x3c` as its first argument and S as its second. No result is explicitly consumed. |
| 13 | Store Q at R₄ + C. |
| 14 | Invoke F-000000000002 with R and R₂; obtain T. |
| 15 | Store Q at R₂ + C. |
| 16 | Invoke F-000000000003 with T and R₂. No result is explicitly consumed. |
| 17 | Invoke F-000000000004 with R₂ and R₂; obtain U. |
| 18 | Store B at Q + B, using Q’s address value in the represented address calculation. |
| 19 | Store U at U + `0x18`. |
| 20 | Store Q again at R₄ + C. |
| 21 | Store R₄ at R₄ + `0x18`. |
| 22 | Invoke F-000000000005 with R₄, U shifted left by two bits, R₂, and R₄, in that order; obtain V. |
| 23 | Store R₂ at Q. |
| 24 | Store R₄ at V + `0x3c`. |
| 25 | Perform the final global-derived assignment, with the separate value evaluations described below. |
| 26 | Enter an unconditional infinite loop containing no further explicit calls, memory accesses, or return. |

There are **ten explicit stores before F-c512cbdbd236 and nine afterward**, including event 25: **19 explicit stores total**.

The word at C + `0x3c` supplies the first argument at event 12. S supplies the second. The representation does not establish a relative physical read order between them, nor does it prove that obtaining S requires a fresh stack load at that point.

### Final global-derived assignment

Event 25 references the global associated with `0x0000003c` separately in two roles:

- **Gₐ**: the value evaluated for the destination-address component.
- **Gᵥ**: the value evaluated for the stored data.

The resulting operation stores Gᵥ at Gₐ + `0x18`, subject to the unresolved arithmetic and address semantics.

The representation does **not** establish the relative evaluation order of these two global references. It also does not establish either two physical loads or one guaranteed shared load. The contract therefore specifies two value evaluations without prescribing how many machine loads implement them.

Both references belong to this late assignment, after the preceding calls and stores. They must not be replaced by an entry-time snapshot.

If both evaluations observe the same value G, the assignment can be described conditionally as storing G at G + `0x18`. Without that condition, a single-G description is too strong.

### Aliasing and intervening effects

No disjointness is established among destinations, argument sources, the global location, or memory examined by dependencies.

Consequences include:

- Events 9 and 11 are distinct stores even though their expressions match.
- Event 10 can alias their destination.
- Events 13 and 20 are distinct pointer stores, separated by calls and other stores.
- Earlier writes can change the word read at event 12.
- Earlier writes or dependency effects can change the global before event 25.
- A dependency can modify memory later overwritten by this function.

The representation carries R, its shifts, and the other intermediate values across calls. This is established represented dataflow, not proof of any undocumented register-preservation convention or dependency guarantee.

## Side effects

### Memory and state

The function explicitly performs 19 stores. This excludes dependency-internal writes.

The visible memory-derived inputs include the word at C + `0x3c` and the final assignment’s global references. S is also stack-associated, but its physical acquisition time is not established. This is not a complete inventory of machine-level reads: instruction fetches, implicit calling-convention activity, and dependency accesses are outside the supplied behavioral detail.

The listed order is represented program order. Atomicity, synchronization, hardware visibility, compiler treatment, and memory-mapped-device semantics are unknown.

No explicit global save-and-restore mechanism appears. Dependencies must not be assumed to preserve globals or referenced memory.

### Ownership and lifetime

No allocation or release is explicit in this body.

The evidence does not establish that P, B, C, or Q identifies an owned allocation, that U or V is a valid object address, or that storing a value into a derived address creates an object.

Passing values to dependencies does not establish borrowing, retention, transfer, or release. Dependency ownership effects remain unknown. Successful completion of earlier calls also does not establish that later destinations remain live or accessible.

### Failure and partial execution

There is no explicit validation, error return, handler, rollback, or recovery path.

If execution stops at a call or access, earlier effects need not be undone. Subsequent effects are not guaranteed. An unused dependency result does not prove that the dependency cannot signal failure through other means.

The final loop is not an explicit success or error notification. It is simply the terminal control behavior shown after the final store.

## Dependencies

Only F-c512cbdbd236 has a supplied dependency identity. The remaining IDs below are opaque call-site labels; they do not establish distinct function identities.

| Opaque ID | Known target derivation | Visible input and result contract |
|---|---|---|
| F-000000000001 | L + `0xffe` | No explicit arguments shown; produces R |
| F-c512cbdbd236 | Supplied direct dependency identity | Word at C + `0x3c`, then S; result unused |
| F-000000000002 | `0x0891132b` + `0x588` | R, R₂; produces T |
| F-000000000003 | `0x0891133f` + `0x308` | T, R₂; result unused |
| F-000000000004 | K + `0x448` | R₂, R₂; produces U |
| F-000000000005 | `0x08911367` + `0x1b0` | R₄, U shifted left by two, R₂, R₄; produces V |

K denotes the particular code-symbol address selected in the supplied representation. Its numeric value is not asserted here.

These target derivations are known static facts. Function identities, valid executable targets, full calling conventions, hidden inputs, normal-return behavior, error conventions, register preservation, memory effects, global effects, and ownership contracts remain unknown unless separately supplied.

## Acceptance examples

These are conditional consequences, not executed tests. Each assumes the relevant dependencies return, required accesses succeed, and applicable arithmetic is defined or follows the stated machine interpretation.

### Shifted values and repeated writes

For W = `0x00000010` and R = `0x00000040`:

- W₂ = `0x00000004`.
- R₂ = `0x00000010`.
- R₄ = `0x00000004`.
- Event 6 stores `0x00000004` at `0x00000050`.
- Events 9 and 11 each store `0x00000010` at `0x00000114`.
- Event 21 stores `0x00000004` at `0x0000001c`.
- F-000000000005 receives `0x00000004` as both its first and fourth arguments.

Combining events 9 and 11 into one observable store would not preserve the represented sequence.

### Unsigned boundary

For R = `0xffffffff`, R₂ = `0x3fffffff` and R₄ = `0x0fffffff`.

These shifts do not propagate the high bit as a sign bit. This example establishes shift results only; it does not establish that every address subsequently derived from R is valid.

### Zero result

For R = zero:

- Events 9 and 11 target `0x14`.
- Events 13, 15, and 20 target C.
- Event 21 targets `0x18`.
- Event 23 stores zero at Q.

There is no explicit rejection of zero R. Whether these accesses succeed is environment-dependent.

### Aliased destinations

For B = zero, event 18 stores zero at Q. Event 23 later stores R₂ at Q.

If both complete, event 23 is the later explicit write to that destination. Intervening dependencies may introduce additional effects; the function does not preserve the earlier zero.

### Equal late global evaluations

Suppose an earlier dependency changes the global to `0x00002000`, and both final evaluations observe that value. Event 25 stores `0x00002000` at `0x00002018`.

An entry-time snapshot would not satisfy this case.

### Distinct late global evaluations

If the execution environment permits the two final evaluations to observe Gₐ = `0x00002000` and Gᵥ = `0x00003000`, event 25 stores `0x00003000` at `0x00002018`.

This is a conditional requirement, not a claim that differing observations occur or that two physical loads are required. It specifies the roles of the evaluated values without imposing an evaluation order.

### Final-call shift and completion

For U = `0x00000003`, its left shift by two produces `0x0000000c`, so that value is the second visible argument to F-000000000005.

If all calls and accesses complete, execution then reaches the final infinite loop after the remaining stores. No normal return or explicit completion notification follows.

## Uncertainties

### Established static evidence

The supplied representation establishes the explicit call and store sequence, 19-store count, unsigned right shifts, repeated writes, target derivations listed above, dependency-result uses, two global references in the final assignment, and unconditional terminal loop.

### Inferred intent

No domain-level intent is adopted. Apparent field offsets and pointer-shaped values are insufficient to identify a data structure, lifecycle operation, or intended success condition.

### documented_only

The following remain unresolved:

- Fidelity of the supplied representation to actual executable behavior.
- Complete calling convention and provenance of additional entry state.
- Pointer width and address-space rules.
- Overflow and signed-shift behavior where not established.
- Alignment, byte order, faults, and access permissions.
- Physical load count and relative evaluation order for the final global references.
- Physical acquisition time of S.
- Numeric value of K and identities of unresolved dependencies.
- Dependency side effects, ownership, lifetimes, and error conventions.
- Concurrency and memory-mapped-device behavior.
- Reachability of the represented execution sequence.
- External interruption or termination of the final loop.

These unknowns must not be replaced with assumptions that dependencies preserve globals, return valid pointers, or allocate or release memory.

## Evidence

This contract is based solely on the supplied restricted static representation and dependency map. Prior prose and reviewer feedback are not independent proof of behavior.

The store count follows from ten explicit stores before F-c512cbdbd236 and nine afterward. The final assignment separately references the global in its address and data roles. The later indirect targets retain their supplied constant or code-symbol-plus-offset derivations.

No independent implementation inspection, runtime tests, executable smoke checks, or dynamic observations were performed. Acceptance examples are conditional arithmetic and ordering consequences, not observed execution results.
