<!-- Trusted provenance: F-cd93a7610bac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cd93a7610bac

## Purpose

F-cd93a7610bac selects one of five construction paths using a name reference, processes an associated value reference through dependencies, and returns a tagged result when the selected path reports success.

The established result categories are:

- Tags **1**, **6**, and **2**: a payload reference obtained from one dependency, followed by an operation receiving that destination, the input value reference, and a dependency-produced 32-bit argument.
- Tag **8**: a payload reference returned directly by a value-processing dependency.
- Tag **7**: a payload operation supplied with a temporary sequence of four validated byte values.

These categories describe calls and result-field writes, not guaranteed payload contents. In particular, the supplied body does not establish that the payload operation copies, preserves, or otherwise reproduces its source.

Explicit failures return null. Failures after successful result acquisition invoke a cleanup dependency before returning. Allocation, copying, textual conversion, and destruction are plausible dependency roles, but their full contracts are not supplied.

## Interface

### Arguments

| Position | Width and interpretation |
|---|---|
| First | Opaque 32-bit value, forwarded unchanged to F-7f0efa3b3175. |
| Second | Opaque 32-bit value, forwarded unchanged to F-7f0efa3b3175. |
| Third | Address represented in 32 bits, identifying a description with readable 32-bit fields at byte offsets 4 and 8. |

The field at offset **4** supplies the name reference. The field at offset **8** supplies the value reference. The function reads them in that order, before checking whether the value reference is zero. It does not directly access the description’s offset-0 field.

The captured references are retained for subsequent operations. The description fields are not reread, including after dependencies run.

### Return and visible result layout

The return is either null or the non-null result reference obtained from F-7f0efa3b3175.

The function addresses these result fields:

| Byte offset | Width | Direct use |
|---|---:|---|
| 0 | 32 bits | Success tag: 1, 6, 2, 8, or 7. |
| 4 | 32 bits | Payload reference. |

The result’s total extent, initialization, and any additional fields are unknown. For a successful execution, the selected path must permit its result-field accesses.

The tag write is the final explicit construction operation before successful return. No explicit failure path writes a success tag. This does not establish that the tag was initially unset or that dependencies leave it unchanged.

For tags 1, 6, 2, and 7, dependencies run after the payload-field write. Without preservation guarantees, the payload field at return cannot be assumed to retain the value originally written there.

### Preconditions and limits

The initial description reads have no preceding null or accessibility check. The name reference is not checked for zero before comparison. A zero value reference receives an explicit diagnostic and null return; a nonzero reference is not otherwise validated here.

Requirements for readable name/value storage, encoding, termination, and accessible extent depend on the consuming dependencies. The visible body does not establish those requirements.

No separation between input storage and result storage is established. The acquired result may alias the description, name data, or value data. Therefore, input immutability is not guaranteed.

The supplied representation uses 32-bit words and references, signed 32-bit component checks, and one-byte temporary components. It does not establish a portable interface for a wider-pointer data model.

## Behavior

### Initial reads and acquisition

The function first captures the name reference, then the value reference.

If the captured value reference is zero, it calls F-4fc89ec41a6d with:

**(0x22, 0x75, 0x7C, 0, 0)**

It then returns null. It does not acquire a result, compare names, or invoke cleanup on this path.

Otherwise, F-7f0efa3b3175 receives all three original arguments in their original order. If it returns null, F-4fc89ec41a6d receives:

**(0x22, 0x75, 0x41, 0, 0)**

The function then returns null without invoking cleanup.

Changes made by result acquisition to the description fields do not replace the captured name and value references used by this function.

### Ordered selection

After acquiring a non-null result, the function compares the captured name against up to five fixed comparison objects using F-399ef0370885.

Each call receives the captured name reference first and the corresponding fixed-object reference second. A zero return selects that alternative; any nonzero return continues to the next alternative.

| Comparison position | Selected construction |
|---|---|
| First | Shared payload operation, tag 1 |
| Second | Shared payload operation, tag 6 |
| Third | Shared payload operation, tag 2 |
| Fourth | Direct dependency-produced payload, tag 8 |
| Fifth | Four-component path, tag 7 |

Selection is first-match. No later comparison occurs after selection.

The supplied representation associates the first object with an email-related label. This is evidence of likely intent, not proof of its bytes or an accepted public spelling. The other comparison-object contents are likewise unavailable. Equality semantics, case handling, encoding, and aliases remain dependency-defined.

If all five returns are nonzero, the function performs, in order:

1. F-4fc89ec41a6d with **(0x22, 0x75, 0x75, 0, 0)**.
2. F-968c88301966 with selector **2**, the fixed name-context label reference, and the captured name reference.
3. F-975dbf132545 with the acquired result reference.
4. Null return.

The context-label description identifies its apparent role, not guaranteed literal text.

### Shared construction for tags 1, 6, and 2

The first three alternatives differ only in the final tag.

The function calls F-f9450f0778b9 with **0x16**, decimal **22**. It writes the returned 32-bit reference into the result’s offset-4 field, even if that reference is zero.

A zero return takes the common construction-failure path described below.

For a nonzero return:

