<!-- Trusted provenance: F-22f5c8cfa30b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-22f5c8cfa30b

## Purpose

F-22f5c8cfa30b performs an ordered transition on caller-supplied storage. It overwrites the first 32-bit word, conditionally invokes F-029930293dcb after each of three field checks, overwrites the first word again, obtains an input through a two-stage external indirection, invokes F-03b9616af95f, and returns the original storage address.

**Established static behavior:** The checked fields are at byte offsets +32, +24, and +28, in that order. Each nonzero value observed causes one invocation of F-029930293dcb. After those decisions, the second first-word store precedes both the read of an externally stored pointer and the read through that pointer used to obtain the final dependency’s leading input.

**Inferred intent:** The first-word replacements and delegated operations resemble a staged lifecycle transition. This does not establish destruction, inheritance, resource release, or any particular representation of the first word.

The contract covers the visible wrapper. Dependency effects remain unknown unless expressly constrained by an acceptance example.

## Interface

### Explicit inputs

| Input | Established role |
|---|---|
| Object address | Identifies storage accessed as 32-bit words; also returned on normal completion. |
| Second input | A 32-bit value with no explicit use in the visible body. |
| Third input | A 32-bit value supplied to F-03b9616af95f. |
| Fourth input | A 32-bit value supplied to F-03b9616af95f. |

Two additional 32-bit quantities appear as recovered machine-state inputs to F-03b9616af95f. The supplied representation does not establish them as ordinary public parameters, explain their provenance, or show their initialization within this function.

The second explicit input has no visible data-flow use. This is not a guarantee that its incoming machine representation is unobservable to dependencies under the actual calling convention.

### Object layout

Offsets are measured in bytes from the object address.

| Offset | Width | Visible access |
|---|---:|---|
| +0 | 32 bits | Written twice using two externally supplied 32-bit values from distinct references. |
| +24 | 32 bits | Read for the second conditional invocation. |
| +28 | 32 bits | Read for the third conditional invocation. |
| +32 | 32 bits | Read for the first conditional invocation. |

The distinct external references supplying the first-word stores need not contain unequal values.

The highest directly accessed byte is +35. No interpretation is established for intervening storage, and these accesses do not prove the object’s total size.

Each conditional decision compares a complete 32-bit word with zero. Every nonzero bit pattern qualifies, including a value with only its most significant bit set.

### External storage

The wrapper uses four distinguishable external roles:

- A reference supplying the first 32-bit value stored at object offset +0, called the **entry-state reference**.
- A distinct reference supplying the second 32-bit value stored there, called the **final-state reference**.
- A **pointer-bearing global**, read after the second object store to obtain a pointer.
- The **pointed-to storage**, read through that pointer to obtain the leading input for F-03b9616af95f.

The final two roles are separate accesses, not a single direct read of the final argument from a global.

The object words and explicit scalar inputs are established as 32-bit quantities. The supplied material does not independently establish the pointer width or the width and semantic type of the value read from the pointed-to storage; the relevant external declaration and dependency prototype are absent.

### Output

On normal completion, the function returns the original object address. It does not return a dependency result, status flag, replacement address, or resource count.

The returned address is not a guarantee of continued object lifetime, retained ownership, or safe dereference. The final dependency may change memory or lifetime in ways not established here.

### Preconditions for normal completion

The following accesses must remain valid when reached:

- Reading the entry-state reference and writing the first object word.
- Reading object words at +32, +24, and +28 in sequence.
- Reading the final-state reference and writing the first object word again.
- Reading the pointer-bearing global.
- Reading through the pointer obtained from that global.

In particular, accessibility of the pointer-bearing global does not imply accessibility of its target. Both are required for the final input acquisition to complete normally.

Any invoked dependency must return normally for execution to reach subsequent wrapper operations. If it changes memory accessibility, the remaining accesses must still be valid.

There is no visible null check, bounds check, alignment repair, pointer validation, or lifetime validation. Exact alignment requirements and invalid-access consequences are not established by the supplied representation.

## Behavior

### Entry-state replacement

The function obtains a 32-bit value from the entry-state reference and stores it at object offset +0.

This is the first visible object access. The previous first word is overwritten without a visible read, comparison, or preservation step.

The store precedes all conditional field reads. Thus any conditional dependency invocation occurs after the wrapper has performed this replacement. It does not follow that the first word remains unchanged throughout those invocations.

### Ordered conditional decisions

The function next performs three decisions in this order:

