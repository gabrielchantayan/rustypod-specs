<!-- Trusted provenance: F-e5c47acd678a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e5c47acd678a

## Purpose

F-e5c47acd678a returns whether an input byte string is accepted by any candidate reached through an externally supplied, zero-terminated table of candidate addresses.

Matching is asymmetric. Each nonzero candidate byte accepts either the identical input byte or an input byte whose unsigned value is exactly 32 greater. After the candidate terminates, the input may contain ordinary space bytes before its terminating zero. This is a numeric comparison rule, not general case-insensitive comparison.

The function returns a 32-bit value of 1 on acceptance and 0 on encountering a zero table entry before acceptance.

The table is located through a global table-base value. That value is evaluated separately on every outer iteration, before the corresponding table-entry read. A single fixed table for the entire call is therefore a conditional interpretation requiring a stable global base, not an unconditional property of the function.

These conclusions derive from static inspection only. No runtime tests have been performed.

## Interface

### Arguments

The represented interface has four argument positions:

| Position | Represented width | Observed role |
|---|---:|---|
| First | 32 bits, signed integer representation | Used as the input byte-string address. |
| Second | 32 bits | Not used by the function body. |
| Third | 32 bits | Not used by the function body. |
| Fourth | 32 bits, unsigned | Replaced with zero-extended input bytes during comparison. Its incoming value does not affect the result or determine a memory access. |

The first argument is used for byte-addressed reads. Its integer representation does not establish an original pointer declaration or a complete pointer-provenance model.

The second and third arguments impose no observed memory-access requirements.

When the comparison needs an input byte, the fourth argument’s working value is overwritten before use. When the candidate byte is zero, comparison stops without requiring that working value. No write through the fourth argument is present.

### External state

The function uses two distinct levels of external state:

1. A global value supplying the table base for the current lookup.
2. A four-byte table entry at that base plus four times the current table index.

The table index starts at zero. Every rejected candidate advances it by one. Every outer iteration evaluates the global base again, including the iteration that encounters a terminating table entry.

Each nonzero table entry supplies a candidate address. A zero entry ends the search with result 0. The address obtained from a nonzero entry is retained for that candidate’s processing; the table entry is not reread for each candidate byte.

The supplied representation does not include the declaration of the global base storage. Its exact declared type and storage width remain unresolved. Table entries themselves are read as four-byte quantities and used as addresses under the represented 32-bit address model.

### Result

The only explicit result is a 32-bit acceptance value:

- **1:** a candidate was accepted.
- **0:** a zero table entry was reached before acceptance.

No distinct result reports invalid addresses, missing terminators, arithmetic-range exhaustion, or inaccessible memory.

### Preconditions for defined completion

All reads actually taken must access suitable readable storage, and all address calculations must remain valid under the target execution model.

This includes:

- Readable global table-base storage at every outer iteration reached.
- A readable four-byte entry at each address calculated from that iteration’s base and current index.
- Readable candidate bytes along every comparison path taken.
- Readable input bytes along every comparison and suffix-checking path taken.
- Valid index increments, multiplication by four, and address additions.
- Eventual acceptance or a reachable zero entry for normal terminating completion.

A global base that changes can redirect a later lookup. Accessibility must therefore hold for the addresses actually selected, not merely for entries in the table selected initially.

The function does not validate these conditions. An initially zero table entry avoids input and candidate-byte reads, but still requires the global base read and the first four-byte entry read.

## Behavior

### Outer lookup and candidate selection

The observable lookup order is:

1. Evaluate the global table-base value.
2. Read the four-byte entry at that base plus four times the current table index.
3. Return 0 if the entry is zero.
4. Otherwise, process the addressed candidate against the input, starting at byte offset zero.

Acceptance returns 1 immediately. Rejection advances the table index and repeats the lookup, beginning with another evaluation of the global base.

With a stable global base and stable table contents, this examines successive entries of one table in increasing index order. Without that stability, the same increasing index can be applied to different bases. Changing the base does not reset the index.

There is no longest-match selection, best-match selection, or special duplicate handling. The first candidate accepted along the actual lookup sequence ends the function.

### Byte comparison and read ordering

For each comparison offset, the function first reads the candidate byte and interprets it as an unsigned value from 0 through 255.

If that byte is nonzero, it reads the input byte at the same offset and also interprets that byte as an unsigned value from 0 through 255.

Comparison stops when either:

- The candidate byte read is zero; or
- The candidate byte is nonzero and the subsequently read input byte is zero.

Consequently, a zero candidate byte suppresses the input read during that comparison iteration. A nonzero candidate byte causes an input read even if the input byte turns out to be zero.

For a pair of nonzero bytes:

- Equal values match.
- Unequal values match only when the input value equals the candidate value plus 32.
- Any other pair records a mismatch.

A mismatch is persistent for the current candidate. Later matching bytes do not clear it.

