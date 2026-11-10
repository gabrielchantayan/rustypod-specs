<!-- Trusted provenance: F-b437c6ec1cbc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b437c6ec1cbc

## Purpose

F-b437c6ec1cbc performs fixed-address writes and then enters a repeating sequence of external calls, pointer-derived accesses, overlapping stores, and conditional control transfers. No ordinary return is represented.

The established contract is the ordering and data dependence of those interactions. A device-specific purpose, allocation protocol, decoding operation, or memory-management role is not established.

This document describes static evidence, not verified execution. Where signed arithmetic, recovered types, or dependency behavior leave an outcome unresolved, that uncertainty remains part of the contract.

## Interface

### Explicit inputs

| Argument position | Represented width | Established consumption |
|---|---:|---|
| First | 32 bits | No explicit consumption shown. |
| Second | Signed 32 bits | Supplies an initial destination after multiplication by four. |
| Third | 32 bits | No explicit consumption shown. |
| Fourth | 32 bits | No explicit consumption shown. |
| Fifth | Signed 32 bits | Participates in an initial address calculation with an inherited pointer and an added eight. |

Unused explicit arguments are not proven irrelevant to the actual ABI. Dependencies may observe machine state beyond their represented argument lists.

Additional entry inputs are an inherited pointer, an inherited 32-bit word, and a global arithmetic operand used to derive an initial destination. Neither inherited value is initialized by this operation before use. The inherited pointer is eventually dereferenced for a 32-bit read.

### Results and termination

There is no represented return value, output object, or normal termination. Results consist of access attempts, dependency invocations, and state carried into subsequent iterations.

An inaccessible address, a trapping arithmetic operation, a nonreturning dependency, or the breakpoint may prevent later events. No handler, rollback, or alternative return contract is supplied.

### Widths and arithmetic

The representation distinguishes 8-, 16-, 32-, and 64-bit stores. Addresses and pointer conversions are represented through 32-bit quantities; dependency results used below are represented as 64-bit quantities with separately consumed low and high 32-bit portions.

Unsigned 32-bit addition, multiplication, and left shifts retain the low 32 bits. Narrow stores retain the destination width’s low bits. This does **not** establish wrapping semantics for signed arithmetic.

In particular, several address calculations involve signed multiplication or signed left shifts. Their outcomes outside the signed range are **documented_only** unknowns. Endianness, alignment requirements, and the actual processor’s address rules are also unresolved.

## Behavior

### Entry effects

The global arithmetic operand is read before the first store. Its product with 32 supplies the base for the second initial destination.

The operation first attempts a 16-bit store of `0xECA0` at four times the second argument, followed by another 16-bit store of `0xECA0` at the global-derived base plus six. Both address calculations depend on the represented signed-arithmetic semantics.

Four fixed-address, 32-bit stores follow in this order:

| Address | Value |
|---|---:|
| `0x00000030` | `0x00000000` |
| `0x20000040` | `0x20000000` |
| `0x00000028` | `0x00000040` |
| `0x0000002C` | `0x00000000` |

The initial accumulator is the inherited word masked by `0x0897ECA0`. The initial working word is `0x44BF64FE`.

A signed intermediate combines the inherited pointer’s represented integer value, the fifth argument, and eight. After that calculation, the inherited word is written to a global designated here as the **reset-associated global**. The intermediate is then stored as a 32-bit value at twice its own value interpreted as an address. Signed overflow in either the intermediate calculation or the doubling remains unresolved.

Next, `0x0897ECA0` is written to a second global, designated here as the **instruction-associated global**, and the byte `0xFE` is written at `0xA0000014`.

These global designations identify roles in the supplied material. They do not establish hardware semantics or exact alias relationships.

### Header behavior

The first header uses `0xA0000000` for both destinations, zero as its payload, and `0x40000000` as its increment.

Every header has four distinct stores. At its primary destination, an 8-bit zero precedes a 16-bit truncation of the working word, which precedes a 32-bit payload store. The accumulator is then increased by the current increment using unsigned 32-bit arithmetic, and the updated accumulator is written as 32 bits at the secondary destination.

The first three stores overlap by construction. All four overlap on the initial header. They must not be collapsed into a single final-value write: intermediate transactions can be externally observable.

A header begins an inner cycle with explicit call argument zero. Later returns to the header do not restore entry-time state.

### Common inner-cycle interactions

At the beginning of each inner cycle, the low byte of the current increment is retained. This snapshot precedes F-D01 and precedes the read that replaces the increment.

F-D01 is invoked with the current inner-cycle argument. Its computed target depends on carried dispatch state, described under Dependencies. The low 32-bit portion of its result supplies both the address of a 16-bit store and, after truncation, that store’s value.

A separate self-address store follows: the unsigned 32-bit product of the working word and 64 is written as a 32-bit value at the address represented by that product.

The operation then reads a 32-bit word through the current inherited-pointer-derived pointer. This becomes the current increment. At the working-word address plus four, a 16-bit store records the low 16 bits of the old pointer value plus four. A replacement pointer is subsequently read as 32 bits from the address given by F-D01’s high result portion plus `0x50`.

