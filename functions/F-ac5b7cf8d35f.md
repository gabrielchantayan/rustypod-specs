<!-- Trusted provenance: F-ac5b7cf8d35f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ac5b7cf8d35f

## Purpose

F-ac5b7cf8d35f performs an ordered sequence of indirect calls, memory transfers, two updates to a shared global location, and a final call to address zero. The supplied representation has no explicit conditional branches or loops.

Its application-level purpose is **documented_only**. Repeated scaling and field offsets suggest a structured representation, but do not establish objects, arrays, pointer tags, allocation rules, or valid address ranges.

This contract specifies the directly represented behavior. It does not establish that the represented function boundary is accurate, that any dependency returns, or that execution reaches the final operation.

## Interface

### Explicit inputs and result

| Input | Represented width | Direct use |
|---|---:|---|
| First argument | 32 bits | No explicit use in the body. |
| Second argument | 32 bits | Forwarded to the first dependency. |
| Third argument | Signed 32 bits | Shifted left by 11 and by 12 for dependency arguments. |
| Fourth argument | Unsigned 32 bits | Logically shifted right by 23 for an early dependency argument. |

No return value is declared. A normal return is represented after the final address-zero call, conditional on that call and all preceding operations completing normally.

### Implicit inputs

The body also depends on:

- An incoming link-related machine value determining the first call target.
- An implicit address used for an early byte read.
- An implicit 32-bit value supplying an early byte-store value.
- A readable global containing an anchor address.
- A readable 24-bit global contribution used in the final global update.
- A global location accepting two explicit 32-bit writes.
- An additional machine-level byte destination exposed by the last nonzero-target call.
- Dependency results and any required calling-convention state.

The provenance, lifetime, and validity of these inputs are not established.

### Widths, layout, and arithmetic

The represented memory operations include 8-bit byte accesses, a 16-bit halfword store, 32-bit word accesses, and consumption of a **24-bit global value** in the final concatenation. The 24-bit contribution is a semantic input width; the representation does not establish a particular machine load instruction or sequence for obtaining it.

Some dependencies supply 64-bit results. Those results are separated into low and high 32-bit halves; this does not itself imply a 64-bit memory access.

A byte store retains the low eight bits of its supplied value. A word-to-byte transfer reads a full 32-bit word before storing its low byte.

For the address and bit relationships below, scaling and addition use a 32-bit machine-width interpretation: discarded high bits are not recoverable. “Scaled by four” means a two-bit left shift at that width. Logical right shifts operate on unsigned 32-bit values. This model describes bit-level consequences, not a claim that signed shifts or overflow are portable source-language operations.

Recurring offsets are:

- Plus 1: byte fields or byte-table entries.
- Plus 17: byte-store destinations.
- Plus 20: word fields.
- Plus 116: word sources for byte-field transfers.

These offsets establish access locations, not object sizes or bounds. Endianness, alignment requirements, access permissions, and the exact machine arithmetic rules remain external requirements.

## Behavior

### Dependency notation and execution conditions

F-000000000001 through F-000000000098 label call occurrences in execution order. They are opaque descriptive IDs, not recovered target identities. Different occurrences may reach the same implementation.

“No explicit arguments” means none are expressed in the supplied representation; it does not exclude implicit register or calling-convention inputs. Unless specified otherwise, a call’s result is not explicitly consumed.

Each step is conditional on preceding calls returning and preceding accesses completing. Dependencies may change memory and globals. Values explicitly retained by the caller remain distinct from later reads of their original locations, subject to the actual machine convention supporting that retention.

### 1. Initial calls and byte-derived base

F-000000000001 receives the second argument. Its target depends on the incoming link-related value.

F-000000000002 supplies a 32-bit result. F-000000000003 receives that result followed by the fourth argument logically shifted right by 23.

F-000000000004 through F-000000000006 execute with no explicit arguments. F-000000000007 then supplies a 32-bit result.

Before forwarding that result, the function:

