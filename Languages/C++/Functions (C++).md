```cpp
#include <iostream>

void greet() {
    
    std::cout << "Hello";
}

int main () {
	
	greet();
}

// output: Hello
```

```cpp
/*
- Function gets age and returns it 
- Declare variable in main called age to be able to use the returned age from the age function
- Becuase age has been declared matching to the function it will be able to show it to the screen
*/

#include <iostream>

int get_age() {
	
	int age;
	
	std::cout << "What is your age";
	std::cin >> age;
	
	return age;
}

int main () {
	
	int age  = get_age();
	
	std::cout << "Your age is " << age << std::endl;
}
```

```cpp
/* 
1. returns an int 
2. that int is passed down into another function
3. that function outputs it to the screen
*/

#include <iostream>


int getBalance()
{
    return 500;
}

void displayBalance(int balance)
{
    std::cout << balance;
}

int main() {
	
	displayBalance(getBalance());

}
```

```cpp
// a function which takes in a string as an argument and outputs that to screen 

#include <iostream>
#include <string>

void greet(std::string name)
{
    std::cout << "Hello " << name << std::endl;
}

int main()
{
    std::string user;
    
    std::cout << "What is your name: ";
    
    std::getline(std::cin, user);
    

    greet(user);
}
```

```cpp
#include <iostream>

int* createNumber()
{
    int* ptr = new int(10);
    return ptr;
}

void changeValue(int* ptr)
{
    *ptr = 50;
}

int main()
{
    int* myPtr = createNumber();   // gets heap memory with value 10

    std::cout << *myPtr << std::endl;  // 10

    changeValue(myPtr);            // modify through pointer

    std::cout << *myPtr << std::endl;  // 50

    delete myPtr;                  // IMPORTANT: free memory
    myPtr = nullptr;
}
```