```c
// Writing to a file, "w" wipes whatever was there before

#include <stdio.h>

int main()
{
    FILE *file = fopen("data.txt", "w");

    if (file == NULL)          // always check, the file might not open
    {
        printf("Could not open file\n");
        return 1;
    }

    fprintf(file, "Hello world\n");
    fprintf(file, "Balance: %d\n", 100);

    fclose(file);              // don't forget or the writes may not be saved
}
```

```c
// Reading from a file line by line

#include <stdio.h>

int main()
{
    FILE *file = fopen("data.txt", "r");

    if (file == NULL)
    {
        printf("Could not open file\n");
        return 1;
    }

    char line[100];

    while (fgets(line, sizeof(line), file) != NULL)
    {
        printf("%s", line);    // line already has its newline on the end
    }

    fclose(file);
}
```

```c
// Appending, "a" adds to the end instead of wiping

#include <stdio.h>

int main()
{
    FILE *file = fopen("data.txt", "a");

    fprintf(file, "New transaction\n");

    fclose(file);
}
```

Modes

|Mode|Does|
|---|---|
|"r"|Read, fails if the file doesn't exist|
|"w"|Write, creates or wipes the file|
|"a"|Append, creates if needed|
