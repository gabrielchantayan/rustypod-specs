<!-- Trusted provenance: F-6fa0e85e50eb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6fa0e85e50eb

## Purpose

F-6fa0e85e50eb dispatches an input object through an optional preliminary dependency, one mandatory callback, and a conditional final callback.

Dispatch depends on fields in the input object and a callback table reached through the context. The function reads relevant fields again after dependencies return. Consequently, mutations made by an earlier dependency can affect subsequent callback selection and eligibility.

A nonzero dependency result terminates processing immediately and is returned unchanged. A zero result permits processing to continue; zero does not by itself mean that the dependency had no effects.

The apparent intent is staged handling of selected object categories. The meanings of the category values and callback roles are not established.

## Interface

The function receives two arguments:

- **Context:** an address supporting both a table-pointer read at byte offset 0 and a context-field read at byte offset 80.
- **Input:** an address supporting a discriminator read at byte offset 4 and, on particular paths, an address-valued read at byte offset 8.

It returns an integer result. Zero means that no invoked dependency returned a nonzero result. Any nonzero result is the exact value returned by the dependency that stopped processing.

### Widths and address model

The supplied representation uses integer accesses and a layout consistent with 32-bit integers and a 32-bit address model. Under that model:

| Location | Accessed value | Width |
|---|---|---|
| Context + 0 | Callback-table address | 32-bit address |
| Context + 80 | Value used in the final-stage comparison | 32-bit integer |
| Input + 4 | Discriminator | 32-bit integer |
| Input + 8 | Nested-object address or comparison value, depending on the discriminator | 32-bit word |
| Nested object + 4 | Nested discriminator | 32-bit integer |
| Callback table + 224 | Final callback address | Function address |
| Callback table + 268 | Special dispatch callback address | Function address |
| Callback table + 272 | General dispatch callback address | Function address |

Offsets are byte offsets, except that context offset 80 is derived from an index of 20 integer elements with four-byte integers.

The underlying ABI is not independently supplied. Exact integer width, pointer width, alignment requirements, and calling convention therefore remain documented assumptions rather than separately verified platform facts. The offsets explicitly expressed as byte offsets are directly supported by the supplied representation.

### Preconditions

For each reached operation, the corresponding address must be readable and any selected callback address must be callable with the original two arguments.

In particular:

- Every execution requires a readable input discriminator.
- Every execution that reaches ordinary dispatch requires a readable context table pointer and the selected table entry.
- A discriminator of 22 at the dispatch stage requires a readable input word at offset 8 and a readable nested discriminator through that address.
- A discriminator of 15 at the final stage requires readable words at input offset 8 and context offset 80.
- A successful final-stage comparison requires a readable and callable final callback entry.

These are path-dependent requirements. The function does not validate them. An early nonzero result can prevent later accesses and therefore avoid requiring otherwise-unused fields to be valid.

## Behavior

### Preliminary classification and optional invocation

The first relevant operation is a read of the input discriminator at byte offset 4.

Two discriminator values qualify for preliminary processing:

- 32549, or hexadecimal 7F25.
- 32548, or hexadecimal 7F24.

The first value is recognized directly. The second is recognized by comparing the result of subtracting 31744, hexadecimal 7C00, with 804, hexadecimal 324.

For a qualifying discriminator, the function invokes **F-a547ec0920d8**, passing the original context and input arguments.

- A nonzero result is returned immediately.
- A zero result allows dispatch to continue.
- Any mutations performed by this dependency remain relevant to subsequent reads.

For a nonqualifying discriminator, this dependency is skipped.

The arithmetic used in the second recognition path has an important boundary qualification. Both qualifying values and their arithmetic are representable in a signed 32-bit model. However, subtracting 31744 from sufficiently negative signed 32-bit values would underflow under language-level signed arithmetic. The supplied representation alone does not establish whether such inputs receive machine-level modular behavior or have another execution-model interpretation. No defined behavior for those exceptional arithmetic cases is asserted here.

### Mandatory dispatch selection

Unless the preliminary dependency has returned nonzero, the function reads the input discriminator again. It does not reuse the initial discriminator.

