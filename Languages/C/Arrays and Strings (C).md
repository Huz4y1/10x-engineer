```c
#include <stdio.h>

int main()
{
    int scores[5] = {10, 20, 30, 40, 50};

    printf("%d\n", scores[0]);   // 10, indexes start at 0
    printf("%d\n", scores[4]);   // 50

    scores[2] = 99;              // change one

    // C does not check bounds, scores[9] compiles and quietly breaks things
}
```

Looping an array

```c
#include <stdio.h>

int main()
{
    int scores[5] = {10, 20, 30, 40, 50};

    int length = sizeof(scores) / sizeof(scores[0]);  // total bytes / one element

    for (int i = 0; i < length; i++)
    {
        printf("%d\n", scores[i]);
    }
}
```

```c
/*
1. a string is just a char array ending in the null terminator '\0'
2. "John" is 4 letters but takes 5 slots
3. that terminator is how the string functions know where to stop
*/

#include <stdio.h>

int main()
{
    char name[] = "John";

    printf("%c\n", name[0]);  // J
    printf("%s\n", name);     // John
}
```

string.h gives you the tools for working with them

```c
#include <stdio.h>
#include <string.h>

int main()
{
    char name[20] = "John";
    char copy[20];

    printf("%zu\n", strlen(name));       // 4, does not count the '\0'

    strcpy(copy, name);                  // copy name into copy
    printf("%s\n", copy);                // John

    strcat(copy, " Smith");              // stick on the end
    printf("%s\n", copy);                // John Smith

    if (strcmp(name, "John") == 0)       // 0 means they match, == won't work
    {
        printf("Same string\n");
    }
}
```
