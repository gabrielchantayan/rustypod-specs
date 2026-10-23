<!-- Trusted provenance: F-0e89b7dfee22; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0e89b7dfee22

## Purpose

F-0e89b7dfee22 performs a conditional lookup and returns a 32-bit value obtained through the lookup result.

The established behavior has three outcomes:

- If a 32-bit field in the supplied object is zero when first inspected, return zero without invoking the dependency.
- Otherwise, invoke F-a2a686313150. If its result is zero, return zero without reading through that result.
- If the dependency returns a nonzero result, follow a two-stage memory path from that result and return the final 32-bit value.

The likely intent is to retrieve a value associated with a signed selector from an object-associated lookup context. That interpretation is inferred, not established: the meanings of the objects, selector, lookup result, and returned bits are unknown.

Zero is not a uniquely distinguishable failure indication. It can result from either early-return condition or from a successful memory traversal whose final value is zero.

## Interface

### Inputs

The function has two explicit inputs:

| Input | Established representation | Use |
|---|---|---|
| Object address | 32-bit integer used as an address | Locates a 32-bit field at byte offset `0x1B0` |
| Selector | Signed 16-bit integer | Sign-extended to a signed 32-bit argument for F-a2a686313150 |

The first input is used as a memory address rather than as an ordinary numeric identifier. The supplied evidence does not establish a public object type, total object size, or allocation origin.

The selector’s signedness is behaviorally significant. Its effective range is −32768 through 32767. Conversion to the dependency argument preserves its signed numeric value; it does not reinterpret negative selectors as positive 16-bit values.

How a caller supplies values outside the signed 16-bit range is outside the established interface. The evidence does not define an external argument-marshalling rule for such values.

### Output

The function returns one 32-bit quantity:

- All-zero bits on either early-return path.
- Otherwise, the 32 bits read from the final memory location.

The return representation does not establish whether a nonzero result is an integer, address, handle, flags field, or another encoding. No signed interpretation of the final value is established.

There is no explicit status output, output buffer, or separate indication of which path produced the result.

### Required memory layout

The following locations are relevant, conditionally:

| Base | Byte offset | Access width | Role |
|---|---:|---:|---|
| Input object address | `0x1B0` | 32 bits | Initial condition and dependency’s first argument |
| Nonzero dependency result | `0x7C` | 32 bits | Address of a further record |
| Further record address | `0x04` | 32 bits | Returned value |

These are byte offsets, not element indices. The visible accesses do not establish any other fields, padding, record lengths, or ownership relationships.

For normal completion, each location actually accessed must support the corresponding 32-bit read. The evidence does not establish the target’s alignment requirements or behavior on invalid addresses.

## Behavior

### Initial condition

The function first reads the 32-bit field at object offset `0x1B0` and compares it with zero.

If that read yields zero:

- The function returns the all-zero 32-bit result.
- F-a2a686313150 is not called.
- Neither location derived from a dependency result is read.
- There is no visible second read of the object field on this path.

This is a check of the object’s field, not a check that the input object address itself is nonzero or valid. The function attempts the field read without a visible input-address guard.

### Dependency invocation

If the initial field read is nonzero, the visible expression reads the same 32-bit field again to obtain the dependency’s first argument.

This distinction matters: the evidence shows a condition read followed by an argument read, rather than a single captured value reused for both purposes. It does not establish concurrency synchronization, atomicity, or the behavior of concurrent mutation. Nevertheless, a behavioral account must not silently replace the two visible reads with an assumed stable snapshot.

F-a2a686313150 is invoked once on this branch, with:

1. The value obtained by the second read of the object field.
2. The selector converted from signed 16-bit to signed 32-bit.

The selector conversion requires no additional memory access described by this function. No observable ordering relative to that conversion, beyond supplying the converted argument to the call, is significant in the supplied evidence.

The dependency is not visibly called again, retried, or given an alternative selector.

### Dependency-result handling

If the dependency returns normally with a zero 32-bit result, this function returns zero. It does not attempt to read memory at an offset from that zero result.

If the dependency returns normally with a nonzero result, the function:

- Reads a 32-bit address at byte offset `0x7C` from the result.
- Reads a 32-bit value at byte offset `0x04` from that address.
- Returns the value from the second read.

The second read necessarily depends on the address produced by the first. No additional null check or validity check is visible between these reads.

In particular, a zero address obtained from the `0x7C` field does not trigger a defined zero-return path. The function still attempts the subsequent read at that address plus `0x04`. Whether that attempt succeeds, faults, or has another environment-specific consequence is unknown.

The input object field is not visibly consulted again after the dependency call.

### Widths and address arithmetic

The visible data model uses 32-bit address-bearing quantities and 32-bit memory loads, except for the signed 16-bit selector input.

The three offsets are fixed byte displacements. The evidence does not establish bounds checks, overflow checks, or a portable rule for address arithmetic that exceeds the address space. Normal-completion examples therefore assume that each required address calculation identifies an accessible location in the execution environment.

