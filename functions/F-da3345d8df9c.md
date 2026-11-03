<!-- Trusted provenance: F-da3345d8df9c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da3345d8df9c

## Purpose

F-da3345d8df9c conditionally visits objects associated with a supplied base address. It either bypasses visitation when a gating condition and a masked-state comparison both permit that, or invokes an object operation repeatedly until an operation reports a nonzero result or visitation ends.

The observable return is a packed 64-bit value, not simply the result of the last object operation. Its lower 32 bits contain either the original base-address bits or a visitation result. Its upper 32 bits contain the final contents of a mutable local word that initially holds the supplied mask.

The broad interpretation of this function as conditional dispatch over a collection is inferred intent. The branch conditions, dependency-call order, callback arguments, and return packing are established by the supplied static representation.

## Interface

### Inputs

The function accepts four arguments in a representation using 32-bit integers and addresses:

| Argument | Representation | Established use |
|---|---|---|
| Base | 32-bit integer used as an address | Supplies a state word at byte offset `0x48` and an associated address at byte offset `0xA8`; also becomes the lower return word on the bypass path. |
| Mask | Unsigned 32-bit integer | Participates in the state comparison, initializes a mutable local word, and is passed unchanged as a value to object operations. |
| Expected value | Unsigned 32-bit integer | Is compared against the entire masked state word, initializes a second local word, and is passed unchanged as a value to object operations. |
| Initial object pointer | Pointer to 32-bit integer storage | Initializes a mutable object-pointer slot passed by address to the visitation dependency. |

These are descriptive argument names, not application-level types. The evidence does not establish that the base denotes a particular class, that the mask represents flags, or that the fourth argument denotes a valid initial collection member.

### Output

The output is 64 bits, constructed from two 32-bit words:

- **Upper word:** the final contents of the mutable word initially set to the mask.
- **Lower word:**
  - the original base bits when visitation is bypassed;
  - otherwise, the most recently returned object-operation result, or zero if no object operation returned nonzero.

The concatenation preserves the two words’ bit patterns. It does not sign-extend a negative object-operation result into the upper word.

No independent success/failure convention for the packed return is established. In particular, a nonzero lower word can be an address on one path and an operation result on another.

### Address and layout preconditions

The represented accesses require:

- A readable 32-bit state word at base plus `0x48` when the state comparison is reached.
- An address at base plus `0xA8` suitable for F-8ca21d3a8cca when visitation is selected.
- For each visited object, a readable first 32-bit word interpreted as a dispatch-table address.
- At dispatch-table byte offset `0xE8`, a readable callable entry compatible with the observed invocation.

The source representation uses 32-bit address-bearing values. It does not establish valid behavior for address-offset overflow, invalid addresses, alignment violations, or incompatible callable entries. The offsets are byte offsets, not element indices.

## Behavior

### Initial state

Before calling any dependency, the function establishes:

- An operation-result value of zero.
- A mutable 32-bit word initialized to the mask.
- A second mutable 32-bit word initialized to the expected value.
- A mutable pointer slot initialized to the initial object pointer.

The first mutable word later supplies the upper half of the return value. Its initial value must not be confused with an invariant value: visitation dependencies receive an address through which it can be modified.

The object-operation arguments, by contrast, use the original mask and expected-value arguments. Changes to the mutable visitation state do not replace those arguments.

### Gate and state comparison

F-2adbf0d1a289 is called without explicit arguments.

If its result is zero, visitation is selected immediately. The function does not perform its direct state-word read at base plus `0x48` on that branch.

If its result is nonzero, the function then reads the 32-bit state word at base plus `0x48`, computes its bitwise AND with the original mask, and compares the resulting full 32-bit word with the original expected value.

Visitation is selected if those values differ. Visitation is bypassed if they are equal.

Consequently:

- A nonzero gate result is necessary, but not sufficient, for bypass.
- The expected value is not itself masked before comparison.
- State bits outside the mask do not affect the comparison.
- Expected-value bits outside the mask prevent equality.
- Any effects of F-2adbf0d1a289 occur before the state read. The comparison therefore observes the state available after that call, not a pre-call snapshot.

