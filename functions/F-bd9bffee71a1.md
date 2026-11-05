<!-- Trusted provenance: F-bd9bffee71a1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bd9bffee71a1

## Purpose

F-bd9bffee71a1 initializes traversal state, repeatedly requests an entry, and selects one of two actions using byte-valued fields in the current entry and an original owner object.

For each successful traversal request:

- If the entry’s flag byte and the original owner’s flag byte are both zero, it calls F-353d18cfd7f5 with the original owner address.
- Otherwise, it obtains a 32-bit value from F-34c6a5fd38ed, passes that value and an entry-relative address to F-bdea19dff782, calls F-e0a692eba8b7, and then writes the byte value 2 to an entry status location.

After a traversal request returns zero, it calls F-d0d3f100effc and returns a 64-bit value assembled from two mutable 32-bit local words.

“Owner,” “entry,” “flag,” “status,” and “traversal” describe observable roles, not established application types. Neither the domain meaning of these operations nor the semantic meaning of the returned 64-bit value is established.

## Interface

### Inputs

The supplied representation has four 32-bit arguments:

| Argument role | Established use |
|---|---|
| Original owner address | Used directly for conditional owner-byte reads and calls to F-353d18cfd7f5. Also copied into a mutable local word. |
| Traversal-source address | Advanced by eight bytes for initialization. Also copied into the local word subsequently used as traversal output. |
| Initial state word A | Copied into the first of two adjacent 32-bit state words. Its address is passed to traversal dependencies. |
| Initial state word B | Copied into the second state word. This function does not otherwise access it directly. |

The first two arguments are address-bearing 32-bit values. The supplied representation does not establish their original declared types.

### Local storage roles

Four local words receive the four argument values before the first dependency call:

- **Mutable owner word:** initially the original owner address.
- **Traversal-output word:** initially the traversal-source address; explicitly cleared after initialization.
- **First state word:** initially state word A.
- **Second state word:** initially state word B.

The two state words are adjacent in the supplied representation and together occupy eight bytes. This does not establish that a dependency receiving the first word’s address accesses exactly four bytes, exactly eight bytes, or any particular structured object.

The mutable owner word and traversal-output word are separate from the original argument values. Changes to those local words do not redirect this function’s uses of the original owner argument.

### Addressed layouts

| Base role | Byte offset | Direct operation by this function |
|---|---:|---|
| Original owner | 114 | Conditionally read one byte |
| Traversal source | 8 | Form an address for F-8ca21d3a8cca |
| Current traversal output | 4 | Form an address for F-bdea19dff782 |
| Current traversal output | 8 | Read one byte after each nonzero traversal result |
| Current traversal output | 9 | Write one byte with value 2 on the transformed-entry branch |

The offset-4 address is passed to a dependency rather than directly dereferenced here. Its target’s width, representation, and ownership are unknown.

### Output

Normal completion returns a 64-bit bit-pattern assembled as follows:

- Bits 31–0 are the final contents of the mutable owner word.
- Bits 63–32 are the final contents of the traversal-output word.

Both components are taken after F-d0d3f100effc returns. Assembly preserves each component’s 32-bit pattern; it does not sign-extend either component.

The low word is not guaranteed to equal the original owner address. The high word is not guaranteed to be zero when traversal ends. No success, error, pointer, or ownership interpretation is established for the combined value.

### Preconditions

A defined normal execution requires:

- Dependencies to accept the supplied arguments and exposed local storage.
- Every nonzero traversal result to leave a usable address for the following offset-8 byte read.
- The original owner’s offset-114 byte to be readable whenever the entry flag is zero.
- The traversal-output address used at the final transformed-branch store to permit an offset-9 byte write.
- Addresses passed to dependencies to satisfy those dependencies’ own requirements.
- Address calculations to remain within the supported address model.

There is no explicit null, bounds, alignment, lifetime, or overflow validation. The evidence does not establish wrapping address arithmetic or wider host-pointer semantics. The acceptance examples below assume representable, accessible addresses without overflow.

## Behavior

### 1. Establish local values

Before calling any dependency, the function copies all four input words into their corresponding local storage.

The mutable owner word begins with the original owner address. Its address is later passed to dependencies, so its value cannot be treated as immutable.

The traversal-output word begins with the traversal-source address. The two state words begin with the supplied state values.

### 2. Initialize traversal

F-8ca21d3a8cca is called with:

1. The address of the first state word.
2. The traversal-source address plus eight bytes.

On entry to this call, the traversal-output word contains its initial copy of the traversal-source argument. This is an entry-state assertion only. The source does not establish that the initializer preserves this word, the mutable owner word, or other reachable storage throughout the call. Mutation through a valid alias remains unknown.

After the initializer returns normally, the function explicitly writes zero to the traversal-output word. This occurs after initialization, regardless of that word’s value at initializer return.

