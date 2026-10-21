# Static Code Verification - Code Review Report

| | |
|---|---|
| **File reviewed** | `game.cpp` (135 lines, C++) |
| **Method** | Manual static review against the checklist in `Review_Checklist.md` |
| **Reviewer** | Talha Khalil (Sp-24/BSSE/Sem-6/Sec-B/133) |
| **Submitted to** | Ms. Saleha |
| **Course** | Software Verification & Validation |

**Severity scale used:**

| Severity | Meaning |
|---|---|
| **Critical** | Undefined behaviour or memory corruption that is actually triggered by the current `main()`; likely crash or security hole |
| **High** | Undefined behaviour or wrong result on a reachable path |
| **Medium** | Wrong result for some inputs, or missing validation |
| **Low** | Design, maintainability or performance problem; no wrong output today |

## Code Review Report

| ID | Line number | Defect type | Why it is a problem | Proposed solution | Severity |
|---|---|---|---|---|---|
| D-01 | 17–22 | Initialization | Members are assigned in the constructor body instead of being initialised. `name` is default-constructed and then copied again. | Use an initializer list: `Monster(int i, std::string n, int h, int a) : id(i), name(std::move(n)), health(h), attack(a) {}` | Low |
| D-02 | 10–15 | Classes (encapsulation) | All `Monster` data is `public`, so any code can set `health` to a negative value or change `id`. Nothing protects the class invariants. | Make the members `private`. Add getters and a `takeDamage(int)` method that keeps `health >= 0`. | Low |
| D-03 | 27, 31–33 | Initialization | The constructor sets `monsterCount` but leaves the `monsters[]` pointers holding garbage. Any read past `monsterCount` (see D-10) dereferences a random address. | Initialise the array: `Monster* monsters[MAX_MONSTERS] = {};` (all `nullptr`), or use `std::vector<std::unique_ptr<Monster>>`. | Medium |
| D-04 | 25–38, 133 | Classes (Rule of Three) | `Game` owns raw pointers and has a destructor, but no copy constructor or copy assignment. `Game copy = game;` (L133) copies the pointers. When `main` ends, both destructors delete the same monsters: a **double delete**. | Delete copying (`Game(const Game&) = delete; Game& operator=(const Game&) = delete;`), or implement a deep copy, or use `std::vector<std::unique_ptr<Monster>>` (Rule of Zero). | Critical |
| D-05 | 41 (triggered at 119) | Arrays / buffers (overflow) | `spawnMonster` never checks `monsterCount < MAX_MONSTERS`. `main` spawns 6 monsters into an array of 5, so L119 writes `monsters[5]` past the end and corrupts `monsterCount`/memory. `readBoss` (L108) adds a 7th. | `if (monsterCount >= MAX_MONSTERS) { std::cerr << "Monster limit reached\n"; return false; }` before inserting. Return `bool` so the caller knows. | Critical |
| D-06 | 47 | Control flow (`=` instead of `==`) | `monsters[i]->id = id` **assigns** instead of comparing. For any non-zero `id` the condition is always true, so it returns the first monster and overwrites its id. `findMonster(99)` returns the Goblin and renames it to 99. | Change to `if (monsters[i]->id == id)`. Turn on `-Wall -Wparentheses`, which flags this. | Critical |
| D-07 | 54–56 (called at 123) | Pointers (null dereference) | `findMonster` returns `nullptr` when the id is not found, but `attackMonster` uses `m->health` without checking. `attackMonster(99, 20)` in `main` dereferences null once D-06 is fixed. | `if (m == nullptr) return -1;` (or throw `std::invalid_argument`) before using `m`. | High |
| D-08 | 55–56 | Arithmetic (signed/unsigned) | `dmg` is `unsigned`, so `m->health - dmg` is computed as unsigned and can **never be < 0**. The check is dead code. A negative `damage` also becomes a huge unsigned value. | Keep everything signed: `if (damage < 0) return -1; if (m->health - damage <= 0) { m->health = 0; return 0; }`. Remove the `unsigned` variable. | High |
| D-09 | 53, 56–58 | Control flow (logic) / missing validation | When damage would kill the monster the function returns 0 **without** applying the damage, so a monster can never die. A negative `damage` is accepted and *increases* health. | Validate `damage >= 0`. Clamp: `m->health = std::max(0, m->health - damage); return m->health;` | Medium |
| D-10 | 64–65 | Loops (off-by-one) | `i <= monsterCount` runs one step too far and reads `monsters[monsterCount]`. That is an uninitialised pointer (D-03), or past the array when it is full, so it is undefined behaviour or a crash. | Use `i < monsterCount`. | Critical |
| D-11 | 70–73 | Function call (dangling reference) | `getMonsterLabel` returns `std::string&` to the local `label`, which is destroyed when the function returns. Printing it at L124 reads freed stack memory. | Return by value: `std::string getMonsterLabel(int id)`. | High |
| D-12 | 71–72 | Pointers (null dereference) | `findMonster` can return `nullptr`, but `m->id` and `m->name` are used without a check. | `if (!m) return "Unknown monster";` (or throw). | High |
| D-13 | 77–79 | Initialization (uninitialised variable) | `double sum;` is never initialised, so the total starts from garbage and the average is random. | `double sum = 0.0;` | High |
| D-14 | 80 | Arithmetic (division by zero / truncation) | If `monsterCount == 0`, this divides by zero. The `double` result is also silently truncated to `int` by the `int` return type. | `if (monsterCount == 0) return 0.0;` and return `double` (or round explicitly with `std::lround`). | Medium |
| D-15 | 84–86 | Pointers (use after free) | `delete m;` runs **before** `m->name` is printed, which reads freed memory. If the id is not found, `m` is `nullptr` and `m->name` crashes. | Check for `nullptr`, print the name first, then delete: `if (!m) return; std::cout << "Defeated " << m->name << "\n"; delete m;` | Critical |
| D-16 | 85, 36–37 | Pointers (dangling slot → double delete) | `defeatMonster` deletes the monster but leaves its pointer in `monsters[]` and does not reduce `monsterCount`. Later `findMonster` and `totalMonsterHealth` read freed memory, and the destructor deletes it **again**. | Remove the entry from the array: find its index `k`, `delete monsters[k]`, shift the remaining pointers left, `monsterCount--`. Or erase it from a `std::vector<std::unique_ptr<Monster>>`. | Critical |
| D-17 | 90–91 | Arithmetic (integer division) / missing validation | `attack * armorPercent / 100` is evaluated in **int**, so the fraction is lost before converting to `double` (e.g. `7 * 25 / 100 = 1`, not 1.75). `armorPercent` is not range-checked, and a large `attack * armorPercent` can overflow. | `return attack * (armorPercent / 100.0);` and validate `0 <= armorPercent <= 100`. If armour is meant to *reduce* damage, use `attack * (100 - armorPercent) / 100.0`. | Medium |
| D-18 | 94–99 (called at 128) | Control flow (missing return) | When `score <= 100` the function reaches the end without a `return`, which is undefined behaviour. `main` calls `playerRank(50)`, which is exactly this path. | Add a final `return 3;` (or `else return 3;`) so every path returns a value. Compile with `-Wreturn-type`. | High |
| D-19 | 102, 105 | Arrays / buffers (input overflow) | `std::cin >> name` into `char[10]` has no length limit. Any boss name of 10+ characters overflows the heap buffer. This is a classic security vulnerability. | Use `std::string name; std::cin >> name;` (or `std::getline`). If a char array is required: `std::cin >> std::setw(10) >> name;`. | Critical |
| D-20 | 103, 107 | Initialization / input validation | `health` is uninitialised, and if the user types non-numeric input `cin` fails, so `health` is garbage or 0. A negative or zero health is also accepted. | `int health = 0;` then `if (!(std::cin >> health) || health <= 0) { /* reject and re-prompt */ }`. | Medium |
| D-21 | 102, 109 | Pointers (mismatched delete) | Memory from `new char[10]` is freed with `delete` instead of `delete[]`, which is undefined behaviour. | Use `delete[] name;`. Better: use `std::string` (D-19) so no manual memory is needed. | High |
| D-22 | 121 | Pointers (memory leak) | `bonus` is allocated with `new` but never used or deleted, so the memory leaks. | Remove it, or create it on the stack (`Monster bonus(0, "Slime", 1, 0);`), or add it through `spawnMonster`. | Low |
| D-23 | 17, 40 | Function call (inefficient parameter passing) | `std::string` is passed **by value**, which makes an extra copy on every call. | Pass `const std::string&` (or by value plus `std::move` into the member). | Low |
| D-24 | 45, 62, 76 | Classes (const-correctness) | `findMonster`, `totalMonsterHealth` and `averageAttack` do not modify the object but are not `const`, so they can't be called on a `const Game&`. | Mark them `const`, e.g. `int totalMonsterHealth() const`. | Low |

