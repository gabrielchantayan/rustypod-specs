<!-- Trusted provenance: F-fc64ed0c96d4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fc64ed0c96d4

## Purpose

F-fc64ed0c96d4 is a conditional indirect-dispatch wrapper. It calls F-e6c2a72b1331 to obtain a value interpreted as a pointer. If that pointer is null, the wrapper returns a four-byte zero. Otherwise, it obtains a dispatch-table address from the pointed-to storage, selects the entry at byte offset 284, invokes that entry, and returns the invocation’s four-byte result.

The established behavior is selection and forwarding, not validation or resource management. The wrapper checks only the initial pointer for null. It does not visibly check the table address or selected callable address.

Interpreting the returned pointer as an object and the table as a virtual-method table is plausible intent, not an established interface requirement. No particular class, operation name, or application-level meaning is established.

## Interface

**Explicit inputs.** The supplied representation declares no parameters for F-fc64ed0c96d4. The wrapper passes no explicit arguments to F-e6c2a72b1331 or to the selected indirect callable.

This does not establish the absence of implicit inputs. Registers, global state, thread-local state, or calling-convention-specific context may matter to either dependency. In particular, the evidence does not establish whether the selected callable receives an implicit receiver.

**Output.** A normally completed invocation returns a four-byte result:

- If F-e6c2a72b1331 returns a null pointer, the result is zero across those four bytes.
- Otherwise, the result is the four-byte value produced by the selected callable, without a visible comparison, conversion, or status interpretation.

The result’s signedness and semantic type are unknown. It must not be assumed to represent a Boolean, error code, address, or count.

**Referenced storage.** The non-null path uses the following layout:

| Location | Observed use |
|---|---|
| Address returned by F-e6c2a72b1331 | Start of storage whose first integer-sized value supplies the table base |
| Table base plus 284 bytes | Location from which the indirect callable address is obtained |

The offset is hexadecimal 0x11c, equivalent to decimal 284. It is a byte displacement, not a displacement of 284 table entries.

A four-byte integer and four-byte pointer interpretation is consistent with the supplied representation. Under that interpretation, the first read occupies bytes 0–3 of the returned storage, and the callable-address read occupies bytes 284–287 relative to the table base. Pointer width and target ABI are not independently supplied, so those precise storage extents remain conditional on that interpretation.

**Normal-completion preconditions.** For the non-null path to complete normally, the initial pointer must permit the first required read, the resulting table-base arithmetic must identify readable callable-address storage, and the selected address must support invocation under the effective calling convention. The wrapper does not enforce these conditions.

## Behavior

The following ordering is established by the supplied representation. Reads or writes performed internally by dependencies are outside this ordering description unless explicitly identified.

1. **Acquire the candidate pointer.**  
   The wrapper invokes F-e6c2a72b1331 once. The result is interpreted as a pointer to storage containing an integer at its beginning. There is no visible retry or alternative provider.

2. **Test the pointer for null.**  
   The wrapper compares that returned pointer with null before performing its own dereference of the pointed-to storage.

3. **Return zero on the null branch.**  
   If the pointer is null, the wrapper returns a four-byte zero. It does not read a table base, fetch a callable address, or invoke the indirect callable on this branch. F-e6c2a72b1331 has nevertheless already run; its effects are not undone.

4. **Read the table base on the non-null branch.**  
   The wrapper reads the integer stored at the beginning of the returned storage. That value is used as the base for selecting the callable-address slot. No additional field of the initially returned storage is visibly inspected.

5. **Locate and read the callable-address slot.**  
   The wrapper adds the fixed byte displacement 284 to the base value and reads the callable address at that location. There is no visible range check, table-length check, null-table check, or null-callable check.

6. **Invoke and forward.**  
   The selected address is invoked with no explicit arguments shown. If the invocation returns normally, its four-byte result becomes the wrapper’s result. There is no visible post-call reread of the original storage or table and no visible modification of the returned value.

This ordering does not imply that the pointer or table existed before F-e6c2a72b1331 ran. That dependency could create, select, or modify the storage before returning it. Nor does the ordering establish synchronization with other threads or protection against concurrent changes.

The null check protects only against dereferencing the initial null pointer in this wrapper. A non-null but invalid pointer does not receive equivalent handling. Likewise, a zero table base is not treated as the initial-null case: the wrapper still proceeds toward the displaced callable-address read.

## Side effects

The wrapper contains no explicit stores to object storage, table storage, or globals. It contains no visible allocation, release, reference-count adjustment, lock operation, logging operation, or error-reporting operation.

