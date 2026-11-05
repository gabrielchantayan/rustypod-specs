<!-- Trusted provenance: F-ab03ddc82f92; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ab03ddc82f92

## Purpose

F-ab03ddc82f92 performs a table-directed lookup for a byte sequence and returns a one-byte result associated with the first matching candidate encountered.

For eligible input lengths, it derives a value from the input’s first and last bytes, passes that value to F-870c4fcf7295, and uses a secondary result from that dependency to select a candidate chain. Each candidate is screened by its stored length before F-abcb7c332db0 is called to determine whether it matches. A zero comparison result selects the candidate’s result byte.

The function returns `0x17` when the input length is ineligible or the candidate chain terminates without a match. This value is not necessarily reserved: a successful candidate can also contain `0x17`.

The lookup interpretation is inferred intent. The byte accesses, arithmetic, branch conditions, dependency calls, and return selection described below are established by the supplied static evidence.

## Interface

### Inputs

- **Input buffer:** an address identifying a sequence of bytes. Each byte is eight bits.
- **Input length:** a 32-bit unsigned value.

The entry condition interprets the length as a signed 32-bit value and requires it to be greater than one. Consequently:

| Unsigned input length | Entry behavior |
|---|---|
| `0` or `1` | Return `0x17` immediately |
| `2` through `0x7FFFFFFF` | Enter the lookup path |
| `0x80000000` through `0xFFFFFFFF` | Return `0x17` immediately |

For an eligible length, the function directly reads the first byte and the byte at offset length minus one. It may subsequently pass the buffer and the full unsigned length to F-abcb7c332db0.

A usable buffer must therefore support those endpoint reads and any accesses required by the comparison dependency. The full buffer-access contract of that dependency is unknown. For an ineligible length, this function does not dereference the input buffer.

### Output

The output is one byte:

- The selected candidate’s stored result byte, if a candidate passes both the length screen and the comparison.
- `0x17` otherwise.

No separate success indicator, matched index, or error status is returned.

### External state

The function depends on externally supplied addresses for:

1. A byte-mapping table.
2. A chain-head table.
3. A shared candidate-metadata region.
4. A base address for candidate comparison data.
5. A candidate-result table.

The supplied evidence does not establish how these addresses or their contents are initialized, validated, synchronized, or owned.

## Behavior

### 1. Length gate

The signed interpretation of the 32-bit length is tested before any input-byte read, table lookup, or dependency call shown in this function.

If that signed value is at most one, execution returns `0x17`. There is no fallback lookup for these lengths.

This gate is not an unsigned “length at least two” test: unsigned values with the high bit set take the immediate-return path.

### 2. Endpoint-derived selector

For an eligible length, the function reads:

- The input byte at offset zero.
- The input byte at offset length minus one.
- The byte-mapping entry indexed by each of those byte values.

Each input byte is used as an unsigned index in the range 0–255. Each mapping result is also an unsigned byte.

The value supplied as the first argument to F-870c4fcf7295 is the bitwise exclusive-or of:

- Three times the mapped last byte.
- The mapped first byte shifted left by two bits.
- The complete 32-bit input length.

The mapping values are widened before multiplication or shifting. The first product is at most 765; the shifted value is at most 1020. Neither operation overflows a 32-bit unsigned value.

The second argument to F-870c4fcf7295 is `0x7F`.

The supplied expression does not establish the relative evaluation order of the first-byte and last-byte operands. In particular, it does not justify promising that either endpoint, or either mapping entry, is always read first.

The ordinary return value of F-870c4fcf7295 is not used in the supplied representation. A secondary register result is instead used as an offset into the chain-head table. Its meaning, range, and signedness are not established here.

### 3. Capture lookup bases and read the chain head

After F-870c4fcf7295 returns, the function obtains the addresses needed for candidate traversal.

The supplied sequence establishes these loads before the initial chain-head byte is read:

1. Load the candidate comparison-data base.
2. Load the candidate-metadata address for length-byte accesses.
3. Load that metadata address again and add 117 bytes for two-byte offset accesses.
4. Load that metadata address again and subtract 116 bytes for successor-byte accesses.
5. Read a chain-head byte using the chain-head table address and the dependency’s secondary result.

These are three separate loads of the candidate-metadata address, not a single demonstrated snapshot. If the external address storage changes between loads, the length, offset, and successor views need not share a common base.

