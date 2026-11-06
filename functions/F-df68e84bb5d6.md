<!-- Trusted provenance: F-df68e84bb5d6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-df68e84bb5d6

## Purpose

F-df68e84bb5d6 conditionally updates a secondary object associated with a selected primary object. Selection depends on a 32-bit field in the supplied context. The primary object’s flags determine whether any update is attempted.

Two update paths are visible:

- A nonzero context state triggers two primary-object method calls, a secondary-object operation, an optional numeric update, and clearing of the context state.
- A zero context state triggers a secondary-object method using a signed 16-bit value obtained from that object.

The application-level meaning of the objects, state, flags, and numeric quantity is not established. Describing the operation as synchronization or initialization would be inferred intent, not a demonstrated contract.

## Interface

### Invocation and result

The function takes one value used as the base address of a context object. The supplied representation uses a 32-bit integer for this address and 32-bit object pointers and words.

The function has no represented return value. Normal early exits do not supply a success or failure indicator.

### Referenced layouts

All offsets below are byte offsets from the indicated base. These are partial layouts, not complete object definitions.

| Object | Offset | Width | Observed use |
|---|---:|---:|---|
| Context | `0x4E8` | 32 bits | Compared with zero to select one of two global values |
| Context | `0xDC` | 32 bits | Address of a state-bearing object |
| State-bearing object | `0x18` | 32 bits | Tested for zero; later overwritten with zero on one path |
| Primary object | `0x00` | 32 bits | Address of a method table |
| Primary object | `0x34` | 32 bits | Association value passed to a resolver |
| Primary object | `0x48` | 32 bits | Flags tested with mask `0x1800` |
| Secondary object | `0x00` | 32 bits | Address of a method table |
| Secondary object | `0x58` | 32 bits | Read and narrowed to a signed 16-bit method argument |

The observed indirect method-table entries are:

| Receiver | Table offset | Explicit arguments | Result use |
|---|---:|---|---|
| Primary object | `0x114` | Primary object | First 32-bit arithmetic operand |
| Primary object | `0xCC` | Primary object, integer `2` | Second 32-bit arithmetic operand |
| Secondary object | `0x164` | Secondary object, signed 16-bit value widened to the integer argument width | No result used |

The exact calling conventions and the wider interfaces of these methods are not established.

### Memory and execution preconditions

For ordinary completion, every object field read on the selected path must be readable, every called method entry must be valid, and the eventual state field must be writable when the nonzero-state path reaches its final store.

There is no initial validation of the context address.

The first resolver result must be valid for a read at offset `0x34`. A later null comparison does not protect this earlier read. The state-bearing object address is also dereferenced without a null check when the flag condition permits entry.

These are access requirements, not evidence that the function enforces them.

## Behavior

### 1. Select a lookup value

The function first reads the context’s 32-bit field at offset `0x4E8`.

- If the field is zero, it selects the first of two distinct global lookup-value locations.
- If the field is nonzero, it selects the second.

In either branch, it calls F-938c950e5046 with no explicit arguments. The selected global value is loaded after that call returns. Thus, a change to that global made by F-938c950e5046 can affect the value used in the next call.

The branch decision precedes F-938c950e5046 and is not recomputed afterward. If that dependency changes the context’s selector field, the already-selected branch still determines which global is read.

The function then calls F-9e50c66a96dd with:

1. The result from F-938c950e5046.
2. The selected global value.

The meanings and concrete values of the two globals are unknown.

### 2. Resolve the primary and secondary objects

The result from F-9e50c66a96dd is passed to F-682adfd5a561 together with the integer `0x1580`. The returned address is retained as the primary object.

The next operation reads the primary object’s 32-bit association field at offset `0x34`.

- If that field is zero, the function returns immediately.
- Otherwise, that association value is passed to F-682adfd5a561 with the integer `0x3B80`.

The second resolver result is retained as the secondary object.

The function returns if the secondary result is null or if the retained primary address is null. The primary null comparison occurs only after the earlier primary-field dereference. It must not be interpreted as a safe null-primary failure path.

