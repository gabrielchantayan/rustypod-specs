<!-- Trusted provenance: F-fea5fa804e19; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fea5fa804e19

## Purpose

F-fea5fa804e19 performs an ordered, short-circuit selection using up to three calls to F-efdae002d289. It supplies each call with a candidate value and an address within an implicitly supplied context.

The first dependency call that returns a nonzero result determines the outcome. That outcome causes a 32-bit read-modify-write of a context field and a return value of 5, 6, or 7. If all three dependency calls return zero, the function instead stores an implicitly supplied 32-bit word in another context field and returns 13.

This is a structural description, not a semantic interpretation of the dependency. The evidence does not establish whether nonzero means a match, successful parsing, recognition, or another condition. Similarly, the context word modified on the first three outcomes has a flag-like layout, but its application-level meaning is unknown.

## Interface

### Explicit input

The function has one visible explicit argument:

| Input | Representation | Established use |
|---|---|---|
| Primary candidate | Opaque 32-bit word | Passed unchanged as the first argument of the initial call to F-efdae002d289 |

The input is not directly dereferenced, compared, or transformed by F-fea5fa804e19. Its interpretation belongs to the dependency and is undocumented here.

### Implicit inputs

The visible behavior also requires values not represented in the explicit argument list:

| Input | Established use |
|---|---|
| Context base | Supplies the base address for dependency arguments and direct context accesses |
| Fallback word | Supplies the 32-bit value stored when all dependency calls return zero |
| First fallback candidate | Loaded from a fixed storage location when the initial call returns zero |
| Second fallback candidate | Loaded from a different fixed storage location when both preceding calls return zero |

The mechanism by which the context base and fallback word arrive is not established as a public calling convention. They must not be treated as ordinary explicit parameters merely because the function needs them.

The fallback candidates are values read from storage, not proven literal constants. Their actual values, interpretation, mutability, and ownership are unknown.

### Output

The function returns one opaque 32-bit word. On ordinary completion, the visible logic selects exactly one of these values:

| Condition | Return value |
|---|---:|
| Initial dependency result is nonzero | 5 |
| Initial result is zero; first fallback result is nonzero | 6 |
| First two results are zero; second fallback result is nonzero | 7 |
| All three results are zero | 13 |

There is no distinct visible error return. A negative dependency result is nonzero and therefore selects the corresponding 5, 6, or 7 outcome rather than being propagated as an error.

### Context layout

Only the following relative locations are established:

| Byte offset from context base | Access by this function |
|---:|---|
| `0x1C` / 28 | Address passed as the second argument to every dependency call |
| `0x28` / 40 | 32-bit word read and written on a nonzero-result outcome |
| `0x38` / 56 | 32-bit word written only when all results are zero |

Passing the address at offset 28 does not establish the size or type of the object at that address. The dependency may access a region whose extent is unknown.

## Behavior

### Ordered dependency evaluation

The function first calls F-efdae002d289 with:

- The explicit primary candidate.
- The context address at byte offset 28.

If this call returns nonzero, no fallback candidate is read or submitted by the visible logic. The function proceeds directly to the first context update and returns 5.

If the initial call returns zero, the function obtains the first fallback candidate from its fixed storage location and calls F-efdae002d289 with that value and the context address at offset 28.

If the second call returns nonzero, the second fallback candidate is not read or submitted by the visible logic. The function performs the second context update and returns 6.

Only when both preceding results are zero does the function obtain the second fallback candidate and make the third call. A nonzero third result selects the third context update and return value 7. A zero third result selects the fallback-word store and return value 13.

Thus, ordinary execution makes exactly one, two, or three dependency calls. A later call is conditional on every earlier result being zero. The visible body contains no retry loop.

### Updates selected by a nonzero result

For each of the first three outcomes, the function reads the 32-bit word at context offset 40 after the selected dependency call has returned. It then writes a transformed 32-bit value back to the same location.

Let the value actually read at that point be \(W\). The stored value is:

\[
(W \mathbin{\&} \texttt{0xFFFF000F}) \mathbin{|} S
\]

where the selection value \(S\) is:

| Selected call | Selection value | Forced set bit |
|---|---:|---:|
| Initial call | `0x00000040` | 6 |
| First fallback call | `0x00000080` | 7 |
| Second fallback call | `0x00000100` | 8 |

Bit numbering starts at zero for the least significant bit.

For all three transformations:

- Bits 0–3 retain their values from the word read after the dependency call.
- Bits 16–31 retain their values from that same read.
- Bits 4–15 are cleared before the selected bit is set.
- Exactly one of bits 6, 7, and 8 is set in the resulting word.
- Any previously set, unselected bits within bits 4–15 are lost.

This is replacement of a 12-bit region with a one-bit selection, not an operation that merely adds a new flag while retaining old selections.

The preservation guarantee is relative to the value read at the time of the update. It is not a guarantee that these bits retain their function-entry values: F-efdae002d289 may have modified the context before the read.

After the store, the function returns 5, 6, or 7 according to which call produced the nonzero result.

### Update when every result is zero

If all three dependency results are zero, the function writes the implicit fallback word as a 32-bit value at context offset 56 and returns 13.

On this path:

- The visible body does not read the previous word at offset 56.
- The visible body does not perform the read-modify-write at offset 40.
- No further dependency call is made after the third zero result.

These are restrictions on this function’s direct operations only. They do not rule out accesses or changes made by its dependency.

### Width and address constraints

