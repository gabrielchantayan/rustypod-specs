<!-- Trusted provenance: F-9ef1ccf2e050; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9ef1ccf2e050

## Purpose

F-9ef1ccf2e050 coordinates one of two operation sequences over an opaque endpoint:

- **Zero-mode sequence:** repeatedly obtain a 32-bit descriptor, classify it by its low 24 bits, and either continue, acquire and dispatch a payload, finish successfully on a particular marker, or invoke a continuation operation.
- **Nonzero-mode sequence:** perform up to three dependent operations in a fixed order, stopping at the first nonzero status.

Every path that returns normally invokes a common finalization dependency. An initialization dependency is invoked before either sequence begins.

The descriptor values and operation ordering are established by the supplied static evidence. Interpreting the sequences as a receive loop and a corresponding send operation is only inferred intent; no transport, protocol, or resource model is established.

## Interface

The function accepts four 32-bit arguments and returns a 32-bit status.

| Argument | Representation | Established role |
|---|---|---|
| Context | Opaque 32-bit value | Passed to the payload-dispatch dependency in zero mode and to the middle operation in nonzero mode. |
| Endpoint | Opaque 32-bit value | Passed to the dependencies that obtain descriptors, obtain payloads, continue processing, or perform the nonzero-mode sequence. |
| Mode | Signed 32-bit integer | Zero selects the repeated-processing sequence. Every nonzero value selects the other sequence. Also initializes the function’s local payload storage. |
| Initial descriptor | Unsigned 32-bit integer | Initializes the function’s local descriptor storage before any dependency call. Relevant to the descriptor-fetch sequence, although the fetch dependency may replace it. |

No interpretation of the Context or Endpoint values as pointers, handles, indexes, or owned resources is established. Their 32-bit widths do not establish what external objects, if any, they designate.

Two separate, addressable, 32-bit local storage objects are used:

- **Descriptor storage:** initialized from Initial descriptor and passed by address to F-4b3a084696b2.
- **Payload storage:** initialized from Mode and passed by address to F-7d397109ac66 when a payload descriptor is encountered.

These objects are reused across iterations. The function does not reset either object before subsequent calls that receive its address.

The return value is either:

- zero for the recognized terminal descriptor in zero mode; or
- the nonzero status that stops a dependency sequence; or
- the status of the final operation in nonzero mode.

Zero is the continuation/success discriminator at the call sites. The meanings of individual nonzero values are unknown. Negative and positive nonzero 32-bit statuses are treated identically for branching.

## Behavior

### Entry and initialization

Before invoking any dependency, the function initializes its local payload and descriptor storage from the corresponding arguments.

It then invokes F-bfade2068003 with the ordered numeric arguments **55** and **1**. Any return value from this invocation is ignored. Its completion does not create a checked success condition: if it returns normally, execution proceeds to the selected mode.

Mode selection depends on the original Mode argument being exactly zero or nonzero. Different nonzero Mode values do not select different operation sequences.

### Zero-mode descriptor processing

When Mode is zero, payload storage initially contains zero. Descriptor processing proceeds repeatedly, without a locally enforced iteration limit.

Each iteration begins by invoking F-4b3a084696b2 with:

1. Endpoint;
2. the address of descriptor storage.

The dependency can access the existing descriptor value through that address and can modify the storage. The function does not require or verify a complete overwrite.

If F-4b3a084696b2 returns nonzero, descriptor processing stops immediately. The resulting descriptor contents are not classified by this function. The nonzero status is retained for return after finalization.

If the dependency returns zero, the function classifies the current descriptor value as follows.

#### Descriptor layout

Classification uses two disjoint portions of the unsigned 32-bit descriptor:

| Portion | Mask | Use |
|---|---|---|
| Low 24 bits | `0x00FFFFFF` | Selects the descriptor category. |
| High 8 bits | `0xFF000000` | Examined only when the low 24 bits equal 1. |

These are numeric bit selections, not byte-array accesses. The function performs no byte-order conversion.

#### Low 24 bits equal 1

When the low 24 bits equal 1, the high 8 bits determine whether processing ends:

- If the high 8 bits equal `0xC0`, the complete descriptor is `0xC0000001`. The function invokes finalization and returns zero.
- For every other high-byte value, the function immediately begins another descriptor-fetch iteration.

