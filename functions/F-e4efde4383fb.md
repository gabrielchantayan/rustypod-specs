<!-- Trusted provenance: F-e4efde4383fb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e4efde4383fb

## Purpose

F-e4efde4383fb describes a stateful, call-driven loop that performs arithmetic, invokes several unresolved dependencies, writes through computed addresses, and eventually constructs a 64-bit return value.

Two conditions lead to the shared return sequence:

- A global gate is zero after a particular dependency call.
- On the nonzero-gate path, a value derived from a secondary result channel satisfies a signed, width-sensitive test.

The nonzero-gate path otherwise updates the state used by the next iteration. The available evidence does not establish a higher-level purpose such as rendering, decoding, allocation, or device control. Assigning one would exceed the evidence.

This contract distinguishes behavior visible in the supplied static representation from unresolved machine-level details. In particular, the apparent three-argument interface is incomplete: entry register state, indirect call targets, and secondary result channels affect execution.

## Interface

### Explicit inputs

The displayed interface has three explicit arguments:

| Input | Width | Observed use |
|---|---:|---|
| First argument | 32 bits | No explicit use appears in the supplied body. |
| Second argument | 32 bits | Passed to the first dependency call in each iteration; replaced on the continuation path. |
| Third argument | 32 bits | Incremented by 47 at the start of each iteration, passed to a dependency, and stored in memory. Replaced on the continuation path. |

The absence of an explicit use of the first argument does not establish that its incoming machine register is irrelevant to unresolved callees. It establishes only that the displayed body does not name that argument in an operation.

### Additional entry state

The body consumes additional state not exposed by the displayed argument list:

- A 32-bit base value used for address formation, a byte read, shifting, and XOR.
- An auxiliary 32-bit value passed to the first dependency.
- An offset added to the base value for a later dependency.
- An incoming link-related value used to calculate the first call target.
- Secondary result-channel values associated with dependency calls.
- Global storage supplying a gate, a dependency argument, an XOR operand, and the high half of the return value.
- An address within the current stack frame, passed to a dependency.

A valid caller must supply the machine-level context needed by these operations. The ordinary three-argument declaration is not sufficient to establish a safe, independently callable interface.

### Output

On an ordinary return, the result is a 64-bit bit pattern assembled from two 32-bit halves:

- **High half:** the value read from the return global, minus 84.
- **Low half:** a path-dependent 32-bit value shifted left by one.

The halves are concatenated. The low-half operation is not a 64-bit shift with carry into the high half.

No error code, success flag, or return-value interpretation is established.

### Memory and execution preconditions

For ordinary completion:

- Every dynamically selected call target must be executable under the applicable calling convention.
- Each byte, halfword, and word access actually reached must be valid.
- Dependency execution must return normally when subsequent behavior is expected.
- Computed addresses must satisfy the platform’s alignment and accessibility requirements.

The body contains no visible validation of these conditions. Invalid accesses or call targets have no specified recovery behavior.

## Behavior

### Arithmetic model and limitations

The representation uses 32-bit values for the principal calculations, 16-bit values for two stores, and an 8-bit memory read. The 64-bit dependency result and final return value are explicitly split into 32-bit halves.

Several displayed signed expressions can overflow or shift negative values. Consequently, their source-language appearance is not sufficient to define all machine-level behavior. Where examples below use truncation to 32 bits or sign-propagating right shift, those are conditional interpretations of the target operations, not claims that the displayed signed expressions are portable language semantics.

### 1. Beginning an iteration

Each iteration begins by adding 47 to the current third-argument value. Call this updated value the **iteration value**.

The function then:

1. Reads one byte at offset 1 from the current base.
2. Invokes the first dependency, using a target calculated from the current link-related state.
3. Supplies, in order, the current auxiliary value, the byte read, the iteration value, and the current second-argument value.
4. After that dependency returns, stores the iteration value as a 32-bit value at base offset `0x24`.

The byte read precedes the call. The word store follows it. The dependency can therefore observe or modify memory before the explicit word store occurs.

The base used for these operations is represented as retained state. The evidence does not establish an ABI guarantee that arbitrary dependencies preserve its underlying machine register.

### 2. Preparing the common dependency sequence

A second dependency is invoked with no explicit arguments shown.

A secondary 32-bit result associated with this stage is increased by 48. Call the resulting value the **adjusted value**.

The function derives:

- The base shifted left by one.
- A product formed by multiplying the adjusted value by 2, then by the adjusted value, then by 2 again.

Absent overflow, the product equals four times the square of the adjusted value. Under 32-bit truncating arithmetic, only its low 32 bits remain relevant.

The function next invokes three dependencies in order:

1. A dependency receiving base plus the incoming offset, the adjusted value, and the shifted base.
2. A dependency receiving a value read from the argument global.
3. A dependency with no explicit arguments shown.

Only after the third of these calls returns does the function read the global gate.

### 3. Zero-gate exit

If the gate is zero, the function does not perform the nonzero-gate stores or calls described below.

Instead, it forms the low-half source by shifting the secondary result associated with the gate-check stage right by 9. The displayed operation is signed.

It then enters the shared return sequence.

The gate is not read before the preceding calls. A dependency can therefore affect which branch is taken by changing the gate.

### 4. Nonzero-gate processing

If the gate is nonzero, the following operations occur in order.

First, the function stores the previously computed product as a 32-bit value at the address obtained by doubling the adjusted value and adding `0x34`.

This address is derived from an arithmetic value. The evidence does not establish that it belongs to a valid object, nor does the body perform a bounds check.

Next, a dependency is invoked with:

- The shifted base.
- A secondary 32-bit value represented as available at this stage.
- A pointer into the current stack frame.

The following dependency has no explicit arguments shown and produces a displayed 64-bit result. Its low and high 32-bit halves are used separately.

The function then:

1. Takes the low half of that result as the prospective low-half return source.
2. Adds 10 to the secondary value. Call this the **continuation value**.
3. Stores the product as a 32-bit value at secondary-value offset `0x3e`.
4. Tests whether the continuation value multiplied by 65,536 is negative.

The product store occurs before this test. Thus, the early exit described next is not a side-effect-free rejection.

### 5. Signed-test exit

If the signed test is true, control transfers directly to the shared return sequence.

The low-half return source is the low 32 bits of the preceding 64-bit dependency result. The later halfword stores, continuation calls, global XOR read, and loop-state update do not occur.

The supplied dependency map associates this shared destination with F-882dff43a182. The body shows a direct control transfer to the destination rather than a normal dependency call with a separately established argument-and-return boundary.

For a 32-bit truncating multiplication followed by a signed test, the condition examines whether bit 15 of the continuation value is set. It is not equivalent to testing whether the continuation value itself is negative.

### 6. Continuation path

If the signed test is false, the function continues in this order:

1. Stores the low 16 bits of the 64-bit dependency result at the address formed from its high 32 bits plus `0x3c`.
2. Invokes another dependency with no explicit arguments shown.
3. Invokes a further dependency, also with no explicit arguments shown, obtaining a 32-bit result used as an address.
4. Stores the 16-bit value `0x0026` at that address plus 2.

The low half of the 64-bit result is truncated for the first halfword store. Its high half supplies the address base; the result is not treated as one 64-bit pointer.

The next iteration value is then prepared by multiplying the continuation value by `0x10000000` and shifting the 32-bit result right by 9. The displayed right shift is signed.

A candidate next base is formed by adding that shifted value to the continuation value.

The auxiliary state and link-related state are replaced with fixed values. Their interpretation beyond their later uses is not established.

The XOR global is read before the final dependency call of the iteration. That call receives:

- The address-producing dependency result shifted left by one.
- The continuation value multiplied by 65,536, minus 81.

After the call:

- The next base becomes the candidate base XOR the previously read global value.
- The second-argument state becomes another secondary result-channel value.
- The new base is assigned to a stack-associated 32-bit slot displayed at displacement `0x130`.

The next iteration begins by adding 47 to the newly prepared third-argument state.

A mutation of the XOR global by the final dependency does not change the operand already read for the current XOR. It may affect a later iteration.

### 7. Shared return sequence

Both exit paths converge on the same return construction:

1. Read the return global.
2. Subtract 84 to obtain the high 32-bit half.
3. Shift the selected low-half source left by one.
4. Concatenate the halves and return.

The return global is read at exit, after all earlier calls and stores reached on that path. Its entry value is not cached for this purpose.

There is no visible iteration limit. Termination depends on the gate, the signed-test condition, dependency behavior, and valid execution.

## Side effects

### Explicit memory operations

The body visibly performs these accesses, in path order:

| Access | Width | Condition |
|---|---:|---|
| Read at base plus 1 | 8 bits | Every iteration |
| Write at base plus `0x24` | 32 bits | After the first call |
| Read argument global | 32-bit value as used | Common path |
| Read gate global | Storage width unresolved | After the common calls |
| Write product at twice adjusted value plus `0x34` | 32 bits | Nonzero gate |
| Write product at secondary value plus `0x3e` | 32 bits | Nonzero gate, before signed test |
| Write low result half at high result half plus `0x3c` | 16 bits | Continuation only |
| Write `0x0026` at returned address plus 2 | 16 bits | Continuation only |
| Read XOR global | 32-bit value as used | Continuation only |
| Assign new base to stack-associated slot | 32 bits | Continuation only |
| Read return global | 32-bit value as used | Either ordinary exit |

