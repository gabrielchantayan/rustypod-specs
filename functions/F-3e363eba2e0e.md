<!-- Trusted provenance: F-3e363eba2e0e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3e363eba2e0e

## Purpose

F-3e363eba2e0e traverses records obtained through an opaque iteration interface and selects one of two actions for each record:

- Invoke F-353d18cfd7f5 when both a record-specific byte and an object-specific byte are zero.
- Otherwise, if a second record byte equals 2, pass the record’s 32-bit payload to F-0f8f09404dad and subsequently write 1 to that second byte.

The first action takes precedence over the second. A record that satisfies the first action’s condition does not have its second byte inspected by this function during that iteration.

This is an established description of the supplied static representation. Interpreting the bytes as flags, the payload as a resource, or the calls as activation, queuing, or cleanup would be inferred intent rather than established behavior.

## Interface

### Inputs

The function has four arguments:

| Argument | Represented width | Established use |
|---|---:|---|
| Object address | 32 bits | Base address for an optional byte read at offset `0x72`, an address passed to F-353d18cfd7f5, and a derived address at offset `0x50` passed to F-0f8f09404dad. |
| Collection address | 32 bits | Used to derive the address at offset `0x08` passed to F-8ca21d3a8cca. Also initially copied into a local 32-bit slot that is subsequently reset before iteration. |
| Initial state word A | 32 bits | Copied into local storage whose address is passed to the iteration-related dependencies. |
| Initial state word B | 32 bits | Copied into a second local word associated with the apparent iteration-state storage. Its interpretation is not established. |

The two address arguments are represented as 32-bit integers in the supplied material. Their uses are address-like; this does not establish a portable integer-to-pointer interface for another execution environment.

### Return value

There is no explicit return value. The visible result consists of dependency calls and memory changes. The function does not return a record count, status, error code, or processed payload.

Normal return is reached only after an iteration request returns zero and F-d0d3f100effc returns.

### Locally maintained state

The function maintains:

- A 32-bit record-address/output slot. It initially receives the collection address, is explicitly reset to zero after initialization, and is passed by address to F-b38a3aba0e18.
- Two local 32-bit words initialized from state words A and B. The address of the first is passed to the initialization, iteration, and finalization dependencies.

The supplied representation suggests an eight-byte iteration-state area formed from the two state words. That aggregate layout and the meaning of its fields are not independently established. The visible contract is that the same first-word address is supplied to all three iteration-related dependencies.

### Memory accessed directly

| Address or region | Access |
|---|---|
| Object address + `0x72` | Conditional one-byte read |
| Object address + `0x50` | Address calculation and dependency argument; no direct access by this function |
| Collection address + `0x08` | Address calculation and dependency argument; no direct access by this function |
| Current record address + `0x08` | One-byte read on every successful iteration |
| Current record address + `0x09` | Conditional one-byte read and, on one branch, a one-byte write |
| Current record address + `0x0c` | Conditional 32-bit read |

No complete object, collection, or record layout is established. In particular, the function does not directly inspect the first eight bytes of a record.

### Preconditions for defined normal execution

All addresses actually dereferenced must designate valid storage for the required access width. The record byte at offset `0x09` must be writable whenever the payload-processing branch reaches its post-call write.

The dependencies must support the supplied local-state addresses, argument shapes, and lifetimes. A nonzero iteration result must leave an address usable for the accesses taken by that iteration.

The function performs no visible null checks, bounds checks, alignment checks, or address-overflow checks. These requirements are preconditions, not validated guarantees.

## Behavior

### Initialization

The function first copies the collection address and the two state-word arguments into local storage.

It then calls F-8ca21d3a8cca with:

1. The address of local state word A.
2. The collection address plus `0x08`.

The return value of this call, if any, is not used.

After the call returns, the record-address/output slot is explicitly set to zero. Thus, its initial copy of the collection address is not the value supplied to the first iteration request.

The function does not visibly initialize the two local state words again after the initialization call.

### Iteration request and termination

Each iteration begins with a call to F-b38a3aba0e18, passing:

1. The same local-state address used during initialization.
2. The address of the record-address/output slot.

