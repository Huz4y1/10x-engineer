```cpp
//Catcheds all errors
#include <iostream>

int main()
{
    try
    {
        int x = 10;
        int y = 0;

        if (y == 0)
        {
            throw std::runtime_error("Division by zero!");
        }

        std::cout << x / y;
    }
    catch (std::exception& e)
    {
        std::cout << "Error: " << e.what() << std::endl;
    }
}
```

```cpp
#include <string>

// ValueError

#include <iostream>
#include <stdexcept>

int convertToInt(const std::string& text)
{
    if (text == "abc")
    {
        throw std::invalid_argument("Invalid number format");
    }

    return std::stoi(text);
}

int main()
{
    try
    {
        int x = convertToInt("abc");
    }
    catch (const std::invalid_argument& e)
    {
        std::cout << "Value error: " << e.what() << std::endl;
    }
}
```