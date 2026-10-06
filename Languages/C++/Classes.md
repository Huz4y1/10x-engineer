Use classes when something has data and behaviour. For example:

|Class|Data|Behaviour|
|---|---|---|
|BankAccount|balance|deposit()|
|Student|name|display()|
|Car|speed|accelerate()|
|InventoryItem|quantity|updateStock()|

```cpp
#include <iostream>

class BankAccount
{
private:
    double balance;

public:

    void deposit(double amount)
    {
        balance += amount;
    }

    void showBalance()
    {
        std::cout << balance;
    }
};



BankAccount account;

account.deposit(100);

account.showBalance();
```

```cpp
#include <iostream>
#include <string>

class BankAccount
{
private:
    double balance;
    
public:
		
		std::string name;
		
    void deposit(double amount)
    {
        balance += amount;
    }

    void showBalance()
    {
        std::cout << balance;
    }
};



BankAccount account;

account.deposit(100);

account.showBalance();


int main() {
	
	BankAccount Account;
	
	std::string username;
	std::cout << "Enter your name";
	std::getline(std::cin, username);
	
	Account.name = username;
	
	std::cout << "Welcome " << Account.name;
	
}
```