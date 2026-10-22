<!-- Trusted provenance: F-4dae6fdf8946; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4dae6fdf8946

## Purpose

F-4dae6fdf8946 dispatches one operation through an object-associated function table, optionally publishes the operation’s 32-bit result, and returns a separate 32-bit status.

The wrapper reports a nonzero status only when both conditions hold:

- The dispatched operation returns zero.
- The third input is nonzero.

The nonzero wrapper status is the bit pattern `0xFFFFFFFD`. Every other normally completed case returns zero. In particular, a nonzero dependency result does not cause the wrapper to report failure, regardless of its sign.

The likely intent is to distinguish an unsuccessful nonempty operation from a permitted empty operation. That interpretation is inferred, not established: the source does not identify the third input as a length, identify the dependency result as a count, or establish what the operation does.

## Interface

### Inputs

The recovered interface has four arguments:

1. **Dispatch-holder address.** An address of storage whose first 32-bit word supplies an object address. The wrapper follows this address through an object-associated function table.
2. **Forwarded word.** A 32-bit value passed unchanged as the dependency’s second argument. Its semantic type is unknown; it might encode a scalar, address, or another representation.
3. **Conditioning integer.** A 32-bit integer passed unchanged as the dependency’s third argument. The wrapper also uses its zero-versus-nonzero value after the call.
4. **Optional result destination.** Either a null address or an address at which the wrapper stores the dependency’s 32-bit result.

The dependency receives four arguments, in this order:

- The object address obtained from the dispatch holder.
- The forwarded word.
- The conditioning integer.
- The fixed integer value two.

The constant two is an argument value, not a byte offset, result code, or additional wrapper-level dispatch selector.

### Outputs

The function returns a 32-bit status:

| Dependency result | Conditioning integer | Wrapper return |
|---|---|---|
| Zero | Zero | `0x00000000` |
| Zero | Nonzero | `0xFFFFFFFD` |
| Nonzero | Any value | `0x00000000` |

Under a signed 32-bit interpretation, `0xFFFFFFFD` represents negative three. The bit pattern is the primary contract; the source does not establish how callers interpret it.

When the optional destination is nonnull, it receives the dependency result rather than the wrapper return status. These two outputs serve distinct roles and can differ.

### Preconditions

For ordinary completion, the dispatch holder, object, function table, and selected table entry must support the reads and invocation described below. A nonnull result destination must support a 32-bit write after the dependency returns.

The wrapper performs no visible validation of those conditions. The source does not establish recoverable behavior for invalid addresses, inaccessible memory, an invalid target, incompatible calling conventions, or an unwritable destination.

## Behavior

### Dispatch structure and layout

The recovered representation uses 32-bit words and integer-sized address representations. Within that representation, the dispatch structure is:

- Offset zero in the dispatch holder contains an object address.
- Offset zero in that object contains a function-table address.
- Byte offset `0x10` in that table contains the callable target.

The table displacement is sixteen bytes. Under the recovered four-byte entry representation, this corresponds to the fifth entry when counted from zero. The established fact is the byte displacement; a larger table layout and the meanings of other entries are unknown.

The object address is also passed as the dependency’s first argument. The dependency does not receive the dispatch-holder address in its place.

### Observable sequence

For normal completion, the wrapper establishes the object and dispatch target, then makes one indirect call. There is no visible retry, alternate target, or second operation.

The dependency executes before the wrapper writes the optional result destination. Once the dependency returns normally, the wrapper retains its 32-bit result and:

- Writes that result to the optional destination if the destination is nonnull.
- Selects the wrapper status using the dependency result and the conditioning integer.
- Returns the selected status.

Thus a nonnull destination is written even when the wrapper subsequently returns `0xFFFFFFFD`. A zero conditioning integer does not suppress dispatch or suppress the optional result write.

The source establishes the pointer-following relationships needed to obtain the target. It does not establish an instruction-level ordering or exact count of repeated reads while evaluating the dispatch expression. No contract should depend on such details, particularly if the dispatch structures can change concurrently.

### Result classification

The wrapper tests the dependency result for equality with zero. It does not classify negative results separately, compare magnitudes, or verify agreement between the result and the conditioning integer.

Likewise, the conditioning integer is tested only for zero versus nonzero. A negative value satisfies the nonzero condition just as a positive value does.

Consequences include:

- A negative dependency result leads to wrapper status zero.
- A zero dependency result with a negative conditioning integer leads to `0xFFFFFFFD`.
- A zero dependency result with a zero conditioning integer leads to status zero.
- A nonzero dependency result with a zero conditioning integer also leads to status zero.

