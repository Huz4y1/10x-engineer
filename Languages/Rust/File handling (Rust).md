Creating a file

```rust
use std::fs::File;

fn main() {
    let file = File::create("notes.txt")
        .expect("Failed");
}
```

Creating directories

```rust
use std::fs;

fn main() {
    fs::create_dir("data")
        .expect("Failed");
}
```

Checking if files exist

```rust
use std::path::Path;

fn main() {
    if Path::new("data.txt").exists() {
        println!("Exists");
    } else {
        println!("Not found");
    }
}
```

Reading all content of a file

```rust
use std::fs;

fn main() {
    let content = fs::read_to_string("data.txt")
        .expect("Failed to read file");

    println!("{}", content);
}
```

Writing to a file

```rust
use std::fs;

fn main() {
    fs::write("data.txt", "Hello")
        .expect("Failed to write");
}
```

Appending to a file

```rust
use std::fs::OpenOptions;
use std::io::Write;

fn main() {
    let mut file = OpenOptions::new()
        .append(true)
        .create(true)
        .open("data.txt")
        .expect("Failed");

    writeln!(file, "New line")
        .expect("Failed");
}
```

Reading line by line

```rust
use std::fs::File;
use std::io::{BufRead, BufReader};

fn main() {
    let file = File::open("data.txt")
        .expect("Failed");

    let reader = BufReader::new(file);

    for line in reader.lines() {
        println!("{}", line.unwrap());
    }
}
```