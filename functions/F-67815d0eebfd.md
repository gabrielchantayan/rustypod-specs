<!-- Trusted provenance: F-67815d0eebfd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-67815d0eebfd

## Purpose

F-67815d0eebfd conditionally routes a request through one of two paths:

- If a value obtained from the supplied receiver equals the third argument, invoke a second indirect operation on that receiver.
- Otherwise, resolve the third argument through F-283eda65a14c and, if resolution produces a nonzero result, pass that result and the fifth argument to F-ca0f66276073.

The function produces no explicit return value. Its observable result consists of the selected calls and any effects those calls perform.

This routing behavior is established by the supplied static representation. Describing the third argument as a receiver identifier, or describing the fifth argument as an event or message, would be inferred intent rather than an established contract.

## Interface

The function takes five positional arguments.

| Position | Representation | Established use |
|---|---|---|
| First | Pointer to an object whose initial word supplies an indirect-dispatch base address | Receiver for both indirect operations; source of their dispatch information |
| Second | Opaque 32-bit value | Forwarded only on the equal-result path |
| Third | 32-bit integer | Compared with the first indirect operation’s result; passed to F-283eda65a14c on mismatch |
| Fourth | Opaque 32-bit value | Forwarded only on the equal-result path |
| Fifth | Opaque 32-bit value | Forwarded to whichever final operation is selected |

The represented return type is void. There is no explicit success indicator, error code, or returned object.

### Widths and interpretation

The supplied representation uses 32-bit integers and four-byte opaque values. The third argument, the first indirect operation’s result, and the result of F-283eda65a14c are represented as 32-bit integers.

The second, fourth, and fifth arguments have no established interpretation beyond their four-byte width. They might encode scalar values, addresses, handles, or other bit patterns. This function does not inspect or dereference them.

The receiver’s first word is used as an address-bearing dispatch base. This is consistent with a 32-bit object-dispatch layout. The representation does not independently establish every property of the underlying ABI, including the physical pointer width, calling convention, or exact declared types of the indirect targets.

The comparison between the initial indirect result and the third argument is equality, not ordering. Signedness therefore does not change which matching 32-bit bit patterns select the equal-result path. Likewise, the later test of the resolution result distinguishes zero from every nonzero value; it does not require a positive result.

### Receiver layout and validity requirements

The function uses two byte offsets from a dispatch base obtained through the receiver:

- Offset 0xE4, or 228 bytes: target of the initial indirect operation.
- Offset 0x68, or 104 bytes: target of the equal-result operation.

These are offsets into the referenced dispatch structure, not offsets directly into the receiver.

For an ordinary completed invocation, the receiver must support reading its initial address-bearing word, and the initial dispatch structure must support obtaining a callable target at offset 0xE4. If the equal-result path is selected, the receiver and the dispatch structure used at that later point must also support obtaining a callable target at offset 0x68.

The function performs no explicit validation of these conditions. Null receivers, invalid dispatch addresses, missing entries, incompatible calling conventions, and stale receiver storage have no defined recovery behavior in the supplied representation.

## Behavior

### Initial indirect operation

Every ordinary invocation begins by obtaining a dispatch base through the receiver’s initial word and obtaining an indirect call target at byte offset 0xE4 from that base.

That target is called with the receiver as its sole explicit argument. Its represented 32-bit result is compared with the third argument.

This initial call occurs before either routing path. It is not merely a value read: the target may have side effects. Nothing supplied establishes that it preserves the receiver, its dispatch information, global state, or other reachable memory.

### Equal-result path

If the initial indirect result equals the third argument, the function performs a second indirect operation through the receiver.

For this operation, dispatch information is obtained again through the receiver. The target is taken from byte offset 0x68 of the dispatch base used at that time. The operation receives five explicit arguments, in this positional order:

1. The original receiver pointer.
2. The original fifth argument.
3. The original second argument.
4. The original fourth argument.
5. A literal zero.

The fifth supplied argument thus precedes the second and fourth supplied arguments in the outgoing call. The third argument is not forwarded to this operation.

The source representation obtains the dispatch base separately for this call rather than retaining the base used for the initial call. Consequently, the contract does not require the two indirect operations to use the same dispatch structure. If the initial call changes the receiver’s dispatch base while leaving the receiver valid, the later operation uses the subsequently obtained base.

After the equal-result operation completes, F-67815d0eebfd returns without an explicit value. Neither F-283eda65a14c nor F-ca0f66276073 is called by the outer function on this path.

### Unequal-result path

If the initial indirect result differs from the third argument, the function calls F-283eda65a14c with the third argument as its sole explicit argument.

The result of F-283eda65a14c is tested against zero:

- A zero result ends the outer function without another represented call.
- A nonzero result causes F-ca0f66276073 to be called with that result as its first argument and the original fifth argument as its second argument.

After F-ca0f66276073 completes, the outer function returns without an explicit value.

Every nonzero represented result qualifies for forwarding, including negative 32-bit values. No additional range check, address validation, or type discrimination is shown.

The nonzero result is an opaque value at this boundary. Its use as an argument does not establish whether it denotes a pointer, identifier, reference, or another kind of value.

The second and fourth arguments are not explicitly forwarded on this path. The indirect operation at offset 0x68 is not performed by the outer function.

### Ordering and completion

The established outer-function ordering is:

- Initial dispatch lookup and indirect call.
- Comparison of the returned value with the third argument.
- Exactly one selected routing path.
- Return, provided all selected calls complete normally.

On the unequal-result path, resolution completes before the zero test and any call to F-ca0f66276073. On the equal-result path, the second indirect call occurs only after the initial call has returned and the equality condition has selected that path.