A return value equal to zero ends iteration. Any nonzero value continues into record processing; the visible condition does not distinguish positive from negative nonzero values.

The function does not independently verify that the output slot changed or contains a nonzero address. A nonzero result accompanied by an invalid output address therefore leads to an unchecked record access.

The output slot is not reset between successful iteration requests. Whether a dependency overwrites it, retains it, or uses its prior value is not established.

### Record selection and action precedence

For each nonzero iteration result, the function first reads the byte at current record address plus `0x08`.

If that byte is nonzero:

- The object byte at offset `0x72` is not read by this function.
- Processing proceeds to the second-record-byte decision.

If the record byte is zero:

- The function reads the object byte at offset `0x72`.
- If that object byte is also zero, it calls F-353d18cfd7f5 with the unadjusted object address.
- Otherwise, processing proceeds to the second-record-byte decision.

When F-353d18cfd7f5 is selected, this function does not additionally inspect the record byte at offset `0x09`, read the payload, call F-0f8f09404dad, or perform the explicit record-byte write during that iteration.

There is no visible “already handled” guard around F-353d18cfd7f5. Multiple yielded records can cause repeated calls. Its effects can influence later iterations, including later reads of the object byte, but those effects are opaque.

### Conditional payload processing

The second-record-byte decision is reached only when the first-action condition was not satisfied.

The function reads the byte at current record address plus `0x09`.

- If it is not exactly 2, this function takes no further record-specific action before requesting another iteration.
- If it is exactly 2, the function reads the 32-bit word at current record address plus `0x0c`.

That word is passed to F-0f8f09404dad together with the object address plus `0x50`. The first argument is the derived object address; the second is the payload word.

After F-0f8f09404dad returns normally, the function writes the one-byte value 1 at offset `0x09` of the address then held in the record-address/output slot.

The ordering is significant:

1. The second byte is read and compared with 2.
2. The 32-bit payload is read.
3. F-0f8f09404dad is called.
4. The byte is written as 1.

The write is not conditional on a returned success status. No dependency return value is checked on this branch.

The record-address slot was exposed by address to the iteration dependency. No independent guarantee establishes that it cannot be changed indirectly during later calls. Consequently, identifying the post-call write target with the record used for the earlier reads requires address stability across the payload call. Under that condition, the visible transition is from byte value 2 to byte value 1 on the same record.

### Finalization

After an iteration request returns zero, the function calls F-d0d3f100effc with the local-state address.

No further record processing occurs after that terminating result. The function returns after finalization returns.

There is no visible alternate finalization path for a dependency that does not return normally or for a failed memory access.

## Side effects

### Direct writes

The direct writes visible in this function are:

- Initialization of local storage.
- Resetting the local record-address/output slot to zero before the first iteration request.
- Writing the one-byte value 1 at record offset `0x09` after the payload-processing call returns.

No other external-memory write is explicit in the supplied body.

### Indirect effects and ownership

Every dependency may have additional effects not established here. The function provides no basis for assuming that dependencies preserve global state, preserve reachable object fields, allocate memory, release memory, or transfer ownership.

The payload is passed as a 32-bit value. Its interpretation as a pointer, identifier, handle, count, or other quantity is unknown. Passing it does not itself establish ownership transfer.

Likewise, the final call’s placement after iteration suggests a lifecycle role, but does not prove that it frees or releases anything.

Addresses into local storage are passed to dependencies. Their validity is limited to this invocation’s lifetime. Any retention and later use of those addresses would require a dependency contract not supplied here.

### Observable ordering

On normal execution:

- Initialization precedes the explicit output-slot reset.
- The reset precedes the first iteration request.
- Each successful iteration request precedes that iteration’s record reads.
- The record byte at `0x08` is read before any optional object-byte or second-record-byte read.
- The optional object-byte read precedes branch selection.
- The payload read precedes F-0f8f09404dad.
- F-0f8f09404dad returns before the explicit byte write.
- A zero iteration result precedes finalization.

These are source-level sequencing facts, not guarantees about concurrent observation or machine-level memory barriers.

## Dependencies

