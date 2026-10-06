A struct groups related variables together under one name.

```c
#include <stdio.h>

struct Student
{
    char name[50];
    int age;
    double grade;
};

int main()
{
    struct Student s1 = {"John", 20, 78.5};

    printf("%s\n", s1.name);    // dot to get at a member
    printf("%d\n", s1.age);

    s1.age = 21;                // and to change one
}
```

typedef saves you writing struct every time

```c
#include <stdio.h>

typedef struct
{
    char name[50];
    int age;
} Student;

int main()
{
    Student s1 = {"John", 20};   // no "struct" needed now

    printf("%s is %d\n", s1.name, s1.age);
}
```

```c
/*
1. if you have a pointer to a struct, the dot won't work directly
2. (*ptr).age is ugly, so C gives you ->
3. this is how you pass a struct to a function without copying it
*/

#include <stdio.h>

typedef struct
{
    char name[50];
    int age;
} Student;

void haveBirthday(Student *s)
{
    s->age = s->age + 1;    // -> is shorthand for (*s).age
}

int main()
{
    Student s1 = {"John", 20};

    haveBirthday(&s1);

    printf("%d\n", s1.age);   // 21
}
```

Array of structs

```c
#include <stdio.h>

typedef struct
{
    char name[50];
    int age;
} Student;

int main()
{
    Student class[3] = {
        {"John", 20},
        {"Sara", 22},
        {"Ali", 19}
    };

    for (int i = 0; i < 3; i++)
    {
        printf("%s is %d\n", class[i].name, class[i].age);
    }
}
```