That limited statement does not imply that the complete invocation is side-effect-free. F-e6c2a72b1331 and the selected callable may read or modify shared state, allocate or release storage, perform I/O, or fail to return. Their effects are unspecified by the available evidence.

No ownership transfer is established:

- The wrapper does not visibly retain or release the initially returned pointer.
- It does not visibly copy the referenced object or table.
- It does not visibly free either storage region.
- The selected callable’s ownership behavior is unknown.

The required storage must remain usable until the corresponding reads and invocation occur. The wrapper supplies no visible lifetime enforcement.

No explicit recovery path is shown for invalid memory, an invalid callable address, or a failure inside a dependency. The resulting fault, exception, termination, or non-returning behavior depends on the runtime and dependencies. It must not be specified as a zero return: zero is the explicit fallback only for an initially null pointer.

## Dependencies

**F-e6c2a72b1331**

- Invoked once before the branch decision.
- Receives no explicit arguments in the supplied representation.
- Supplies the value interpreted as the initial pointer.
- Has unknown side effects, ownership rules, failure behavior, and implicit inputs.

**Indirect callable selected from the table**

- Selected using the base value read from the initial storage and the fixed byte displacement 284.
- Invoked only after F-e6c2a72b1331 returns a non-null pointer and the required address reads complete.
- Receives no explicit arguments in the supplied representation.
- Supplies the four-byte result forwarded by the wrapper.
- Has no established opaque function ID in the supplied dependency map.

The indirect target’s identity is determined by referenced storage, not by a named direct call. Its signature, receiver convention, and preservation of global state remain unknown.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence, not reports of executed tests.

**Null provider result.** If F-e6c2a72b1331 returns null normally, the wrapper must return four-byte zero. It must not subsequently perform its own table-base read or indirect invocation. Any effects already performed by F-e6c2a72b1331 remain possible.

**Nonzero indirect result.** If the provider returns valid storage, the first stored integer identifies a usable table base, and the callable address at base plus 284 invokes successfully and returns a four-byte value of 37, the wrapper must return 37. It must not replace that result with a success flag or provider-derived value.

**Zero indirect result.** If the same non-null path returns zero from the selected callable, the wrapper must also return zero. Consequently, a zero wrapper result alone cannot distinguish the null branch from a successful indirect invocation returning zero.

**Full-width result preservation.** If the selected callable returns a four-byte pattern whose highest bit is set, that four-byte result must be forwarded without visible Boolean normalization or rejection. Whether a caller interprets it as signed or unsigned is outside the established contract.

**Byte-offset selection.** Under a four-byte pointer-table interpretation, the callable-address slot begins 71 pointer widths after the table base. Acceptance must use the entry at byte offset 284, not entry number 284.

**Invalid downstream address.** If the initial pointer is non-null but the table or callable address is unusable, the evidence does not require a zero result or graceful recovery. Such an input violates the conditions for guaranteed normal completion.

**Non-returning dependency.** If either dependency does not return, the wrapper has no established alternative path that produces a result.

## Uncertainties

The following are **documented_only**:

- The target’s pointer width, byte order, alignment requirements, and exact calling convention.
- The exact machine interpretation of the integer table-base value and its conversion into an address.
- Overflow behavior when adding 284 near an address-space boundary. No bounds or overflow guard is shown; wraparound or language-level signed-overflow behavior must not be invented.
- Whether the initial storage is an object, handle, singleton, or another structure.
- Whether an implicit receiver or other hidden argument reaches the indirect callable.
- The callable’s identity, semantic operation, ownership effects, and failure model.
- Whether either dependency mutates globals, releases referenced memory, or interacts with concurrent execution.
- Whether additional compiler-generated or runtime behavior exists beyond the supplied representation.

The inferred object-and-method-table interpretation is useful context only. Acceptance does not depend on naming the storage as an object or assigning the selected slot a particular method role.

## Evidence

**Established static evidence:** one direct dependency call; one null-pointer branch; a zero return on that branch; one first-field integer read on the other branch; fixed displacement 0x11c; one callable-address read and indirect invocation; forwarding of a four-byte result; no explicit stores or cleanup operations in the wrapper.

**Inferred intent:** conditional delegation through a table-backed interface, with zero as the fallback when no initial pointer is available.

**Not established:** runtime validity of either branch’s surrounding environment, exact ABI details, dependency internals, application-level result meaning, or successful operation with any concrete object.

No runtime tests have been performed.
