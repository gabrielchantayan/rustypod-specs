<!-- Trusted provenance: F-6caf95f9c95b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6caf95f9c95b

## Purpose

F-6caf95f9c95b conditionally establishes a two-sided association between two caller-supplied objects. It first checks two prerequisites and asks a dependency whether the association already exists. If no association is reported, it attempts two insertions, one against each object. Only when both insertions report success does it increment one 32-bit field in each object.

If the second insertion fails, the function requests removal of the first insertion. The result of that removal is not checked.

This association interpretation is **inferred intent**. The established behavior consists of dependency calls, temporary-storage preparation, conditional field updates, and a packed 64-bit return value. The meanings of the objects, collections, counters, and insertion results are not supplied.

## Interface

The function accepts four 32-bit arguments:

| Argument | Established use |
|---|---|
| First object | Passed to the association query; used as the base for accesses at byte offsets 12 and 16; stored in a temporary supplied to the second insertion. |
| Second object | Passed to the second prerequisite check and the association query; used as the base for accesses at byte offsets 8 and 12; stored in a temporary supplied to the first insertion. |
| Payload word A | Initially occupies the first word of an eight-byte temporary region. |
| Payload word B | Initially occupies the second word of that eight-byte region. |

The first and second arguments are represented as 32-bit integers in the supplied material but are also used as addresses. The contract therefore treats them as 32-bit object-address values without asserting a higher-level pointer type.

The return value is a 64-bit composite with two independently significant halves:

- **Low 32 bits:** a status bit pattern.
- **High 32 bits:** the final contents of a four-byte local return-companion slot.

The companion slot initially contains the second object argument. On the path that attempts the second insertion, it is passed by address to a preparation dependency and then overwritten with the first object argument before that insertion. Consequently, the high half is not universally the second argument and should not be discarded when describing the interface.

The following low-half values are established:

| Bit pattern | Signed 32-bit interpretation | Condition |
|---|---:|---|
| `0x00000000` | 0 | Association query reports nonzero, or both insertions report nonzero. |
| `0xffffffce` | −50 | Either prerequisite reports zero. |
| `0xffffff94` | −108 | The first insertion reports zero, or the second insertion reports zero. |

These signed interpretations describe the low 32 bits only. They do not imply that the complete 64-bit return is a sign-extended status.

### Caller obligations

For any path that reaches address-based operations, the caller must supply object values suitable for those operations and for the dependencies receiving them. In particular, the successful two-insertion path requires readable and writable 32-bit fields at:

- First object, byte offset 12.
- Second object, byte offset 8.

The insertion and rollback dependencies receive addresses at first-object offset 16 and second-object offset 12. Their required storage sizes, alignment, initialization, and object invariants are unknown.

The supplied material does not establish null checks, address-range checks, lifetime checks, synchronization requirements, or permitted aliasing. Passing prerequisite checks is not evidence that all subsequent accesses are safe.

## Behavior

### Temporary state

Before the prerequisite calls, the function establishes:

1. A low-half status initially equal to zero.
2. A four-byte companion slot initially containing the second object argument.
3. An eight-byte temporary region whose first word contains payload word A and whose second word contains payload word B.

The two words of the eight-byte region are adjacent, with payload word B four bytes after payload word A. No byte-order interpretation beyond these 32-bit word positions is established.

### Prerequisite checks

The function first calls F-918216db7a4d with no explicit arguments.

- If its result is zero, the function returns immediately with low half `0xffffffce`.
- F-d94e68c9f6c2 is not called on this path.

If the first result is nonzero, the function calls F-d94e68c9f6c2 with the second object argument.

- If this result is zero, the function returns with low half `0xffffffce`.
- Otherwise, processing continues.

On either early-return path, the companion slot has not been passed by address or overwritten by this function; its established value is the original second object argument.

Neither prerequisite result is interpreted beyond zero versus nonzero. Their signs and other numeric properties do not select different behavior.

### Association query

After both prerequisites report nonzero, the function calls F-2d87d84fa14c with the first and second object arguments, in that order.

If the query returns nonzero:

- No insertion is attempted.
- No rollback is requested.
- Neither of the two direct field increments occurs.
- The return low half remains zero.
- The companion slot remains the second object argument.

Treating this as an “already associated” result is inferred intent. Statically, any nonzero query result selects this branch, including a value that a dependency might otherwise use as an error indication.

If the query returns zero, the function attempts to create the association.

### First insertion

The function calls F-c67cb20988a1 with:

- The address of the eight-byte temporary region.
- A size argument of 8.

After that call returns, the function overwrites the first 32-bit word of the region with the second object argument. It does not directly overwrite the region’s second word at this point.

It then calls F-bf997e89a5cb with:

- First-object address plus 16 bytes.
- The address of the eight-byte temporary region.

Thus, at entry to this insertion call:

- The region’s first word is the second object argument.
- The region’s second word is whatever remains after F-c67cb20988a1 returns.

The latter must not automatically be described as payload word B or as zero. The preparation dependency’s behavior is unavailable.

If the first insertion returns zero:

- No second insertion is attempted.
- No rollback is requested.
- Neither direct field increment occurs.
- The low return half becomes `0xffffff94`.
- The companion slot remains the second object argument under the explicit local-storage operations shown.

If the first insertion returns nonzero, that exact 32-bit result is retained for possible rollback.

### Second insertion

After a nonzero first insertion result, the function calls F-c67cb20988a1 again, this time with:

- The address of the four-byte companion slot.
- A size argument of 4.

It then overwrites that slot with the first object argument and calls F-bf997e89a5cb with:

- Second-object address plus 12 bytes.
- The address of the companion slot.

The second insertion therefore receives a pointer to a four-byte temporary whose entry value is the first object argument.

The insertion dependency is not shown as taking a size argument. The differing temporary widths are a property of the supplied call sites, not proof of how much memory that dependency reads or writes.

### Successful insertion pair

If the second insertion returns nonzero, the function performs two direct updates, in this order:

1. Read the 32-bit field at first-object offset 12, add one, and write the result back to that field.
2. Read the 32-bit field at second-object offset 8, add one, and write the result back to that field.

The second read occurs after the first write. If the two field addresses coincide, the second update therefore observes the first update’s stored value, subject to the underlying execution model. The supplied function does not reject aliasing or combine the two updates into one.

Each field and its stored result are 32 bits wide. No overflow check, saturation, or overflow-specific return path is shown. The exact language-level treatment of signed overflow is not established by the supplied representation; a wider, unbounded mathematical counter must not be assumed.

The low return half remains zero. The high half is the companion slot’s value after the second insertion returns. It was set to the first object argument immediately before that call, but the dependency receives its address and may modify it.

### Second-insertion failure and rollback

If the second insertion returns zero, the function calls F-63ecaced33d1 with:

1. First-object address plus 16 bytes.
2. The literal value 1.
3. The exact nonzero result returned by the first insertion.

The function does not inspect a rollback result. It does not retry either insertion, attempt another removal, or increment either field. Its low return half becomes `0xffffff94`.

The high half is the companion slot’s final contents. It was set to the first object argument before the second insertion, but that insertion can modify it through the supplied pointer. The available material also does not establish whether dependencies retain temporary pointers or make subsequent changes through them.

A failure status therefore does not establish that the association state is unchanged: the first insertion occurred, and successful reversal is not verified.

## Side effects

The function itself directly writes only temporary storage and, on successful completion of both insertions, the two object fields described above.

Additional side effects may occur through every dependency call. No supplied dependency contract establishes preservation of globals, object contents, allocator state, or other external state.

Important boundaries are:

- A prerequisite failure prevents all later calls, but does not establish that the prerequisite calls were side-effect-free.
- A nonzero query result prevents direct insertion and counter work, but does not establish that the query was read-only.
- A zero insertion result is treated as failure without proof that the failed call made no changes.
- Rollback is requested only after a nonzero first insertion and a zero second insertion.
- Neither counter is directly updated on an insertion-failure path.
- There is no explicit allocation or release operation in this function. Dependencies may perform either; ownership cannot be inferred from their call shapes.

