<!-- Trusted provenance: F-6fd2e8501aef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6fd2e8501aef

## Purpose

F-6fd2e8501aef performs an ordered sequence of dependency calls and direct memory accesses. It accepts one explicit 32-bit value, mutates byte-, halfword-, and word-sized storage, captures and reloads global state, exposes stack storage to a dependency, and finally calls address zero.

No application-level purpose is established. Scaling and repeated offsets do not establish allocation, object construction, serialization, cleanup, or service dispatch.

This contract distinguishes:

- **Established static evidence:** represented calls, arguments, access widths, captured values, and ordering.
- **Conditional consequences:** behavior if the represented operations execute with the stated widths and preceding operations complete.
- **documented_only unknowns:** unavailable dependency contracts, execution semantics, memory-map properties, and calling-convention details.

The ordering below is established by the supplied representation. Whether that representation accurately describes executable control flow remains unresolved.

## Interface

### Explicit and implicit inputs

The explicit input is an uninterpreted 32-bit value. The first dependency receives four represented arguments: the input, the input again, zero, and the input again.

Additional execution inputs include:

- An incoming return-link value used to derive the first call target by adding 4094.
- An incoming 32-bit register observation used by the first direct word store.
- Ordinary dependency results, including several represented 64-bit results.
- Additional register observations attached to particular call boundaries.
- Current contents of computed addresses, two labeled global locations, and a three-byte region at absolute address 9.
- An address within this invocation’s stack storage.

The association of additional register observations with dependency outputs is represented statically but is not independently ABI-verified.

There is no explicit return value or represented success indicator. Normal return requires every preceding operation to complete and the final address-zero call to return.

### Widths and access requirements

The explicit input, ordinary scalar results, represented addresses, and most arithmetic are 32 bits. Paired results are 64 bits, split into low and high 32-bit portions. Direct accesses are 8, 16, or 32 bits as specified below.

Unsigned byte captures range from 0 through 255. Byte stores retain the low eight bits; halfword stores retain the low sixteen bits. Reading a word and truncating it for a byte store still requires a word-sized read.

Arithmetic descriptions use 32-bit bit-vector consequences where applicable. Unsigned right shifts are logical. Some operands are represented as signed integers or characters; overflow, left-shift behavior, and address arithmetic for those expressions require target-level confirmation. This contract does not silently substitute portable signed-language semantics.

Successful execution requires each accessed region to permit its specified access width and each reached target to be callable. No such validity is established. Alignment, endianness, storage extents, and low-address mappings are **documented_only**.

### Contract notation

The following names are document-local observation labels, not source identifiers:

- \(r_n\): the last represented result of dependency family F-D\(n\).
- \(s_n\): the second call’s result where a family uses it.
- \(\ell_n,h_n\): the low and high halves when \(r_n\) is paired.
- \(a_0\) through \(a_{21}\): additional register observations identified below.
- **G1** and **G2**: the first and second labeled global storage locations.

A **selected byte update through address \(p\)** means an unsigned byte read at \(p\), followed by a byte write at the numeric value of that captured byte plus 17. Its destination is therefore between addresses 17 and 272. Each separately described selected update performs a fresh selector read unless an explicit saved selector is specified.

## Behavior

### Call-boundary conventions

The body is represented as straight-line execution: no local branch, loop, retry, or early error return appears. Each subsequent effect depends on preceding calls returning and memory accesses completing.

To preserve every represented call boundary without assigning unavailable semantic identities, the ordered account uses opaque dependency families:

- An **acquisition family** has five calls. F-D\(n\)-1 receives the stated arguments. F-D\(n\)-2 through F-D\(n\)-5 then execute, each with no represented arguments. The fifth supplies \(r_n\).
- A **relay family** has seven calls. The first receives the stated arguments; the second has no represented arguments and supplies \(s_n\); the third receives \(s_n\), unless explicitly given different arguments. Four further calls, numbered 4 through 7, have no represented arguments and finish with \(r_n\).
- A **sampling family** has four calls. The first receives the stated arguments; the second has no represented arguments. The described sample or captured result is then passed to the third call. The fourth has no represented arguments and supplies \(r_n\).
- A **double-relay family** has nine calls. Calls 2 and 4 have no represented arguments; call 3 receives call 2’s result, and call 5 receives call 4’s result. Calls 6 through 9 have no represented arguments. Call 1 receives the stated arguments.

