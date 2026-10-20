<!-- Trusted provenance: F-dcec9e84dbd3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dcec9e84dbd3

## Purpose

F-dcec9e84dbd3 repeatedly performs an ordered sequence of two indirect calls followed by software interrupt number 91. If execution continues beyond the interrupt, the function selects a fixed address base for the first call in subsequent iterations and repeats indefinitely.

The supplied representation contains no normal return path, termination condition, result processing, or explicit error handling. Its broader operational purpose is unknown. Describing it as a dispatcher, scheduler, shutdown routine, or exception loop would exceed the available evidence.

This contract distinguishes the visible control-flow sequence from the behavior of its unresolved dependencies. In particular, an indirect call or interrupt may prevent the next visible operation from being reached.

## Interface

### Entry and output

The apparent interface has:

- No explicit parameters.
- No explicit return value.
- An implicit incoming address-base value used to calculate the first indirect-call target on the first iteration.

The origin and validity of that incoming value are not established. It must not be treated as an ordinary caller-supplied parameter with a known calling convention.

No normal return is present in the visible control flow. This does not establish that every invocation runs forever: a dependency may fail, terminate execution, transfer control elsewhere, or never return.

### Argument values

The first indirect call receives three visible argument values, in this order:

| Position | Visible value | Interpretation supported by the evidence |
|---|---:|---|
| First | 0 | Zero-valued argument; semantic type unknown |
| Second | 0 | Zero-valued argument; semantic type unknown |
| Third | 0xFFFFFFA5 | Hexadecimal integer value; under a 32-bit representation, unsigned 4,294,967,205 or signed −91 |

The two zero values are not established as null pointers, lengths, flags, handles, or status values. The third value is not established as an error code or interrupt parameter.

The second indirect call has no explicit arguments in the supplied representation. This does not prove that its ABI-level behavior is independent of registers, stack contents, global state, or other ambient execution state.

Neither call’s return value is visibly inspected or used.

### Width and address assumptions

The supplied representation uses integer-valued address arithmetic and constants consistent with a 32-bit address space. It does not independently establish the target ABI, integer width, instruction-set state, pointer encoding, or indirect-call alignment requirements.

Accordingly:

- The 32-bit interpretation of 0xFFFFFFA5 must remain distinct from a proven signed argument type.
- Address calculations below describe the visible arithmetic, not a guarantee that the resulting address is executable.
- No alignment adjustment, address masking, or instruction-set-state conversion is shown.
- Overflow behavior for the first iteration’s unknown incoming base is not established.

## Behavior

### 1. Initial call-target selection

Before the first indirect call in each iteration, the function establishes a fixed integer base of 0x08579C95 for the second indirect call.

The first indirect-call target is calculated by adding 0xEB6, or 3,766, to the current implicit address base.

On the first iteration, that base comes from the incoming execution context. The function does not visibly initialize it before its first use.

On every later iteration reached through the visible loop, that base is 0x083D4CC2. The resulting first-call target is therefore:

**0x083D4CC2 + 0xEB6 = 0x083D5B78.**

This distinction between the initial and subsequent iterations is part of the contract. Replacing the first target with the later fixed target would not preserve the visible behavior unless the incoming base already had the same value.

### 2. First indirect call

The function invokes the calculated first target with the three argument values listed in the interface.

The visible sequence does not:

- Validate the calculated target.
- Check whether the target is executable.
- Test the argument values.
- Inspect a returned result.
- Branch on a success or failure indication.

The next operation is reached only if this call returns control in a way that permits the function to continue. No behavior is specified here for an invalid target, incompatible calling convention, exception, nonlocal transfer, or nonreturning callee.

### 3. Second indirect call

If the first call returns normally, the function invokes the target calculated from the previously established fixed base:

**0x08579C95 + 0xCB6 = 0x0857A94B.**

The arithmetic does not overflow a 32-bit unsigned address representation.

The second call has no explicit arguments, and its result is not visibly used. There is no visible condition between the first call and this call.

The target’s odd numeric address is retained as supplied. Whether that address encodes execution state, is directly callable, or requires architecture-specific interpretation is unknown. The contract must not silently round or otherwise normalize it.

### 4. Software interrupt

If the second call returns normally, the function issues software interrupt number **0x5B**, decimal **91**.

The evidence does not identify:

- The interrupt handler.
- The operating system or execution environment.
- Arguments consumed by the interrupt.
- Results produced by the interrupt.
- Whether the interrupt returns to the following operation.
- Whether it changes memory, registers, scheduling, privilege, or process state.

The visible operation is an interrupt request with that number, not a proven invocation of a particular named system service.

### 5. Base replacement and repetition

If execution continues after the interrupt, the function replaces the base used for the first indirect call with **0x083D4CC2**.

