<!-- Trusted provenance: F-ca7b35ab10db; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca7b35ab10db

## Purpose

The supplied static representation describes a procedure that performs ordered indirect calls, writes 32-bit values to arithmetically derived addresses, performs several byte reads and byte writes, and then enters an unconditional loop containing further calls and byte writes.

No application-level purpose is established. In particular, the evidence does not establish that the procedure initializes an object, implements a protocol, manages storage, or processes a conventional buffer.

The recovered control flow is explicitly unreliable: the supplied material reports invalid instruction data and removed unreachable regions. This contract therefore distinguishes:

- **Established static evidence:** operations and ordering visible in the supplied representation.
- **Inferred interpretation:** a 32-bit machine-word interpretation of its arithmetic and accesses.
- **documented_only unknowns:** actual executable behavior where instruction recovery, calling conventions, dependencies, or memory validity are unresolved.

All behavior below is conditional on execution following the represented path and each preceding call and memory access completing.

## Interface

### Explicit input

The procedure has one explicit input, represented as a signed 32-bit integer. It is used as both:

- A value written to memory.
- An operand in calculating the destination of that write.

There is no established pointer type, object type, length, or permitted input range.

### Additional incoming state

The representation consumes two additional incoming machine-state values without establishing them as formal parameters:

- A 32-bit word used for multiplication and unsigned right shifts.
- A control-address value used to derive the first indirect-call target.

It also consumes a secondary register result following an early sequence of calls. The evidence does not provide the calling convention needed to identify its producer or assign it a reliable semantic type.

Consequently, the explicit argument alone does not determine behavior.

### Result and termination

The represented signature supplies no return value. The visible path has no normal return: it reaches a loop whose continuation condition is always true.

That does not establish that every invocation runs forever. A dependency may fail to return, alter control flow, or fault; a memory access may fault; and the recovered control flow may be inaccurate.

### Width and address conventions

For describing the visible arithmetic:

- A **word** is 32 bits.
- A **byte** is 8 bits.
- A word store writes four bytes; a byte store writes one.
- An unsigned right shift inserts zero bits.
- A byte store retains only the low eight bits of its supplied value.
- A byte read produces a value from 0 through 255.

The formulas below use low-32-bit arithmetic for scaling, addition, and left shifts. This is a machine-level interpretation, not a claim that overflowing signed expressions have defined source-language semantics. Address width, overflow behavior, alignment enforcement, and byte order require executable or architectural confirmation.

The notation **word destination of X** means the address formed by multiplying X by four and adding hexadecimal 14. Hexadecimal 14 is a byte displacement of 20, not an established structure-field offset.

### Preconditions for traversing the visible path

The evidence establishes no input validation. Successful traversal requires, at minimum:

- Executable targets for every indirect call.
- Accessible memory at every computed destination and byte-read address.
- Any alignment required by the actual machine for four-byte stores.
- Dependency behavior compatible with the subsequent continuation.

These requirements do not imply that the procedure checks them.

## Behavior

### Dependency notation

Opaque IDs below label individual call sites for this contract. They are not recovered symbol identities. Distinct IDs may resolve to the same implementation.

A range such as **F-1001 through F-1005** denotes five calls in increasing order. “No explicit arguments” means that the representation supplies none; it does not establish that the target consumes no registers or global state.

An unused result is not evidence that a call has no side effects.

### 1. Initial write and incoming-word transformation

First, the procedure writes the explicit input to its own word destination.

It then derives:

- **Scaled incoming word:** the incoming word multiplied by eight.
- **First shifted word:** the incoming word unsigned-right-shifted by eight.

It calls **F-1001 through F-1005**:

1. F-1001 receives the scaled incoming word. Its target depends on the incoming control-address value.
2. F-1002 through F-1005 receive no explicit arguments.
3. The result of F-1005 is retained as **result A**.

Next, it writes the scaled incoming word to the address obtained by adding hexadecimal 14 directly to that scaled word. This address is not its word destination.

### 2. Secondary-register-derived value

The procedure calls **F-1101 through F-1105**:

- F-1101 receives result A shifted left by two.
- The remaining four calls receive no explicit arguments.

After these calls, it consumes the secondary register value and derives **high-byte word** by multiplying that value by hexadecimal 01000000.

It stores the high-byte word at the address formed by multiplying the secondary register value by hexadecimal 04000000 and adding hexadecimal 14.

It also derives **second shifted word** by unsigned-right-shifting the first shifted word by eight. Thus, under the stated width interpretation, the second shifted word is the original incoming word shifted right by sixteen.

No mask separately restricts either secondary-register-derived multiplication to a byte before the multiplication.

### 3. First two result-indexed word stores

The procedure calls **F-1201 through F-1205**:

- F-1201 receives the high-byte word.
- The other calls receive no explicit arguments.
- F-1205 produces **result B**.

It writes the high-byte word to the word destination of result B.

It then calls **F-1301 through F-1305**:

- F-1301 receives result B multiplied by four.
- The other calls receive no explicit arguments.
- F-1305 produces **result C**.