Endianness and exact alignment rules are not established. Potential aliasing is also unresolved. These locations must not be presumed distinct merely because they have different roles.

### Ownership and lifetime

No allocation or release operation is explicitly identifiable in this body. That does not establish that dependencies allocate nothing or release nothing.

The function passes computed addresses and a stack-frame pointer to dependencies. The evidence does not establish whether those dependencies retain pointers, access additional memory, mutate globals, or transfer ownership. No ownership guarantee should be inferred from an unresolved call.

The stack-associated assignment is visible in the supplied representation, but its exact correspondence to an externally observable machine store remains unresolved.

## Dependencies

The following opaque labels identify unresolved call sites for this contract. They do not establish that every label denotes a distinct implementation.

| Opaque ID | Established role |
|---|---|
| F-000000000001 | First call per iteration; target depends on link-related state; receives four explicit values. |
| F-000000000002 | Supplies the secondary result used to form the adjusted value. |
| F-000000000003 | Receives base-plus-offset, adjusted value, and shifted base. |
| F-000000000004 | Receives the argument-global value. |
| F-000000000005 | Last common-path call before the gate read; associated secondary state is used afterward. |
| F-000000000006 | Receives shifted base, secondary state, and a stack-frame pointer. |
| F-000000000007 | Produces the displayed 64-bit result. |
| F-000000000008 | Continuation-only call before obtaining an address-producing result. |
| F-000000000009 | Produces the address used for the `0x0026` halfword store. |
| F-00000000000a | Final continuation call; precedes replacement of loop state. |
| F-882dff43a182 | Mapped shared return destination, reached by direct transfer. |

“No explicit arguments shown” is not a guarantee of a zero-input ABI. Likewise, secondary result annotations do not independently establish multi-result calling conventions or preservation across intervening calls.

## Acceptance examples

These examples are conditional static expectations, not runtime results.

### Zero gate

Given valid calls and accesses, if the gate is zero when read:

- The byte read, first call, base-offset word store, and remaining common calls have already occurred.
- Neither nonzero-gate product store occurs.
- The low return half comes from the right-shifted secondary result, subsequently shifted left by one.
- The high return half comes from the return global read at exit.

If that secondary result is 1,024 and the return global is 100, the return halves are low 4 and high 16.

### Nonzero gate with early exit

Assume truncating 32-bit arithmetic. If the continuation value is 32,768, multiplication by 65,536 has bit pattern `0x80000000`, so the signed test takes the exit.

Both product stores on the nonzero-gate path have already occurred. Neither continuation-only halfword store occurs.

If the 64-bit dependency result has low half 3 and the return global is 84, the final return halves are low 6 and high 0.

### Width-sensitive predicate

Under the same arithmetic assumption:

- Continuation value 32,767 does not take the signed-test exit.
- Continuation value 32,768 does.
- Continuation value 65,536 does not: the multiplication truncates to zero.

These cases distinguish the observed width-sensitive test from an ordinary sign test on the continuation value.

### Halfword truncation

If continuation is reached and the 64-bit result has low half `0x12345678` and high half P, the halfword write at P plus `0x3c` stores `0x5678`.

The later halfword write stores `0x0026` at the address returned by F-000000000009 plus 2.

### Dependency-induced global changes

If F-000000000005 changes the gate before returning, the subsequent gate read observes that change.

If F-00000000000a changes the XOR global, the current iteration still uses the value read before that call.

## Uncertainties

The following remain **documented_only**:

- Complete entry ABI, including hidden incoming register requirements.
- Exact producers, lifetimes, and preservation rules for secondary result channels.
- Whether any unresolved call sites resolve to the same dependency.
- Dependency memory effects, ownership effects, exceptions, and nonreturning behavior.
- Exact widths and overlap relationships of global storage symbols.
- Target handling of signed overflow, signed left shift, and negative right shift.
- Alignment requirements, endianness, address-space mapping, and access-fault behavior.
- Exact stack-slot addressing and observability.
- Whether the shared return destination is independently callable.
- Higher-level purpose and interpretation of the return value.
- Conditions guaranteeing termination.

The warning about overlapping global symbols prevents treating the named global roles as proof of separate storage.

## Evidence

Established static evidence consists of the supplied body representation and dependency mapping: explicit access widths, relative access order, arithmetic constants, call-site argument expressions, branch placement, loop-state replacements, and two-half return construction.

Inferred intent is limited to descriptive roles such as “gate,” “continuation value,” and “return global.” Those labels describe observed use, not domain meaning.

The evidence is a recovered representation rather than verified machine semantics. Its signed operations, hidden register inputs, secondary outputs, and stack notation require the qualifications above.

No runtime tests have been performed.