Unless otherwise stated, a family below is an acquisition family. Every numbered call is a distinct occurrence and observable boundary; the numbering does not assert distinct executable targets.

“No represented arguments” does not establish an empty machine-level parameter list. No call is assumed pure, terminating, or preserving memory or globals.

### Entry and initial word construction

F-D01 receives the four entry arguments specified in the interface. After its four subsequent calls, a word store at \(4r_1+20\) writes the incoming implicit 32-bit register observation.

F-D02 receives \(4r_1\). Its result establishes a saved scale \(4r_2\), which is written as a word at \(16r_2+20\). F-D03 receives that saved scale; after its calls, a word at \(4r_3+20\) receives the same saved scale.

F-D04 receives \(4r_3\). A word at \(16r_4+20\) then receives \(4r_4\). F-D05 receives \(4r_4\) twice and is followed by a zero word store at \(4r_5+20\).

F-D06 receives \(4r_5\). Its result drives a selected byte update through
\[
((r_6\mathbin{\&}\texttt{0x3fffffff})\gg16)+1,
\]
writing the low byte of \(4r_6\).

These stores distinguish destinations scaled by four from destinations scaled by sixteen. They cannot be normalized into one field-access convention.

### First selected updates and samples

F-D07 is a relay family whose first call receives \(4r_6\), whose second has no represented arguments, and whose third receives zero rather than the second result. After its remaining four calls, a selected update through \(((r_6\mathbin{\&}\texttt{0x3fffffff})\gg17)+1\) writes zero.

F-D08 receives \(4r_7\). F-D09 is a sampling family beginning with \(4r_8\) and zero. After its second call, an unsigned byte is captured from \(8r_6\); call 3 receives that byte in all four positions. Call 4 supplies auxiliary observation \(a_0\). A selected update through \(((r_6\mathbin{\&}\texttt{0x3fffffff})\gg7)+1\) then writes zero.

F-D10 receives \(a_0\) and zero. After its calls, a fresh byte read at absolute address 1 selects a zero byte store. F-D11 receives \(4r_{10}\).

F-D12 has five calls with a different argument progression:

1. Call 1 receives \(4r_{11},4r_{11}\), and the earlier byte sampled from \(8r_6\), shifted left by 31.
2. Call 2 has no represented arguments and supplies \(a_1\).
3. Call 3 receives \(a_1\gg18\) and returns a 32-bit value.
4. Call 4 receives that value in all four positions.
5. Call 5 has no represented arguments and returns paired result \(r_{12}\).

A selected update through \((a_0\gg9)+1\) writes zero.

F-D13 is a relay beginning with \(h_{12}\). It forwards its second result normally. Afterwards, a selected update through \(\ell_{12}+1\) writes zero. F-D14 receives \(4r_{13}\).

F-D15 is a sampling family beginning with \(4r_{14}\) twice. Between calls 2 and 3, it reads a word at
\[
\ell_{12}+((\ell_{12}\gg19)\ll31).
\]
Call 3 receives the word four times; call 4 supplies \(a_2\). A selected update through \((\ell_{12}\gg9)+1\) writes zero.

Under 32-bit bit-vector semantics, the offset in that word address is either zero or \(\texttt{0x80000000}\), determined solely by original bit 19 of \(\ell_{12}\).

### Further byte, paired-result, and stack observations

F-D16 receives \(a_2\). Its following selected update through \(a_2+1\) writes zero. F-D17 receives \(4r_{16}\).

F-D18 is a sampling family beginning with \(4r_{17}\) twice. It reads one byte at \(a_2+14\) between calls 2 and 3 and passes that byte four times. Its last call returns a pair. A selected update through \((a_2\gg9)+1\) writes zero.

F-D19 is a relay beginning with \(h_{18}\). After its calls, a selected update through \(\ell_{18}+1\) writes zero. F-D20 receives \(4r_{19}\).

F-D21 is a sampling family beginning with \(4r_{20}\) twice. There is no direct memory sample between its second and third calls: call 3 receives \(s_{21}\) four times. Call 4 supplies \(a_3\). A selected update through \(4096\ell_{18}+1\) writes zero.

Let the retained shift from this point be \(a_3\gg19\). F-D22 receives, in order, \(a_3,a_3,a_3\gg19,a_3\). A selected update through \(a_3+1\) then writes zero. F-D23 receives \(4r_{22}\).

