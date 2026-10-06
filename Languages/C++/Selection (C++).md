```cpp
// Simple if, else if and else selection statements based on user input 

#include <iostream>

int age;

std::cout << "\nWhat is your age: ";
std::cin >> age;

if (age < 18) {
	std::cout << "You are not an adult";
}

else if (age < 0) {
	std::cout << "This is not a valid age";
}

else {
	std::cout << "You are an adult";
}

```

```cpp
/* 
- Simple switch case selection statements based on user input 
- Only allowed to use switch case on integers and single character strings
- Can't be used with decimals or full strings 
- Better to use if/else if/ else if user input is not int or char
*/

#include <iostream>

int choice;

std::cout << "1. Blue\n";
std::cout << "2. Orange\n";
std::cout << "3. Green\n";

std::cin >> choice;

switch(choice)
{
    case 1:
        std::cout << "Blue";
        break;

    case 2:
        std::cout << "Orange";
        break;

    case 3:
        std::cout << "Green";
        break;

    default:
        std::cout << "Invalid";
}
```