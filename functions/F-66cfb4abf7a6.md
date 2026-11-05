<!-- Trusted provenance: F-66cfb4abf7a6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-66cfb4abf7a6

## Purpose

F-66cfb4abf7a6 performs an ordered sequence of operations on three groups of 32-bit fields within an input object. It then derives a destination address through four dependent calls, writes a 32-bit value obtained from global storage to that destination, invokes a final dependency on the destination, and returns its address.

The established behavior is the ordering of calls, field reads, arithmetic, one explicit destination write, and the returned address. The available evidence does not establish the semantic type of the object or whether the operation constitutes destruction, clearing, initialization, or another lifecycle transition. Those interpretations are **documented_only**, not contractual guarantees.

## Interface

### Input

The function accepts one address-bearing argument represented as a 32-bit integer in the supplied static evidence. The argument is used as the base address of an object; it is not itself read as an object value.

The directly accessed object fields are:

| Offset from input address | Width | Direct use |
|---|---:|---|
| `0x2c` | 32 bits | Argument to a group dependency; subsequently reread as an address-bearing value |
| `0x30` | 32 bits | Argument to the same group dependency |
| `0x34` | 32 bits | Subsequently read for a subtraction |
| `0x38` | 32 bits | Argument to another group dependency; subsequently reread as an address-bearing value |
| `0x3c` | 32 bits | Argument to the same group dependency |
| `0x40` | 32 bits | Subsequently read for a subtraction |
| `0x44` | 32 bits | Argument to a third group dependency; subsequently reread as an address-bearing value |
| `0x48` | 32 bits | Argument to the same group dependency |
| `0x4c` | 32 bits | Subsequently read for a subtraction |

The address at input offset `0x24` is also passed to a dependency. The function does not directly load a value from that location before passing its address.

For successful execution, the input and all subsequently derived addresses must support the accesses actually performed by this function and its dependencies. There is no explicit null check, range check, alignment check, or object-type check in the available evidence.

### Output

On normal completion, the function returns the address of a 32-bit storage location. It explicitly writes that location before the final dependency call.

The returned address is derived from dependency results. It is not established to equal the input address, to lie within the input object, or to identify newly allocated storage.

The return value is an address, not a status code. No separate error result is visible.

### Representation constraints

The visible fields, address-bearing intermediate values, global value, and explicit store are 32 bits wide. Offsets are byte offsets.

The evidence does not establish byte order, a wider-host calling convention, or portable behavior for addresses that cannot be represented in 32 bits. No 64-bit arithmetic or widening is shown.

## Behavior

### 1. Process the group beginning at offset `0x44`

The function first calls F-dde71669704e with three arguments:

- The address of the field at input offset `0x44`.
- The 32-bit value read from input offset `0x44`.
- The 32-bit value read from input offset `0x48`.

Both value arguments are obtained before that call begins. Their relative read order is not established.

After F-dde71669704e returns, the function rereads the field at offset `0x44`. This fresh value is used for the next subtraction and as the first argument to a later call. The function also reads the field at offset `0x4c` after F-dde71669704e returns.

It subtracts the freshly read value at `0x44` from the value at `0x4c`, using the visible 32-bit integer representation. It passes the result and the constant `0x1c`, decimal 28, to F-870c4fcf7295.

After F-870c4fcf7295 returns, the function calls F-ff1bdc376dcd with:

1. The value previously reread from offset `0x44`.
2. The 32-bit result returned by F-870c4fcf7295.
3. Zero.

In particular, the first argument is retained across the call to F-870c4fcf7295; it is not freshly loaded afterward. A change to the field during that dependency does not, by itself, change this retained argument.

The available evidence does not establish what F-870c4fcf7295 computes. Its second argument being 28 is insufficient to guarantee division, element counting, rounding, or validation.

### 2. Process the group beginning at offset `0x38`

After the preceding F-ff1bdc376dcd call returns, the function calls F-660635c2f259 with:

- The address of the field at input offset `0x38`.
- The current 32-bit value at offset `0x38`.
- The current 32-bit value at offset `0x3c`.

These fields are read at this stage, not captured before the earlier group was processed. Earlier dependencies may therefore affect the values supplied here.

After F-660635c2f259 returns, the function rereads the field at offset `0x38` and reads the field at offset `0x40`. It subtracts the former from the latter and applies a signed right shift by four bits to the difference.

It then calls F-ff1bdc376dcd with:

1. The freshly read value from offset `0x38`.
2. The shifted difference.
3. Zero.

For a nonnegative, representable difference, the shift yields the integer quotient rounded down after division by 16. It does not test that the difference is a multiple of 16. Negative differences and overflowing subtraction require the representation qualifications described under Uncertainties.

### 3. Process the group beginning at offset `0x2c`

After the second F-ff1bdc376dcd call returns, the function calls F-0d68745132e3 with:

- The address of the field at input offset `0x2c`.
- The current 32-bit value at offset `0x2c`.
- The current 32-bit value at offset `0x30`.

After that dependency returns, the function rereads offset `0x2c` and reads offset `0x34`. It subtracts the value at `0x2c` from the value at `0x34`, then applies a signed right shift by five bits.

It calls F-ff1bdc376dcd with:

1. The freshly read value from offset `0x2c`.
2. The shifted difference.
3. Zero.

For a nonnegative, representable difference, the second argument is the quotient rounded down after division by 32. There is no visible divisibility check.

### 4. Derive the final destination

Only after all three group sequences complete does the function make four calls to F-289ec4311c64.

The first call receives the input address advanced by `0x24` bytes.

Each remaining argument depends on the preceding call’s returned value:

| Call | Argument |
|---|---|
| Second | First returned address minus 8 bytes |
| Third | Second returned address minus 8 bytes |
| Fourth | Third returned address minus 12 bytes |

The final destination is the fourth returned address minus 8 bytes.

These are four distinct calls, not one call followed by a fixed adjustment. The dependencies’ return values must be respected individually. The evidence does not establish that any call returns its input unchanged.

### 5. Write, finalize, and return

After the fourth F-289ec4311c64 call returns and the destination address is derived, the function reads a 32-bit value from a fixed global storage location and writes that value to the destination.

It then calls F-22abefe7f288 with the destination address.

If that call returns normally, the function returns the same destination address. No destination reload, result substitution, or explicit second store is shown after the final call.

The final dependency may alter the destination’s contents. Consequently, the explicit write does not establish the value present at that address when the function returns.

## Side effects

The only direct memory write visible in the function is the final 32-bit destination store. The three group dependencies receive addresses of fields inside the input object and may modify those fields or other accessible state. Their complete write sets are unknown.

The observable call order is:

1. F-dde71669704e.
2. F-870c4fcf7295.
3. F-ff1bdc376dcd.
4. F-660635c2f259.
5. F-ff1bdc376dcd.
6. F-0d68745132e3.
7. F-ff1bdc376dcd.
8. Four consecutive calls to F-289ec4311c64.
9. The global read and destination write.
10. F-22abefe7f288.

The global value is read after the four address-derivation calls, not at function entry. A dependency’s earlier change to that global may therefore affect the stored value.

No ownership transfer, allocation, deallocation, reference-count change, or resource release can be established solely from these calls. In particular:

- Passing a field address does not establish ownership of the referenced storage.
- Passing zero as a third argument does not establish clearing or releasing behavior.
- Returning a destination address does not establish that the destination remains valid after the final dependency.
- Unknown dependencies must not be assumed to preserve globals, input fields, or memory ownership.

There is no visible recovery, rollback, or error translation. If an access fails or a dependency does not return normally, later stages are not established to occur. Earlier side effects are not visibly undone.

## Dependencies

