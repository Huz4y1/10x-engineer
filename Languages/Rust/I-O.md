basic input for text from a user .

We need to create an empty mutable string

we print a message to the screen to tell the user what to do so they can have a field to type in

We then need to read the input

if the input fails we need to handle the error

we then trim the endline because when the line is read the endline is included

we then print the name to the screen of what the user inputted

  

```rust
use std::io;

fn main() {
    let mut name = String::new();
    
     println!("Enter your name:");

    io::stdin()
        .read_line(&mut name)
        .expect("Failed to read line");
        
    let name = name.trim();


    println!("You entered: {}", name);
}
```

  

taking an integer as user input

does the same thing but when its read rust converts it into an i32

```rust
use std::io;

fn main() {
    let mut age = String::new();

    io::stdin()
        .read_line(&mut age)
        .expect("Failed");

    let age: i32 = age.trim().parse().unwrap();

    println!("Age is {}", age);
}
```

  

Taking in multiple user inputs

```rust
use std::io;

fn main() {
    let mut name = String::new();
    let mut age = String::new();

    println!("Name:");

    io::stdin()
        .read_line(&mut name)
        .expect("Failed");

    println!("Age:");

    io::stdin()
        .read_line(&mut age)
        .expect("Failed");

    let age: i32 = age.trim().parse().unwrap();

    println!("{} is {}", name.trim(), age);
}
```

output

Theres a few different ways you can print to screen

```rust
println!("{}", name);
println!("{} is {}", name, age);
println!("{name} is {age}");
```