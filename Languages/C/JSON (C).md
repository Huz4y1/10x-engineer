C has no JSON support at all, and no garbage collector, so every parsed document is memory you own and must free. The library everyone starts with is cJSON: one `.c` file, one `.h`, no dependencies.

Getting it

```bash
curl -O https://raw.githubusercontent.com/DaveGamble/cJSON/master/cJSON.c
curl -O https://raw.githubusercontent.com/DaveGamble/cJSON/master/cJSON.h

gcc main.c cJSON.c -o main
```

That is the whole install. See [[Header files (C)]].

The memory rule

This is the part that matters more than the API.

```
  cJSON_Parse()      allocates a whole tree      -> you must cJSON_Delete()
  cJSON_Print()      allocates a string          -> you must free()
  cJSON_GetObjectItem()  borrows a pointer       -> do NOT free
  item->valuestring  points INTO the tree        -> do NOT free, and it
                                                    dies with the tree
```

One `cJSON_Delete` on the root frees every child. Never free a child yourself. If you need a string to outlive the tree, `strdup` it.

Parsing

```c
/*
1. cJSON_Parse returns NULL on invalid input
2. GetObjectItem borrows - no free, and NULL if the key is missing
3. always check the type before reading the union member
4. one Delete at the end frees the entire tree
*/

#include <stdio.h>
#include <string.h>
#include "cJSON.h"

int main(void) {
    const char *text = "{\"name\":\"Huzayl\",\"age\":21,\"active\":true}";

    cJSON *root = cJSON_Parse(text);
    if (root == NULL) {
        const char *err = cJSON_GetErrorPtr();
        if (err) fprintf(stderr, "parse error near: %s\n", err);
        return 1;
    }

    cJSON *name = cJSON_GetObjectItemCaseSensitive(root, "name");
    if (cJSON_IsString(name) && name->valuestring != NULL) {
        printf("%s\n", name->valuestring);      // output: Huzayl
    }

    cJSON *age = cJSON_GetObjectItemCaseSensitive(root, "age");
    if (cJSON_IsNumber(age)) {
        printf("%d\n", age->valueint);          // output: 21
    }

    cJSON_Delete(root);      // frees name and age too
    return 0;
}
```

The check-then-read pattern is not optional. Reading `valuestring` on a number gives you a garbage pointer and a segfault.

The value union

```c
  item->valuestring     char*    when cJSON_IsString
  item->valueint        int      when cJSON_IsNumber
  item->valuedouble     double   when cJSON_IsNumber
  item->type                     the tag
```

Type checks:

```c
cJSON_IsString(item)   cJSON_IsNumber(item)   cJSON_IsBool(item)
cJSON_IsTrue(item)     cJSON_IsFalse(item)    cJSON_IsNull(item)
cJSON_IsArray(item)    cJSON_IsObject(item)   cJSON_IsInvalid(item)
```

Use `cJSON_GetObjectItemCaseSensitive`, not `cJSON_GetObjectItem`. The non-sensitive version is legacy and slower.

A helper worth writing

The check-every-time boilerplate gets old immediately. Write these once:

```c
/*
1. returns the default when the key is missing or the wrong type
2. this removes about 80% of the noise from real parsing code
*/

static const char *json_str(const cJSON *obj, const char *key, const char *def) {
    const cJSON *it = cJSON_GetObjectItemCaseSensitive(obj, key);
    return (cJSON_IsString(it) && it->valuestring) ? it->valuestring : def;
}

static int json_int(const cJSON *obj, const char *key, int def) {
    const cJSON *it = cJSON_GetObjectItemCaseSensitive(obj, key);
    return cJSON_IsNumber(it) ? it->valueint : def;
}

static int json_bool(const cJSON *obj, const char *key, int def) {
    const cJSON *it = cJSON_GetObjectItemCaseSensitive(obj, key);
    if (cJSON_IsTrue(it))  return 1;
    if (cJSON_IsFalse(it)) return 0;
    return def;
}

// usage
const char *name = json_str(root, "name", "unknown");
int age = json_int(root, "age", 0);
```

Arrays and filtering

There is no filter function. You loop and you build.

```c
/*
1. cJSON_ArrayForEach walks an array, borrowing each element
2. "filtering" means either acting inside the loop, or copying
   matching items into your own struct array
*/

const char *text =
    "[{\"name\":\"Ana\",\"age\":34},"
    " {\"name\":\"Bo\",\"age\":19},"
    " {\"name\":\"Cy\",\"age\":41}]";

cJSON *arr = cJSON_Parse(text);
if (!cJSON_IsArray(arr)) { cJSON_Delete(arr); return 1; }

// simple: act on matches as you go
cJSON *item = NULL;
cJSON_ArrayForEach(item, arr) {
    int age = json_int(item, "age", 0);
    if (age > 30) {
        printf("%s is %d\n", json_str(item, "name", "?"), age);
    }
}

// by index
int n = cJSON_GetArraySize(arr);
for (int i = 0; i < n; i++) {
    cJSON *it = cJSON_GetArrayItem(arr, i);
    printf("%s\n", json_str(it, "name", "?"));
}

cJSON_Delete(arr);
```

