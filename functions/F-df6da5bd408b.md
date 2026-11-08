<!-- Trusted provenance: F-df6da5bd408b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-df6da5bd408b

## Purpose

F-df6da5bd408b updates a byte-sized state in a caller-supplied object and conditionally performs one of two transition paths:

- A request equal to zero can initiate processing associated with a linked object.
- A nonzero request can initiate processing associated with an attached object and subsequently clear that attachment.

The requested state is compared as a full 32-bit unsigned value but stored as one byte. Consequently, requests outside the byte range do not behave like requests normalized to their low byte.

The return value is path-dependent. It may be the existing state byte, zero, an object field, or the first word of a temporary context after dependency calls. It is not established as a uniform success code or object handle.

These behaviors are established only by inspection of the supplied source representation. Semantic labels such as “state,” “linked object,” and “attachment” are descriptive conveniences, not independently verified application meanings.

## Interface

### Inputs

| Input | Represented width | Contract |
|---|---:|---|
| Primary object address | 32 bits | Identifies storage containing the fields accessed below. No null or bounds check is shown. |
| Requested state | 32-bit unsigned | Compared without truncation against the existing byte. Its low eight bits are stored when the comparison differs. |
| First supplemental value | 32 bits | Copied unchanged into a temporary context on the zero-request linked-object path. |
| Second supplemental value | 32 bits | Copied unchanged into the same context. |

The supplemental values are opaque. Their interpretation as integers, addresses, flags, or other data is unknown.

### Primary object layout

Offsets are byte offsets from the primary object address.

| Offset | Access width | Observed role |
|---|---:|---|
| `0x20` | Address passed onward | Embedded region supplied to F-ddca41e891a8. Its extent is unknown. |
| `0x3D` | 8 bits | Stored state byte; read first and conditionally overwritten. |
| `0x48` | 32 bits | Tested against zero; its address can be passed to F-9e4be868dfec. |
| `0x54` | 32 bits | Attachment value, sometimes interpreted as an address; conditionally cleared. |
| `0x58` | 32 bits | Determines whether attachment processing proceeds; can be returned directly. |
| `0x60` | 8 bits | Enables further transition processing after a state change. |
| `0x78` | 32 bits | Linked-object address used by the zero-request path. |
| `0xF8` | 8 bits | Selects between a dependency call and indirect dispatch. |

### Other layouts

For a nonzero linked-object address:

| Offset | Access or use |
|---|---|
| `0x34` | Read as a 32-bit auxiliary address. |
| `0x80` | Used as an address passed to dependencies; not directly dereferenced by this function. |
| `0xA0` | Read as an 8-bit condition. |

A temporary, four-word context is constructed in this order:

| Context offset | Initial contents |
|---|---|
| `0x00` | Linked-object address |
| `0x04` | Linked-object address plus `0x80` |
| `0x08` | First supplemental value |
| `0x0C` | Second supplemental value |

The supplied representation places these words consecutively and passes the address of the first word to several dependencies. Their intended aggregate type and the dependencies’ permitted access extent are not established.

### Output and validity requirements

The result is a 32-bit unsigned value. Byte-derived results are zero-extended.

All executed memory accesses must be valid. In particular, conditional paths can require storage through offset `0xF8` in the primary object, through offset `0xA0` in the linked object, and valid dispatch-table storage for an attachment.

Address additions use the represented 32-bit address model. The function shows no overflow, alignment, or range checks. Valid execution requires resulting addresses to designate appropriate storage; arithmetic wraparound is not a supported object-boundary mechanism.

## Behavior

### Initial comparison and state update

The first observable object access reads the byte at `0x3D`. That byte is zero-extended and compared with the entire requested state.

If they are equal:

- No write to the state byte occurs.
- The enable byte at `0x60` is not read.
- No dependency is called.
- The existing byte value is returned as a 32-bit unsigned result.

If they differ:

1. The low eight bits of the requested state are written to `0x3D`.
2. The enable byte at `0x60` is read.
3. If that byte is zero, the function returns zero immediately.

The state write therefore precedes every subsequent enable check, dependency call, and transition-specific return. There is no rollback when later processing does nothing or returns zero.

If the enable byte is nonzero, the full requested state selects the remaining path. Only an exactly zero request selects the zero-request path.

For example, a request of `0x100` stores zero but still selects the nonzero-request path.

### Zero-request path

The function first reads the 32-bit field at primary-object offset `0x54`.

- If it is nonzero, neither F-ddca41e891a8 nor F-658efc125080 is called at this stage.
- If it is zero, F-ddca41e891a8 is called with the address of the primary object’s region at `0x20`.
- Only if that call returns zero is F-658efc125080 then called with the primary object address.