1. Read the 32-bit word at +32; invoke F-029930293dcb if it is nonzero.
2. After the first decision and any resulting invocation return, read the 32-bit word at +24; invoke F-029930293dcb if it is nonzero.
3. After the second decision and any resulting invocation return, read the 32-bit word at +28; invoke F-029930293dcb if it is nonzero.

These reads do not form an entry-time snapshot. A preceding invocation may change the value or accessibility of a later field before the wrapper reads it.

Each field is checked once. There is no visible loop, retry, or reread after a resulting invocation. A normally completed execution therefore contains between zero and three invocations of F-029930293dcb.

The visible calls show no explicit arguments. They do not establish whether the actual dependency receives the object address, triggering value, field address, or other machine state. No argument semantics should be assigned from the conditional checks alone.

The wrapper itself does not clear, decrement, or otherwise rewrite the three checked fields.

### Final-state replacement

After the third decision and any resulting invocation return, the function obtains a 32-bit value from the final-state reference and stores it at offset +0.

This reference is distinct from the entry-state reference. Their contents may nevertheless be equal.

The final-state acquisition occurs after the conditional calls. Its value must not be treated as captured at entry: no supplied dependency contract guarantees preservation of externally referenced storage.

The second store occurs regardless of whether the conditional invocation count was zero, one, two, or three, provided execution reaches it normally. It replaces any intervening first-word change if the destination remains writable.

### Two-stage acquisition of the final leading input

After the second object store, the wrapper performs the following ordered reads:

1. Read a pointer from the pointer-bearing global.
2. Read a value from the storage designated by that pointer.

The value from the second read is the leading input to F-03b9616af95f. The pointer itself is an intermediate address, not the established leading argument.

Both reads occur after the second first-word store. Neither the pointer nor the pointed-to value is established as an entry-time snapshot. Earlier dependencies may have changed either storage location.

The pointer-bearing global and pointed-to storage have separate accessibility requirements and separate aliasing possibilities:

- If the pointed-to storage overlaps the object, the second read observes that object storage at this later stage.
- If the pointer-bearing global overlaps object storage, an earlier object store may affect which pointer is obtained.
- If either location overlaps another external reference, earlier reads, stores, or dependency effects may interact through that overlap.

No such aliasing is proven, but disjointness is not established either. In particular, the wrapper provides no protection against an object store corrupting an aliased pointer-bearing location.

### Final dependency and return

The recovered invocation of F-03b9616af95f presents six inputs in this order:

1. The value read from the pointed-to storage.
2. The original object address.
3. The third explicit input.
4. The fourth explicit input.
5. One recovered machine-state quantity.
6. Another recovered machine-state quantity.

This input ordering does not establish a complete calling convention or machine-level argument evaluation order. The two dependent reads obtaining the first input are ordered as described above.

F-03b9616af95f is invoked once on every normal path reaching this stage. Its result, if any, is not visibly inspected.

After that invocation returns, the wrapper returns the original object address without a further visible object read or write.

### Arithmetic and interruption

Object offsets arise from four-byte word indexing: six, seven, and eight words correspond to +24, +28, and +32 bytes. No input-dependent indexing, length arithmetic, counter update, or explicit overflow check appears.

Signed interpretation is unnecessary for the zero comparisons. No narrowing conversion of a checked field is shown.

The fixed offsets do not establish safe address formation for an arbitrary object address. No address-wrap handling is visible.

If an access fails or a dependency does not return normally, the representation supplies no alternate completion path. There is no visible recovery, rollback, or error-status return.

## Side effects

### Direct wrapper effects

On normal completion, the wrapper directly performs:

- Acquisition of the entry-state value and a 32-bit store at +0.
- Three 32-bit reads at +32, +24, and +28, interleaved with their conditional invocations.
- Acquisition of the final-state value and another 32-bit store at +0.
- A read of the pointer-bearing global followed by a read through the obtained pointer.
- Zero to three invocations of F-029930293dcb.
- One invocation of F-03b9616af95f.

No direct allocation, release, or write to a checked field is visible. Dependencies may perform such operations; their absence from the wrapper is not evidence of their absence from the complete execution.

### Ownership and lifetime

No ownership transfer is established for the object, the checked words, the external references, the pointer-bearing global, or its target.

The checked words may represent pointers, handles, flags, or other values. Their semantics are unknown. Nonzero checks alone do not establish resource ownership or release.

Likewise, neither dependency may be assumed to preserve the object, preserve globals, release resources, or retain references. Returning the original address imposes no documented post-call lifetime guarantee.

### Preservation and concurrency

