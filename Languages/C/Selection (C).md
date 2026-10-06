```c
#include <stdio.h>

int main()
{
    int score = 72;

    if (score >= 70)
    {
        printf("Distinction\n");
    }
    else if (score >= 50)
    {
        printf("Pass\n");
    }
    else
    {
        printf("Fail\n");
    }
}
```

C has no bool by default, 0 is false and anything else is true

```c
int loggedIn = 1;

if (loggedIn)
{
    printf("Welcome back\n");
}
```

Switch, don't forget break or it falls through to the next case

```c
#include <stdio.h>

int main()
{
    char choice = 'b';

    switch (choice)
    {
        case 'a':
            printf("Deposit\n");
            break;
        case 'b':
            printf("Withdraw\n");
            break;
        default:
            printf("Not an option\n");
    }
}
```
