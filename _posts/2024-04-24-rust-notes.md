---
layout: post
title: "Notes on Learning Rust"
tags: rust
---

This is a collection of notes that I took while I was learning Rust from the rust book website.

# Shadowing

you can define multiple variables with the same name without specifying `mut` if the variables are different types. 
The more recent definition will be used. Useful for convenience things like

```
let spaces = "     ";
let spaces = spaces.len();
```

# memory scope for heap data

```
{
  let s = String::from("hello"); // allocate s into memory
  /// do whatever
 } // s is dropped from memory due to drop() being called on it once scope is left
 ```
 
 # stack versus heap memory and ownership
 
 two '5' values put onto the stack
 ```
 let x = 5;
 let y = x;
 ```
 
 1. text data allocated in heap
 2. s1 String created with pointer to heap data
 3. s2 String created with pointer to same heap data
 4. 'ownership' of heap data is thusly transferred from s1 to s2, so s1 is useless now. This is also called a "move" 
 
 this prevents drop() being called twice on the same heap memory location after the scope is over (double free error)
 ```
 let s1 = String::from("hello");
 let s2 = s1;
 println!("{s1}"); // error here
 ```
 
 # producing copies: heap objects
 
 1. text data created in heap
 2. s1 String points to heap data
 3. heap data is cloned
 4. s2 String points to cloned heap data
 ```
 let s1 = String::from("hello");
 let s2 = s1.clone();
 ```
 
 # producing copies: stack objects
 
 1. 5 created in stack, labeled x
 2. another 5 created in stack (same value as x), labeled y

All data types with the `Copy` trait are full copied like this during reassignment. You are not able to `Drop` and `Copy` on the same data type.

```
let x = 5;
let y = x;
```

Tuples implement copy trait if all subelements implement copy trait.

# passing variables into functions

passing heap objects (Drop trait) directly into a function causes the value to be 'moved' into the function, the same way that redefining variable works. After the function call, the String's pointer to the value is no longer valid.

```
let s = String::from("hello");
some_function(s);

// s is no longer a valid pointer to the String value
```

Similarly, passing Copy trait objects into a function just creates a copy.

```
let x = 5;
some_other_function(x); // a copy of x is created in the scope of the other function
// can still use x to do stuff
```

The opposite is also true using return values out of functions.

```
fn some_function(s: String) -> String { s }
let s1 = String::from("hello");
let s2 = some_function(s1) // s1 is moved into (function scope) s, and s is moved into s2
```

# References

```
fn main() {
    let s1 = String::from("hello"); // s1 String points to heap memory text data
    let len = calculate_length(&s1); // &s1 points to s1 String, which itself points to heap memory text data

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    // s: String would have a pointer to heap data
    // s: &String would have a pointer to a String, which has a pointer to heap data
    println!("{}",s.len()); // len() is a property of String itself and so dot notation auto dereferences?
    println!("{}", *s); // need to dereference to actually get the value
    s.len()
} // s goes out of scope
```

# mutable references

```
fn main() {
    let mut s = String::from("hello"); // create String with pointer to heap data, marked as mutable
    // let r1 = &mut s; // can't do this and also call change() below
    change(&mut s); // creates a mutable pointer to mutable String
}

fn change(some_string: &mut String) {
  // mutable pointer to String is modified??
    some_string.push_str(", world");
}
```

can't have multiple mutable references to same object in the same scope. it's fine if the first reference is block scoped out somehow though.

Can have multiple immutable borrows, but not immutable and immutable borrows in the same scope and used at the same time. However, you CAN have an mutable borrow as long as all your immutable borrows happened already and there are no more immutable borrow calls after your mutable borrow calls.

# the dot operator

the dot operator will reference/dereference "as many times as it takes to match the function signature"

```
// works
(&mut &mut &mut ref_to_string).push_str("without dereferencing |");
```

# dangling references

if you try to manually assign a pointer to something that is later dropped out of scope, rust will give a dangling pointer error. 

# Slices

string slices `&str` are a pointer to a part of a longer String. Constant string slices `let x = "asdf"` are pointers to a string hardcoded in the binary file. 