On the bypass path, none of the initialization, visitation, object-operation, or finalization calls described below occurs. The lower return word is the original base value, and the upper word is the mask initialized before the gate call.

### Visitation initialization

When visitation is selected, F-8ca21d3a8cca is called with:

1. The address of the mutable word initially containing the mask.
2. The address obtained from base plus `0xA8`.

The second argument is an address into the base-associated storage. There is no explicit wrapper-level dereference of that location to obtain a different pointer before this call.

The initialization call’s return value is not used.

The supplied representation places the first mutable word, the second mutable word, and the object-pointer slot consecutively in a 32-bit stack layout. Interpreting them as a structured visitation record is plausible, but the dependency’s actual record format is not provided. In particular, the mask and expected-value copies are not proven to remain unchanged.

### Obtaining and invoking objects

After initialization, the function repeatedly calls F-b38a3aba0e18 with:

1. The address of the first mutable word.
2. The address of the mutable object-pointer slot.

The initial object pointer therefore seeds an in/out slot. It is not directly used as an object by this function before the first visitation call.

If F-b38a3aba0e18 returns zero, visitation ends without invoking an object operation for that iteration. The pointer slot is not dereferenced by the wrapper on this exit, even if the dependency modified it.

If F-b38a3aba0e18 returns nonzero, the wrapper uses the slot’s current pointer as the object:

1. It reads the object’s first 32-bit word as a dispatch-table address.
2. It obtains the callable entry at byte offset `0xE8` in that table.
3. It calls that entry with the current object pointer, the original mask, and the original expected value.

The returned 32-bit integer replaces the operation-result value.

A zero operation result causes another call to F-b38a3aba0e18. A nonzero result stops visitation immediately; there is no additional visitation call before finalization.

No object-pointer, dispatch-table, or callable-entry validation is visible between a nonzero visitation result and invocation. A dependency that reports an available object must provide storage suitable for these accesses.

The function imposes no iteration limit. Termination depends on visitation eventually returning zero or an object operation returning nonzero.

### Finalization and return

After visitation ends normally, F-d0d3f100effc is called once with the address of the first mutable word.

This finalization occurs in all normal visitation outcomes:

- No object is produced.
- One or more objects return zero and visitation later ends.
- An object operation returns nonzero.

It does not occur on the bypass path.

After finalization returns, the function packs the final first mutable word into the upper 32 bits and the operation-result value into the lower 32 bits. Finalization can therefore affect the upper return word if it writes that mutable word.

The operation-result value is zero when no object was invoked or when every invoked object returned zero. A nonzero result is preserved exactly as a 32-bit bit pattern.

There is no visible recovery path for a dependency or object operation that fails to return normally. Finalization is not established to run after a fault, exception, or other nonlocal exit.

## Side effects

### Direct effects

The wrapper directly:

- Reads the base-associated state word only when the gate returns nonzero.
- Initializes and updates local result and visitation storage.
- Reads object dispatch information after each nonzero visitation result.
- Invokes the selected object operation.
- Constructs the packed return value.

No explicit write to base-associated storage is shown. This does not imply that the storage remains unchanged: dependencies receive an interior address, object operations receive object pointers, and any called function may have other access paths.

### Dependency-mediated effects

All dependency and object-operation side effects remain part of the observable execution. They are not constrained by the wrapper beyond call order and explicit arguments.

In particular, the supplied evidence does not establish that any callee:

- Preserves globals or base-associated state.
- Avoids allocation or deallocation.
- Leaves visited objects valid after returning.
- Restricts writes to the first mutable word.
- Treats the pointer slot as output-only.
- Leaves dispatch information unchanged between visits.

### Ownership and lifetime

No explicit allocation, retain, release, or ownership transfer appears in the wrapper.

The name “finalization” describes the final dependency call’s position, not proven resource-release semantics. It is unknown whether F-d0d3f100effc releases resources, updates bookkeeping, or performs another operation.

Addresses of local storage are supplied synchronously to dependencies. Retaining and later using those addresses would require a lifetime arrangement not established here.

## Dependencies