The linked-object field at `0x78` is read after these conditional calls. Consequently, a dependency’s changes to that field can affect which linked object is subsequently selected. The initial attachment condition is not rechecked before continuing.

If the linked-object field is zero, the function returns zero.

Otherwise, the function constructs the four-word temporary context described above, using the linked-object address just read and both supplemental values. It then calls F-3fac671648b8, with no explicit arguments shown.

Further processing requires three conditions, evaluated in this order:

1. F-3fac671648b8 returned nonzero.
2. The byte at linked-object offset `0xA0` is zero.
3. A byte reached through an external global pointer is nonzero.

The latter two reads are short-circuited: the linked-object byte is not read if the dependency returns zero, and the global byte is not read if the linked-object byte is nonzero.

If any condition fails, no subsequent dependency in this path is called. The first word of the temporary context is returned.

If all three conditions pass:

1. The 32-bit auxiliary address at linked-object offset `0x34` is read and retained.
2. F-e57b05520b65 is called with:
   - The address of the temporary context.
   - The linked-object address plus `0x80`.
   - That same address plus `0x80` again.
3. If the retained auxiliary address is nonzero, F-ee8733a08210 is called with the context address and the retained auxiliary address plus `0xD0`.
4. F-ffe0d92c57b7 is called with the linked-object address.
5. If its result is nonzero, F-c9b5d17d8d1e is called with that result and the context address, followed by F-4d037ecf0f82 with the same two arguments.
6. The current first word of the temporary context is returned.

The auxiliary address used in step 3 is the value read before F-e57b05520b65. It is not refreshed after that call. Likewise, the linked-object address remains the selected value rather than being reread from the primary object.

Calls receiving the context address can affect the eventual result by modifying its first word. No restoration of the original linked-object address is shown.

### Nonzero-request path

The function reads the attachment field at `0x54`.

If that field is zero, it returns zero without reading `0x58`.

If the attachment is nonzero, the function reads the 32-bit field at `0x58`.

If `0x58` is nonzero, that field is read again for the return value, and the function returns it. No attachment processing or clearing occurs on this branch. In ordinary stable memory, the two reads agree; the supplied representation does not establish atomicity or synchronization.

If `0x58` is zero:

1. The 32-bit field at `0x48` is read.
2. If it is nonzero, F-9e4be868dfec is called with the primary object address and the address of its field at `0x48`.
3. The attachment field at `0x54` is read again and retained.
4. The byte at `0xF8` is read.
5. If that byte is zero, F-6fdcf5f1105e is called, with no explicit arguments shown.
6. Otherwise, if the retained attachment is nonzero, an indirect call is made through that attachment’s dispatch structure.
7. The primary object’s field at `0x54` is overwritten with a 32-bit zero.
8. The function returns zero.

For indirect dispatch, the retained attachment is interpreted as an address. Its first 32-bit word supplies a dispatch-table address, and the callable target is read from that table at byte offset `4`.

The call’s complete ABI-level argument contract is not established by the supplied representation. In particular, no explicit attachment argument is shown.

F-9e4be868dfec can affect the attachment subsequently used because `0x54` is reread after that call. If it leaves the attachment zero and the flag at `0xF8` is nonzero, indirect dispatch is skipped, but the final zero write still occurs.

If a dependency or indirect target does not return normally, the later clear and return are not established to occur.

## Side effects

Direct writes performed by this function are limited to:

- One byte at primary-object offset `0x3D` when the initial comparison differs.
- Four bytes at primary-object offset `0x54` on the completed nonzero-request processing branch.
- Initialization of the temporary context on the zero-request linked-object path.

Dependencies may have additional effects. Nothing supplied establishes that they preserve the primary object, linked object, global storage, or temporary context.

The function does not directly show allocation or release of heap storage. Clearing `0x54` does not, by itself, prove destruction, deallocation, ownership transfer, or reference-count adjustment. Those effects may occur inside dependencies, but remain undocumented.

The temporary context is local to this invocation. Its address is passed to dependencies only during the call. Retaining and later using that address would require a lifetime mechanism not shown here.

The external global pointer and its target are read only on the specified gated path. This function shows no direct write to them.