Both temporary addresses refer to function-local storage. That storage is not a durable return object. Whether the insertion dependency copies data, consumes it synchronously, stores its address, or assumes ownership is unknown. Any requirement that the data remain valid after return needs a dependency contract not present here.

No atomicity, locking, exception handling, fault recovery, or transactional guarantee is established.

## Dependencies

| Opaque ID | Established invocation and use |
|---|---|
| F-918216db7a4d | Called first with no explicit arguments. Zero causes the prerequisite-failure return. |
| F-d94e68c9f6c2 | Called with the second object only if the first prerequisite reports nonzero. Zero causes the same failure status. |
| F-2d87d84fa14c | Called with both object arguments after the prerequisites. Zero selects insertion; nonzero selects immediate status-zero return. |
| F-c67cb20988a1 | Called with temporary address and byte count: first 8, later 4 if the first insertion succeeds. No result is inspected. Its effects on the supplied bytes are unknown. |
| F-bf997e89a5cb | Called for each insertion. Zero means failure to this caller; any nonzero value means success to this caller. The first nonzero result is also used as a rollback argument. |
| F-63ecaced33d1 | Called only after second-insertion failure, with the first insertion target, literal 1, and first insertion result. No outcome is checked. |

The names “prerequisite,” “query,” “preparation,” “insertion,” and “rollback” describe observed roles or inferred intent, not independently established dependency semantics.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence. They are not runtime test results.

1. **First prerequisite rejects.**  
   If F-918216db7a4d returns zero, the return is a 64-bit composite with the original second argument in the high half and `0xffffffce` in the low half. No other dependency is called.

2. **Second prerequisite rejects.**  
   If the first prerequisite returns nonzero and F-d94e68c9f6c2 returns zero, the association query is not called. The return halves match the preceding rejection case.

3. **Query reports nonzero.**  
   If both prerequisites pass and the query returns any nonzero value, no preparation, insertion, rollback, or direct counter update occurs. The return high half is the second argument and the low half is zero.

4. **First insertion fails.**  
   If the query returns zero and the first insertion returns zero, exactly the eight-byte preparation and first insertion are attempted after the query. The low return half is `0xffffff94`; the second insertion and rollback are absent.

5. **Both insertions succeed without changing the companion slot.**  
   Assume suitable disjoint fields initially hold 7 and 20, dependencies do not otherwise change them, and both insertions return nonzero. The fields become 8 and 21, in that write order. If the second insertion leaves its temporary unchanged, the return high half is the first argument and the low half is zero.

6. **Second insertion fails and changes its temporary.**  
   Suppose the first insertion returns a nonzero token, the second writes a different 32-bit value into its supplied temporary and returns zero, and rollback does not subsequently alter that temporary. Rollback receives the original first-insertion token. The return high half is the modified temporary value, not necessarily either object argument; the low half is `0xffffff94`.

7. **Counter fields alias.**  
   If the two counter addresses are identical, initially contain 10, and no dependency changes that location, successful insertions lead to successive stored values 11 and 12. This scenario is conditional on the overlapping object layout being valid.

## Uncertainties

The following are **documented_only**:

- Actual object types, layouts beyond the accessed offsets, and meanings of the two updated fields.
- Dependency semantics, global effects, memory ownership, pointer retention, and failure atomicity.
- Whether the preparation dependency clears, transforms, records, or otherwise processes its supplied region.
- Whether payload word B survives preparation and whether either payload word has semantic significance.
- Whether nonzero query values distinguish existing associations from errors.
- Whether rollback succeeds or restores all effects of the first insertion.
- Valid address ranges, alignment, object lifetimes, concurrency rules, and permitted overlap.
- Exact overflow behavior, target byte order, calling-convention interpretation of the packed return, and any caller convention that ignores its high half.
- Behavior when a dependency does not return normally or a memory access faults.

## Evidence

The supplied static function representation establishes the branch conditions, call order, arguments, temporary widths, direct field offsets, ordered updates, rollback trigger, and composition of the 64-bit return.

It does not include dependency implementations, caller contracts, object definitions, or execution observations. Association semantics and counter interpretations are inferred intent, not additional established guarantees.

No runtime tests have been performed.
