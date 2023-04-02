# Thread Borrow Checking

Threads allow us to write parallel code. In Rust, threads too are bound by the rules of borrow checking. This helps us ensure that our Rust code has no data races. However, there are some places where our code does comply to the borrow checking rules, but the rust compiler is not able to correctly prove it.

An example would be:
```Rust
use std::thread;

// We know that `vec` lives longer than the threads, So this code does
// work within the bounds of borrow checking

fn main() {
    let vec = vec![1, 2, 3, 4];
    let handle = thread::spawn(|| for i in &vec { println!("{i}"); });
    let handle2 = thread::spawn(|| for i in &vec { println!("{i}"); });
    
    handle.join().unwrap();
    handle2.join().unwrap();
}
```

However, we get the following error:
```
error[E0373]: closure may outlive the current function, but it borrows `vec`, which is owned by the current function
 --> src/main.rs:9:32
  |
9 |     let handle = thread::spawn(|| for i in &vec { println!("{i}"); });
  |                                ^^           --- `vec` is borrowed here
  |                                |
  |                                may outlive borrowed value `vec`
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:9:18
  |
9 |     let handle = thread::spawn(|| for i in &vec { println!("{i}"); });
  |                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
help: to force the closure to take ownership of `vec` (and any other referenced variables), use the `move` keyword
  |
9 |     let handle = thread::spawn(move || for i in &vec { println!("{i}"); });
  |                                ++++

...
```

Here we see that the lifetime of a thread is considered as `'static`. Hence, any borrows in this thread must also live as long as `'static`.

> **NOTE:** There are certain other important pieces related to threads such as `move`. However, for now we are only going to focus on borrowing values into threads.

In this paper, I try to improve the borrow checking related to threads and thus allow a wider range of code to be valid in Rust.

## Current solution to this problem

To solve this very problem, we have _scoped threads_ in Rust. These have special borrow checking functionality similar to what we are trying to achieve.

The previous example with scoped threads would look like:
```Rust
fn main() {

    let vec = vec![1, 2, 3, 4];
    
    thread::scope(|s| {
        s.spawn(|| for i in &vec { println!("{i}"); });
        s.spawn(|| for i in &vec { println!("{i}"); });
    });
    
}
```

And this does compile and run as expected.

However, I would argue that ... _\<Argue something here>_

## Proposed Solution

I suggest certain modifications to the way borrow checking works with threads to allow the example code to work without having to use `thread::scope`.

```Rust
use std::thread;

// We know that `vec` lives longer than the threads, So this code does
// work within the bounds of borrow checking

fn main() {
    let vec = vec![1, 2, 3, 4]; // <-- vec is created

    //           v------------ Thread 1 starts here
    let handle = thread::spawn(|| for i in &vec { println!("{i}"); });
    //           v------------ Thread 2 starts here
    let handle2 = thread::spawn(|| for i in &vec { println!("{i}"); });
    
    handle.join().unwrap();  // <-- Thread 1 ends here
    handle2.join().unwrap(); // <-- Thread 2 ends here
} // <-- vec dropped here
```

Here, the borrow checker should be able to prove that `vec` lives longer than both the threads and hence make this code compile without any errors.

#### The way I propose this works is:
- We follow the thread join handle throughout the flow of the program
- And assign the lifetime to the thread where the join handle is joined

So in the example above, the thread would have the lifetime of `main` rather than `'static`. Thus, proving that `vec` does live as long as the thread.
