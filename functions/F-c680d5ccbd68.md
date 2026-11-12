<!-- Trusted provenance: F-c680d5ccbd68; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c680d5ccbd68

## Purpose

F-c680d5ccbd68 performs an ordered sequence of indirect dependency calls, calculated memory accesses, global-state access, and zeroing operations.

The supplied static representation establishes the sequence and its data dependencies. It does not establish an application-level purpose, such as allocation, serialization, initialization, or device control.

No conditional branch or loop is visible. Execution may nevertheless stop at any dependency invocation or memory access. If all preceding operations complete, the routine attempts a call to address zero with three explicit zero arguments. A return is represented after that call; successful normal return is not established.

## Interface

### Explicit inputs

The represented interface has three 32-bit inputs.

| Input | Visible use |
|---|---|
| First input | No explicit use in the body. |
| Second input | No explicit use in the body. |
| Third input, **P** | Contributes to one dependency argument and one word-write address. |

Unused explicit inputs are not proven irrelevant to machine-level execution. The displayed dependency signatures may omit register-based arguments or other implicit inputs.

### Implicit inputs and preconditions

Execution also depends on:

- An incoming link-register value used to form the first call target.
- Dependency results, including two represented 64-bit results.
- Contents and accessibility of calculated memory locations.
- A global value captured and subsequently used as an address.
- A register-derived pointer whose origin is unresolved.
- Dependency calling conventions and side effects.
- The execution environment’s treatment of low numeric addresses and address zero.

For the complete visible sequence to execute, each reached dependency must return, its target must be callable, and every reached memory access must be permitted. No visible validation enforces these conditions. Alignment requirements, address permissions, and executable-target validity are not checked.

### Outputs

The represented return type is void. There is no explicit status value.

Observable outputs comprise:

- Dependency invocations and their displayed arguments.
- Direct memory writes.
- Assignment of a dependency result to global storage.
- Any additional effects of invoked dependencies.

There is no established guarantee of normal return.

### Widths, layout, and arithmetic

A **word** is 32 bits, a **halfword** is 16 bits, and a **byte** is 8 bits. All offsets below are byte offsets.

For compactness, **Q(x)** denotes the low 32 bits of four times the 32-bit value x. Byte stores retain only the low eight bits of their source value. Right shifts of the unsigned quantities described below are logical.

Some scaling operations appear with signed intermediate types. Consequently, modular arithmetic descriptions specify the apparent 32-bit bitvector dataflow, not a claim that overflowing signed source-language arithmetic is defined. The same caution applies to multiplication by sixteen and signed address additions.

The representation does not establish multi-byte byte order, pointer width independently of the displayed address expressions, or the platform’s treatment of address overflow.

## Behavior

### Dependency identifiers and ordering

F-d001 through F-d067 identify dependency **call occurrences**, in execution order. They are opaque identifiers, not claims that the occurrences resolve to different functions.

“No displayed arguments” means that the supplied representation gives no explicit arguments. It does not establish an argument-free machine interface.

The following sequence describes the path on which every preceding operation completes.

### 1. Retain a word and publish its low byte

F-d001 is invoked with no displayed arguments. Its represented 32-bit result is passed to F-d002.

F-d003, F-d004, and F-d005 are then invoked with no displayed arguments. F-d006 follows and supplies a 32-bit result **A**.

The routine:

1. Reads a word at address Q(A) + 0x74, retaining it as **V**.
2. Writes the low byte of V at address Q(A) + 1.
3. Invokes F-d007 with Q(A).

The word read precedes the byte write. Subsequent uses of V refer to the retained value, not a fresh read of its original address.

### 2. Republish V and derive an argument from P

F-d008, F-d009, and F-d010 are invoked with no displayed arguments. F-d011 then supplies a 32-bit result **B**.

V is written as a word at address:

**16 × B + 0x14**

The multiplication is by sixteen, unlike the multiplication by four used for most other calculated word-write addresses.

F-d012 is then invoked with the 32-bit result of shifting P right by two and then left by sixteen. Under 32-bit bitvector arithmetic, the argument is:

**((P >> 2) AND 0xFFFF) × 0x10000**

Only bits 2 through 17 of P determine this argument. Higher bits are discarded by the left shift.

Next, the 32-bit result of F-d013 is passed to F-d014, and the 32-bit result of F-d015 is passed to F-d016. These calls occur after the word write and F-d012.

### 3. Write V through another calculated address

F-d017, F-d018, and F-d019 are invoked with no displayed arguments. F-d020 then supplies a 32-bit result **C**.

The routine writes V as a word at Q(C) + 0x14 and then invokes F-d021 with Q(C).

No disjointness from earlier write locations is established.

### 4. Store a scaled result through the P-derived address

F-d022, F-d023, and F-d024 are invoked with no displayed arguments. F-d025 supplies a 32-bit result **D**.

The routine writes Q(D) as a word at:

**(P AND 0xFFFFFFFC) + 0x14**

This address calculation explicitly clears P’s low two bits. It is distinct from the shifted-and-truncated P-derived argument used by F-d012.

The routine then:

- Invokes F-d026 with Q(D), Q(D).
- Passes the 32-bit result of F-d027 to F-d028.
- Passes the 32-bit result of F-d029 to F-d030.

There is no visible validation or status check between obtaining D, writing Q(D), and invoking these dependencies.

### 5. Clear a calculated word

F-d031, F-d032, and F-d033 are invoked with no displayed arguments. F-d034 supplies a 32-bit result **E**.

A word-sized zero is written at Q(E) + 0x14. F-d035 is then invoked with Q(E).

The clear covers four bytes; it is not a byte or halfword clear.

### 6. Publish a paired result and capture a global address

F-d036, F-d037, and F-d038 are invoked with no displayed arguments. F-d039 returns a represented 64-bit quantity.

Let **L** denote its low 32 bits, **H** its high 32 bits, and **T** the value Q(L). The routine then performs these operations in order:

1. Writes T as a word at Q(H) + 0x14.
2. Reads a global word and retains its address-valued contents as **G**.
3. Invokes F-d040 with T, H, T, T.
4. Invokes F-d041, F-d042, and F-d043 with no displayed arguments.
5. Obtains a 32-bit result **J** from F-d044.
6. Writes T as a word at Q(J) + 0x14.
7. Reads a word at G + 0x14, retaining it as **W**.
8. Invokes F-d045 with Q(J).

G is captured before F-d040, but W is read after F-d040 through F-d044 and after the second write of T.

A dependency changing the global address slot does not, in the represented dataflow, replace the already captured G. A dependency changing memory at G + 0x14 can affect the later W read. Likewise, if Q(J) + 0x14 aliases G + 0x14, the direct write of T precedes that read.

### 7. Perform the first byte-indexed write

F-d046, F-d047, and F-d048 are invoked with no displayed arguments. F-d049 returns a second represented 64-bit quantity.

Let **X** denote its low 32 bits and **Y** its high 32 bits.

The routine:

1. Reads an unsigned byte at address **(V >> 23) + 1**.
2. Adds 0x11 to that byte’s numeric value.
3. Writes the low byte of W at the resulting address.
4. Invokes F-d050 with Y, Y, Y, Q(X).

The lookup address ranges from 1 through 0x200, inclusive. The destination ranges from 0x11 through 0x110, inclusive.

The lookup supplies an eight-bit numeric address component. It is not a full-width pointer read.

### 8. Perform the second byte-indexed write

The 32-bit result of F-d051 is passed to F-d052. F-d053, F-d054, and F-d055 are then invoked with no displayed arguments. F-d056 supplies a 32-bit result **K**.

The routine:

1. Reads an unsigned byte at address **((X AND 0x3FFFFFFF) >> 21) + 1**.
2. Adds 0x11 to that byte’s numeric value.
3. Writes the low eight bits of **W >> 22** at the resulting address.
4. Invokes F-d057 with Q(K).

This write selects bits 22 through 29 of W, not its most significant byte.

The lookup address again ranges from 1 through 0x200, and the destination from 0x11 through 0x110. Both byte-indexed writes may target the same address. The second then overwrites the first at that location, subject to intervening dependency effects.

The second lookup observes memory at its own read time. Earlier direct writes or dependency effects may have changed its lookup byte.

### 9. Complete the calls, assign global state, and clear memory

F-d058, F-d059, and F-d060 are invoked with no displayed arguments. F-d061 supplies a 32-bit result **M**.

The remaining sequence is:

1. Invoke F-d062 with Q(M), Q(M).
2. Invoke F-d063 with no displayed arguments.
3. Obtain a represented 32-bit result from F-d064.
4. Pass that result to F-d065.
5. Invoke F-d066 and assign its result to a designated global result slot.
6. Write a zero byte through the unresolved register-derived pointer.
7. Write a zero halfword at **(X AND 0x3FFFFFFF) >> 7**.
8. Invoke F-d067 at address zero with three explicit zero arguments.

The halfword destination ranges from zero through 0x007FFFFF. Its calculation does not guarantee halfword alignment.

The producer of the byte-clear pointer is not established. Its proximity to F-d066 does not establish that F-d066 returned or initialized it.

No recovery path is visible if the global assignment, either clear, or the final invocation fails.

## Side effects

### Direct memory access

The complete visible path contains **five explicit reads in total**:

| Read category | Count |
|---|---:|
| Non-global word reads: V and W | 2 |
| Non-global byte-index lookups | 2 |
| Global-address capture: G | 1 |
| **Total** | **5** |

Thus, there are **four explicit non-global reads**, not five. The separate global-address capture supplies the fifth read.

There are **eleven direct non-global writes**:

- Six word writes: three publications of V, one publication of Q(D), one word clear, and two publications of T would be seven; however, V is published as a word only **twice**. The six word writes are therefore the two V stores, the Q(D) store, the word clear, and the two T stores.
- Four byte writes: the initial low-byte publication of V, the two byte-indexed writes, and the final unresolved-pointer clear.
- One halfword clear.

In addition, F-d066’s result is assigned to global storage. The global-address capture and global-result assignment are the two explicit global accesses.