1. Reads an unsigned byte at the implicit address plus 1.
2. Zero-extends it into a retained **small base**, ranging from 0 through 255.
3. Writes bits 22 through 29 of the implicit 32-bit value to the byte at small base plus 17.

F-000000000008 receives the result of F-000000000007 scaled by four. The byte read and store precede this call.

### 2. First global write and small-base transfer

F-000000000009 through F-000000000011 execute with no explicit arguments.

F-000000000012 supplies a 32-bit result. F-000000000013 receives that result scaled by four in both argument positions.

F-000000000014 supplies the **initial global value**. The function then:

1. Writes that 32-bit value to the writable global.
2. Passes the retained value to F-000000000015.
3. Calls F-000000000016 with no explicit arguments.
4. Reads a 32-bit word at small base plus 116.
5. Writes its low byte at small base plus 1.

The small base is not reread. The retained initial global value is also distinct from the global’s subsequent contents.

### 3. Wide-result forwarding and first scaled base

F-000000000017 receives the third argument shifted left by 11.

F-000000000018 supplies a 64-bit result. F-000000000019 receives:

1. Its low 32-bit half.
2. Its high 32-bit half.
3. The initial global value logically shifted right by 23.

The third argument uses the retained result of F-000000000014, not a fresh global read.

F-000000000020 through F-000000000022 execute with no explicit arguments. F-000000000023 supplies a result whose scaled-by-four value becomes the **first scaled base**.

The function reads a word at first scaled base plus 116 and writes its low byte at first scaled base plus 1. F-000000000024 then receives the first scaled base.

### 4. Retained scaled value and repeated transfer

F-000000000025 through F-000000000027 execute with no explicit arguments.

F-000000000028 supplies a 32-bit result. Scaling it by four produces the **retained scaled value**.

The function performs a fresh word read at first scaled base plus 116 and writes that word’s low byte at first scaled base plus 1. It does not reuse the preceding transfer’s word.

F-000000000029 receives:

1. The third argument shifted left by 12.
2. The retained scaled value.
3. The retained scaled value again.
4. The unscaled result of F-000000000028 shifted left by 18.

F-000000000030 supplies a 32-bit value forwarded to F-000000000031. F-000000000032 supplies another 32-bit value forwarded to F-000000000033.

### 5. Payload capture and copies

F-000000000034 through F-000000000036 execute with no explicit arguments.

F-000000000037 supplies a result whose scaled-by-four value becomes the **payload base**. The function reads a full 32-bit **retained payload** at payload base plus 116 and writes its low byte at payload base plus 1.

F-000000000038 receives the payload base.

F-000000000039 through F-000000000041 execute with no explicit arguments. F-000000000042 supplies an index. The function writes the full retained payload at 16 times that index plus 20.

F-000000000043 receives the retained scaled value logically shifted right by two, then shifted left by 16. This does not recover bits discarded when the retained scaled value was formed.

F-000000000044 supplies a value forwarded to F-000000000045. F-000000000046 supplies a value forwarded to F-000000000047.

F-000000000048 through F-000000000050 execute with no explicit arguments. F-000000000051 supplies an index. The function writes the retained payload at four times that index plus 20. F-000000000052 then receives four times that index.

Both full-word copies use the captured payload, not a reread of payload base plus 116.

### 6. Scaled-result store and zero word

F-000000000053 through F-000000000055 execute with no explicit arguments.

F-000000000056 supplies a result that is scaled by four. The function writes this scaled result as a 32-bit word at the retained scaled value, with its bottom two bits cleared, plus 20.

Under the stated width model, those bottom bits are already zero because the retained scaled value was formed by shifting left by two. The explicit mask remains part of the represented address calculation.

F-000000000057 receives the newly scaled result twice.

F-000000000058 supplies a value forwarded to F-000000000059. F-000000000060 supplies a value forwarded to F-000000000061.

F-000000000062 through F-000000000064 execute with no explicit arguments. F-000000000065 supplies an index. The function writes a full 32-bit zero at four times that index plus 20. F-000000000066 receives four times that index.

### 7. Link stores and anchor snapshot

