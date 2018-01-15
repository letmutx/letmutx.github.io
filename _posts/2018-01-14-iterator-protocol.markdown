---
layout: post
title:  "The Iterator Protocol"
date:   2018-01-14 11:57:48 +0530
categories: rust iterator
---

Last time we saw how to implement a linked-list. But what good is a list if you can't iterate over it? In this post, we will implement the [Iterator][iterator] trait for our list which will enable us to use our list with a `for` loop[^loop] among the other cool stuff we get.

First, let us take a look at the trait itself:

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    
    // there are more things, we will look at them later
}
```

The `Item` field in the trait definition is an **associated type**. In this case, we are supposed to give it the type of the values the iterator is going to produce when we iterate over it.

Then, there's the `next` method which is going to produce the values when called by the `for` loop. It will either produce `Some(value)` or `None` denoting the end of the sequence.

Let us go ahead and implement the `Iterator` trait for our list:

```rust
impl<T> Iterator for List<T> {
    type Item = T;
    
    fn next(&mut self) -> Option<Self::Item> {
        let node = self.head.take();
        match node {
            Some(mut node) => {
                self.head = node.next.take();
                Some(node.data)
            }
            None => None,
        }
    }
}
```

That's it. Now you can use a `for` loop with it. Cool, eh?

```rust
fn main() {
    let mut list = List::new();

    for i in 0..12 {
        list.append(i);
    }

    for i in list {
        println!("{}", i);
    }

}
```

```
$ rustc list.rs
$ ./list
0
1
2
...
11
```

But wait! Remember the single owner rule in Rust? That's at play here. So, when we used the iterator, we were removing those elements from the list too. You can check for yourself by modifying the `main` function and check that the second iteration doesn't print anything:

```rust
fn main() {
	let mut list = List::new();
	
	for i in 0..12 {
		list.append(i);
	}
	println!("First iteration");
	for i in list {
		println!("{}", i);
	}
	println!("Second iteration");
	for i in list {
		println!("{}", i);
	}
}
```

But that's shady behaviour[^behaviour] and more often than not, we don't want to do that.

(BTW, there's also a hidden bug[^bug] here.)

We can't also keep the values in the list and use them from outside because of the single owner rule. So, instead of iterating over the values we will change our implementation to iterate over the references(`&T`) of the values in the list. That way, the compiler won't complain and we can use the values by deferencing the references.

We can't do it with our current `List` definition because we don't know which one to return next. We can add an offset field to the `List` structure and use it, but that's unnecessary overhead if we don't want to use the iterator and also add more unnecessary logic everywhere. 

A common pattern that the standard library uses is to define a wrapper structure which will maintain necessary state for iteration. Some method (usually called `iter`) on the list will construct this wrapper which can then be used to iterate. Thanks to type inference, we don't need to remember the wrapper types :)

```rust
struct IterList<'a, T: 'a> {
    head: Option<&'a Node<T>>,
}

impl<'a, T> Iterator for IterList<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        match self.head.take() {
            Some(node) => {
                let next = node.next
                     // Converts Option<Box<Node<T>> to Option<&Box<Node<T>>
                     .as_ref() 
                     // map the &Box<Node<T>> to to &Node<T>
		     .map(|node: &'a Box<Node<T>>| node.as_ref());
                self.head = next;
                Some(&node.data)
            }
            None => None,
        }
    }
}
```

Now the method:

```rust
impl<T> List<T> {
	fn new() -> Self {
        List {
            head: None,
            len: 0usize,
        }
    }
    
    ...
    
    fn iter<'a>(&'a self) -> IterList<T> {
        IterList { 
        	head: self.head
        	          .as_ref()
        	          .map(|node: &'a Box<Node<T>>| node.as_ref())
       }
    }
}
```

We now have an iterator over `&T`. You must be wondering what all those `'a`s are. They are called **lifetime parameters**, help prevent dangling pointers. `a` is the name of the lifetime, can be any identifier. Usually, the compiler fills in lifetimes most of the times(just like type inference) using [lifetime elision][elision] rules but we need to specify them otherwise.

The `T: 'a` on the struct definition is a bound on the lifetime of `T`(similar to a trait bound). It says that `T` lives at least as long as the structure it is inside - and so we get valid references to `T` as we iterate over the list.

## Safety

Most languages don't allow modifications to the underlying collection when iterating over it. It is either undefined behaviour or an exception at runtime. Rust can catch such errors at compile times and reject programs using lifetimes.

So as long as `IterList` is in scope, you can't modify the original list as `IterList` borrows the `Node` inside the list and so the list itself. As long as the borrow exists, we can't modify the original list because only mutable or immutable references can exist at the same time.

## Lazy

Iterators are lazy. They don't do anything until you use them. This allows us to create huge sequences like

```rust
let sum = (1..100000000).take(5).sum();
```
which returns the sum of the first five elements without actually creating all the elements.
## The good stuff

Remember the other methods we in the [Iterator][iterator] trait that we didn't implement? We don't have to! We get all that for free! They are all combinators and you can compose them in several ways to get a lot of functionality. Go check them out! You can find the full code [here][gist]

Thanks for reading this far. If you have any suggestions, find any mistakes please let me know!

[^bug]: We are removing things from the list, but we forgot to update the length field. It is a logical error and Rust can't do anything about it. 

[^behaviour]: Not always. But for the purpose of our discussion, it is.

[^loop]: Actually `for` needs [IntoIter][intoiter]. But `IntoIter` is [implemented][all] for all types that implement `Iterator`.
[intoiter]: https://doc.rust-lang.org/std/iter/trait.IntoIterator.html
[all]: https://doc.rust-lang.org/std/iter/index.html#for-loops-and-intoiterator
[iterator]: https://doc.rust-lang.org/stable/std/iter/trait.Iterator.html
[copy]: https://doc.rust-lang.org/std/marker/trait.Copy.html
[elision]: https://doc.rust-lang.org/book/second-edition/ch10-03-lifetime-syntax.html#lifetime-elision
[gist]: https://gist.github.com/letmutx/64dbcf77518fe4991f4eea85960c9dee