| Opaque ID | Observed role | Established interaction |
|---|---|---|
| F-2adbf0d1a289 | Gate | Called once without explicit arguments; zero forces visitation, while nonzero permits the subsequent state comparison. |
| F-8ca21d3a8cca | Visitation initialization | Called only on the visitation path with the first mutable word’s address and base plus `0xA8`; return value ignored. |
| F-b38a3aba0e18 | Visitation advance or availability check | Receives the mutable-state address and object-pointer-slot address; zero ends visitation, nonzero causes invocation through the slot’s current pointer. |
| F-d0d3f100effc | Visitation finalization | Called once after normal visitation termination; receives the mutable-state address. |

There is also an indirect object-operation dependency selected through dispatch-table byte offset `0xE8`. No opaque function ID for that target is supplied. Its internal behavior is `documented_only`; the established contract is limited to target selection, argument values, result width, and the zero/nonzero continuation rule.

## Acceptance examples

These examples are conditional static acceptance criteria, not observed runtime results. Unless otherwise stated, every invoked callee returns normally and supplies valid storage.

### Gate forces visitation without a state read

Given a zero result from F-2adbf0d1a289:

- The wrapper must select visitation.
- It must not directly read the state word at base plus `0x48`.
- It must call initialization before the first visitation call.

This does not assert that the gate or another dependency never accesses that state.

### Matching masked state bypasses visitation

Given:

- A nonzero gate result.
- Mask `0x0000000F`.
- Expected value `0x00000005`.
- State word `0x000000A5`.

The masked state is `0x00000005`, so visitation is bypassed. The upper return word is `0x0000000F`; the lower word is the base’s original 32-bit representation. No visitation finalization occurs.

### Expected bits outside the mask prevent bypass

Given a nonzero gate result, mask `0x0000000F`, and expected value `0x00000015`, visitation is required regardless of the state word. The masked state cannot contain bit `0x00000010`.

### Zero mask

With mask zero:

- A nonzero gate result and expected value zero permit bypass for every readable state value.
- A nonzero gate result and nonzero expected value require visitation.
- A zero gate result requires visitation regardless of the expected value.

### No object is produced

If visitation is selected and its first availability result is zero:

- No object operation is invoked.
- Finalization is called.
- The lower return word is zero.
- The upper return word is whatever remains in the first mutable word after finalization.

### Continue on zero, stop on nonzero

If successive visited objects return zero, zero, and `0xFFFFFFF9`:

- All three object operations are invoked in order.
- Each receives the original mask and expected value.
- There is no fourth visitation call after the third operation.
- Finalization follows.
- The lower return word is `0xFFFFFFF9`, without sign extension into the upper word.

### Finalization changes the returned upper word

If the first mutable word contains one value before finalization and finalization writes `0x12345678` there, the returned upper word must be `0x12345678`. Returning the original mask instead would not satisfy the observed behavior.

## Uncertainties

The following remain `documented_only`:

- The application-level meaning of the gate and masked state.
- The exact visitation-record layout and dependency access extent.
- The interpretation of storage at base plus `0xA8`.
- Whether the initial object pointer meaningfully selects a starting position.
- The ownership and resource effects of every dependency.
- The identities and semantics of indirect object operations.
- Any global-state, synchronization, reentrancy, or thread-safety guarantees.
- Behavior after invalid memory access, address overflow, or nonlocal callee exit.
- Whether the packed upper word is an intentional public result or an artifact of the represented interface.
- The externally intended signedness and interpretation of the complete 64-bit return.

A collection traversal and a success-like nonzero stopping result are reasonable interpretations, not established semantic guarantees.

## Evidence

This contract is based solely on the supplied static function representation and its opaque dependency mapping.

Established evidence includes the gate’s short-circuit ordering, 32-bit masked comparison, dependency argument order, indirect dispatch offset, zero/nonzero loop decisions, normal-path finalization, and two-word return construction.

The 32-bit address model and consecutive local-storage layout follow the supplied representation; they have not been independently confirmed against an executable or ABI description.

No runtime tests have been performed. The acceptance examples describe conditional expected behavior rather than measured execution.
