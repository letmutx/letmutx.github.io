---
layout: post
title:  "LinkedList in Rust"
date:   2018-01-03 11:57:48 +0530
categories: rust data-structure
---

I have never actually implemented any data structure in Rust from scratch. So, I thought why not spend a couple of hours doing that over the last weekend.

If you don't know already, Rust is a programming language designed to be safe and fast. The compiler requires all programs to be written following certain _ownership_ rules, as they call it and ensures that programs don't access invalid memory. There's more to it, but that would suffice for now.

I will assume you are familiar with generics, types, regular programming constructs like `if`, `for` etc.. I will also assume that you have some familiarity with how the borrow checker works.

In this post, let's implement a generic list which supports `append` and `remove` operations and later on, ways to extend the list to support more operations.

Like many languages, Rust supports `struct`s. Let's start by defining our `List` and `Node` structures. Our `List` would contain a length field, and a pointer to the first node. `Node` will contain the actual data along with a pointer to the next `Node`.

```rust
struct List<T> {
    head: Option<Box<Node<T>>>,
    len: usize
}

struct Node<T> {
    data: T,
    next: Option<Box<Node<T>>>
}
```

You probably guessed it, but `<T>` denote the use of generic types as in many other languages.

`Option` and `Box` must have piqued your interest.

In Rust, `Box` is a heap-backed pointer type(like a pointer returned by `malloc`). `Box<T>` is a pointer to the memory location which contains a value of type `T`. Also, `Box` is the **owner** of whatever it points to which means that you can change the contents of the memory location pointed to by the `Box` only through it.

There is no such thing as an empty `Box`. This means that `Box` always points to a valid memory location. To represent a `NULL` pointer, we make use of the `Option` type and wrap our `Box` inside that. `Option` is an enum, which either contains some value denoted by `Some(val)` or `None` which denotes nothing is inside. In this case, we are telling that the head is either a pointer to a `Node` or nothing.

Let's go ahead and create an associated function(similar to a _static method_ in other languages) which will return an empty list. In Rust, we define associated functions and methods for a `struct` inside an `impl` block. We will see why when we implement the `remove` method.

```rust
struct Node<T> {
    data: T,
    next: Option<Box<Node<T>>>
}

struct List<T> {
    head: Option<Box<Node<T>>>,
    len: usize
}

impl<T> List<T> {
    // Create an empty List
    fn new() -> Self {
        List {
            head: None,
            len: 0usize
        }
    }
}

fn main() {
    let _ = List::new();
}
```

```shell
$ rustc list.rs
```

Before going on to implement other functionality, let's talk a bit about the ownership rules I mentioned above because they will come into play around this point.

In Rust, there's always an owner to a value. If you do `let list = List::new();`, list is the owner of the value returned by `List::new`. If you assign, list to another variable(`list2`) using a `let` expression like below, you can no longer access the `list` variable. You effectively transferred the  **ownership** of the value from `list` to `list2`.

```rust
let list = List::new();

let list2 = list;
```
But having only owned values doesn't cut it, so we also have 

* mutable(`&mut T`) references: can modify the contents and 
* immutable(`&T`) references: can't modify the contents.

Ownership rules ensure that at any time there is either a single mutable reference or multiple immutable references but not both. I won't get into the details why(read the [book][book]). All programs should follow the rules or else they won't compile.

## Appending to the list

### Attempt #1

```rust
impl<T> List<T> {
    fn new() -> Self {
        List {
            head: None,
            len: 0usize,
        }
    }

    fn append(&mut self, data: T) {
        let new_node = Some(Box::new(Node {
            data: data,
            next: None,
        }));
        if self.head.is_none() {
            self.head = new_node;
            self.len += 1;
            return;
        }
        let mut head = self.head.as_mut().unwrap();
        loop {
            let next = head.next.as_mut(); // <- right here
            if let Some(next) = next {
                head = next;
            } else {
                break;
            }
        }
        head.next = new_node;
        self.len += 1;
    }
}
```

Of course, this didn't compile. The reason is that the mutable reference inside the loop should be valid till the end of the method to enable us to do  `head.next = new_node;`. But when we hit the second iteration of the loop, we get another mutable reference which violates the single mutable reference rule. So, `rustc` complains:

```shell
$ rustc list.rs
error[E0499]: cannot borrow `head.next` as mutable more than once at a time
  --> test.rs:33:24
   |
33 |             let next = head.next.as_mut();
   |                        ^^^^^^^^^ mutable borrow starts here in previous iteration of loop
...
43 |     }
   |     - mutable borrow ends here
```

### Attempt #2

After thinking for a while, I came up with an idea along these lines: _move_ the `head` out of the `List`, taking _ownership_ of the first node, iterate till the end checking the next pointer for `None` at every node and inserting the new node there.

As we move the value out of the list, there would only be a single owner and we can mutate it safely. All good, Rust is happy and so am I! 

But there's a small gotcha here. Until you add the new element at the end, you can put the node back into it's position as that would get us back to the same problem we started with. Recursion to our rescue!

```rust
impl<T> List<T> {
    fn new() -> Self {
        List {
            head: None,
            len: 0usize,
        }
    }

    fn append(&mut self, data: T) {
          // allocate new node on the heap
        let new_node = Some(Box::new(Node {
            data: data,
            next: None,
        }));
        // move first node out of the list
        if let Some(mut head) = self.head.take() {
            add_node(&mut *head, new_node);

            // put back the first node
            self.head = Some(head);

        } else { // list is empty
            self.head = new_node;
        }
        // increment the length
        self.len += 1;
    }
}

fn add_node<T>(node: &mut Node<T>, new_node: Option<Box<Node<T>>>) {
    if node.next.is_none() {
        // append to the list
        node.next = new_node;
    } else {
        // move the value out of the next pointer
        let mut next = node.next.take().unwrap();

        // send in a mutable reference
        add_node(&mut next, new_node);
        
        // put back the node
        node.next = Some(next);
    }
}
```

That's it! You can append a value to the list now.

```rust
fn main() {
    // mut here says that you are going to modify the value of list later on
    let mut list = List::new();
    
    list.append(7);
    list.append(8);
    
    let first = list.head.unwrap().data;
    let second = list.head.unwrap().next.unwrap().data;
    
    println!("{} {}", first, second);
}
```

## Removing from the list

Removing from the list is pretty straightforward, given that we have already seen how to implement `append`.

```rust
struct Node<T> {
	...
}

struct List<T> {
	...
}

impl<T> List<T> {
	fn new() -> Self {
		...
	}

	fn append(&mut self) {
		...
	}
}

impl<T: PartialOrd> List<T> {
    fn remove(&mut self, data: T) {
        if let Some(mut head) = self.head.take() {
            if head.data == data {
                self.head = head.next.take();
                self.len -= 1;
            } else {
                if remove_node(&mut *head, data) {
                    self.len -= 1;
                }
                self.head = Some(head);
            }
        }
    }
}

fn remove_node<T: PartialOrd>(node: &mut Node<T>, data: T) -> bool {
    if node.next.is_none() {
        false
    } else {
        let mut next = node.next.take().unwrap();
        if next.data == data {
            node.next = next.next.take();
            true
        } else {
            if remove_node(&mut *next, data) {
                node.next = Some(next);
                true
            } else {
                node.next = Some(next);
                false
            }
        }
    }
}
```

Perhaps, the most interesting thing here is the new syntax: `impl<T: PartialOrd>` which says - implement the `remove` method for this list if the underlying type `T` can be compared using `==`. [PartialOrd][partialord] is a **trait** (similar to an _interface_) and `T: PartialOrd` is a _trait bound_ on `T`.

With multiple `impl` blocks having different trait bounds, we can implement additional functionality for our list depending on what the underlying type supports.

At this point, you are probably bored reading stuff and want to do something on your own. Try changing our append only list to be able to insert/remove element at an index.

You can find the full code sample [here][gist]. If you find any mistakes or have suggestions, I am all ears.


[book]: https://doc.rust-lang.org/stable/book/second-edition/ch04-00-understanding-ownership.html

[partialord]: https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html

[gist]: https://gist.github.com/letmutx/64dbcf77518fe4991f4eea85960c9dee
