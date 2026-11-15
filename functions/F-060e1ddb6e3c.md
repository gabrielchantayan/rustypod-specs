<!-- Trusted provenance: F-060e1ddb6e3c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-060e1ddb6e3c

## Purpose

The supplied static representation describes an operation that interleaves indirect calls, 8-bit memory accesses, 32-bit memory accesses, and transformations of call results. It does not establish a reliable application-level purpose.

The visible behavior includes writes to derived addresses and to absolute addresses 13 and 17. Several addresses are obtained by reading a single byte and using that byte as an address component. Such accesses must not be reinterpreted as ordinary object-field updates without additional evidence.

The represented path ends at a bad-instruction boundary rather than a demonstrated normal return. This contract therefore describes the visible prefix of execution, conditional on the representation being accurate and on execution reaching each operation.

**Established static evidence:** argument transformations, explicit memory-access widths, data dependencies, ordering of visible operations, and the terminal boundary.

**Inferred intent:** repeated scaling and masking may encode or manipulate compact references or machine state. No particular encoding, object model, or device protocol is established.

**Unknown:** the intended service, validity of the recovered control flow, dependency contracts, and behavior beyond the terminal boundary.

## Interface

### Explicit inputs

The represented interface has three arguments:

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit use in the visible body |
| Second argument | 32 bits | No explicit use in the visible body |
| Third argument | 32 bits, represented as signed | Passed after left scaling by three bits to the first indirect call; later passed without scaling to another call |

The absence of explicit uses of the first two arguments does not prove that they are irrelevant to execution. The representation does not establish what incoming machine state an indirect dependency can observe.

### Implicit inputs

The visible body also consumes:

- An incoming link or return-address value, used to derive the first call target.
- Two incoming 32-bit register values not established as explicit arguments.
- Additional values exposed through secondary result registers after selected calls.
- Memory contents at addresses derived from those values and from call results.

For this contract, **incoming base** and **incoming payload** name the two incoming register values. These are descriptive roles, not additional declared parameters.

### Output and completion

No explicit return value or normal return is established. The represented path terminates at an invalid-instruction marker.

A caller must not rely on:

- Successful return.
- A status code.
- Failure atomicity.
- A fully initialized output object.
- Restoration of incoming registers or global state.

### Preconditions

Reaching any particular operation requires all earlier calls and accesses to complete. Safe execution additionally requires:

- Every reached indirect target to be executable under the applicable calling convention.
- Every reached read and write to be permitted at its actual derived address.
- Applicable 32-bit alignment and access rules to be satisfied.
- Any hidden register inputs to have meaningful values.

The body contains no visible checks establishing these conditions.

## Behavior

### Width and notation rules

A **byte** is an 8-bit value. A **word** is a 32-bit value.

“Low byte” means retaining bits 0–7. An unsigned byte read is zero-extended before participating in address calculation. Consequently, a byte-derived address plus 17 lies between 17 and 272, before considering any architecture-specific addressing rules.

“Scaled by four,” for example, denotes the represented 32-bit multiplication or left shift. This is not evidence that the operand is an element index or pointer.

Unsigned right shifts have zero-fill behavior. Left shifts and signed multiplications are presented as bit-level transformations in the supplied representation; overflow and execution semantics require the actual instruction set. This contract does not grant unrestricted mathematical-integer semantics or portable signed-overflow semantics.

The ordered stages below describe the body’s explicit accesses. Dependencies may perform additional accesses between them. A stored value is not presumed to remain unchanged across a call.

### 1. Entry calls and first byte writes

The first dependency receives the third argument shifted left by three bits. Its target is derived from the incoming link value plus a fixed displacement. Several subsequent calls use targets derived from fixed anchors.

After the fifth visible call:

1. Read a byte at incoming base plus 1.
2. Use that byte’s unsigned value as a low-address base.
3. Write the low byte of incoming payload shifted right by 22 bits to that base plus 17.

The latest call result, shifted left by two bits, is passed to another dependency, followed by four further calls.

Next:

1. Read a word at the retained low-address base plus 116.
2. Write its low byte to that base plus 1.

