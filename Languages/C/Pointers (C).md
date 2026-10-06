A pointer holds the memory address of something else rather than a value.

```c
/*
1. & gives you the address of a variable
2. * in a declaration means "this is a pointer"
3. * on an existing pointer dereferences it, gets the value at that address
*/

#include <stdio.h>

int main()
{
    int age = 20;
    int *ptr = &age;      // ptr holds the address of age

    printf("%d\n", age);   // 20  the value
    printf("%p\n", &age);  // address of age
    printf("%p\n", ptr);   // same address
    printf("%d\n", *ptr);  // 20  the value at that address

    *ptr = 30;             // write through the pointer
    printf("%d\n", age);   // 30
}
```

```c
// a pointer that points at nothing should be NULL, and check before using it

int *ptr = NULL;

if (ptr != NULL)
{
    printf("%d\n", *ptr);
}
```

Pointer arithmetic moves by the size of the type, not by 1 byte

```c
#include <stdio.h>

int main()
{
    int nums[3] = {10, 20, 30};
    int *ptr = nums;      // array name already is the address of the first element

    printf("%d\n", *ptr);        // 10
    printf("%d\n", *(ptr + 1));  // 20, jumped 4 bytes not 1
    printf("%d\n", *(ptr + 2));  // 30

    ptr++;
    printf("%d\n", *ptr);        // 20
}
```

```c
// arrays and pointers are basically the same thing here

#include <stdio.h>

void printAll(int *arr, int length)
{
    for (int i = 0; i < length; i++)
    {
        printf("%d\n", arr[i]);   // arr[i] is the same as *(arr + i)
    }
}

int main()
{
    int nums[3] = {10, 20, 30};

    printAll(nums, 3);   // arrays decay to a pointer, so length must be passed too
}
```
