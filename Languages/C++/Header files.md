Header files (.h) are files where you declare things like functions, classes and variables and use them later in the main.cpp file

- header (.h) declares things that you can use later
- source (.cpp) defines the logic that you declared in the header files
- main file uses it all

```cpp
//Header file called bank.h

#ifndef BANK_H
#define BANK_H

#include <string>

// Function declaration
void deposit(int& balance, int amount);

// Class declaration
class Account
{
public:
    std::string name;
    int balance;

    void showBalance();
};

#endif
```

```cpp
//Header source file called bank.cpp
#include <iostream>
#include "bank.h"

// function definition
void deposit(int& balance, int amount)
{
    balance += amount;
}

// class method definition
void Account::showBalance()
{
    std::cout << "Name: " << name << std::endl;
    std::cout << "Balance: " << balance << std::endl;
}
```

```cpp
//main file called main.cpp
#include <iostream>
#include "bank.h"

int main()
{
    int balance = 100;

    deposit(balance, 50);

    std::cout << "Balance: " << balance << std::endl;

    Account user;
    user.name = "John";
    user.balance = balance;

    user.showBalance();
}
```