If this new value is 22, hexadecimal 16, it performs two additional reads in this order:

1. Read the nested-object address from input byte offset 8.
2. Read the nested discriminator at byte offset 4 of that object.

If the input discriminator is not 22, neither nested read occurs.

The special dispatch callback is selected only when both conditions hold:

- The newly read input discriminator is 22.
- The nested discriminator is 42, hexadecimal 2A.

Otherwise, the general dispatch callback is selected.

After making that selection, the function reads the table address from context byte offset 0, then obtains the selected callback address:

- Special selection: table byte offset 268, hexadecimal 10C.
- General selection: table byte offset 272, hexadecimal 110.

The selected callback is invoked with the original context and input arguments.

Exactly one of these two callbacks is invoked on every execution that reaches this stage.

- A nonzero result is returned immediately.
- A zero result permits the final-stage check.

The table address is obtained after the discriminator and any nested reads. If preliminary processing changed the context’s table address, that changed address is the one available to this dispatch stage.

### Conditional final invocation

After a zero result from mandatory dispatch, the function reads the input discriminator again.

If the discriminator is not 15, hexadecimal 0F, the final callback is skipped and the function returns zero. Neither the input word at offset 8 nor the context word at offset 80 is read for the final comparison on this path.

If the discriminator is 15, the function performs these reads in order:

1. Read the word at input byte offset 8.
2. Read the context word at byte offset 80.

The two values are compared for equality.

If they differ, the function returns zero without reading or invoking the final callback.

If they are equal, the function reads the table address from context byte offset 0, reads the callback address at table byte offset 224, hexadecimal E0, and invokes **F-C03** with the original arguments.

- A nonzero result is returned unchanged.
- A zero result leads to a zero return from F-6fa0e85e50eb.

The final-stage table lookup is separate from the mandatory-dispatch lookup. A callback that changes the context’s table address can therefore change which final callback is reached.

### Overall ordering and termination

The maximum invocation sequence is:

1. F-a547ec0920d8.
2. Exactly one of F-C01 or F-C02.
3. F-C03.

The first invocation is conditional on the initial discriminator. The last is conditional on discriminator and equality checks performed after mandatory dispatch.

There is no retry, fallback from one dispatch callback to the other, or result aggregation. Once any invoked dependency returns nonzero, no subsequent stage is executed.

If a dependency fails to return normally, the supplied representation establishes no local recovery mechanism.

## Side effects

No direct write to the context, input, nested object, or callback table is visible in F-6fa0e85e50eb.

Observable effects can nevertheless occur through every invoked dependency. The function passes the original addresses, not copies or snapshots.

The contract does not assume that any dependency:

- Preserves the input discriminator.
- Preserves the context’s table address.
- Preserves the context comparison field.
- Preserves global state.
- Leaves referenced memory allocated.
- Avoids allocation, release, I/O, synchronization, or other external effects.

Any memory needed by a later reached operation must remain valid after earlier dependencies return. This is a requirement for defined continuation, not a guarantee supplied by the dispatcher.

There is no explicit ownership transfer, allocation, or release in the dispatcher itself. Dependency ownership effects are unknown.

Aliasing between the context, input, nested object, and tables is not explicitly prohibited. However, readable layout and lifetime requirements still apply to every reached access.

The read order described above follows the supplied representation. It does not establish atomicity, volatile-access semantics, memory barriers, or behavior under unsynchronized concurrent mutation.

## Dependencies

The following identifiers describe dependencies without asserting semantic roles beyond the observed calls.

| Opaque ID | Resolution | Invocation condition |
|---|---|---|
| F-a547ec0920d8 | Direct dependency | Initial discriminator is 32549 or 32548, subject to the arithmetic qualification above |
| F-C01 | Callback from table offset 268 | Dispatch-stage discriminator is 22 and nested discriminator is 42 |
| F-C02 | Callback from table offset 272 | Every other mandatory-dispatch case |
| F-C03 | Callback from table offset 224 | Mandatory dispatch returns zero, the later discriminator is 15, and the compared words are equal |

F-C01, F-C02, and F-C03 are contract-local opaque identifiers for callback roles, not established identities of particular executable functions. Multiple slots may contain the same address. A slot’s address may also change between invocations.

