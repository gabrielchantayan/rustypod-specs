<!-- Trusted provenance: F-9ee01d6cc6b6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9ee01d6cc6b6

## Purpose

F-9ee01d6cc6b6 performs an ordered sequence of indirect calls, byte accesses, and 32-bit memory accesses. Addresses and stored values depend on explicit arguments, additional incoming machine state, call results, and earlier memory reads.

No application-level purpose is established. Repeated offsets suggest structured storage, but initialization, allocation, decoding, serialization, and object construction are not established interpretations.

The supplied representation terminates at an invalid-instruction boundary, not an ordinary return. This contract describes its visible execution prefix, conditional on reaching each operation. It does not promise successful completion or define behavior beyond that boundary.

## Interface

### Inputs

The displayed interface has four 32-bit arguments:

- **First argument:** no direct use is shown.
- **Second argument:** passed twice to the first indirect call.
- **Third argument:** no direct use is shown.
- **Fourth argument, P:** an unsigned 32-bit value used in addresses and arithmetic.

Absence of direct use does not prove that the first and third arguments are irrelevant to unresolved machine-level behavior.

Additional incoming state comprises:

- A link-related value used to derive the first indirect target.
- An unsigned 32-bit **incoming data value**.
- An **incoming address value**, used both as an address and, shifted left by two, as the first call’s third argument.

Later operations also consume secondary result-register values whose exact producers are unresolved.

### Outputs and termination

No ordinary return value or success status is established. Observable outputs consist of memory writes and interactions with indirect dependencies.

The represented endpoint is an invalid-instruction boundary. Its actual runtime consequence is **documented_only**: the supplied data does not establish a normal return, a specific exception, or subsequent execution.

### Widths and notation

Throughout this contract:

- **word(x)** retains the low 32 bits.
- **byte(x)** retains the low 8 bits.
- **B(a)** reads an unsigned byte at address *a*.
- **W(a)** reads a 32-bit word at address *a*.
- **low(R)** and **high(R)** select the lower and upper 32-bit portions of a represented 64-bit result.
- Address calculations, multiplication, and left shifts use 32-bit wrapping arithmetic unless stated otherwise.
- Right shifts of named unsigned values are logical.

These are descriptive machine-width conventions. Some displayed expressions use signed intermediate types; exact correspondence to valid instructions remains unresolved.

A byte used as an address is zero-extended. Thus **B(a) + 0x11** lies between **0x11** and **0x110**, inclusive.

Access widths are established; byte order, alignment rules, atomicity, and device-memory behavior are not.

## Behavior

### Ordering and call notation

The phases below follow the represented order. Each read remains a distinct observable access. Dependencies may change memory or machine state between accesses; repeated address expressions do not authorize read elimination.

A **five-call run** consists of one leading call with the stated arguments followed by four calls with no displayed arguments. When used, the run’s result is the final call’s result. “No displayed arguments” does not establish an absence of implicit inputs.

All progression is conditional on earlier calls returning and earlier memory accesses succeeding. Names below identify retained data dependencies; they do not establish undocumented register-preservation guarantees.

### 1. Entry and initial byte writes

The first call receives the second explicit argument twice and **word(incoming address value << 2)**. Its target is derived from the incoming link-related value. Four calls with no displayed arguments follow; retain the last result as **entry result**.

Execution reads **B(incoming address value + 1)** and writes **byte(incoming data value >> 22)** at that byte plus **0x11**.

A five-call run receives **word(entry result << 2)**. Call its result **initial result**. Execution writes **byte(initial result << 2)** at **P + 0x0F**, then reads **B(word(P × 0x400 + 1))** and writes the same byte at the selected byte plus **0x11**.

Define **packed input = word((P & 0x003FFFFF) << 8)**.

Three calls with no displayed arguments follow. Retain the third result as **initial continuation**. Execution freshly reads **B(word(P × 0x400 + 1))** and writes **byte((initial result & 0x3FFFFFFF) >> 20)** at that byte plus **0x11**.

The two reads can return different bytes, including because of intervening writes or dependencies.

### 2. Packed-input addressing

A five-call run receives **word(initial continuation << 2)**. Execution then reads **B(word(packed input × 0x80000 + 1))** and writes zero at the resulting byte plus **0x11**.

A five-call run receives four arguments: **word(packed input << 3)** twice, **packed input**, and zero. Retain its final result. Execution rereads the same packed-input-derived byte address, names the zero-extended value **small base**, and writes zero at **small base + 0x11**.