F-D24 is a sampling family beginning with \(4r_{23}\) twice. Its third call receives the same stack-storage address in all four positions. Its fourth returns a pair. A selected update through
\[
(a_3\gg19)\times\texttt{0x80000000}+1
\]
writes zero, conditional on the represented 32-bit arithmetic.

F-D25 is a relay beginning with \(h_{24}\). A selected update through \((\ell_{24}\gg19)+1\) follows, writing zero. F-D26 receives \(4r_{25}\).

F-D27 is a sampling family beginning with \(4r_{26}\) twice. Its second call supplies auxiliary offset \(a_4\). An unsigned byte is then captured at \(\ell_{24}+a_4\), and call 3 receives that captured byte four times. Call 4 supplies \(a_5\). A selected update through \((\ell_{24}\gg9)+1\) writes zero.

F-D28 receives \(a_5\). After its calls, a selected update through \(a_5+1\) writes zero. F-D29 receives \(4r_{28}\).

F-D30 is a sampling family beginning with \(a_5\ll31\) and \(4r_{29}\). It reads a halfword at the sum of \(a_5\gg19\) and the byte captured by F-D27, then passes that halfword four times. Its last call returns a pair.

### Paired destinations and direct field-like writes

A word at \(4h_{30}+20\) receives \(a_5\gg19\). F-D31 then receives \(\ell_{30},h_{30},0\). Another word at \(4r_{31}+20\) receives the same saved shift.

F-D32 receives \(4r_{31}\). A byte at \(\ell_{30}+13\) receives the low byte of \(2\ell_{30}\).

F-D33 receives a freshly read word at \(4r_{32}+68\). After its calls, the byte at the saved address \(\ell_{30}+13\) is overwritten with 4. The intervening calls separate these two byte writes and may observe or modify that location.

F-D34 receives \(4r_{33}\) and a freshly read word at \(((r_{32}\mathbin{\&}\texttt{0x3fffffff})\gg20)+68\). A word at \(256r_{34}+20\) then receives \(4r_{32}\).

F-D35 receives \(64r_{34}\) twice and
\[
((r_{32}\mathbin{\&}\texttt{0x3fffffff})\gg17)\ll31.
\]
A word at \(4r_{35}+20\) receives \((r_{34}\mathbin{\&}\texttt{0x03ffffff})\gg13\).

F-D36 receives \(4r_{35}\). Its following word store writes \(4r_{36}\) at \(16r_{36}+20\). F-D37 receives \(4r_{36}\) and \(r_{36}\ll24\), then a word at \(4r_{37}+20\) receives the retained value \(4r_{36}\).

F-D38 receives \(4r_{37}\); a word at \(16r_{38}+20\) receives \(4r_{38}\). F-D39 receives \(4r_{38},4r_{38},r_{38}\ll24\), after which a word at \(4r_{39}+20\) receives \(4r_{38}\).

F-D40 receives \(4r_{39}\); a word at \(16r_{40}+20\) receives \(4r_{40}\). Retain \(r_{40}\ll22\) separately. F-D41 receives \(4r_{40},4r_{40},0,r_{40}\ll22\), then a word at \(4r_{41}+20\) receives \(4r_{40}\).

F-D42 receives \(4r_{41}\). After its calls, a zero word is written at \(16r_{40}+36\), followed by a word read at \(16r_{40}+116\). Call the captured word **saved base**.

### Saved-base state and repeated selectors

F-D43 is a relay beginning with saved base, \(4r_{42}\), and the retained \(r_{40}\ll22\). Its third call forwards \(s_{43}\). After all seven calls, a word at saved-base-plus-36 receives
\[
q=s_{43}\gg20.
\]

F-D44 receives a fresh word read from saved-base-plus-116. After its calls, a selected update through saved-base-plus-one writes the low byte of \((r_{42}\mathbin{\&}\texttt{0x3fffffff})\gg20\).

F-D45 receives \(q\gg21\) and the retained \(r_{40}\ll22\). Afterwards, a new selector read at saved-base-plus-one selects a zero write. Because \(q\le4095\), the first argument is zero under the represented unsigned width.

F-D46 receives \(4r_{45}\). F-D47 is a sampling family beginning with \(4r_{46}\) twice. Its second call supplies \(a_6\); a byte at \(a_6+21\) is captured, passed once to call 3, and retained. After call 4, a selected update through \(1024q+1\) writes zero.