A nonterminal descriptor in this category does **not** cause a payload fetch, payload dispatch, or continuation call.

The terminal comparison is exact across the two portions. A descriptor whose high byte is `0xC0` but whose low 24 bits are not 1 is not terminal.

#### Low 24 bits equal 50

When the low 24 bits equal decimal **50** (`0x32`), the high 8 bits do not affect classification.

The function invokes F-7d397109ac66 with:

1. Endpoint;
2. the address of payload storage.

If this dependency returns nonzero, processing stops immediately. The payload-dispatch dependency is not called, even if payload storage was modified before the failure status was returned. The nonzero status is retained for return after finalization.

If F-7d397109ac66 returns zero, the function invokes F-6134d252a6a2 with:

1. Context;
2. the current 32-bit payload value.

Any return value from F-6134d252a6a2 is ignored. After it returns normally, the function starts another descriptor-fetch iteration. It does not invoke the continuation dependency between this dispatch and the next fetch.

Payload storage persists between payload fetches. If a later successful fetch leaves all or part of that storage unchanged, this function dispatches the resulting value without validation or normalization.

#### Every other low-24-bit value

For every category other than 1 and 50, the function invokes F-fa92e3e1239e with Endpoint.

- A zero result begins another descriptor-fetch iteration.
- A nonzero result stops processing and is retained for return after finalization.

The descriptor value is not passed explicitly to this continuation dependency. The function performs no additional local interpretation of that descriptor.

### Nonzero-mode sequence

For every nonzero Mode value, the function executes this ordered, short-circuiting sequence:

1. Invoke F-56ffc99ff66d with Endpoint and numeric value **1**.
2. Only if the first operation returns zero, invoke F-7de8e89baf04 with Endpoint, numeric value **50**, and Context.
3. Only if the second operation also returns zero, invoke F-77aa48fed92a with Endpoint and numeric value **1**.

The result of the last operation actually invoked becomes the function’s return status after finalization.

Consequently:

- A nonzero first result prevents both later calls.
- A nonzero second result prevents the third call.
- If the third call occurs, its result is returned whether zero or nonzero.

This sequence performs no descriptor fetch, payload fetch, payload dispatch, or zero-mode continuation call. Initial descriptor does not influence its local control flow.

### Finalization and return

Every normal return is preceded by exactly one invocation of F-85722f62e20a with the ordered numeric arguments **55** and **1000**.

Finalization occurs both for the recognized terminal descriptor and for dependency-status exits. Any return value from finalization is ignored; it does not replace the selected return status.

There is no demonstrated recovery path for a dependency that does not return normally. The evidence establishes finalization on the function’s normal return paths, not on external termination, faults, or nonlocal transfers.

## Side effects

The function’s direct writes are the initialization of its two local 32-bit storage objects. It also exposes those objects by address to the descriptor-fetch and payload-fetch dependencies.

The observable dependency-call order is constrained:

- Initialization precedes every mode-specific call.
- Each successful descriptor fetch precedes its descriptor-specific action.
- Successful payload acquisition precedes payload dispatch.
- Each nonzero-mode operation must return zero before the next operation can begin.
- Finalization follows the status-producing operation or terminal-descriptor recognition and precedes return.

There is no explicit allocation, deallocation, global-state access, or ownership transfer in the function body. This does not establish that its dependencies lack those effects.

In particular:

- Context and Endpoint may designate resources whose state dependencies modify.
- Passing Context to a dependency does not establish whether that dependency borrows, retains, or releases a related object.
- Passing a local address does not establish that a dependency refrains from retaining it. Those local objects have no caller-visible lifetime after this invocation returns.
- Initialization, dispatch, continuation, and finalization must not be assumed to preserve global state.
- There is no explicit rollback of earlier effects when a later operation returns nonzero.

The zero-mode sequence can continue indefinitely if dependencies keep returning zero and no terminal descriptor is produced. The fixed finalization argument 1000 is not evidence of an overall loop timeout.

## Dependencies

