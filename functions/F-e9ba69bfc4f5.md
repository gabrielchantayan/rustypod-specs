<!-- Trusted provenance: F-e9ba69bfc4f5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e9ba69bfc4f5

## Purpose

F-e9ba69bfc4f5 coordinates two dependencies, copies two resulting 32-bit words into a destination object, and returns a 64-bit value assembled from two other words.

The established behavior is a sequence of two calls, two destination stores, and one packed return. The first dependency receives addresses of mutable working words. The second receives an address to another working word and the current values of the first pair. The final destination remains based on the original first argument, even if the first dependency changes the corresponding working word.

Any interpretation as construction, initialization, decoding, or object assignment is inferred intent, not an established semantic contract. The available evidence does not identify the application-level meaning of the arguments or destination fields.

## Interface

### Inputs

The interface exposes four arguments:

| Argument | Established representation | Observable role |
|---|---|---|
| A | 32-bit integer in the supplied representation; also used as a destination address | Seeds a mutable working word and independently identifies the base for the final destination stores. |
| B | 32-bit word | Passed by value to the first dependency and used to seed a separate mutable working word. |
| C | 32-bit word | Seeds the working word whose address is passed to the second dependency. |
| D | 32-bit word | Seeds the companion word subsequently copied to the destination. |

The signedness of B, C, and D is not established. They should be treated as bit patterns unless a dependency contract supplies a stronger interpretation.

A has two distinct roles. Its initial bits are copied into mutable working storage, but its original value is also retained for destination addressing. These roles must not be collapsed into one mutable pointer.

### Working state

The function establishes four addressable, 32-bit working words:

- An A-derived word, initially equal to A.
- A B-derived word, initially equal to B.
- A C-derived word, initially equal to C.
- A D-derived companion word, initially equal to D.

The supplied representation places the C-derived and D-derived words consecutively, with the companion four bytes after the C-derived word. This is relevant because the second dependency receives the address of the C-derived word. It does not, by itself, establish how many bytes that dependency is permitted to access.

These working words are local to the invocation. They are not caller-supplied output buffers.

### Destination layout

Two explicit destination writes occur:

| Byte offset from original A | Width | Source at the time of the write |
|---|---:|---|
| 0x40 | 32 bits | Current C-derived working word |
| 0x44 | 32 bits | Current D-derived companion word |

The writes occupy adjacent four-byte fields: bytes 64–67 and bytes 68–71 relative to the original destination base.

The wrapper does not explicitly write the preceding 64 bytes or any bytes after offset 0x47. Dependencies may have additional effects; those are not bounded by these two stores.

### Output

The result is a 64-bit bit pattern formed from:

- Bits 63–32: the final B-derived working word.
- Bits 31–0: the final A-derived working word.

Each component contributes exactly 32 bits. A high bit set in the A-derived word does not sign-extend into the upper half. Likewise, the B-derived word is not reduced to a flag or status code.

The semantic type of this result is unknown. The evidence establishes packing, not whether consumers interpret the result as a pair, an integer, a handle, or another representation.

### Preconditions

For normal completion:

- Both dependencies must return normally.
- Storage reached through the original A at offsets 0x40 and 0x44 must remain valid and writable when the respective stores occur.
- Any dependency access through the supplied working-storage addresses must be valid under that dependency’s actual contract.
- Address formation and 32-bit stores must be supported by the execution environment.

The function performs no explicit null, range, capacity, alignment, or object-lifetime checks. The evidence does not establish portable behavior for invalid addresses, address overflow, or unsupported alignment.

## Behavior

### Initialization and first dependency

Before either dependency is invoked, the function initializes the four working words from A, B, C, and D.

It then invokes F-6a10d0fb9111 with three arguments, in this order:

1. B by value.
2. The address of the B-derived working word.
3. The address of the A-derived working word.

The first argument is the original B value. It is not a deferred read of the B-derived working word after the dependency starts running.

The two address arguments allow the dependency to affect values that are subsequently used by this function. Its return value is not consumed.

No stronger guarantee about F-6a10d0fb9111 is established here. In particular, the evidence does not prove that it writes either addressed word, restricts its accesses to those words, preserves globals, or avoids other memory effects.

### Second dependency

After the first dependency returns, the function invokes F-7551853739b9 with:

1. The address of the C-derived working word.
2. The then-current B-derived working word, passed by value.
3. The then-current A-derived working word, passed by value.

Thus, any changes made by the first dependency to the A-derived and B-derived words affect the values passed to the second dependency.

The second and third arguments are values, not addresses of the corresponding working words. Passing them does not directly grant the second dependency a reference to those words. However, this does not prove that those words cannot change through other aliases or retained addresses.

The second dependency’s return value is also ignored.

The address of the C-derived word is the only working-storage address explicitly passed in this call. The adjacent D-derived word is subsequently read by the wrapper, but the available evidence does not establish whether F-7551853739b9 writes one word, two words, or some other documented extent.

### Destination updates

Only after both dependency calls return does the function perform its explicit destination stores.

First, it reads the current C-derived working word and writes its 32-bit value at offset 0x40 from original A.

Second, it reads the current D-derived companion word and writes its 32-bit value at offset 0x44 from original A.

This order is significant. The function does not present these fields as a single atomic 64-bit update. If execution fails before the second store completes, the first field may already have changed. No rollback is shown.

The destination base is the original A, not the potentially changed A-derived working word. A first-dependency change to that working word can affect the second call and the returned low half without redirecting the explicit destination stores.

If no dependency changes the D-derived companion word, the second destination store writes D unchanged. It is not established that the companion always remains unchanged: the second dependency receives the address of the immediately preceding word, and the dependencies’ complete memory effects are unknown.

### Return formation