These accesses are ordered: the increment read uses the old pointer, whereas later pointer-dependent behavior uses the replacement. Earlier stores or dependency effects may change either read’s contents.

Before F-D02 is invoked, the carried inner-cycle argument is assigned `0x0897EDDF`. F-D02 and F-D03 nevertheless have no explicit arguments in the supplied representation. F-D02 precedes F-D03, which precedes F-D04. F-D04 receives the increment read through the old pointer.

F-D05 follows F-D04 and receives that increment logically shifted right by three. Its high 32-bit result is retained as a pointer for later record writes. Its low 32-bit result is interpreted as signed for multiplication by 32; the resulting represented 32-bit quantity supplies both the address and value of another self-address store.

That store precedes the unsigned comparison against `0x14`. A branch cannot protect the destination from the preceding access.

### Small-address continuation

If the derived quantity is less than `0x14`, the next inner cycle begins with argument zero. No header or extended-branch operations intervene.

The current pointer remains the replacement pointer obtained during the completed cycle. The working word and accumulator have not been changed by this branch. The increment remains the word read through the old pointer. Dispatch state is the state established before F-D05, not the entry-time state.

When multiplication by 32 has an ordinary representable result, or under an explicitly assumed low-word wrapping interpretation, a result below `0x14` can only be zero. A blanket wrapping interpretation is not established for the signed multiplication.

### Extended-branch interactions

For a derived quantity of at least `0x14`, a 32-bit store of the working word at that quantity plus `0x58` precedes a 16-bit store of the replacement pointer’s low bits at the working-word address plus `0x0E`.

The increment is then replaced by the working word shifted left by two using unsigned 32-bit arithmetic.

The next reset-associated write combines the previously saved byte as its low eight bits with a separately referenced three-byte reset component as its upper 24 bits. That component is consumed at this later point. It is not specified as an entry-time snapshot, and intervening dependencies may have changed it. Exact overlap with the earlier full-width reset write is unresolved.

F-D06 is invoked after this write, receiving the replacement pointer interpreted as unsigned and logically shifted right by two. F-D07 follows and supplies another represented 64-bit result.

F-D07’s low result portion is treated as a destination base; its high portion becomes the carried secondary header destination. The following stores are ordered:

| Order | Width | Destination | Value |
|---:|---:|---|---|
| First | 8 bits | Twice the low-result address | Saved byte from the start of this cycle |
| Second | 8 bits | Low-result address plus eight | Low eight bits of the high result portion |
| Third | 32 bits | Twice the current accumulator | High-result pointer retained from F-D05 |
| Fourth | 64 bits | Low-result address | Complete F-D07 result |

The second store lies immediately beyond the final eight-byte store’s region, absent wrapping or other aliasing. The first and third destinations may alias other destinations; no disjointness is established.

Next, the accumulator is derived from the F-D05 high-result pointer shifted left by two, and the working word is derived from the replacement pointer shifted left by eight. Both expressions involve a signed 32-bit interpretation before shifting. Their definedness is therefore conditional; unsigned wrapping must not be silently substituted.

The F-D05 high-result pointer also receives a three-word record:

| Offset | Width | Recorded value |
|---:|---:|---|
| `0` | 32 bits | Increment derived from the prior working word shifted left by two |
| `4` | 32 bits | New working word derived from the replacement pointer |
| `8` | 32 bits | Unshifted replacement pointer |

These writes occur in offset order, after the four F-D07-related stores. They establish a layout, not allocation or ownership.

The next header payload is the current increment multiplied by `0x100` using unsigned 32-bit arithmetic. The primary header destination is set to zero before the continuation decision.

### Unresolved continuation decision

The decision value depends on a signed interpretation of the replacement pointer, a left shift by 31, a signed right shift by 32, conversion to unsigned, and a logical right shift by 20.

A right shift by 32 does not have ordinary defined semantics for a 32-bit operand in the represented expression model. Consequently, the decision value and branch reachability are **documented_only** unknowns. Neither branch may be declared impossible solely from this expression.

Conditional on a zero decision value, execution returns to the header. Its primary destination is zero, its secondary destination is F-D07’s high result portion, and its accumulator, increment, working word, payload, and dispatch state are carried forward.

Conditional on a nonzero decision value, that value is written as 32 bits at the payload-derived address plus `0x20`. Dispatch state changes again, and the next inner-cycle argument is derived by interpreting the payload as signed and shifting it right by 29. F-D08 is then invoked with breakpoint immediate `4`.

If execution resumes, the working word is replaced by the signed interpretation of F-D07’s high result portion shifted right by 18. Execution continues with another common inner cycle, not a header. The signed right-shift behavior and breakpoint continuation remain unresolved.

## Side effects

Accesses may target low addresses, fixed high addresses, globals, inherited-pointer storage, or dependency-derived storage. No represented bounds check, null rejection, alignment check, accessibility check, or rollback precedes them.

