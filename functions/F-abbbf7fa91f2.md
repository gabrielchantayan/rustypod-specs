<!-- Trusted provenance: F-abbbf7fa91f2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-abbbf7fa91f2

## Purpose

F-abbbf7fa91f2 repeatedly invokes opaque dependencies and performs interleaved memory reads and writes. Its represented effects include 8-, 16-, and 32-bit stores, address derivation, a software breakpoint, and a persistent adjustment to an implicit execution-frame base.

**Established static evidence:** the supplied representation has an unconditional loop, no normal return, and no explicit validation or recovery branch. Another iteration follows if execution reaches the loop boundary.

**Inferred intent:** the memory operations may propagate or initialize state. No application-level purpose, object model, or completion criterion is established.

This contract describes the supplied static representation, conditional on dependencies returning, memory accesses succeeding, and execution continuing past the breakpoint. It does not establish that the representation captures every machine-level register effect.

## Interface

### Inputs and outputs

The displayed interface has no explicit parameters and no explicit return value. No normal return is represented.

Execution nevertheless depends on:

- An incoming 32-bit link-like value used to derive the first indirect call target.
- An implicit frame base used for frame-relative accesses.
- An incoming register-derived address contribution used near the end of each iteration.
- Two 32-bit global values loaded at separate points.
- Memory reached through those values, dependency results, implicit register outputs, and calculated addresses.
- Dependency behavior and the execution environment’s breakpoint handling.

No explicit allocation, release, bounds check, null check, alignment correction, or error result appears.

### Widths and interpretation

| Quantity or operation | Represented width and treatment |
|---|---|
| Byte read or write | 8 bits |
| Unsigned halfword read | 16 bits, zero-extended when subsequently used as a word |
| Halfword write | Low 16 bits of the supplied value |
| Word read or write | 32 bits |
| Paired dependency result | 64 bits, separated into low and high 32-bit portions |
| Address-like value or global pointer value | 32 bits |
| Additions, scaling, and left shifts described below | Low 32-bit result |

The logical right shift by 26 operates on an unsigned 32-bit value. The fixed right shift by 18 also operates on an unsigned 32-bit value.

Descriptions of wrapping arithmetic specify the represented 32-bit dataflow; they do not assert portable overflow semantics for a source-language implementation. Endianness, alignment requirements, address tagging, and access-fault behavior are unresolved.

### Descriptive terms

“Frame base,” “record pointer,” “selector,” and “field value” describe how values are used, not established object types.

The **first pair**, **second pair**, and **final pair** are three separate 64-bit dependency results. A pair’s high portion may be used as an address without any represented validation.

A value described as **retained** is reused in the supplied representation rather than reloaded. This is not a guarantee that an opaque dependency preserves physical registers, allocation lifetime, or pointed-to memory.

## Behavior

### 1. Ordered initialization and first calls

At the beginning of each iteration, the byte later written at offset `0x15` is initialized to zero.

F-d01 is then invoked with two zero arguments. Its target is the 32-bit sum of the current link-like value and `0xFFE`.

Only **after F-d01 returns**, a second byte is initialized to zero. This second byte contributes to an argument of F-d06. Its initialization precedes F-d02; the representation does not initialize both bytes before F-d01.

The next operations occur in this order:

1. Invoke F-d02.
2. Invoke F-d03.
3. Read the first 32-bit global pointer value.
4. Invoke F-d04.
5. Invoke F-d05, obtaining the first pair.
6. Invoke F-d06 with two 32-bit arguments:
   - The first pair’s low portion.
   - The sum of its high portion, its low portion, and the second initialized byte.

The byte contribution is zero in the represented dataflow. The sum retains only 32 bits; no separate carry is supplied.

After F-d06 returns, an associated 8-bit register-derived output is written at offset `0x11` from the retained first global pointer.

F-d07 is invoked next. Only after it returns is the first initialized byte, zero, written at offset `0x15` from that same retained pointer.