| Opaque function ID | Established interaction | Unknown contract |
|---|---|---|
| F-dde71669704e | Receives the address and two values from the `0x44` group | Mutation, ownership, validation, and failure behavior |
| F-870c4fcf7295 | Receives a 32-bit difference and 28; supplies the next call’s second argument | Arithmetic meaning and exceptional cases |
| F-ff1bdc376dcd | Called three times with an address-bearing value, a computed value, and zero | Meaning, argument interpretation, and memory effects |
| F-660635c2f259 | Receives the address and two values from the `0x38` group | Mutation, ownership, validation, and failure behavior |
| F-0d68745132e3 | Receives the address and two values from the `0x2c` group | Mutation, ownership, validation, and failure behavior |
| F-289ec4311c64 | Called four times in a return-value-dependent address chain | Mapping, returned-address validity, and side effects |
| F-22abefe7f288 | Receives the destination after the explicit store | Final contents, lifetime effects, and failure behavior |

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

### Post-call field changes affect subsequent arguments

Suppose F-dde71669704e changes offset `0x44` from `0x1000` to `0x2000` and leaves offset `0x4c` as `0x2054`.

The next F-870c4fcf7295 call must receive the difference 84 and the constant 28. The following F-ff1bdc376dcd call must receive `0x2000` as its first argument and the actual dependency return value as its second argument. It must not assume that return value is 3.

### Nonmultiple differences are not rejected locally

Suppose, after F-660635c2f259 returns, offset `0x38` contains `0x3000` and offset `0x40` contains `0x3023`.

The nonnegative difference is 35. The next F-ff1bdc376dcd call receives `0x3000`, 2, and zero. No local rejection of the remaining three bytes is visible.

Similarly, a nonnegative difference of 95 in the `0x2c` group yields a shifted value of 2, not 3.

### Zero differences still reach the dependency

If the post-call values at offsets `0x2c` and `0x34` are equal, F-ff1bdc376dcd is still called with a zero second argument. No local zero-count fast path is visible.

### Address derivation uses each returned value

Suppose the four F-289ec4311c64 calls return, respectively, `0x5008`, `0x6008`, `0x700c`, and `0x8008`.

After the initial input-relative call, the subsequent arguments must be `0x5000`, `0x6000`, and `0x7000`. The final destination must be `0x8000`.

The function must store the then-current global value at `0x8000`, pass `0x8000` to F-22abefe7f288, and return `0x8000` if that call returns normally.

### Final dependency effects remain observable

If F-22abefe7f288 overwrites the destination, the function does not restore its earlier stored value. If the dependency returns normally, the returned address remains the derived destination regardless of that content change.

## Uncertainties

The following are **documented_only**:

- Semantic names and types for the three field groups.
- Whether the field pairs describe ranges, capacities, or unrelated quantities.
- Whether the shifted values represent element counts.
- Whether F-870c4fcf7295 performs division by 28.
- The identity and semantic role of the global value.
- Dependency ownership, allocation, release, synchronization, and error contracts.
- Validity requirements for addresses returned by F-289ec4311c64.
- Thread-safety, reentrancy, and behavior under concurrent field mutation.

The static representation shows 32-bit signed subtraction and signed right shifts. For nonnegative differences within the signed 32-bit range, the stated numerical results are unambiguous. Exact behavior for subtraction overflow, negative shifted values, and address arithmetic overflow is not independently established. A portable contract must not silently replace these operations with unbounded arithmetic, unsigned shifts, or checked arithmetic.

The relative order of reads used to prepare a single call is also not established. The ordering across completed dependency calls and subsequent field rereads is established.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes the field offsets and widths, dependency identities and call order, post-call field rereads, constants 28, 4, and 5 in the arithmetic stages, four dependent address transformations, the late global read, the explicit 32-bit store, and return of the derived destination after the final call.

Lifecycle descriptions and container interpretations are inferred intent only and are not used as acceptance requirements. Dependency implementations and runtime observations were not supplied. No runtime tests have been performed.
