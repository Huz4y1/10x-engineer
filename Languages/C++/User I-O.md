```cpp
// Getting a string as user input
#include <iostream>
#include <string>

std::string name;

std::cout << "Enter name: ";
std::getline(std::cin, name); //only need getline() for strings 
```

```cpp
// Getting an integer as user input
#include <iostream>

int age;

std::cout << "Enter your age: ";
std::cin >> age; /*This works for every data type including single character 
string except for a full string */

```