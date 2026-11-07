<!-- Trusted provenance: F-a4ea0691ddc5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a4ea0691ddc5

## Purpose

F-a4ea0691ddc5 selects between an acquisition-and-conversion path and an alternate operation involving a configured source and a caller-supplied output record.

The acquisition path obtains a status and temporary data, invokes three further dependencies regardless of that status, and then either:

- Writes six converted components to selected locations in the output record and clears a state field; or
- Clears two bits in the output record and returns one of two error values.

The alternate path delegates an operation with a length argument of 12 and returns zero if that dependency returns normally.

A date/time interpretation is **inferred intent**, supported by the addition of 2000 to one component and five accompanying byte-sized results. Calendar semantics, component names, encodings, and valid ranges are not established. Likewise, the alternate operation may be cache-like, but this is not an established guarantee.

## Interface

### Inputs

**Selector:** A 32-bit integer. Zero allows selection of the alternate path; any nonzero value forces acquisition after the initial state-field test. On acquisition, the selector is passed unchanged to F-a8da4de9bd1d.

**Output address:** Caller-supplied storage accessed by byte offset. Successful acquisition directly writes a 16-bit field at offset 0 and individual bytes through offset 10. Failure reads and writes byte 11. The alternate dependency receives this address and a length argument of 12.

A caller permitting any branch must provide at least 12 accessible bytes, subject to the unknown access requirements of the alternate dependency. The failure branch requires byte 11 to be readable as well as writable.

The function performs no explicit null, size, accessibility, or alignment checks. The 16-bit store requires alignment acceptable to the target environment. Its byte order is not established by the supplied evidence.

### Return value

The explicit result is 32 bits:

| Result | Established condition |
|---|---|
| `0x00000000` | Acquisition status is zero and conversion completes, or the alternate dependency returns normally. |
| `0xFFFFFFFA` | Acquisition status equals `0x15`. |
| `0xFFFFFFFB` | Acquisition status is nonzero and differs from `0x15`. |

The two nonzero results represent −6 and −5 under signed 32-bit interpretation. Their domain-specific meanings are unknown.

A zero result on the alternate path does not independently certify that 12 bytes were transferred or that the destination is valid. The dependency’s result is not inspected.

### Direct output layout

On successful acquisition:

| Output byte offset | Width | Stored value |
|---|---:|---|
| 0–1 | 16 bits | First conversion result narrowed to signed 16 bits, increased by 2000, then stored in 16 bits |
| 2 | 8 bits | Low eight bits of the second conversion result |
| 3 | 8 bits | Low eight bits of the third conversion result |
| 8 | 8 bits | Low eight bits of the fourth conversion result |
| 9 | 8 bits | Low eight bits of the fifth conversion result |
| 10 | 8 bits | Low eight bits of the sixth conversion result |

There are no direct success-path writes to offsets 4–7 or 11.

On acquisition failure, the only direct output write is to byte 11. Its low two bits are cleared and its upper six bits are preserved from the byte read for that operation.

## Behavior

### State observations and branch selection

The state-base configuration is read more than once. These observations have distinct purposes and must not be treated as a single snapshot:

1. At entry, the function reads the configured state-base address and retains that address for a possible final reset.
2. It separately reads the configured state-base address again, then reads the 32-bit field at byte offset 4 from that newly obtained base.
3. If that field is nonzero and the selector is zero, it reads the configured state-base address yet again and freshly reads the 32-bit field at offset 4 for the first argument to D-01.

The selection reads do **not** use the captured reset base. D-01’s field argument is **not** a reuse of the field value tested earlier.

The branch conditions are short-circuited in this order:

- If the selection field read yields zero, select acquisition. The selector does not need to be tested, and D-01 is not called.
- Otherwise, if the selector is nonzero, select acquisition without calling D-01.
- Otherwise, call D-01 with the freshly read field value and a configured reference argument. A nonzero result selects acquisition; zero selects the alternate path.

Even a nonzero selector does not bypass the initial selection-field read.

There is no established equality between the captured base, the base used for the initial field test, and the base used to obtain D-01’s argument. Similarly, a nonzero initial field observation does not guarantee a nonzero value passed to D-01. Mutation between reads can make these observations differ. No synchronization is visible.

The configured reference argument is supplied to D-01, but its representation and interpretation are unknown. The relative evaluation order of the separate arguments to that call is not established.

### Alternate path

The alternate path calls D-02 with:

1. The caller’s output address.
2. A configured source address.
3. The integer length 12.