F-D48 is a relay beginning with that retained byte. Its third call receives \(s_{48}\) and \(q\gg21\). After its calls, a fresh selected update through \(1024q+1\) writes zero. F-D49 receives \(4r_{48}\).

F-D50 is a sampling family beginning with \(4r_{49}\) twice. Its second call supplies \(a_7\); a byte at \(a_7+21\) is captured and passed once to call 3. Call 4 supplies \(a_8\). A selected update through \(1024q+1\) writes zero. Retain
\[
n=(a_8\mathbin{\&}\texttt{0x1fffffff})\gg18.
\]

F-D51 receives the byte captured by F-D50. A fresh selected update through \(1024q+1\) writes zero. F-D52 receives \(4r_{51}\).

F-D53 is a sampling family beginning with zero and \(4r_{52}\). Its second call supplies \(a_9\); an unsigned byte at \(a_9+21\) is captured and passed once to call 3. Call 4 supplies \(a_{10}\). A word at \(4a_{10}+20\) receives the zero-extended captured byte.

### Word propagation and extracted fields

F-D54 receives \(a_{10}\). A word at \(4r_{54}+20\) receives the saved \(a_{10}\). F-D55 receives \(4r_{54}\); a word at \(16r_{55}+20\) receives \(4r_{55}\).

F-D56 receives \(n\ll13\). A word at \(4r_{56}+20\) receives the same retained shift. F-D57 receives \(4r_{56}\); a word at \(16r_{57}+20\) receives \(4r_{57}\).

F-D58 receives \(4r_{57}\). A word at \(4r_{58}+20\) receives \(4r_{57}\). F-D59 receives \(4r_{58}\). Retain the 32-bit scale \(t=r_{59}\ll2\); a selected update through \(1024n+1\) writes its low byte.

F-D60 is a relay beginning with \(t\). After its calls, a selected update through address 1 writes the low byte of \((r_{59}\mathbin{\&}\texttt{0x3fffffff})\gg20\). The selector address is 1 because the retained scalar used to form it is explicitly zero.

F-D61 receives \(4r_{60}\). Retain \(m=(t\mathbin{\&}\texttt{0x1fffffff})\gg6\).

F-D62 is a sampling family beginning with \(4r_{61}\) twice. Its third call receives \(s_{62}\) once. Call 4 supplies \(a_{11}\). A selected update through \(m+1\) writes zero. Retain \(a_{11}\ll3\).

F-D63 is a relay beginning with \(a_{11}\). Its following selected update through \(m+1\) writes zero. F-D64 receives \(4r_{63}\).

F-D65 is a sampling family beginning with \(4r_{64}\) twice. Call 3 receives \(s_{65}\) once; call 4 supplies \(a_{12}\). A word at \(4a_{12}+20\) receives the saved \(s_{65}\).

F-D66 receives \(a_{12}\); a word at \(4r_{66}+20\) receives the saved \(a_{12}\). F-D67 receives \(4r_{66}\); a word at \(16r_{67}+20\) receives \(4r_{67}\).

F-D68 receives \(4r_{67}\). A word at \(4r_{68}+20\) receives \(((4r_{67})\mathbin{\&}\texttt{0x3fffffff})\gg19\). F-D69 receives \(4r_{68}\); a word at \(16r_{69}+20\) receives \(4r_{69}\).

F-D70 receives a freshly read word at \(4r_{69}+20\), shifted right by 21. After its calls, a word at \(4r_{70}+20\) receives the retained \(a_{11}\) shifted left by seven in total. F-D71 receives \(4r_{70}\). Only after its fifth call is G1 explicitly cleared to zero.

The read used by F-D70 is not a read from the preceding destination \(16r_{69}+20\); the two address formulas remain distinct.

### Zero-based word sequences

F-D72 receives zero twice; a zero word is then stored at \(4r_{72}+20\). F-D73 receives \(4r_{72}\); a word at \(16r_{73}+20\) receives \(4r_{73}\).

F-D74 receives \(4r_{73}\) twice and a freshly read word at \(4r_{73}+20\), shifted right by 21. Afterwards, a word at \(4r_{74}+20\) receives \(4r_{73}\). F-D75 receives \(4r_{74}\); a word at \(16r_{75}+20\) receives \(4r_{75}\).

F-D76 receives three zeros followed by a freshly read word at \(4r_{75}+20\), masked with \(\texttt{0x1fffffff}\) and shifted right by 18. A zero word is then stored at \(4r_{76}+20\). F-D77 receives \(4r_{76}\); a word at \(16r_{77}+20\) receives \(4r_{77}\).