This is call-level and dispatch-dependency ordering. It does not establish a left-to-right evaluation order for otherwise independent outgoing arguments, atomic memory access, inter-thread synchronization, or a hardware memory-ordering guarantee.

No retry, fallback after a failed selected operation, or alternate destination is present.

## Side effects

### Direct effects

The outer function explicitly reads receiver-derived dispatch information and invokes the selected targets. It shows no explicit assignment to receiver fields, payload storage, or global variables.

The absence of explicit writes in the outer function does not make the function side-effect-free. Any invoked target may modify accessible state, perform I/O, allocate storage, release storage, or terminate execution; the supplied material does not establish those effects either way.

The outer function does not explicitly dereference the second, fourth, or fifth arguments. It also does not dereference the result of F-283eda65a14c before forwarding a nonzero value.

### Ownership and lifetime

No explicit allocation, deallocation, reference-count adjustment, ownership transfer, or retention operation appears in the outer function.

Passing the receiver or an opaque value to a dependency does not establish that the dependency borrows, retains, consumes, or releases it. Those ownership properties remain documented_only.

In particular, successful completion of the initial indirect operation is not evidence that the receiver remains valid. Validity is still required for the subsequent receiver-based dispatch when equality selects that path.

### Errors and exceptional behavior

There is no explicit error-reporting interface and no local handling of invalid memory, invalid call targets, exceptions, or dependency failure.

A zero resolution result has a specific established consequence: the function returns without calling F-ca0f66276073. It is not established that zero means “not found,” “invalid identifier,” or any particular error category.

If a selected dependency does not return normally, the later operations described for that path are not established to occur. The supplied representation contains no recovery mechanism that would force completion.

## Dependencies

| Dependency | Inputs at this boundary | Result used by F-67815d0eebfd |
|---|---|---|
| F-283eda65a14c | Third argument | Represented 32-bit result; zero ends the path, nonzero is forwarded |
| F-ca0f66276073 | Nonzero resolution result, then fifth argument | No result is exposed or used by the outer function |

Two additional dependency identities are unresolved:

- The indirect target obtained at dispatch offset 0xE4 receives the receiver and supplies the value used for routing.
- The indirect target obtained at dispatch offset 0x68 receives the receiver, fifth argument, second argument, fourth argument, and zero.

No opaque function IDs were supplied for those indirect targets. Their identities cannot be equated with either named dependency.

All call counts in this contract describe calls made directly by F-67815d0eebfd. They do not constrain calls performed internally by a dependency.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests. Unless stated otherwise, they assume valid storage, compatible call targets, and normally returning dependencies.

### Matching initial result

Given a third argument of 17, if the initial indirect operation returns 17:

- The outer function invokes the operation selected at offset 0x68.
- Its outgoing arguments are the receiver, fifth argument, second argument, fourth argument, and zero, in that order.
- The outer function does not call F-283eda65a14c or F-ca0f66276073.

The same branch rule applies when both compared values are zero or both have the same high-bit-set 32-bit representation.

### Mismatch with zero resolution

Given a third argument of 17, if the initial indirect operation returns 18 and F-283eda65a14c returns zero:

- F-283eda65a14c receives 17.
- F-ca0f66276073 is not called by the outer function.
- The offset-0x68 operation is not called by the outer function.
- The function returns without an explicit result.

### Mismatch with nonzero resolution

If the initial result differs from the third argument and F-283eda65a14c returns 0x12345678:

- F-ca0f66276073 receives 0x12345678 as its first argument.
- Its second argument is the original fifth argument.
- The original second and fourth arguments are not explicitly forwarded on this path.

A represented result of −1 also selects this call because it is nonzero. Whether F-ca0f66276073 accepts that value is outside this function’s established contract.

### Changed dispatch information

If the initial indirect operation changes the receiver’s dispatch base to another valid structure and returns a matching value, the equal-result operation is selected from offset 0x68 of the later-obtained structure, not necessarily the original structure.

If that mutation instead leaves the receiver or dispatch structure invalid, no successful-call or graceful-recovery guarantee follows.

### Opaque zero-valued forwarded arguments

A zero second, fourth, or fifth argument does not independently alter branch selection. On a path that forwards such an argument, its zero value is forwarded without an explicit check.

Whether the receiving dependency accepts it remains unknown.

## Uncertainties

The following remain documented_only:

- The semantic role of the receiver and each opaque argument.
- The identities and full signatures of the indirect targets.
- The complete object and dispatch-structure layouts.
- The underlying ABI beyond the widths represented in the supplied material.
- Dependency ownership, retention, release, and global-state behavior.
- Whether the initial indirect operation is normally observational or mutating.
- Whether a nonzero resolution result is a valid address or another kind of token.
- Dependency error semantics, exception behavior, and termination behavior.
- Thread safety, reentrancy, synchronization, and behavior under concurrent mutation.
- Any caller-level guarantees that make invalid receivers or unsupported argument values unreachable.

A plausible intent is to distinguish a receiver-local case from an externally resolved case. That interpretation is not needed to establish the routing contract and is not asserted as fact.

## Evidence

The established behavior comes solely from the supplied static function representation:

- Five positional inputs and no explicit return value.
- An initial receiver-derived indirect call at byte offset 0xE4.
- Equality comparison between that call’s result and the third argument.
- An equal-result indirect call at byte offset 0x68 with the specified reordered arguments and trailing zero.
- An unequal-result call to F-283eda65a14c.
- A zero/nonzero test controlling the call to F-ca0f66276073.
- No explicit allocation, release, state assignment, retry, or error handler in the outer function.

The contract distinguishes those observations from inferred intent and unresolved dependency behavior. No runtime tests have been performed.