F-000000000067 through F-000000000069 execute with no explicit arguments. F-000000000070 supplies a 64-bit result.

The function:

1. Scales its low 32-bit half by four, producing the **retained link value**.
2. Writes that value at four times the high 32-bit half plus 20.
3. Reads the anchor global and retains its value as the **anchor address**.
4. Calls F-000000000071 with the retained link value, the unscaled high half, the retained link value, and the retained link value.

F-000000000072 through F-000000000074 execute with no explicit arguments. F-000000000075 supplies an index.

The function writes the retained link value at four times that index plus 20, then reads a word at anchor address plus 20. That word becomes the **anchor payload**.

This read uses the earlier anchor snapshot, but reads the payload at this later point. F-000000000076 then receives four times the index from F-000000000075.

### 8. First late byte-indexed update

F-000000000077 through F-000000000079 execute with no explicit arguments. F-000000000080 supplies a 64-bit result, retained as the **late low word** and **late high word**.

Before the next call, the function:

1. Logically shifts the retained payload right by 23.
2. Reads a byte at that result plus 1.
3. Zero-extends the byte into a destination base.
4. Writes the low byte of the anchor payload at that base plus 17.

F-000000000081 receives the late high word three times, followed by the late low word scaled by four.

F-000000000082 supplies a value forwarded to F-000000000083. F-000000000084 through F-000000000086 execute with no explicit arguments. F-000000000087 supplies a 32-bit result.

### 9. Second late byte-indexed update

Before forwarding the result of F-000000000087, the function:

1. Clears the late low word’s top two bits.
2. Logically shifts the remaining value right by 21.
3. Reads a byte at that result plus 1.
4. Writes bits 22 through 29 of the anchor payload at the byte-derived base plus 17.

F-000000000088 receives the result of F-000000000087 scaled by four.

F-000000000089 through F-000000000091 execute with no explicit arguments. F-000000000092 supplies a result. F-000000000093 receives that result scaled by four twice.

### 10. Final output sequence

F-000000000094 executes with no explicit arguments. F-000000000095 supplies a 32-bit value forwarded to F-000000000096.

F-000000000097 supplies an 8-bit result and exposes an additional machine-level byte destination.

The following order is explicit:

1. F-000000000097 completes.
2. The function writes one zero byte to the additional destination.
3. It reads the 24-bit global contribution.
4. It combines that contribution with the returned byte: the contribution occupies bits 8 through 31, and the returned byte occupies bits 0 through 7.
5. It writes the combined 32-bit value to the writable global used earlier.
6. It clears the late low word’s top two bits, logically shifts right by seven, and writes a 16-bit zero at the resulting address.
7. F-000000000098 invokes address zero with three explicit zero arguments.
8. The function returns without a declared result if that call returns.

The 24-bit contribution is read after the byte-zero store, not captured earlier. Its read must not be reordered ahead of that store in this contract. No particular physical load width, instruction, or atomicity follows from the represented 24-bit value.

## Side effects

Directly represented effects include:

- Ninety-eight call occurrences, including the address-zero invocation.
- Two 32-bit writes to the same global location.
- A late read of a 24-bit global contribution.
- Four word-to-byte transfers, two through the same retained scaled base.
- Two full-word copies of the retained payload.
- Stores of scaled results and one 32-bit zero.
- Three byte-indexed stores at byte-derived bases plus 17.
- A final byte-zero store through an additional dependency output.
- A final halfword-zero store.

Every byte-derived destination base ranges from 0 through 255, placing its plus-17 destination between numeric addresses 17 and 272. Both late byte-table read addresses range from 1 through 512. These bounds do not establish mapped or valid memory.

Disjointness is not established. Destinations may overlap globals, source fields, dependency-visible memory, or each other. Global declarations also carry an overlap warning; semantic widths must not be converted into assumptions about independent storage.

Ordering therefore matters even where a store appears redundant. In particular, aliasing may make the final byte-zero store affect the subsequently read 24-bit contribution.

There is no explicit allocation, release, ownership transfer, synchronization, rollback, or error-reporting mechanism. Dependency effects are additional and unresolved. A fault or nonreturning call can prevent all later operations; no recovery path is represented.

