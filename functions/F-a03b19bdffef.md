<!-- Trusted provenance: F-a03b19bdffef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a03b19bdffef

## Purpose

F-a03b19bdffef is a synchronous indirect-dispatch wrapper. It prepares a writable, invocation-local 32-bit value from its second input, then invokes a callable obtained through the object-like structure referenced by its first input.

The dispatched callable receives five arguments: the original first input, the address of the local value, a callable pointer obtained from the same dispatch slot, the original fourth input, and the original first input again.

The wrapper does not itself interpret the local value after dispatch and does not expose a return value. Its third input has no established effect.

This describes the supplied static evidence. An object-method or interface-adapter interpretation is plausible, but the object’s type, the operation’s meaning, and the callable’s behavior are not established.

## Interface

### Inputs

The supplied declaration has four explicit inputs:

| Position | Established representation | Use by this wrapper |
|---|---|---|
| First | Pointer to an integer-sized field | Used as the receiver-like value, dereferenced to locate dispatch information, and passed twice to the selected callable. |
| Second | Exactly four bytes, or 32 bits | Copied into invocation-local writable storage whose address is passed to the callable. |
| Third | Exactly four bytes, or 32 bits | Not referenced by the wrapper body. |
| Fourth | Exactly four bytes, or 32 bits | Forwarded to the callable as its fourth argument. |

The three four-byte inputs have no established semantic type. They may encode integers, handles, flags, or other values, but none of those interpretations is required by the visible behavior. No signedness-dependent operation is performed on the second, third, or fourth input.

The first input is not established as a particular class or structure type. The visible representation requires an accessible leading integer-sized field that participates in address formation. Its interpreted value supplies the base from which a callable slot is located.

### Output

The wrapper has no explicit return value.

Any result produced by the dispatched callable is ignored by the wrapper. A callable may communicate through memory or other side effects, but those effects are not specified by the wrapper’s visible body.

### Preconditions for normal completion

Normal completion requires all memory accesses and the indirect call to be valid under the target execution environment:

- The first input must permit the leading field to be read.
- The value read from that field must support access to the dispatch slot at byte displacement 28.
- The slot must contain a callable representation valid for the indirect invocation.
- The selected callable must accept the argument arrangement described below.
- The callable must return normally if normal return from the wrapper is expected.

There are no explicit checks enforcing these conditions. Null pointers, inaccessible storage, invalid callable representations, and incompatible calling conventions have no wrapper-defined recovery behavior.

## Behavior

### Local value preparation

Before dispatch, the wrapper initializes a local four-byte storage object with the second input.

The established operation is a value copy at the represented 32-bit width. The wrapper does not perform visible masking, arithmetic, byte-order conversion, or semantic validation. It passes the address of this copy, not an address into the caller’s original argument storage.

This distinction is observable to a callable that reads or modifies the pointed-to value. A modification changes the invocation-local copy. The wrapper contains no subsequent operation that copies the modified value back into caller-owned storage.

### Dispatch lookup

The wrapper obtains dispatch information by reading the leading field addressed by the first input. It uses that field’s value as an address base and accesses a callable slot at byte displacement 28, equivalent to hexadecimal 1C.

The displacement is an address offset, not a count of entries in the outer object. The evidence does not establish enough of the target ABI to make an unconditional claim about pointer width, slot stride, or a method index. Describing this as a particular numbered virtual method would therefore add assumptions beyond the visible byte offset.

The dispatch-slot expression occurs twice:

- Once to select the callable that will execute.
- Once to provide a callable pointer as the third argument of that invocation.

Each occurrence also involves reading the leading field through the first input. The wrapper therefore does not visibly cache a single resolved callable and reuse that cached value for both purposes.

For stable, ordinarily readable memory, both expressions identify the same slot and ordinarily produce the same callable value. That equality is conditional on the relevant memory remaining unchanged.

### Callable arguments

The indirect invocation supplies five arguments in this positional order:

| Position | Value supplied |
|---|---|
| First | The wrapper’s original first input. |
| Second | Address of the initialized local four-byte value. |
| Third | A callable pointer obtained through the same leading-field and byte-displacement lookup used for dispatch. |
| Fourth | The wrapper’s original fourth input. |
| Fifth | The wrapper’s original first input again. |

The first and fifth arguments are identical pointer values as supplied to the wrapper. They are not separate object copies.

The second argument designates writable local storage. Its initialized contents are the second input, subject to the normal interpretation of the supplied representation.

The third argument is a callable pointer value, not the callable’s return value and not the address of the dispatch slot itself.

The wrapper’s third input does not contribute to this argument list. In particular, it is not the third argument of the indirect invocation.

### Observable ordering

Initialization of the local four-byte value precedes the indirect-call expression. The local value is therefore initialized before the callable can access it through the supplied address.

The callable designator and all five argument values must be obtained before execution transfers to the selected callable. However, the supplied expression does not establish a single relative evaluation order among the callable lookup and the argument expressions.

Consequently, the contract does not require one particular dispatch-slot read to precede the other. It also does not guarantee a common snapshot if the leading field or callable slot can change during expression evaluation.

The source-level representation contains two lookup expressions. It does not, by itself, establish an exact count or ordering of physical bus transactions, processor loads, or accesses after optimization. Special behavior for volatile, device-backed, or concurrently modified storage is not specified.

### Completion

If the selected callable returns normally, the wrapper returns without further explicit processing.

In particular, the wrapper does not:

- Inspect the local value after the call.
- Branch on the callable’s result.
- Return the callable’s result.
- Perform an explicit cleanup operation.
- Invoke an additional visible dependency.

If the callable does not return normally, the wrapper provides no visible mechanism to convert that outcome into a successful return or a status value.

## Side effects

### Effects directly established for the wrapper

The wrapper writes the second input into invocation-local four-byte storage. It reads the leading field referenced by the first input and the dispatch slot derived from that field. It then makes one explicit indirect call.

No explicit write to caller-owned storage, object fields, dispatch entries, or globals appears in the wrapper body. No explicit allocation, deallocation, reference-count adjustment, logging operation, or error-reporting operation appears.

These statements describe direct wrapper operations only. They do not constrain the selected callable’s effects.

### Effects delegated to the callable

The selected callable receives the first input twice and an address to the local value. It may be able to modify reachable object state, the local value, globals, or external resources. The supplied evidence does not establish which such actions occur.

There is no basis for assuming that the callable preserves global state, leaves the dispatch structure unchanged, or preserves the object referenced by the first input. Likewise, passing a pointer does not establish whether the callable assumes ownership, releases an object, or retains a reference.

### Lifetime and ownership

The four-byte storage is local to this invocation and is not an explicitly allocated heap object.

Its address is available during the indirect invocation. No durable lifetime beyond the wrapper invocation is established. A callable that retains this address cannot rely on the wrapper to keep the storage valid after return.

The wrapper performs no explicit ownership transfer or release operation for the first input or for resources that might be encoded in the four-byte inputs. Any ownership convention implemented by the callable remains unknown.

## Dependencies

The only visible behavioral dependency is a dynamically selected callable reached through the first input’s dispatch information.

No separate opaque function ID is supplied for that callable. Its identity must remain unresolved rather than being assigned a guessed ID or descriptive implementation name.

The dependency contract established by the call site is limited to:

- Indirect selection from byte displacement 28.
- Five positional arguments with the representations described above.
- Access to a local four-byte value through the second argument.
- No use by the wrapper of any returned value.
- Normal continuation only if the callable returns normally.

The supplied evidence does not identify a concrete implementation, calling-convention annotation, accepted object type, result type, error model, or state-preservation guarantee for this dependency.

## Acceptance examples

The following are conditional acceptance scenarios derived from static evidence. They are not reports of executed tests.

### Stable dispatch and local-value observation

Given a valid first input, stable dispatch storage, a second input of 0x12345678, and a compatible callable that reads its second argument before modifying it:

- The callable observes the initialized 32-bit value 0x12345678 through that argument.
- Its first and fifth arguments equal the wrapper’s first input.
- Its fourth argument equals the wrapper’s fourth input.
- Its third argument equals the callable selected from the stable dispatch slot.

If that callable returns normally, the wrapper returns without exposing a result.

### Modification of the local value

Given a callable that overwrites the four-byte value addressed by its second argument and then returns normally:

- The modification applies to the invocation-local copy.
- The wrapper does not subsequently inspect or return that modified value.
- The wrapper performs no copy-back operation.

This does not imply that the callable cannot separately modify caller-owned state through another reachable address.

### Unused third input

Given otherwise identical valid invocations, differing only in the wrapper’s third input:

- The wrapper’s explicit local initialization and dispatch expressions are unchanged.
- None of the five explicit callable arguments is derived from that third input.

This is not an assertion about undocumented ABI-level inspection by an arbitrary callable.

### Full-width second input

Given a second input whose most significant bit is set, such as 0x80000000:

- The local object receives that 32-bit value without a visible range check or arithmetic transformation.
- The wrapper does not treat the high bit as an error condition.

No signed numeric meaning is assigned to that pattern.

### Invalid first input

Given a first input that cannot support the required leading-field read:

- There is no explicit validation branch.
- No wrapper-defined error code or fallback behavior is available.
- Normal completion is not guaranteed.

A particular trap, exception, or process outcome is environment-dependent.

### Changing dispatch storage

If the leading field or callable slot can change between the two lookup evaluations, the contract does not require the invoked callable to equal the callable pointer passed as its third argument. No synchronization or single-snapshot mechanism is visible.

## Uncertainties

The following matters are **documented_only**:

- Exact machine pointer width, integer-field width, alignment requirements, and address representation.
- Calling convention and detailed parameter-passing rules.
- The semantic type and required layout of the object-like input beyond the visible leading-field access.
- Whether the dispatch storage is immutable, synchronized, volatile, or externally mutable.
- The identity and implementation of the selected callable.
- The meanings of the second and fourth inputs.
- Why the third input is unused and why the first input is passed twice.
- Ownership, retention, deallocation, global mutation, and error behavior inside the callable.
- Behavior following invalid memory access, incompatible invocation, or abnormal callee exit.

The object-method interpretation is inferred intent, not an established interface guarantee. No runtime tests have been performed.

## Evidence

The supplied static body establishes a four-input wrapper, a local four-byte copy of the second input, an indirect call through byte displacement 28 from a base read through the first input, five explicit callable arguments, and an unconditional normal return after that call completes.

It also establishes that the third input is unused in the visible body, the dispatch lookup is expressed twice, and neither the callable’s result nor the potentially modified local value is subsequently consumed.

The evidence does not include the callable implementation, target ABI documentation, surrounding object definitions, caller behavior, or execution observations. Claims about those subjects remain conditional or documented_only rather than being treated as established behavior.