The constants `0x1580` and `0x3B80` are established resolver arguments. Their interpretation as type identifiers, class identifiers, or another kind of lookup discriminator is unknown.

### 3. Apply the primary flag gate

After both resolutions, the function reads the primary object’s 32-bit flags at offset `0x48`.

Processing continues only when masking those flags with `0x1800` produces exactly `0x0800`. Equivalently:

- Bit `0x0800` must be set.
- Bit `0x1000` must be clear.
- Bits outside `0x1800` do not affect this decision.

If the condition fails, the function returns without entering either update path.

This flag read occurs after the second resolver call. Changes that resolver makes to the primary object can therefore affect the gate.

### 4. Choose the update path

When the flag gate succeeds, the function reads the context’s state-bearing object address at offset `0xDC`, then reads that object’s 32-bit field at offset `0x18`.

The resulting zero/nonzero test selects one of the following paths. There is no repeated state test after the selected path’s callbacks.

### 5. Nonzero-state path

When the tested state is nonzero, the function performs these calls in order:

1. Invoke the primary object’s method at table offset `0x114`, passing the primary object.
2. Invoke the primary object’s method at table offset `0xCC`, passing the primary object and integer `2`.
3. Multiply the two returned 32-bit integer values and test the product against zero.

The primary method-table address is obtained for each invocation. The representation does not cache one table address across both method calls. A change made by the first method can therefore affect the dispatch of the second method.

The product determines the next calls:

- If the product is zero, call F-a9537de18d9e with the secondary object.
- If the product is nonzero, first call F-a9537de18d9e with the secondary object, then call F-84d4e6783ea3 with that same retained secondary address and a derived integer value.

The derived value is represented as the product multiplied by `0x10000`, followed by a right shift of 16 bits.

After the selected dependency calls return, the function reloads the state-bearing object address from context offset `0xDC` and writes a 32-bit zero at offset `0x18` of that object. It then returns.

Consequently, if a callback changes the context’s state-bearing object address, the final store targets the newly referenced object, not necessarily the object whose state was originally tested. The function does not re-read the state to decide whether that store remains necessary.

F-a9537de18d9e is called once on every normally completed nonzero-state path. F-84d4e6783ea3 is called only when the initial product comparison is nonzero.

### 6. Zero-state path

When the tested state is zero, the function invokes the secondary object’s method at table offset `0x164`.

The arguments are:

1. The retained secondary-object address.
2. The 32-bit field at secondary offset `0x58`, narrowed to a signed 16-bit value and then widened to the integer argument width.

The narrowing uses the low 16 bits; the upper 16 bits do not contribute to the resulting argument under the represented target conversion. For example, low bits `0xFFFF` represent `-1`, not `65535`.

This path does not explicitly clear or otherwise write the context state. It does not call either primary-object method, F-a9537de18d9e, or F-84d4e6783ea3.

The exact ordering of the secondary method-table reads relative to the numeric argument-field read is not established by the supplied expression.

### Arithmetic boundary

The arithmetic operands and intermediate integer expressions are represented as 32-bit signed values. The source does not establish the target’s complete overflow and signed-shift semantics.

A common target interpretation of the multiply-by-`0x10000` and shift sequence is a signed interpretation of the product’s low 16 bits. That interpretation is inferred, not an unconditional guarantee for every input.

In particular:

- The product can exceed the signed 32-bit range.
- Even a representable product can overflow when multiplied by `0x10000`.
- A right shift of a negative intermediate requires target-specific signed-shift semantics.

The zero/nonzero decision is applied to the product before the derived-value calculation. It is not a test of the final narrowed value. Under wrapping arithmetic, a nonzero product can therefore cause F-84d4e6783ea3 to be called with zero.

Overflow-sensitive results remain documented_only pending authoritative target arithmetic evidence.

## Side effects

The only explicit memory store visible in this function is the 32-bit zero written to the state field after the nonzero-state update sequence.

All dependency and indirect-method calls may have additional side effects. Their behavior is not supplied. This contract does not assume that they:

- Preserve globals or object fields.
- Preserve method-table addresses.
- Preserve the context’s state-bearing object address.
- Leave either retained object alive.
- Allocate, retain, release, or transfer ownership in any particular way.
- Return normally.

