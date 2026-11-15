<!-- Trusted provenance: F-6e7a0eb16c2b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6e7a0eb16c2b

## Purpose

The supplied static representation describes a routine that interleaves numerous unresolved indirect calls with byte and 32-bit memory accesses. Its explicit operations transform call results and an unsigned input into addresses, packed values, and arguments for later calls.

A higher-level purpose—such as object initialization, serialization, rendering, or resource management—is **not established**. The repeated offsets and scaling operations do not establish a data structure or ownership model.

The representation reports invalid instruction data and terminates at a bad-instruction boundary. This contract therefore describes the **visible execution prefix**, conditional on the representation being accurate. It does not establish the behavior of a complete executable function.

## Interface

### Input

One explicit unsigned 32-bit input is shown. It serves both as:

- A numeric value, including logical right shifts by 2 and 24 bits.
- A memory address, notably for byte reads at input address plus `0x01`.

These uses do not establish whether the caller supplies a pointer, encoded handle, tagged value, or something else.

There is also an implicit execution-context dependency: the first indirect call target is derived from an incoming return-link value. That value is not part of the explicit parameter list.

### Output and termination

The displayed interface has no explicit return value. No normal return is visible.

The visible prefix ends at an invalid-instruction termination marker. This is not evidence of a deliberate application-level error API, nor proof that the actual executable always traps there.

### Width and arithmetic conventions

For describing the visible operations:

- A byte is 8 bits; byte loads are unsigned.
- Word reads and writes are 32 bits.
- Some call results are represented as 64-bit quantities, subsequently separated into low and high 32-bit halves.
- A byte store retains only the low 8 bits of its value.
- Unsigned right shifts insert zeroes.
- Address calculations and left shifts are described using 32-bit bit-vector arithmetic, retaining the low 32 bits.

The last convention is a descriptive model, not proof of executable overflow semantics. Several displayed expressions use signed intermediates; overflow and signed left shifts require instruction-level evidence before they can be treated as a portable language contract.

Memory endianness, alignment requirements, fault behavior, and calling convention are not established.

### Preconditions

Execution of the complete visible prefix requires:

- Every reached indirect target to be executable and callable under the actual ABI.
- Every reached memory access to be permitted.
- Any word alignment requirements to be satisfied.
- Each dependency to return whenever later operations are expected.
- Required register-carried values to exist with the meanings assumed by the representation.

There are no visible checks enforcing these conditions.

## Behavior

### Interpretation of ordering

The following stages preserve the order of the explicit memory effects and their material call-result dependencies. Calls also occur between many adjacent stages.

Repeated groups of indirect calls are not collapsed into semantic no-ops. Even calls with no displayed arguments and unused displayed results can change memory or external state. Consequently, two accesses to the same address at different stages need not observe the same contents.

Descriptive names below identify captured values, not inferred source-level objects.

### 1. Input dispatch and initial byte updates

The first call receives the input. A subsequent call produces a represented 64-bit result. Another call receives, in order:

1. Its low half.
2. Its high half.
3. Its low half again.
4. The input logically shifted right by 2.

After additional calls, a returned word is multiplied by four to form the **first scaled base**. The routine reads a word at that base plus `0x74`, then writes its low byte at the base plus `0x01`. A following call receives the scaled base.

After another call group, it reads the byte at input plus `0x01`, adds `0x11` to that byte value, and writes zero at the resulting address. Thus the destination is between `0x11` and `0x110`; it is not input plus `0x11`.

An ensuing call receives an additional register-carried word whose defining operation is not established.

### 2. Input-derived and descriptor-derived byte updates

A subsequent call result is retained as the **descriptor base** and passed to another call. After more calls, the routine:

- Reads the byte at input plus `0x01` again.
- Writes the input’s high byte to the address formed by adding `0x11` to that freshly read byte.
- Passes another returned value shifted left by 2 to a call.

After further calls, a returned value shifted left by 2 becomes the **working base**. The routine then:

1. Reads a word at descriptor base plus `0x04`, retaining it as the **descriptor word**.
2. Reads a byte at descriptor word plus `0x01`.
3. Writes zero at that byte value plus `0x11`.
4. Calls a dependency with the working base repeated three times.

After another call group, it reads a byte at `(descriptor word shifted right by 2) + 0x01`. This byte becomes the **small base**. It writes zero at small base plus `0x11`, then calls a dependency with another returned value shifted left by 2.

The small base lies between 0 and 255. This is distinct from the earlier descriptor word.

### 3. Captured words and packed call arguments

After additional calls, the routine reads:

- A word at working base plus `0x04`, retained as the **field word**.
- A word at small base plus `0x74`, retained as the **payload word**.

It writes the payload word’s low byte at small base plus `0x01`.

The next material call receives:

- The payload word.
- Working base shifted left by 19, twice.
- A recently returned value shifted left by 2.

Another call returns a word that is forwarded with payload word shifted right by 2. The field word is logically shifted right by 22, yielding the **upper field**, a value from 0 through 1023.