The function does not inspect D-02’s return value. If D-02 returns normally, F-a4ea0691ddc5 returns zero.

This path does not directly invoke the acquisition or conversion dependencies, mask output flags, or reset the captured state field. Actual destination contents and any source access depend on D-02.

### Acquisition and intermediate transformations

The acquisition path performs these calls in order:

1. F-a8da4de9bd1d receives the selector and the address of an eight-byte temporary region. Its 32-bit status is retained.
2. F-1b1da0aa8204 receives a second eight-byte temporary region and the first region.
3. F-a8474adba0ce receives the second region, a configured argument, and a third eight-byte temporary region.
4. F-541356c82e59 receives a component-storage address and the third region.

Only after all four calls return does the function examine the retained acquisition status. A nonzero status therefore does not suppress the three subsequent transformation calls.

The configured argument for F-a8474adba0ce is read for that call; it is not visibly captured alongside the entry state base. Earlier dependencies are not assumed to preserve its configuration.

The first temporary region is not visibly initialized before acquisition. No direct initialization of the other temporary regions or component storage is shown either. Consequently, if a dependency leaves data unwritten, subsequent operations can consume unspecified contents. The dependencies’ production guarantees and tolerance of failed acquisition are unknown.

The component-storage address is subsequently used to read bytes at relative offsets 0, 1, 2, 4, 5, and 6. Byte 3 is not directly consumed by this function. These accesses do not establish the complete storage layout or F-541356c82e59’s write extent.

### Acquisition failure

For any retained nonzero acquisition status:

1. Read output byte 11.
2. Clear bits 0 and 1 of that value.
3. Write the resulting byte back to output offset 11.
4. Return `0xFFFFFFFA` for status `0x15`; otherwise return `0xFFFFFFFB`.

There are no component-conversion calls on this branch and no direct state-field reset.

The function does not directly initialize or overwrite the remaining output bytes. Their preservation is conditional on the absence of dependency effects and relevant aliasing.

### Acquisition success

When the retained acquisition status is zero, the function calls F-a56604637c76 six times. Each call receives one component byte. Each destination write completes before the next conversion call:

| Order | Component-storage offset | Result treatment | Output offset |
|---:|---:|---|---:|
| 1 | 6 | Narrow to signed 16 bits, add 2000, store 16 bits | 0–1 |
| 2 | 5 | Retain low eight bits | 2 |
| 3 | 4 | Retain low eight bits | 3 |
| 4 | 2 | Retain low eight bits | 8 |
| 5 | 1 | Retain low eight bits | 9 |
| 6 | 0 | Retain low eight bits | 10 |

The component bytes are read for their respective calls; the function does not visibly capture all six inputs before starting conversion. Effects of an earlier dependency call must not be assumed irrelevant to later reads.

For the first result, the signed 16-bit value lies between −32768 and 32767. Adding 2000 produces an intermediate between −30768 and 34767, within signed 32-bit range. The final 16-bit store retains the low 16 bits. Values outside signed 16-bit range after addition are not rejected.

The remaining results are narrowed to eight bits without visible range validation. No calendar checks, consistency checks, or independent conversion-error branches are established.

After all six output writes, the function writes a 32-bit zero to offset 4 from the state-base address captured at entry. It does not refresh that base for the reset. It then returns zero.

## Side effects

Direct effects consist of configuration and state reads, ordered dependency calls, selected output writes, and a conditional state reset.

The state reset targets the entry-captured address even if a dependency changes the configured base. Conversely, branch selection uses separate configuration reads, not that captured address.

Successful acquisition does not directly clear either low bit of output byte 11. Acquisition failure does not directly reset the state field. The alternate path does neither.

No explicit allocation or release is present. The caller supplies the output storage; the function does not directly release it. Temporary-storage addresses are passed to dependencies, but the evidence establishes neither ownership transfer nor permission to retain those addresses. Dependency retention and lifetime requirements remain unknown.

Dependencies are not assumed to preserve configuration, global state, output memory, or other reachable storage. Therefore, statements about unchanged bytes describe absence of direct writes, not unconditional end-to-end preservation.

Aliasing is unchecked. For example, output storage overlapping the captured reset field can be modified again by the final 32-bit reset after the six component writes. Other overlaps may affect subsequent reads. General overlap safety is not established.