Importantly, a mismatch does not stop byte traversal. The offset continues increasing, and later candidate and input bytes continue to be read until one of the zero-byte stopping conditions occurs. Rejection therefore cannot be used as an early-read boundary.

### Width and arithmetic of the alternate match

Candidate bytes are zero-extended to 32-bit unsigned values before the alternate-value calculation. Adding 32 does not truncate the result back to eight bits.

Accordingly:

- Candidate values 1 through 223 can have an alternate accepted input value within the byte range.
- Candidate values 224 through 255 can match only exactly, because their alternate values exceed 255.
- Candidate zero is a terminator, not a byte eligible for alternate matching.
- Input zero stops comparison before the numeric match rule is evaluated for that pair.

Under ASCII-compatible byte values, uppercase `A` accepts either `A` or `a`, but lowercase `a` does not accept uppercase `A`. The rule also applies to nonletters: `@` accepts a backtick, and `[` accepts `{`.

No locale processing, alphabetic-range restriction, or symmetric case folding is established.

### Eligibility after comparison

After comparison stops, the function first checks whether any mismatch was recorded.

If there was a mismatch, the candidate is rejected without the candidate-ending reread or input-suffix scan.

If there was no mismatch, the function rereads the candidate byte at the stopping offset. This new read must produce zero for suffix checking to proceed.

For stable candidate storage:

- Stopping because the candidate ended permits suffix checking.
- Stopping because the input ended before a nonzero candidate byte rejects the candidate.

The second candidate-byte read is an observable part of the access sequence. Its result is not replaced by the earlier value. If storage changes between reads, the later value controls this eligibility check.

### Input-suffix checking

Once the candidate-ending reread produces zero and no mismatch has been recorded, the function reads the input at the stopping offset.

It advances past consecutive bytes equal to 32, the ordinary space byte. At the first byte not equal to 32, it stops advancing and reads that same position again to test for zero.

Acceptance requires that final read to produce zero.

For stable input storage, this accepts:

- An exact-length match.
- A matching candidate followed by one or more ordinary spaces and then zero.

It rejects a suffix containing any other nonzero byte. Tabs, line feeds, carriage returns, and other whitespace-like bytes receive no special treatment. Leading spaces are not skipped.

An empty candidate reaches suffix checking at offset zero. It accepts an empty input or an input consisting entirely of ordinary spaces followed by zero.

The read that ends the space scan and the subsequent zero-check read are distinct. Stable-input interpretations depend on those reads agreeing.

### Bounds and termination

No explicit maximum table length, candidate length, input length, or suffix length is present.

The table index and byte offset are represented as signed 32-bit integers. The table displacement uses multiplication by four; string displacement uses the byte offset directly. This differs from the unsigned arithmetic used for the alternate byte match.

The supplied representation does not establish portable overflow behavior for index increments, displacement multiplication, or address additions. The contract does not assume wrapping, saturation, validation, or recovery.

Missing terminators can lead to further reads, invalid access, or failure to complete normally. There is no bounded-search fallback.

## Side effects

### External reads

The function’s read sequence includes:

1. The global table-base value on every reached outer iteration.
2. The four-byte table entry selected by that value and the current index.
3. Candidate bytes in increasing offset order.
4. Corresponding input bytes only when the candidate byte just read is nonzero.
5. A candidate-byte reread at the stopping offset, only if no mismatch was recorded.
6. Input reads for suffix scanning, only if that candidate reread is zero.
7. A repeated input read at the position where suffix scanning stopped.

A rejected candidate causes the next outer iteration to evaluate the global base anew. The input comparison offset restarts at zero for each nonzero entry.

No later table-base evaluation or table-entry read is reached after acceptance.

### Writes and ownership

No explicit writes to the global base, table, candidate storage, input storage, or other external memory appear in the supplied body.

Changes are confined to working state: the table index, byte offset, mismatch state, current candidate address, and temporary byte values. Replacing an argument’s working value is not a write through a caller-provided pointer.

No allocation, deallocation, ownership transfer, external call, logging, or persistent retention of the input or candidate address is observed. The externally supplied memory is used for reading; its ownership is not acquired or released by this function.

### Stability and external mutation

There is no explicit synchronization or whole-call snapshot mechanism.

The conventional description of matching against one fixed table requires a stable global base and stable relevant table contents. The conventional string-matching description additionally requires stable candidate and input bytes.

If the global base changes, a later lookup uses the new base with the already-advanced table index. A base change alone does not replace the candidate address already selected for current processing.

Repeated byte reads can also observe different values if the environment permits changes or read-sensitive storage. These access-order facts do not establish which concurrent mutations are valid under the target memory model.

## Dependencies

No direct calls to other functions are present. There are no callable dependencies to identify with additional opaque function IDs.

External dependencies are:

- Readable storage supplying the global table-base value.
- Readable four-byte entries reached from each evaluated base and index.
- Candidate storage selected by nonzero entries.
- Input storage reached through the first argument.
- An execution environment supporting the represented integer and address operations.

The table’s producer, initialization rules, lifetime, mutability, and owning component are not established. No assumption about an unknown callee preserving globals or managing memory is needed because the supplied body contains no calls.

## Acceptance examples

These are conditional deductions from static evidence, not executed tests. Except for the base-change example, assume a stable global base, stable readable storage, valid arithmetic, and a terminating zero table entry. Displayed strings use ASCII-compatible byte values and terminate with zero.

| Candidates in table order | Input | Result | Reason |
|---|---|---:|---|
| None | Any first-argument value | 0 | Global base and first entry are read; input is not read. |
| `CAT` | `CAT` | 1 | Exact byte matches. |
| `CAT` | `cat` | 1 | Each input byte is the candidate byte plus 32. |
| `CAT` | `CaT` | 1 | Exact and alternate matches can be mixed. |
| `cat` | `CAT` | 0 | The alternate rule is not symmetric. |
| `CAT` | `CAT` followed by three spaces | 1 | The suffix consists only of permitted spaces. |
| `CAT` | `CAT` followed by a tab | 0 | A tab is not byte 32 or zero. |
| `CAT` | ` CAT` | 0 | Leading spaces are not skipped. |
| `CAT` | `CATS` | 0 | A non-space suffix remains. |
| `CATS` | `CAT` | 0 | Input ends before the candidate. |
| Empty string | Empty string | 1 | Suffix checking immediately finds zero. |
| Empty string | Three spaces | 1 | The entire input is an allowed suffix. |
| Empty string | A tab | 0 | The suffix is not allowed. |
| `@` | Backtick | 1 | The unsigned byte values differ by 32. |
| `DOG`, then `CAT` | `cat` | 1 | The first candidate is rejected; the second accepts. |

Additional boundary and access examples:

- **High-byte arithmetic:** A candidate containing byte 255 followed by zero does not accept input byte 31 followed by zero. The alternate value is 287, not 31.
- **Zero input against a nonempty candidate:** Candidate byte 224 against input zero stops comparison and rejects the candidate under stable storage. There is no wrapped alternate match.
- **Mismatch followed by inaccessible storage:** An early mismatch does not prevent later reads. Normal rejection is not guaranteed if the continuing scan reaches inaccessible storage.
- **Acceptance before an invalid later entry:** If the first candidate accepts, later entries and later global base evaluations are not reached.
- **Empty candidate with inaccessible input:** An empty candidate still requires input-suffix reads. It does not make the input address irrelevant.
- **Redirected lookup:** Suppose index zero selects a candidate from one base and that candidate is rejected. If the global base changes before the next lookup, index one selects the entry four bytes beyond the new base—not the new base’s index-zero entry. A zero there produces result 0; a nonzero value selects another candidate, subject to the same access requirements.
- **Terminating first entry with inaccessible global storage:** An intended empty table does not avoid the prerequisite global read. No defined result is established if that read cannot be performed.

## Uncertainties

### Established static evidence

The supplied representation establishes per-iteration global base evaluation, four-byte entry reads, increasing table indexes, conditional input reads, unsigned byte comparison, nontruncated addition of 32, persistent mismatch state, conditional candidate rereading, suffix rereading, and the two explicit return values.

It also establishes the absence of explicit external writes and direct function calls in this body.

### Inferred intent

The behavior resembles recognition of predefined tokens with limited uppercase-candidate/lowercase-input tolerance and optional trailing spaces.

That interpretation does not justify replacing the numeric rule with conventional case-insensitive comparison, broad whitespace trimming, or a whole-call table snapshot.

### documented_only

The following remain unresolved:

- The global base storage’s declaration, exact width, initialization, lifetime, and update policy.
- Actual table contents, allocation extent, and ownership.
- Whether the global base or table entries can validly change during execution.
- Whether empty candidates or non-ASCII bytes occur in practice.
- Original interface declarations and calling-convention details.
- Application-level roles of the unused arguments.
- Target endianness and precise pointer-provenance rules.
- Behavior after invalid reads or arithmetic-range exhaustion.
- Validity and semantics of concurrent mutation or read-sensitive storage.
- Application-level error handling outside this function.

These unknowns are not resolved by assuming immutability, validation, bounded data, or recovery.

## Evidence

This contract is based solely on the supplied static representation of F-e5c47acd678a.

The outer lookup expression evaluates an external global base before dereferencing the indexed four-byte entry on every iteration. The representation contains no whole-call base snapshot. The byte-processing structure supports the comparison, reread, and suffix-access ordering documented above.

No external implementation inspection, runtime execution, or tests were performed. All acceptance examples are deductions under their stated conditions, not observed results.
