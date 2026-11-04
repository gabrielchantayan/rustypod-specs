<!-- Trusted provenance: F-3ec7d07fef90; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3ec7d07fef90

## Purpose

F-3ec7d07fef90 examines two character sequences using a restart-at-the-next-position search. Its established direct behavior is to compare the second sequence against successive starting positions in the first sequence, stopping normally when it encounters the second sequence’s zero terminator.

If the first sequence ends while the second sequence still requires a nonzero character, the function invokes an unresolved indirect dependency with the integer argument zero. If that invocation returns normally, this function then returns.

The inferred intent is a substring-presence check with a separate action for absence. That interpretation does not establish the purpose of the indirect dependency, and the interface does not return a match location or a Boolean result.

## Interface

**Inputs**

- **First sequence:** a character pointer designating the sequence searched.
- **Second sequence:** a character pointer designating the sequence sought.

Both inputs are used as addresses of individual character-sized elements. Comparisons are exact character-value comparisons, not locale-aware comparisons or case folding.

The directly visible interface has no length arguments, destination buffer, output pointer, or explicit context object. Zero-valued characters control termination.

**Output**

The function has no declared return value. Normal return alone does not distinguish:

- completion after encountering the second sequence’s terminator; and
- return after the unresolved indirect dependency has itself returned.

No match offset or pointer is exposed through a direct output.

**Widths and representation**

Each explicit data read accesses one character-sized storage unit. Pointer advancement is by one such unit, not by a word or by a character encoding’s logical character length.

The supplied evidence does not independently establish the number of bits in a character-sized unit, pointer width, pointer representation, or execution ABI. On a target with eight-bit character units, these are byte reads. An eight-bit target is not a prerequisite that can be established from the provided fragment alone.

Character signedness does not change the visible decisions for ordinary valid reads: the decisions use equality between character values and comparison with zero. The value one is not a wildcard or a special terminator.

**Preconditions**

For defined direct execution:

- Both input addresses must permit the initial character read.
- Subsequent characters reached by the search must remain readable.
- Address advancement and candidate-position advancement must remain valid for the underlying objects.
- Input storage must remain alive throughout the accesses that use it.

A conventional sufficient input condition is that both pointers designate readable, zero-terminated character sequences. The actual required readable extent depends on the path taken. For example, an empty second sequence requires only the initial first-sequence read, not traversal of the first sequence to its terminator.

There is no visible null-pointer check, length bound, or storage-lifetime check.

## Behavior

### Comparison rule

For each comparison position, the function obtains a character from the first sequence and then a character from the second sequence.

The comparison continues to the next position only when:

- the character obtained from the first sequence is nonzero; and
- both obtained character values are equal.

Thus, two nonzero equal characters extend the current candidate match. Two zero characters do not extend it: they leave the comparison phase and trigger termination handling.

Although the visible expression mentions the character value one, that does not introduce a separate accepted case. A first-sequence character of value one continues the comparison only when the second-sequence character also has value one.

### Candidate search order

The first candidate begins at the initial first-sequence address. The second sequence is compared from its initial address.

When a comparison fails with both obtained characters nonzero, the current candidate is rejected. The next candidate begins exactly one character after the previous candidate’s start. Comparison of the second sequence restarts at its beginning.

The search therefore considers candidate starts in increasing address order, one character apart. It does not skip the already-compared prefix length after a partial match fails.

Consequences include:

- Potential matches may overlap.
- Previously read characters may be read again.
- The same second-sequence prefix may be read repeatedly.
- There is no visible preprocessing, prefix table, allocation, or search acceleration.

### Completion when the second sequence ends

After leaving the comparison phase, the function first checks whether the character obtained from the second sequence is zero.

If it is zero, the function returns normally without invoking the unresolved indirect dependency.

This rule applies regardless of the character obtained from the first sequence at that position. Therefore:

- A second sequence whose nonzero characters match a prefix of the remaining first sequence is accepted.
- The first sequence need not end where the second sequence ends.
- A match ending exactly at the first sequence’s terminator is accepted.
- An empty second sequence is accepted after the function reads one character from each input.

The last point is important: an empty second sequence does not make the first input unused. The initial first-sequence read still occurs before normal completion.

### Exhaustion of the first sequence

If the obtained second-sequence character is nonzero, the function next considers whether the obtained first-sequence character is zero.

If the first-sequence character is zero, the function invokes the unresolved indirect dependency with the integer value zero.

No later candidate is attempted after this invocation returns. Instead, the function returns immediately.

The supplied evidence does not establish whether the dependency:

- returns normally;
- terminates execution;
- transfers control elsewhere;
- raises an error through some external mechanism; or
- performs other effects.

Consequently, reaching this path establishes an attempted indirect invocation, not a particular process-level failure outcome.

### Access and decision order

The source-level sequence for each comparison position is:

1. Obtain the first-sequence character.
2. Obtain the second-sequence character.
3. Determine whether the pair extends the candidate match.
4. Advance both comparison cursors by one character-sized unit.
5. Either continue comparison or handle the stopping pair.

When comparison stops, the second-sequence zero check takes precedence over the first-sequence zero check.

This precedence makes a pair of zero characters a normal-completion case, not an indirect-invocation case.

Cursor advancement occurs before either stopping decision. It is local pointer-state movement, not a write through either input pointer. The advanced cursors are not dereferenced again on a normal-completion path or after the indirect invocation returns.

These are source-level access and decision relationships. The fragment alone does not establish a hardware transaction trace, access atomicity, or compiler treatment of externally changing memory.