| Opaque function ID | Established call contract within this function |
|---|---|
| F-bfade2068003 | Invoked once at entry with 55 and 1. Result ignored. |
| F-4b3a084696b2 | Receives Endpoint and writable descriptor storage. Zero permits descriptor classification; nonzero exits. |
| F-7d397109ac66 | Receives Endpoint and writable payload storage for category 50. Zero permits dispatch; nonzero exits. |
| F-6134d252a6a2 | Receives Context and the acquired payload value. Result ignored; normal completion resumes fetching. |
| F-fa92e3e1239e | Receives Endpoint for categories other than 1 and 50. Zero repeats processing; nonzero exits. |
| F-56ffc99ff66d | First nonzero-mode operation; receives Endpoint and 1. |
| F-7de8e89baf04 | Second nonzero-mode operation; receives Endpoint, 50, and Context. |
| F-77aa48fed92a | Third nonzero-mode operation; receives Endpoint and 1. |
| F-85722f62e20a | Invoked once before every normal return with 55 and 1000. Result ignored. |

These descriptions specify call-site behavior only. Dependency names are not evidence of their internal semantics.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Each assumes that invoked dependencies return normally and that any address-based storage access is valid.

1. **Immediate terminal descriptor.**  
   Mode is zero. The first descriptor fetch returns zero and leaves `0xC0000001` in descriptor storage. The observable sequence is initialization, descriptor fetch, finalization. The function returns zero. No payload or continuation dependency is called.

2. **Category 1 without termination.**  
   A successful fetch produces `0x80000001`. The function fetches another descriptor without invoking payload acquisition, dispatch, or continuation. If the next successful fetch produces `0xC0000001`, it then finalizes and returns zero.

3. **High byte alone does not terminate.**  
   A successful fetch produces `0xC0000002`. The function invokes F-fa92e3e1239e because the low 24 bits equal 2, not 1. If that call returns a nonzero status, finalization occurs and that status is returned.

4. **Payload category ignores the high byte.**  
   A successful fetch produces `0xAB000032`. Payload acquisition returns zero and supplies a chosen 32-bit value. Dispatch receives Context and that value. The next mode-specific call is another descriptor fetch.

5. **Payload acquisition fails.**  
   A category-50 descriptor is obtained successfully, but payload acquisition returns a nonzero status. Dispatch does not occur. Finalization occurs, and the acquisition status is returned.

6. **Descriptor fetch fails.**  
   Descriptor fetch returns a nonzero status, even if it also writes a terminal-looking descriptor. The function does not classify that descriptor. It finalizes and returns the fetch status.

7. **Continuation controls repetition.**  
   A successful fetch produces category 0. A zero continuation result causes another fetch. A nonzero continuation result instead causes finalization and return of that result.

8. **Nonzero-mode short-circuiting.**  
   Mode is −1. The first nonzero-mode operation returns zero; the second returns a nonzero status. The third is not called. Finalization occurs, and the second operation’s status is returned. Mode 1 selects the same sequence.

9. **All nonzero-mode operations succeed.**  
   All three operations return zero in order. Finalization follows the third operation, and the function returns zero. Initial descriptor has no effect on this sequence.

10. **Unchanged descriptor storage.**  
    Mode is zero and Initial descriptor is `0xC0000001`. If the first descriptor-fetch dependency returns zero without modifying descriptor storage, the function recognizes the initialized value as terminal. The function does not independently require a fresh descriptor write.

## Uncertainties

The following remain **documented_only**:

- The protocol or domain meanings of descriptor categories 1 and 50.
- Whether Context and Endpoint represent pointers, handles, or other opaque values.
- Whether descriptor and payload acquisition always overwrite all 32 bits on success.
- The meanings of individual nonzero statuses.
- The semantics and units of the fixed arguments 55, 1, 50, and 1000.
- Whether dependencies allocate, release, retain, or mutate resources or global state.
- Dependency behavior on invalid inputs, faults, nonlocal exits, or concurrent access.
- Reentrancy and thread-safety requirements.

A plausible intent is a marker-and-payload processing sequence paired with an alternate emission sequence. That interpretation is not required to explain the established behavior and must not add validation, retries, ownership rules, or recovery behavior.

## Evidence

This contract is based solely on static inspection of the supplied function representation and its supplied opaque dependency mapping.

Established evidence includes the four 32-bit inputs, two initialized local storage objects, exact masks and constants, mode split, ordered dependency invocations, zero/nonzero branching, ignored dependency results, and normal-return finalization paths.

No dependency implementations or external protocol definitions were supplied. No runtime tests have been performed. The acceptance examples describe consequences of the visible control flow under their stated conditions; they are not runtime verification claims.