There is no demonstrated locking, atomic access protocol, exception recovery, or transactional rollback.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-ddca41e891a8 | Receives primary-object address plus `0x20`; a zero result permits the next call. |
| F-658efc125080 | Receives the primary object address; its result is not used. |
| F-3fac671648b8 | Called after context initialization; nonzero result permits further conditions. No explicit arguments are shown. |
| F-e57b05520b65 | Receives the context address and two identical linked-object-derived addresses. |
| F-ee8733a08210 | Conditionally receives the context address and a retained auxiliary address plus `0xD0`. |
| F-ffe0d92c57b7 | Receives the linked-object address; a nonzero result becomes the first argument to two subsequent calls. |
| F-c9b5d17d8d1e | Receives the preceding nonzero result and the context address. |
| F-4d037ecf0f82 | Receives those same arguments after F-c9b5d17d8d1e returns. |
| F-9e4be868dfec | Receives the primary object address and the address of its field at `0x48`. |
| F-6fdcf5f1105e | Called when the later `0xF8` byte is zero; no explicit arguments are shown. |

An additional indirect dependency is selected through an attachment’s dispatch table. It has no supplied opaque ID.

No dependency’s ownership policy, global-preservation guarantees, or detailed effects are established.

## Acceptance examples

These are conditional acceptance criteria derived from static inspection, not executed tests. Unless stated otherwise, dependencies return normally and memory does not change asynchronously.

1. **Unchanged byte state.** With stored state `7` and requested state `7`, return `7`; do not read `0x60`, write the state, or call dependencies.

2. **Disabled transition processing.** With stored state `2`, request `0`, and enable byte zero, write zero to `0x3D` and return zero. Do not inspect transition-specific attachment or linked-object fields.

3. **Comparison is not byte-normalized.** With stored state zero, request `256`, enable byte nonzero, and attachment zero, perform the state-byte write, take the nonzero-request path, and return zero. The existing and newly stored bytes both being zero does not eliminate the transition.

4. **Zero request without a linked object.** With a changed, enabled zero request and attachment nonzero, skip the preliminary two dependencies. If `0x78` is zero when read, return zero.

5. **Preliminary dependency changes linkage.** With attachment zero, F-ddca41e891a8 returning zero, and F-658efc125080 changing `0x78`, use the linked-object value read after those calls, not an earlier snapshot.

6. **Short-circuited linked-object processing.** With a valid linked object and F-3fac671648b8 returning zero, do not read linked-object offset `0xA0` or the external global byte. Under the additional condition that the context remains unchanged, return the linked-object address.

7. **Retained auxiliary address.** If all gates pass and the auxiliary address initially read is nonzero, F-ee8733a08210 receives that retained address plus `0xD0`, even if F-e57b05520b65 changes the linked object’s auxiliary field.

8. **Context-derived result.** If a dependency receiving the context changes its first word to a particular 32-bit value and no later dependency changes it, return that value.

9. **Nonzero attachment with a nonzero return field.** With a changed, enabled nonzero request, nonzero attachment, and a stable `0x58` value of `19`, return `19`. Do not call attachment-processing dependencies or clear `0x54`.

10. **Attachment reread after processing.** With `0x58` zero and `0x48` nonzero, allow F-9e4be868dfec to replace the attachment. A subsequent nonzero `0xF8` flag selects indirect dispatch through the replacement, not the original attachment.

11. **Clearing follows the callback.** With `0x58` zero and `0xF8` zero, call F-6fdcf5f1105e before writing zero to `0x54`. If that call returns normally, clear the field even if the dependency assigned it another value.

## Uncertainties

- **documented_only — Application meaning:** The actual meanings of state values, enable status, attachments, linkage, and supplemental values are unknown.
- **documented_only — Dependency contracts:** Mutation, allocation, release, reentrancy, retained pointers, and global effects are not established.
- **documented_only — Calling convention:** Calls with no explicit arguments and the indirect call may involve ABI details absent from the representation.
- **documented_only — Context extent:** Four consecutive initialized words are visible; a formal aggregate declaration and callee access bounds are not.
- **documented_only — Failure behavior:** Invalid addresses are unchecked. Fault handling, exceptions, and nonlocal exits are unspecified.
- **documented_only — Concurrency:** Repeated reads are visible, but their concurrency semantics are not.
- **Inferred intent:** Updating a state byte and conditionally processing related objects resembles transition handling. That interpretation does not establish lifecycle or ownership semantics.
- **Representation limit:** Widths and ordering above describe the supplied representation, not independently verified machine execution.

## Evidence

The static basis is the supplied representation of F-df6da5bd408b and the supplied opaque dependency mapping.

Established observations include the full-width comparison against a zero-extended byte, truncated byte write, short-circuit branch ordering, conditional field rereads, four-word temporary initialization, dependency argument relationships, indirect dispatch lookup, final attachment clear, and path-dependent return values.

No implementation access, runtime execution, smoke checks, or tests were performed. The acceptance examples specify expected behavior under their stated conditions; they are not verification results.
