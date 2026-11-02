<!-- Trusted provenance: F-2d525ad4ccd5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2d525ad4ccd5

## Purpose

F-2d525ad4ccd5 conditionally increments a signed 32-bit field in a caller-supplied object. When the calculated increment result is exactly four, it invokes additional dependencies and selects its return value according to their results. Every visible normal path ends with a common trailing dependency call.

The trigger is equality with four after an increment, not an “at least four” condition. With a field that remains stable between the function’s initial reads, an entry value of three triggers the additional calls. An entry value of four or greater does not cause a direct increment and does not trigger those calls.

The field may represent progress or an attempt count, but that interpretation is inferred intent. The object’s domain role and the meanings of the return values are not established.

## Interface

### Input and relevant layout

The function accepts one object address represented by a 32-bit integer-sized argument.

Its direct operations and dependency arguments use these locations:

| Location | Established use |
|---|---|
| Object address +0x6C | Interior address passed to two dependencies |
| Object address +0x74 | Interior address passed to one dependency |
| Object address +0x8C | Four-byte field read as a signed 32-bit integer and conditionally written |

The two interior addresses are eight bytes apart. Their referents’ sizes, types, independence, and ownership are unknown. Passing these addresses does not establish that dependencies access disjoint storage.

The function also has an external-state dependency on the nonzero threshold subpath:

1. An address-bearing value is read from a global location.
2. Four bytes are read at that newly obtained base value plus four.
3. Those four bytes are supplied as the second argument to F-438680ffebfe.

The global location holding the base and the storage addressed through that base are distinct access steps. The global base is not established as immutable, and it is not captured before the preceding dependency calls.

The target read is explicitly four bytes. The supplied representation does not independently declare the global base’s storage type or load width; its established role is to supply the address used for the subsequent read.

### Output

The function returns a 32-bit value selected as follows:

| Condition | Selected result |
|---|---|
| Threshold-specific branch is not entered | 3 |
| Threshold branch is entered and F-efdae002d289 returns zero | 1 |
| Threshold branch is entered and F-efdae002d289 returns nonzero | The 32-bit result of F-d126edc75a34 |

These are result-selection rules, not established success or error classifications.

Once selected, the result is retained across the remaining calls. The return values of F-438680ffebfe and F-5fef27ef670e do not replace it.

### Preconditions

For execution without invalid accesses:

- The field at object offset +0x8C must be readable.
- When the increment path is taken, that field must be writable.
- When the reset path is taken, it must remain writable after F-efdae002d289 returns.
- Interior addresses passed to dependencies must satisfy those dependencies’ requirements.
- On the nonzero threshold subpath, the global base must be readable after F-d126edc75a34 returns, and the address obtained from that read plus four must permit a four-byte read.
- All invoked dependencies must receive arguments and state valid under their own contracts.

No null check, bounds check, explicit alignment check, or local access-failure recovery is visible.

Byte order, alignment requirements, complete object extent, and behavior of address arithmetic at invalid or boundary addresses are not established.

## Behavior

### Initial comparison and increment

The function initially selects return value 3.

It reads the four-byte field at object offset +0x8C and compares the value with four using signed 32-bit interpretation.

If the observed value is four or greater:

- No direct field write occurs.
- All threshold-specific dependencies are skipped.
- Neither the global base nor its target is read by this function’s threshold-specific operations.
- F-5fef27ef670e is invoked with the original object address.
- If that call returns normally, the function returns 3.

If the first observed value is less than four, the function reads the field again to obtain the increment operand. It adds one, writes the calculated 32-bit result to the field, and tests that calculated result for equality with four.

The equality test uses the calculated increment result, not a third read of the field.

If the result is not four, the threshold-specific dependencies and external-state reads are skipped. F-5fef27ef670e is then invoked with the original object address, and the function returns 3 if that call returns normally.

With a stable field, only an entry value of three reaches the threshold-specific branch. Negative values are incremented without rejection or clamping.

### Threshold decision dependency

When the calculated increment result equals four, the function has already written four to offset +0x8C before invoking F-efdae002d289.

That dependency receives, in argument order:

1. Object address +0x74.
2. Object address +0x6C.

Its result is tested only for equality with zero.

#### Zero result

If F-efdae002d289 returns zero:

1. The selected return value becomes 1.
2. The function directly writes a four-byte zero to object offset +0x8C.
3. F-d126edc75a34 and F-438680ffebfe are skipped.
4. The global base and the four-byte target are not read by this function.
5. F-5fef27ef670e is invoked with the original object address.
6. If execution continues normally, the function returns 1.

