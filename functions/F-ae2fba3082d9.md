<!-- Trusted provenance: F-ae2fba3082d9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ae2fba3082d9

## Purpose

F-ae2fba3082d9 dispatches object-related requests. Depending on request and representation, it retrieves entry-like content, constructs a shared geometry-like result, exposes state or addresses, formats associated content, or invokes position-changing operations.

Two result channels are distinct:

- A 32-bit return word, zero or one on ordinary completion.
- A caller-provided 32-bit output slot, written only on specified paths.

Return one can accompany output zero. Return zero does not guarantee output preservation: dependencies and aliasing may affect it. “No direct output write” refers only to this function’s explicit stores.

The operations below are established from supplied static evidence. Terms such as “entry,” “geometry,” “registration,” and “cleanup” describe inferred roles, not verified dependency semantics.

## Interface

### Arguments and widths

| Argument | Established interpretation |
|---|---|
| Object address | 32-bit address used for object-relative accesses |
| Representation | 32-bit value compared with external representation values |
| Request | Unsigned 32-bit selector; ordering comparisons sometimes interpret its bits as signed |
| Output address | Address of a writable 32-bit slot whenever an output-writing path is reached |

Return values, output words, and exposed addresses are 32 bits.

### Object layout

Offsets are bytes from the object address.

| Offset | Width | Observed use |
|---|---|---|
| +0x18 | 8 bits | Mode |
| +0x1c | Inline region, extent unknown | Its address is passed to dependencies |
| +0x8b8 | 32 bits | Current position; also exposed by address |
| +0x8bc | Signed 32 bits | Lower bound |
| +0x8c0 | Signed 32 bits | Upper bound; refreshed on one path |
| +0x8c4 | 32 bits | Associated-content branch choice |
| +0x8c8 | 32 bits | Formatting branch choice |
| +0x8e0 | 8 bits | Alternate-resource branch choice |

The associated-subobject argument is object address plus 0x1c, not a pointer loaded from there. No position/bounds invariant is established.

### External-value labels

Labels identify external storage roles, not guaranteed-distinct values:

- **P:** primary dispatch pivot and relative-entry selector.
- **C:** current-entry selector.
- **D:** a fresh C value ANDed with T shifted right by 15.
- **E, T:** entry and geometry representations.
- **A, L, U, V:** position-address, lower-bound, refreshed-upper-bound, and current-value representations.
- **S, N, R:** formatted, numeric-count, and alternate-resource representations.
- **Baseline position resource:** read by both 0x848c and 0x848d on their high-bound branch.
- **Replacement position resource:** additionally read by 0x848c, replacing the baseline value.
- **Entry registration value, relative registration value, current registration value:** three distinct external globals supplying the second argument to F-804adc29bbfb at their respective initialization sites.
- **Common registration value:** one external global supplying F-804adc29bbfb’s third argument at all three sites.

The registration globals have distinct storage identities; their values need not differ. The common registration value is freshly read at each reached call, not captured once for the invocation.

### Preconditions and ownership

Every reached access requires valid storage. No object or output null check is visible. Paths without direct output stores do not themselves dereference output.

Output may alias object fields or shared state. Returned addresses may identify object-owned or shared storage. Ownership transfer, retention rights, synchronization, allocation, release, and lifetime are not established. Cleanup-like calls do not prove deallocation.

## Behavior

### Initialization and snapshots

Initialization occurs before dispatch rejection.

Retain a shared-base address. Independently reload its global and test bit zero of the 32-bit word at that fresh base plus four. If clear, reload the global again and invoke F-b255b26284e6 with that acquisition base plus four.

On nonzero acquisition:

1. Invoke F-398e0e9a318a on the retained base plus 0x20, with fixed external data.
2. After it returns, read the entry registration value and common registration value. Invoke F-804adc29bbfb with the preparation result as first argument, the entry registration value as second, and the common registration value as third.
3. Invoke F-d08d3ca17b81 on the retained base plus four.

Both registration reads occur after preparation returns and before registration is invoked. Their relative evaluation order is not established. The retained, tested, and acquired bases need not coincide.

Next capture, in order, the default formatting resource, a formatting-object pointer, a four-word descriptor pointer, and C. Captured C controls the later movement decision; dispatch performs additional external reads.

Later initialization tests examine bit zero of 32-bit words at offsets +8, +0x0c, +0x10, and +0x14 from the retained base, respectively for current entry, geometry, relative entries, and descriptor. Acquisition uses separately loaded guard addresses. Completion reloads the applicable guard address after intervening calls. These locations need not coincide.

### Dispatch precedence

1. Equality with a fresh P read selects relative handling.
2. Otherwise, signed request below a separately read signed P enters the lower region.
3. Otherwise, only literal requests 0x8488–0x848f are considered.

