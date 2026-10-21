# Static Code Verification  Review Checklist

| | |
|---|---|
| **File reviewed** | `game.cpp` (135 lines, C++) |
| **Program** | Text-based monster battle game |
| **Method** | Manual static review (desk check, no execution) |
| **Reviewer** | Talha Khalil (Sp-24/BSSE/Sem-6/Sec-B/133) |
| **Submitted to** | Ms. Saleha |
| **Course** | Software Verification & Validation |

I went through the code line by line against the 8 checklist categories below. For each check, **ISSUE** means at least one defect was found and **OK** means the code passes that check. The defect IDs (D-xx) point to the full details in `Code_Review_Report.md`.

## Review Checklist

| # | Category | Check point | Result | Defect (line) |
|---|---|---|---|---|
| 1 | **Initialization** | Are all local variables initialised before they are used? | **ISSUE** | D-13 (L77 `sum`), D-20 (L103 `health`) |
| | | Does every constructor initialise all class members? | **ISSUE** | D-03 (L27, L31–33 `monsters[]` never initialised) |
| | | Do constructors use member initializer lists? | **ISSUE** | D-01 (L17–22) |
| | | Are constants defined once and used instead of magic numbers? | **OK** | `MAX_MONSTERS` (L8) is used for the array size |
| 2 | **Pointers (null / delete)** | Is every pointer checked for `nullptr` before it is dereferenced? | **ISSUE** | D-07 (L56), D-12 (L72), D-15 (L86) |
| | | Is memory never used after `delete`? | **ISSUE** | D-15 (L85–86) |
| | | Is every `new` matched by exactly one `delete`? | **ISSUE** | D-16 (L85 + L37 double delete), D-22 (L121 leak) |
| | | Is `new[]` released with `delete[]`? | **ISSUE** | D-21 (L102 / L109) |
| 3 | **Arrays / buffers** | Is the index checked before writing into an array? | **ISSUE** | D-05 (L41, overflowed by L119) |
| | | Is the index within bounds on every read? | **ISSUE** | D-10 (L64–65) |
| | | Is user input limited to the size of the buffer? | **ISSUE** | D-19 (L102, L105) |
| | | Are buffer sizes given by named constants? | **ISSUE** | D-19 (L102 uses a magic `10`) |
| 4 | **Loops** | Are loop bounds correct (no off-by-one)? | **ISSUE** | D-10 (L64 uses `<=`) |
| | | Does every loop terminate? | **OK** | L36, L46, L64, L78 are all bounded `for` loops |
| | | Does the loop body only use initialised data? | **ISSUE** | D-10 (L65), D-13 (L79) |
| | | Are loop counter types consistent with the bound? | **OK** | `int i` vs `int monsterCount` |
| 5 | **Arithmetic** | Is signed and unsigned arithmetic not mixed? | **ISSUE** | D-08 (L55–56) |
| | | Is integer division avoided where a fractional result is needed? | **ISSUE** | D-17 (L91), D-14 (L80 truncation) |
| | | Is every division protected against division by zero? | **ISSUE** | D-14 (L80) |
| | | Can any calculation overflow, or produce an invalid value from bad input? | **ISSUE** | D-09 (L58 negative damage), D-17 (L91) |
| 6 | **Control flow** | Do comparisons use `==` and not `=`? | **ISSUE** | D-06 (L47) |
| | | Does every path of a non-void function return a value? | **ISSUE** | D-18 (L94–99) |
| | | Are conditions logically correct? | **ISSUE** | D-09 (L56–57) |
| | | Are invalid or "not found" cases handled? | **ISSUE** | D-07 (L123 id 99), D-20 (L107) |
| 7 | **Function call** | Does no function return a reference or pointer to a local variable? | **ISSUE** | D-11 (L70–73) |
| | | Are return values (e.g. `nullptr` from `findMonster`) checked by callers? | **ISSUE** | D-07 (L54), D-12 (L71), D-15 (L84) |
| | | Are large objects passed by `const&`? | **ISSUE** | D-23 (L17, L40) |
| | | Are argument ranges validated (damage, armour %)? | **ISSUE** | D-09 (L53), D-17 (L90) |
| 8 | **Classes** | Does a class that owns raw pointers follow the Rule of Three? | **ISSUE** | D-04 (L25–38, copied at L133) |
| | | Is class data encapsulated (private members)? | **ISSUE** | D-02 (L10–15) |
| | | Does the destructor free every owned resource exactly once? | **ISSUE** | D-16 (L36–37 after L85) |
| | | Are read-only member functions marked `const`? | **ISSUE** | D-24 (L45, L62, L76) |

## Summary by Category

| # | Category | Checks | OK | ISSUE | Overall result |
|---|---|---|---|---|---|
| 1 | Initialization | 4 | 1 | 3 | **ISSUE** |
| 2 | Pointers (null / delete) | 4 | 0 | 4 | **ISSUE** |
| 3 | Arrays / buffers | 4 | 0 | 4 | **ISSUE** |
| 4 | Loops | 4 | 2 | 2 | **ISSUE** |
| 5 | Arithmetic | 4 | 0 | 4 | **ISSUE** |
| 6 | Control flow | 4 | 0 | 4 | **ISSUE** |
| 7 | Function call | 4 | 0 | 4 | **ISSUE** |
| 8 | Classes | 4 | 0 | 4 | **ISSUE** |
| | **Total** | **32** | **3** | **29** | |

**Conclusion:** every category has at least one issue. In total I found **24 defects**: 7 Critical, 7 High, 5 Medium and 5 Low. The most serious are memory-safety defects: buffer overflows, use-after-free, double delete and dangling references. They cause undefined behaviour and can crash the program as it currently runs in `main()`. The code should not be accepted until the Critical and High defects are fixed. Details and fixes are in `Code_Review_Report.md`.
