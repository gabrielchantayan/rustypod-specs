<!-- Trusted provenance: F-6bc2740ba7df; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6bc2740ba7df

## Purpose

F-6bc2740ba7df obtains a byte-string configuration associated with `FT2_DEBUG` and conditionally updates a table of 76 configuration values.

Its established behavior is narrower and less forgiving than a general configuration parser:

- Names are matched exactly against a 76-entry name table.
- Accepted values are single ASCII digits from `0` through `6`.
- A match at name-table index zero updates every value-table entry.
- A match at any other index updates only the corresponding entry.
- Parsing advances according to fixed byte positions, not according to a complete token grammar.
- Some ordinary-looking endings cause a read beyond the string’s first zero terminator.

The apparent intent is configuration of per-category debugging levels, including an all-categories selector. That interpretation is inferred from the lookup key and update pattern. The actual category names, the meaning of each level, and the effects of those values on other operations are not established.

## Interface

**Explicit inputs:** None.

**Return value:** None. The function has a void interface and provides no explicit success, failure, or parse-status result.

**Implicit inputs:**

- The result of calling F-f37f1ac64985 with the zero-terminated lookup key `FT2_DEBUG`.
- A global value identifying the destination value table.
- A global value identifying a name-pointer table.
- The bytes of the returned input and candidate names.

**Data layout established by the supplied representation:**

| Object | Layout |
|---|---|
| Input | Byte-addressed data, with zero bytes used as termination markers |
| Name-pointer table | 76 entries, each accessed at a four-byte stride |
| Candidate name | Byte string whose terminating zero participates in exact-match verification |
| Destination value table | 76 entries, each four bytes wide |
| Written level | A 32-bit unsigned value in the range 0–6 |
| Name index | An integer in the range 0–75 when matched; otherwise a negative sentinel |

The representation uses 32-bit integer and pointer-sized operations. The sentinel for an invalid level is the 32-bit value `0xffffffff`, interpreted as negative when checking whether the level is acceptable.

**Memory preconditions:**

- The destination base must support the writes selected by the input. A broadcast requires writable storage for all 76 four-byte entries, totaling 304 bytes.
- Name-table entries examined during matching must be readable, and each candidate name must support the byte reads required by comparison.
- Input storage must remain readable at every cursor and scan position reached by the parser. Merely providing a conventional zero-terminated string does not guarantee this condition.
- Pointer differences and indexing must be meaningful under the represented 32-bit address model.

There is no explicit validation of these memory preconditions.

## Behavior

### Lookup and state capture

The function first calls F-f37f1ac64985 with the lookup key `FT2_DEBUG`.

After that call returns, it reads and retains the destination-table base from global state. This read occurs before the null-result check. Consequently, even a null lookup result does not bypass that global read.

If the lookup result is null, the function returns without parsing or writing destination entries.

The destination base used for subsequent writes is the value captured after the dependency call. The function does not visibly refresh it between assignments. The name-table base, by contrast, is referenced during candidate lookup; the supplied representation does not establish an equivalent one-time snapshot for it.

### Current-byte handling

At each outer parsing position, the function reads the current input byte.

If that byte is zero, it returns.

If the byte is one of the following, it advances exactly one byte and repeats:

- ASCII space, `0x20`
- Horizontal tab, `0x09`
- Comma, `0x2c`
- Semicolon, `0x3b`
- Equals sign, `0x3d`

No other byte is treated as an outer separator. In particular, carriage return and line feed do not receive this treatment.

A separator is recognized only when it is the current outer byte. Once a name scan has begun, these same bytes are not delimiters.

### Scanning for a colon

For a current byte that is neither zero nor an outer separator, the function scans forward for a colon, `0x3a`, or a zero byte.

The candidate name is the complete byte range from the current position up to, but excluding, the colon. Spaces, tabs, commas, semicolons, and equals signs inside that range remain part of the candidate.

There is no trimming, case folding, escaping, quoting, or character decoding.

If the scan reaches a zero byte without finding a colon:

- No name lookup or destination write occurs for that scanned range.
- The next outer position is the byte immediately after that zero.
- The next outer iteration reads that position.

Thus, a nonempty colon-free string does not necessarily cause a clean return at its terminator. Behavior after that scan depends on storage beyond the terminator.

If a colon is encountered at the starting position, the candidate is empty. The function performs neither name lookup nor level parsing for that colon. It advances to the byte immediately after the colon and resumes the outer loop.

### Name matching