Within the lower region, form D and test equality first. Equality selects current-entry handling. Otherwise:

- Signed request at least signed D: test a fresh C, then relative selectors 0x8484 and 0x8485.
- Signed request below signed D: test 0x7e80, 0x7e81, then 0x8481.

Unmatched requests return zero. All literal-request behavior is conditional on reaching its branch; external selector values can intercept it.

### Relative entries

P and routed requests 0x8484–0x8485 share this behavior.

Read mode before testing E. Representation unequal to E returns zero without a direct output write. With E and mode other than three, write zero and return one.

For mode three:

- Compute displacement as request minus 0x8485 using unsigned 32-bit arithmetic.
- Add current-position bits modulo 2³²; interpret the result as a signed candidate.
- Check candidate against lower bound inclusively, then upper bound inclusively only if the first check passes.
- Out-of-range candidates write zero and return one without retrieval.
- Otherwise invoke F-bfda5fbfe0b0 with object, temporary result storage, candidate, and zero.

Inspect the temporary’s first 32-bit word only after nonzero retrieval. Retrieval zero, or a zero first word, invokes F-52cae007644e before writing zero and returning one.

On success, invoke F-edf86093a19f with object and the captured candidate. Then test the relative-entry guard. Successful acquisition causes:

1. F-c5a3e2a1a1d4 with externally read storage and initializer values, stride 0xb8, and count three.
2. After that initializer returns, fresh reads of the relative registration value and common registration value.
3. F-804adc29bbfb with **literal zero**, the relative registration value, and the common registration value.
4. F-d08d3ca17b81 using a freshly read relative guard address.

The initializer’s result is not registration’s first argument. No ordering between the two registration argument reads is established.

Load the relative-array base. Compute destination as base plus displacement times 0xb8 plus 0xb8, modulo 2³². Invoke F-8c586dbad7fd with the temporary and destination; then write that same computed destination to output without reloading the base. Return one.

Thus 0x8484 selects the base slot and 0x8485 the following slot. P’s destination depends on its value. No slot-range validation is visible.

### Current entries

Mode zero returns zero without a direct output write. E precedes T.

**E representation**

Invoke F-bfda5fbfe0b0 with object, temporary storage, current position read for the call, and three. Zero invokes F-52cae007644e, then writes zero and returns one. Unlike relative retrieval, a zero first temporary word does not itself reject success.

After nonzero retrieval:

1. Reread current position and invoke F-edf86093a19f.
2. Test the current-entry guard.
3. On successful acquisition, invoke F-a891c2749310 with a freshly read shared-entry address.
4. After that initializer returns, freshly read the current registration value and common registration value. Invoke F-804adc29bbfb with the initializer’s result, the current registration value, and the common registration value.
5. Complete the guard using a fresh guard-address read.

The two registration argument reads have no established relative order. Neither reuses a registration value captured at entry.

Reload the shared-entry address for F-8c586dbad7fd, passing the temporary first. Independently reload the shared-entry address for output. Application destination and returned address can differ.

After output, compare request with captured C. If equal:

- Post-output position unequal to upper bound invokes F-2690d8d63608 with object and one.
- Equal values cause a position reread; nonzero invokes F-e7b5b40d1609 with object and zero.
- Equal values followed by a zero reread invoke neither.

Return one. No rollback is visible.

**T representation, only after E fails**

F-1e7091dc86a0 obtains a handle temporary from the associated subobject. Capture whether it is nonzero, zero an auxiliary word, and invoke F-01b37b84d143 on that auxiliary word.

Captured absence causes F-01b37b84d143 on the handle temporary, then return zero without a direct output write.

Otherwise:

- F-9d06d181244d obtains a record using selector three. Consume two 32-bit dimensions and a 16-bit orientation. Swap dimensions for orientation 0x005a or 0x010e.
- Invoke F-56fd8d6069bf, then F-47c987e17054 using its result; both receive the associated-subobject address.
- Derive a receiver from the then-current handle: its first 32-bit pointer word if nonzero, otherwise null.
- Invoke F-cb9fa74c0a71 with three.
- Then read the captured receiver’s virtual-table entry at byte offset 0x24 and invoke F-20d91e7a468b with the derived values and two output-word addresses.

No controlled null rejection precedes virtual dispatch.

Test the geometry guard; successful acquisition is followed directly by completion. Capture the geometry-storage pointer. Store zero, zero, the virtual operation’s second output, and its first output, in that order. The first store uses a separate geometry-global read; the remaining stores use the captured pointer.

Invoke F-cca11044abb5 with the captured pointer and a four-word descriptor containing zero, zero, and the selected dimensions. Write the captured geometry address to output, invoke F-01b37b84d143 on the handle temporary, and return one.

Other representations return zero without a direct output write.

### Numeric values and addresses

