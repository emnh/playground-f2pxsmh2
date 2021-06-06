# Fast Connected Components for 6x12 bitboard

This optimized code was made for Smash the Code contest. We assume that you are using one 6x12 bitboard per color,
represented as the least significant bits of an 128 bit integer (__128int_t), which is supported by GCC.
You can expect about say 800k-1M separations of one color into connected components,
versus say 200k-250k for a regular bit-BFS per index.

The main idea is to create a lookup table for the transition between 2 rows.
This allows you to iterate over the rows 2 by 2 and connect the components already computed in the table.

For a row of 6 columns, the maximum number of components is 3, for example 101010, 010101 or 101101.
You can see this because each component must be separated by a space (0),
so 4 components (1) have 3 separators (0), which makes 7,
which is greater than the columns, 6, so there are at most 3 components.

This means we can keep 3 running active components and seal components as soon as they finish.
There is a caveat to this, in that one running component can terminate while another running component keeps it active.
This happens for example with this configuration:
111
101
001
001

How many components can there be total? 72 / 2 = 36, using more or less the same logic as before, with a checkerboard pattern.
We also save one "component" number 0 for inactive components, the background, so let it be 37 total.

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

Say for example:
101010
111001

Here we have input components (notice the first and second component are equal):
101000 101000 000010
111000 111000 000001

And the output components are:
101000 000000 000000
111000 000001 000000

So since the first output component is equal to the first and second input component we store index 011 for the first output component to indicate this.
So tableIndex[101010111001][0] = 011.
And tableRow[101010111001][0] = 111000.

Further, the two other components do not continue an input component, so we have:
So tableIndex[101010111001][1] = 000.
And tableRow[101010111001][1] = 000001.

And for the last we have:
So tableIndex[101010111001][2] = 000.
And tableRow[101010111001][2] = 000000.

To compute the components of the table you can use regular bit-BFS on each index which matches pattern "^1|01" in the 2 rows.

Once you have the table initialized, the iteration proceeds as follows:
First you prepend an empty row to your grid, so that the transition into the components of the first row is easy and fits into the loop.
Start with the top two rows (row 0 and row 1) of the grid, then lookup that in the table.
Continue by shifting down one row and looking up row 1 and row 2 in the table.

First, save oldComponents[3] = runningComponents[3] and do runningComponents[3] = 0;
For each row looked up, there are 3 cases: continue component, new component and terminate component.
If the tableIndex is > 0, we continue, else if tableRow > 0, new component, and else terminate component.

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

For a new running component, allocate a new component index and set the indexed component to the tableRow.
runningComponent[0] = componentCount++, components[runningComponent[i]] = tableRow.

For terminating a running component, just set its index to 0: runningComponent[i] = 0.


```C++ runnable
#include <iostream>

using namespace std;

int main() 
{
    cout << "Complete code will be forthcoming!";
    return 0;
}
```

# Advanced usage

If you want a more complex example (external libraries, viewers...), use the [Advanced C++ template](https://tech.io/select-repo/598)
