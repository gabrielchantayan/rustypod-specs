<!-- Trusted provenance: F-b020d0b6ae5e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b020d0b6ae5e

## Purpose

F-b020d0b6ae5e conditionally processes a 32-bit request against a receiver and records the request in receiver storage.

The supplied static representation establishes two paths:

- **Record-only path:** a nonzero receiver flag causes an immediate request write and return.
- **Active path:** a zero flag permits an option write, request lookup, receiver-related dependency calls, temporary initialization, conditional processing, a request write, and temporary finalization.

These labels describe control flow, not application semantics. The evidence does not identify the request as a resource, event, selection, or other domain object. Likewise, “initialization,” “processing,” and “finalization” describe call placement and use; they do not establish ownership or resource-management behavior.

A conditional receiver-relative access uses a displacement read from separate global storage. That displacement is an input obtained during execution, not an established immutable layout constant.

## Interface

The function accepts five arguments and has no explicit return value.

| Position | Contract name | Established representation | Use |
|---|---|---|---|
| 1 | Receiver | Address represented by a 32-bit integer | Base for direct accesses and a subobject address |
| 2 | Request | Opaque 32-bit value | Lookup input and request-field value |
| 3 | Processing argument | 32-bit integer | Zero test and unchanged forwarding to optional processing |
| 4 | Suppression argument | 32-bit integer | Zero test controlling selection of a 16-bit processing input |
| 5 | Option | Opaque 8-bit value | Active-path receiver-field write |

There is no explicit success indicator or returned object. Observable behavior consists of direct memory accesses, dependency calls, effects of those dependencies, and whether execution returns normally.

### Receiver storage

The following fixed offsets are measured in bytes from the receiver address:

| Offset | Access width | Established use |
|---|---:|---|
| `0xD4` | Address formation only | Subobject address passed to lookup |
| `0x104` | 32 bits | Request destination |
| `0x108` | 8 bits | Option destination |
| `0x10C` | 8 bits | Initial path-selection flag |

Forming the address at `0xD4` is not itself a direct read of that subobject. Any access through the lookup dependency is governed by that dependency.

An additional 16-bit read uses an address formed by adding a displacement obtained from separate global storage to the receiver address. Neither a fixed numeric displacement nor the complete receiver layout is supplied.

### Separate global displacement storage

On one conditional path, the function directly reads a separate global data location to obtain the displacement for the receiver-relative 16-bit access.

The supplied representation does not explicitly declare that global location’s storage width or signedness. It establishes its use as an operand in the receiver-relative address calculation, but not enough information to specify its independent binary layout or extension rules.

The displacement is obtained after the additional accessor call on the optional-processing path. Dependencies executed before that read may affect its value.

### External storage and temporary storage

Temporary initialization uses two external reads through separately obtained addresses:

- An 8-bit read at offset `0x2C` from the first accessor result.
- A 32-bit read at offset `0x28` from the second accessor result.

Those accessor results need not identify the same object.

The supplied representation provides 92 bytes of invocation-local temporary storage. The temporary’s logical type, alignment requirements, initialized extent, and internal layout are not established.

### Preconditions

Normal defined execution requires valid storage for each reached direct access and satisfaction of each invoked dependency’s contract.

In particular:

- The receiver flag must be readable on every invocation.
- The request destination must be writable whenever the request write is reached.
- The option destination must be writable on the active path.
- The first two accessor results must support their respective external reads.
- On the conditional displacement path, the separate global location must be readable, the resulting address calculation must be valid, and the resulting address must support a 16-bit read.
- Objects and handles passed to dependencies must satisfy those dependencies’ validity and lifetime requirements.

No explicit receiver-null check, accessor-result-null check, address-range check, or alignment check is shown. Lookup returning zero has separate explicit handling.

## Behavior

### Path selection and record-only behavior

The initial direct access is an 8-bit read at receiver offset `0x10C`. Only zero versus nonzero matters.

For a nonzero flag, the function writes the original 32-bit request at receiver offset `0x104` and returns. It does not directly write the option, invoke a listed dependency, initialize the temporary, read the global displacement, or perform the conditional 16-bit read.

For a zero flag, execution takes the active path. The function does not directly reread the flag later. A dependency changing that flag therefore does not cause an explicit local branch reevaluation.

### Active-path lookup and failure boundary

The active path writes the option byte at receiver offset `0x108` before invoking F-1f46730b1ced.

F-1f46730b1ced receives the address at receiver offset `0xD4` and the original request. Its result is represented as a 32-bit value and tested for zero.

A zero result causes a call to F-a817d8d58c10 with no explicit arguments. The supplied representation identifies that dependency as non-returning. Under that annotation, this path does not reach receiver-update calls, temporary initialization, optional processing, or the final request write.

The failure mechanism is not established. In particular, the evidence does not distinguish termination, throwing, transfer of control, or another non-returning action.