Another five-call run receives the preceding run’s final result shifted left by two.

Define **wide base = word(packed input × 0x100000)**. Execution reads **W(small base + 0x74)** and writes its low byte at **small base + 1**. The source access is a full word read, not a byte read.

### 3. Paired results and record-base acquisition

The next call receives **wide base**, **packed input**, **word((packed input & 0x1FFF) << 17)**, and **small base**.

A call with no displayed arguments yields a represented 64-bit result. The following call receives its low word, high word, low word again, and **word((packed input & 0x0FFF) << 18)**.

Four further calls produce a result. Its fourfold value becomes **first copy base**. Execution reads **W(first copy base + 0x74)** and writes the low byte at **first copy base + 1**.

A five-call run receives **first copy base**. Execution then reads **B(wide base + 1)** and writes zero at that byte plus **0x11**.

A call receives an unresolved secondary result-register value. The following call produces **record base**. A five-call run receives **record base**; name its final result **record precursor**.

Execution freshly reads **B(wide base + 1)** and writes **byte(wide base >> 24)** at the selected byte plus **0x11**. A five-call run then receives **word(record precursor << 2)**.

### 4. Record traversal and saved fields

Define **scaled record** as the preceding run’s final result shifted left by two.

Execution reads **record link = W(record base + 4)**, then **B(record link + 1)**, and writes zero at that byte plus **0x11**.

A five-call run receives **scaled record** three times. Name its final result **traversal result**.

Execution reads **B((record link >> 2) + 1)**, retaining the zero-extended byte as **secondary base**, and writes zero at **secondary base + 0x11**.

The next five-call run receives **word(traversal result << 2)**. This leading argument is an established dependency on the immediately preceding run, not an unresolved secondary-register value.

After that run, execution reads **saved field = W(scaled record + 4)**, then **payload = W(secondary base + 0x74)**, and writes **byte(payload)** at **secondary base + 1**.

The next call receives **payload**, **word(scaled record << 19)** twice, and the latest run result shifted left by two. A following call produces a result passed to another call together with **payload >> 2**.

Define **field fragment = saved field >> 22**.

### 5. Full-word propagation

Four further calls produce a result whose fourfold value becomes **copy base**. Execution reads **copied word = W(copy base + 0x74)** and writes its low byte at **copy base + 1**.

A five-call run receives **copy base**. Execution then writes the complete **copied word** at **word(secondary index × 4 + 0x14)**. Here **secondary index** is an unresolved secondary result-register value, also passed to the next call.

A following call produces a result. The next call receives that result and **word((field fragment & 0x1FF) << 21)** twice.

Four further calls produce **field index**. Execution writes **word(field fragment << 23)** at **word(field index × 4 + 0x14)**, then calls a dependency with **word(field index × 4)**.

### 6. Masked indices and replacement writes

Four calls produce unsigned **raw index**. Define:

- **saved scaled index = word(raw index × 4)**.
- **masked index = raw index & 0x3FFFFFFF**.
- **quarter index = masked index >> 2**.

Execution writes **saved scaled index** at **word(raw index × 16 + 0x14)**. The address uses the unmasked value.

A five-call run receives **masked index**, producing **slot index**. Execution writes **masked index** at **word(slot index × 4 + 0x14)**.

A five-call run receives **word(slot index × 4)**, producing unsigned **next index**. Define **eightfold slot = word(slot index × 8)**.

Execution writes **byte(eightfold slot)** at **payload + 1**, then writes **word(next index << 2)** at **eightfold slot + 0x34**.

A call receives **word(next index << 2)**. The following call produces a result passed to another dependency; four further calls produce a represented 64-bit result.

Define **masked next index = next index & 0x3FFFFFFF**. Execution writes **byte(eightfold slot)** at **masked next index + 1**, then writes the paired result’s high word at **eightfold slot + 0x34**.

That word write targets the earlier word-write address. Intervening calls may observe or modify the earlier value.

### 7. Byte fields and absolute address 0x11

A five-call run receives the preceding paired result’s low word shifted left by two. Retain its result as **byte-field precursor**.

Define **sixteenfold slot = word(slot index × 16)**. Execution writes **byte(sixteenfold slot)** at **((next index & 0x3FFFFFFF) >> 20) + 0x11**, then **byte(saved scaled index)** at **sixteenfold slot + 0x0D**.