For a nonempty candidate before a colon, the function searches name-table entries in ascending index order, from zero through 75.

For each entry:

1. It obtains that entry’s candidate-name address.
2. It compares bytes from the beginning against the input name range.
3. If all bytes in the input name range agree, it reads the candidate-name byte immediately following that range.
4. The entry matches only if that following byte is zero.

Accordingly, matching is exact in both bytes and length. An input name that is merely a prefix of a longer table name does not match that entry.

The first matching table entry wins. If duplicate names exist, later duplicates are not reached for that name. This matters especially if index zero duplicates another entry: the index-zero broadcast behavior takes precedence.

Failure to match any of the 76 entries produces an invalid-name result. The function still proceeds with the same level-byte handling and cursor advancement described below.

The name length is represented using a 32-bit address difference. The comparison bound uses a signed 32-bit interpretation, while the eventual full-length equality check uses the stored length value. The evidence does not justify extending this into unrestricted large-buffer semantics. Ordinary matching descriptions require a nonnegative, representable signed 32-bit name length.

### Level decoding

For every nonempty name followed by a colon, the function reads the byte immediately after the colon.

If that byte is zero:

- The level remains invalid.
- No destination update occurs.
- The next outer position is one byte beyond that zero.

If that byte is nonzero:

- The function subtracts ASCII `0`, `0x30`, using 32-bit unsigned arithmetic.
- The result is accepted only if it is at most six.
- Every other result is replaced by the invalid-level sentinel.
- The next outer position is three bytes after the colon.

This admits exactly ASCII `0` through `6`. For bytes below ASCII `0`, subtraction wraps in the 32-bit unsigned domain and produces a value greater than six, so those bytes are rejected. Bytes above ASCII `6` are also rejected.

The parser does not accumulate multiple digits or check the byte following the level as a delimiter.

The fixed advance is significant: for a nonzero level byte, the byte two positions after the colon is skipped unconditionally. It can be a separator, another digit, a name character, or a zero terminator.

### Destination updates

An update occurs only when both the name and level are valid.

For a matched index from one through 75, the function performs one four-byte write of the decoded level to that index.

For matched index zero, it writes the level to all 76 entries in ascending index order, beginning with index zero and ending with index 75.

The broadcast is a sequence of writes, not an established atomic transaction. No rollback or synchronization is visible.

Input assignments are processed in encounter order. Therefore:

- A later valid assignment to an ordinary index replaces an earlier value at that index.
- A later broadcast replaces all earlier assignments.
- An ordinary assignment after a broadcast changes only its own entry.
- Invalid names and invalid levels cause no destination write for that occurrence.

For a nonempty name, all name matching and level decoding for that occurrence precede its destination writes. The next outer input-byte read follows those writes. If destination memory aliases input or name storage, those writes can therefore affect later parsing or lookup behavior; the function does not prevent such aliasing.

## Side effects

**Established effects:**

- One call to F-f37f1ac64985.
- A destination-base global read after that call.
- Input-byte reads and, when applicable, name-table and candidate-name reads.
- Zero, one, or 76 destination writes per parsed nonempty colon occurrence.

The function contains no explicit input-buffer writes, allocation, release, logging, or error reporting. This does not establish that F-f37f1ac64985 lacks such effects.

No ownership transfer is visible. The function uses the returned address without visibly releasing it. Whether the address denotes borrowed storage, owned storage, static storage, or another lifetime arrangement is **documented_only**.

Malformed entries are not reported through a status value. Some are ignored; others influence subsequent parsing through the fixed cursor advance. Inaccessible memory, invalid table pointers, concurrent mutation, and out-of-range address arithmetic have no locally established recovery behavior.

## Dependencies

### F-f37f1ac64985

**Observed use:** Called once with the key `FT2_DEBUG`; its result is treated as either null or an address from which bytes can be read.

**Established dependency contract at this call site:**

- A null result causes the caller to return after capturing the destination base.
- A nonnull result becomes the initial parsing position.

**Unknowns — documented_only:**

- How the key is resolved.
- Whether the dependency accesses an environment, registry, configuration store, or another source.
- Whether it allocates, retains, frees, or mutates storage.
- Whether it changes either global table reference or other global state.
- Whether it reports errors separately.
- What lifetime, accessibility, or termination guarantees accompany a nonnull result.

Because destination-base capture occurs after the call, any dependency-induced change to that global before return affects the base subsequently used by this function.

No additional function calls are visible in the supplied body.

## Acceptance examples