## Side effects

The function contains no explicit writes to either input sequence. It also contains no explicit allocation, release, global-memory write, output operation, or persistent state update.

Its direct state changes are confined to local comparison cursors, a candidate-start cursor, and temporary character and comparison values.

Input ownership is not visibly transferred. There is no explicit operation that acquires, retains, copies into owned storage, or releases either input.

The unresolved indirect dependency is a separate side-effect boundary. Its invocation is visible, but its implementation and contract are not. In particular:

- Do not assume that it preserves global state.
- Do not assume that it leaves either input allocation unchanged.
- Do not assume that it owns or releases any memory.
- Do not assume that it returns.

Only the integer zero is explicitly supplied to that invocation. Neither input pointer is explicitly passed as an argument.

The input pointers may overlap or designate the same readable storage; the direct comparison logic introduces no input-write conflict. Effects caused by concurrent modification, special memory mappings, or the unresolved dependency are not established.

## Dependencies

**F-3ec7d07fef90** has one visible call site: an indirect invocation reached when the first sequence supplies zero and the second sequence supplies a nonzero character at the stopping comparison.

No opaque function ID, concrete target, or callable signature for that dependency was supplied. Its identity is therefore **documented_only**; this contract does not invent an ID or assign it the behavior of a familiar error-handling routine.

Established call-site properties:

- The call is conditional.
- One explicit argument has integer value zero.
- The call’s result is not used as a return value of F-3ec7d07fef90.
- If control returns normally to the call site, F-3ec7d07fef90 returns next.

The argument’s effective ABI width and the target’s interpretation of it are unknown. The source-level literal alone does not establish a parameter type for an unresolved target.

## Acceptance examples

The following examples are conditional static acceptance criteria, not executed tests. Unless stated otherwise, displayed strings represent readable character sequences with a trailing zero.

1. **Match at the beginning**

   First sequence: `alpha-tail`. Second sequence: `alpha`.

   The function compares the five matching nonzero characters. It then reads the hyphen from the first sequence and zero from the second sequence. It returns without invoking the indirect dependency.

2. **Match after rejected candidates**

   First sequence: `xxcatyy`. Second sequence: `cat`.

   Candidate starts at the first and second characters are rejected. The candidate beginning at `c` matches the second sequence’s nonzero characters. Reading the second sequence’s terminator causes normal return.

3. **Overlapping candidate starts**

   First sequence: `aaaab`. Second sequence: `aaab`.

   The initial candidate fails after a shared prefix. The next candidate starts at the second character of the first sequence, not after that shared prefix. That candidate completes normally.

4. **Match ending at both terminators**

   First sequence: `cat`. Second sequence: `cat`.

   After the three nonzero matches, the stopping comparison obtains zero from both sequences. The second-sequence zero check wins, so there is no indirect invocation.

5. **First sequence ends during a candidate**

   First sequence: `ab`. Second sequence: `abc`.

   After matching `a` and `b`, the comparison obtains zero from the first sequence and `c` from the second. The indirect dependency is invoked with zero. Normal return from F-3ec7d07fef90 is conditional on that dependency returning.

6. **Empty second sequence**

   First sequence: `anything`. Second sequence: empty.

   One first-sequence character and the second sequence’s zero are read. The function returns normally. It does not scan the rest of the first sequence.

7. **Both sequences empty**

   Both initial reads obtain zero. The function returns without invoking the indirect dependency.

8. **Empty first sequence, nonempty second sequence**

   First sequence: empty. Second sequence: `x`.

   The initial reads obtain zero and `x`, respectively. The indirect dependency is invoked with zero.

9. **Character value one**

   A first-sequence character with numeric value one does not match a second-sequence character with numeric value two. It is an ordinary mismatch. Matching nonzero characters both having value one extend a candidate normally.

10. **Embedded zero**

    First-sequence storage contains `ab`, then zero, then further readable characters. Second sequence: `bc`.

    The function does not search past the first sequence’s zero for a later candidate. It eventually reaches the indirect-invocation path.

## Uncertainties

The following remain **documented_only**:

- The indirect dependency’s identity, signature, effects, return behavior, and intended meaning.
- Target character-unit width, pointer width, and ABI details.
- Behavior for invalid addresses, insufficient readable storage, or invalid pointer arithmetic.
- Behavior under concurrent mutation or externally changing memory.
- Machine-level load ordering, atomicity, and speculative access behavior.
- Any broader caller convention that interprets return as success or treats the indirect invocation as a particular error.

Substring checking is inferred intent. Exact character matching, candidate restart order, termination precedence, and the conditional indirect invocation are established by the supplied static evidence.

There is no visible defensive handling for malformed or unterminated input. This contract does not replace that absence with an assumed validation policy.

## Evidence

This contract is based solely on the supplied static function fragment and the supplied mapping to F-3ec7d07fef90.

The fragment establishes:

- Two character-pointer inputs and no declared return value.
- Character-at-a-time reads from both inputs.
- Continued comparison only for equal, nonzero first-sequence characters.
- One-character candidate-start advancement after a nonterminal mismatch.
- Restart of the second-sequence comparison at each candidate.
- Normal completion when the obtained second-sequence character is zero.
- Conditional indirect invocation with zero when the obtained first-sequence character is zero and the second is nonzero.
- Immediate return if that invocation returns.
- No explicit input-memory writes or direct memory-management operations.

No implementation access, runtime tests, or machine-level execution observations were used. The acceptance examples are deductions from the visible control flow, not reports of observed execution.