## Dependencies

F-000000000001 through F-000000000098 are unresolved call occurrences. Their explicit argument relationships and consumed results are specified above.

Most targets are represented by computed code addresses. The first also depends on incoming machine state. The last target is explicitly address zero. No separate behavioral contracts or verified target identities are supplied.

Conditional execution requirements are:

- Each preceding dependency must return normally to reach the next operation.
- Returned indices and address-like values must produce accessible memory at the time of use.
- The additional destination from F-000000000097 must permit a byte store.
- The actual calling convention must support the represented retained values and machine-level outputs.

No dependency is assumed to preserve globals or pointed-to storage, allocate memory, release memory, or transfer ownership. Passing a scaled value does not establish that it is a valid pointer.

## Acceptance examples

These are conditional static consequences, not executed tests. All assume preceding operations complete and reached accesses are permitted.

1. **Unused explicit input.** Changing only the first argument changes no directly represented expression. Undisclosed calling-convention effects remain unresolved.

2. **Unsigned extraction.** A fourth argument of hexadecimal FFFFFFFF supplies 511 as the second argument to F-000000000003.

3. **Early narrowing.** An implicit source value of hexadecimal FFFFFFFF produces an initial stored byte of 255. The destination still depends on the separate byte read.

4. **Fresh repeated transfer.** If an intervening dependency changes the word at first scaled base plus 116, the second transfer uses the changed word’s low byte.

5. **Payload snapshot.** A captured payload of hexadecimal 12345678 produces hexadecimal 78 in the payload-base byte store and hexadecimal 12345678 in both subsequent payload copies, regardless of later changes to the original payload location.

6. **Discarded bits.** If F-000000000028 returns hexadecimal 40000001, the retained scaled value is 4 under the stated width model. F-000000000043 receives hexadecimal 00010000.

7. **Anchor timing.** Changing the anchor global during F-000000000071 does not redirect the later anchor-payload read. Changing memory at the retained anchor address plus 20 can change its result.

8. **Final concatenation.** If F-000000000097 returns hexadecimal AB and the subsequently read 24-bit contribution is hexadecimal 123456, the final global write is hexadecimal 123456AB.

9. **Contribution-read ordering.** If the final byte-zero destination overlaps the contribution’s storage, the contribution is obtained after that store. The resulting numeric contribution depends on the actual layout; no endianness is assumed.

10. **Final overwrite order.** If the halfword destination overlaps previously written memory, its two-byte zero store follows the final global write and precedes the address-zero call.

11. **Interrupted execution.** If an early store faults, the body represents no rollback of completed stores or dependency effects and no explicit path to later calls.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid-input domain, and intended data structures.
- Accuracy of the represented function boundary and call signatures.
- Dependency identities, behavior, return conventions, and termination.
- Provenance and lifetime of implicit inputs and the additional byte destination.
- Memory mapping, alignment tolerance, endianness, permissions, and concurrency rules.
- Exact storage overlap among globals and other accessed locations.
- The machine mechanism and atomicity of obtaining the 24-bit contribution.
- Whether low numeric addresses or address zero are usable.
- Exact fault, exception, and nonreturning-call behavior.
- Ownership, allocation, and lifetime consequences.
- Precise machine semantics underlying signed arithmetic and shifts.

Scaling and recurring offsets support only an inference of structured data manipulation. They do not establish safe dereference conditions or an ownership protocol.

## Evidence

Established static evidence consists of the supplied representation’s declared widths, ordered calls, consumed results, masks, shifts, explicit memory accesses, global updates, 24-bit concatenation operand, and final address-zero call.

The opaque dependency IDs are contract labels for occurrences, not verified function identities. The 32-bit arithmetic interpretation is stated explicitly rather than asserted as portable source-language behavior.

No runtime tests, implementation execution, dependency inspection, memory-map validation, ownership validation, or machine-instruction verification have been performed. Acceptance examples are conditional deductions from the static representation, not observed outcomes.