After the destination stores, the function uses the current B-derived and A-derived working words to form the return value.

The return is therefore based on their values at the end of the operation, not expressly on snapshots captured immediately after the first dependency. Under the ordinary case where later operations do not modify those words, the return reflects the first dependency’s results.

No allocation, deallocation, loop, branch, retry, explicit error conversion, or explicit cleanup appears in the wrapper.

## Side effects

### Direct effects

The wrapper’s direct effects are:

- Initialization of invocation-local working storage.
- Two ordered dependency calls.
- A 32-bit destination store at original A plus 0x40.
- A subsequent 32-bit destination store at original A plus 0x44.

There is no explicit global access in the wrapper. This does not establish that the overall operation is global-state-free: both dependencies may access or change globals.

### Ownership and lifetime

The wrapper does not visibly allocate, transfer ownership of, or release the destination object. It treats A as an address usable for the final stores.

The working-storage addresses passed to dependencies refer to storage belonging to the active invocation. They do not establish a caller-owned result buffer or an explicit heap allocation. The evidence does not establish whether either dependency retains an address, transfers it elsewhere, or releases unrelated resources.

No ownership or release behavior should be attributed to a dependency solely because the wrapper ignores its return value.

### Errors and partial completion

There is no explicit error result or success test.

If the first dependency does not return normally, the second dependency and both explicit destination stores are not reached through the shown sequence.

If the second dependency does not return normally, neither explicit destination store is reached.

If a destination store fails, earlier dependency effects are not undone by any shown mechanism. A failure at the second store may leave the first destination field updated.

A dependency’s returned value does not select an error path: both returns are discarded.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-6a10d0fb9111 | Called once before the second dependency, with original B and addresses of the B-derived and A-derived working words. | Meaning of B; read/write extent; result semantics; failure behavior; global effects; ownership effects; address retention. |
| F-7551853739b9 | Called once after the first dependency returns, with the C-derived word’s address and the current B-derived and A-derived words by value. | Output extent; whether the companion word is written; result semantics; failure behavior; global effects; ownership effects; address retention. |

Normal completion of this wrapper implies one completed call to each dependency, in the listed order. It does not imply that either dependency semantically succeeded.

## Acceptance examples

These examples are conditional requirements derived from the visible sequence. They are not runtime observations. Unless stated otherwise, assume valid destination storage, normal dependency returns, and no additional aliasing or side effects beyond those specified.

### Unchanged working words

Suppose:

- A is 0x00100000.
- B is 0x11223344.
- C is 0x55667788.
- D is 0x99AABBCC.
- Neither dependency changes any working word.

Required results:

- F-6a10d0fb9111 receives 0x11223344 as its first argument.
- F-7551853739b9 receives 0x11223344 and 0x00100000 as its second and third arguments.
- Address 0x00100040 receives the 32-bit value 0x55667788.
- Address 0x00100044 then receives the 32-bit value 0x99AABBCC.
- The returned bit pattern is 0x1122334400100000.

### Changed pair does not redirect the destination

Suppose original A is 0x00100000, and the first dependency changes:

- The A-derived word to 0x00200000.
- The B-derived word to 0xAABBCCDD.

If later operations leave those words unchanged:

- The second dependency receives 0xAABBCCDD and 0x00200000.
- The explicit destination stores still target 0x00100040 and 0x00100044.
- The return is 0xAABBCCDD00200000.

Writing instead to 0x00200040 and 0x00200044 would violate the established destination selection.

### Conditional two-word output

Suppose the second dependency’s actual contract permits writing both consecutive C-derived and D-derived words, and it leaves them as 0x01020304 and 0xA0B0C0D0.

The wrapper must copy those values to offsets 0x40 and 0x44, respectively. This example does not establish that the dependency actually has a two-word output contract.

### Full-width packing

Suppose the final B-derived word is 0x80000000 and the final A-derived word is 0xFFFFFFFF.

The returned bit pattern must be 0x80000000FFFFFFFF. Treating the low word as signed and extending it across the high half would not match the packing operation.

### Dependency return values do not gate stores

Suppose both dependencies return normally, but either returns a value that some external convention might call an error.

The wrapper still proceeds through the shown calls and destination writes. It contains no test of either returned value. This does not establish that such an external convention exists.

## Uncertainties

The following are **documented_only**:

- Application-level meanings of the arguments, output fields, and packed return.
- Complete signatures and behavioral contracts of both dependencies.
- Whether the second dependency treats its first argument as a one-word output, a two-word output, or another layout.
- Whether either dependency retains working-storage addresses or accesses additional storage.
- Global-state changes, allocation, release, and ownership effects inside dependencies.
- Target byte order and external ABI details. The destination values and return-half placement are established independently of byte serialization.
- Exact behavior for invalid, wrapping, or unaligned destination addresses.
- Concurrency, reentrancy, synchronization, and visibility guarantees.
- The effect of unusual aliasing between the destination and invocation-local storage.
- Exception, trap, cancellation, or other nonlocal-exit behavior beyond interruption of the visible sequence.

An inferred intent is that the first dependency prepares a pair of values and the second prepares data for two adjacent destination fields. That interpretation is plausible but is not required to explain the established behavior.

## Evidence

The supplied static representation establishes four 32-bit argument-derived working words, two dependency invocations in a fixed order, two explicit 32-bit destination stores at offsets 0x40 and 0x44, and a 64-bit return assembled from two 32-bit halves.

It also distinguishes the original destination base from the mutable A-derived working word, shows which arguments are passed by value versus by address, and shows that both dependency return values are unused.

No dependency bodies, caller contracts, object definitions, ABI documentation, or runtime observations were provided. No runtime tests have been performed. Dependency effects and application-level interpretations remain conditional wherever the wrapper alone cannot establish them.
