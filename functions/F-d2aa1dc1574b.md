<!-- Trusted provenance: F-d2aa1dc1574b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d2aa1dc1574b

## Purpose

F-d2aa1dc1574b conditionally writes a supplied byte to an address derived from two fields of a supplied descriptor.

A nonzero descriptor selects both the destination base and an index. The caller-supplied third argument is replaced by the descriptor’s base field before the write decision. Consequently, the third argument is not a fallback destination: a zero descriptor suppresses the operation, and a nonzero descriptor whose base field is zero also suppresses the write.

The established behavior is address calculation and a single-byte store. Any interpretation as updating an element flag, status, or record attribute is inferred intent, not an established contract.

## Interface

The function accepts three arguments in this order:

| Argument | Represented width | Role |
|---|---:|---|
| Descriptor address | 32-bit integer/address representation | Zero disables all memory accesses. Otherwise, identifies the descriptor fields to read. |
| Byte value | 8 bits | Value stored if the write conditions are satisfied. No signed interpretation is required. |
| Initial base value | 32-bit integer/address representation | Has no effect on the observable operation: it is either replaced by the descriptor’s base field or ignored because the descriptor is zero. |

The function returns no value. It provides no success flag, error code, or destination address.

The supplied representation uses 32-bit integer fields and an 8-bit store. It does not independently establish the native pointer width, machine byte order, alignment requirements, or complete address-arithmetic semantics. The layout and examples below use that represented 32-bit field model; portability beyond it is documented_only.

For a nonzero descriptor, the relevant layout is:

| Offset from descriptor | Width | Meaning established by use |
|---|---:|---|
| 12 bytes | 32 bits | Index used in destination-address calculation |
| 20 bytes | 32 bits | Base value that replaces the third argument |

The function does not establish a complete descriptor size or require that intervening bytes contain any particular values. Only these two fields are accessed, and the index field is accessed only when the loaded base is nonzero.

For executions with ordinary, representable arithmetic, the destination is:

**loaded base + 20 × loaded index − 17 bytes**

This is a byte-address calculation. The factor 20 is not multiplied again by a pointer element size.

## Behavior

### Disabled descriptor

When the descriptor argument is zero, the function performs no memory read or write and returns without a result.

Neither the byte value nor the initial base value changes that outcome. In particular, supplying a nonzero third argument does not enable a write when the descriptor is zero.

### Base selection

When the descriptor argument is nonzero, the first observable memory operation is a 32-bit read at descriptor offset 20.

The result becomes the effective base for this invocation. The supplied third argument is no longer relevant.

If the loaded base is zero, the function returns without reading the index field and without writing a byte. This remains true even if the original third argument was nonzero.

There is no preliminary validation of the nonzero descriptor. The offset-20 field must therefore be readable for an ordinary successful invocation along this path.

### Index selection and destination

If the descriptor and loaded base are both nonzero, the function reads the 32-bit index field at descriptor offset 12.

It then derives the destination by scaling that index by 20 bytes, adding the loaded base, and applying a displacement of minus 17 bytes. Provided these operations have ordinary representable results, index zero selects 17 bytes before the base; index one selects 3 bytes after the base.

No bounds check, index normalization, or rejection of negative indices is present. The represented index is an integer rather than an explicitly unsigned quantity. A negative index is not itself a reason to skip the store, although safe execution still depends on address arithmetic and destination validity.

### Store and completion

The function writes exactly the supplied 8-bit value to the derived destination. It does not first read that destination and does not preserve or combine selected bits from its previous contents.

There is no second store, explicit field update, or subsequent read. After the store, the function returns without a value.

For an execution that reaches the write, the source-level memory-access order is:

1. Read the base field at descriptor offset 20.
2. Read the index field at descriptor offset 12.
3. Write one byte at the derived destination.

This describes the supplied function’s accesses, not a hardware memory-ordering or synchronization guarantee.

### Arithmetic limits

The represented index and base fields are 32 bits. The scaling factor is 20, and the final displacement is negative 17.

The supplied text does not establish what happens when the multiplication or subsequent address arithmetic exceeds the applicable integer range. It does not justify choosing wrapping, trapping, saturation, or wider intermediate arithmetic. Overflowing cases are therefore documented_only.

No overflow example should be accepted solely because a destination could be obtained by reducing the mathematical result modulo \(2^{32}\). Such reduction requires evidence not present here.

## Side effects