After further calls, another result multiplied by four forms a new base. A word read at that base plus `0x74` is retained as the **copied word**; its low byte is written at that base plus `0x01`. The base is then passed to a call.

### 4. Word stores and field packing

After more calls:

- The copied word is written at four times an unresolved register-carried value plus `0x14`.
- That register-carried value is passed to a call.
- A following result is passed to another call together with two copies of `(upper field masked to 9 bits) shifted left by 21`.

A later returned value supplies an address: four times that value plus `0x14`. The routine stores upper field shifted left by 23 there, then passes the scaled address base to a call.

These two encodings are different:

- The call arguments preserve nine bits of the upper field.
- The 32-bit store of upper field shifted left by 23 retains only the nine bits that fit; the tenth bit is discarded.

### 5. Index-derived stores

A later unsigned result becomes the **index word**. Before masking it, the routine:

- Computes four times index word, retained as the **scaled index**.
- Writes scaled index at sixteen times index word plus `0x14`.

It then masks index word to 30 bits, yielding the **masked index**, and retains masked index shifted right by 2 as the **quarter index**. A call receives masked index.

A later result becomes the **slot value**. The routine writes masked index at four times slot value plus `0x14`, then calls a dependency with four times slot value.

Another result becomes the **link word**. The routine:

1. Writes the low byte of eight times slot value at payload word plus `0x01`.
2. Writes link word shifted left by 2 at eight times slot value plus `0x34`.
3. Calls a dependency with link word shifted left by 2.

It then retains link word masked to 30 bits as the **link base**.

### 6. Replacement link data and byte metadata

Following additional calls, a represented 64-bit result is obtained. The routine:

- Writes the low byte of eight times slot value at link base plus `0x01`.
- Replaces the word at eight times slot value plus `0x34` with the result’s high half.
- Calls a dependency with the result’s low half shifted left by 2.

After another call group, it computes the **wide slot base** as sixteen times slot value and performs two byte stores:

- At `(link base shifted right by 20) + 0x11`, the low byte of wide slot base.
- At wide slot base plus `0x0D`, the low byte of scaled index.

A dependency receives two copies of another result shifted left by 2.

A later result is passed with its own logical right shift by 3. After additional calls, the routine:

- Writes the low byte of wide slot base to absolute address `0x11`.
- Again writes the low byte of scaled index at wide slot base plus `0x0D`.
- Calls a dependency with another result shifted left by 2.

The repeated store is observable and is not removable merely because its value matches an earlier store.

### 7. Expanded-node data

After further calls, the routine writes:

- Scaled index as a word at link base plus `0x14`.
- The low byte of link base shifted left by 1 at link base plus `0x0D`.

A newly returned unsigned word becomes the **expansion word**. Three derived quantities are retained:

- **Expansion base:** expansion word multiplied by 32.
- **Expansion payload:** expansion word shifted left by 7.
- **Expansion fragment:** expansion word masked to 27 bits, then shifted right by 17.

The expansion fragment is between 0 and 1023.

After three more calls, the last result becomes the **child value**. The routine writes expansion payload at expansion base plus `0x14`, and the low byte of child value shifted left by 2 at expansion base plus `0x0D`. A call receives child value shifted left by 2.

Following another call group, the routine:

1. Writes the low byte of quarter index shifted left by 1 at expansion fragment plus `0x07`.
2. Reads the byte at expansion base plus `0x01`.
3. Writes the low byte of expansion fragment at that byte value plus `0x11`.
4. Calls a dependency with child value shifted left by 21 repeated three times.

### 8. Low-address writes and retained selector

A later result becomes the **transfer word** and is passed to a call. After further calls:

- The byte at expansion base plus `0x01` is read again and retained as the **selector**.
- Zero is written at selector plus `0x11`.
- A dependency receives another result shifted left by 2.

After another call group, the routine computes the **selector base** as twice selector and retains the low byte of transfer word shifted right by 3 as the **transfer byte**.

It then:

- Writes transfer byte at expansion base plus `0x0D`.
- Writes selector at absolute address `0x0D`.
- Calls a dependency with selector base, `0x98`, and `0x260`.

A following call sequence consumes another register-carried unsigned value shifted right by 3. Its provenance is unresolved.

Another result is shifted left by 2 and passed to a dependency whose return is itself shifted left by 2 for the next material call. Between those calls, the routine writes:

- Transfer byte at selector base plus `0x0D`.
- `0xC0` at absolute address `0x0D`.

Selector base is between 0 and 510.

### 9. Linked word writes

After additional calls, a result becomes the **edge value**. The routine:

1. Writes edge value shifted left by 2 at selector base plus `0x24`.
2. Reads a word at selector base plus `0x74`, retaining the **edge base**.
3. Calls a dependency with edge base, edge base shifted left by 1, edge value shifted left by 5, and `(edge base masked to 31 bits) shifted right by 2`.

After further calls it writes edge value shifted left by 5 at edge base plus `0x24`. It then reads a word at edge base plus `0x74` and supplies that word to a call.

