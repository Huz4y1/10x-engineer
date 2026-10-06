While loop

```c
#include <stdio.h>

int main()
{
    int count = 0;

    while (count < 5)
    {
        printf("%d\n", count);
        count++;
    }
}
```

Do while, runs the body at least once before checking

```c
int number;

do
{
    printf("Enter a number above 10: ");
    scanf("%d", &number);
}
while (number <= 10);
```

For loop

```c
for (int i = 0; i < 5; i++)
{
    printf("%d\n", i);
}
```

```c
// break leaves the loop, continue skips to the next go around

for (int i = 0; i < 10; i++)
{
    if (i == 3)
    {
        continue;   // skip printing 3
    }

    if (i == 6)
    {
        break;      // stop the loop entirely
    }

    printf("%d\n", i);
}

// output: 0 1 2 4 5
```
