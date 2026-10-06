```cpp
// Writing to a file

#include <fstream>

int main()
{
std::ofstream file("data.txt");

file << "Hello world\n";
file << "Balance: 100\n";

file.close();
}
```

```cpp
// Reading from a file

#include <fstream>
#include <iostream>
#include <string>

int main()
{
    std::ifstream file("data.txt");

    std::string line;

    while (std::getline(file, line))
    {
        std::cout << line << std::endl;
    }
}
```

```cpp
#include <fstream>

//Appending 
std::ofstream file("data.txt", std::ios::app);

file << "New transaction\n";
```