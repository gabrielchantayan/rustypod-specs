<!-- Trusted provenance: F-26cba1b9b1be; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-26cba1b9b1be

## Purpose

F-26cba1b9b1be publishes a 32-bit status, invokes F-438680ffebfe with a branch-dependent address, and writes a one-byte Boolean result. Its normal return distinguishes the status bit pattern representing −1 from every other 32-bit status.

The function evaluates a global record-pointer slot twice before publishing the status:

1. It obtains and retains a record address.
2. It separately evaluates the slot to obtain the address used for the status read.

These are distinct source-level evaluations, not evidence of two physical machine loads. Without a stability assumption, the status-source address is not established to equal the retained address. The retained address supplies the non-sentinel dependency argument and the post-call flag evaluation; the status read through the separately obtained address determines the branch and return value.

The application meanings of the status, Boolean result, record contents, and dependency operation are not established.

## Interface

The function accepts four arguments and returns a 32-bit value.

| Argument | Established contract |
|---|---|
| First | An opaque 32-bit value. This function does not visibly inspect, dereference, or forward it. |
| Second | An address receiving a 32-bit status write before either branch calls the dependency. |
| Third | An address receiving a one-byte write of zero or one. The write precedes the dependency on the sentinel branch and follows it on the non-sentinel branch. |
| Fourth | An opaque 32-bit value passed unchanged as the first argument of F-438680ffebfe. |

The second and third arguments must designate writable storage when their respective writes occur. No visible validation establishes that condition.

The function’s normal returns are:

- **0:** the initially obtained status equals the 32-bit representation of −1.
- **1:** the initially obtained status has any other 32-bit value.

These return values do not report dependency success. The dependency’s result, if any, is ignored.

### Referenced storage

The supplied representation establishes these uses, without establishing a complete record type:

| Storage | Relative location | Established use |
|---|---|---|
| Global record-pointer slot | Not applicable | Evaluated first to retain an address, then separately to obtain the status-source address. |
| Status-source record | Offset 0 | A 32-bit status is read before the status output is written. |
| Retained record | Offset 4 | On the non-sentinel branch, a 32-bit integer is evaluated after the dependency returns and converted to a character for a zero/nonzero decision. |
| Retained record | Offset 5 | On the non-sentinel branch, this address is supplied as the dependency’s second argument. |
| Distinct fixed object | Object address | On the sentinel branch, its address is supplied as the dependency’s second argument. |

The two record roles may refer to the same storage, but that equality requires an additional assumption. A stable global slot during the two initial evaluations is sufficient; stability is not established by the function itself.

The supplied types indicate four-byte scalar arguments and return values, 32-bit integers, and a one-byte Boolean output. They do not establish the complete record extent, target byte order, alignment rules, physical load widths, or calling convention. The fixed object’s size and contents are unknown.

## Behavior

### Initial observations and status publication

The visible source-level sequence is:

1. Evaluate the global pointer slot and retain the resulting record address.
2. Separately evaluate that slot to obtain the address from which the status will be read.
3. Read the 32-bit status at offset 0 of that status-source record.
4. Write the obtained status through the second argument.
5. Select a branch by comparing the obtained status with −1.

The status is not read through the retained address merely because that address was obtained first. It is read through the result of the second slot evaluation.

This distinction does not specify a concurrency mechanism or guarantee that concurrent changes are observable in any particular way. It records the source-level evaluation structure. Whether a compiled implementation uses one physical slot load, two loads, or another equivalent arrangement is not established.

The branch decision uses the status already obtained, not a subsequent read of the status destination or either record. Therefore, assuming execution remains valid:

- The status-output write cannot change the selected branch by altering the already obtained status.
- A dependency modification of either record’s status does not change the selected branch.
- A dependency modification of the status output does not change the function’s return selection.
- If the two initial slot evaluations produce different addresses under an execution model that permits this, the branch follows the second address’s status, while later non-sentinel record operations use the first address.

There are no visible null, bounds, alignment, or output-pointer checks before the initial reads and status write.

### Sentinel branch

If the obtained status equals −1:

1. The status destination has already received that value.
2. The function writes zero through the third argument as a one-byte value.
3. It calls F-438680ffebfe with the fourth argument unchanged and the fixed object’s address.
4. If the dependency returns normally and subsequent execution completes normally, the function returns 0.