The reset occurs after F-efdae002d289 returns. It therefore overwrites any value that dependency left in the field, provided the original storage remains valid and writable.

This does not guarantee a zero field at function return. The trailing dependency executes after the reset and may change the object.

#### Nonzero result

If F-efdae002d289 returns any nonzero value:

1. F-d126edc75a34 is invoked with the original object address and a second argument of zero.
2. Its 32-bit result becomes the selected return value.
3. After that call returns, the function reads the current address-bearing value from the global base location.
4. Using that newly read base, it reads four bytes at base plus four.
5. F-438680ffebfe is invoked with object address +0x6C as its first argument and the four-byte value just obtained as its second argument.
6. F-5fef27ef670e is invoked with the original object address.
7. If execution continues normally, the function returns the previously selected result from F-d126edc75a34.

There is no direct reset of offset +0x8C on this subpath. Its final value is nevertheless unknown because dependency mutation is not constrained by the supplied evidence.

Both external reads occur after F-d126edc75a34 returns. A dependency may change the global base itself, the contents of a target, or both. The target used by this function is determined by the later global-base read, not by an earlier snapshot of the base.

After the four-byte target read, F-438680ffebfe receives the captured value. Later changes to the global base or target do not retroactively change that argument.

### Common trailing call

F-5fef27ef670e receives the original object address on every visible normal control-flow path, including paths with no increment.

This is not an established exception-safe cleanup guarantee. An invalid access, a nonreturning dependency, or another abnormal transfer of control may prevent the trailing call. No local recovery is visible.

### Integer boundaries and multiple reads

For a stable field, the increment operand is between −2,147,483,648 and 3 inclusive. Adding one remains within signed 32-bit range. In particular:

- The minimum signed value becomes −2,147,483,647.
- Three becomes four.
- Four and larger signed values are excluded from the increment path.

The initial comparison and the increment operand come from separate reads. If the field changes between them, the second value need not satisfy the condition established by the first read.

No synchronization or consistency mechanism is visible. If an intervening change makes the second value the maximum signed 32-bit integer, the supplied representation does not establish a portable arithmetic outcome for adding one. Stable-field behavior must not be generalized into an unconditional overflow guarantee or an assumed wrapping rule.

## Side effects

### Direct memory operations

The established direct object writes are limited to:

- The increment-result write at +0x8C after an initial signed value below four.
- The zero write at +0x8C after an increment result of four and a zero result from F-efdae002d289.

No direct write through +0x6C or +0x74 is visible. Dependencies receive those addresses and may access storage according to contracts not supplied here.

On the nonzero threshold subpath, the function directly reads the global base and then the four-byte target at that base plus four. It does not directly write either external location. Dependency-mediated changes remain possible.

### Observable ordering

Where all corresponding operations occur, their established order is:

1. Initial field read and signed comparison.
2. Second field read.
3. Increment-result write.
4. F-efdae002d289 call.
5. Either:
   - Direct zero reset; or
   - F-d126edc75a34 call, global-base read, four-byte read at the newly obtained base plus four, and F-438680ffebfe call.
6. F-5fef27ef670e call.
7. Return of the selected result.

This ordering describes the supplied static representation. It does not establish atomicity, memory barriers, inter-thread visibility guarantees, or a supported concurrent-mutation model.

### Ownership and lifetime

No direct allocation, release, ownership transfer, or object replacement is visible.

No dependency is established to borrow, retain, release, or preserve any supplied storage. Those properties remain unknown rather than implicitly guaranteed.

The original object must remain usable for subsequent accesses on the selected path. In particular, the zero-result path writes to it after F-efdae002d289 returns, and all normal paths subsequently pass its original address to F-5fef27ef670e.

On the nonzero subpath, validity of an earlier global base or its former target is insufficient. The base read after F-d126edc75a34 must yield an address whose plus-four location supports the target read.

### Errors

There is no separate established error-reporting channel or local handling of memory faults, dependency failures, or abnormal exits.

Zero and nonzero results from F-efdae002d289 select branches; they are not established error categories. F-d126edc75a34’s returned bits are forwarded without interpretation or substitution.

## Dependencies