There is no established preservation guarantee for dependency-visible memory or external state. Later reads use the state present when they occur.

The stated access order describes the visible wrapper sequencing. It does not establish atomicity, thread safety, synchronization, or observations available to concurrent readers.

## Dependencies

| Opaque ID | Invocation condition | Established interface information | Unresolved behavior |
|---|---|---|---|
| F-029930293dcb | Once for each nonzero word observed at +32, +24, and +28, in that order | No explicit arguments are shown; no result is visibly consumed | Actual arguments, memory and global effects, ownership, lifetime, failure, and return behavior |
| F-03b9616af95f | Once after the second store and the two-stage leading-input acquisition, if execution reaches it | Receives the recovered six-input sequence; no result is visibly consumed | Complete calling convention, input meanings, memory effects, ownership, lifetime, and failure behavior |

The wrapper contains no visible retry or compensating action for either dependency.

The external state references, pointer-bearing global, and pointed-to storage are also required dependencies. Their definitions, lifetimes, mutability, and aliasing relationships were not supplied.

## Acceptance examples

These are conditional expectations from static evidence, not executed tests.

### All checked values are zero

Given valid accesses, zero at each checked offset when read, and a normally returning final dependency:

- The entry-state store precedes reads at +32, +24, and +28.
- F-029930293dcb is not invoked.
- The final-state store occurs.
- The pointer-bearing global is read, then its target is read.
- F-03b9616af95f is invoked with the target value as its leading input.
- The original object address is returned.

### All checked values are nonzero

If each checked word is nonzero when reached and all accesses and invocations permit normal continuation, F-029930293dcb is invoked three times, following checks at +32, +24, and +28 respectively.

A word containing only the high bit set qualifies just as a word containing one does.

### An earlier invocation changes later conditions

Suppose +32 is nonzero and its resulting invocation changes +24 to zero and +28 to a nonzero value. If these values remain until their reads, the wrapper skips the +24 invocation and performs the +28 invocation.

There are two conditional invocations, regardless of the entry-time values at +24 and +28.

### Both state references contain the same value

If both state references yield the same 32-bit pattern, both stores still occur in their established positions. Equality of their contents does not eliminate the second store or change the dependency sequence.

### A dependency redirects the leading-input pointer

Suppose a conditional invocation changes the pointer-bearing global to identify different valid storage. If no later operation changes it, the wrapper reads that updated pointer after the second object store and obtains the leading input from the new target.

The former target is not the established source of that argument.

### The target aliases the first object word

Assume the pointer-bearing global remains readable, yields the original object address, and the target read is a compatible 32-bit access. If no intervening effect changes that word, the leading input is the value written by the second store.

This example does not establish the target read’s width generally or require such aliasing in actual executions.

### The pointer-bearing global aliases writable object storage

If an object store overlaps the pointer-bearing global, the later pointer read uses the resulting contents. If those contents do not designate valid readable storage, normal completion is not established. The wrapper performs no visible validation or repair.

### Continuation fails

If the first conditional invocation never returns normally, no later conditional read, second store, external pointer read, target read, final invocation, or normal return is established.

If the global pointer read succeeds but the target read fails, the final dependency invocation and normal return are likewise not established.

## Uncertainties

The following remain **documented_only**:

- Numeric values and semantic identities of the two state words.
- Whether either first-word value denotes a dispatch structure, type, lifecycle state, or something else.
- Semantic types of the checked fields.
- Pointer width and the width and type of the indirectly read leading input.
- Complete argument interfaces and effects of both dependencies.
- Provenance and meaning of the two recovered machine-state quantities.
- Ownership, allocation, release, and lifetime effects.
- Aliasing among object storage, state references, the pointer-bearing global, and its target.
- Alignment, endianness, invalid-access behavior, and address-wrap consequences.
- Concurrency, reentrancy, and synchronization guarantees.

Lifecycle-transition intent remains an inference, not an acceptance requirement.

## Evidence

The supplied static representation establishes two 32-bit first-word stores, three ordered 32-bit zero comparisons with conditional calls, a final call using an indirectly read leading input, and return of the original object address.

Four-byte indexing establishes the field offsets. The final leading-input expression establishes a pointer read from external global storage followed by a read through that pointer. Its placement establishes that both occur after the second object store.

Distinct references establish separate sources for the state words, not unequal contents. The absence of visible dependency-result handling establishes only that this wrapper does not explicitly consume those results.

Dependency bodies, external declarations, a complete calling-convention description, and runtime observations were not supplied. No runtime tests have been performed.
