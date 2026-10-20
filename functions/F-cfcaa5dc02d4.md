<!-- Trusted provenance: F-cfcaa5dc02d4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cfcaa5dc02d4

## Purpose

F-cfcaa5dc02d4 validates a selector and a destination-like argument, then requests an operation on one of six regularly spaced locations through an external dependency.

The established behavior is narrower than “copy a record.” The supplied representation establishes the arguments passed to the dependency, but not what that dependency does with them. A plausible intended use is to transfer a 20-byte record into caller-supplied storage. That interpretation is **inferred intent**, not a guaranteed postcondition.

The function returns one of three 32-bit status patterns on its visible normal-return paths:

- `0xffffffff`: selector rejected.
- `0xfffffffc`: destination-like argument rejected as zero.
- `0x00000000`: the dependency invocation returned normally.

A zero result does not independently establish that memory was copied or that the dependency reported success.

## Interface

### Inputs

**Selector**

A 32-bit unsigned integer selecting an entry.

- Accepted values: 0 through 5, inclusive.
- Rejected values: 6 through `0xffffffff`, inclusive.
- The comparison is unsigned. A bit pattern representing a negative signed integer is therefore a large selector and is rejected.

**Destination-like argument**

A 32-bit signed integer in the supplied representation.

- Zero is rejected.
- Every nonzero bit pattern passes the visible validation.
- There is no visible check for sign, alignment, addressability, capacity, writability, or overlap with another region.

Calling this argument “destination-like” describes its apparent role, not an established pointer type or storage guarantee. The external dependency’s interface is not supplied.

### Output

The output is a 32-bit quantity. Its signed interpretation is not established by the supplied return type.

| Return bits | Signed 32-bit interpretation | Visible condition |
|---|---:|---|
| `0xffffffff` | −1 | Selector exceeds 5 |
| `0xfffffffc` | −4 | Selector is accepted and the second argument is zero |
| `0x00000000` | 0 | Both checks pass and the dependency returns normally |

The signed interpretations are provided only to distinguish common caller presentations of the same bits. No symbolic error names are established.

### Preconditions

No additional precondition is needed to describe the two rejection paths: neither reaches the dependency.

For an accepted call to complete safely, the dependency’s actual argument requirements must be satisfied. Those requirements are unknown. In particular, this contract cannot establish that supplying a writable 20-byte region is sufficient.

The function also relies on a global base quantity used to form the dependency’s second argument. Its storage type, initialization requirements, lifetime, and validity constraints are not supplied.

## Behavior

### Validation and precedence

The selector is checked first.

If its unsigned value exceeds 5, the function returns `0xffffffff`. The destination-like argument is not subjected to its zero check on this path, and the dependency is not invoked.

For an accepted selector, the destination-like argument is checked against zero. If it is zero, the function returns `0xfffffffc`, again without invoking the dependency.

This establishes error precedence. If both arguments would fail their respective checks, selector rejection wins.

The second check is equality with zero, not a range or pointer-validity check. A negative signed value passes it.

### Selection arithmetic

After both checks pass, the function forms an offset equal to the selector multiplied by 20.

The reachable offsets are:

| Selector | Offset in decimal | Offset in hexadecimal |
|---:|---:|---:|
| 0 | 0 | `0x00` |
| 1 | 20 | `0x14` |
| 2 | 40 | `0x28` |
| 3 | 60 | `0x3c` |
| 4 | 80 | `0x50` |
| 5 | 100 | `0x64` |

These products fit within 32 bits. Consequently, multiplication overflow is not an issue on any accepted path.

The offset is added to the global base quantity. The supplied representation does not identify that quantity’s type sufficiently to establish the width, conversion rules, or overflow behavior of this addition. No visible check rejects an unusable resulting value.

The spacing and operation-size argument suggest six contiguous 20-byte records. If the dependency interprets its second argument as a byte address and its third argument as a byte count, selector 5 would identify a 20-byte span beginning 100 bytes after the base. Under that interpretation, all six entries collectively occupy 120 bytes. Those byte-address and byte-count semantics remain conditional.

### Dependency invocation

For accepted arguments, the function invokes **F-dependency-01** once, with three ordered arguments:

1. The original destination-like argument.
2. The global base quantity plus the selector-dependent offset.
3. The constant 20.

F-dependency-01 is a document-local opaque identifier for the external call whose identity and implementation were not supplied.

Argument position is established; argument evaluation order is not. The supplied representation does not justify a finer ordering among preparing the first argument, obtaining the base quantity, and preparing the constant.

The dependency’s result is not inspected or propagated. After the invocation returns normally, F-cfcaa5dc02d4 returns zero.

There is no visible retry, alternate dependency invocation, fallback, or post-call validation. There is also no visible distinction between different dependency return values.

### Abnormal completion

The normal-return contract is conditional on the dependency returning normally.

If argument formation or the dependency invocation faults, terminates execution, fails to return, or otherwise transfers control elsewhere, the supplied representation establishes no recovery behavior and no status result for that execution.