On a valid write path, the only explicit externally visible modification is one byte at the calculated destination. On either skip path, the function makes no explicit modification.

The function performs no allocation, deallocation, ownership transfer, reference-count adjustment, or retention of a pointer for later use. It provides no evidence that any memory is released.

The destination may alias the descriptor. No disjointness check is present. If the destination falls inside either descriptor field, the store can change one byte of that field. Both field reads required for the current invocation occur before this store; the function does not reload either field afterward.

Aliasing may therefore affect later invocations even though it does not trigger another calculation in the current invocation. The numerical effect of changing a byte within a 32-bit field depends on byte order, which is not established.

Required memory validity is path-dependent:

- A zero descriptor requires no readable descriptor storage.
- A nonzero descriptor requires a readable 32-bit field at offset 20.
- A nonzero loaded base additionally requires a readable 32-bit field at offset 12.
- Reaching the store requires the calculated byte address to be writable.

No local error handling is present for invalid storage, unsuitable alignment, inaccessible destinations, or arithmetic faults. The resulting failure mode and recovery behavior are unknown; successful return must not be assumed in those cases.

No locking, atomic access, memory barrier, or concurrent-mutation handling is established.

## Dependencies

No external function calls are present. There are no callee dependencies requiring additional opaque function IDs.

The behavior depends directly on the execution environment’s integer representation, address interpretation, memory permissions, and access semantics. These are prerequisites, not demonstrated services supplied by another function.

No assumption about an unknown callee preserving global state or managing memory is necessary: this function invokes none. The available evidence nevertheless does not identify the accessed storage as heap, stack, global, device-backed, or any other storage class.

## Acceptance examples

These examples are conditional specifications, not runtime observations. Unless stated otherwise, assume valid readable fields, representable arithmetic, and writable destination storage.

### Zero descriptor ignores a usable third argument

Given a zero descriptor, a byte value of 166, and a nonzero initial base value:

- No descriptor field is read.
- No destination byte is written.
- The function returns without a value.

### Zero loaded base suppresses the index read

Given a nonzero readable descriptor whose offset-20 field contains zero:

- The offset-20 field is read.
- The offset-12 field is not read.
- No store occurs, regardless of the third argument.

This path does not require the index field to be accessible, provided reading the base field is itself valid.

### Index one writes three bytes beyond the loaded base

Given a loaded base of 4096, an index of 1, and a byte value of 166:

- The destination is 4099.
- That byte becomes 166.
- The function explicitly modifies no other byte.

### Index zero writes before the base

Given a loaded base of 4096 and an index of 0:

- The destination is 4079.
- The function does not treat zero as an invalid index or suppress the store.

### Negative index is not filtered

Given a loaded base of 4096 and an index of −1:

- The mathematical destination is 4059.
- No explicit negative-index check suppresses the write.
- Successful completion remains conditional on the environment supporting the represented arithmetic and destination access.

### Third-argument changes do not select another destination

For two invocations with identical descriptor fields and byte values, changing only the third argument does not change the derived destination or either skip decision, absent unrelated concurrent memory changes.

### Aliasing can modify the descriptor

If the calculated destination equals a byte within the descriptor:

- The supplied byte is stored there.
- The already-read base and index are not reloaded.
- A later invocation may observe the modified descriptor contents.

## Uncertainties

The following remain documented_only:

- The business meaning of the descriptor, index, and destination byte.
- Whether the 20-byte stride corresponds to records or another layout.
- Why the selected byte is displaced by minus 17.
- Native pointer width, byte order, and alignment rules.
- Overflow and out-of-range address-arithmetic behavior.
- Fault handling for invalid reads or writes.
- Concurrency guarantees and effects of asynchronous field changes.
- Storage ownership and allocation extent.
- Whether descriptor aliasing is intentional.

None of these unknowns permits adding validation, fallback selection, bounds enforcement, memory management, or error reporting to the established behavior.

## Evidence

Established static evidence consists of the supplied body for F-d2aa1dc1574b: the descriptor-zero guard, replacement of the third argument from offset 20, the loaded-base guard, the conditional index read at offset 12, the 20-byte scaling and minus-17 displacement, and the single 8-bit store.

The record-like interpretation is inferred from the stride and displacement only. It is not an acceptance requirement.

No runtime tests have been performed. The acceptance examples state consequences of the supplied static behavior under their explicit validity and arithmetic assumptions; they are not reported executions.