A later represented 64-bit result supplies two halves. The routine writes transfer word shifted right by 3 at four times the high half plus `0x14`. A call receives three copies of the low half shifted left by 2, followed by the high half shifted right by 3.

Another represented 64-bit result is passed onward as low half, high half, low half, and high half shifted right by 3.

After further calls, the earlier low half shifted left by 2 is stored at four times a newly returned value plus `0x14`. The scaled destination base is passed to a call.

### 10. Final visible updates

A later result becomes the **terminal word**; terminal word shifted left by 2 becomes the **terminal scale**.

The routine reads the byte at edge base plus `0x01` and writes the low byte of terminal scale at that byte value plus `0x11`.

After three calls, it reads the byte at edge base plus `0x01` again and writes the low byte of `(terminal word masked to 30 bits) shifted right by 20` at the newly computed byte-derived destination. A call receives another result shifted left by 2.

Following another call group, the routine:

- Reads a byte at 1024 times terminal scale plus `0x01`.
- Writes zero at that byte value plus `0x11`.
- Calls a dependency with terminal scale shifted left by 3 twice, terminal scale, and another result shifted left by 2.

After further calls, it repeats the byte read and zero store. These are fresh accesses, not reuse of the previous byte.

A final call receives another result shifted left by 2. Four more indirect calls are displayed before the invalid-instruction boundary.

## Side effects

Established explicit effects include:

- Numerous byte stores through byte-derived addresses.
- Word stores at computed addresses with offsets including `0x14`, `0x24`, and `0x34`.
- Absolute byte writes to `0x0D` and `0x11`.
- Calls whose additional memory, device, allocation, synchronization, or global-state effects are unknown.

No allocation, release, ownership transfer, rollback, or cleanup is explicitly established. Passing a value to a dependency proves neither that it is a pointer nor that ownership transfers.

The visible prefix is not transactional. A fault or nonreturning call can leave earlier effects in place. Unknown dependencies may also overwrite previously stored values.

## Dependencies

All displayed calls are indirect. The supplied mapping identifies no external callee.

The following opaque dependency IDs classify unresolved call sites; they do **not** assert that sites in the same class share a target:

| Opaque ID | Visible role |
|---|---|
| F-U001 | Initial call through an incoming execution-context-derived target. |
| F-U002 | Calls with no displayed arguments, including calls whose results are unused. |
| F-U003 | Calls consuming one or more displayed 32-bit values. |
| F-U004 | Calls represented as producing a 64-bit result split into two halves. |
| F-U005 | Calls associated with unresolved additional register-carried outputs. |

Targets, calling conventions, argument completeness, return widths, global preservation, and memory ownership remain **documented_only**. No class may be replaced with a no-op merely because its effects are not represented.

## Acceptance examples

These examples are conditional static obligations, not executed tests. They assume execution reaches the relevant operation without intervening changes except those stated.

1. **Byte-derived addressing.** If the byte read at input plus `0x01` is `0xA5`, the first zero store targets address `0xB6`. It does not target input plus `0x11`.

2. **Fresh input read.** If the first such read yields `0xA5` but a dependency changes it to `0x02`, the later high-input-byte store targets `0x13`. If the input is `0xAB123400`, the stored byte is `0xAB`.

3. **Low-byte extraction.** If a word read at a scaled base plus `0x74` is `0x12345678`, the byte written at that base plus `0x01` is `0x78`, independently of memory endianness.

4. **Distinct masks.** If the field word is `0xFFFFFFFF`, upper field is `0x3FF`. Each nine-bit packed call argument is `0x3FE00000`; the later upper-field shift by 23 stores `0xFF800000` under the stated bit-vector model.

5. **Zero is not a null guard.** A zero byte selector directs the corresponding byte-derived store to `0x11`. No visible conditional skips it.

6. **Preserved ordering.** If execution reaches both absolute stores at `0x0D`, selector is written first and `0xC0` later, with intervening calls and a computed-address store. A final-memory snapshot alone does not capture this behavior.

7. **Interrupted execution.** If a dependency does not return, subsequent explicit writes are not required by this prefix. Earlier writes are not visibly undone.

## Uncertainties

The following remain **documented_only**:

- Whether the displayed control flow and function boundary are valid.
- Actual indirect target identities and callee behavior.
- The provenance of additional register-carried values.
- Whether represented 64-bit results are true scalar returns or ABI-related multi-register values.
- Executable overflow, alignment, byte-order, and memory-fault semantics.
- Whether low absolute addresses denote mapped storage, device registers, or invalid memory.
- Any concurrency, reentrancy, volatile-access, or atomicity guarantees.
- The routine’s domain purpose, successful postcondition, and normal termination behavior.

Repeated four-byte scaling may suggest encoded references, and repeated offsets may suggest related layouts. Those are inferred possibilities, not contract requirements.

## Evidence

This document uses only the supplied static representation. Established observations are limited to its explicit widths, expressions, access order, indirect calls, and truncated ending.

No implementation was accessed, no dependency was resolved, and no runtime tests were performed. The acceptance examples are arithmetic consequences of the stated conditional model, not measured execution results.
