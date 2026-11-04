<!-- Trusted provenance: F-aead6369d4c3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aead6369d4c3

## Purpose

F-aead6369d4c3 coordinates a primary operation and, under one specific condition, a resource-producing operation followed by a second primary operation.

Its distinguishing behavior is controlled by an optional pointer to a signed 32-bit word:

- An absent pointer selects a single primary operation.
- A pointer whose initial word is nonzero also selects a single primary operation, with that pointer passed through.
- A pointer whose initial word is zero selects a discovery-style primary operation without an output pointer. A strictly positive result then triggers resource acquisition and a follow-up primary operation.

“Discovery,” “resource acquisition,” and “follow-up” describe the apparent roles of the calls, not established dependency semantics. The visible contract establishes their arguments, sequencing, and result handling. It does not establish what resource is produced or what the primary operation does.

A successful acquisition causes the acquired word to be published to the caller after the follow-up call. The follow-up call’s return value does not affect either that publication or the wrapper’s returned result.

## Interface

### Inputs

The interface has two arguments:

1. **Opaque input word:** a 32-bit value. The wrapper does not interpret its contents. It places this value in a temporary context supplied to F-927fe5381082.
2. **Optional word pointer:** either null or a pointer to a signed 32-bit word.

For a nonnull word pointer, its target must be readable when the wrapper evaluates the initial branch condition. If the zero-initialized branch reaches successful acquisition, that same target must also be writable when the wrapper publishes the acquired word.

On the direct path, the pointer is passed to F-927fe5381082. Any stronger requirements imposed by that dependency are unknown.

### Return value

The return is a signed 32-bit integer.

The wrapper normally returns the result of its first call to F-927fe5381082. There is one explicit substitution:

- If that first result is strictly positive, the initial caller word was zero, and F-43da6b23bd44 returns zero, the wrapper returns **−1**.

The wrapper does not propagate the return value of the follow-up F-927fe5381082 call.

### Temporary context layout

The wrapper constructs a temporary context with the following visible 32-bit layout:

| Byte offset | Width | Initial contents |
|---|---:|---|
| 0 | 4 bytes | Opaque input word |
| 4 | 4 bytes | Original optional word pointer |
| 8 | 4 bytes | Word read from a shared source |

The context is supplied by address to F-927fe5381082. The second field retains the original pointer even when the separate output-pointer argument passed to that dependency is null.

The visible layout is consistent with a 32-bit pointer representation. Portability to a different pointer width is not established. Byte order and any larger dependency-defined context layout are not established.

The context is temporary storage belonging to this invocation. Its address does not establish permission for a dependency to retain or use it after the wrapper returns.

### Other temporary storage

When acquisition succeeds, the wrapper creates a separate signed 32-bit temporary initialized with the acquired word. The follow-up primary call receives a pointer to that temporary.

This temporary is distinct from the caller’s word. A dependency’s modification of the temporary is not itself published to the caller.

## Behavior

### 1. Capture shared values and prepare context

Before selecting a branch, the wrapper evaluates the shared word source for two purposes:

1. It saves a word for use as the third argument of a possible follow-up primary call.
2. It initializes the context’s final word from the same shared source.

These are separate source-level evaluations. Their equality is not an unconditional contract if the shared source can change.

The wrapper also initializes the context’s first two fields with the opaque input word and the original optional pointer.

The first primary call obtains its third argument from another evaluation of the shared source at the call site. It does not explicitly use either of the previously captured words for that argument.

Consequently, three values must be distinguished:

- The early saved word reserved for a possible follow-up call.
- The word initially stored in the context.
- The shared-source value supplied to the first primary call.

They may coincide in ordinary stable state, but the visible behavior does not require that they always coincide.

These statements describe source-level data dependencies and ordering. The supplied material does not establish machine-level load counts, atomicity, or synchronization.

### 2. Select the initial path

The wrapper first checks whether the optional pointer is null.

- If it is null, no initial dereference occurs.
- If it is nonnull, the wrapper reads the pointed-to signed 32-bit word and tests whether it equals zero.

The result of this initial test determines the branch. The wrapper does not repeat that test after a dependency call.

#### Absent pointer or initially nonzero word

The wrapper calls F-927fe5381082 with five arguments, in this order:

1. Address of the temporary context.
2. The original optional pointer, including null if absent.
3. The current shared-source word evaluated for this call.
4. The 32-bit all-ones value, **0xFFFFFFFF**.
5. Zero.