No corresponding reset of the mutable owner word occurs. If initialization changes that word through a valid alias, the changed value remains relevant to later operations and result assembly.

### 3. Request an entry

F-b38a3aba0e18 is called with:

1. The address of the first state word.
2. The address of the traversal-output word.

Its result is tested against zero:

- Zero ends traversal.
- Any nonzero result enters the loop body.

The result need not be exactly 1. Its sign does not affect the zero-versus-nonzero decision.

The function clears the traversal-output word only once, between initialization and the first traversal request. It does not clear the word before later requests. Later calls therefore receive whatever value preceding operations left there.

After a nonzero result, the function reads one byte at offset 8 from the address then held in the traversal-output word. There is no separate validation of that address.

### 4. Select the action

The entry byte is tested for equality to zero.

If it is nonzero, the transformed-entry branch is selected without a direct owner-flag read.

If it is zero, the function reads one byte at offset 114 from the original owner address:

- A zero owner byte selects the owner-action branch.
- A nonzero owner byte selects the transformed-entry branch.

Only zero versus nonzero matters. All nonzero byte patterns select the same branch.

The owner flag is read anew on each iteration whose entry flag is zero. It is not cached across iterations. Dependency effects on owner memory can therefore affect later branch choices.

### 5. Owner-action branch

When both tested bytes are zero, F-353d18cfd7f5 is called with the original owner address.

For this iteration, the function does not directly:

- Call F-34c6a5fd38ed.
- Call F-bdea19dff782.
- Call F-e0a692eba8b7.
- Write the status byte at offset 9.

The absence of a direct write does not establish that entry memory is unchanged: F-353d18cfd7f5 has an unknown memory-effect contract.

After the call returns normally, execution proceeds to another traversal request.

### 6. Transformed-entry branch

When either tested flag is nonzero, the function performs these operations in order:

1. Calls F-34c6a5fd38ed with the mutable owner word’s address and two zero-valued arguments.
2. Retains that call’s 32-bit result.
3. Calls F-bdea19dff782 with the address currently held in the traversal-output word plus four bytes, followed by the retained result.
4. Calls F-e0a692eba8b7 with the mutable owner word’s address.
5. Writes the byte value 2 at offset 9 from the address then held in the traversal-output word.

The offset-4 address is obtained after F-34c6a5fd38ed returns. The final store’s address is obtained after F-e0a692eba8b7 returns.

The entry address is not fixed for the whole branch by an explicit snapshot. If a dependency can modify the traversal-output word through a valid alias, the initial flag read, offset-4 dependency argument, and final status store need not refer to the same entry. Whether such an alias exists is unknown; preservation must not be assumed.

Both F-34c6a5fd38ed and F-e0a692eba8b7 receive the mutable owner word’s address. Changes to that word affect subsequent uses of the word and the eventual low return component. They do not redirect direct owner-flag reads or F-353d18cfd7f5, which use the original owner argument.

The 32-bit result from F-34c6a5fd38ed is forwarded regardless of its bit pattern. There is no failure test on that value. Other dependency results in this branch are unused.

The status store occurs only if all three calls return normally and execution reaches the store. Afterward, execution proceeds to another traversal request.

### 7. Terminate and return

When F-b38a3aba0e18 returns zero, the function calls F-d0d3f100effc with the first state word’s address.

After that call returns normally, the function assembles the return value from the mutable owner word and traversal-output word. It does not explicitly reset either component after the terminating traversal request or after the final dependency call.

A traversal result of zero therefore does not itself constrain the returned upper word.

## Side effects

The function directly performs:

- Four initial local-word copies.
- One explicit clearing of the traversal-output word after initialization.
- An offset-8 entry-byte read for every nonzero traversal result.
- An offset-114 original-owner-byte read only when the entry byte is zero.
- An offset-9 byte write of 2 for every transformed-entry branch that reaches its final store.

Dependency calls may have additional externally observable effects. No dependency is assumed to preserve globals, reachable memory, or exposed local storage without its own supporting contract.

This function has no explicit allocation, deallocation, retention, or ownership-transfer operation. That does not prove the dependencies lack such operations. In particular, neither F-e0a692eba8b7 nor F-d0d3f100effc is established to release memory merely because of its position in the sequence.

Local addresses are exposed to dependencies. Whether a dependency retains those addresses, whether retention is permitted, and whether another valid alias exists are unknown. The function establishes no local-storage lifetime beyond its invocation.