F-d08 then supplies a 16-bit result and an associated implicit address-bearing register output. The result is written as a halfword at offset `0x0C` from that address.

Neither destination has a represented size, validity check, or ownership guarantee. The address accompanying F-d08 is not an explicit returned pointer in the displayed interface.

### 2. Second pair and frame-relative accesses

F-d09 produces the second pair. Two writes immediately follow, in order:

1. Write a zero halfword at offset `0x2C` from its high portion, interpreted as an address.
2. Write its low portion logically shifted right by 26 as a word at frame offset `0x26C`.

The second write contains a value from 0 through 63.

A retained 32-bit quantity is then assigned `0x21C48960`. The following sequence occurs:

1. Invoke F-d10.
2. Read a 32-bit pointer from frame offset `0x320`.
3. Invoke F-d11.
4. Invoke F-d12.
5. Write the second pair’s low portion, shifted left by two and truncated to 32 bits, at offset `0x54` from the pointer read in step 2.

The pointer is not reread after F-d11 or F-d12. Those calls remain ordered before the write; they may affect memory or execution state, and no destination-lifetime guarantee is established.

### 3. Narrow address, index, and breakpoint

F-d13, F-d14, F-d15, F-d16, and F-d17 are invoked in that order.

An unsigned halfword is then read at offset `0x34` from `0x21C48960`, giving the **narrow address**. Thus the read address is `0x21C48994`, and the loaded value is between 0 and 65,535.

Next:

1. Invoke F-d18.
2. Read a word at frame offset `0x280`.
3. Pass that word to F-d19.
4. Retain F-d19’s 32-bit result as the **index**.
5. Execute a software breakpoint with immediate `0x92`.

All subsequent behavior is conditional on continuation past the breakpoint. No particular trap, termination, or resume policy is established.

Following continuation:

- Shift the index left by two, retaining 32 bits.
- Derive a fixed address quantity by logically shifting `0x21C48960` right by 18 and multiplying by four. The result is `0x21C4`.
- Read a word at offset `0x44` from the narrow address. Retain it as the **record pointer**.

The narrow address is used directly as a base, despite originating from a 16-bit read.

### 4. Intermediate calls and halfword writes

Invoke F-d20, then F-d21. Retain F-d21’s 32-bit result.

Read a word at offset `0x04` from the narrow address. Invoke F-d22 with:

1. F-d21’s retained result.
2. The newly read word.

Then:

1. Read the second 32-bit global value.
2. Invoke F-d23.
3. Write the low 16 bits of the retained global value at the address formed by adding that value to the shifted index.

The second global value supplies both the destination base and the stored data. This is a **16-bit write with a four-byte index stride**, not a word-sized table update.

Invoke F-d24. Read a word at `0x21C4 + 0x54`, namely `0x2218`, and retain it as the **later pointer**.

Invoke F-d25, obtaining an address-bearing result. Write the halfword `0x21C4` at offset `0x04` from that result.

Invoke F-d26 and F-d27 in order.

### 5. Final pair and frame-base transition

Read an unsigned halfword at offset `0x34` from the later pointer. This is the **selector**.

Invoke F-d28, obtaining the final pair. Its low portion is the **payload**; its high portion is the **pair address**.

The represented order is:

1. Write the payload as a word at frame offset `0x258`, using the frame base before the adjustment below.
2. Read an unsigned halfword at offset `0x14` from the pair address. Retain its zero-extended value as the **field value**.
3. Derive the payload shifted left by six, retaining 32 bits.
4. Derive the pair address multiplied by four, retaining 32 bits.
5. Decrease the implicit frame base by `0xF4`, or 244 bytes.
6. Derive the selector multiplied by four.
7. Write the payload as a word at address `0x74` plus the selector multiplied by four.

The last destination has no separate global or frame base in its represented address expression. Since the selector is an unsigned halfword, this destination ranges from `0x74` through `0x40070`, inclusive.

No restoration of the frame base appears before the next iteration.

### 6. Record updates and byte transfer