| Opaque ID | Arguments supplied | Result handling and established role |
|---|---|---|
| F-8ca21d3a8cca | Local-state address; collection address + `0x08` | Called once before iteration. Result unused. Initialization role inferred from placement. |
| F-b38a3aba0e18 | Local-state address; record-output-slot address | Called repeatedly. Zero terminates; any nonzero result triggers record processing. |
| F-353d18cfd7f5 | Object address | Called when record byte `0x08` and object byte `0x72` are both zero. Result unused. |
| F-0f8f09404dad | Object address + `0x50`; 32-bit record payload | Called on the alternate branch when record byte `0x09` equals 2. Result unused; followed by a byte write if it returns. |
| F-d0d3f100effc | Local-state address | Called once after normal iteration termination. Result unused. Finalization role inferred from placement. |

Dependency implementations and contracts are unavailable. No stronger semantics are assigned to them.

## Acceptance examples

These examples are conditional static expectations, not executed tests. They assume valid memory, normally returning dependencies, and stable record addresses except where stated.

1. **Immediate exhaustion.**  
   The first iteration request returns zero. Expected: initialization, one iteration request, finalization, and return. No record-field read or record-action call occurs.

2. **First action takes precedence.**  
   A successful iteration yields a record whose byte at `0x08` is zero, while the object byte at `0x72` is zero. Even if the record byte at `0x09` contains 2, expected: F-353d18cfd7f5 is called; this function does not read that second byte or payload and does not perform its explicit write for this iteration.

3. **Nonzero record byte bypasses the object flag read.**  
   The record byte at `0x08` is nonzero and its byte at `0x09` is 2. Expected: no direct read of object offset `0x72`; payload read and F-0f8f09404dad call; then byte `0x09` becomes 1 if the call returns and the record address remains stable.

4. **Object byte enables the alternate decision.**  
   The record byte at `0x08` is zero, the object byte at `0x72` is nonzero, and the record byte at `0x09` is 2. Expected: both initial bytes are read, F-353d18cfd7f5 is not called, and payload processing occurs.

5. **Second byte differs from 2.**  
   The first-action condition is false and the record byte at `0x09` is 0, 1, or 255. Expected: no payload read, no payload-processing call, and no explicit external write by this function for that iteration.

6. **Repeated record under stable state.**  
   A record is yielded twice, the first-action condition remains false, and its second byte initially equals 2. If no dependency changes that byte or redirects the output slot, the first visit writes 1; the second visit does not repeat payload processing.

7. **Negative nonzero iteration result.**  
   An iteration request returns a negative nonzero 32-bit result and supplies a valid record address. Expected: record processing continues. Termination requires zero, not a positive-only success value.

8. **Payload call does not return.**  
   Payload processing is selected, but F-0f8f09404dad does not return normally. The visible body does not reach its subsequent byte write, next iteration request, or finalization call.

## Uncertainties

The following are **documented_only**:

- Complete object, collection, record, and iterator-state layouts.
- Exact aggregate layout of the two local state words.
- Meaning of the flag-like bytes and the payload.
- Dependency allocation, release, ownership, aliasing, and global-state effects.
- Whether iteration can repeat records, mutate records, or retain supplied local addresses.
- Whether the record-output slot remains unchanged during action calls.
- Dependency failure conventions beyond the iterator’s visible zero/nonzero distinction.
- Alignment requirements, byte order, and target-specific address-conversion rules.
- Overflow behavior for the 32-bit address additions. No widening or overflow handling is visible; a portable wraparound guarantee is not established.
- Concurrency, atomicity, and synchronization behavior.

The byte comparisons use only zero and 2, so signed-versus-unsigned interpretation of the byte type does not change the visible branch predicates.

## Evidence

The supplied static body establishes argument uses, explicit access widths and offsets, branch precedence, call arguments, zero/nonzero iteration control, and the post-call byte write.

Initialization, iteration, and finalization terminology describes apparent roles inferred from call placement and data flow. It does not establish the dependencies’ internal behavior.

No dependency bodies, external layout definitions, calling-convention documentation, or runtime traces were supplied. No runtime tests have been performed. The acceptance examples state conditional expectations derived from static evidence, not observed execution results.