## deref coercion of Strings to string slices

if you have a function defined as 

```
fn func(s: &str) -> &str {}
```

and you pass in a String instead of a string slice `&str`, the String will be auto-sliced into a string slice of the entire String value. 

```
let my_string = String::from("hello world");
func(&my_string); // ok
func(&my_string[..2]); // also ok
```

# Slices in general

```
let a = [1, 2, 3, 4, 5];
let slice = &a[1..3];
assert_eq!(slice, &[2, 3]);
```

can also do general purpose array slices, which have type signature `&[T]` like `&[i32]`

# Structs

when defining members of a struct def, you can omit the definition if a variable name matches the struct property name:

```
fn build_user(email: String, username: String) -> User {
    User {
        active: true,
        username,
        email,
        sign_in_count: 1,
    }
}
```

can also copy values from another struct using `..` notation

```
fn main() {
    // --snip--

    let user2 = User {
        email: String::from("another@example.com"),
        ..user1 // email won't get overwritten with the user1 value. this must come last.
    };
}
```

Note that doing this will make `user1` unusable because a datatype implementing `Drop` is moved, i.e. the `username` field. If only `Copy` data types were moved, then `user1` would still be usable.

can also make tuple structs that only define the types and not the names:

```
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

fn main() {
    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);
}
```

not sure what the purpose of unit-like structs are.

```
struct AlwaysEqual;

fn main() {
    let subject = AlwaysEqual;
}
```

# struct methods

```
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    println!(
        "The area of the rectangle is {} square pixels.",
        rect1.area()
    );
}
```

# println! debugging struct output

1. add the attribute `#[derive(Debug)]` to the beginning of the program
2. use `println!("{:?}", my_struct)` or `println!("{:#?}", my_struct)` for pretty print

can also use the `dbg!` macro and just plop in values such as `dbg!(30 * x)`

# enums

enums represent data classes that can either be one thing or one of others. the example in the book is ip address types:

```
enum IpAddrKind {
    V4,
    V6,
}

```

Instead of defining a struct to actually house ip addresses, you can attach data to the enum definitions for each possible type:

```
    enum IpAddr {
        V4(String),
        V6(String),
    }
    let home = IpAddr::V4(String::from("127.0.0.1"));
    let loopback = IpAddr::V6(String::from("::1"));
```

the enum "things" don't all have to be the same type either.

```
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

```

# the Option enum

```
    let some_number = Some(5);
    let some_char = Some('e');

    let absent_number: Option<i32> = None;

```

Lets coders identify when things exist or not by checking if the value is a Some(something) or just None

Can use the output of the Option<T> enum using a match

```
    fn plus_one(x: Option<i32>) -> Option<i32> {
        match x {
            None => None,
            Some(i) => Some(i + 1),
        }
    }

    let five = Some(5);
    let six = plus_one(five);
    let none = plus_one(None);

```

# matches

the output of the match is the result of the match item if it's a single line. if it's a block, the output is the last non-semicolon line. 

The last arm of a match, if it's a string, will be coerced into a catchall variable name. If you don't care about the value, you can put a `_` instead.

```
    let dice_roll = 9;
    match dice_roll {
        3 => add_fancy_hat(),
        7 => remove_fancy_hat(),
        other => move_player(other),
    }

    fn add_fancy_hat() {}
    fn remove_fancy_hat() {}
    fn move_player(num_spaces: u8) {}

```

in the ultimate "otherwise do nothing" case, you can do `_ => ()` 

# if let

if you only care for one possible outcome of an enum or something you can use if-let instead. Compare the following blocks of code:

```
    let config_max = Some(3u8);
    match config_max {
        Some(max) => println!("The maximum is configured to be {}", max),
        _ => (),
    }
```

```
    let config_max = Some(3u8);
    if let Some(max) = config_max {
        println!("The maximum is configured to be {}", max);
    }
```

if config_max can be coerced into Some(value), the if statement will proceed.

# misc package notes

```
use std::io;
use std::io::Write;

// can become
use std::io::{self, Write};
```

globs: 

```
use std::collections::*;
```

module names should match their file names for them to be correctly detected in a rust package/crate.