## Side effects

### Effects established for this function

The visible behavior includes:

- Validation of the two input values.
- Use of a global base quantity on the accepted path.
- One external dependency invocation on that path.
- Return of a fixed status pattern on each normal-return path.

There is no explicit destination-memory write, allocation, release, or global assignment in the function itself. Any such effect of an accepted call would have to arise through the dependency or through behavior not described by the supplied representation.

On either rejection path, the global base expression and dependency invocation are not reached. This is a control-flow statement, not a claim about speculative processor activity.

### Effects not constrained by the available evidence

The dependency may read or write memory, alter global state, acquire or release resources, or have other externally visible effects. Its name and body are unavailable, so none of these possibilities can be resolved here.

In particular, this contract does not assert that the dependency:

- Copies exactly 20 bytes.
- Preserves the source region or unrelated memory.
- Preserves the global base quantity.
- Leaves either argument region valid after returning.
- Accepts overlapping regions.
- Avoids allocation or deallocation.
- Preserves ownership.

### Ownership and lifetime

No ownership transfer is expressed directly by the wrapper’s visible operations. That does not establish an ownership guarantee for the overall call.

Whether the dependency retains an argument, releases storage, or requires storage to outlive the call is **documented_only**: the necessary dependency contract is absent.

## Dependencies

### F-dependency-01

**Identity:** Document-local opaque label for the single external dependency.

**Invocation condition:** Selector is at most 5 and the destination-like argument is nonzero.

**Invocation count:** Exactly one per accepted execution reaching the call.

**Arguments:** Original second input; global base plus selector times 20; constant 20.

**Return handling:** Any returned value is ignored. Normal return is followed by a zero status from F-cfcaa5dc02d4.

**Unknown contract:** Parameter types, address interpretation, size units, memory-access order, error signaling, ownership, global-state effects, concurrency requirements, and abnormal behavior.

### Global base quantity

A global data reference supplies the base used in the dependency’s second argument.

The available evidence does not establish:

- Whether the base denotes an address or another numeric quantity.
- Its precise declared type or arithmetic width.
- How it is initialized or changed.
- Whether its referent contains six valid entries.
- Whether concurrent access requires synchronization.

No immutability assumption is justified merely because this function contains no explicit assignment to it.

## Acceptance examples

These examples are conditional checks against the visible contract, not reports of executed tests.

1. **Selector 6, nonzero destination-like argument.**  
   Expected result: `0xffffffff`. F-dependency-01 is not invoked.

2. **Selector 6, zero destination-like argument.**  
   Expected result: `0xffffffff`, not `0xfffffffc`. Selector validation takes precedence.

3. **Selector 0, zero destination-like argument.**  
   Expected result: `0xfffffffc`. The dependency is not invoked.

4. **Selector 0, acceptable nonzero argument, dependency returns normally.**  
   The dependency receives the original second input, the base with zero offset, and 20. The function returns zero.

5. **Selector 5, acceptable nonzero argument, dependency returns normally.**  
   The dependency’s second argument uses an offset of 100. The function returns zero.

6. **Selector `0xffffffff`.**  
   Expected result: `0xffffffff`, regardless of the destination-like argument. The selector is unsigned and does not designate an entry before the base.

7. **Selector 2, negative nonzero second input.**  
   The wrapper’s checks pass. The dependency is invoked with that original input and an offset of 40. Safe completion is not established merely by passing validation.

8. **Dependency returns a failure-indicating value normally.**  
   If the dependency has such a convention, the wrapper still returns zero. The returned value is not examined.

9. **Dependency changes global state and returns normally.**  
   The wrapper contains no visible restoration step. Preservation of that state cannot be required from this evidence.

10. **Dependency does not return normally.**  
    No wrapper status result is guaranteed.

## Uncertainties

The following remain **documented_only**, rather than being filled in by inferred conventions:

- Exact dependency identity and semantics.
- Whether the apparent destination and source arguments are pointers.
- Whether 20 denotes bytes.
- Exact global-base type and addition semantics.
- Required storage capacity, alignment, accessibility, and lifetime.
- Overlap behavior.
- Ownership and resource-management effects.
- Thread safety and synchronization.
- Meaning of the numeric statuses beyond their branch conditions.
- Fault handling and other abnormal completion behavior.

The inferred record-transfer purpose must not be used to strengthen these unknowns into implementation guarantees.

## Evidence

**Established static evidence:** The supplied representation contains two ordered validation branches, selector-dependent multiplication by 20, addition to a global base quantity, one three-argument external invocation, and fixed 32-bit return patterns.

**Inferred intent:** The six accepted selectors, 20-unit spacing, destination-like first dependency argument, and constant operation size suggest fixed-size record transfer.

**Unavailable evidence:** Dependency implementation or authoritative contract, global-data declaration and contents, caller requirements, and runtime observations.

No runtime tests have been performed.