Invoke F-d29. Write the field value as a 32-bit word at offset `0x54` from the retained record pointer.

Invoke F-d30. Write the same field value as a 32-bit word at offset `0x64` from that pointer.

Both writes zero-extend the original 16-bit field; neither is a halfword write.

Invoke F-d31. Write the low 16 bits of the record pointer at the address formed from:

- The payload shifted left by six.
- An associated implicit register-derived address contribution.

Invoke F-d32, then F-d33.

Read one byte at the address equal to the selector plus four times the selector. This is five times the selector, with no separate pointer base. Its range is 0 through 327,675.

Invoke F-d34. Write the retained byte at the 32-bit sum of:

- The field value.
- The incoming register-derived address contribution.
- `0x19`.

The byte read precedes F-d34 and is not repeated afterward.

Invoke F-d35. Write the scaled pair address—the pair address multiplied by four—as a word at offset `0x44` from the record pointer.

### 7. Final link-like assignment, argument read, and repetition

The final sequence is ordered as follows:

1. Invoke F-d36.
2. After F-d36 returns, assign `0x08712031` to the represented link-like state.
3. Read a word at offset `0x44` from the scaled pair address.
4. Invoke F-d37 with that word.
5. If F-d37 returns, repeat the loop unconditionally.

The link-like assignment is **before the argument read and before F-d37**, not after F-d37.

Consequently, if the argument read faults, the represented assignment has already occurred. If F-d37 does not return, the assignment likewise precedes that invocation. The dependency may also encounter implicit execution state associated with this sequence; its actual observations are unknown.

In the represented dataflow, the next iteration uses the assigned link-like value, yielding the first-call target `0x0871302F`. This does not prove that a physical register retains `0x08712031` across F-d37. Machine-level preservation and the dependency’s register effects remain unresolved.

## Side effects

The direct side effects are the ordered memory writes, software breakpoint, frame-base adjustment, link-like assignment, and dependency invocations described above.

Important constraints:

- The writes do not constitute an atomic transaction. Calls and reads intervene between related writes.
- Retained pointers and scalar values are reused across calls without represented revalidation.
- Dependencies may alter globals, destination contents, frame storage, or allocation lifetime. No preservation contract is available.
- Different destinations may alias each other, earlier read locations, globals, or frame storage. Disjointness is not established.
- The frame-base decrement persists in the represented loop.
- The link-like assignment occurs even if the subsequent argument read prevents F-d37 from being invoked.
- A fault, nonreturning dependency, or noncontinuing breakpoint can leave an arbitrary prefix of the described effects completed.

No explicit ownership transfer, allocation, deallocation, synchronization, rollback, or cleanup appears. This absence does not constrain what opaque dependencies may do.

## Dependencies

F-d01 through F-d37 are opaque **call-site identifiers**, introduced here in invocation order. They do not establish distinct underlying functions. Except for F-d01’s link-derived target, the other calls use fixed address calculations in the supplied representation; their executable identities and contracts are unavailable.

| Identifier | Established interface contribution |
|---|---|
| F-d01 | Two zero arguments; link-derived target |
| F-d02–F-d04 | No displayed arguments or consumed explicit results |
| F-d05 | First 64-bit pair |
| F-d06 | Two 32-bit arguments; associated 8-bit register output |
| F-d07 | No consumed explicit result |
| F-d08 | 16-bit result; associated implicit address output |
| F-d09 | Second 64-bit pair |
| F-d10–F-d18 | Ordered calls without consumed explicit results |
| F-d19 | Frame-loaded word argument; 32-bit index result |
| F-d20 | No consumed explicit result |
| F-d21 | 32-bit result |
| F-d22 | Two 32-bit arguments |
| F-d23–F-d24 | No consumed explicit results |
| F-d25 | Address-bearing result |
| F-d26–F-d27 | No consumed explicit results |
| F-d28 | Final 64-bit pair |
| F-d29–F-d30 | No consumed explicit results |
| F-d31 | Associated implicit address contribution |
| F-d32–F-d36 | No consumed explicit results |
| F-d37 | One word argument, read after the link-like assignment |