A call receives **word(byte-field precursor << 2)** twice. The following call produces an unsigned result passed to another dependency together with that result logically shifted right by three.

Four further calls produce **absolute-write precursor**. Execution writes **byte(sixteenfold slot)** at absolute address **0x11**, then again writes **byte(saved scaled index)** at **sixteenfold slot + 0x0D**.

The next five-call run receives **word(absolute-write precursor << 2)**: specifically, the last result obtained before those two writes.

Execution then writes **saved scaled index** at **masked next index + 0x14**, followed by **byte(masked next index << 1)** at **masked next index + 0x0D**.

### 8. Layout fields and a long-lived unsigned result

Name the preceding run’s result **layout result**. Define:

- **layout base = word(layout result × 32)**.
- **layout value = word(layout result << 7)**.
- **small fragment = (layout result & 0x07FFFFFF) >> 17**.

Three calls produce **layout seed**. Execution writes **layout value** at **layout base + 0x14**, then **byte(layout seed << 2)** at **layout base + 0x0D**.

A five-call run receives **word(layout seed << 2)**. Execution writes **byte(quarter index << 1)** at **small fragment + 7**, then reads **B(layout base + 1)** and writes **byte(small fragment)** at that byte plus **0x11**.

A call receives **word(layout seed << 21)** three times. The immediately following call, with no displayed arguments, produces **retained unsigned result**, a full unsigned 32-bit value. This name remains distinct from every subsequent result.

A five-call run receives **retained unsigned result** without an additional shift. Name its final result **selection result**.

Execution reads **selected byte = B(layout base + 1)** and writes zero at **selected byte + 0x11**.

The next five-call run receives **word(selection result << 2)**.

Define **double-byte base = selected byte × 2** and **stored byte = byte(retained unsigned result >> 3)**. Execution writes **stored byte** at **layout base + 0x0D**, then **selected byte** at absolute address **0x0D**.

The next call receives **double-byte base**, **0x98**, and **0x260**.

### 9. Absolute-byte replacement and linked writes

A call with no displayed arguments follows. Another call receives an unresolved unsigned secondary result-register value logically shifted right by three.

Four subsequent calls produce a result. A further result-producing call receives that result shifted left by two.

Execution writes **stored byte** at **double-byte base + 0x0D**, then **0xC0** at absolute address **0x0D**.

A five-call run receives the preceding result-producing call’s result shifted left by two. Name its final result **link result**.

Execution writes **word(link result << 2)** at **double-byte base + 0x24**, then reads **linked base = W(double-byte base + 0x74)**.

The next call receives **linked base**, **word(linked base << 1)**, **word(link result << 5)**, and **(linked base & 0x7FFFFFFF) >> 2**.

Four further calls follow. Execution writes **word(link result << 5)** at **linked base + 0x24**, then reads **W(linked base + 0x74)** and passes the complete word to another dependency.

### 10. Final paired results and endpoint

Four calls produce a represented 64-bit result. Define **paired index** as its high word and **paired value** as its low word shifted left by two.

Execution writes **retained unsigned result >> 3** as a complete 32-bit word at **word(paired index × 4 + 0x14)**.

This uses the full unsigned value retained in phase 8. It is neither **stored byte** nor a newer call result.

The next call receives **paired value** three times and **paired index >> 3**. A following call produces another represented 64-bit result. The next dependency receives its low word, high word, low word again, and high word logically shifted right by three.

Four calls produce an index. Execution writes **paired value** at that index times four plus **0x14**, then calls a dependency with the fourfold index.

Four calls produce unsigned **final result**. Execution reads **B(linked base + 1)** and writes **byte(final result << 2)** at that byte plus **0x11**.

Three calls produce **tail intermediary**. Execution freshly reads **B(linked base + 1)** and writes **byte((final result & 0x3FFFFFFF) >> 20)** at the selected byte plus **0x11**.

The next five-call run receives **word(tail intermediary << 2)**.

Execution reads **B(word((final result << 2) × 0x400 + 1))** and writes zero at that byte plus **0x11**.

A five-call run receives **word(final result << 5)** twice, **word(final result << 2)**, and the preceding run’s final result shifted left by two. Execution rereads the same derived byte address and writes zero at the freshly selected byte plus **0x11**.

A final five-call run receives the latest run’s final result shifted left by two. The visible representation then ends at the invalid-instruction boundary.

## Side effects