It derives **scaled C** as result C multiplied by four, and writes scaled C to the address formed by multiplying result C by sixteen and adding hexadecimal 14.

### 4. First forwarding sequence

The procedure calls **F-1401 through F-1409**, in this order:

| Dependency | Explicit input | Result used |
|---|---|---|
| F-1401 | Scaled C | No |
| F-1402 | None | Forwarded to F-1403 |
| F-1403 | Result of F-1402 | No |
| F-1404 | None | Forwarded to F-1405 |
| F-1405 | Result of F-1404 | No |
| F-1406 through F-1408 | None | No |
| F-1409 | None | Retained as result D |

It writes scaled C to the word destination of result D.

### 5. Second scaled result and forwarding sequence

The procedure calls **F-1501 through F-1505**:

- F-1501 receives result D multiplied by four.
- The remaining calls receive no explicit arguments.
- F-1505 produces **result E**.

It derives **scaled E** as result E multiplied by four, and writes scaled E to the address formed by multiplying result E by sixteen and adding hexadecimal 14.

It then calls **F-1601 through F-1609**, using the same argument/result pattern as the preceding nine-call forwarding sequence, except that F-1601 receives scaled E. The final result, from F-1609, is **result F**.

It writes the second shifted word to the word destination of result F.

### 6. Additional word stores and retained byte-source address

The procedure calls **F-1701 through F-1705**:

- F-1701 receives result F multiplied by four.
- The other calls receive no explicit arguments.
- F-1705 produces **result G**.

It derives **scaled G** as result G multiplied by sixteen and writes result G multiplied by four to the address obtained by adding hexadecimal 14 to scaled G.

It also derives **byte-source base** by clearing the top bit of scaled E and unsigned-right-shifting the result by seven. Its possible range is zero through hexadecimal 00FFFFFF.

The next calls are **F-1801 through F-1805**:

- F-1801 receives three explicit arguments: scaled G, scaled G, and the first shifted word unsigned-right-shifted by seven.
- F-1802 through F-1805 receive no explicit arguments.
- F-1805 produces **result H**.

The third argument to F-1801 equals the incoming word shifted right by fifteen under the stated unsigned interpretation.

After these calls, the procedure overwrites the word at scaled G plus hexadecimal 14 with scaled G itself.

The destination is the same as the preceding store involving result G. The two stored values need not be equal.

### 7. First two byte writes through a reread selector

The procedure calls **F-1901 through F-1905**:

- F-1901 receives result H shifted left by two.
- The remaining calls receive no explicit arguments.
- F-1905 produces **result I**.

It reads one byte at byte-source base plus one. Treating that byte as an unsigned address component, it adds hexadecimal 11 and writes the low byte of result I shifted left by two to that destination.

The destination therefore lies between hexadecimal 11 and hexadecimal 110, inclusive. It is not a pointer-sized value loaded from memory.

Next it calls **F-2001 through F-2007**:

| Dependency | Explicit input | Result used |
|---|---|---|
| F-2001 | Three copies of result I shifted left by two | No |
| F-2002 | None | Forwarded to F-2003 |
| F-2003 | Result of F-2002 | No |
| F-2004 through F-2006 | None | No |
| F-2007 | None | Retained as result J |

It then rereads the byte at byte-source base plus one, adds hexadecimal 11 to that newly read selector, and writes the low byte of result I after masking with hexadecimal 3FFFFFFF and shifting right by twenty.

Because of byte truncation, this write carries bits 20 through 27 of result I. The selector is reread; intervening dependencies may change it.

### 8. A separate byte destination and two selector-based clears

The procedure calls **F-2101 through F-2105**:

- F-2101 receives result J shifted left by two.
- The remaining calls receive no explicit arguments.
- F-2105 produces **result K**.

It writes the low byte of result K shifted left by two to the address formed by shifting the second shifted word right by seven and adding hexadecimal 1C. This address is equivalently the incoming word shifted right by twenty-three, plus hexadecimal 1C.

After that write, it rereads the byte at byte-source base plus one and writes zero to the selector plus hexadecimal 11 destination.

It then calls **F-2201 through F-2207**:

- F-2201 receives the second shifted word shifted right by seven.
- F-2202 receives no explicit arguments.
- F-2203 receives result K masked with hexadecimal 3FFFFFFF and shifted right by six.
- F-2204 through F-2207 receive no explicit arguments.
- F-2207 produces **result L**.

It again rereads the selector at byte-source base plus one and writes zero to the selected destination.

### 9. Preparation for the repeating phase

The procedure calls **F-2301 through F-2305**:

- F-2301 receives result L shifted left by two.
- The remaining calls receive no explicit arguments.
- F-2305 produces **result M**.

It derives **loop state** by masking result M with hexadecimal 3FFFFFFF and shifting right by twenty-five. Initially, loop state is between zero and 31.

It derives a separate **loop selector address** from result K:

- Shift result K left by two.
- Multiply that result by hexadecimal 400.
- Add one.

It reads a byte at this address and writes zero to that byte’s value plus hexadecimal 11.

The loop selector address is retained independently of subsequent loop results.

