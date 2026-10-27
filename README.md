# Sudoku as a Constraint Satisfaction Problem (C++)
This code was written in Fall 2025 and adapted for Mac in Fall 2026. In this Sudoku as a **constraint satisfaction problem (CSP)** which is the parsed through by the arc-consistency algorithm **AC-3** and **backtracking search with the MRV heuristic**.

## Sudoku as a CSP

| CSP part | How Sudoku is modeled |
|----------|-----------------------|
| **Variables** | The 81 cells, numbered 0 to 80 |
| **Domains** | The values 1 to 9 for each cell |
| **Constraints** | Two cells must hold different values if they share a row, column, or 3x3 subgrid |
| **Assignment** | A value chosen for each cell |

### Variables

A cell in row `a` and column `b` (each 0 to 8) has the index `9*a + b`.

```
 0  1  2 |  3  4  5 |  6  7  8
 9 10 11 | 12 13 14 | 15 16 17
18 19 20 | 21 22 23 | 24 25 26
---------+----------+---------
27 28 29 | 30 31 32 | 33 34 35
...
```

For example, row 7, column 7 is cell `9*7 + 7 = 70`.

### Constraints

All Sudoku rules are converted into **binary constraints**, stored in a boolean matrix:

```cpp
bool C[81][81];
```

`C[x][y]` is `true` if and only if cells `x` and `y` must be different, meaning they are in the same row, column, or 3x3 subgrid. Each cell is constrained against 20 others (8 in its row, 8 in its column, and 4 more in its subgrid), and each one is an **arc** in the constraint graph.

Arcs are stored in this format:

```cpp
struct arcType {
    int source;
    int destination;
};
```

So `arcType c = {xi, xj}` means "cell `xi` must differ from cell `xj`".

### Domains

```cpp
vector<int> domain[81];
```

Each cell has a list of values it could still take. An empty cell starts with `{1,2,3,4,5,6,7,8,9}`. A cell given by the puzzle (a **unary constraint**) is reduced right away to its one value, so if cell 70 is a 4, then `domain[70] = {4}`. Inference works by removing values from these lists.

### Assignment

```cpp
vector<int> assignment;
```

A vector of 81 integers, with the value of each cell and `0` for empty cells. A puzzle is solved when all 81 cells are assigned without breaking a constraint.

### Puzzle input

Each puzzle is a string of 81 characters read row by row, with `0` for an empty cell:

```
"000500002700600800001000300076050000002000000080000700390072600020000090050018030"
```

## Logic Functions

### `revise`

```cpp
bool revise(int xi, int xj, vector<int> domain[81], const bool C[81][81]);
```

Makes one arc `(xi, xj)` **consistent**. A value `x` in the domain of `xi` is kept only if some value `y` in the domain of `xj` satisfies the constraint, which for Sudoku means `x != y`.
**Returns** `true` if any value was removed, and `false` if the domain was unchanged.
For example, if `xj` is already fixed to `{5}`, then 5 is removed from `xi`'s domain, because no value in `xj` can differ from 5.

### `AC3`

```cpp
bool AC3(vector<int> domain[81], const bool C[81][81]);
```

Enforces consistency across **every** arc in the puzzle, repeating until nothing changes.
On many puzzles AC-3 alone removes a large share of the possibilities, and it fills in some cells completely. It works as a preprocessing step before the search. It changes the `domain` array in place.

### `backtrackCSP_MRV`

```cpp
vector<int> backtrackCSP_MRV(vector<int> assignment, int count,
                             vector<int> domain[81], const bool C[81][81]);
```

Backtracking search with the **Minimum Remaining Values (MRV)** heuristic. `count` is the number of cells already assigned, and `assignment` holds the current partial solution.
The basic `backtrackCSP` given with the assignment is the same search, but it picks the next unassigned cell in order (cell 0, 1, 2, ...), with no heuristic.

## Compile and run 

```
g++ -std=c++17 -O2 -o sudoku Sudoku_CSP.cpp
./sudoku (for mac)
.\sudoku.exe (for windows)
```
## Example Output

When the program starts, it loads the first puzzle, shows the board, and shows the starting domain of every cell.

### The Puzzle

The cells show their digit and empty cells are blank. The `#` lines separate the nine 3x3 subgrids.

```
#===========#===========#===========#
#   |   |   # 5 |   |   #   |   | 2 #
#---+---+---#---+---+---#---+---+---#
# 7 |   |   # 6 |   |   # 8 |   |   #
#---+---+---#---+---+---#---+---+---#
#   |   | 1 #   |   |   # 3 |   |   #
#===========#===========#===========#
#   | 7 | 6 #   | 5 |   #   |   |   #
#---+---+---#---+---+---#---+---+---#
#   |   | 2 #   |   |   #   |   |   #
#---+---+---#---+---+---#---+---+---#
#   | 8 |   #   |   |   # 7 |   |   #
#===========#===========#===========#
# 3 | 9 |   #   | 7 | 2 # 6 |   |   #
#---+---+---#---+---+---#---+---+---#
#   | 2 |   #   |   |   #   | 9 |   #
#---+---+---#---+---+---#---+---+---#
#   | 5 |   #   | 1 | 8 #   | 3 |   #
=====================================
```

### The CSP Domain

Each cell is drawn as a small 3x3 block of digits, and the position of a digit shows which value it is:

```
1 2 3
4 5 6
7 8 9
```

A digit is shown if that value is still possible for the cell, and a `.` means the value has been ruled out.

