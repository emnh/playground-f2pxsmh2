# Fast Connected Components for 6x12 bitboard

## Deprecated
This article is now deprecated in favour of a [faster and simpler version using BFS](https://tech.io/playgrounds/75040/fast-6x12-connected-components-using-bit-optimized-bfs). 

## Introduction
This optimized code was made for [Smash the Code](https://www.codingame.com/multiplayer/bot-programming/smash-the-code) contest.
We assume that you are using one 6x12 bitboard per color,
represented as the least significant bits of an 128 bit integer (__uint128_t), which is supported by GCC.
You can expect about say 800k-1M separations of one color into connected components per 100ms,
versus say 200k-250k for a regular bit-BFS per index.
It depends on the hardware, but in general expect a 4x speedup over normal bit-BFS.
The output is an array of components, which are themselves 6x12 bitboards.

## Resources
Alternative ways to do this algorithm can be found at Wikipedia (perhaps a bit-optimized version of one of these can be even faster than this algorithm(?)):
[Connected-component labeling](https://en.wikipedia.org/wiki/Connected-component_labeling) and
[Flood fill](https://en.wikipedia.org/wiki/Flood_fill).

## Preliminaries

The main idea is to create a lookup table for the transition between 2 rows.
This allows you to iterate over the rows 2 by 2 and connect the components already computed in the table.

For a row of 6 columns, the maximum number of components is 3, for example 101010, 010101 or 101101.
You can see this because each component must be separated by a space (0),
so 4 components (1) have 3 separators (0), which makes 7,
which is greater than the columns, 6, so there are at most 3 components.

This means we can keep 3 running active components and seal components as soon as they finish.
There is a caveat to this, in that one running component can terminate while another running component keeps it active.
This happens for example with this configuration:
```
111000
101000
001000
001000
```

How many components can there be total? 6 * 12 / 2 = 72 / 2 = 36, using more or less the same logic as before, with a checkerboard pattern.
We also save one "component" number 0 for inactive components, the background, so let it be 37 total.

## The structure of the lookup table

So what do we have so far?
We have a lookup table for two rows, with index 0 to 2^12 = 4096.
We'll soon describe what we store in the table.
Second we have 3 active components, which are indices into a 37 element table of components.

What do we need to store in the table?
For each of the 3 running components, we need to know its transition from one row to the next.

What is a transition?
Well there are 3 input components (first row) and 3 output (second row) components.
So for each running component, we need to know which combination, if any, of the previous components it's a merger of.
This can be done as a bit index with 3 compoents of 3 bits,
where one bit is set if the input component should be continued in the output component.

In addition we store the second row, and this row will be added to the output components,
shifted into the correct row of the current iteration on the 6x12 grid.

The input components are indexed from bit index 0 to 5 in the first row.
The output components are indexed from bit index 6 to 11 in the second row.
To compute the components of the table you can use regular bit-BFS on each index which matches pattern "^1|01" in the 2 rows.

## An example of the lookup table

Say for example:
```
101010
111001
```

Here we have input components (notice the first and second component are equal):
```
101000 101000 000010
111000 111000 000000
```

And the output components are:
```
101000 000000 000000
111000 000001 000000
```

So since the first output component is equal to the first and second input component we store index 011 for the first output component to indicate this.
```
So tableIndex[101010111001][0] = 011.
And tableRow[101010111001][0] = 111000.
```

Further, the two other components do not continue an input component, so we have:
```
So tableIndex[101010111001][1] = 000.
And tableRow[101010111001][1] = 000001.
```

And for the last we have:
```
So tableIndex[101010111001][2] = 000.
And tableRow[101010111001][2] = 000000.
```

## How to use the table in iteration over bitboard / grid rows

Once you have the table initialized, the iteration proceeds as follows:
First you prepend an empty row to your grid, so that the transition into the components of the first row is easy and fits into the loop.
Start with the top two rows (row 0 and row 1) of the grid, then lookup that in the table.
Continue by shifting down one row and looking up row 1 and row 2 in the table.

First, save oldComponents[3] = runningComponents[3] and do runningComponents[3] = 0;
For each row looked up, there are 3 cases: continue component, new component and terminate component.
If the tableIndex is > 0, we continue, else if tableRow > 0, new component, and else terminate component.

### Continue component
If tableIndex is > 0, then add (|, or, union) the oldComponents corresponding to the tableIndex bits together to form the new runningComponent.
Then add the tableRow also to the component to account for new bits of this row.
There is a special case to handle, which is if two or more bits are set in the index, because this means to merge two components.
For the first index you can just use the same component number on the running component, but for second and third,
you must replace all references in oldComponents and runningComponents with references to the first index,
and zero out the merged component (could better be swapped out with last component, components[componentCount - 1] and decrease componentCount (--)).

Here's the merge handling:
```C++
    components[oldComponents[i]].board = 0;
    int mergeComponent = oldComponents[i];
    for (int j = 0; j < maxComponentsPerRow; j++) {
        if (oldComponents[j] == mergeComponent) {
            oldComponents[j] = runningComponents[c];
        }
        if (runningComponents[j] == mergeComponent) {
            runningComponents[j] = runningComponents[c];
        }
    }
```

### New component
For a new running component, allocate a new component index and set the indexed component to the tableRow.
runningComponent[0] = componentCount++, components[runningComponent[i]] = tableRow.

### Terminate component
For terminating a running component, just set its index to 0: runningComponent[i] = 0.

## Optimizations

You can skip empty rows (this is done in the example code).

It is possible to further optimize the code to achieve twice the speed, for 2M iterations per 100ms.
One big improvement you can make is to preprocess the input and remove solitary bits with no neighbours,
that is, components of size 1,
as for StC we are only interested in components of size 4 or more.
You can do this with a few bit operations on the grid.

## Code

```C++ runnable
#pragma GCC optimize("Ofast","unroll-loops","omit-frame-pointer","inline")
#include <iostream>
#include <cstdlib>
#include <chrono>

using namespace std;

using time_interval_t = std::chrono::microseconds;
using myClock = std::chrono::high_resolution_clock;

const int rowCount = 12;
const int colCount = 6;
const int twoRowsCellCount = 2 * colCount;
const int colorCount = 6;
const int cellCount = rowCount * colCount;
const int backgroundCount = 1;
const int maxComponents = backgroundCount + cellCount / 2;
const int maxComponentsPerRow = colCount / 2 + colCount % 2;
const int lookupTableSizeFor2Rows = 1 << (2 * colCount);

constexpr bool DEBUG = false;

inline int clz_u128 (__uint128_t u) {
  uint64_t hi = u>>64;
  uint64_t lo = u;
  int retval[3]={
    __builtin_ctzll(hi)+64,
    __builtin_ctzll(lo),
    128
  };
  int idx = !hi + ((!lo)&(!hi));
  return retval[idx];
}

template <class T>
class mybitsetT {
public:
  T data;

  mybitsetT<T>() {
    data = 0;
  }

  mybitsetT<T>(T data) {
    this->data = data;
  }

  const mybitsetT<T> boundsLimit() const {
    return mybitsetT<T>(data & (((T) 1 << cellCount) - (T) 1));
  }

  const mybitsetT<T> boundsLimit(const int N) const {
    return mybitsetT<T>(data & (((T) 1 << N) - (T) 1));
  }

  const mybitsetT<T> operator~ () const noexcept {
    return mybitsetT<T>(~data);
  }

  const mybitsetT<T> operator& (const mybitsetT<T>& rhs) const noexcept {
    return mybitsetT<T>(data & rhs.data);
  }

  mybitsetT<T>& operator&= (const mybitsetT<T>& rhs) noexcept {
    data &= rhs.data;
    return *this;
  }

  const mybitsetT<T> operator| (const mybitsetT<T>& rhs) const noexcept {
    return mybitsetT<T>(data | rhs.data);
  }

  mybitsetT<T>& operator|= (const mybitsetT<T>& rhs) noexcept {
    data |= rhs.data;
    return *this;
  }

  const mybitsetT<T> operator<< (size_t pos) const noexcept {
    return mybitsetT<T>(data << pos);
  }

  mybitsetT<T>& operator<<= (size_t pos) noexcept {
    data <<= pos;
    return *this;
  }

  const mybitsetT<T> operator>> (size_t pos) const noexcept {
    return mybitsetT<T>(data >> pos);
  }

  mybitsetT<T>& operator>>= (size_t pos) noexcept {
    data >>= pos;
    return *this;
  }

  const bool operator== (const mybitsetT<T>& rhs) const noexcept {
    return data == rhs.data;
  }

  const uint64_t to_ullong() const {
    return (uint64_t) data;
  }

  void set(const int i) {
    data |= ((T) 1 << i);
  }

  void set(const int i, const bool value) {
    data ^= (-value ^ data) & ((T) 1 << i);
  }

  bool test(const int i) const {
    return (data >> i) & (T) 1;
  }

  bool any() const {
    return data != 0;
  }

  // TODO: specialize for smaller T
  const int getFirstSetBit() const {
    //return countr_zero(data);
    return clz_u128(data);
  }

  // TODO: specialize for smaller T
  const int popcount() const {
    uint64_t hi = data >> 64;
    uint64_t lo = data;
    return __builtin_popcountll(lo) + __builtin_popcountll(hi);
  }
};

typedef mybitsetT<__uint128_t> mybitset;
typedef mybitsetT<__uint16_t> twoRowsBitset;
typedef mybitsetT<__uint8_t> oneRowBitset;

const mybitset firstTwoRowsSizeBig { (1 << twoRowsCellCount) - 1 };

class Bitboard {
public:
  // Most Significant Bit is the last bit, map lower right, index cellCount - 1, rightmost bit.
  // Least Significant Bit is the first bit, map upper left, index 0, leftmost bit.
  // To shift one row down, shift right: << colCount.
  // To shift one row up, shift left: >> colCount.

  mybitset board;

  void initRandom() {
    for (int i = 0; i < cellCount; i++) {
      bool val = rand() % 2 == 0;
      board.set(i, val);
    }
  }

  int getIndex(int x, int y) {
    return x + y * colCount;
  }

  bool get(int x, int y) {
    return board.test(getIndex(x, y));
  }

  bool test(int x, int y) {
    return board.test(getIndex(x, y));
  }

  void set(int x, int y, bool value) {
    board.set(getIndex(x, y), value);
  }

  inline void print() {
    if constexpr (DEBUG) {
      cerr << "BOARD:" << endl;
      for (int y = 0; y < rowCount; y++) {
        for (int x = 0; x < colCount; x++) {
          cerr << (get(x, y) ? 1 : 0) << " ";
        }
        cerr << endl;
      }
      cerr << endl;
    }
  }
};

class BitboardComponentTable {
public:
  // Lookup for first row will be just setting row A to zero and row B to first row
  uint8_t lookupTable2RowsIndex[lookupTableSizeFor2Rows][maxComponentsPerRow];
  oneRowBitset lookupTable2RowsComponent[lookupTableSizeFor2Rows][maxComponentsPerRow];

  BitboardComponentTable() {

    const int nbCount = 4;
    const int maxQueue = nbCount * twoRowsCellCount;
    int queue[maxQueue];

    for (int i = 0; i < lookupTableSizeFor2Rows; i++) {
      for (int componentIndex = 0; componentIndex < maxComponentsPerRow; componentIndex++) {
        lookupTable2RowsIndex[i][componentIndex] = 0;
        lookupTable2RowsComponent[i][componentIndex] = 0;
      }
    }

    const auto& findComponents = [this, &queue]
      (twoRowsBitset& remaining,
       int startIndex,
       int& componentIndex,
       uint64_t twoRows
      )
    {
      twoRowsBitset ret;
      if (!remaining.test(startIndex)) {
        return ret;
      } else {
        componentIndex++;
        if (componentIndex >= maxComponentsPerRow) {
          if constexpr (DEBUG) {
            throw new out_of_range("componentIndex");
          }
        }
      }
      for (int qi = 0; qi < maxQueue; qi++) {
        queue[qi] = 0;
      }

      int qIndex = 0;
      int qlen = 1;
      queue[qIndex] = startIndex;
      while (qlen > 0 && qIndex < maxQueue) {
        int topBitIndex = queue[qIndex];
        qIndex++;
        qlen--;

        if (!remaining.test(topBitIndex)) {
          continue;
        }
        remaining.set(topBitIndex, false);
        ret.set(topBitIndex);

        if (startIndex >= colCount && topBitIndex >= colCount) {
          lookupTable2RowsComponent[twoRows][componentIndex].set(topBitIndex - colCount);
        }

        int nbs[nbCount];
        nbs[0] = ((topBitIndex + 1) != colCount) ? topBitIndex + 1 : -1;
        nbs[1] = (topBitIndex != colCount) ? topBitIndex - 1 : -1;
        nbs[2] = topBitIndex + colCount;
        nbs[3] = topBitIndex - colCount;

        for (int nbi = 0; nbi < nbCount; nbi++) {
          int nb = nbs[nbi];
          if (0 <= nb && nb < twoRowsCellCount && remaining.test(nb)) {
            if (qIndex + qlen < maxQueue) {
              queue[qIndex + qlen] = nb;
              qlen++;
            } else {
              if constexpr (DEBUG) {
                throw new out_of_range("qIndex + qlen");
              }
            }
          }
        }
      }

      return ret;
    };

    for (uint16_t twoRows = 0; twoRows < lookupTableSizeFor2Rows; twoRows++) {
      // For the first row we are interested first: if it continues or ends.
      // If it continues, we are interested in which bits to add to it.
      twoRowsBitset remaining { twoRows };

      twoRowsBitset firstRemaining { twoRows };
      int firstRowComponentIndex = -1;
      twoRowsBitset oldComponents[3] = {0, 0, 0};
      for (int startIndex = 0; startIndex < colCount; startIndex++) {
        // For every space we reset so we can find next component
        if (!remaining.test(startIndex)) {
          firstRemaining = remaining;
        }
        twoRowsBitset oldComponent =
          findComponents(firstRemaining, startIndex, firstRowComponentIndex, twoRows);
        if (oldComponent.any()) {
          oldComponents[firstRowComponentIndex] = oldComponent;
        }
      }

      // If it ends, we are interested in what bits replace it.
      twoRowsBitset secondRemaining { twoRows };
      int secondRowComponentIndex = -1;
      for (int startIndex = colCount; startIndex < twoRowsCellCount; startIndex++) {
        // For every space we reset so we can find next component
        if (!remaining.test(startIndex)) {
          secondRemaining = remaining;
        }
        twoRowsBitset newComponent =
          findComponents(secondRemaining, startIndex, secondRowComponentIndex, twoRows);
        if (newComponent.any()) {
          for (int i = 0; i < maxComponentsPerRow; i++) {
            if (newComponent == oldComponents[i]) {
              lookupTable2RowsIndex[twoRows][secondRowComponentIndex] |= (1 << i);
            }
          }
        }
      }
    }
  }
};

BitboardComponentTable& bitboardComponentTable = *(new BitboardComponentTable());

class BitboardComponents {
public:
  int componentCount = 0;
  Bitboard components[maxComponents];

  const static int nbCount = 4;
  const static int maxQueue = nbCount * cellCount;
  int queue[maxQueue];

  // This is classic BFS, not needed, just for comparison.
  mybitset findComponents(
    mybitset& remaining,
    int startIndex,
    int& componentIndex
    ) {
    mybitset ret;
    if (!remaining.test(startIndex)) {
      return ret;
    } else {
      componentIndex++;
    }
    for (int qi = 0; qi < maxQueue; qi++) {
      queue[qi] = 0;
    }

    int qIndex = 0;
    int qlen = 1;
    queue[qIndex] = startIndex;
    while (qlen > 0 && qIndex < maxQueue) {
      int topBitIndex = queue[qIndex];
      qIndex++;
      qlen--;

      if (!remaining.test(topBitIndex)) {
        continue;
      }
      remaining.set(topBitIndex, false);
      ret.set(topBitIndex);

      int nbs[nbCount];
      nbs[0] = ((topBitIndex + 1) % colCount != 0) ? topBitIndex + 1 : -1;
      nbs[1] = ((topBitIndex % colCount) != 0) ? topBitIndex - 1 : -1;
      nbs[2] = topBitIndex + colCount;
      nbs[3] = topBitIndex - colCount;

      for (int nbi = 0; nbi < nbCount; nbi++) {
        int nb = nbs[nbi];
        if (0 <= nb && nb < cellCount && remaining.test(nb)) {
          if (qIndex + qlen < maxQueue) {
            queue[qIndex + qlen] = nb;
            qlen++;
          } else {
            if constexpr (DEBUG) {
              throw new out_of_range("qIndex + qlen");
            }
          }
        }
      }
    }

    return ret;
  }

  // This is classic BFS, not needed, just for comparison.
  void GetComponentsBFS(Bitboard board) {
    mybitset firstRemaining { board.board };
    int firstRowComponentIndex = -1;
    while (firstRemaining.any()) {
      int startIndex = firstRemaining.getFirstSetBit();
      mybitset oldComponent =
        findComponents(firstRemaining, startIndex, firstRowComponentIndex);
      if (oldComponent.any()) {
        components[firstRowComponentIndex].board = oldComponent;
        componentCount = firstRowComponentIndex + 1;
      }
    }
  }

  void GetComponents(const Bitboard& board) {
    components[0].board = 0;
    componentCount = 1;

    int currentComponents[maxComponentsPerRow];
    for (int i = 0; i < maxComponentsPerRow; i++) {
      currentComponents[i] = 0;
    }

    // Make row least significant colCount bits zero, the rest set to board.board. I.e. top row is zero.
    mybitset rows { board.board.boundsLimit() };
    rows <<= colCount;

    int y = 0;
    while (rows.any()) {
      const uint64_t bits = (rows & firstTwoRowsSizeBig).to_ullong();
      rows >>= colCount;
      if ((bits >> colCount) == 0) {
        // Terminate components if second row is empty
        for (int c = 0; c < maxComponentsPerRow; c++) {
          currentComponents[c] = 0;
        }
        y++;
        continue;
      }

      int oldComponents[maxComponentsPerRow];
      for (int c = 0; c < maxComponentsPerRow; c++) {
        oldComponents[c] = currentComponents[c];
      }

      for (int c = 0; c < maxComponentsPerRow; c++) {
        const uint32_t componentIndex = bitboardComponentTable.lookupTable2RowsIndex[bits][c];
        const Bitboard componentChange { bitboardComponentTable.lookupTable2RowsComponent[bits][c].data };
        
        currentComponents[c] = 0;

        if (componentIndex > 0) {
          // Continue old components
          for (int i = 0; i < maxComponentsPerRow; i++) {
            const bool copyOldComponent = (componentIndex >> i) & 1UL;
            if (copyOldComponent) {
              
              if (currentComponents[c] <= 0) {
                currentComponents[c] = oldComponents[i];
              } else if (currentComponents[c] != oldComponents[i]) {
                components[currentComponents[c]].board |= components[oldComponents[i]].board;

                // Merge                
                components[oldComponents[i]].board = 0;
                int mergeComponent = oldComponents[i];
                for (int j = 0; j < maxComponentsPerRow; j++) {
                  if (oldComponents[j] == mergeComponent) {
                    oldComponents[j] = currentComponents[c];
                  }
                  if (currentComponents[j] == mergeComponent) {
                    currentComponents[j] = currentComponents[c];
                  }
                }

              }
            }
          }
          components[currentComponents[c]].board |= componentChange.board << (y * colCount);
        } else if (componentChange.board.any()) {
          // Create new component
          currentComponents[c] = componentCount;
          components[currentComponents[c]].board = 0;
          componentCount++;
          components[currentComponents[c]].board |= componentChange.board << (y * colCount);
        } else {
          // Terminate component
          currentComponents[c] = 0;
        }
      }
      y++;
    }
  }
};

uint64_t x { 1 }; /* The state can be seeded with any value. */
uint64_t next()
{
    uint64_t z = (x += UINT64_C(0x9E3779B97F4A7C15));
    z = (z ^ (z >> 30)) * UINT64_C(0xBF58476D1CE4E5B9);
    z = (z ^ (z >> 27)) * UINT64_C(0x94D049BB133111EB);
    return z ^ (z >> 31);
}

class BitboardColors {
public:
  Bitboard boards[colorCount];
  bool valid = false;

  void initRandom() {
    // This will initialize probably around 5-15 bits, experimentally found. I think it's 1/(2^3) = 1/8 prob for each bit of the 72, so 9 expected.
    __uint128_t r1 = (next()) | ((((__uint128_t) next()) << twoRowsCellCount) & ~(((__uint128_t) (1) << 64) - 1));
    __uint128_t r2 = (next()) | ((((__uint128_t) next()) << twoRowsCellCount) & ~(((__uint128_t) (1) << 64) - 1));
    __uint128_t r3 = (next()) | ((((__uint128_t) next()) << twoRowsCellCount) & ~(((__uint128_t) (1) << 64) - 1));
    __uint128_t r = r1 & r2 & r3 & (((__uint128_t) (1) << cellCount) - 1);
    // cerr << "COUNT: " << mybitset(r).popcount() << endl;
    boards[1].board = r;
  }
};

int main() {
  const int boardCount = 10;
  BitboardColors boards[boardCount];
  for (int i = 0; i < boardCount; i++) {
    boards[i].initRandom();
  }

  const uint64_t maxi = 1000000;

  // First game loop: MUST uncomment to test perf on CodinGame, to read input data, or perf is very low.
//   for (int i = 0; i < 8; i++) {
//       int colorA; // color of the first block
//       int colorB; // color of the attached block
//       cin >> colorA >> colorB; cin.ignore();
//   }
//   int score1;
//   cin >> score1; cin.ignore();
//   for (int i = 0; i < 12; i++) {
//       string row; // One line of the map ('.' = empty, '0' = skull block, '1' to '5' = colored block)
//       cin >> row; cin.ignore();
//   }
//   int score2;
//   cin >> score2; cin.ignore();
//   for (int i = 0; i < 12; i++) {
//       string row;
//       cin >> row; cin.ignore();
//   }

  int i = 0;
  BitboardComponents bc;
  bool done = false;
  uint64_t bits = 0;
  auto start = myClock::now();

  for (int i = 0; i < maxi; i++) {
    // boards.initRandom();
    bc.GetComponents(boards[next() % boardCount].boards[1]);
    bits ^= bc.components[0].board.to_ullong();
  }
  const auto elapsed2 =
    std::chrono::duration_cast<time_interval_t>(myClock::now() - start);
  cerr << "advanced elapsed: " << elapsed2.count() << ", iterations: " << maxi << ", iterations/100ms: " << 100000 * maxi / elapsed2.count() << endl;

  start = myClock::now();
  for (int i = 0; i < maxi; i++) {
    // boards.initRandom();
    bc.GetComponentsBFS(boards[next() % boardCount].boards[1]);
    bits ^= bc.components[0].board.to_ullong();
  }
  const auto elapsed3 =
    std::chrono::duration_cast<time_interval_t>(myClock::now() - start);
  cerr << "normal elapsed: " << elapsed3.count() << ", iterations: " << maxi << ", iterations/100ms: " << 100000 * maxi / elapsed3.count() << endl;
  cerr << bits << endl;

  // "x rotation": the column in which to drop your pair of blocks followed by its rotation (0, 1, 2 or 3)
  cout << "0 1" << endl;

  return 0;
}
```

# Advanced usage

If you want a more complex example (external libraries, viewers...), use the [Advanced C++ template](https://tech.io/select-repo/598)