It returns the dependency’s signed 32-bit result without further calls and without an explicit caller-word assignment.

In particular:

- An initially negative caller word is nonzero and takes this path.
- A positive dependency result does not trigger acquisition on this path.
- A null pointer does not select the acquisition path, even if the first result is positive.

Any caller-word or shared-state changes performed by F-927fe5381082 remain dependency behavior rather than explicit wrapper writes.

#### Initially zero caller word

The wrapper calls F-927fe5381082 with the same five-argument structure, except that its second argument is null.

The context still contains the original nonnull caller pointer in its second field. Passing null as the separate second argument therefore does not establish that the dependency cannot discover or affect the caller’s word.

The first result is interpreted as a signed 32-bit integer:

- If it is zero or negative, the wrapper returns it immediately.
- If it is strictly positive, the wrapper proceeds to acquisition.

No unsigned interpretation is used for this positivity decision.

### 3. Acquire a word after a positive first result

Only on the initially-zero path, and only after a strictly positive first result, the wrapper calls F-43da6b23bd44 with:

1. The positive first result.
2. Zero.
3. Zero.

The returned word is tested for equality with zero.

#### Zero acquisition result

If F-43da6b23bd44 returns zero:

- The wrapper changes its eventual return value to −1.
- It does not make the follow-up primary call.
- It does not explicitly assign to the caller’s word.

This does not guarantee that the caller’s word is still zero. Earlier dependency activity may have changed reachable state.

#### Nonzero acquisition result

Any nonzero signed 32-bit result is accepted, including a negative value. There is no visible positivity, range, alignment, or address-validity check.

The wrapper:

1. Initializes a separate temporary word with the acquisition result.
2. Calls F-927fe5381082 again.
3. After that call returns normally, writes the original acquisition result to the caller’s word.
4. Returns the positive result from the first primary call.

The follow-up F-927fe5381082 arguments are:

1. Address of the same temporary context used for the first call.
2. Address of the separate temporary word initialized from the acquisition result.
3. The early saved shared word.
4. **0xFFFFFFFF**.
5. Zero.

The context is not visibly rebuilt between the two primary calls. If the first dependency call modified it, those modifications are presented to the follow-up call.

The follow-up call’s return value is discarded. If it modifies the separate temporary word, that modified word is also not the value subsequently assigned to the caller. Publication uses the acquisition result retained by the wrapper.

### 4. Arithmetic and boundary behavior

The wrapper performs no visible size arithmetic, multiplication, pointer arithmetic, or adjustment of a dependency result.

Its relevant integer decisions are:

- Caller word equals zero or does not equal zero.
- First primary result is strictly greater than zero.
- Acquisition result equals zero or does not equal zero.

The all-ones argument is established as a 32-bit bit pattern. Whether F-927fe5381082 interprets it as signed −1, an unsigned maximum, a sentinel, a length, or flags is unknown.

## Side effects

### Explicit wrapper effects

The wrapper initializes its temporary context and, conditionally, a separate temporary word.

Its only explicit write through the caller-provided pointer occurs after:

1. The pointer was initially nonnull.
2. Its initially observed word was zero.
3. The first primary result was strictly positive.
4. Acquisition returned a nonzero word.
5. The follow-up primary call returned normally.

That write stores the acquisition result, not the follow-up result and not the final contents of the temporary passed to the follow-up call.

The wrapper performs no visible explicit write to the shared source.

### Dependency effects

Dependency calls can have additional effects not described by the supplied implementation:

- F-927fe5381082 receives the temporary context by address.
- On the direct path, it also receives the original optional pointer.
- On the acquisition path’s first call, the context still contains the original pointer despite the separate pointer argument being null.
- On the follow-up call, it receives the temporary acquisition word by address.
- Either dependency may affect shared state through means not visible here.

The contract therefore does not promise that dependencies preserve the context, the caller’s word, shared values, or other reachable state.

### Ownership and lifetime

A nonzero acquisition result is treated as an opaque word. Its interpretation as an allocation address, handle, identifier, or another kind of token is unknown.

The wrapper contains no explicit release operation. It does not visibly reclaim the acquired resource if the follow-up call reports failure. Because that return value is ignored, such a reported failure does not prevent publication.

Neither dependency’s ownership rules are established. Publication to the caller is not, by itself, proof of an ownership transfer.

If a dependency does not return normally, later wrapper actions are not established to occur. In particular, publication follows the follow-up call; it does not precede it.