The function performs no visible arithmetic on the final loaded value. All 32 bits are returned without a visible mask, sign conversion, or interpretation.

## Side effects

No direct memory write is visible in F-0e89b7dfee22. Its direct observable operations are conditional memory reads, at most one dependency invocation, and a return.

This does not establish that the overall operation is side-effect-free. F-a2a686313150 may have effects not described by the supplied evidence, including changes to memory or global state. Its implementation and contract are unavailable.

In particular:

- Global preservation is not established.
- Preservation of the input object or associated records across the call is not established.
- Allocation, release, reference-count changes, and ownership transfer by the dependency are not established.
- The lifetime and validity of the dependency result and the further record cannot be assumed beyond the prerequisites needed for the subsequent reads.

There is no explicit allocation, deallocation, retain, release, or ownership-transfer operation in this function’s visible body. Returning bits does not by itself establish that the caller acquires an owned resource.

No local error reporting, recovery, cleanup, or exception handling is visible. If a required read cannot complete, or the dependency does not return normally, the evidence does not specify a normal return value for that execution.

## Dependencies

### F-a2a686313150

**Established call contract at this call site:**

- Called only after the initial object-field read is nonzero.
- Called at most once per invocation of F-0e89b7dfee22.
- Receives a 32-bit first argument from the subsequent read of that field.
- Receives the sign-extended selector as its second argument.
- Produces a 32-bit result that this function compares with zero.
- A nonzero result is subsequently used as the base address for a read at offset `0x7C`.

**Unknown dependency behavior — documented_only:**

- Meaning and validity requirements of its arguments.
- Meaning of a zero result beyond this caller’s treatment of it.
- Structure, ownership, and lifetime of a nonzero result.
- Whether it modifies globals, input-associated state, or other memory.
- Whether it allocates, releases resources, invokes callbacks, or fails to return normally.

No further function dependency is visible.

## Acceptance examples

These are conditional examples derived from static evidence, not executed tests. Unless stated otherwise, all accessed memory is valid and stable for the relevant reads, and the dependency returns normally.

### Initial field is zero

Given that the first read at object offset `0x1B0` produces zero, the result must be zero. The dependency must not be invoked. Memory associated with a hypothetical dependency result need not be readable because that path is not taken.

### Lookup produces no result

Given that the initial object-field read is nonzero and F-a2a686313150 returns zero, the result must be zero. No read at dependency-result offset `0x7C` may occur on this path.

### Successful traversal returns exact bits

Given that:

- The initial object-field read is nonzero.
- The dependency returns a nonzero address.
- The 32-bit field at that address plus `0x7C` identifies an accessible further record.
- The 32-bit value at the further record plus `0x04` is `0x89ABCDEF`.

The returned bits must be `0x89ABCDEF`. A high set bit does not introduce a visible transformation.

### Successful traversal also returns zero

Under the same valid traversal conditions, if the final 32-bit field is zero, the function must return zero. That result alone cannot distinguish this path from either early return.

### Negative selector

Given a selector of −1 and an invocation that reaches the dependency, its second argument must represent signed 32-bit −1, with bit pattern `0xFFFFFFFF`, rather than positive 65535.

For selector −32768, the corresponding argument is signed 32-bit −32768, with bit pattern `0xFFFF8000`.

### Different values from the two object-field reads

If the execution environment permits the first field read to yield a nonzero value and the second to yield zero, the visible behavior calls F-a2a686313150 with zero as its first argument. The initial nonzero condition does not constitute a second validation.

This example describes the two-read structure; it does not assert a supported concurrent-mutation model.

### Missing further-record address

If the dependency result is nonzero but its field at `0x7C` contains zero, no graceful zero-return behavior is established. The next attempted read is at address `0x04`, subject to the execution environment’s memory behavior.

## Uncertainties

The following remain **documented_only**:

- Semantic names and purposes of all records and fields.
- Public calling convention, byte order, and alignment rules.
- Thread safety, synchronization, and visibility of concurrent changes.
- Fault behavior for inaccessible or unsuitable memory.
- Address-overflow behavior.
- Dependency side effects and failure modes.
- Resource ownership and record lifetimes.
- Whether zero conventionally means absence, error, or a valid stored value.

The inferred lookup intent must not be used to add validation, caching, retries, ownership guarantees, or a stronger error contract.

## Evidence

Established facts come solely from static inspection of the supplied function representation: its branch conditions, input widths, conversion, dependency call, memory-access expressions, offsets, and return paths.

The two-read description reflects the visible expressions. It is not an independently verified machine-level instruction trace or a concurrency guarantee.

No dependency implementation, caller behavior, runtime observation, or execution trace was supplied. No runtime tests have been performed. The acceptance examples are consequences of the visible control flow under their stated prerequisites, not claims of observed execution.
