<!-- Trusted provenance: F-4a73b9be0f3a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4a73b9be0f3a

## Purpose

F-4a73b9be0f3a performs an ordered sequence of dependency calls, four-byte memory transfers, arithmetic, and initialization-related writes. It obtains a context address, stores a global-derived value through that address, obtains an anchor address, copies five fields from an incoming record into anchor-relative storage, adjusts one copied field, invokes two further dependencies, and returns an address 584 bytes before the anchor.

These operations are established by the supplied static representation. Object construction or initialization is inferred intent, not an established type or ownership contract. The evidence does not establish the allocation boundaries, lifetime, destruction procedure, or semantic meaning of the resulting storage.

The order of accesses matters. The incoming record is not captured as a complete snapshot, dependencies can affect subsequently read memory, and indirect stores may modify global storage.

## Interface

### Effective input

The representation declares no ordinary parameters but consumes an incoming stack value. This contract therefore identifies one effective input:

- **Incoming record address:** an address-valued quantity represented in the function’s 32-bit integer domain.

The exact calling convention, stack-slot arrangement, and public function signature are documented_only. The apparent absence of declared parameters does not establish that the function requires no input.

The original incoming record address is passed to F-c5a3e2a1a1d4 before the function directly reads the following fields:

| Incoming byte offset | Access width | Direct use |
|---|---:|---|
| `+0x68` | 4 bytes | Copied to anchor offset `−0x1c` |
| `+0x6c` | 4 bytes | Copied to anchor offset `−0x18` |
| `+0x70` | 4 bytes | Copied to anchor offset `−0x14` |
| `+0x74` | 4 bytes | Copied to anchor offset `−0x10`, which is subsequently adjusted |
| `+0x78` | 4 bytes | Copied to anchor offset `−0x0c`; the same read value supplies the subtraction operand |

These are record-relative locations, not five independently supplied arguments. Their public types and meanings are unknown.

### Intermediate addresses and values

The function uses three dependency results:

- **Context address:** returned by F-7f6becd152e6. It receives a four-byte store at offset zero. Its byte offset `+0x220` is passed to F-5b24edfb22c0.
- **Intermediate result:** returned by F-5b24edfb22c0. Adding `0x28` produces the first argument to F-c5a3e2a1a1d4.
- **Anchor address:** returned by F-c5a3e2a1a1d4. Subsequent direct field accesses and the returned address are expressed relative to this anchor.

The context displacement is **544 bytes**, not 136 bytes: the supplied representation advances a pointer to four-byte elements by `0x88` elements.

These descriptions identify address uses, not allocation ownership or public pointer types.

### Result

On normal completion, the function returns:

**anchor address − `0x248`**, meaning 584 bytes before the anchor.

The exposed result is a 32-bit integer quantity used as an address. A public pointer type is not established.

The final dependency’s return value is ignored. It does not replace the saved anchor as the basis for this function’s return value.

### Preconditions for normal completion

The visible operations require:

- Readable four-byte source fields at the incoming offsets listed above.
- A readable global-derived value for the context store and an accessible global-derived argument for the anchor-producing call.
- Writable four-byte destinations at the context address and specified anchor-relative locations.
- Readable storage at anchor offset `−0x10` when the arithmetic step rereads it.
- Dependency results that permit the subsequent address calculations, memory accesses, and calls.
- Dependencies that accept the supplied arguments and return normally.

There are no visible null, bounds, alignment, or failure-result checks. Required alignment and address validity rules are not independently established. These preconditions do not promise that a dependency preserves the incoming record, globals, or any previously obtained storage.

## Behavior

### Context acquisition and first store

F-7f6becd152e6 is called without explicit arguments.

After it returns, the function reads a global-derived four-byte value and stores it at offset zero of the returned context address. This establishes one four-byte transfer, not a larger header copy.

The global read occurs after the dependency returns. Any value observed before that call need not be the value stored.

F-5b24edfb22c0 is then called with the context address plus `0x220` bytes. The context store precedes this call.

### Anchor acquisition

After F-5b24edfb22c0 returns, F-c5a3e2a1a1d4 is called with these five positional arguments:

1. The intermediate result plus `0x28`.
2. A value read from a second global location.
3. The numeric constant `0x6c`, or 108.
4. The numeric constant `5`.
5. The original incoming record address.

The second global is read after F-5b24edfb22c0 returns. Neither preceding dependency is assumed to preserve it.

The returned value becomes the anchor address. The meanings of `0x6c` and `5` are unknown. In particular, `0x6c` is not established as an allocation size, record size, or guarantee of accessible anchor-relative storage.

### Ordered field transfers