Copying into your own structs

For anything beyond a quick print, get out of the JSON tree and into real C structs. See [[Structs (C)]].

```c
/*
1. count matches first so you can allocate exactly once
2. strdup the strings - they must outlive cJSON_Delete
3. free both the strings and the array when done
*/

typedef struct {
    char *name;
    int   age;
} User;

cJSON *arr = cJSON_Parse(text);

int total = cJSON_GetArraySize(arr);
User *users = malloc(sizeof(User) * total);
int count = 0;

cJSON *item = NULL;
cJSON_ArrayForEach(item, arr) {
    int age = json_int(item, "age", 0);
    if (age <= 30) continue;                  // the filter

    users[count].name = strdup(json_str(item, "name", ""));
    users[count].age  = age;
    count++;
}

cJSON_Delete(arr);      // tree gone, our strdup'd copies survive

for (int i = 0; i < count; i++) {
    printf("%s %d\n", users[i].name, users[i].age);
}

// cleanup
for (int i = 0; i < count; i++) free(users[i].name);
free(users);
```

That `strdup` is the single most important line. Without it every `name` becomes a dangling pointer the moment you delete the tree, and it will appear to work in testing.

See [[Memory (C)]].

Sorting

Once in a struct array, `qsort` from the standard library:

```c
static int by_age(const void *a, const void *b) {
    const User *x = a, *y = b;
    return (x->age > y->age) - (x->age < y->age);
}

qsort(users, count, sizeof(User), by_age);
```

Building JSON

```c
/*
1. AddItemToObject TRANSFERS ownership to the parent
2. so you only Delete the root
3. Print allocates - you free that string yourself
*/

cJSON *root = cJSON_CreateObject();
cJSON_AddStringToObject(root, "name", "Huzayl");
cJSON_AddNumberToObject(root, "age", 21);
cJSON_AddBoolToObject(root, "active", 1);

cJSON *tags = cJSON_CreateArray();
cJSON_AddItemToArray(tags, cJSON_CreateString("rust"));
cJSON_AddItemToArray(tags, cJSON_CreateString("hardware"));
cJSON_AddItemToObject(root, "tags", tags);      // root now owns tags

cJSON *addr = cJSON_CreateObject();
cJSON_AddStringToObject(addr, "city", "London");
cJSON_AddItemToObject(root, "address", addr);

char *out = cJSON_Print(root);            // pretty, allocates
// char *out = cJSON_PrintUnformatted(root);   // compact

printf("%s\n", out);

free(out);            // the string
cJSON_Delete(root);   // the tree
```

Two separate frees, for two separate allocations. Forgetting the `free(out)` is the most common leak in cJSON code.

Reading a file

```c
#include <stdlib.h>

char *read_file(const char *path) {
    FILE *f = fopen(path, "rb");
    if (!f) return NULL;

    fseek(f, 0, SEEK_END);
    long len = ftell(f);
    fseek(f, 0, SEEK_SET);

    char *buf = malloc(len + 1);
    if (!buf) { fclose(f); return NULL; }

    fread(buf, 1, len, f);
    buf[len] = '\0';
    fclose(f);
    return buf;
}

char *text = read_file("samples/users.json");
cJSON *root = cJSON_Parse(text);
free(text);              // cJSON copied what it needed, safe to free now
// ... use root ...
cJSON_Delete(root);
```

See [[File handling (C)]].

Checking for leaks

```bash
gcc -g -fsanitize=address main.c cJSON.c -o main && ./main

# or
valgrind --leak-check=full ./main
```

Run this on any JSON code you write in C. The mistakes are all invisible until they are not.

Big integers

`valueint` is an `int`, so it overflows past 2^31. For 64-bit ids read `valuedouble` (exact to 2^53) or parse `valuestring` with `strtoll` if the API sends ids as strings, which good ones do. See [[JSON]].

Other libraries

```
  cJSON        simplest, most common, fine for config and API responses
  jansson      nicer API, reference counted, needs linking
  parson       even smaller than cJSON
  yajl         streaming/SAX, for documents too big for memory
```

Start with cJSON. Move to yajl only if you need to parse something that will not fit in RAM.