There is no wrapper-level addition, subtraction, multiplication, size conversion, or range calculation. The meaningful width-sensitive operations are transporting 32-bit values, following the recovered address representation, accessing the table at a sixteen-byte displacement, and storing a 32-bit result.

## Side effects

The wrapper’s explicit data write is the optional 32-bit result store. If the destination is null, that store is omitted; the dependency is still called.

The indirect call may have additional effects. The source provides no basis for assuming that the dependency preserves globals, object contents, dispatch structures, external resources, or memory reachable through its arguments.

No explicit allocation, deallocation, ownership transfer, reference-count update, or resource-release operation appears in the wrapper. This absence does not establish that the dependency performs none of those actions.

Aliasing is not explicitly rejected. If the destination overlaps readable dispatch storage or object storage, the wrapper’s own store occurs after dispatch. Such overlap may therefore modify storage used earlier in the invocation. Whether the overall call remains valid also depends on dependency behavior and the validity of the destination when the store occurs.

The wrapper does not explicitly reread the dispatch structure after the dependency returns. Nevertheless, this does not imply that the structure remains unchanged or valid.

No visible error handler translates a failed read, failed indirect invocation, failed output write, or abnormal dependency termination into either documented return status. The ordinary return contract applies only when execution reaches the return normally.

## Dependencies

**F-indirect-001** is a specification-local opaque identifier for the unresolved indirect dependency. It is not a claim that a globally identified implementation has been recovered.

Established invocation contract:

- Target: the callable entry at byte offset `0x10` in the object-associated table.
- Argument one: the object address.
- Argument two: the forwarded 32-bit word.
- Argument three: the conditioning 32-bit integer.
- Argument four: the fixed value two.
- Consumed result: a 32-bit value classified as zero or nonzero and optionally stored unchanged.

The dependency’s semantic operation, parameter constraints, memory ownership rules, side effects, result meanings, and abnormal-completion behavior are unknown. The wrapper’s result handling does not justify assigning this dependency conventional success, error, count, or boolean semantics.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Each assumes a valid dispatch chain, a compatible dependency, and normal completion.

1. **Zero result with a positive conditioning integer.**  
   With conditioning integer seven and dependency result zero, the wrapper returns `0xFFFFFFFD`. A nonnull destination receives zero before that return.

2. **Zero result with a zero conditioning integer.**  
   The dependency is still invoked with third argument zero and fourth argument two. If it returns zero, the wrapper returns zero. A nonnull destination receives zero.

3. **Nonzero result with a positive conditioning integer.**  
   With conditioning integer seven and dependency result three, the wrapper returns zero and, if requested, stores three. It does not require the result to equal seven.

4. **Negative dependency result.**  
   If the dependency returns `0xFFFFFFFF`, the wrapper returns zero for every conditioning-integer value. A nonnull destination receives `0xFFFFFFFF`; the wrapper does not translate it into its own nonzero status.

5. **Negative conditioning integer.**  
   With conditioning integer negative one and dependency result zero, the wrapper returns `0xFFFFFFFD`. No wrapper-level nonnegative-range check precedes dispatch.

6. **Null destination.**  
   A null destination omits only the output store. With a nonzero conditioning integer and a zero dependency result, the wrapper still returns `0xFFFFFFFD`.

7. **Dependency does not return normally.**  
   If the dependency never returns normally, the source does not establish a completed result store or wrapper return. Neither normal return value may be assumed.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings and valid ranges of the forwarded word and conditioning integer.
- Whether either forwarded input denotes memory, a size, an offset, or a mode.
- Meaning of the fixed dependency argument two.
- Ownership, lifetime, and alignment requirements beyond the accesses shown.
- Dependency mutation, allocation, release, global-state effects, and reentrancy.
- Concurrent mutation rules for the dispatch holder, object, table, and output destination.
- Fault, exception, and other abnormal-control-flow behavior.
- Whether callers interpret the wrapper status as signed or unsigned.
- Portability beyond the recovered 32-bit representation.

Intent remains separate from these unknowns: the conditional status resembles rejection of a zero result for a nonzero request, but that resemblance does not establish a particular operation or error category.

## Evidence

Established static evidence consists of the supplied function body: one indirect dispatch through a table entry at byte offset `0x10`, four dependency arguments including a fixed final value of two, an optional post-call 32-bit result store, and a return decision based solely on two zero comparisons.

The supplied body contains no explicit input validation, retry, allocation, release, or global-variable access. Effects inside F-indirect-001 remain unresolved.

No implementation access or runtime tests were performed. The acceptance examples express consequences of the supplied static evidence, not observed execution results.
