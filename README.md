# Static Code Review – `game.cpp`

**Software Verification & Validation – Lab Task: Static Code Verification**

## Created by

| Name | Roll No. |
|---|---|
| **Talha Khalil** | Sp-24/BSSE/Sem-6/Sec-B/133 |
| **Abdullah Siddiqui** | 094 |

**Submitted to:** Ms. Saleha  
**Program:** BSSE, Semester 6, Section B, Lahore Garrison University

## About the code

`game.cpp` is a small text-based **monster battle game** in C++ (135 lines). It has two classes:

- **`Monster`**: holds a monster's `id`, `name`, `health` and `attack`.
- **`Game`**: keeps up to `MAX_MONSTERS` (5) monsters in a fixed-size array of pointers. It can spawn, find, attack and defeat monsters, and calculate the total health, average attack and a monster label.

There are also helper functions: `calculateDamage` (attack reduced by armour %), `playerRank` (rank from a score) and `readBoss` (reads a boss from user input). `main()` spawns 6 monsters, calls each function once, defeats a monster, reads a boss and finally copies the game.

The code was given to us with deliberate defects, so that we could find them **without running it**.

## What we did

We carried out a **manual static code verification**, a desk check of the source without executing it, in two steps:

### 1. Review Checklist → [`Review_Checklist.md`](Review_Checklist.md)

We checked the code against 8 categories, with 4 check points each (32 checks in total). We marked each one **ISSUE** or **OK** and linked it to the defect it found.

| # | Category | Result |
|---|---|---|
| 1 | Initialization | ISSUE |
| 2 | Pointers (null / delete) | ISSUE |
| 3 | Arrays / buffers | ISSUE |
| 4 | Loops | ISSUE |
| 5 | Arithmetic | ISSUE |
| 6 | Control flow | ISSUE |
| 7 | Function call | ISSUE |
| 8 | Classes | ISSUE |

29 of the 32 checks found an issue, and 3 passed.

### 2. Code Review Report → [`Code_Review_Report.md`](Code_Review_Report.md)

For every defect we recorded the **line number, defect type, why it is a problem, proposed solution and severity**. We found **24 defects**:

| Severity | Count |
|---|---|
| Critical | 7 |
| High | 7 |
| Medium | 5 |
| Low | 5 |

The most important ones:

- **Line 47:** `=` used instead of `==`, so `findMonster` always returns the first monster and overwrites its id.
- **Lines 41 / 119:** a 6th monster is written into a 5-slot array (buffer overflow).
- **Line 64:** off-by-one loop (`<=`) reads past the last monster.
- **Lines 85–86:** the monster's name is printed after it has been deleted (use-after-free), and the destructor later deletes it again.
- **Line 105:** the boss name is read into a 10-character buffer with no limit (buffer overflow).
- **Line 133:** `Game` is copied without a copy constructor, so both copies delete the same monsters (double delete).

We also traced `main()` step by step and showed that most of the Critical defects are actually triggered when the program runs as written.

## Recommendations

- Fix all Critical and High defects before the code is accepted.
- Use `std::vector<std::unique_ptr<Monster>>` and `std::string` instead of raw pointers and `char[]`. This removes most of the memory defects.
- Compile with `-Wall -Wextra -Werror`, and run `cppcheck` / `clang-tidy` and AddressSanitizer to confirm the fixes.

## Files

| File | Description |
|---|---|
| [`game.cpp`](game.cpp) | Source code under review |
| [`Review_Checklist.md`](Review_Checklist.md) | Review checklist (Category, Check point, Result, Defect) |
| [`Code_Review_Report.md`](Code_Review_Report.md) | Code review report (Line number, Defect type, Why it is a problem, Proposed solution, Severity) |