These counts describe displayed accesses, not unique storage locations. Calculated locations may alias global storage or each other.

### Aliasing and ordering

No visible contract makes any accessed regions disjoint. Writes may overlap later read locations, other writes, dependency-accessible storage, globals, or executable memory where the environment permits such access.

Ordering is therefore significant. In particular:

- V is captured before its initial byte publication.
- G is captured before F-d040.
- The second T store precedes the W read.
- Each byte-index lookup occurs immediately before its associated direct byte write, after the dependencies specified above.
- The global-result assignment precedes the byte and halfword clears.

Atomicity, volatile semantics, memory barriers, concurrency safety, and interrupt safety are not established.

### Ownership and errors

No direct allocation or release operation is identified. Address-like arguments do not establish ownership transfer.

Dependencies have no supplied ownership contracts. They must not be assumed to preserve globals, retain or release memory, or leave accessed regions unchanged.

The routine exposes no explicit error result, dependency-status check, rollback, or cleanup-on-failure behavior. Faults and nonreturning dependencies can prevent all subsequent operations.

## Dependencies

F-d001 through F-d067 have no supplied behavioral contracts beyond the displayed calls and consumed results.

Their visible target-formation categories are:

| Occurrences | Target relationship |
|---|---|
| F-d001 | Incoming link-register value plus 0xFFE. |
| F-d012, F-d026 | Occurrence-associated embedded code anchor plus 0xF9C. |
| F-d014, F-d028 | Occurrence-associated embedded code anchor plus 0xF90. |
| F-d050 | Occurrence-associated embedded code anchor plus 0xFEC. |
| F-d064 | Occurrence-associated embedded code anchor plus 0xB0. |
| F-d067 | Address zero. |
| All remaining occurrences | Occurrence-associated embedded code anchor plus 0xFFE. |

An anchor is not necessarily shared between occurrences. These relationships do not establish ordinary function entry points, valid instruction boundaries, or distinct dependency identities.

F-d039 and F-d049 supply represented 64-bit results split into low and high 32-bit portions. Their actual machine-level return convention remains unresolved.

## Acceptance examples

These are conditional static expectations, not observed execution results. They assume successful preceding operations and the stated 32-bit arithmetic interpretation.

1. **Initial word capture.** If F-d006 returns 0x100 and the word at 0x474 is 0x12345678, the routine reads that word, writes byte 0x78 at 0x401, and invokes F-d007 with 0x400. Later represented uses of V retain 0x12345678 rather than rereading 0x474.

2. **Different uses of P.** For P = 0x00040007, F-d012 receives 0x00010000. The P-derived word-write address is 0x00040018. The dependency argument uses shift truncation; the address uses low-bit clearing.

3. **Captured global address.** If G is captured as 0x1000 and F-d040 subsequently changes the global address slot to 0x2000, the later W read still targets 0x1014. Its contents are read at that later point.

4. **Direct-write aliasing.** If Q(J) equals captured G, the second T store and the W read use the same word address. With ordinary memory semantics and no external interference between them, W equals T.

5. **First byte lookup.** If V = 0x01800000, the first lookup reads address 4. If that byte is 0xA0 and W = 0xA5C01234, the routine writes 0x34 at address 0xB1.

6. **Second byte selection.** For W = 0xA5C01234, the second byte-indexed write stores 0x97, regardless of its lookup-selected destination. It does not store 0xA5.

7. **Potentially unaligned clear.** If X = 0x00000080, the final halfword destination is address 1. An environment requiring aligned halfword accesses may fault there; no corrective branch is visible.

8. **Terminal zero-target call.** If all earlier operations complete, a call to address zero is attempted with three explicit zero arguments. This establishes the attempted invocation, not successful execution or normal return.

## Uncertainties

The following remain **documented_only**:

- Application purpose, intended data structures, and interpretation of low numeric addresses.
- Accuracy of the represented function boundary and void interface.
- Omitted register arguments and other implicit inputs.
- Signed-overflow behavior and target address arithmetic.
- Pointer width, byte order, alignment rules, and access permissions.
- Dependency identities, calling conventions, side effects, ownership behavior, and returnability.
- Origin and validity of the register-derived byte-clear pointer.
- Exact declared width of the global result slot and storage overlap relationships.
- Whether address zero is executable and whether its target returns.
- Exception, fault, concurrency, and interrupt semantics.

**Inferred intent:** repeated scaling followed by writes at offset 0x14 may suggest indexed records or encoded references. The byte lookups may suggest compact addressing. Neither interpretation is required by the established behavior.

## Evidence

This contract is based on the supplied static representation: displayed input widths, arithmetic, direct memory accesses, dependency ordering, argument flow, result splitting, global access, and the final zero-target invocation.

The direct-read accounting is two non-global word reads, two non-global byte reads, and one global-address capture: five explicit reads total.

The supplied global-overlap annotation prevents treating named global storage as necessarily independent objects. The incoming-link-dependent target and unresolved register-derived pointer limit conclusions about interface completeness.

No implementation access, dependency inspection, runtime execution, or runtime tests have been performed. Acceptance examples are conditional arithmetic and ordering consequences, not runtime observations.