The comparison-data base and the three derived metadata addresses are then retained across candidate comparisons. Later dependency calls do not cause those address values to be refreshed in the shown behavior.

The chain-head byte is a one-based candidate reference:

- `0` means the chain is empty.
- A value from `1` through `255` selects candidate index one less than that byte.

Thus the candidate index used by this function is always in the range 0–254.

### 4. Candidate metadata layout

For candidate index \(k\), the function uses the following relative locations:

| View | Location | Access |
|---|---|---|
| Length view | Captured length base plus \(k\) bytes | One unsigned byte |
| Offset view | Captured offset base plus \(2k\) bytes | One unsigned 16-bit value |
| Successor view | Captured successor base plus \(k\) bytes | One unsigned byte |
| Result table | Result-table base plus \(k\) bytes | One returned byte |

When the three metadata-address loads agree, these correspond to:

- Length at metadata offset \(k\).
- Comparison-data offset at metadata offset \(117 + 2k\).
- Successor at metadata offset \(k - 116\).

These are relative views, not evidence of separately allocated arrays. The negative displacement used for successor accesses requires accessible storage before the metadata address for some candidate indices.

The offset access is a two-byte typed read. Its byte order and alignment behavior depend on the target execution environment; the supplied body alone does not establish them.

### 5. Length screening and comparison

For each candidate, the function first reads its stored one-byte length and compares that unsigned byte with the full 32-bit input length.

There is no truncation of the input length to eight bits for this comparison. Therefore:

- Lengths from 2 through 255 can pass the screen.
- Eligible lengths from 256 through `0x7FFFFFFF` cannot pass any candidate’s length screen.
- Those larger eligible lengths still perform the endpoint reads, selector call, chain-head read, length reads, and successor traversal.

Only if the candidate length equals the input length does the function read the candidate’s 16-bit comparison-data offset.

That offset is zero-extended and added to the previously captured comparison-data base. F-abcb7c332db0 is then called with:

1. The resulting candidate-data address.
2. The original input-buffer address.
3. The original unsigned 32-bit input length.

A return value equal to zero means the candidate is accepted by this function. Every nonzero return value means it is rejected. The function does not otherwise interpret that result.

The evidence does not independently establish that F-abcb7c332db0 performs ordinary bytewise equality, nor does it establish which bytes it reads, whether it writes memory, or whether it changes external state.

### 6. Success and continuation

On a zero comparison result, the function reads the result-table address at that point, reads the byte at the selected candidate index, and returns that byte immediately.

The result-table address is not captured before traversal. A change to its external address storage during a dependency call can therefore affect the result read.

After a length mismatch or a nonzero comparison result, the function reads the selected candidate’s successor byte using the retained successor address.

The successor uses the same one-based representation as the chain head:

- `0` terminates traversal and causes return of `0x17`.
- A nonzero byte selects the next candidate index after subtracting one.

There is no visited-set, cycle detection, or explicit iteration bound. A chain that repeatedly supplies nonzero successor references can continue indefinitely, subject to memory behavior and dependency behavior.

The first candidate in traversal order whose stored length matches and whose comparison returns zero determines the result. Later candidates are not examined.

## Side effects

### Direct effects established here

The function:

- Reads the input endpoints on the eligible-length path.
- Reads mapping, chain-head, metadata, and result storage as described above.
- Calls F-870c4fcf7295 once on that path.
- Calls F-abcb7c332db0 once for each encountered candidate whose stored length matches.
- Performs no explicit memory writes, allocations, or releases in the supplied body.

No write through the input-buffer address is directly shown. This does not prove that the input remains unchanged, because it is passed to F-abcb7c332db0.

### Dependency effects and state visibility

Neither dependency is established to be pure, to preserve globals, or to leave memory unchanged.

This matters because:

- State changed by F-870c4fcf7295 can affect the bases loaded afterward and the chain-head read.
- Candidate comparisons can alter memory subsequently read through retained metadata addresses.
- A comparison can affect the result-table address or result byte read on success.
- A rejected comparison can affect the successor byte read immediately afterward.

Retaining an address does not freeze the contents at that address.

No ownership transfer is explicitly represented by the caller. The dependencies’ ownership or retention behavior is unknown; this function supplies no visible cleanup or compensating release.

### Errors and exceptional execution