A successful prefix of effects remains observable if a later operation fails. In particular, the small-address comparison follows its self-address store, and the zero-decision continuation attempts header writes through address zero.

Dependencies may modify globals, reachable memory, or external state between explicit accesses. This contract does not assume they preserve reset state, pointed-to contents, or any other global.

No allocation, deallocation, reference counting, or ownership transfer is established. The operation neither promises to create valid storage for its destinations nor establishes that any dependency acquires or releases that storage.

## Dependencies

These opaque IDs are documentation handles for call sites, not recovered public interfaces.

| ID | Explicit contract established here |
|---|---|
| **F-D01** | Computed call with one 32-bit argument and a represented 64-bit result. Target is the carried dispatch base plus `0xA1E`. |
| **F-D02** | Fixed computed call; no explicit arguments or consumed result shown. |
| **F-D03** | Fixed computed call after F-D02; no explicit arguments or consumed result shown. |
| **F-D04** | Fixed computed call receiving the 32-bit word read through the old pointer. No consumed result shown. |
| **F-D05** | Fixed direct call receiving that word logically shifted right by three; represented 64-bit result. |
| **F-D06** | Fixed computed call receiving the replacement pointer’s unsigned value logically shifted right by two. No consumed result shown. |
| **F-D07** | Fixed computed call; no explicit arguments; represented 64-bit result. |
| **F-D08** | Breakpoint operation with immediate `4`; continuation unspecified. |

For F-D01, the initial dispatch base is `0x08BFED9A`. The base established before F-D05 is `0x0897EDF1`; extended processing subsequently establishes `0x0897EE1D`; the breakpoint path establishes `0x08790D52`. Header entry does not reset it. These states select different computed targets on subsequent F-D01 invocations.

All dependencies have **documented_only** unknown ABI details, hidden inputs, preservation guarantees, failure behavior, and ownership effects. Explicit argument and result descriptions do not resolve those unknowns.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **First header transactions.** If entry processing reaches the first header, it attempts, at `0xA0000000`, an 8-bit zero store, a 16-bit store of `0x64FE`, a 32-bit zero store, and a 32-bit store of the masked inherited word increased by `0x40000000`, in that order.

2. **F-D01 result separation.** If its low and high result portions are `0x1000` and `0x2000`, respectively, the first result-dependent access is a 16-bit store of `0x1000` at `0x1000`. The later replacement-pointer read targets `0x2050`; it does not use `0x2000` directly as the replacement pointer.

3. **Zero derived destination.** If F-D05’s low portion is zero, the multiplication produces zero without signed overflow. A 32-bit zero-address store is attempted before the small-address branch. If it succeeds, the next inner cycle starts with argument zero and without a header.

4. **Representable extended case.** If F-D05’s low portion is one, the derived quantity is `0x20`. After its self-address store, extended processing first attempts a 32-bit working-word store at `0x78`.

5. **Saved-byte timing.** If a cycle starts with increment `0x123456AB` but the later pointer read produces `0x00000008`, extended processing uses `0xAB` for the saved-byte stores and reset composition. F-D04 receives eight, and F-D05 receives one.

6. **Reset-state mutation.** If an earlier dependency changes the three-byte reset component, its later value participates in reset composition. Reusing an assumed entry-time upper portion would not satisfy the represented read timing.

7. **Record layout.** If the prior working word is one, the replacement pointer is `0x100`, and the extended path reaches its record writes with the relevant shifts defined, offsets zero, four, and eight receive `4`, `0x10000`, and `0x100`.

8. **Conditional continuations.** A zero decision value requires the next header’s primary destination to be zero. A nonzero decision value requires its associated store before breakpoint `4`; continuation after that breakpoint is required only if execution resumes.

## Uncertainties

- **documented_only — execution model:** ISA, endianness, alignment, mapped memory, and fault handling are unavailable.
- **documented_only — signed arithmetic:** Several initial addresses, the F-D05-derived quantity, and later pointer-derived shifts lack established overflow semantics.
- **documented_only — decision expression:** The shift by 32 prevents a portable determination of the continuation branch.
- **documented_only — interface recovery:** The five arguments and inherited inputs may not capture the complete ABI.
- **documented_only — global aliases:** The supplied overlap warning does not establish exact reset-component boundaries or byte layout.
- **documented_only — dependencies:** Hidden inputs, memory effects, nonlocal exits, and ownership effects remain unspecified.
- **Inferred intent only:** Fixed-address transactions and a breakpoint suggest low-level machine interaction, but identify no particular subsystem.

## Evidence

Established claims derive only from explicit widths, constants, access ordering, data dependencies, and control transfers in the supplied static material.

Comments, symbolic names, and the prior draft are not independent proof. The warning about a removed unreachable region establishes neither that region’s contents nor the correctness of its removal.

No implementation access, runtime execution, smoke run, or tests were performed. Acceptance examples express conditional consequences of the static evidence, not observed outcomes.