It then repeats the sequence without a visible condition.

The base replacement occurs **after** the interrupt, not before it. If the interrupt does not return to this continuation, the visible replacement and next iteration are not reached.

After a completed first iteration, all further iterations reached through this path use 0x083D5B78 as the first indirect-call target and 0x0857A94B as the second indirect-call target.

## Side effects

### Observable ordering

For an iteration that reaches every operation, the visible order is:

1. Establish the fixed base used for the second call.
2. Calculate and invoke the first indirect target with three arguments.
3. Calculate and invoke the second indirect target.
4. Issue software interrupt 91.
5. Replace the first call’s base for subsequent iterations.
6. Begin the next iteration.

The calls and interrupt must not be reordered relative to one another. Their unresolved side effects may make their order significant.

### Memory and ownership

No explicit data-memory load, data-memory store, allocation, deallocation, buffer access, or ownership transfer is shown outside the unresolved calls and interrupt.

This is a limitation of the visible function body, not a claim that the function is free of memory effects. The dependencies may read or write arbitrary state allowed by the actual execution environment.

No ownership contract is established for any argument. In particular, the zero arguments do not justify assuming either ownership transfer or the absence of resource-related behavior.

The representation does not establish whether its internal values occupy registers or memory. No externally observable storage layout can be specified for them.

### State and errors

The only explicit persistent change across visible iterations is replacement of the first call’s address base.

There is no visible global-state preservation mechanism, rollback, retry limit, diagnostic emission, exception handler, or error return. If a dependency returns an error-like value through its ordinary return channel, this function does not visibly inspect that value.

Indirect state changes remain possible and unspecified. The absence of explicit arguments to a dependency must not be interpreted as absence of effects.

## Dependencies

The following opaque labels identify unresolved dependency sites for this document only. They are not recovered function identities.

| Opaque ID | Invocation | Established behavior |
|---|---|---|
| F-unknown-01 | First indirect-call site | Target depends on the incoming base initially, then on a fixed base; receives three explicit arguments |
| F-unknown-02 | Second indirect-call site | Target arithmetic yields 0x0857A94B; no explicit arguments |
| F-unknown-03 | Software-interrupt service | Invoked through interrupt number 91; service semantics unknown |

F-unknown-01 may reach different functions on the initial and later iterations. These labels do not establish that the three dependencies are distinct implementations.

No dependency is known to preserve globals, preserve arbitrary machine state, return normally, or own or release memory.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not executed tests.

### First iteration with a different incoming base

Given an incoming base of 0x08000000 and a compatible execution environment, the first target calculation yields **0x08000EB6**.

If that call returns normally, the next call target is 0x0857A94B. If that call also returns, interrupt 91 follows. Only if execution continues beyond the interrupt is the next iteration reached with first-call target 0x083D5B78.

### Initial base already equal to the replacement base

Given an incoming base of 0x083D4CC2, both the first iteration and every subsequent reached iteration calculate 0x083D5B78 for the first call.

This equality is conditional on the incoming value; it is not a general property of entry.

### Ordinary error-like return

If the first dependency returns normally with a value interpreted by that dependency as an error, the visible function still proceeds to the second call. There is no result-based branch in this function.

The same principle applies to the second call: an ordinary returned value does not visibly suppress the interrupt.

### Nonreturning dependency

If the first call never returns, the second call and interrupt are not reached on that execution path.

If the second call never returns, the interrupt is not reached.

If the interrupt does not resume at the following operation, this function’s visible base replacement and next iteration are not reached.

### Repeated normal continuation

If both calls and the interrupt repeatedly return to the expected continuation, the function repeats indefinitely. No finite iteration count produces a normal return through the visible body.

## Uncertainties

The following are **documented_only**:

- Exact processor architecture, ABI, integer widths, and function-pointer representation.
- Origin and validity of the incoming first-call address base.
- Argument types and signedness at the first dependency’s actual interface.
- Behavior when the initial address addition exceeds the machine address range.
- Executability and instruction-set interpretation of either calculated target.
- Hidden inputs, outputs, clobbers, and side effects of all dependencies.
- Interrupt-service identity and continuation semantics.
- External termination, exception, and fault behavior.

An inferred intent is that the function repeatedly coordinates external operations. No more specific purpose is established.

## Evidence

The supplied static representation establishes an unconditional loop containing two ordered indirect calls, interrupt number 0x5B, and a post-interrupt replacement of the base used by the first call.

It also establishes the visible constants, argument order, absence of explicit result handling, and absence of a normal return path.

The fixed target addresses in this document are arithmetic derivations from those constants. They do not establish successful execution at those addresses.

No runtime tests, traces, dependency implementations, ABI documentation, or memory maps were provided. No runtime verification has been performed.