No rollback or recovery mechanism is visible. If a dependency does not return normally or a memory access faults, earlier calls and writes may already have taken effect. The explicit return table describes normal completion of the relevant operations, not all possible execution outcomes.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-a8da4de9bd1d | Receives the selector and first temporary-region address. Supplies the acquisition status used after the transformation chain. |
| F-1b1da0aa8204 | Receives the second and first temporary-region addresses, in that order. Its return is not inspected. |
| F-a8474adba0ce | Receives the second region, a configured argument, and the third region. Its return is not inspected. |
| F-541356c82e59 | Receives component-storage and third-region addresses. Its return is not inspected. |
| F-a56604637c76 | Receives one component byte per call and supplies a value narrowed for an output write. |
| D-01 | Receives a freshly read state-field value and a configured reference argument. Only its zero/nonzero result controls this function. |
| D-02 | Receives the output address, configured source address, and length 12. Its return is ignored. |

D-01 and D-02 are locally assigned opaque identifiers. Their call shapes do not establish comparison, copying, overlap handling, validation, or failure semantics.

## Acceptance examples

These are **conditional static expectations**, not executed tests. Unless stated otherwise, they assume dependencies return normally, required memory is accessible, and no additional dependency effects or aliasing alter the stated observations.

1. **Zero selection field:** If the separately read selection field is zero, acquisition is selected for any selector. Neither D-01 nor D-02 is called.

2. **Forced acquisition still reads state:** With a nonzero selector, the initial selection-field read still occurs. If that read succeeds, acquisition is selected without D-01. F-a8da4de9bd1d receives the selector unchanged.

3. **Fresh argument after a nonzero test:** Suppose the initial selection-field read yields 7 and the selector is zero. If the field changes to zero before the fresh argument read, D-01 receives zero, not 7. Its result still determines the branch; the function does not reapply the initial zero-field shortcut to this fresh value.

4. **Distinct selection and reset bases:** Suppose the entry capture obtains address A, the initial selection read uses address B, and D-01’s fresh field read uses address C. If acquisition succeeds, the final reset targets A plus 4, regardless of B, C, or the then-current configured base.

5. **Alternate selection:** With a nonzero initial selection field, selector zero, and D-01 returning zero, D-02 receives the output address, configured source address, and length 12. F-a4ea0691ddc5 returns zero after D-02 returns, without invoking the acquisition chain.

6. **Distinguished failure:** If acquisition returns `0x15`, all three subsequent transformations still run. If output byte 11 is `0xAF` when the failure read occurs, it is written as `0xAC`. The return is `0xFFFFFFFA`; no component conversions or direct state reset occur.

7. **Other failure:** If acquisition returns 1, the same mask is applied, but the return is `0xFFFFFFFB`.

8. **Successful conversion:** Conversion results of 24, 12, 31, 23, 59, and 58 in call order produce a 16-bit value of 2024 at offset 0 and bytes 12, 31, 23, 59, and 58 at offsets 2, 3, 8, 9, and 10. The captured state field is then reset.

9. **Width boundaries:** A first result whose retained 16-bit pattern is `0xFFFF` becomes signed −1 before addition, producing 1999. A retained pattern of `0x7FFF` produces the stored pattern `0x87CF` after addition. A later result with low bits `0x0123` stores byte `0x23`. None triggers a direct range error.

10. **Unwritten success bytes:** Without relevant aliasing or indirect effects, successful acquisition preserves output offsets 4–7 and 11, including both low bits of byte 11.

## Uncertainties

The following remain **documented_only**:

- Component meanings, units, encoding, valid ranges, and calendar interpretation.
- The actual conversion performed by F-a56604637c76.
- Dependency return widths beyond the values retained or tested here.
- D-01’s comparison or decision semantics.
- D-02’s access extent, transfer semantics, overlap support, and failure behavior.
- Complete temporary and component-storage layouts and dependency access extents.
- Dependency allocation, release, retention, ownership, and global-state behavior.
- Native byte order and precise alignment requirements.
- Synchronization requirements and complete outcomes under concurrent mutation.
- Domain meanings of status `0x15` and the two negative return values.

Unknown concurrent behavior does not erase the established repeated-read structure: the reset base is captured first, the selection base is read separately, and D-01’s base and field are freshly read if that call is reached.

## Evidence

The supplied static representation establishes the distinct state-base reads, short-circuit branch structure, dependency argument relationships, call order, delayed status check, component-read and output-write order, retained widths, failure mask, captured-base reset, and explicit returns.

Dependency bodies, authoritative data-type definitions, runtime traces, and executable observations were not supplied. No runtime tests have been performed. The acceptance examples are derived expectations under their stated conditions, not observed results.