## Dependencies

### F-927fe5381082

The wrapper uses this dependency for both its first operation and its conditional follow-up.

Established call-facing properties:

- Five arguments are supplied.
- The first is the address of the temporary context.
- The second is either the original optional pointer, null, or the address of a separate temporary word.
- The third is a shared-source-derived 32-bit word.
- The fourth is the 32-bit all-ones pattern.
- The fifth is zero.
- The first call’s signed result controls the wrapper’s result and conditional acquisition.
- The follow-up call’s result is ignored.

The dependency’s purpose, memory-access bounds, error convention, ownership behavior, and global-state effects are **documented_only** unknowns.

### F-43da6b23bd44

This dependency is called only after a positive first primary result on the initially-zero path.

Established call-facing properties:

- Three arguments are supplied: the first primary result, zero, and zero.
- Zero return means the wrapper returns −1.
- Every nonzero return enables the follow-up call and later publication.

Its resource type, failure causes, ownership rules, and side effects are **documented_only** unknowns.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests. Unless otherwise stated, assume all dependencies return normally and do not independently modify the caller’s word.

1. **Absent optional pointer.**  
   Given a null pointer and a first primary result of 12, there is one primary call and no acquisition call. The wrapper returns 12.

2. **Initially nonzero caller word.**  
   Given an initial caller word of 7 and a first primary result of 24, the original pointer is passed as the primary call’s second argument. No acquisition occurs. The wrapper returns 24 and makes no explicit caller-word assignment.

3. **Initially negative caller word.**  
   Given an initial caller word of −3, the direct path is selected. The sign of that word has no additional significance.

4. **Zero caller word with zero first result.**  
   Given an initial caller word of zero and a first primary result of zero, the first call receives null as its separate second argument. No acquisition occurs. The wrapper returns zero.

5. **Zero caller word with negative first result.**  
   Given a first primary result of −8, the wrapper returns −8 without acquisition or an explicit caller-word assignment.

6. **Acquisition failure.**  
   Given an initial zero word, first primary result 64, and acquisition result zero, the wrapper returns −1. It makes no follow-up primary call and no explicit caller-word assignment.

7. **Successful acquisition and publication.**  
   Given an initial zero word, first primary result 64, and acquisition result 4096, the follow-up call receives a pointer to a temporary initially containing 4096. After that call returns, the caller’s word becomes 4096. The wrapper returns 64.

8. **Follow-up reports failure.**  
   Under the preceding conditions, suppose the follow-up call returns −9. The wrapper still publishes 4096 and returns 64. It does not propagate −9.

9. **Follow-up changes its pointed-to temporary.**  
   Suppose acquisition returns 4096 and the follow-up call changes its temporary target to 8192. The wrapper subsequently writes 4096, not 8192, to the caller’s word.

10. **Negative nonzero acquisition result.**  
    Given first primary result 64 and acquisition result −2, the nonzero acquisition test succeeds. Subject to normal dependency return, the caller’s word becomes −2 and the wrapper returns 64.

11. **Shared source changes during dependency execution.**  
    Suppose the early saved shared word is A and the first dependency later changes the shared source to B. The follow-up call still receives A as its third argument. Any context modifications made by the first call are not visibly reset.

## Uncertainties

The following remain **documented_only**:

- The meaning of the opaque input word.
- The identity, mutability, and synchronization properties of the shared source.
- The semantic meaning of the context fields to dependencies.
- Whether the first positive result denotes a size, count, identifier, or something else.
- Whether acquisition produces memory, another resource, or merely a numeric value.
- Whether dependencies retain supplied addresses or release resources.
- Dependency error conventions beyond the wrapper’s explicit comparisons.
- Behavior for invalid, misaligned, inaccessible, or concurrently modified caller storage.
- ABI details beyond the visible 32-bit words and context layout.
- Exception, fault, cancellation, or nonlocal-exit behavior.

The apparent two-stage resource workflow is inferred intent. The argument sequences, branch conditions, result handling, and explicit publication order are the established behavior.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the supplied opaque dependency mapping.

The visible control flow establishes one primary call on every normal path, one conditional acquisition call, and one conditional follow-up primary call. It also establishes that the follow-up result is ignored and that successful acquisition is published only after the follow-up returns.

No dependency implementations, runtime observations, tests, allocation traces, or external API contracts were provided. No runtime tests have been performed.