These examples are conditional on valid memory, stable nonaliasing tables, and the specified hypothetical table contents. They are static consequences, not runtime test results.

For examples using `alpha`, assume its exact name occurs only at index 5. For `all`, assume its exact name occurs at index zero. A “final zero” means an actual zero byte after the displayed text.

| Input or condition | Required observable result |
|---|---|
| Dependency returns null | Destination base is read; no parsing or destination writes occur. |
| Input begins with a zero byte | Immediate parser return; no destination writes. |
| Input is ` ,;=` followed by a tab and a final zero | Each listed separator is skipped; no destination writes. |
| `alpha:3;` followed by a final zero | Index 5 receives the 32-bit value 3. The semicolon is skipped by the post-level advance, and the next read sees the final zero. |
| `all:2;` followed by a final zero | All 76 entries receive 2, in ascending index order. |
| `all:2;alpha:6;` followed by a final zero | All entries are first written as 2; index 5 is subsequently written as 6. |
| `alpha:6;all:2;` followed by a final zero | Index 5 is first written as 6, then every entry is written as 2. |
| `alpha:7;` followed by a final zero | No destination update: the name matches but the level is invalid. |
| `alpha:/;` followed by a final zero | No destination update: unsigned subtraction rejects `/`, which is below `0`. |
| An unknown name followed by `:4;` and a final zero | No destination update; level handling and cursor advancement still occur. |
| `alph:3;` followed by a final zero, with no exact `alph` entry | No update, even though `alph` prefixes `alpha`. |
| `:alpha:3;` followed by a final zero | The leading empty-name colon is skipped. The subsequent `alpha:3` assignment updates index 5. |
| `alpha:36;` followed by a final zero | Index 5 receives 3. The `6` is skipped, not combined into 36; the semicolon is then processed as an outer separator. |
| `alpha:3X` followed by a final zero | Index 5 receives 3. `X` is skipped without delimiter validation. |
| `alpha,other:3;` followed by a final zero | The candidate name is `alpha,other`. It updates nothing unless that complete byte string is a table name. |

Boundary examples require explicit storage assumptions:

- For `alpha:3` followed immediately by a zero terminator, index 5 is written as 3, but the parser skips that terminator and next reads one byte beyond it. A clean return cannot be asserted from the conventional string alone.
- For `alpha:` followed immediately by zero, no value is written. The next outer read is one byte beyond that zero.
- For a nonempty colon-free input followed by zero, the scan reaches the terminator and the next outer read is beyond it.
- If two readable zero bytes follow a colon-free range, the scan reaches the first, and the next outer iteration reads the second and returns.
- A colon immediately followed by zero is different when the colon itself is the outer starting byte: the empty-name path advances to that zero, which is then read normally and causes return.

These distinctions are required behavior of the supplied control flow, not recommended formatting rules.

## Uncertainties

**documented_only — actual table contents.** The name strings, category meanings, and initial destination values are not supplied. Examples cannot establish real accepted names.

**documented_only — intended grammar.** The fixed extra-byte skip suggests an expectation of a separator after each level, but the implementation does not require one. This inferred expectation must not replace the actual advancement behavior.

**documented_only — safe extent of returned storage.** The caller’s read-ahead behavior is established; whether the dependency supplies enough backing storage is not.

**documented_only — concurrency and observability.** There is no visible locking or transactional mechanism. The supplied representation does not establish atomicity, thread safety, or guarantees for concurrent readers and writers.

**documented_only — extreme sizes and invalid addresses.** The source representation includes signed 32-bit comparisons and address conversions. Behavior for oversized spans, address wraparound, inaccessible memory, or invalid alignment is not established as a portable contract.

**documented_only — exact physical access count.** Logical access order is described above. The supplied representation is not sufficient to guarantee a particular count of machine loads for repeated byte expressions or global references.

**Inferred intent:** The key and broadcast entry suggest debug-level configuration. No claim is made that level zero disables debugging, that larger levels increase verbosity, or that index zero has any externally meaningful name.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the supplied opaque dependency mapping.

Established evidence includes the single dependency call, post-call destination-base capture, null check, five outer separator bytes, colon search, 76-entry exact-name search, single-byte level conversion, signed validity checks, four-byte destination stride, ascending broadcast order, and cursor advancement past skipped bytes and some zero terminators.

No runtime tests have been performed. No external dependency implementation, table contents, ownership documentation, compiled execution trace, or platform ABI documentation was supplied. The acceptance examples are conditional deductions from the visible behavior, not observed executions.