This branch does not visibly evaluate the retained record’s offset-4 integer or compute its offset-5 address.

The function performs no post-call restoration of either output. If the dependency changes the byte destination through accessible storage, the byte present at return need not be zero. Similarly, the status destination need not retain −1 after dependency activity or overlapping writes.

The initial retained-address evaluation still occurs on this branch. Its presence does not imply that this branch dereferences the retained address: the initial status dereference instead uses the separately evaluated status-source address.

### Non-sentinel branch

If the obtained status is any 32-bit value other than −1:

1. The status destination has already received the obtained status.
2. The function calls F-438680ffebfe with the fourth argument unchanged and an address calculated from the retained record address plus five bytes.
3. After the dependency returns, the function evaluates the 32-bit integer at offset 4 of the retained record.
4. It converts that integer to a character and tests the converted value for zero.
5. It writes zero through the third argument if the converted value is zero, or one otherwise.
6. If execution completes normally, it returns 1.

The byte output is normalized by this function’s post-call write. It is not a copy of the full offset-4 integer.

The post-call evaluation uses the retained address from the first global-slot evaluation. It does not use the status-source address merely because that address supplied the branch status, and it does not obtain a new address from the global slot after the call.

If the dependency redirects the global slot, the continuation still refers to the retained record. Whether that record remains allocated, accessible, and suitable for the offset-4 evaluation is not established. Those conditions are necessary for normal continuation; dependency preservation cannot be assumed.

### Widths, conversion, and address arithmetic

The sentinel comparison concerns the complete 32-bit status. Zero, positive values, and negative values other than −1 all select the non-sentinel branch.

The Boolean decision follows conversion of a 32-bit integer to a character. Under an eight-bit truncating conversion model, zero versus nonzero depends only on the integer’s least significant eight bits. Thus, an integer value of 256 can produce a zero Boolean output.

That conversion model is a qualification, not an independently established target property. The supplied material does not determine the exact character-conversion rules or identify a particular physical record byte as the flag. Character signedness does not change zero versus nonzero once the resulting eight-bit pattern is fixed.

The dependency address is five **bytes** beyond the retained base, not five integer elements. The supplied representation converts the retained pointer to a 32-bit integer form before adding five. Pointer-conversion behavior and arithmetic outside the representable range are not established. No modulo wraparound or overflow recovery is promised.

Normal-operation examples assume the conversion and addition produce the intended usable address. Dependency requirements at that address remain unknown.

## Side effects

At the source level, the function explicitly performs:

- Two ordered evaluations of the global record-pointer slot.
- One 32-bit status read through the address obtained by the second evaluation.
- One 32-bit write through the second argument.
- One one-byte write through the third argument.
- One dependency invocation on either normally reached branch.
- On the non-sentinel branch only, one post-call integer evaluation at offset 4 of the address retained by the first slot evaluation.

These counts describe visible source operations, not physical memory transactions, atomicity, or compiler-generated access widths.

There is no explicit assignment to the global slot, allocation, deallocation, or ownership transfer. Output-pointer aliasing can nevertheless modify storage otherwise described as a record, the global slot, or another object.

The function does not visibly reject overlapping destinations. Relevant ordering constraints are:

- The status read precedes the status-output write.
- The status-output write precedes the dependency on both branches.
- On the sentinel branch, the zero-byte write precedes the dependency.
- On the non-sentinel branch, the offset-4 evaluation and Boolean write follow the dependency.
- The non-sentinel offset-4 evaluation precedes that branch’s Boolean write.

Consequently, a status-output destination overlapping the retained record’s offset-4 field can influence the later Boolean result. A Boolean destination overlapping the status destination can change part of the previously written status. Such effects depend on valid aliasing, layout, byte order, and the applicable execution model.

The dependency’s side effects are not constrained by the caller’s visible body. It may affect accessible outputs, records, the global slot, or memory lifetime; whether it actually does so is unknown.

No visible recovery path handles invalid memory access, unusable address conversion, or a dependency that fails to return normally. Such events do not establish a normal return value.

## Dependencies

### F-438680ffebfe

This is the only visible called function.