F-D78 receives \(4r_{77}\), then a word at \(4r_{78}+20\) receives that same retained scale. F-D79 receives \(4r_{78}\); a word at \(16r_{79}+20\) receives \(4r_{79}\).

F-D80 receives \(r_{79}\ll6\). A word at \(4r_{80}+20\) receives that retained shift. F-D81 receives \(4r_{80}\). A fresh selector read at absolute address 1 then selects a byte store of the low byte of \(4r_{81}\).

F-D82 is a relay beginning with \(4r_{81}\), but its third call receives zero instead of the second result. Afterwards, a fresh selected update through address 1 writes the low byte of \((r_{81}\mathbin{\&}\texttt{0x3fffffff})\gg20\). F-D83 receives \(4r_{82}\).

### Fixed-target call and further low-address sampling

F-D84 is a sampling family beginning with \(4r_{83}\) twice. Its third call is the separately represented fixed-target dependency and receives \(s_{84}\) twice. Call 4 supplies \(a_{13}\). A fresh selected update through address 1 writes zero.

F-D85 receives \(a_{13}\) and zero. A new selected update through address 1 writes zero. F-D86 receives \(4r_{85}\).

F-D87 is a sampling family beginning with \(4r_{86}\) twice. It captures an unsigned byte at saved \(s_{84}+1\) and passes it once to call 3. Call 4 supplies \(a_{14}\). A selected update through \((a_{13}\gg9)+1\) writes zero.

F-D88 receives \(a_{14}\). After its calls, a selected update through address 1 writes the low byte of \((s_{84}\mathbin{\&}\texttt{0x003fffff})\gg12\). F-D89 receives \(4r_{88}\).

F-D90 is a sampling family beginning with zero and \(4r_{89}\). It reads a byte at the numeric value of the byte captured by F-D87 plus one and passes it once. Call 4 supplies \(a_{15}\). A selected update through \((a_{14}\gg9)+1\) writes zero.

F-D91 receives \(a_{15}\). Its following selected update through address 1 writes zero: the represented stored expression masks the unsigned byte captured by F-D87 and shifts it right by 12. F-D92 receives \(4r_{91}\).

F-D93 is a sampling family beginning with \(4r_{92}\) and zero. It captures a fresh unsigned byte at absolute address 1, passes it once, and obtains \(a_{16}\) from call 4. A word at \(4a_{16}+20\) receives that captured byte, zero-extended.

### Further scaled-state construction

F-D94 receives \(a_{16}\); a word at \(4r_{94}+20\) receives the saved \(a_{16}\). F-D95 receives \(4r_{94}\); a word at \(16r_{95}+20\) receives \(4r_{95}\).

F-D96 receives \(4r_{95}\), then a zero word is stored at \(4r_{96}+20\). F-D97 receives \(4r_{96}\); a word at \(16r_{97}+20\) receives \(4r_{97}\).

F-D98 is a relay beginning with \(4r_{97}\). Afterwards, a word at \(4r_{98}+20\) receives that retained scale. F-D99 receives \(4r_{98}\); a word at \(16r_{99}+20\) receives \(4r_{99}\).

F-D100 receives zero. A word at \(4r_{100}+20\) receives \(4r_{99}\). F-D101 receives \(4r_{100}\); a word at \(16r_{101}+20\) receives \(4r_{101}\). Retain the separate 32-bit value \(v=r_{101}\ll3\).

F-D102 receives \(((4r_{101})\mathbin{\&}\texttt{0x7fffffff})\gg21\) twice. A word at \(4r_{102}+20\) receives the saved \(4r_{101}\). F-D103 receives \(4r_{102}\); a word at \(16r_{103}+20\) receives \(4r_{103}\).

F-D104 receives \(4r_{103}\) and \(v\gg22\). A word at \(4r_{104}+20\) receives \(4r_{103}\). F-D105 receives \(4r_{104}\), with its final result represented as an 8-bit character. A selected update through \(((4r_{103})\gg1)+1\) writes the low byte of four times that character’s represented value.

F-D106 is a relay beginning with \(v\gg22,v,v\gg22\). Its third call receives \(s_{106},v\gg22,v\gg22\). A selected update through \((v\gg22)+1\) writes zero. F-D107 receives \(4r_{106}\); retain its scale \(4r_{107}\).

### Direct base writes and offset-20 propagation