The primary and secondary addresses are retained across calls; the function does not re-resolve either object before later use. Safe continued access therefore requires the dependencies to leave those retained addresses valid wherever the subsequent path uses them.

There is no visible allocation, deallocation, reference-count operation, rollback, exception handling, or error reporting performed directly by this function. A dependency may nevertheless perform any of those operations.

If a call fails to return normally, the later calls and final state store are not established as occurring. The function supplies no visible cleanup path for that case.

## Dependencies

| Opaque ID | Established role in this function |
|---|---|
| F-938c950e5046 | Called once with no explicit arguments after branch selection and before loading the selected global |
| F-9e50c66a96dd | Called with that result and the selected global; produces the first resolver input |
| F-682adfd5a561 | Resolves the primary using `0x1580`; conditionally resolves the secondary using `0x3B80` |
| F-a9537de18d9e | Called with the secondary on the nonzero-state path, before any numeric update |
| F-84d4e6783ea3 | Called with the secondary and derived integer only for a nonzero product |

The three indirect methods have no supplied opaque dependency IDs. They are identified here only by receiver and method-table offset. No semantic names or preservation guarantees are assigned to them.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They have not been executed. Unless stated otherwise, required memory is valid and all dependencies return normally.

1. **Zero association:** The first resolver returns a valid primary whose field at `0x34` is zero. The function returns without the second resolver call, the flag read, or either update path.

2. **Missing secondary:** The primary association is nonzero, but the second resolver returns null. The function returns without reading the primary flags or context state.

3. **Flag combinations:** A masked flag value of `0x0800` permits the state test. Masked values `0x0000`, `0x1000`, and `0x1800` do not. Unmasked bits do not change this decision.

4. **Zero state and signed argument:** The gate passes, the state is zero, and the secondary field at `0x58` is `0x1234FFFF`. The secondary method at table offset `0x164` receives integer `-1`. No explicit state store follows.

5. **Nonzero state and zero product:** The two primary methods return `0` and `7`. F-a9537de18d9e is called once; F-84d4e6783ea3 is not called. The currently referenced state-bearing object’s field at `0x18` is then cleared.

6. **Nonzero state and small positive product:** The primary methods return `2` and `3`. The product is `6`, and the derived calculation remains within the positive signed 32-bit range. F-a9537de18d9e precedes F-84d4e6783ea3, whose numeric argument is `6`. The state store follows both calls.

7. **Callback redirects state storage:** The original state is nonzero, and a callback replaces the context’s state-bearing object address with another valid address. The final zero store targets the replacement object. This function does not explicitly clear the original object.

8. **Global changes during selection:** The initial selector chooses the first global location. F-938c950e5046 changes that global before returning. F-9e50c66a96dd receives the updated global value; the selector is not retested.

9. **Null primary:** The first resolver returns null. A clean return is not an acceptance requirement: the function attempts the association read before its later null comparison.

## Uncertainties

The following remain documented_only:

- Concrete global values and their application meaning.
- Complete context, primary, secondary, and state-bearing object layouts.
- Dependency ownership, lifetime, mutation, failure, and reentrancy contracts.
- Meaning of resolver discriminators and flag bits.
- Semantic purpose of the two primary method results.
- Exact overflow, signed conversion, and signed-shift behavior at arithmetic boundaries.
- Aliasing between the referenced objects and global storage.
- Behavior for invalid addresses, invalid method tables, concurrent mutation, or non-returning dependencies.
- Any synchronization mechanism outside this function.

No clamping, retries, validation, rollback, or ownership policy may be inferred from the visible call sequence.

## Evidence

Established static evidence consists of the supplied function representation and dependency-ID mapping: field widths and offsets, direct call arguments, branch conditions, indirect dispatch offsets, represented arithmetic, and the final state store.

Inferred intent is limited to treating the objects as associated primary and secondary entities and recognizing a possible signed-16-bit normalization idiom. These interpretations do not replace the observable contract.

No runtime tests, dependency implementations, machine-level arithmetic checks, or external object-layout validation have been performed. The acceptance examples are conditional specifications, not observed execution results.