| Branch | First dependency argument | Second dependency argument |
|---|---|---|
| Obtained status equals −1 | Fourth argument, unchanged | Fixed object’s address |
| Obtained status differs from −1 | Fourth argument, unchanged | Address retained by the first global-slot evaluation, advanced by five bytes through the represented integer-form calculation |

The dependency’s return value is not used. It neither selects the branch nor supplies the Boolean result directly.

The following dependency properties are **documented_only**:

- Meaning and validity requirements of its first argument.
- Whether its second argument identifies text, binary content, or another representation.
- Required accessible extent and alignment.
- Whether it reads, writes, retains, copies, or releases addressed storage.
- Whether it changes the global slot, either relevant record, or either output.
- Whether retained-record storage remains valid after the call.
- Error reporting, exceptions, termination, and normal-return guarantees.

No ownership or preservation guarantee follows merely from passing an address.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Unless stated otherwise, accesses and conversions are valid, relevant storage is disjoint, address arithmetic is representable, and the dependency returns normally without modifying the outputs. Examples using a single record additionally assume the two initial global-slot evaluations yield the same address.

1. **Sentinel status.**  
   If the status-source record contains −1, the status output receives −1, the byte output receives zero before the dependency, the dependency receives the fixed-object address, and the function returns 0.

2. **Zero status.**  
   With status zero and a post-call offset-4 integer whose character conversion is zero, the dependency receives retained base plus five, the byte output becomes zero after the call, and the return is 1.

3. **Other negative status.**  
   With status −2 and a post-call character conversion yielding nonzero, the status output receives −2, the byte output becomes one, and the return is 1.

4. **Nonzero full integer producing false.**  
   Under eight-bit truncating conversion semantics, a post-call offset-4 integer of 256 produces byte output zero. The function still returns 1 when the obtained status is non-sentinel.

5. **Different initial slot results.**  
   Conditionally, if the applicable execution model permits the first evaluation to yield one valid record and the second to yield another, a non-sentinel status from the second record controls the branch and status output. The dependency address and post-call flag evaluation use the first record. This example does not promise defined behavior for unsynchronized concurrent mutation.

6. **Different initial records with sentinel status.**  
   Under the same conditional model, if the second record supplies −1, the function takes the sentinel branch regardless of the first record’s contents. It passes the fixed-object address, not either record-relative address.

7. **Dependency changes the retained flag field.**  
   On a non-sentinel call, if the dependency changes the retained record’s offset-4 integer from a value converting to zero to one converting to nonzero, the function subsequently writes one.

8. **Dependency redirects the global slot.**  
   If the dependency selects a different record globally but leaves the retained record valid, the post-call flag evaluation still uses the retained record. There is no post-call slot evaluation in the visible body.

9. **Dependency changes the sentinel byte output.**  
   If the dependency changes the previously written zero byte to seven, the function does not normalize it again. With no other writes, it returns 0 leaving seven at that destination.

10. **Dependency changes the status output.**  
    If the dependency overwrites the status destination, the function does not restore it. Its return remains determined by the status obtained before the original output write.

## Uncertainties

**Established static evidence:** the two ordered global-slot evaluations; retention of the first address; the status read through the second address; output-write ordering; sentinel comparison; dependency argument selection; post-call flag evaluation through the retained address; and normal return values.

**Inferred intent:** the status may represent availability or validity, the fixed object may supply fallback content, and the retained record’s offset-5 address may identify associated content. None of these interpretations is required by the contract.

**Documented_only unknowns:**

- Application meanings of status, flag, and dependency arguments.
- Whether the global slot is stable across its initial evaluations.
- Whether the two resulting addresses identify the same record.
- Global initialization, record ownership, and lifetime.
- Full storage layouts, extents, and alignment.
- Target byte order and exact character-conversion semantics.
- Pointer-to-integer conversion and out-of-range arithmetic behavior.
- Physical load counts, access widths, and atomicity.
- Dependency side effects and failure behavior.
- Concurrency, synchronization, and behavior during concurrent mutation.

No thread-safety guarantee is established. Distinguishing the two source-level slot evaluations does not resolve concurrent-access semantics.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. It separates explicit caller operations from unknown dependency behavior and separates source-level evaluations from physical machine accesses.

No implementation was executed. No runtime tests, memory traces, dependency inspection, or target-platform validation were performed. The acceptance examples are conditional specifications, not observed results.