After the anchor-producing dependency returns, the function performs the following four-byte transfers in order:

| Order | Source relative to incoming record | Destination relative to anchor |
|---:|---|---|
| 1 | `+0x6c` | `−0x18` |
| 2 | `+0x68` | `−0x1c` |
| 3 | `+0x70` | `−0x14` |
| 4 | `+0x74` | `−0x10` |
| 5 | `+0x78` | `−0x0c` |

Each source is read before its corresponding destination is written. There is no visible complete-record snapshot before the first destination write.

Consequently:

- Values modified by earlier dependencies are read in their modified state.
- An earlier destination write can affect a later source read if storage overlaps.
- These transfers cannot generally be replaced with reads of five immutable entry-time values.

The value read for the fifth transfer is retained for the arithmetic step. That step does not perform a second source read at incoming offset `+0x78`.

### Subtraction and negative-result clamp

After writing the fifth copied field, the function reads the four-byte value currently at anchor offset `−0x10`.

It subtracts the retained fifth-source value and stores the subtraction result back at anchor offset `−0x10`. It then compares that result with zero using a signed comparison:

- A negative result causes an additional four-byte store of zero at the same destination.
- A zero or positive result causes no additional clamp store.

The subtraction-result store precedes the conditional zero store even when the result is negative. A description that includes only the final clamped value omits an established write.

For non-overlapping storage and signed 32-bit subtraction without overflow, the value immediately after this step is the greater of zero and the fourth copied value minus the fifth copied value.

That mathematical description has two limits:

1. With overlap, the destination reread may observe a value changed by a later copy rather than the fourth source value.
2. The representation establishes 32-bit arithmetic operands and a signed negative-result test, but does not independently establish behavior for signed-overflow cases. Such cases remain documented_only; neither saturation nor wraparound is promised.

### First anchor-relative dependency

F-438680ffebfe is called with:

1. Anchor address minus `0x28`.
2. A fixed embedded address.

The second argument is address-valued. Its target’s type, role, and invocation contract are unknown. This specification does not assume callback semantics.

The dependency’s return value is ignored. Its memory and ownership effects remain unknown.

### Explicit clearing

After F-438680ffebfe returns, the function performs two four-byte stores in this order:

1. Zero at anchor offset `−0x20`.
2. Zero at anchor offset `−0x04`.

These stores do not occur before the preceding call. F-438680ffebfe is therefore not guaranteed to observe zero at either location.

### Final dependency and return

F-861a66c91b48 is called with four positional arguments:

1. Anchor address minus `0x248`.
2. Anchor address plus `0x220`.
3. Zero.
4. Zero.

After normal return from this call, F-4a73b9be0f3a returns anchor address minus `0x248`.

There are no direct field stores after this dependency call. The function therefore does not restore any field that the final dependency changes.

## Side effects

### Direct writes

The function directly performs:

- One four-byte global-derived store through the context address.
- Five four-byte field-copy stores near the anchor.
- One four-byte subtraction-result store at anchor offset `−0x10`.
- One additional four-byte zero store there if the signed result is negative.
- Two four-byte zero stores after F-438680ffebfe returns.

This totals **nine direct four-byte stores** on the nonnegative-result path and **ten** on the negative-result path. The totals count stores, not distinct destinations.

No visible store explicitly names a global symbol as its destination. Stores through the context and anchor-relative addresses may nevertheless modify global storage; their storage class and aliasing relationships are unknown. Dependencies may also modify globals.

### Aliasing and observation order

No non-aliasing guarantee is established among:

- The incoming record.
- The context destination.
- Anchor-relative destinations.
- The global-derived input locations.
- Storage read or written by dependencies.

Thus, a direct store can affect a later global-derived read if their addresses alias. Likewise, an anchor-relative destination can itself designate global storage.

The function’s access order must be retained when such aliasing is possible. In particular, the arithmetic step rereads its destination after the fifth field-copy store.

The listed ordering is the ordering expressed by the supplied representation. It does not establish volatile semantics, hardware-atomic accesses, synchronization, or inter-thread visibility guarantees.

### Ownership and lifetime

No explicit release operation appears in the visible body.

This does not establish that dependencies avoid allocation, release, ownership transfer, or invalidation. Nor does returning an anchor-relative address establish that the caller owns it.

The lifetime and release responsibility for the incoming record, context storage, intermediate result, anchor-associated storage, and returned address are documented_only.

### Errors and partial effects

There is no visible error-result branch, retry, rollback, or cleanup path. Dependency results are used without visible validation.

If a memory access fails or a dependency does not return normally, subsequent operations and ordinary return are not established to occur. Prior stores are not visibly undone.