| Opaque function ID | Invocation condition | Arguments | Result use |
|---|---|---|---|
| F-efdae002d289 | Calculated increment equals four | Interior address +0x74, then interior address +0x6C | Zero versus nonzero selects the subpath |
| F-d126edc75a34 | Threshold reached and preceding result is nonzero | Original object address, then zero | Its 32-bit result becomes the selected return value |
| F-438680ffebfe | After F-d126edc75a34 and both external reads complete | Interior address +0x6C, then the four-byte target value | Return value unused |
| F-5fef27ef670e | Every visible normal path | Original object address | Return value unused |

The external-state dependency consists of a global base read followed by an indirect four-byte read. Neither the base nor the target contents are established as preserved across dependency calls. In particular, changes made by F-efdae002d289 or F-d126edc75a34 may affect which target is read.

Dependency contracts, mutation effects, ownership behavior, and normal-return guarantees are unavailable.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not runtime observations. Unless stated otherwise, the field is stable between its initial reads, memory accesses are valid, and dependencies return normally.

| Conditions | Required behavior |
|---|---|
| Entry field is 0 | Write 1; skip threshold-specific calls and external reads; invoke F-5fef27ef670e; return 3 |
| Entry field is 2 | Write 3; skip threshold-specific calls and external reads; invoke the trailing dependency; return 3 |
| Entry field is −1 | Write 0 as an increment result, not as the threshold reset; invoke only the trailing dependency; return 3 |
| Entry field is −2,147,483,648 | Write −2,147,483,647 without increment overflow; skip threshold-specific behavior; return 3 after the trailing call |
| Entry field is 4 | Perform no direct field write; skip threshold-specific behavior; return 3 after the trailing call |
| Entry field is 2,147,483,647 | Exclude the increment by signed comparison; perform no direct field write; return 3 after the trailing call |
| Entry field is 3; F-efdae002d289 returns zero | Write 4 before that call and zero afterward; skip processing, notification, and both external reads; invoke the trailing dependency; return 1 |
| Entry field is 3; F-efdae002d289 returns nonzero | Write 4; call F-d126edc75a34 with second argument zero; then read the global base, read four bytes at that base plus four, and call F-438680ffebfe; invoke the trailing dependency; return the processing result |

Additional mutation and ordering criteria:

- If F-efdae002d289 writes nine to the field and returns zero, the direct reset must overwrite nine with zero before the trailing call.
- If F-d126edc75a34 changes the global base from one valid target region to another, the subsequent target read must use the newly read base. Contents at the previous target must not be substituted.
- If F-d126edc75a34 leaves the base unchanged but changes the four bytes at base plus four, F-438680ffebfe must receive the value obtained by the later target read.
- If F-d126edc75a34 changes both the base and the new target’s contents, the argument must come from the new base’s plus-four location as observed at the later read.
- If a dependency leaves the global base invalid for the subsequent target access, successful completion is not guaranteed. No fallback to an earlier base is established.
- If F-d126edc75a34 returns 0xFFFFFFFF, that same 32-bit pattern must be returned after the later calls complete normally.
- If the trailing dependency changes the field, the final field value need not equal the last direct write.
- If the first field read observes three and the second observes two, the function writes three and skips the threshold branch. This describes the separate reads, not a supported concurrency guarantee.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the object, field, external value, and return values.
- Complete object layout and the referents of the interior addresses.
- Dependency mutation, retention, release, failure, and nonreturning behavior.
- Global-base storage type and independently specified load width.
- Semantic type and ownership of the four-byte target value.
- Alignment, byte order, and boundary-address arithmetic.
- Thread safety, reentrancy, atomicity, and behavior under concurrent modification.
- Arithmetic behavior when an intervening change makes the increment operand overflow signed 32-bit range.
- Any intended interpretation as a fourth attempt, phase transition, or retry limit.

The exact trigger and access order are established static behavior. Their domain purpose is not.

## Evidence

This contract is based on the supplied static function representation and opaque dependency mapping. It does not rely on the prior draft as independent evidence.

The representation establishes the field offsets, signed comparison, separate field reads, conditional writes, exact equality trigger, dependency arguments, result selection, and common trailing call.

On the nonzero threshold subpath, it also establishes two external access steps after F-d126edc75a34 returns: reading the global base value, then reading four bytes at that value plus four before invoking F-438680ffebfe. It does not establish that dependencies preserve either the base or its target.

Dependency bodies and runtime evidence were not supplied. No runtime tests have been performed. The acceptance examples are conditional requirements, not exercised results.