There is no explicit error-reporting or recovery branch. A dependency’s failure behavior is unspecified here. If a call fails to return normally, subsequent operations—including the status store or final traversal call—are not guaranteed to occur.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-8ca21d3a8cca | Called once after local initialization, with the first state word’s address and traversal-source address plus eight bytes. |
| F-b38a3aba0e18 | Repeatedly receives the first state word’s address and traversal-output word’s address. Zero terminates; nonzero causes entry processing. |
| F-353d18cfd7f5 | Receives the original owner address only when both tested bytes are zero. Its result is unused. |
| F-34c6a5fd38ed | Receives the mutable owner word’s address and two zeros on the transformed-entry branch. Produces the 32-bit value forwarded to F-bdea19dff782. |
| F-bdea19dff782 | Receives the then-current traversal-output address plus four bytes and the preceding 32-bit result. Its result is unused. |
| F-e0a692eba8b7 | Receives the mutable owner word’s address after F-bdea19dff782 and before the direct status store. Its result is unused. |
| F-d0d3f100effc | Receives the first state word’s address once after normal traversal termination and before return assembly. Its result is unused. |

These interactions establish neither dependency ownership rules nor preservation guarantees.

## Acceptance examples

These are conditional acceptance criteria, not executed tests.

### Empty traversal

If initialization returns normally and the first F-b38a3aba0e18 call returns zero:

- No entry flag or owner flag is directly read.
- Neither per-entry action executes.
- F-d0d3f100effc is called.
- The return value uses the two component words as they stand after that final call.

If, additionally:

- F-8ca21d3a8cca preserves the mutable owner word, so it still contains the original owner argument after initialization;
- The first traversal request and F-d0d3f100effc also preserve that owner word; and
- The first traversal request and F-d0d3f100effc preserve the zero placed in the traversal-output word after initialization;

then the low return word equals the original owner argument and the high word is zero.

Initializer preservation of the traversal-output word is unnecessary for this conclusion because the function explicitly clears that word afterward. Initializer preservation of the owner word is necessary unless its post-initialization value is otherwise established.

### Both flags zero

Given a nonzero traversal result, an accessible entry byte of zero at offset 8, and an accessible original-owner byte of zero at offset 114:

- The owner byte is read after the entry byte.
- F-353d18cfd7f5 receives the original owner address.
- The transformed-entry calls do not execute for that iteration.
- No direct status-byte store occurs for that iteration.

### Nonzero entry flag

Given a nonzero traversal result and an entry flag of 1, 2, or 255:

- No direct owner-byte read is performed for branch selection.
- F-34c6a5fd38ed, F-bdea19dff782, and F-e0a692eba8b7 execute in that order.
- If they return normally and the final target is writable, the function then stores byte value 2 at offset 9 from the then-current traversal-output address.

### Zero entry flag and nonzero owner flag

Given a zero entry flag and a nonzero original-owner flag, the transformed-entry sequence executes after both reads. The zero entry flag alone does not select the owner-action branch.

### Mutable owner value

If F-34c6a5fd38ed changes the mutable owner word and F-e0a692eba8b7 changes it again, the later value becomes the low return component provided no subsequent operation changes it.

Direct owner-flag reads and F-353d18cfd7f5 still use the original owner argument.

### Nonzero upper return component

If the terminating traversal call returns zero while leaving a nonzero traversal-output word, and F-d0d3f100effc preserves that word, the nonzero pattern becomes the upper 32 bits of the result.

### Changed final store target

If a dependency changes the traversal-output word through a valid alias before the final status store, that store uses the changed address. It is not required to target the entry whose flag was initially read. This example is conditional on a valid alias; the source does not establish one.

## Uncertainties

### Established static evidence

The supplied body establishes argument-copy ordering, call ordering, zero/nonzero predicates, conditional byte reads, the final byte store, exposed local addresses, and 64-bit result assembly.

The initializer’s entry state is established. Preservation of that state throughout or after the call is not.

### Inferred intent

The initialization, repeated requests, and final dependency call suggest a traversal lifecycle. The offset-4 operation followed by a byte value of 2 suggests an entry-associated update and state transition. These interpretations are not required guarantees.

### documented_only

The following remain unresolved:

- Concrete object types, sizes, alignment requirements, and field meanings.
- Dependency implementations, errors, ownership rules, and memory effects.
- Dependency access extents through supplied addresses.
- Existence, validity, or retention of aliases to local storage.
- Traversal order, repeated entries, and termination guarantees.
- Concurrency permissions and synchronization.
- Semantic interpretation of the 64-bit result.
- External byte order for any stored representation of that result.
- Address-overflow behavior and platform-specific address semantics.

All argument and return-component words are 32 bits. Offsets 4, 8, 9, and 114 are byte offsets, not evidence of element-scaled indexing. No modular wrapping, signed-overflow rule, or wider-pointer interpretation is selected by this contract.

## Evidence

This contract is based on static inspection of the supplied function body and the supplied opaque dependency mapping. No dependency bodies, external type definitions, or platform ABI specification were supplied.

No runtime tests have been performed. The acceptance examples describe conditional consequences of the visible operations, not observed executions.
