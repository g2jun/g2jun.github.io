# Ownership

## Ownership is about memory management, especially garbage collection

Stack memory is managed automatically, but only if a value has a fixed size and its size is known at compile time can it be stored on the stack. Values whose sizes are unknown until runtime or changeable in runtime have to be stored in the heap. Allocating memory in the heap can be handled by Rust when constructing a value of a data type, but there needs to be a way for Rust to decide when to deallocate the memory.

The main purpose of ownership is to enable automatic management of heap memory without using a runtime garbage collector. Therefore, every value stored in the heap must have an owner. When the owner is not accessible anymore, the value it owns in the heap must not be accessible either, and the memory used to store that value should be deallocated. However, if one value has two owners, both of them will try to deallocate its memory (unless they are doing reference counting at runtime), which will cause a memory safety error called "double free". Therefore, there should be only one owner responsible for deallocation at a time.

A variable holds a value directly. Whether the value is a piece of data we can use directly or it is an address pointing to a memory location in the heap (or stack), it has a fixed size, so it can be stored on the stack. Memory on the stack is managed in frames, and variables in a single scope are stored in a single frame. When a variable goes out of scope, in other words, the variable's scope ends, the stack frame for this scope is deallocated automatically, so the variable is not accessible anymore. If the variable owns a value in the heap, the memory occupied by that value needs to be deallocated. If that value owns another value, then the deallocation needs to be done recursively. Rust handles this memory cleanup process by running the destructor of the variable automatically when it goes out of scope. Thereafter, that variable is said to be *dropped*.
- [The Rust Programming Language, Brown Version](https://rust-book.cs.brown.edu/ch04-01-what-is-ownership.html#variables-live-in-the-stack)
- [Destructors](https://doc.rust-lang.org/reference/destructors.html)
- [Trait Drop](https://doc.rust-lang.org/core/ops/trait.Drop.html)

## Creation and transfer of ownership

When a value is constructed and bound to a variable, the variable becomes the owner of this value. If a value itself is an *owning pointer*, such as `Box<T>`, `Vec<T>`, or `String`, that points to another value in the heap, then it owns that value. When a value is assigned to a new variable, or is bound to a variable in a pattern, or is passed to a parameter in a function call, the ownership is transferred from the old owner to the new variable or the function parameter, the value is said to be *moved*. Thereafter, the value cannot be accessed through the old owner.

There are two exceptions though.

### Exception one: `Copy` types are not moved

1. If the type of a value implements `Copy`, then the value will not be moved, but *copied* bit by bit, also known as a shallow copy.
   - [Moved and copied types](https://doc.rust-lang.org/reference/expressions.html#moved-and-copied-types)
   - [Special types and traits](https://doc.rust-lang.org/reference/special-types-and-traits.html#copy)
   - [Trait Copy](https://doc.rust-lang.org/core/marker/trait.Copy.html)

   The Rust compiler implements `Copy` for some types:

   * Scalar types: integers, floats, `bool` and `char`.
   * Tuples, if all elements are `Copy` types
   * Arrays of `Copy` types
     - [Primitive Type array](https://doc.rust-lang.org/core/primitive.array.html)
   * Shared references (`&T`)
     - [Shared references (&)](https://doc.rust-lang.org/reference/types/pointer.html#r-type.pointer.reference.shared.copy)
     * Mutable references are *not* `Copy` because a value can only have one of them at a time.
       -[Mutable References (&mut)](https://doc.rust-lang.org/reference/types/pointer.html#r-type.pointer.reference.mut.copy)

   As a rule of thumb, values that can be entirely stored on the stack can be copied shallowly. When their owners go out of scope, their memory will be deallocated altogether with the stack frame. No heap allocation or deallocation is involved.
   - [Stack-Only Data: Copy](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html#stack-only-data-copy)

   `Copy` and `Drop` are mutually exclusive. Copying a `Drop` type may lead to a double free, it is hard for the compiler to correctly decide when to run the destructors. So it is not allowed to implement `Copy` on a type if the type or any of its parts has implemented `Drop`.
   - [Trait Drop](https://doc.rust-lang.org/core/ops/trait.Drop.html#copy-and-drop-are-exclusive)
   - [Trait Copy](https://doc.rust-lang.org/core/marker/trait.Copy.html#when-cant-my-type-be-copy)

### Exception two: A value cannot be moved out from behind a reference

2. References are *non-owning* pointers. They do not own the values they point to, instead they are said to *borrow* them. By dereferencing a reference, we can access the value it borrows, but we cannot move it.
   - [References (& and &mut)](https://doc.rust-lang.org/reference/types/pointer.html#references--and-mut)
   - [Borrow operators](https://doc.rust-lang.org/reference/expressions/operator-expr.html#borrow-operators)
   - [The dereference operator](https://doc.rust-lang.org/reference/expressions/operator-expr.html#the-dereference-operator)

### Partial move of compound types

Some types are compound types, meaning they can group multiple values into one type. These values can be accessed individually, sometimes they can be moved separately, it is called a partial move. If a compound value is partially moved, it contains deinitialized parts. Until those parts are reinitialized, the compound value can no longer be used as a whole.

* Tuple fields can be moved out of the tuple separately.
  - [Mutating Different Tuple Fields](https://rust-book.cs.brown.edu/ch04-03-fixing-ownership-errors.html#fixing-a-safe-program-mutating-different-tuple-fields)

* Array elements *cannot* be moved out of the array. Array elements are accessed by indexing the array, but the index is not always a fixed value known at compile time, it can be calculated at runtime, so the Rust compiler cannot always keep track of the states of individual elements, whether they are deinitialized or not.
  - [Mutating Different Array Elements](https://rust-book.cs.brown.edu/ch04-03-fixing-ownership-errors.html#fixing-a-safe-program-mutating-different-array-elements)

* `struct` fields can be moved out of the `struct` separately if the `struct` itself does not implement `Drop` and is stored in a local variable (so that the fields will not be dropped by any custom destructors).
  - [Field access expressions](https://doc.rust-lang.org/reference/expressions/field-expr.html#borrowing)

* `Vec<T>` elements *cannot* be moved out of the vector, just like arrays. From the implementation's perspective, an index expression `a[b]` is equivalent to `*std::ops::Index::index(&a, b)`, or `*std::ops::IndexMut::index_mut(&mut a, b)` in a mutable place expression context. That means indexing a vector borrows the vector as a whole and returns a reference to an element, the result then gets dereferenced. As mentioned above, dereferencing a reference cannot move the value behind the reference.
  - [index expression](https://doc.rust-lang.org/reference/expressions/array-expr.html#r-expr.array.index.trait)

## Summary of rules on ownership

The Brown book clearly states that "ownership is primarily a discipline of heap management." Therefore, its rules specify that each *heap* value must be owned by exactly one variable and be deallocated once this variable goes out of scope. The official book seems to generalize the ownership rules to cover all values not limited to heap. Since stack-only values do not need to manage heap memory and they are copied rather than moved, meaning ownerships are never transferred but only created on copy, the same rules can apply to them without change. So the ownership rules listed in the official book are:

> * Each value in Rust has an owner.
> * There can only be one owner at a time.
    - With the exception of [Rc<T>, the Reference Counted Smart Pointer](https://doc.rust-lang.org/book/ch15-04-rc.html).
> * When the owner goes out of scope, the value will be dropped.
- [The Rust Programming Language](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html#ownership-rules)

The Brown book adds another one:
> * Heap data can only be accessed through its current owner, not a previous owner.
- [The Rust Programming Language, Brown Version](https://rust-book.cs.brown.edu/ch04-01-what-is-ownership.html#summary)
This is for memory safety. Because the current owner can be dropped, the memory of the owned value can be deallocated, leaving the previous owner to point to deallocated memory. Accessing deallocated memory is a memory safety error called "use after free".

## Glossary

* Variable: A variable is a component of a stack frame, and it holds a value directly.
  - [The Rust Reference](https://doc.rust-lang.org/reference/variables.html)

* Scope: the region of source code where a variable may be accessed by its name.
  - [The Rust Reference](https://doc.rust-lang.org/reference/glossary.html#scope)

* The scope of a variable is the block it is defined in. A block is a syntax group enclosed by `{` and `}`.
  - [The Rust Programming Language, First Edition](https://doc.rust-lang.org/1.30.0/book/first-edition/variable-bindings.html#scope-and-shadowing)
  - [Pattern binding scopes](https://doc.rust-lang.org/reference/names/scopes.html#pattern-binding-scopes)
  - [Block expressions](https://doc.rust-lang.org/reference/expressions/block-expr.html)

# Aliasing XOR Mutability

XOR stands for eXclusive OR, meaning A or B, but not both, so aliasing and mutability cannot exist at the same time. The purpose of this principle is to improve memory safety and avoid data races.

When two variables refer to the same memory location, these two variables are aliasing, then neither one of them should be mutated, because mutation may cause deallocation of the current memory (think of resizing a vector), leaving the other variable to point to deallocated memory, which will lead to a memory safety error called "use after free".

When two variables read a value from a memory location at the same time, then they do some computation based on the same value respectively, then they write their results back into the memory, one of the results will get overwritten by the other. This can cause a data race in concurrent programming. Even only one variable needs to write its result back into the memory, the other one is not aware of the value change at the memory location during its lifetime, and when it reads from the memory again (think of an iterator), it may get a changed value, which probably is not the behavior we want. If these happen in a multi-threaded setting, the results are nondeterministic.

To prevent these issues, at any point in time for a memory location, only one variable can have the write permission, and *as long as it holds the write permission*, it is also the only one who has the read permission, nobody else can even read that memory. When more than one variable has read permission on a memory location, during their lifetimes, nobody can write to that location. A region of memory cannot be deallocated until nobody is referring to it anymore.

Further reading: [Rust: A unique perspective](https://limpet.net/mbrubeck/2019/02/07/rust-a-unique-perspective.html)

## Rules of references

Making references is called borrowing in Rust, it is the main form of aliasing. The book lists the rules of references as follows.
- [Borrow operators](https://doc.rust-lang.org/reference/expressions/operator-expr.html#borrow-operators)

> * At any given time, you can have either one mutable reference or any number of immutable references.
> * References must always be valid.
- [The Rules of References](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html#the-rules-of-references)

To elaborate on the rules above:

* For a variable to be borrowed as mutable, it must be mutable in the first place. The variable then can be mutated through dereferencing the mutable reference.
  - [Mutability](https://doc.rust-lang.org/reference/expressions.html#mutability)
* When a variable is borrowed as mutable, it cannot be accessed by its variable name or borrowed as immutable during the lifetime of the mutable borrow.
* When a variable is borrowed as immutable, it cannot be mutated (with the exception of *interior mutability*) or moved, or borrowed as mutable during the lifetime of the immutable borrow.
  - [Shared references (&)](https://doc.rust-lang.org/reference/types/pointer.html#shared-references-)
  - [Module cell](https://doc.rust-lang.org/stable/std/cell/index.html)
* A value cannot be dropped during the lifetime of *any* of its borrows.

### Some special characteristics of borrowing a reference

* When a reference itself is borrowed, it sometimes *forfeits* some of its permissions during the borrow of itself.

  * When a mutable reference is borrowed as mutable again, this new mutable reference takes both write and read permissions from the original reference and becomes the only one who has access to the memory location the original reference refers to. The original reference becomes unusable during the lifetime of this new mutable borrow.
  * When a mutable reference is borrowed as immutable, it shares the read permission with the new immutable reference and forfeits its write permission during the lifetime of this new immutable borrow.
  * When an immutable reference is borrowed as immutable again, it shares its read permission with the new borrow. If it is borrowed as mutable, the mutable borrow takes the read permission and renders the original reference unusable during the lifetime of this new borrow.

* If a reference `r` borrows a variable `v` (`r = &v` or `r = &mut v`), and then a variable `b` borrows the *dereferencing* of `r` (`b = & *r` or `b = &mut *r`), we can consider the reference `r` is live during the lifetime of `b`, which means the variable `v` is borrowed during the lifetime of `b`. Whether it is borrowed as mutable or immutable is still determined by `r`, who borrowed it originally.

* If a reference `r` is dereferenced multiple times and then get borrowed by `b`, we need to do the analysis recursively.
  1. Remove one layer of dereference from the expression being borrowed.
     For example, if `b = & **r` or `b = &mut **r`, the expression being borrowed is `**r`. Removing one layer of dereference results in `*r`.
  2. Consider the reference in the resulting expression (e.g., `*r` or `r`) from step one to be live during the lifetime of `b`, which means the variable borrowed by it is still borrowed during the lifetime of `b`. If the result is no longer a dereference expression, the analysis is done.
  3. Inspect the type of the resulting reference from step one. If it is an *immutable* reference, the analysis is done. Otherwise, substitute the result for the expression being borrowed and go back to step one.
  - [Reborrow constrains](https://rust-lang.github.io/rfcs/2094-nll.html#reborrow-constraints)

## Glossary

* Aliasing: variables and pointers referring to overlapping regions of memory.
  - [Rust Avoids Simultaneous Aliasing and Mutation](https://rust-book.cs.brown.edu/ch04-02-references-and-borrowing.html#rust-avoids-simultaneous-aliasing-and-mutation)
  - [The Rustonomicon](https://doc.rust-lang.org/nomicon/aliasing.html)

* Reference: A reference is like a pointer in that it's an address we can follow to access the data stored at that address; that data is owned by some other variable. Unlike a pointer, a reference is guaranteed to point to a valid value of a particular type for the life of that reference.
  - [References and Borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)

* Lifetime:
  * For a reference, its lifetime starts from where it is introduced and ends after where it is *last used*.
  * For a value, its lifetime starts from where it is constructed and ends when it is dropped. It is also called the scope of the value.
  - [The Rust Programming Language, Brown Version](https://rust-book.cs.brown.edu/ch04-02-references-and-borrowing.html#permissions-are-returned-at-the-end-of-a-references-lifetime)
  - [The Rust RFC Book](https://rust-lang.github.io/rfcs/2094-nll.html#what-is-a-lifetime)