| Cell looks like | Meaning |
|-----------------|---------|
| All nine digits `1 2 3 / 4 5 6 / 7 8 9` | Empty cell, any value still possible |
| A single digit with dots elsewhere (like `. 5 .` in the middle) | Given cell, domain is one value |
| Fewer than nine digits | Some values have been removed by `revise` or `AC3` |

Starting domains for the first puzzle (before any inference):

```
 1 2 3  1 2 3  1 2 3    . . .  1 2 3  1 2 3    1 2 3  1 2 3  . 2 .
 4 5 6  4 5 6  4 5 6    . 5 .  4 5 6  4 5 6    4 5 6  4 5 6  . . .
 7 8 9  7 8 9  7 8 9    . . .  7 8 9  7 8 9    7 8 9  7 8 9  . . .

 . . .  1 2 3  1 2 3    . . .  1 2 3  1 2 3    . . .  1 2 3  1 2 3
 . . .  4 5 6  4 5 6    . . 6  4 5 6  4 5 6    . . .  4 5 6  4 5 6
 7 . .  7 8 9  7 8 9    . . .  7 8 9  7 8 9    . 8 .  7 8 9  7 8 9

 1 2 3  1 2 3  1 . .    1 2 3  1 2 3  1 2 3    . . 3  1 2 3  1 2 3
 4 5 6  4 5 6  . . .    4 5 6  4 5 6  4 5 6    . . .  4 5 6  4 5 6
 7 8 9  7 8 9  . . .    7 8 9  7 8 9  7 8 9    . . .  7 8 9  7 8 9


 1 2 3  . . .  . . .    1 2 3  . . .  1 2 3    1 2 3  1 2 3  1 2 3
 4 5 6  . . .  . . 6    4 5 6  . 5 .  4 5 6    4 5 6  4 5 6  4 5 6
 7 8 9  7 . .  . . .    7 8 9  . . .  7 8 9    7 8 9  7 8 9  7 8 9

 1 2 3  1 2 3  . 2 .    1 2 3  1 2 3  1 2 3    1 2 3  1 2 3  1 2 3
 4 5 6  4 5 6  . . .    4 5 6  4 5 6  4 5 6    4 5 6  4 5 6  4 5 6
 7 8 9  7 8 9  . . .    7 8 9  7 8 9  7 8 9    7 8 9  7 8 9  7 8 9

 1 2 3  . . .  1 2 3    1 2 3  1 2 3  1 2 3    . . .  1 2 3  1 2 3
 4 5 6  . . .  4 5 6    4 5 6  4 5 6  4 5 6    . . .  4 5 6  4 5 6
 7 8 9  . 8 .  7 8 9    7 8 9  7 8 9  7 8 9    7 . .  7 8 9  7 8 9


 . . 3  . . .  1 2 3    1 2 3  . . .  . 2 .    . . .  1 2 3  1 2 3
 . . .  . . .  4 5 6    4 5 6  . . .  . . .    . . 6  4 5 6  4 5 6
 . . .  . . 9  7 8 9    7 8 9  7 . .  . . .    . . .  7 8 9  7 8 9

 1 2 3  . 2 .  1 2 3    1 2 3  1 2 3  1 2 3    1 2 3  . . .  1 2 3
 4 5 6  . . .  4 5 6    4 5 6  4 5 6  4 5 6    4 5 6  . . .  4 5 6
 7 8 9  . . .  7 8 9    7 8 9  7 8 9  7 8 9    7 8 9  . . 9  7 8 9

 1 2 3  . . .  1 2 3    1 2 3  1 . .  . . .    1 2 3  . . 3  1 2 3
 4 5 6  . 5 .  4 5 6    4 5 6  . . .  . . .    4 5 6  . . .  4 5 6
 7 8 9  . . .  7 8 9    7 8 9  . . .  . 8 .    7 8 9  . . .  7 8 9
```

Blank lines separate the rows of cells, and the larger gaps separate the 3x3 subgrids. At this point only the givens are reduced to one value, and every empty cell still holds all nine, because no inference has run yet.

### Testing `revise`

The program then lets you test `revise` on any pair of cells, three times:

```
Testing the revise function...
Enter the two cells you want to resolve (indexed 0-80 as discussed) :
```

Type two cell indices separated by a space (for example `0 3`) and press Enter. The first cell is `xi`, the one whose domain may shrink, and the second is `xj`, the one it is checked against.
For example, `0 3` asks whether cell 0 must lose any values given cell 3 (the given 5 in the top row). Cell 3 has only one possible value, so `5` is removed from cell 0's domain.


## Helper Functions
The remaining functions were not written by me but help set up and execute the sudoku game. 

| Function | Role |
|----------|------|
| `fillConst` | Builds the constraint matrix `C` |
| `getSudokuCSP` | Turns an 81-character puzzle string into the starting `assignment` and `domain` |
| `getSudokuBoard`, `displayBoard`, `fillDisplayBoard` | Draw the puzzle or solution as a text board |
| `displaySudokuCSP` | Draws every cell's remaining domain, useful for seeing what `revise` and `AC3` removed |
| `displaySudokuSimple` | Draws a compact version of an assignment |
| `main` | Loads the puzzles, tests `revise` interactively, then runs AC-3 and MRV backtracking on sample puzzles |

The `main` function also asks for two cell indices to test `revise` by hand: enter any two cells (0 to 80) and it reports whether the first cell's domain changed.