F-D108 is a sampling family beginning with \(4r_{107}\). Its third call receives \(s_{108}\) once; its fourth returns a 32-bit value, here called **propagation base**. A zero byte is then written at \((4r_{107})\times8192+9\).

F-D109 is a relay beginning with propagation base, \(s_{108}\gg22\), and \(s_{108}\). Its last call returns a pair. A byte at propagation-base-plus-nine receives the low byte of propagation base.

F-D110 receives \(4\ell_{109},h_{109}\), and a fresh word at propagation-base-plus-52. Its fifth call supplies \(a_{17}\). A word at \(4a_{17}+20\) receives propagation base; immediately afterwards, a word at the unscaled address \(a_{17}+20\) is captured.

F-D111 is a relay beginning with \(a_{17}\). A word at \(4r_{111}+20\) receives propagation base. Next, a word is captured at the preceding unscaled word capture shifted right by 22, plus 20; call this **chain word**.

F-D112 receives \(4r_{111}\). A word at \(4(\text{chain word}\gg1)+20\) receives \(4r_{112}\). Then a word at unshifted chain-word-plus-20 is captured as **next chain word**.

F-D113 is a relay beginning with chain word shifted right by one. A word at \(4r_{113}+20\) receives saved \(4r_{112}\). Only after that store is G1 read for the first later capture, **G1-first**.

F-D114 receives \(4r_{113}\); its fifth call supplies \(a_{18}\). A word at \(4a_{18}+20\) receives next chain word shifted right by 22.

F-D115 is a relay beginning with \(a_{18},a_{18},\text{G1-first}\gg23\), and next chain word shifted right by 22. Its third call forwards \(s_{115}\). A word at \(4r_{115}+20\) receives the retained next-chain-word shift.

The retained G1-first capture is also shifted right by 23 a second time. That double shift is zero for an unsigned 32-bit capture. It does not require G1 to remain unchanged.

F-D116 receives \(4r_{115}\); its fifth call supplies \(a_{19}\). A zero word is written at \(4a_{19}+20\).

F-D117 is a relay beginning with \(a_{19}\). A word at \(4r_{117}+20\) receives the double-shifted G1-first capture, hence zero under the stated width. A word is then captured at
\[
((a_{19}\mathbin{\&}\texttt{0x1fffffff})\gg19)+20.
\]

### Late selected-byte state and the saved absolute-address byte

F-D118 receives \(4r_{117}\); its fifth call supplies \(a_{20}\). Let **late shift** be the preceding word capture shifted right by 22. A selected update through address 1 writes its low byte.

F-D119 is a relay beginning with \(a_{20},a_{20},a_{20},0\), whose third call receives zero rather than its second result. A selected update through four-times-late-shift-plus-one then writes zero. F-D120 receives \(4r_{119}\).

F-D121 is a sampling family beginning with \(4r_{120}\) twice. Its third call receives \(s_{121}\) once; call 4 supplies \(a_{21}\). A fresh selected update through address 1 writes zero. Retain \(a_{21}\) as **late auxiliary base**.

F-D122 is a relay beginning with late auxiliary base in all three represented positions. Its third call receives \(s_{122}\) and \(s_{121}\gg23\). After its seventh call, the byte at absolute address 1 is captured as unsigned **saved absolute byte**. A zero byte is written at saved-absolute-byte-plus-17.

This is a retained numeric byte, not a deferred instruction to reload address 1.

F-D123 receives \(4r_{122}\). F-D124 is a sampling family beginning with \(4r_{123}\) twice. Immediately after its second call, its captured \(s_{124}\) is stored into G2. Call 3 receives the same saved scalar once; call 4 has no represented arguments.

Only after those calls, a word is read at saved-absolute-byte-plus-116, and its low byte is written at saved-absolute-byte-plus-one. Neither address is derived from a fresh read of address 1. Intervening dependencies may have changed address 1 without changing the saved numeric byte.

### Repeated word-to-byte copies

F-D125 is a relay beginning with late auxiliary base shifted left by 11. Its second result is paired; its third call receives that pair’s low half, high half, and \(s_{124}\gg23\). After its remaining four calls, let **copy base** be \(4r_{125}\). A word at copy-base-plus-116 is read and truncated into a byte at copy-base-plus-one.

F-D126 receives copy base. Retain the scale \(w=4r_{126}\). A fresh word at the same saved copy-base-plus-116 is then read and truncated into another byte write at copy-base-plus-one.