The function has no explicit handling for inaccessible memory, unsuitable alignment, invalid external addresses, invalid secondary selector results, or dependency failures.

`0x17` is an ordinary fallback result, not an established encoding for such failures. Faulting accesses, nonreturning dependencies, or cyclic traversal may prevent any result from being returned.

## Dependencies

### F-870c4fcf7295

**Invocation:** exactly once for an eligible input length, before candidate-base capture.

**Arguments:** the 32-bit endpoint-derived exclusive-or value and `0x7F`.

**Consumed result:** a secondary register value used to address a chain-head byte.

**Unknowns:** the secondary-result contract, valid output range, ordinary return value, memory effects, global effects, and ownership behavior.

The constant `0x7F` does not by itself establish that the secondary result is bounded by 127, masked to seven bits, or computed by a remainder operation.

### F-abcb7c332db0

**Invocation:** only for encountered candidates whose one-byte stored length equals the full input length.

**Arguments:** candidate-data address, input-buffer address, and full unsigned length.

**Consumed result:** zero accepts the candidate; nonzero rejects it.

**Unknowns:** comparison semantics, read extent, write effects, retained references, ownership behavior, and global effects.

## Acceptance examples

These examples are conditional behavioral criteria derived from static evidence, not runtime test results.

1. **Ineligible length without dereference.**  
   With length `1`, the result is `0x17`, with no input read or dependency call shown by this function. The same applies to length `0x80000000`, despite its large unsigned value.

2. **Endpoint transformation.**  
   With length `5`, mapped first byte `2`, and mapped last byte `3`, the first argument to F-870c4fcf7295 is `4`: \(9 \mathbin{\mathrm{XOR}} 8 \mathbin{\mathrm{XOR}} 5\). Its second argument is `0x7F`.

3. **Empty chain.**  
   If the selected chain-head byte is zero, the function returns `0x17` without a candidate-length read or comparison call.

4. **One-based candidate selection.**  
   A chain-head byte of `4` selects candidate index `3`. If the metadata-address loads agree, its length is at metadata offset `3`, its two-byte data offset begins at offset `123`, and its successor is at offset `-113`.

5. **Length mismatch bypasses comparison.**  
   For input length `6` and a selected candidate length of `5`, no comparison-data offset is read for that candidate and F-abcb7c332db0 is not called. The successor is read next.

6. **No low-byte length matching.**  
   For eligible input length `258`, a candidate length byte of `2` does not match. Even though their low bytes agree, no comparison occurs for that candidate.

7. **Accepted candidate.**  
   If a candidate’s length matches and F-abcb7c332db0 returns zero, the function returns that candidate’s result byte without reading its successor. If the stored result is `0x17`, the returned value remains `0x17`.

8. **Rejected candidate and continuation.**  
   If comparison returns a nonzero value and the subsequent successor byte is `2`, candidate index `1` is examined next. If that successor byte is zero instead, the function returns `0x17`.

9. **Cyclic traversal.**  
   If a rejected or length-mismatched candidate continually points back to itself, and no relevant memory changes occur, the function does not reach its fallback return. No finite traversal guarantee applies.

## Uncertainties

The following are **documented_only**:

- The origin, allocation sizes, initialization, and intended invariants of all external tables.
- The validity and range of the selector dependency’s secondary result.
- The semantics and side effects of both dependencies beyond how this caller uses them.
- The target’s byte order, alignment requirements, and invalid-access behavior.
- Whether external state can change concurrently or through reentrant execution.
- Whether candidate chains are guaranteed to terminate.
- Whether `0x17` has a domain-specific meaning or is reserved by data conventions.
- Ownership and lifetime requirements imposed by the dependencies.

Address calculations are represented using 32-bit quantities. No explicit overflow checks are shown. The supplied evidence does not justify treating wrapped or otherwise invalid addresses as valid storage.

## Evidence

**Established static evidence:** the signed length gate; endpoint-based argument construction; dependency-call conditions; post-selector base loads; unsigned byte and 16-bit metadata reads; one-based chain encoding; full-width length comparison; zero-result acceptance; success-time result-table access; and fallback value `0x17`.

**Inferred intent:** endpoint-directed candidate lookup, length filtering, and comparison against stored candidate data.

**Not established:** table contents, dependency internals, valid-address guarantees, concurrency guarantees, or runtime outcomes.

No runtime tests have been performed.