All dependencies receive the original context and input arguments. Each produces an integer result interpreted only as zero or nonzero for continuation. There is no distinction between positive and negative nonzero results.

The internal behavior, ownership rules, and global-state effects of all dependencies are undocumented here.

## Acceptance examples

These examples are conditional acceptance criteria derived from static inspection. They are not reports of executed tests. Each assumes valid memory and normally returning dependencies unless stated otherwise.

### Preliminary termination

Given an initial discriminator of 32549, if F-a547ec0920d8 returns −7, the function returns −7. Neither mandatory dispatch callback nor the final callback is invoked.

The same requirement applies to an initial discriminator of 32548.

### Preliminary mutation affects dispatch

Given an initial discriminator of 32548, suppose F-a547ec0920d8 changes the discriminator to 22, installs a valid nested-object address whose discriminator is 42, and returns zero.

The function must select F-C01 using the post-dependency state. It must not select F-C02 based on the initial discriminator.

### Special dispatch

Given a dispatch-stage discriminator of 22 and a nested discriminator of 42, the callback at table offset 268 is selected.

If it returns 9, the function returns 9 without performing the final-stage discriminator read.

### Nested mismatch

Given a dispatch-stage discriminator of 22 and a readable nested discriminator of 41, the callback at table offset 272 is selected.

The nested reads are still required; the mismatch does not bypass them.

### No nested access for another discriminator

Given a dispatch-stage discriminator of 14, the general callback is selected without interpreting input offset 8 as a nested-object address.

If that callback returns zero and leaves the discriminator at 14, the function returns zero without the final-stage comparison.

### Dispatch mutation enables the final stage

Suppose mandatory dispatch changes the input discriminator to 15, makes the input word at offset 8 equal to the context word at offset 80, and returns zero.

The function invokes F-C03, even if the discriminator was not 15 before mandatory dispatch.

If F-C03 returns 6, the overall result is 6.

### Final comparison mismatch

Given a zero mandatory-dispatch result and a subsequent discriminator of 15, if the input comparison word is 100 and the context comparison word is 101, the function returns zero.

It does not read the final callback entry on this path.

### Updated table selection

Suppose mandatory dispatch returns zero after replacing the context’s table address. If the final-stage eligibility checks succeed, F-C03 is resolved from the replacement table.

The earlier table address must not be treated as a retained snapshot.

### Invalid nested address

Given a dispatch-stage discriminator of 22 and an invalid nested-object address, the function attempts the nested discriminator read before it can invoke either mandatory dispatch callback.

No returned error value or recovery behavior is established for that invalid access.

## Uncertainties

The following matters are **documented_only**:

- The actual ABI, including independently verified integer and pointer widths.
- The exact arithmetic semantics for initial discriminators that would cause signed subtraction underflow in a 32-bit language-level interpretation.
- The semantic meanings of discriminator values 32548, 32549, 22, 42, and 15.
- The meaning of the context word at offset 80.
- Whether nonzero results are errors, successful handling indicators, or another status convention.
- The identities and internal contracts of dynamically selected callbacks.
- Dependency allocation, release, ownership, and global-state behavior.
- Whether invalid addresses, null callback entries, dependency exceptions, or other abnormal exits are handled outside this function.
- Whether concurrent mutation is permitted by the surrounding system.

The names “preliminary,” “special,” “general,” and “final” describe control-flow position and selection only. They do not establish application-level semantics.

## Evidence

This contract is based exclusively on the supplied static representation of F-6fa0e85e50eb and the supplied opaque identity mapping for its direct dependency.

Established static evidence includes:

- The discriminator and callback-table offsets.
- The conditional nested-object reads.
- Repeated discriminator reads across dependency boundaries.
- Separate callback-table lookups for mandatory and final dispatch.
- Original-argument forwarding to every dependency.
- Immediate propagation of nonzero results.
- The absence of explicit dispatcher memory writes or ownership operations.

The 32-bit layout interpretation and staged-handler purpose are identified as assumptions or inferred intent where applicable. Dependency internals and runtime fault behavior are not established.

No runtime tests have been performed.