1. F-1cb585febd84 receives the captured value reference.
2. After that call returns, the function reads the result’s offset-4 field.
3. F-bbc440b3aefa receives that field value as destination, the captured value reference as source, and the 32-bit return from F-1cb585febd84 as its third argument.

The destination is thus a later field read, not necessarily the original return from F-f9450f0778b9. If F-1cb585febd84 changes the field through an alias or other reachable state, the changed value is passed. There is no second zero check on that reread destination.

The third argument is neither range-checked nor checked for zero. Its interpretation as a length or extent is inferred, not established.

A nonzero F-bbc440b3aefa return causes the selected tag to be written at offset 0, followed by return of the acquired result reference. A zero return takes the common construction-failure path.

Neither nonzero success nor the argument pattern establishes the resulting payload bytes, copying semantics, or source ownership.

### Direct dependency-produced payload: tag 8

The fourth alternative calls F-f3f24b910f0c with the captured value reference and a second argument of **0**.

If the return is nonzero, the function:

1. Writes that return into the result’s offset-4 field.
2. Writes **8** at offset 0.
3. Returns the acquired result reference.

If the return is zero, there is no explicit payload-field write on this branch. The function instead performs:

1. F-4fc89ec41a6d with **(0x22, 0x75, 0x77, 0, 0)**.
2. F-968c88301966 with selector **2**, the fixed value-context label reference, and the captured value reference.
3. F-975dbf132545 with the result reference.
4. Null return.

The dependency may itself have changed reachable storage before returning zero. Absence of an explicit payload-field write does not guarantee preservation.

### Four-component construction: tag 7

The fifth alternative calls F-8cfd5431cd53 with:

- The captured value reference.
- A fixed format-object reference.
- Four writable destinations corresponding to signed 32-bit local components, in first-through-fourth order.

The supplied body does **not** expose the format bytes or the dependency implementation. The format object’s label suggests four dotted-decimal conversions, but that is inferred intent. The established contract is the call shape and the caller’s checks, not a particular textual grammar or conversion-count convention.

The function first requires the dependency’s return to equal exactly **4**. If it does not, the function rejects the value without inspecting the four component destinations.

When the return equals 4, the function checks each component in order. For each component, it checks the signed lower bound **0** and then the upper bound **255**. Checks short-circuit at the first failure.

Consequently:

- −1 fails.
- 0 and 255 pass.
- 256 fails.
- A later component is not checked once an earlier failure establishes rejection.

Safe execution requires all four component destinations to contain defined values when inspected. The caller does not visibly initialize them before the dependency call, and return value 4 alone does not prove that the dependency wrote them.

A return-value or range failure produces, in order:

1. F-4fc89ec41a6d with **(0x22, 0x75, 0x76, 0, 0)**.
2. F-968c88301966 with selector **2**, the value-context label reference, and the captured value reference.
3. F-975dbf132545 with the acquired result.
4. Null return.

After all checks pass, the function narrows the components into four adjacent one-byte temporary values in component order. Every accepted value is exactly representable in one byte, so this narrowing preserves its numeric value. No multi-byte packing or byte-order conversion is visible.

It then calls F-f9450f0778b9 with **4** and writes the returned reference to the result’s offset-4 field, including a zero return.

For a nonzero return, F-bbc440b3aefa receives:

- That returned reference as destination.
- The address of the first temporary byte as source.
- **4** as the third argument.

A nonzero return from this operation causes tag **7** to be written and the acquired result to be returned. This establishes the temporary source bytes and supplied arguments, not the resulting payload contents.

A zero reference or zero payload-operation return takes the common construction-failure path. No value-context attachment occurs on those failures.

There is no separate full-input-consumption check. Any handling of trailing text, whitespace, signs, separators, overflow, or termination belongs to the unknown dependency contract.

### Common construction failure

A zero return from F-f9450f0778b9 or F-bbc440b3aefa on the shared or four-component paths causes:

1. F-4fc89ec41a6d with **(0x22, 0x75, 0x41, 0, 0)**.
2. F-975dbf132545 with the acquired result.
3. Null return.

There is no detail-attachment call on this path.

## Side effects

The directly visible effects are the initial input-field reads, dependency calls, result-field writes, and creation of the four-component temporary data.

No write is explicitly addressed through the input description, name, or value references. However, result-field writes may affect aliased input storage. Dependencies also receive input references and may modify reachable storage. Input immutability is therefore not established.

On explicit failure after acquisition, the acquired result reference is supplied once to F-975dbf132545, after the diagnostic calls. This is cleanup delegation, not proof that memory is released. The dependency may inspect the current result fields, whose contents can depend on earlier dependency effects.

On success, this function does not invoke cleanup. That does not establish ownership transfer, exclusive ownership, or the caller’s eventual release procedure.

No explicit transfer of input ownership is established. Dependencies may borrow, retain, transform, or release references according to contracts not provided here.

The four-byte source is temporary local storage. The function provides no persistent source buffer. Whether F-bbc440b3aefa consumes the bytes during the call or retains their address is unknown; retention would create a lifetime concern after return.