F-D127 is a double-relay beginning with late auxiliary base shifted left by 12, \(w,w,r_{126}\ll18\). After its nine calls, let **second copy base** be \(4r_{127}\). A word at second-copy-base-plus-116 is captured as **saved copied word**, and its low byte is written at second-copy-base-plus-one.

F-D128 receives second copy base. A word at \(16r_{128}+20\) receives saved copied word.

F-D129 is a double-relay beginning with \((w\gg2)\ll16\). A word at \(4r_{129}+20\) receives saved copied word. F-D130 receives \(4r_{129}\). Then the distinct direct word store at
\[
(w\mathbin{\&}\texttt{0xfffffffc})+20
\]
receives \(4r_{130}\).

That store precedes all calls in F-D131. Its destination uses the saved scale \(w\), with the bottom two bits explicitly cleared; its value uses the newly obtained \(r_{130}\).

F-D131 is a double-relay beginning with \(4r_{130}\) twice. A zero word is then stored at \(4r_{131}+20\).

### Second later G1 capture and final propagation

F-D132 receives \(4r_{131}\) and returns a pair. A word at \(4h_{132}+20\) receives \(4\ell_{132}\). Immediately after that store, G1 is read again into a separate capture, **G1-second**.

F-D133 receives \(4\ell_{132},h_{132},4\ell_{132},4\ell_{132}\). After its five calls, a word at \(4r_{133}+20\) receives the saved \(4\ell_{132}\). Only then is a word read at **G1-second-plus-20**, producing **final sampled word**.

This dereference uses the earlier G1-second capture. It is not a new G1 reload, and it is unrelated to the separately captured G1-first value.

F-D134 receives \(4r_{133}\) and returns a pair. A selected update through \((\text{saved copied word}\gg23)+1\) writes the low byte of final sampled word.

F-D135 is a relay beginning with \(h_{134},h_{134},h_{134},4\ell_{134}\). After its calls, a selected update through
\[
((\ell_{134}\mathbin{\&}\texttt{0x3fffffff})\gg21)+1
\]
writes the low byte of final sampled word shifted right by 22.

F-D136 receives \(4r_{135}\).

### Final global composition and termination

F-D137 has five calls:

1. Call 1 receives \(4r_{136}\) twice.
2. Call 2 has no represented arguments.
3. Call 3 also has no represented arguments and returns a 32-bit value.
4. Call 4 receives that returned value once.
5. Call 5 has no represented arguments and returns an 8-bit value, together with a represented auxiliary byte pointer.

After call 5, the direct effects occur in this order:

- Write zero through the auxiliary byte pointer.
- Read the represented three-byte region at absolute address 9 and overwrite G2 with a 32-bit composition: that region supplies the upper 24 bits, and the 8-bit call result supplies the low byte.
- Write a zero halfword at \((\ell_{134}\mathbin{\&}\texttt{0x3fffffff})\gg7\).
- Invoke F-D138-1, the address-zero target, with three represented zero arguments.

The final halfword address ranges from zero through \(\texttt{0x007fffff}\), inclusive, and need not be two-byte aligned.

The routine returns only if F-D138-1 returns normally. No interpretation of address zero as a service, trap, or valid function is established.

## Side effects

Direct side effects comprise the ordered byte, halfword, word, and global accesses described above. Dependency effects are additional and unbounded by the supplied contracts.

The routine has no represented transaction or rollback. A fault or nonreturning dependency may leave any preceding stores externally visible.

Important observation rules:

- Every separately described selector update reads memory again, except the explicitly retained saved absolute byte.
- Saved scalar captures remain the represented operands across calls; they are not fresh memory observations.
- Argument reads occur before their receiving call. Calls between reads may change later observations.
- Repeated word-to-byte copies include fresh word reads where specified.
- A word store at a scaled address does not imply a following unscaled read accesses the same location.

The supplied material warns that some global storage labels overlap smaller symbols. G1, G2, the three-byte region, and computed destinations cannot be assumed independent without a resolved layout.

No explicit allocation or release is represented. Ownership of accessed storage is not established. Passing the stack address does not establish permission to retain it, an initialized input extent, a writable extent, or a transfer of ownership.

Reentrancy, concurrency safety, synchronization, and atomicity are **documented_only**.

## Dependencies

F-6fd2e8501aef is the only dependency identity resolved by the supplied private map.

