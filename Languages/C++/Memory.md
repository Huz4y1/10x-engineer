  

Stack:

It’s like plates, the last plate to be placed is the last thing that gets removed

- use when values are simple
- calculations are temporary
- basic user input
- Stores local variables
- Stores function calls
- stores temporary small data
- memory is automatically removed
- very fast
- ends when a function ends

  

Heap:

The heap is used for memory that you allocate. You request the memory, you use it, you must give it back (freeing the memory)

- use the heap when size is unknown at run time
- Slower
- Larger memory sizes
- memory management is manual
- ends when you free it

  

# Allocating memory (The Heap)

Must use pointers when allocating memory because the pointers is how you will access the memory

```cpp
int* age_pointer = new int(20); // new pointer which contains 20

delete age_pointer; // freeing the memory
age_pointer = nullptr; // filling it with null

/* 
 - *age_pointer = data inside = 20
 - age_pointer = address of pointer 
*/
```

```cpp
#include <string>

std::string* name_ptr = new std::string("John"); 

delete name_ptr;
name_ptr = nullptr

/* 
 - *name_ptr = data inside = John
 - name_ptr = address of pointer 
*/
```