The option write has already occurred when lookup or failure handling runs. The function has not yet directly written the request field. These statements concern this function’s direct actions; they do not exclude dependency writes to either field.

### Receiver-related dependency sequence

For a nonzero lookup result, the following dependency order is established:

1. F-d2f99870cc43 receives the lookup result and supplies a 32-bit value.
2. F-bf557f07179f receives the receiver and that supplied value.
3. F-fa8898fa75ae receives the original lookup result.

The original lookup result is retained for possible later use in optional processing.

No ownership conclusion follows from F-fa8898fa75ae’s position in this sequence. Its behavior could affect the result’s state or lifetime, but neither release nor preservation is established. The validity of the later use depends on the actual dependency contracts.

### Temporary initialization

The initialization inputs are captured in a fixed observable order: F-fdd8589eca2d is called, the byte at offset `0x2C` from its result is read, F-fdd8589eca2d is called again, and the 32-bit field at offset `0x28` from that second result is read.

F-2c5a9f9c1eb3 then receives:

| Argument position | Value |
|---|---|
| 1 | Temporary-storage address |
| 2 | Captured 32-bit field |
| 3 | Previously captured byte |

The first byte read precedes the second accessor call. A second accessor call that changes the first object does not cause an explicit reread of that byte. Conversely, the 32-bit field is read only after the second accessor has returned.

The function does not combine these accesses into one accessor result. Dependencies may return different objects or change shared state between the reads.

### Optional-processing selection

After temporary initialization, F-5c5958cfd5fe is called. Its result is compared with the integer value `8`.

If the result equals `8`, optional processing is skipped. There is no additional accessor call, global displacement read, or conditional 16-bit read on this branch. The final request write and temporary finalization are still reached under normal control flow.

For any other result, the function calls F-fdd8589eca2d once more and prepares the five arguments for F-818c773e8c8a.

The suppression argument determines how the final 16-bit argument is selected:

| Suppression argument | Processing argument | Final 16-bit argument | Global displacement read | Receiver-relative 16-bit read |
|---|---|---|---|---|
| Nonzero | Any | Zero | No | No |
| Zero | Zero | Zero | No | No |
| Zero | Nonzero | Value read at the dynamically formed address | Yes | Yes |

For zero suppression, the additional accessor call occurs before the processing argument’s zero test. For nonzero suppression, that accessor call still occurs, but selecting the zero halfword does not require testing the processing argument.

On the path requiring the conditional read, the order is:

- The additional F-fdd8589eca2d call completes.
- The separate global location supplies the displacement.
- That displacement is combined with the receiver address.
- A 16-bit value is read at the resulting address.
- The captured value is supplied to optional processing.

Thus, the address is determined from the displacement available after the additional accessor call, not from a value assumed at entry. Earlier dependencies, including that accessor, may affect both the displacement and the contents at the resulting address. No preservation assumption is justified for either.

F-818c773e8c8a receives these arguments in order:

| Position | Value |
|---|---|
| 1 | Result of the additional accessor call |
| 2 | Temporary-storage address |
| 3 | Original nonzero lookup result |
| 4 | Original processing argument |
| 5 | Selected 16-bit value |

The processing argument is forwarded unchanged, not converted to a Boolean. Zero suppresses the conditional memory reads, not the processing call itself. A negative nonzero processing argument satisfies the nonzero condition and retains its original 32-bit representation when forwarded.

### Normal completion

After optional processing returns, or immediately after it is skipped, the function writes the original request at receiver offset `0x104`. It then calls F-920630e53283 with the temporary-storage address and returns if that call returns normally.

The request write precedes finalization. It is not a guarantee that the request field still contains that value after finalization: the finalizer may access aliased or shared state.

No explicit local recovery or cleanup path is shown for an earlier dependency that fails to return normally.

## Side effects

### Direct-access inventory

The direct accesses established by the supplied representation are:

| Path or condition | Direct access |
|---|---|
| Every invocation | Read 8-bit receiver flag at `0x10C` |
| Record-only path | Write 32-bit request at `0x104` |
| Active path, before lookup | Write 8-bit option at `0x108` |
| Successful active path, before initialization | Read 8 bits at first accessor result plus `0x2C` |
| Same path, after second accessor | Read 32 bits at second accessor result plus `0x28` |
| Selector not `8`, suppression zero, processing argument nonzero | Read the separate global displacement location after the additional accessor |
| Same conditional path | Read 16 bits at receiver plus the displacement just obtained |
| Normal active completion | Write 32-bit request at `0x104`, before finalization |

The displacement read is a distinct direct access, not merely an unknown constant in the receiver layout. Its own storage width remains unresolved.

The function does not directly write the displacement location or clear the initial receiver flag. Dependencies may nevertheless change either.

### Ownership and lifetime

Temporary storage belongs to the invocation and is passed to initialization, optional processing, and finalization dependencies. The function does not provide persistent storage for a retained temporary address.

No direct heap allocation or deallocation is shown. Dependencies may allocate, release, retain, or register resources; none of those behaviors can be assumed absent or present.

