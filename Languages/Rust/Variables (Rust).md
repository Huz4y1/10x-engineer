Variables in rust are immutable by default, which means they cannot change so they are constants

```rust
let age = 17;  // immutable

let mut age = 17; // mutable now you can change the data inside this variable 
```

Strings

```rust
let name = "Bob"; // return type is &str and is read only which means it can't be changed
```

owned strings (like normal strings in other languages)

```rust
let name = String::from("Bob"); //
```

Converting &str → string type

```rust
let name = "Bob";

let owned = name.to_string();

// or other way round 

let name = String::from("Bob");

let slice = &name;
```