Diagnostic attachment, when present, follows the primary diagnostic and precedes cleanup. Global error state, emitted text, and post-cleanup storage state remain dependency-defined.

## Dependencies

| Opaque ID | Established use and boundary |
|---|---|
| F-7f0efa3b3175 | Receives the original three arguments. Its nonzero return becomes the result reference; zero means acquisition failure. Layout initialization and ownership are unknown. |
| F-399ef0370885 | Receives the captured name and a fixed comparison object. Zero selects the corresponding alternative. Comparison semantics are unknown. |
| F-f9450f0778b9 | Receives 22 or 4. Its return is written to the payload field and tested for zero. Allocation is inferred; units, initialization, and ownership are unknown. |
| F-1cb585febd84 | Receives the captured value. Its 32-bit return becomes the third payload-operation argument. Length measurement is inferred. |
| F-bbc440b3aefa | Receives destination, source, and a third 32-bit argument. Nonzero is accepted as success. Copying, transformation, retention, and preservation are not established. |
| F-f3f24b910f0c | Receives the captured value and zero. A nonzero return is stored as the tag-8 payload. |
| F-8cfd5431cd53 | Receives the value, a fixed format object, and four signed 32-bit destinations. The caller requires return 4 and bounded components. Dotted-decimal scanning is inferred, not an established dependency contract. |
| F-4fc89ec41a6d | Receives the specified five-argument primary diagnostic tuples. |
| F-968c88301966 | Receives selector 2, a context-label reference, and the rejected name or value reference. |
| F-975dbf132545 | Receives the acquired result on every explicit post-acquisition failure. Actual reclamation behavior is unknown. |

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes valid accessed storage and dependencies that return normally.

1. **Zero value.** An accessible description with value reference zero produces reason **0x7C** and null. Both initial fields are read; acquisition, comparison, and cleanup do not occur.

2. **Acquisition failure.** With a nonzero captured value and a null acquisition return, expect reason **0x41**, null, and no cleanup call.

3. **Shared-path success.** If the first comparison returns zero, the reference-producing call returns nonzero, and the payload operation returns nonzero, expect a final tag write of **1** and return of the acquired result. Later comparisons do not occur. Payload contents remain dependency-defined.

4. **Later shared alternatives.** Earlier nonzero comparisons followed by selection of the second or third alternative produce tags **6** or **2**, respectively, when the shared construction succeeds.

5. **Destination mutation.** If F-1cb585febd84 changes the result’s payload field through an alias, the shared-path payload operation receives that changed field value, even if zero. It does not necessarily receive the originally stored reference.

6. **Four-component success.** If the fifth alternative is selected and F-8cfd5431cd53 returns 4 while defining components as **0, 255, 1, 254**, the temporary source contains those four bytes in that order. With subsequent nonzero returns, expect tag **7** and the acquired result. Do not require matching payload contents without an additional dependency contract.

7. **Range rejection.** With return 4 and defined components, a component of **−1** or **256** produces reason **0x76**, value-context attachment, cleanup, and null. The four-unit reference request does not occur.

8. **Non-4 return.** Any dependency return other than 4 takes the numeric-input failure path without inspecting component destinations.

9. **Textual input is conditional.** No particular dotted string is unconditionally accepted by this evidence. If the dependency returns 4 and supplies four in-range components for an input, construction proceeds regardless of any trailing material not separately checked here.

10. **Tag-8 rejection.** A zero F-f3f24b910f0c return produces reason **0x77**, value-context attachment, cleanup, and null, without an explicit payload-field write by this branch.

11. **Unknown name.** Five nonzero comparison returns produce reason **0x75**, name-context attachment using the captured name reference, cleanup, and null.

12. **Payload-operation failure.** A zero F-bbc440b3aefa return produces reason **0x41**, cleanup, and null. This function does not write a success tag on that path.

## Uncertainties

The following remain **documented_only**:

- Exact comparison-object, format-object, and diagnostic-label bytes.
- Accepted name spellings and domain meanings of the five tags.
- Textual grammar and conversion semantics of F-8cfd5431cd53, including whether return 4 guarantees four initialized destinations.
- Allocation units and payload representation.
- Full result layout and initial field values.
- Dependency mutation of globals, input storage, result fields, or other aliased objects.
- Copying, transformation, source retention, ownership transfer, and release behavior.
- Required input extent, encoding, and termination.
- Behavior for invalid references, inaccessible memory, dependency faults, or nonlocal exits.

Likely allocation, length-measurement, copying, dotted-decimal scanning, and destruction roles are inferred intent only. The established behavior is the caller’s accesses, argument values, return-value tests, branches, and writes.

## Evidence

This contract is based on the supplied static function representation and opaque dependency mapping, not on the prior draft’s assertions.

Static evidence establishes the 32-bit field accesses, captured-reference reuse, ordered comparisons, dependency arguments, exact check against 4, signed component bounds, ordered byte narrowing, result-field writes, diagnostic ordering, and explicit null-return paths.

No dependency implementation or fixed-object bytes were supplied. Symbolic labels suggest intent but do not establish data contents or dependency semantics.

No runtime tests have been performed.
