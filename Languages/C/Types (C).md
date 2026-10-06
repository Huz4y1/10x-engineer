Common types:

|Type|Meaning|Format specifier|
|---|---|---|
|int|Whole number|%d|
|long|Bigger whole number|%ld|
|short|Small whole number|%hd|
|unsigned int|Whole number, no negatives|%u|
|float|Decimal|%f|
|double|More precise decimal|%lf|
|char|Single character|%c|
|char[]|String (array of char)|%s|

```c
int count = 10;
long population = 8000000000L;
short small = 5;
unsigned int lives = 3;      // 0 and above only
float price = 9.99f;
double distance = 384400.55;
char letter = 'C';
```

sizeof tells you how many bytes a type takes

```c
#include <stdio.h>

int main()
{
    printf("int    %zu bytes\n", sizeof(int));
    printf("float  %zu bytes\n", sizeof(float));
    printf("double %zu bytes\n", sizeof(double));
    printf("char   %zu bytes\n", sizeof(char));

    // sizes depend on the machine/compiler, not fixed by the language
}
```
