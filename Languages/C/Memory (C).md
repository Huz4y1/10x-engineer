Stack memory is automatic and cleaned up when the function ends, heap memory you ask for yourself and have to give back.

|Stack|Heap|
|---|---|
|`int x = 5;`|`malloc`|
|Freed automatically|You must call free|
|Fixed size, known at compile time|Size can be decided while running|
|Fast|Slower|

```c
/*
1. malloc asks for a number of bytes and hands back a pointer
2. it can fail, so always check for NULL
3. free hands the memory back when you're done
*/

#include <stdio.h>
#include <stdlib.h>

int main()
{
    int *nums = malloc(5 * sizeof(int));   // room for 5 ints

    if (nums == NULL)
    {
        printf("Out of memory\n");
        return 1;
    }

    for (int i = 0; i < 5; i++)
    {
        nums[i] = i * 10;
    }

    printf("%d\n", nums[3]);   // 30

    free(nums);
    nums = NULL;               // stops you using a dangling pointer
}
```

calloc does the same but zeroes everything for you

```c
int *nums = calloc(5, sizeof(int));   // count, size  -> all set to 0
```

realloc resizes a block you already have

```c
#include <stdlib.h>

int main()
{
    int *nums = malloc(5 * sizeof(int));

    int *bigger = realloc(nums, 10 * sizeof(int));

    if (bigger == NULL)
    {
        free(nums);      // realloc failed, original is still valid
        return 1;
    }

    nums = bigger;

    free(nums);
}
```

```c
// the classic leak, memory asked for and never given back

void leaky()
{
    int *nums = malloc(100 * sizeof(int));

    // do some work

    return;   // pointer disappears, the 400 bytes are lost until the program ends
}

// every malloc/calloc/realloc needs exactly one matching free
```