The evidence provides no recoverable error code or defined failure sentinel. Fault behavior, exception propagation, and nonlocal control flow are documented_only.

## Dependencies

| Opaque function ID | Established invocation | Unresolved contract |
|---|---|---|
| F-7f6becd152e6 | No explicit arguments; returns the context address | Result validity, allocation, ownership, failure representation, memory and global effects |
| F-5b24edfb22c0 | Receives context address plus `0x220`; its result is subsequently adjusted by `0x28` | Result meaning, accessible extent, mutation, lifetime, failure behavior |
| F-c5a3e2a1a1d4 | Receives adjusted intermediate result, a global-derived value, `0x6c`, `5`, and incoming record address; returns anchor | Constant meanings, storage boundaries, ownership, source mutation, failure behavior |
| F-438680ffebfe | Receives anchor minus `0x28` and a fixed embedded address | Address meaning, memory effects, ownership effects, exceptional behavior |
| F-861a66c91b48 | Receives anchor minus `0x248`, anchor plus `0x220`, zero, zero | Initialization semantics, final memory contents, ownership effects, exceptional behavior |

The call order is exactly the table order. An ignored return value does not imply an absence of side effects.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless stated otherwise, they assume valid non-overlapping storage, normal dependency returns, and no external interference.

### Positive difference

If incoming offsets `+0x74` and `+0x78` contain signed values 20 and 7 when read:

- The fourth copy stores 20 at anchor offset `−0x10`.
- The fifth copy stores 7 at anchor offset `−0x0c`.
- The arithmetic step stores 13 at anchor offset `−0x10`.
- No conditional zero store occurs.

Later dependency effects are outside this intermediate-state guarantee.

### Negative difference

For source values 4 and 9, the arithmetic step stores −5 at anchor offset `−0x10`, then stores zero there.

Writing only zero would not preserve the established sequence.

### Equal operands

For source values 9 and 9, the subtraction-result store writes zero. The negative-result branch is not taken, so it does not add another store.

### Mutation before copying

If F-c5a3e2a1a1d4 modifies incoming offset `+0x68` before returning, the second field transfer reads the subsequently present value. An entry-time snapshot is not the contract.

### Overlapping source and destination

If anchor offset `−0x18` is the same location as incoming offset `+0x68`, the first copy writes into that source location.

The second copy then reads the updated value. It need not reproduce the value originally present at incoming offset `+0x68`.

### Direct mutation of global storage

If the context address designates writable global storage, the first direct store modifies that global storage.

If this destination also aliases the second global-derived argument location, and no intervening operation changes it, the later read for F-c5a3e2a1a1d4 observes the stored value. Direct global mutation does not require a dependency to perform the write.

### Initialization order

F-438680ffebfe is called before either explicit clearing store. F-861a66c91b48 is called after both.

If the final dependency changes either cleared location, the function does not clear it again.

### Return address

For an anchor of `0x10001000`, with valid address arithmetic and normal completion, the function returns `0x10000db8`.

The unused result of F-861a66c91b48 does not change that calculation.

### Overflow boundary

An operand pair whose mathematical difference lies outside signed 32-bit range does not have an established clamped numeric result under this contract. It is a documented_only case, not a required wraparound or saturation example.

## Uncertainties

The following remain documented_only:

- Exact calling convention, input stack arrangement, and public signature.
- Public types and semantic meanings of fields and addresses.
- Values and meanings of both global-derived inputs.
- Whether the global-derived locations are writable or alias indirect destinations.
- Storage classes of the context and anchor-relative destinations, including whether they are global storage.
- Meaning and target type of the fixed embedded address.
- Allocation boundaries, ownership, lifetime, and release responsibility.
- Dependency failure representations and exceptional control flow.
- Signed-overflow and address-wrap behavior.
- Alignment requirements, byte order, and externally observable access granularity.
- Concurrency and synchronization guarantees.
- Final memory contents after dependency effects.

Initialization is plausible intent. It does not establish a fully initialized object, a particular class layout, or transferred ownership.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Established evidence comprises the explicit dependency sequence, four-byte accesses, byte displacements, constants, retained subtraction operand, signed negative-result comparison, conditional zero store, and return-address calculation.

The correction concerning global mutation follows from the destinations themselves: stores use dependency-returned addresses and anchor-relative addresses whose storage classes are not identified. Absence of an explicitly named global destination does not exclude direct writes to global storage.

No dependency bodies, calling-convention specification, allocation metadata, public type definitions, or runtime observations were supplied. No runtime tests have been performed. Acceptance examples describe conditional obligations, not measured outcomes.