Routed 0x7e80 writes upper bound plus one and returns one without representation checking. Routed 0x7e81 writes zero and returns one.

Routed 0x8481 rejects mode zero. Otherwise representation precedence is **A, T, L, U, V**:

| First match | Output |
|---|---|
| A | Object address plus 0x8b8 |
| T | Captured descriptor address |
| L | Lower bound |
| U | F-28fd04957d82 on associated subobject, minus one |
| V | Current position |

Successful matches return one. No match returns zero without a direct output write. U stores the refreshed upper bound before copying it to output.

For T, successful descriptor-guard acquisition writes 2, 2, 0x41, and 0x40, in order, through the captured descriptor pointer, then completes with a fresh guard-address read. Changing the descriptor global during acquisition does not redirect these stores or output. Skipped initialization leaves the descriptor untouched directly.

### Formatting and associated content

Except for explicit captures, formatting-object arguments use fresh global reads. Common extraction invokes F-ecd1af0519f9 with another fresh formatting pointer, writes its 32-bit result, and returns one.

**0x8488:** No representation check. Invoke F-cd58c20fc91a, obtain a receiver through F-4f0f1cd223e8, then an associated record through F-ce99e0891f09. Subtract 0x076c from the upper 16 bits of the record’s second word and compare the result with 0x008c using less-than.

If true, invoke F-6b51e92a470d at receiver virtual-table offset 0x88 with a fresh formatting pointer, eight, and the record address. Otherwise obtain F-7a3d159c02b4 from offset 0x0c of the captured formatting object’s virtual table, but supply a fresh formatting pointer as receiver. Then extract. An unsigned bounded-range interpretation is not established.

**0x8489:** Return zero without a direct output write.

**0x848a:** Require S. If +0x8c4 is zero, obtain context through F-9d0ecc5f6adf, then read and resolve an external resource through F-c0b020cd3882; pass the result to F-151ab4bfe553.

Otherwise obtain a two-word temporary through F-0e5f1b2977d6 with associated subobject and 0xffffffff. Compare its address with the captured formatting pointer. Unless equal, invoke F-151ab4bfe553 with a fresh formatting pointer and the temporary’s second word. Invoke F-289ec4311c64 on the temporary. Then extract.

**0x848b:** Capture upper minus lower plus one before representation checking. S precedes N.

- S: obtain context. Count one selects the captured default resource; otherwise load the alternate resource after context acquisition. Resolve, invoke F-1f62160dfd52 with captured count, then extract.
- N after S fails: write captured count and return one.
- Neither: return zero without a direct output write.

**0x848c and 0x848d:** Require S. Test signed upper bound before context acquisition.

Below one, obtain context, resolve the captured default resource, and invoke F-1f62160dfd52 without position arguments.

At least one:

1. Obtain context through F-9d0ecc5f6adf.
2. Unconditionally read the baseline position resource.
3. For 0x848c only, read the replacement resource, replacing the selected value.
4. Resolve the selected value through F-c0b020cd3882.
5. After resolution, read current-plus-one and upper-plus-one for F-1f62160dfd52.

Then extract. The discarded baseline read remains established for 0x848c. Branch selection and numeric arguments can reflect different object states.

### Requests 0x848e and 0x848f

For 0x848e matching S, read +0x8c8. Zero selects F-438680ffebfe with fixed data; nonzero selects F-1f62160dfd52 with different fixed data. Extract and write output, establishing status one.

Both requests then enter alternate handling; 0x848f starts with status zero:

- Capture R for lookup; independently read R for representation comparison.
- Mismatch returns established status.
- Read +0x8e0 and select its zero/nonzero branch.
- Obtain context through F-9d0ecc5f6adf.
- Afterward load the selected branch’s resource.
- Invoke F-5710a8628cba with context, captured R, and resource.
- Zero returns established status without another direct output write.
- Nonzero is written to output; return one.

If S and the comparison value of R coincide, 0x848e can write output twice.

## Side effects

Direct stores affect output, refreshed upper bound, shared descriptor and geometry storage, and dependency-facing temporaries. Dependencies may modify reachable state, globals, handles, or virtual tables.

Important sequencing includes output before current-entry movement, upper-bound refresh before output, geometry output before handle cleanup, and 0x848e’s first output before alternate lookup.

All three F-804adc29bbfb sites read their site-specific registration global and the same common registration global after their preceding initializer returns. The first arguments differ: entry preparation result, literal zero for relative entries, and current-entry initializer result. Earlier reads do not freeze later registration values.

Relative displacement and slot arithmetic are unsigned 32-bit operations. Counts, refresh subtraction, and signed position increments have no established overflow policy. No comprehensive overflow, address-wrap, returned-pointer, or slot-range checks are visible.

## Dependencies

Complete callee contracts are unavailable.

