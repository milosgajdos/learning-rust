This post contains a list of simple experiments I made playing with [rust](https://www.rust-lang.org/en-US/) to get a better grasp of the ownership rules `rust` compiler imposes on `references`. The experiments listed here are by no means exhaustive, but I wanted to have them available somewhere with my comments for future reference. Equally I figured they might help someone learning about this topic.

You should definitely read some [docs](https://doc.rust-lang.org/rust-by-example/scope/move.html) about `rust` borrowing and ownership first if you've never done any `rust` programming before.

# Immutable (shared) and mutable references

`rust` is one of the most interesting languages I've come across when it comes to systems programming safety and avoiding as many possibilities of unexpected runtime surprises and all kinds of undefined behaviour which is so prevalent in `C/C++` and probably some other compiled languages I haven't hacked on.

`rust` allows to create mutable or immutable references to variables, each option coming with some restrictions strictly checked by `rust` compiler during build time. Learning `rust` requires one to grok these rules in order to be able to write borrowing code with ease. These scarily looking rules can be somewhat distilled as follows:

* If there is at least one existing **immutable** referents to a variable (value) not even the variable (the owner of the value) can modify its value: the value is locked down for reading only!
* If there is a **mutable** reference to a variable holding some value, you can't use the owner of the value at all as long as the mutable reference exists.
* No reference can outlice its referent: this is one of the most amazing features of `rust` as it helps to avoid dangling pointers, so omnipresent in many compiled languages like `C, C++ and Go` for example.

For brevity, we will skip discussing mutable borrows to immutable values as that makes now sense: immutable values should never be able to be modified eg. this won't compile ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=d9d986c66f60fa504070b78f23a1531e)):

```rust
    let a = 30;
    // mutable reference
    let foo = &mut a;
    println!("{}", *foo);
```

# Immutable reference immutably borrowing

The following won't compile ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=987612fe787cd659416d86e94a1c324f)):
```rust
    let mut a = 30;
    let foo = &a;

    // can't modify `a` here because immutable borrow (`foo`) exists
    a = a + 1;
    println!("{}", a);
```

This breaks the first rule: since `foo` is an **immutable** reference to `a`, `a` is **locked down** for reading only

# Immutable reference mutably borrowing

The following won't compile ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=cf3bd05805ff64d4bb3eb451e1fc652c)):
```rust
    let mut a = 30;
    let foo = &mut a;

    // can't modify because mutable borrow exists
    a = a + 1;
    println!("{}", a);
```

This breaks the second rule listed earlier: `foo` is a **mutable** reference to `a`, but despite `a` being mutable it's still not allowed to change its value since mutable reference `foo` exists. Once the `foo` is dropped, `a` becomes modifiable (mutable) again [playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=f5aab35d78c9d671626a394078d19e55):

```rust
    let mut a = 30;

    {
        let foo = &mut a;
        // can't modify here because mutable borrow exists
        // a = a + 1;
    }

    // can modify because mutable borrow is now dropped
    a = a + 1;
    println!("{}", a);
```

# Mutable reference mutably borrowing

So far the above listed the cases where our references were immutable but were borrowing either mutably or immutably. We would get the same reults if our reference was defined mutable i.e. if `foo` was defined as:

```rust
    let mut foo = &mut a;
    // OR
    let mut foo = &a;
```

The difference here is that we can mutate i.e. assign a different reference to `foo` once it's defined as mutable:

```rust
    let mut a = 30;
    let mut foo = &mut a;
    let mut b = 100;
    // `foo` is mutable so we can assign &b to it
    foo = &b;
```

Now this is when it gets interesting. What caught me out when hacking on this was the following code ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=8dda2a094a24493c1c58ad2b275d78cd)):

```rust
    let mut a = 30;
    let mut foo = &mut a;
    let mut b = 10;
    // borrow &b mutably
    foo = &mut b;
    // change `b` by assigning 100 to `foo`; this won't compile
    *foo = 100;
    println!("foo: {}", *foo);
```

The above won't compile with the following error:

```console
error[E0597]: `b` does not live long enough
  --> src/main.rs:9:16
   |
9  |     foo = &mut b;
   |                ^ borrowed value does not live long enough
...
14 | }
   | - `b` dropped here while still borrowed
   |
   = note: values in a scope are dropped in the opposite order they are created
```

First I was like **DUH?!**, is `rust` really complaining about `b` being dropped **after the main()** ??? I mean that should not matter, since once `main()` is finished who cares about what is dopped where!?

It turns out, `rust` compiler probably prunes the syntax tree in order and thus `b` is pruned (dropped) before `foo`, which holds a reference to `b` and that violates the third rule listed at the beginning: no reference is allowed to outlive its referent -- since `b` is dropped before `foo` (because it was initialised after `foo`) whilst `foo` holds a reference to it.

The above code snippet can be easily fixed by initialising/defining `foo` after `b` as follows ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=f67804f23506c19e1582cbb589b84ba6)):
```rust
    let mut a = 30;
    // mutable value `foo` is immutable reference to value `a`
    // mutability means: you can assign other references to it
    let mut b = 10;
    // `foo` is initialised after `b`
    let mut foo = &mut a;
    foo = &mut b;
    *foo = 100;
    println!("foo: {}", *foo);
```

This builds fine, since `foo` now gets dropped before `b` and thus reference to `b` is dropped as well, so `b` can be safely dropped, too.

Now, I wanted to see if `b` value was really changed by assiging to `*foo` so I figured, I'd just print it allongside `*foo`:

```rust
    println!("foo: {}, b: {}", *foo, b);
```

But this fails with the following error:
```console
error[E0502]: cannot borrow `b` as immutable because it is also borrowed as mutable
  --> src/main.rs:10:38
   |
8  |     foo = &mut b;
   |                - mutable borrow occurs here
9  |     *foo = 100;
10 |     println!("foo: {}, b: {}", *foo, b);
   |                                      ^ immutable borrow occurs here
11 | }
   | - mutable borrow ends here
```

It turns out `println!` macro borrows immutably (noted!), so since `foo` already borrows immutably, the build fails as it violates the second rule listed at the beginning: the existence of immutable reference prevents you from using the original value i.e. immutable reference has exclusive read/write access to the referred value.

Rest assured, `*foo` assignment really does modify `b` as you can see by running the below code ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=a42a3ec3e9b5d731765f7881270cf289)):
```rust
    let mut a = 30;
    let mut b = 10;
    {
        let mut foo = &mut a;
        foo = &mut b;
        *foo = 100;
        println!("foo: {}", *foo);
    }
    println!("b: {}", b);
```

# Conclusion

There is way more to references, ownership and borrowing in `rust` than what's described here, but the code snippets along with links to `rust` playground offer at least some basic insights you can build upon by diving deeper into these topics once you've grokked the basics.
