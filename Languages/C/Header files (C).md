Split a program up so the .h says what exists and the .c says how it works.

```c
// maths.h  -  the declarations

#ifndef MATHS_H
#define MATHS_H

int add(int a, int b);
int multiply(int a, int b);

#endif

/*
1. the include guard stops the header being pasted in twice
2. first time through MATHS_H isn't defined, so it gets defined and the body runs
3. second time it's already defined so everything is skipped
*/
```

```c
// maths.c  -  the definitions

#include "maths.h"

int add(int a, int b)
{
    return a + b;
}

int multiply(int a, int b)
{
    return a * b;
}
```

```c
// main.c  -  uses them

#include <stdio.h>
#include "maths.h"     // quotes for your own files, <> for library ones

int main()
{
    printf("%d\n", add(2, 3));
    printf("%d\n", multiply(4, 5));
}
```

Compile every .c file together, headers don't get compiled

```c
// gcc main.c maths.c -o program
// ./program
```

You can also use `#pragma once` at the top of the header instead of the guard, shorter but not standard.