| Opaque IDs | Observed role |
|---|---|
| F-b255b26284e6; F-d08d3ca17b81 | Guard acquisition and completion |
| F-398e0e9a318a; F-804adc29bbfb | Entry preparation; three-argument registration-like operation |
| F-c5a3e2a1a1d4; F-a891c2749310 | Relative/current shared-entry initialization |
| F-bfda5fbfe0b0; F-edf86093a19f; F-52cae007644e; F-8c586dbad7fd | Retrieval, subsequent action, failure action, application |
| F-2690d8d63608; F-e7b5b40d1609 | Position operations |
| F-1e7091dc86a0; F-01b37b84d143 | Handle acquisition and cleanup-like operation |
| F-9d06d181244d; F-56fd8d6069bf; F-47c987e17054; F-cb9fa74c0a71; F-cca11044abb5 | Geometry inputs and transformation |
| F-28fd04957d82 | Upper-bound refresh source |
| F-cd58c20fc91a; F-4f0f1cd223e8; F-ce99e0891f09 | Associated-record preparation |
| F-9d0ecc5f6adf; F-c0b020cd3882; F-5710a8628cba | Context, resolution, alternate lookup |
| F-151ab4bfe553; F-438680ffebfe; F-1f62160dfd52; F-ecd1af0519f9 | Formatting and extraction |
| F-0e5f1b2977d6; F-289ec4311c64 | Associated temporary acquisition and cleanup |
| F-7a3d159c02b4; F-20d91e7a468b; F-6b51e92a470d | Contract-local virtual-site IDs at offsets 0x0c, 0x24, 0x88 |

No dependency is assumed to preserve globals, own arguments, retain storage, or release memory.

## Acceptance examples

Conditional static criteria only; assume branch reachability, valid accessed storage, nonoverflowing signed arithmetic, and ordinary dependency completion.

1. **Entry registration:** If F-398e0e9a318a returns X while changing the entry registration and common registration globals to Y and Z, registration receives X, Y, Z. Those argument reads follow preparation, not invocation entry.

2. **Relative registration:** After successful acquisition and F-c5a3e2a1a1d4, registration receives zero and freshly read relative/common values. A nonzero initializer result must not replace literal zero.

3. **Current registration:** If F-a891c2749310 returns X and changes current/common registration values to Y/Z, registration receives X/Y/Z. The third argument comes from the same global used at the other two sites, but need not have the same value.

4. **Relative retrieval:** E with mode two writes zero and returns one without retrieval. Nonzero retrieval with a zero first temporary word invokes the failure action; that word alone does not reject current-entry E retrieval.

5. **Address reload and aliasing:** If current-entry application changes its destination global from X to Y, output receives Y. Movement decisions occur afterward and can observe an output write that aliases position.

6. **Captured descriptor and precedence:** Acquisition changing the descriptor global does not redirect captured-pointer stores. A precedes T for 0x8481; S precedes N for 0x848b.

7. **Position resources:** On the high-bound branch, 0x848c reads baseline then replacement after context acquisition and resolves only replacement. 0x848d reads and resolves baseline without reading replacement. The low-bound branch uses captured default and omits position arguments.

8. **Alternate timing:** A zero +0x8e0 selects the zero branch even if context acquisition changes that byte. Its resource is loaded afterward. For 0x848f, zero lookup returns zero without a direct output write.

## Uncertainties

- **documented_only — External data:** Actual values, equality relationships, declarations, resources, and storage lifetimes are unavailable.
- **documented_only — Shift:** T’s declaration leaves right-shift signedness, promotions, and some D outcomes unresolved.
- **documented_only — Record comparison:** Ordinary unsigned-16-to-signed-32 promotion permits values below 0x076c to satisfy the comparison. A bounded unsigned interval is not justified; machine-level interpretation is unverified.
- **documented_only — Read semantics:** Special access semantics are unknown. A discarded resource read is not established to be unobservable.
- **documented_only — Evaluation order:** Separate call arguments do not establish a relative read order. This does not erase the established boundary placing registration reads after initializer return.
- **documented_only — Layouts and arithmetic:** Only consumed fields are established. Signed overflow and address-wrap validity are unresolved.
- **documented_only — Failures and ownership:** Exceptions, faults, nonlocal exits, formatting interpretation, retention, release, concurrency, reentrancy, and contention behavior are unspecified. Captured addresses do not guarantee valid referents.

## Evidence

The supplied static representation establishes widths, offsets, branch precedence, direct stores, call boundaries, captures, and reloads.

At all three registration sites, it identifies the first argument, a distinct site-specific second-argument global, and a shared third-argument global. The registration argument reads follow the preceding initializer’s return; dependency-contract uncertainty does not make those inputs unknown.

Role terminology is inferred intent. Dependency bodies and external declarations were not supplied. No runtime tests have been performed.