The evidence does not establish ownership of the lookup result, accessor results, or initialization inputs. It also does not establish whether the temporary retains references to those inputs.

### Arithmetic, aliasing, and synchronization

The request and processing argument undergo no direct numeric transformation. The option is stored as an 8-bit value. The conditional field is read as 16 bits, with a 16-bit zero used on the alternative branches.

Fixed offsets are byte offsets. Address formation uses the receiver or accessor result plus the relevant offset or displacement. No overflow, range, or alignment checks are shown. The precise arithmetic treatment of the separately stored displacement cannot be fixed without its type and target details.

No atomicity, synchronization, concurrency guarantees, or volatile-access semantics are established. Dependencies are not assumed to preserve globals, receiver fields, external storage, or memory reached through aliases.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-1f46730b1ced | Receives receiver subobject address and request; zero result selects failure handling |
| F-a817d8d58c10 | No-argument failure call, annotated as non-returning |
| F-d2f99870cc43 | Receives lookup result and supplies a 32-bit value |
| F-bf557f07179f | Receives receiver and supplied 32-bit value |
| F-fa8898fa75ae | Receives original lookup result before temporary initialization |
| F-fdd8589eca2d | Supplies addresses or values for two initialization reads and, conditionally, optional processing |
| F-2c5a9f9c1eb3 | Receives temporary address, captured 32-bit value, and captured byte |
| F-5c5958cfd5fe | Supplies selector compared with `8` |
| F-818c773e8c8a | Receives the five optional-processing arguments |
| F-920630e53283 | Receives temporary address after the request write |

These roles do not specify dependency internals, ownership policies, failure behavior, or preservation guarantees.

## Acceptance examples

These are conditional obligations derived from static evidence, not executed tests. Each assumes valid reached accesses and normal return from dependencies except where stated otherwise.

1. **Noncanonical nonzero flag.**  
   With a flag byte of `2`, the function takes the record-only path: it writes the request and returns without an option write or dependency calls. The flag need not equal one.

2. **Lookup failure after an option change.**  
   With an initially zero flag and zero lookup result, the option write precedes F-a817d8d58c10. Under its non-return annotation, no temporary initialization or final direct request write follows.

3. **Selector value `8`.**  
   A successful lookup and selector result `8` still permit temporary initialization. They suppress optional processing, the third accessor call, the displacement read, and the conditional halfword read. The request write precedes finalization.

4. **Nonzero suppression.**  
   With selector result other than `8` and nonzero suppression, the additional accessor is called and optional processing receives a zero halfword. Neither conditional memory read is required, regardless of the processing argument.

5. **Zero processing argument.**  
   With selector result other than `8` and both control arguments zero, optional processing still occurs. Its fourth argument is zero and its fifth argument is a 16-bit zero. The displacement location is not read.

6. **Accessor changes the displacement.**  
   With selector result other than `8`, zero suppression, and nonzero processing argument, suppose the additional accessor changes the global displacement from one valid value to another. The conditional read uses the displacement obtained after that call. An earlier cached value does not satisfy the established ordering.

7. **Accessor changes the selected contents.**  
   Under the same branch conditions, suppose the additional accessor changes the 16-bit contents at the address subsequently selected by the displacement. The read occurs after the accessor and supplies the value then read. Preservation of the prior contents cannot be assumed.

8. **Distinct initialization objects.**  
   If the first two accessor calls return different valid addresses, initialization receives the byte from the first and the 32-bit value from the second.

9. **Finalizer overwrites the request.**  
   If finalization changes the request field through shared state, the required behavior is that this function’s request write occurred before finalization—not that its value survives the call.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the arguments, fields, and dependency operations.
- The global displacement location’s declared width, signedness, value, and mutation policy.
- The precise arithmetic interpretation of that displacement and complete target ABI details.
- Complete receiver and external-object layouts.
- Ownership and lifetime of lookup and accessor results.
- Dependency effects on globals, aliases, receiver storage, and external objects.
- Failure and unwinding behavior, including the mechanism behind the non-returning annotation.
- The temporary’s logical object size and resource obligations.
- Signed interpretation or ABI widening of the selected 16-bit processing argument.
- Concurrency and reentrancy requirements.

Treating the initial flag as a deferral mechanism or the last temporary call as resource cleanup is inferred intent, not established semantics.

## Evidence

Established static evidence consists of the supplied function representation: argument representations, fixed access offsets and widths, branch conditions, dependency arguments and ordering, 92-byte local storage extent, and the non-returning annotation.

The conditional expression also establishes a separate global displacement read after the additional accessor call and before the receiver-relative 16-bit read. It does not establish that the displacement is immutable or declare its storage type.

The dependency mapping supplies opaque identities only. No dependency implementation or preservation contract is available.

No runtime tests have been performed. No executable observations or dependency traces support this contract. Acceptance examples specify conditional behavior for later verification, not observed outcomes.
