printf uses format specifiers as placeholders for the values you pass in

```c
#include <stdio.h>

int main()
{
    int age = 20;
    float height = 1.75f;
    char grade = 'A';
    char name[] = "John";

    printf("Name: %s\n", name);
    printf("Age: %d\n", age);
    printf("Height: %.2f\n", height); // .2 = 2 decimal places
    printf("Grade: %c\n", grade);
}
```

Reading a number with scanf, note the & in front of the variable

```c
#include <stdio.h>

int main()
{
    int age;

    printf("What is your age: ");
    scanf("%d", &age);   // & gives scanf the address to write into

    printf("You are %d\n", age);
}
```

```c
/*
1. scanf with %s stops at the first space so it can't read a full line
2. fgets reads a whole line and lets you cap the size so you can't overflow
3. fgets keeps the newline, so strip it
*/

#include <stdio.h>
#include <string.h>

int main()
{
    char name[50];

    printf("What is your name: ");
    fgets(name, sizeof(name), stdin);

    name[strcspn(name, "\n")] = '\0';  // remove trailing newline

    printf("Hello %s\n", name);
}
```