## Summary

| Severity | Count | Defect IDs |
|---|---|---|
| Critical | 7 | D-04, D-05, D-06, D-10, D-15, D-16, D-19 |
| High | 7 | D-07, D-08, D-11, D-12, D-13, D-18, D-21 |
| Medium | 5 | D-03, D-09, D-14, D-17, D-20 |
| Low | 5 | D-01, D-02, D-22, D-23, D-24 |
| **Total** | **24** | |

| Defect type (checklist category) | Count |
|---|---|
| Initialization | 4 (D-01, D-03, D-13, D-20) |
| Pointers (null / delete) | 6 (D-07, D-12, D-15, D-16, D-21, D-22) |
| Arrays / buffers | 2 (D-05, D-19) |
| Loops | 1 (D-10) |
| Arithmetic | 3 (D-08, D-14, D-17) |
| Control flow | 3 (D-06, D-09, D-18) |
| Function call | 2 (D-11, D-23) |
| Classes | 3 (D-02, D-04, D-24) |

## What happens when `main()` runs as written

Tracing `main()` shows that most of the Critical defects are actually triggered, not just possible:

1. **L119**: the 6th `spawnMonster` writes past the end of `monsters[5]` (D-05).
2. **L123**: `attackMonster(99, 20)` → `findMonster` assigns instead of comparing (D-06), so it returns the Goblin and sets its id to 99.
3. **L124**: `getMonsterLabel(2)` → D-06 again turns the Goblin's id into 2, and the returned reference is dangling (D-11).
4. **L125**: `totalMonsterHealth()` reads one slot past the end (D-10).
5. **L126**: `averageAttack()` starts from an uninitialised `sum` (D-13).
6. **L128**: `playerRank(50)` reaches the end of the function with no `return` (D-18).
7. **L130**: `defeatMonster(3)` deletes the first monster, because of D-06, then reads its name after freeing it (D-15).
8. **L131**: `readBoss` overflows the buffer if the name is 10+ characters (D-19) and uses the wrong `delete` (D-21).
9. **L133–135**: `Game copy = game;` makes a shallow copy, and at the end of `main` both destructors delete the same pointers, including the one already deleted at L130 (D-04, D-16).

## Recommendations

- **Fix all Critical and High defects before acceptance.** Most of them disappear if `Game` stores monsters in `std::vector<std::unique_ptr<Monster>>`, `std::string` is used instead of `char*`, and functions return by value.
- **Compile with warnings as errors:** `g++ -std=c++17 -Wall -Wextra -Werror`. That would have caught D-06, D-08, D-13 and D-18 automatically.
- **Run a static analyser** (`cppcheck --enable=all game.cpp` or `clang-tidy`), and use AddressSanitizer (`-fsanitize=address`) during testing to confirm the memory defects are fixed.