“No consumed explicit result” does not mean purity, absence of implicit outputs, or preservation of memory and registers. Failure behavior and ownership effects are unknown for every dependency.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests.

1. **Initialization interrupted by F-d01.**  
   If F-d01 does not return, the byte intended for the offset-`0x15` write has already been initialized to zero. The byte contributing to F-d06’s second argument has not yet reached its represented initialization. F-d02 and subsequent operations are not reached.

2. **Six-bit extraction and truncated shift.**  
   If the second pair’s low portion is `0xFC000003` and execution reaches both writes, the word at frame offset `0x26C` receives 63. The later word at offset `0x54` from the frame-loaded pointer receives `0xF000000C`.

3. **Unsigned field expansion.**  
   If the halfword read from the final pair’s address is `0xFFFF`, the two record writes at offsets `0x54` and `0x64` each contain the 32-bit value 65,535, provided both are reached.

4. **Selector zero.**  
   A selector of zero makes the indexed payload-write address `0x74` and the later source-byte address zero. No represented special case prevents the zero-address read.

5. **Pointer truncation.**  
   If the record pointer’s bits are `0x12345678`, the halfword write following F-d31 contains `0x5678`. Its destination still depends on the shifted payload and implicit address contribution.

6. **Retained destination pointer.**  
   If F-d11 or F-d12 changes the frame slot at offset `0x320`, the subsequent offset-`0x54` write still uses the value previously read from that slot in the represented flow. This does not establish that its destination remains valid.

7. **Breakpoint stops execution.**  
   If the breakpoint prevents continuation, earlier effects may have occurred, but the post-breakpoint record-pointer read and subsequent writes are not reached.

8. **Final argument read faults.**  
   If F-d36 returns and the following argument read faults, the link-like assignment to `0x08712031` has already occurred. F-d37 has not been invoked.

9. **F-d37 does not return.**  
   If its argument read succeeds but F-d37 never returns, the link-like assignment and argument read precede that call, but no next iteration is reached. No preservation of physical link state during the call is implied.

10. **Continued iteration.**  
    If every operation completes and F-d37 returns, the represented loop begins again with a frame base 244 bytes below the preceding iteration’s base. Its represented link-like input is the assigned constant; correspondence to actual preserved machine state remains unresolved.

## Uncertainties

The following remain **documented_only**:

- Application purpose and data-structure meanings.
- Completeness and machine-level accuracy of the supplied representation.
- Dependency identities, calling conventions, register effects, and failure behavior.
- Physical preservation of the link-like state across F-d37.
- ABI meaning of implicit register-derived outputs and incoming contributions.
- Memory ownership, allocation lifetime, and dependency allocation or release behavior.
- Validity of narrow values used as addresses.
- Endianness, alignment, address mapping, and fault handling.
- Breakpoint semantics and whether execution resumes.
- Whether the repeated frame-base decrement accurately represents executable behavior.
- External interruption, concurrency, and atomicity beyond the represented access widths.

These unknowns do not authorize assumptions of safe addresses, stable globals, callee preservation, successful breakpoint continuation, or rollback.

## Evidence

This contract uses only the supplied static function representation. The earlier draft and feedback are not independent behavioral evidence.

Direct evidence establishes the access widths and offsets, call order, paired-result splitting, unsigned halfword reads, narrow-store truncation, breakpoint immediate, frame-base decrement, and unconditional repetition.

The two ordering boundaries are explicit in that representation:

- The offset-`0x15` byte is initialized before F-d01; the argument-contributing byte is initialized after F-d01 and before F-d02.
- The link-like constant is assigned after F-d36, before the final argument read and F-d37.

Derived arithmetic establishes `0x21C4`, the word-read address `0x2218`, the six-bit extraction range, the selector-derived address ranges, and the represented subsequent first-call target.

No implementation inspection beyond the supplied representation, dependency inspection, runtime execution, or runtime tests have been performed.