This is a full word read followed by a byte write, not a byte read from offset 116.

### 2. Incoming-base transformation and paired result

A dependency receives, in order:

- Incoming base multiplied by two.
- The unscaled third argument.
- Incoming base shifted right by two bits.
- The retained low-address base.

A subsequent call supplies a represented 64-bit result. Its low and high 32-bit halves are then passed as:

1. Low half.
2. High half.
3. Low half again.
4. Incoming base masked with `0x7fffffff`, then shifted right by one.

After further calls, a result is multiplied by four. At that derived base, the body reads a word at offset 116 and writes its low byte at offset 1. The scaled base is passed to another dependency.

### 3. Two writes selected through the doubled incoming base

After intervening calls:

1. Read a byte at twice incoming base plus 1.
2. Write zero to the byte-selected address plus 17.

A secondary-register value is passed to a dependency. A following result is passed onward and retained as a **record base**.

After another call sequence:

1. Read the selector byte again at twice incoming base plus 1.
2. Write the low byte of the unsigned doubled incoming base shifted right by 24 bits to the newly selected address plus 17.

The second destination is independently selected. It need not equal the first destination if an intervening dependency changes the selector byte.

### 4. Record-linked byte selection

A later call result is shifted left by two bits and retained as the **working base**.

The body then:

1. Reads a word at record base plus 4.
2. Reads a byte at that word’s value plus 1.
3. Writes zero at the selected address plus 17.
4. Passes working base three times to a dependency.

After further calls:

1. Shift the word read from record base plus 4 right by two bits.
2. Read a byte at that shifted value plus 1.
3. Retain the byte as the **compact base**.
4. Write zero at compact base plus 17.

These selectors come from different addresses.

### 5. Payload extraction and compact-base update

After another call sequence:

1. Read a word at working base plus 4.
2. Retain that word shifted right by 22 bits as the **extracted field**.
3. Read a word at compact base plus 116; retain it as the **payload word**.
4. Write the payload word’s low byte at compact base plus 1.

A dependency receives:

- Payload word.
- Working base shifted left by 19 bits, twice.
- The latest call result shifted left by two bits.

A later dependency receives another call result and payload word shifted right by two bits.

Following additional calls, a result scaled by four becomes another memory base. The body reads its word at offset 116 and writes the low byte at offset 1. It retains the full word.

After further calls, that full word is written at four times a secondary-register value plus 20. The secondary-register value is also passed to a dependency.

### 6. Extracted-field writes

A later dependency receives a call result followed by two copies of:

- Extracted field masked with `0x1ff`, then shifted left by 21 bits.

After more calls, another result selects a word destination at four times that result plus 20. The stored value is extracted field shifted left by 23 bits.

These are distinct transformations. The nine-bit mask belongs to the call arguments, not to the subsequent store expression. Since the extracted field came from a 32-bit word shifted right by 22, it contains at most ten significant bits before later transformations.

### 7. Index-derived state

A later result is retained as the **index result**.

The body:

1. Stores index result multiplied by four at index result multiplied by sixteen plus 20.
2. Masks index result with `0x3fffffff`, retaining the **masked index**.
3. Retains masked index shifted right by two bits as the **reduced index**.
4. Passes masked index to a dependency.

After further calls, a result is retained as the **slot result**. The body stores masked index at four times slot result plus 20, then passes four times slot result to another dependency.

### 8. Paired updates to a shared word destination

A later result becomes the **link result**. With **eightfold slot** meaning slot result multiplied by eight:

1. Write the low byte of eightfold slot at payload word plus 1.
2. Store link result shifted left by two bits at eightfold slot plus 52.
3. Pass that shifted link result to a dependency.

The body retains link result masked with `0x3fffffff` as the **link base**.

After further calls producing another represented 64-bit result:

1. Write the low byte of eightfold slot at link base plus 1.
2. Store the high 32-bit result half at eightfold slot plus 52.
3. Pass the low result half shifted left by two bits to a dependency.

The second word store targets the same expression as the earlier word store. Intervening dependency effects remain observable and unknown.

### 9. Sixteenfold-slot and absolute-address updates