The candidate argument, fallback word, returned value, and directly accessed context words are represented as 32-bit quantities in the supplied material. The dependency result is tested only for zero versus nonzero; its magnitude otherwise has no effect.

The bitwise transformation is a 32-bit operation. It does not involve signed arithmetic or arithmetic overflow.

The context offsets are byte offsets. A usable invocation requires the relevant derived addresses to be valid for their actual accesses. The supplied material does not establish pointer width, alignment rules, or the behavior of address arithmetic at a boundary. In particular, it does not justify treating an overflowing context address as a valid wrapped address.

## Side effects

### Direct effects

On ordinary completion, the visible body performs exactly one direct 32-bit context store:

- A store at offset 40 following the first nonzero dependency result; or
- A store at offset 56 after three zero results.

The offset-40 store is preceded by a read of the same word. The offset-56 store has no corresponding direct read.

Fallback-candidate storage is read conditionally and in order: the first candidate only after the initial zero result, and the second only after two zero results. There is no visible write to either candidate’s storage.

No direct allocation, deallocation, ownership transfer, logging, or external I/O operation is shown.

### Dependency-mediated effects

F-efdae002d289 receives a context-derived address. Its memory accesses, global changes, allocation behavior, ownership behavior, and external effects are unknown.

Consequently:

- Earlier calls may affect later calls.
- Context contents may differ from their entry values before the final store.
- A dependency call may affect a fallback candidate’s storage before that candidate is read.
- No transaction or rollback guarantee is established.
- The absence of a direct write by this function does not establish that the corresponding memory is unchanged.

If a dependency call does not return normally, the evidence supplies no completed return value or final direct store for that execution.

### Ownership and lifetime

The function visibly uses existing values and storage. It neither establishes ownership of them nor visibly releases them.

The context and any dependency-required objects must remain usable for the operations actually performed. Whether the dependency retains references, consumes candidate values, or releases associated resources is documented_only.

## Dependencies

### F-efdae002d289

**Observed interface:** two arguments and a result tested for zero versus nonzero.

**First argument:** the primary candidate, then conditionally the first fallback candidate, then conditionally the second fallback candidate.

**Second argument:** the address at context offset 28 on every call.

**Ordering:** calls are sequential and short-circuited at the first nonzero result.

**Unknown contract:** argument types beyond their visible representation, pointed-to layout, return-value meanings, failure behavior, global preservation, memory ownership, and side effects.

A conforming account of F-fea5fa804e19 must preserve the distinction between the dependency’s raw zero/nonzero result and this function’s own return codes. Those codes are selected locally; they are not forwarded dependency results.

## Acceptance examples

These are conditional contract examples, not reports of executed tests. They assume normal dependency return and valid memory accesses.

1. **Initial selection.**  
   Given a nonzero first result and a post-call word of `0xABCDFFFF` at offset 40, the function makes one dependency call, writes `0xABCD004F` at offset 40, and returns 5. Neither fallback candidate is read by the visible body.

2. **First fallback selection.**  
   Given results zero and then nonzero, with `0x12345678` read at offset 40 after the second call, the function writes `0x12340088` there and returns 6. There is no third call.

3. **Second fallback selection.**  
   Given results zero, zero, and then nonzero, with `0xFFFF00F3` read at offset 40 after the third call, the function writes `0xFFFF0103` there and returns 7.

4. **No selection.**  
   Given three zero results and a fallback word of `0xDEADBEEF`, the function writes `0xDEADBEEF` at offset 56 and returns 13. It performs no direct offset-40 read-modify-write.

5. **Negative result.**  
   A first dependency result of −1 selects the same branch as any other nonzero first result: return 5 and the offset-40 update using `0x40`. The result is not propagated.

6. **Dependency changes the preserved bits.**  
   If a dependency changes the word at offset 40 before returning nonzero, the transformation uses the changed word. Entry-time bits are not restored.

7. **A fallback changes before use.**  
   If an earlier dependency call changes the first fallback candidate’s storage and returns zero, the subsequent candidate read observes the then-current stored value, subject to the underlying memory model. The visible logic does not capture both fallback candidates in advance.

8. **Existing selection bits are removed.**  
   If the selected update reads `0x000001C0` and the initial call was selected, the stored value is `0x00000040`, not `0x000001C0`.

## Uncertainties

The following remain documented_only:

- The semantic purpose of the candidates and context.
- How callers supply the implicit context base and fallback word.
- The full context size, alignment, and object layout.
- The type and extent of the object beginning at offset 28.
- The concrete fallback candidate values.
- The dependency’s side effects, ownership rules, and global-state guarantees.
- Pointer width, endianness, and unaligned-access behavior.
- Behavior under invalid pointers, inaccessible candidate storage, concurrent mutation, or abnormal dependency termination.
- Whether the surrounding execution environment imposes additional calling-convention requirements.

No null check, bounds check, validation step, synchronization operation, or recovery path is visible. Their absence does not establish a particular fault mode; it means this function provides no visible handling for those conditions.

## Evidence

**Established static evidence:** three ordered, conditional calls to F-efdae002d289; zero/nonzero branch decisions; context-relative offsets 28, 40, and 56; 32-bit direct memory operations; mask `0xFFFF000F`; selection values `0x40`, `0x80`, and `0x100`; and return values 5, 6, 7, and 13.

**Inferred intent:** the three nonzero outcomes appear to encode a mutually exclusive selection in a context word. The all-zero outcome appears to record a fallback value elsewhere in the context. Neither interpretation supplies additional behavioral guarantees.

**Verification status:** this contract is based solely on the supplied static material. No runtime tests have been performed.