It calls **F-2401**, passing loop state shifted left by one, then **F-2402**, with no explicit arguments. The latter supplies the initial **loop argument**.

### 10. Unconditional repeating phase

Every represented iteration performs the following ordered actions:

1. Call **F-2501** with the current loop argument and loop state shifted right by eight.
2. Call **F-2502 through F-2505**, with no explicit arguments, retaining the result of F-2505 as **iteration result A**.
3. Read the byte at the retained loop selector address.
4. Write zero at the newly read byte’s value plus hexadecimal 11.
5. Call **F-2506** with iteration result A shifted left by two.
6. Call **F-2507 through F-2510**, with no explicit arguments, retaining the result of F-2510 as **iteration result B**.
7. Replace the loop argument with iteration result B shifted left by two.
8. Replace loop state with iteration result B shifted left by twenty-six.
9. Repeat without a represented termination test.

The second argument to F-2501 is zero on the first iteration because initial loop state is at most 31. On later iterations, that argument reflects only the low six bits of the preceding iteration result B, positioned in bits 18 through 23.

## Side effects

### Memory ordering and aliasing

The procedure contains explicit word stores and byte stores, but no explicit word reads in the supplied representation. Its explicit byte reads select low-address byte-write destinations.

Each selector-based write requires a fresh byte read. A replacement implementation must not merge these reads or assume that repeated writes target the same address.

No non-aliasing guarantees are established. In particular:

- Word stores may overlap one another or later byte-read locations.
- Byte writes may alter a selector used by a later read.
- The separate byte destination may overlap a selector or another destination.
- Dependencies may observe or modify any of these locations unless their contracts establish otherwise.

The required order is the order described above, including calls between stores and repeated selector reads.

### Ownership and state

No allocation, release, ownership transfer, reference counting, or resource handle protocol is established.

The procedure neither establishes ownership of computed addresses nor checks that they designate caller-owned storage. Dependency names, argument shapes, and unused results provide no basis for assigning ownership semantics.

Dependency side effects, global-state preservation, error signaling, and nonlocal control transfers are unknown.

## Dependencies

All dependencies are indirect calls. Their target expressions vary; the first also depends on incoming control state.

The opaque call-site IDs in this contract distinguish invocation positions only. Their actual identities, equivalence relationships, signatures, and implementations are **documented_only** unknowns.

The following must not be assumed:

- That calls with no explicit arguments consume no inputs.
- That unused return values imply no effects.
- That calls preserve globals or memory.
- That any call allocates, retains, or releases storage.
- That all calls return normally.
- That the secondary register value is a conventional additional return value from the immediately preceding call.

## Acceptance examples

These are conditional static acceptance cases, not runtime test results.

1. **Initial store.** With explicit input 3, valid memory, and the stated arithmetic interpretation, the first word store places 3 at hexadecimal 20. No dependency call precedes it.

2. **Incoming-word shifts.** With incoming word hexadecimal 12345678, the first shifted word is hexadecimal 00123456 and the second is hexadecimal 00001234. F-1801 receives hexadecimal 00002468 as its third explicit argument.

3. **Byte truncation.** If result I is hexadecimal 12345678, its first selector-based byte write carries hexadecimal E0. Its later selector-based write carries hexadecimal 23.

4. **Selector changes.** If the selector read before the first byte write yields 5 and the later read yields 9, the corresponding destinations are hexadecimal 16 and hexadecimal 1A. Reusing the first selector would not preserve the represented behavior.

5. **Aliased selector.** If a byte-write destination overlaps a selector location, later reads observe memory after that write and all intervening calls. There is no established immutable-selector rule.

6. **First loop call.** Every permitted initial loop-state value produces zero as the second argument to F-2501.

7. **Subsequent loop call.** If iteration result B is 1, the next loop argument is 4 and the next second argument to F-2501 is hexadecimal 00040000.

8. **Failure before continuation.** If the first word destination is inaccessible, the evidence establishes no recovery path or error result. It does not justify claiming that any dependency is subsequently called.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied representation accurately recovers executable instructions and control flow.
- Actual calling conventions and the provenance of implicit register inputs.
- Dependency identities, argument contracts, register effects, memory effects, and failure behavior.
- Address-space interpretation, alignment rules, byte order, and overflow behavior.
- Whether low computed addresses are mapped, privileged, device-backed, or invalid.
- Whether apparently retained values survive calls as represented.
- Whether any execution reaches the unconditional loop.
- Whether external effects or unrecovered control flow permit termination.

An intent to encode fields, access registers, or manipulate tagged data might be suggested by shifts and masks, but none is established strongly enough to enter the behavioral contract.

## Evidence

This document is based exclusively on the supplied static representation of F-ca7b35ab10db, including its recovery warnings.

The represented stores, byte reads, call ordering, argument expressions, masks, shifts, and unconditional loop support the static behavior described here. The word-width interpretation and derived bit ranges are explicitly conditional arithmetic interpretations.

No dependency implementation, executable validation, architectural documentation, or runtime observation was supplied. No runtime tests have been performed.