The body computes **sixteenfold slot** as slot result multiplied by sixteen.

It then:

1. Writes its low byte at the address formed by link base shifted right by 20 bits, plus 17.
2. Writes the low byte of index result multiplied by four at sixteenfold slot plus 13.
3. Passes two copies of a fresh result shifted left by two bits to a dependency.

A later result is passed onward both unchanged and shifted right by three bits.

After further calls:

1. Write the low byte of sixteenfold slot to absolute address 17.
2. Repeat the byte write at sixteenfold slot plus 13.
3. Pass a fresh result shifted left by two bits to another dependency.

The absolute-address write is explicit. No global-state meaning is established for address 17.

### 10. Link-base state and block-derived writes

The body then:

1. Stores index result multiplied by four at link base plus 20.
2. Writes the low byte of link base shifted left by one bit at link base plus 13.

A new result becomes the **block result**. The body derives:

- **Block base:** block result multiplied by 32.
- **Block payload:** block result shifted left by seven bits.
- **Small selector:** block result masked with `0x07ffffff`, then shifted right by 17 bits.

After three calls, the latest result is retained as the **block field**.

The body stores block payload at block base plus 20 and the low byte of block field shifted left by two bits at block base plus 13. That shifted block field is passed to a dependency.

After further calls:

1. Write the low byte of reduced index shifted left by one bit at small selector plus 7.
2. Read a byte at block base plus 1.
3. Write the low byte of small selector at that selected address plus 17.
4. Pass block field shifted left by 21 bits three times to a dependency.

The small selector is at most 1023, so its direct byte-write destination lies between 7 and 1030.

### 11. Captured selector and fixed-constant call

A subsequent result is retained as the **shift source** and passed unchanged to a dependency.

After further calls:

1. Read and retain the byte at block base plus 1 as the **captured selector**.
2. Write zero at captured selector plus 17.
3. Pass a fresh result shifted left by two bits to another dependency.

After another call sequence:

1. Write the low byte of shift source shifted right by three bits at block base plus 13.
2. Write captured selector to absolute address 13.
3. Call a dependency with twice captured selector, `0x98`, and `0x260`.

The doubled selector is between 0 and 510. The two fixed arguments are decimal 152 and 608. Their meanings are unknown.

A following dependency receives a secondary-register value shifted right by three bits.

### 12. Replacing absolute address 13 and following a word

After further calls, one result shifted left by two bits is passed to a dependency whose result is retained.

Then:

1. Write the same previously derived byte from shift source at twice captured selector plus 13.
2. Write `0xc0`, decimal 192, to absolute address 13.
3. Pass the retained result shifted left by two bits to another dependency.

After further calls, a new result is:

- Stored shifted left by two bits at twice captured selector plus 36.
- Retained shifted left by five bits as the **expanded result**.

A word is read at twice captured selector plus 116 and retained as the **followed base**.

A dependency receives:

1. Followed base.
2. Followed base shifted left by one bit.
3. Expanded result.
4. Followed base masked with `0x7fffffff`, then shifted right by two bits.

After intervening calls, expanded result is stored at followed base plus 36. A word is read at followed base plus 116 and passed to another dependency.

### 13. Final word writes and byte-selected updates

A later represented 64-bit result supplies:

- A high half used to select a word destination at four times that half plus 20.
- A low half shifted left by two bits, retained as the **final scaled value**.

The destination receives shift source shifted right by three bits.

A dependency then receives three copies of final scaled value and the high half shifted right by three bits.

Another represented 64-bit result is passed onward as low half, high half, low half, and high half shifted right by three bits.

After further calls, final scaled value is stored at four times a fresh result plus 20. That scaled destination base is passed to another dependency.

A later result becomes the **terminal result**. The body:

1. Reads a byte at followed base plus 1.
2. Writes the low byte of terminal result shifted left by two bits at the selected address plus 17.

After three calls, it rereads the selector at followed base plus 1 and writes the low byte of terminal result masked with `0x3fffffff`, then shifted right by 20 bits, at the newly selected address plus 17.

### 14. Last repeated zeroing and terminal boundary

Define **terminal base** as terminal result shifted left by two bits.

