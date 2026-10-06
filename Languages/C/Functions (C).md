```c
#include <stdio.h>

void greet()
{
    printf("Hello\n");
}

int main()
{
    greet();
}

// output: Hello
```

```c
/*
- Function returns an int
- main stores it in a variable so it can be used
*/

#include <stdio.h>

int getBalance()
{
    return 500;
}

int main()
{
    int balance = getBalance();

    printf("Balance: %d\n", balance);
}
```

Passing arguments in

```c
#include <stdio.h>

int add(int a, int b)
{
    return a + b;
}

int main()
{
    printf("%d\n", add(4, 6));  // 10
}
```

```c
/*
1. C passes arguments by value, the function gets a copy
2. so to change a caller's variable you pass its address
3. inside the function * gets at the original value
*/

#include <stdio.h>

void doubleIt(int *number)
{
    *number = *number * 2;
}

int main()
{
    int score = 10;

    doubleIt(&score);

    printf("%d\n", score);  // 20
}
```

If you define a function below main you need a prototype above it

```c
#include <stdio.h>

int add(int a, int b);   // prototype, just the signature and a semicolon

int main()
{
    printf("%d\n", add(2, 3));
}

int add(int a, int b)
{
    return a + b;
}
```
