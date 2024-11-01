+++
title = "Lifetimes considered useful"
date = 2024-10-30
+++
I recently ran into the problem of designing an API that required a library user to make choice(s) from a set of values.
To model this let's imagine we are creating a card game as a library, and we want the user of the library to provide a function for choosing a suit of cards:
```rust
#[derive(Clone, Copy)]
enum Suit {
    Clubs,
    Diamonds,
    Hearts,
    Spades,
}

#[derive(Clone, Copy)]
struct Suits<const N: usize>([Suit; N]);

impl<const N: usize> Suits<N> {
    // constructor
    pub fn with_suits(suits: [Suit; N]) -> Suits<N> {
        Suits(suits)
    }
    
    // where chooser is some external function that chooses from the provided suits
    pub fn choose_suit<C>(&self, chooser: C)
    where
        C: FnOnce([Suit; N]) -> Suit
    {
        // have user choose some suit
        let suit = chooser(self.0);

        // do stuff with suit
        // ...
    }
}
```
In the above case, we have a container that holds suits, and we
want the user to choose one suit from our inner suit array. As the
function is currently written however, the user could return any arbitrary
suit, even if it was not contained within our array.
```rust
// here's what the user does:
let suits = Suits::with_suits([Suit::Clubs, Suit::Diamonds]);
    
// this means choose_suit will get a spades, even though our array does not include spades
suits.choose_suit(|_| Suit::Spades);
```
How do we stop a user from selecting a Suit that is not in our array?

We could:
- Just tell them to code better and not misuse our API (easy for us, but goes against Rust's "if it compiles it works" mantra)
- Put in runtime checks to ensure the choice is contained within our choices (works, but it's annoying and incurs a performance penalty)

These options work but kind of suck. The question we should instead be asking is: "How do we prevent the user from even being in a position where they can select an incorrect choice?"

We need some way to prevent a user from creating a choice out of "thin air". One way we can do this is to wrap each value in a type with no public constructors:
```rust
pub struct Choice<T>(T);

impl<T> Choice<T> {
    // so the closure can preview the choice's value
    pub fn view(&self) -> &T {
        &self.0
    }
}
``` 
Then we can model our function like so:
```rust
impl<const N: usize> Suits<N> {
    // ...
    pub fn choose_suit<C>(&self, chooser: C)
    where
        C: FnOnce([Choice<Suit>; N]) -> Choice<Suit>
    {
        // have user choose some suit
        let Choice(suit) = chooser(self.0.map(Choice));

        // do stuff with suit
        // ...
    }
    // ...
}
```
And our user would use it like this:
```rust
let suits = Suits::with_suits([Suit::Clubs, Suit::Diamonds]);
    
// Now we cannot do this, because the type required to be returned is 
// a Choice<Suit>, which cannot be constructed by the library user
// suits.choose_suit(|_| Suit::Spades);

// Instead, we must choose from the two values provided
suits.choose_suit(|[one, two]|
    // view and compare to make choice...
    if one.view() == &Suit::Clubs {
        two
    } else {
        one
    }
);
```
This is almost complete, but one thing we have failed to account for is selecting different choices of the same type. Since our type doesn't have a public constructor, any values of it are guaranteed to only come from us, which we only give to the public as parameters passed to the closure. However, Rust allows the user to move owned values, and so a user could move one of the values out of the closure and use it in a later invocation.
```rust
let suits = Suits::with_suits([Suit::Clubs, Suit::Diamonds]);

// smuggle value into outer scope
let smuggler: Option<Choice<Suit>> = None;
suits.choose_suit(|[one, two]|
    smuggler = Some(one);
    two
);

// here we are creating a new Suits value with suits that do not include our smuggled value
let other_suits = Suits::with_suits([Suit::Diamonds, Suit::Hearts]);

// use smuggled value with different suits, now we have run into our original problem
other_suits.choose_suit(|_| smuggler.unwrap());
```
What user would even think to do this? Most likely no one would, and if they did, it would be their fault for using the library incorrectly. But we want assurances damn it! 

How do we prevent a user from smuggling out a choice? Well, we only want the user to have access to choices in the context of the closure, and after choosing is done there should be no choice values existing. In other words, the choices should only *live* as long as the `choose_suit` function. Sound familiar?

Let's use Rust's borrow checker to manage this:
```rust
// we use PhantomData so we don't have to actually store any additional data at runtime (this instructs Rust to treat Choice as if it held some value &'a (), where () is a Zero-sized type containing no data)
use std::marker::PhantomData;
pub struct Choice<'a, T>(T, PhantomData<&'a ()>);

impl<T> Choice<T> {
    // new private constructor, here we are not actually storing the guard reference, simply "linking" Choice to its lifetime (such that Choice lives at most as long as the guard)
    fn with_guard(t: T, _guard: &'_ ()) -> Choice<'_, T> {
        Choice(t, PhantomData)
    }
    // so the closure can preview the choice's value
    pub fn view(&self) -> &T {
        &self.0
    }
}
``` 
Then we can use this in our function:
```rust
impl<const N: usize> Suits<N> {
    // ...
    pub fn choose_suit<C>(&self, chooser: C)
    where
        C: FnOnce([Choice<'_, Suit>; N]) -> Choice<'_, Suit>
    {
        // our guard
        let guard = ();
        let choices = self.0.map(|suit| Choice::with_guard(suit, &guard));
        // have user choose some suit
        let Choice(suit) = chooser(choices);

        // do stuff with suit
        // ...
        // after we are all done, guard will be dropped here, so there will be no possible way for the Choice values to still exist beyond this call
    }
    // ...
}
```
If we try to use our previous smuggler example, we'll get an error that any Choice we try to smuggle out doesn't live long enough, and thus there will be no way for Choices to escape the closure.