The body:

1. Reads a byte at terminal base multiplied by 1024 plus 1.
2. Writes zero at the selected address plus 17.
3. Calls a dependency with terminal base shifted left by three bits twice, terminal base unchanged, and a fresh result shifted left by two bits.

After further calls, the selector read and zero write are repeated using the same address expression. The selector is reread, not reused.

A subsequent dependency receives a fresh result shifted left by two bits. Four more calls are represented before the bad-instruction boundary.

## Side effects

The explicit effects are byte writes, word writes, and indirect calls. No allocation, release, ownership transfer, synchronization, or rollback protocol is established.

Notable properties:

- Byte-selected destinations use zero-extended 8-bit values, not full-width pointers.
- Word reads at offset 116 contribute their low byte to several offset-1 writes while their full value may remain live.
- Direct stores include offsets 7, 13, 20, 36, and 52.
- Absolute addresses 13 and 17 are written.
- Repeated selector reads can observe intervening changes.
- Later failures do not visibly undo earlier writes.

Memory-mapped I/O, aliasing, and ordinary RAM are all unresolved possibilities. Thus access widths and ordering must not be casually changed.

## Dependencies

All visible dependencies are indirect calls with unresolved contracts. This document uses the opaque reference **F-UNKNOWN** for those unresolved targets collectively. It is not a claim that the call sites share one function.

The evidence supplies no trustworthy identity mapping for them. Some targets derive from fixed anchors plus displacements; the first depends on incoming link state. Target constants alone do not establish names or semantics.

For every F-UNKNOWN occurrence, the following remain unknown:

- Complete calling convention and implicit inputs.
- Whether it returns.
- Whether it changes memory, globals, or register state.
- Whether it allocates, releases, or transfers ownership.
- Whether it signals or handles errors.
- Whether a represented paired result is truly a 64-bit return or two independently meaningful register values.

Calls shown without explicit arguments are not established to have no inputs. Unused represented results do not make those calls removable.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Initial selector.** If the first byte read from incoming base plus 1 yields 5, the next explicit byte destination is address 22. Its value is the low byte of incoming payload shifted right by 22.

2. **Word-to-byte narrowing.** If a relevant offset-116 word read yields `0x123456ab`, the corresponding offset-1 byte write carries `0xab`. The read remains 32 bits.

3. **Selector replacement across calls.** If the selector at twice incoming base plus 1 initially contains 3 and later contains 9, the two writes target addresses 20 and 26. Reusing the first selector would not match the visible behavior.

4. **Low-address writes.** If the captured selector is 7, the fixed-constant dependency receives 14, 152, and 608. Before that call, absolute address 13 receives 7; later it receives 192.

5. **Mask boundary.** A block result of `0xffffffff` gives small selector 1023 after masking and right shift. Its offset-7 write targets address 1030, not a sign-extended address.

6. **Interrupted execution.** If a dependency does not return after earlier stores, those stores are not visibly rolled back. No later store or terminal behavior can be required on that execution.

7. **Completion boundary.** Reaching the final represented calls does not establish a normal return. Acceptance must distinguish observing the visible prefix from proving successful completion.

## Uncertainties

The following are **documented_only**:

- Instruction-level validity of the recovered path.
- Precise overflow behavior for expressions represented using signed intermediates.
- Address width and wraparound rules at execution.
- Endianness, alignment restrictions, and fault behavior.
- Whether low addresses are valid mapped storage.
- Hidden inputs and secondary-register provenance.
- Dependency identities, clobbers, ownership effects, and error conventions.
- Whether the terminal marker denotes an actual invalid instruction, incomplete recovery, or another control-flow boundary.
- Intended application semantics.

No null handling, bounds checking, retry policy, recovery mechanism, or success condition is established.

## Evidence

This contract is based only on the supplied static representation. Its strongest evidence consists of explicit access widths, masks and shifts, ordered data dependencies, call argument expressions, repeated memory reads, absolute-address writes, and the terminal bad-instruction boundary.

Static expressions are not proof that execution reaches them. Dependency behavior and application intent remain unverified. No runtime tests have been performed.