Direct memory writes are either one byte or one 32-bit word. Absolute addresses **0x0D** and **0x11** are actual represented destinations, not abstract field names.

The prefix establishes no allocation, release, ownership transfer, locking, bounds checks, or rollback. Dependency behavior must not be inferred from this absence.

Every access requires suitable mappings and permissions when reached. Derived addresses may be low, unaligned, wrapped, or overlapping. No visible null checks or alignment repairs protect them.

Aliasing can make an earlier write affect a later selector-byte or word read. Earlier effects precede any later fault or nonreturning dependency. No compensating behavior is shown; subsequent dependency or external effects remain unknown.

## Dependencies

The supplied opaque identity is **F-9ee01d6cc6b6**. No opaque identities or contracts are supplied for the indirect callees. They remain **documented_only** unresolved dependencies; no shared identity between call sites is asserted.

Displayed argument order, duplicated arguments, call counts, and consumed results are specified above. Unknown dependency properties include:

- Register and global preservation.
- Memory access and mutation.
- Allocation, release, ownership, and lifetime effects.
- Failure and nonreturning behavior.
- Relationships between distinct call targets.
- Exact producers of secondary result-register values.
- Whether a represented 64-bit result is one semantic value or a register pair.

## Acceptance examples

These are conditional arithmetic and ordering examples, not executed tests.

1. **Truncation:** With **initial result = 0x12345678**, the write at **P + 0x0F** stores **0xE0**. Its associated first selector-byte write also stores **0xE0**; the later extraction stores **0x23**.

2. **Unsigned selector:** A selector read returning **0xFF** directs its **+0x11** write to **0x110**, without sign extension.

3. **Fresh reads:** If the two phase-1 selector reads return **2** and **7**, their writes target **0x13** and **0x18**. The second destination must not reuse the first read.

4. **Wrapping:** For **P = 0x20**, **packed input = 0x2000**. Its products with **0x80000** and **0x100000** both wrap to zero. The associated byte-address expression is therefore **1**, without implying that address is accessible.

5. **Read width:** A word read yielding **0xA1B2C3D4** supplies **0xD4** to its low-byte copy. The required source access remains 32 bits.

6. **Retained value and width:** If **retained unsigned result = 0xF1234567**, **stored byte = 0xAC**. The phase-10 word write instead stores **0x1E2468AC**. Later results do not replace this dependency.

7. **Run-leading arguments:** If **traversal result = 0x40000003**, the next run’s leading argument is **0x0000000C**. The same shift-and-wrap rule applies to **absolute-write precursor**, **selection result**, and **tail intermediary** at their specified boundaries.

8. **Ordered replacement:** The word at **eightfold slot + 0x34** receives **word(next index << 2)** before the later paired high word. Dependencies between those writes can observe the first value.

9. **Absolute replacement:** Address **0x0D** receives **selected byte** and later **0xC0**, conditional on reaching both operations. If the selected byte is zero, the intervening write at **double-byte base + 0x0D** also targets **0x0D**.

10. **Interrupted progress:** A dependency that does not return prevents later specified operations from being reached. Earlier operations have already occurred; no rollback is established.

## Uncertainties

**Established static evidence:** displayed access widths, argument order, arithmetic expressions, call/result dependencies, repeated reads, direct writes, and the abnormal visible endpoint.

**Inferred intent:** repeated offsets may describe fields in structured records. No record schema, application meaning, or collection invariant is established.

**documented_only:** instruction validity, complete control flow, calling convention, address-space layout, endianness, alignment behavior, access atomicity, and post-boundary execution.

**documented_only:** dependency identities and contracts, secondary-register provenance, ownership, lifetime, concurrency, and reentrancy.

**documented_only:** exact instruction-level interpretation of signed intermediate arithmetic. The 32-bit wrapping model records the apparent machine-width behavior without claiming portable signed-arithmetic semantics.

The invalid-instruction warnings materially limit confidence that the complete representation corresponds to valid executable behavior.

## Evidence

The sole evidence is the supplied static representation of F-9ee01d6cc6b6 and its invalid-instruction warnings. The prior draft and feedback are not independent evidence.

The representation explicitly connects the four identified run-leading arguments to preceding call results shifted left by two. It also retains the phase-8 unsigned result through both a byte-truncated use and the later full-word logical-right-shift use.

No dependency bodies, caller contracts, instruction-level validation, or runtime observations were available. No runtime tests have been performed.
