```c
int age = 20; // integers
float temperature = 25.5f; // less precise decimal
double pi = 3.1415926; // precise decimals
char grade = 'A'; // single character, single quotes
```

```c
const double TAX = 0.2; // cannot be changed later
```

C has no string type, you use an array of chars

```c
#include <stdio.h>

int main()
{
    char name[] = "John";      // size worked out for you, ends with '\0'
    char city[20] = "London";  // room for 19 chars + terminator

    printf("%s from %s\n", name, city);
}
```