F-D01-1 through F-D138-1, including the intermediate family members defined above, are document-local opaque dependency IDs. They preserve call occurrence, order, represented arguments, and result dependencies without asserting unavailable function equivalence.

Most targets are represented as code-location-derived addresses with fixed offsets. Exceptions include the incoming-return-link-derived first target, the separately fixed target in F-D84, and address zero in F-D138-1.

Unknown dependency properties include:

- Argument interpretation and machine-level calling convention.
- Additional register-result validity.
- Reads, writes, global mutation, and aliasing effects.
- Memory retention, ownership transfer, allocation, or release.
- Error signaling, exceptions, faults, nonlocal exits, and termination.

No dependency-preservation assumption is needed to justify a saved scalar’s later use. Conversely, no prior store justifies predicting a later memory reload across intervening calls.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Entry argument order.** For input \(\texttt{0x89abcdef}\), F-D01-1 receives that value in positions one, two, and four, with zero in position three.

2. **Unsigned selector range.** A selected update whose selector read yields 255 writes at address 272. Sign-extending that selector is inconsistent with the represented unsigned byte read.

3. **Self-selecting alias.** If a selector address contains a byte selecting that same address as destination, the write uses the byte captured before the write. A later selected update must read again.

4. **Distinct scaling.** If an applicable acquisition result is 3, a destination of \(16r+20\) is address 68, not address 32. This example assumes the address is mapped and the access is permitted.

5. **Intermediate byte visibility.** A dependency in F-D33 may observe the first write at \(\ell_{30}+13\), before the subsequent overwrite with 4. Collapsing the writes loses an observable boundary.

6. **Fresh saved-base selector.** If a dependency changes saved-base-plus-one between the updates following F-D44 and F-D45, the zero write after F-D45 uses the changed selector.

7. **G1-first observation point.** If G1 contains \(\texttt{0x80000000}\) immediately after the store preceding F-D114, G1-first captures that value. Its first right shift by 23 is 256; its second is zero, even if later dependencies change G1.

8. **G1-second capture, not reload.** If G1-second is captured as \(\texttt{0x1000}\), and F-D133 later changes G1 to \(\texttt{0x2000}\), the final sampled word is still read at \(\texttt{0x1014}\), after F-D133’s computed store.

9. **Saved absolute byte across calls.** If the late capture after F-D122 reads 7 from address 1, the immediate zero store is at address 24. The later word-to-byte copy reads a word at address 123 and writes its low byte at address 8, even if intervening dependencies change address 1.

10. **Late aligned-scale store.** If saved \(w=\texttt{0x00000100}\) and \(4r_{130}=\texttt{0x12345678}\), the distinct word store writes that value at \(\texttt{0x114}\) before F-D131-1. It is not deferred until the subsequent zero store.

11. **Word read before byte truncation.** If a saved-base-plus-116 word is \(\texttt{0x123456ab}\), the corresponding byte store writes \(\texttt{0xab}\). An implementation that performs only a byte read has not preserved the represented access width.

12. **Final composition and failure.** If the final byte result is \(\texttt{0x7e}\) and the represented three-byte value is \(\texttt{0x123456}\), G2 receives numeric value \(\texttt{0x1234567e}\). If the following address-zero call faults, normal return is not promised, and preceding stores have already occurred.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid explicit-input domain.
- Correct recovery of executable control flow, targets, argument boundaries, and register observations.
- Signed overflow, character signedness, and target-specific shift and pointer arithmetic.
- Endianness, alignment rules, memory permissions, and validity of low addresses.
- Exact global layout and overlap relationships.
- Dependency contracts, errors, ownership effects, and normal-return guarantees.
- Stack-storage extent, initialization, retention permission, and lifetime obligations.
- Concurrency behavior and atomicity.

The two later G1 reads, their distinct capture points, the saved absolute byte, and the late masked-scale word store are not unresolved ordering questions: their positions and represented dataflow are established statically.

No null-input special case, bounds check, retry, recovery path, or safe failure guarantee is shown.

## Evidence

This contract is derived solely from the supplied static representation. The dependency-family descriptions preserve represented call counts, argument order, intervening access boundaries, and retained-result relationships throughout the body.

Width-derived conclusions are conditional on the represented widths and stated arithmetic interpretation. Application-level interpretations are not established.

No implementation access, dependency inspection, runtime execution, or runtime tests have been performed. Acceptance examples describe conditional consequences, not observed outcomes.
