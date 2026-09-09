---
date: 2026-09-08
title: "Curry-Howard Correspondence: Prove Things With Programs"
description: "A small Intro To Curry-Howard Correspondence "
layout: page
collections: ["post", "math", "functional programming"]
---

The Curry-Howard correspondence can be stated roughly as follows:
> Proofs are programs, propositions are types

In this short post, I will try to explain why, without going too much into the formal mathematics (however interesting it might be). If however, you do want the mathematical side, I will leave some resources at the end of the post

We will first review some notation, then move on to definitions, and finally what the title means

# Definitions

**Proposition**: A mathematical statement which can be asserted or proven within a logical system. Keep in mind that we will be using an intuitionistic system of logic, rather than classical logic.

**Axiom**: Something that is assumed to be true, without prior proof in the system. This is usually assumed to prove other propositions. 

**Connectives**: Symbols that are used to create more complex proposistions from from simpler ones:
- **Conjunction (AND)**: Written A ∧ B represents having both A and B 
- **Disjunction (OR)**: Written A ∨ B represents having either A or B
- **Negation (NOT)**: Written ¬A, read as "not A" represents that assuming A to be true leads to a contradiction 
- **Implication ⇒**: Written as A ⇒ B, read as "A implies B" represents that a proof of A can be converted to a proof of B

These symbols are a bit cumbersome to write, so excuse me for using these as subsitutes:
- ∧ as &
- ∨ as |
- ¬ as ~
- ⇒ as ->

# What is a proof?
A proof can be said to be a step-wise derivation from premises to a conclusion. You assume the premises are true, then show that from that assumption, the conclusion necessarily follows.
Here's a classic (almost overused) example:
1. All men are mortal (Premise)
2. Socrates is a man (Premise)
3. Socrates is mortal (Conclusion)

From two premises, we conclude something new, namely that Socrates is mortal.

## What is a proof in mathematics?
Mathematically, the definition give above also works fine. 
In our discussion however, We will use a more concise symbol notation instead of typing the words out.

As an example, here's the law of excluded middle in two different forms:
> For any proposition, the proposition or its negation is true

Written symbolically:
> A | ~A

Note: This is where the difference between intuitionistic and classical logic comes out nicely. In classical logic, the law of excluded middle would be provable. But in intuitionistic logic, although it isn't outright rejected, it isn't be provable in the general case 

## What is a program
Someone might respond with "a set of instructions that tell the computer what to do"

This definition is correct, but not for our purposes. In the next section I will give a brief introduction to lambda calculus.

## Lambda Calculus
Wikipedia gives this definition:
> [...] lambda calculus is a formal system for expressing computation based on function abstraction and application [...]

(omitted for the sake of simplicity)
In other words: Lambda calculus is a system, in which you can describe computation (much like a program in a programming language) using only function definition, their applications and variables.

Although for simplicity and clarity, I will occasionally use numbers and arithmetic operators even though they aren't primitive to the calculus 

**Function**: Functions are entities in this system, which take in a single argument, and produce a result. They are nameless (although there are systems in which you can give them names).

Functions that need to take in multiple arguments can be made by returning another function
The definitions of a function is also called function abstraction.

As an example, here is the identity function (which takes in a single argument, and returns it back):
```
λx.x
```
Keep this function in mind however, we will be coming back to it.

Anyway let's dissect this guy:
- λ : Think of this as a function declaration keyword, much like ``def`` in Python or ``fn`` in Zig or Rust
- the 'x' after the 'λ': The symbol after the 'λ' but before the '.' is the argument of the function. Here, the function takes a single argument named 'x'
- '.' (the dot): This tells that the function body is about to begin
- the final 'x': This is the value that the function will compute and return.

Here are some more examples:
1. A successor function, which adds 1 to the argument (assuming the argument is an integer)
```
λx. x+1
```

2. A square function
```
λx. x*x
```

3. A function which returns a function (parenthesis added for clarity, and are optional here)
```
λx.(λy.y+1)
```

Note that pure lambda calculus don't have operators like * and + built into them; I'm using an extended notation here just to keep things simple

**Function Application**: This is just calling the function, with some argument(s). Or, *applying* a function to some arguments.
We will take the above examples, and show how to apply them here:

1. Using the identity function on the number 1, which finally returns 1 as the result
```
(λx.x)1
```

2. This outputs 7
```
(λx.x+1)6
```

3. I will edit this one a bit:
```
λx. λy. y*x
```
This is a function which takes in a number (call it x), and returns another function which takes in a number (call it y) and returns y*x
The equivalent Python code:
```py
lambda x: lambda y: y * x
```
or simply
```py
lambda x,y: y * x # although this isn't *exactly* the same as the previous one, because this one returns a single function with two arguments. 
                  # The above one returns a function which takes in a single argument. Try it out for yourself!
```
Example:
```
(λx. λy.y*x)2
```

Is equivalent to doing
```
λy.y*2
```
And doing
```
((λx.λy.y*x)2)2
```
is equivalent to doing:
```
2*2
```
or simply
```
4
```
This example is little extra, so don't worry if you dont get it. Try out the equivalent python code if you want.

## Adding types to our little language
Let's add a notation of types. Given an entity ``x``, we can say that it has the type ``T`` by writing:
```
x:T
```
read as "x has type T".
As a concrete example: ``x: int`` means ``x`` has type ``int``.

Functions too have types in simply-typed lambda calculus.
A type of function is written as: 
```
<type of argument> -> <return type>
```
Example of the beloved identity function:
```
(λx.x) : A -> A
```
Read as "... has type of a function that takes in a value of type A and returns a value of type A"

The more complex 3rd example has the type:
```
integer -> integer -> integer
```
Again, don't worry if you don't understand this, it isn't important for what we need to do here.

# Behold, the correspondence!
Now we come to the meat (or veggies if you prefer) of our post. The Curry-Howard correspondence, says that propositions in logic correspond to types, and proof involving those propositions are programs of those types. 

So when you write a lambda calculus program like the identity function, you are actually writing a proof of the law of identity. 

Consider the law of identity, written symbolically:
> A -> A

And consider the type of the identity function in lambda calculus:
> A -> A

Notice how similar they look!

So here's the basic idea: The proofs in logic in a way correspond to types of a programming language (in this case, lambda calculus).

Type checking means corresponds to checking if the proof is valid, and running the program corresponds to normalizing the proof! This remarkable result lays one of the foundations for ideas that allow you to create things like proof assistants such as Rocq and Lean (be sure to check them out, they're wonderful)

We can actually go further, and get the corresponding lambda calculus counter-parts for each of the logical constructs.

- **Proposition named A**: the type A
- **Implication of kind A implies B**:  Function/λ abstraction of the type ``A -> B``
- **Negation of A**: Function of type ``A -> ⊥`` (where ``⊥`` corresponds to falsehood. It is also called ``empty`` type, and no values have this type). If your logical system somehow proves falsehood, then it is inconsistent and n longer useful.
- **Conjunction A & B**: Pair/Tuple of type (A,B). (Pair with first element of type A, and second element of type B). You can also use the product type AxB, which is essentially the same thing.
- **Disjunction A | B**: A variant type A | B. Here, "variant" means a C++-like variant, where only field is active at a time. So it would mean either type A is active, or type B is active, which corresponds to either A is true or B is true respectively. It is also called the sum type.
- **Proof of A**: Program/Term with type A.

Now we understand that for intuitionistic propositional logic, we can systematically translate the propositions into types and proofs into terms of those types.

And where does this all lead us to?
> Proofs are programs, propositions are types 

# Some subtelties 

Now I'd love to leave it at that ending, it felt quite cinematic in my opinion. But I do want to clarify an important question someone might have when first encountering this idea:
> 1. If I define my identity function as ```λx.x+1```, won't this still be of the type ``A->A`` but not be the identity function anymore, so it won't be the proof for the identity function?

If you do make that change to the function, then yes, it ceases to be the identity function and becomes something else entirely (namely the successor function)

But what about the types? Won't it still be ``A -> A``? No. Look at the body of the abstraction. It's ``x+1``, and the ``+`` operation is only defined for numeric types. Therefore, the type of the abstraction now becomes ``numeric -> numeric``, which isn't the identity function (which takes in an argument of any type and returns it)

> 2. Okay, but what if i do this instead: ```λx. y``` where ``x: A`` and ``y:A``? In that case, the type of the abstraction is still ``A -> A`` but it isn't the identity function

I answer that: Yes, both the statements are correct. The Curry-Howard correspondence doesn't say that every term of the type ``A->A`` is an identity function. For the above function to have ``A -> A``, we'd already need to know that ``y:A``. In the words of simply typed lambda calculus, the type of the abstraction would rather be ``y: A ⊢ λx.y : A->A`` which again, isn't exactly the identity function. (this short post hasn't covered this notation, but it's basically saying that ``y:A``  must be known in the context of the abstraction)

Note that ``A`` here is a placeholer for a type (it is a typing variable) and there are no other operations or values involving ``A``, the only thing we can do with it is to simply return it,  because we cannot be sure that something like adding 1 to it is type-safe.

# Further reading
For reading about lambda calculus, I reccommend Types and Programming Languages by Benjamin C. Pierce. I personally used it (and still am), and it's a wonderful read. It covers lambda calculus from the ground up, as well typing them, and much more 

To read about the correspondence:
- [This is a very good article](https://web2.qatar.cmu.edu/cs/15317/lectures/04-curryhoward.pdf)
- [Another short intro](https://cklixx.people.wm.edu/teaching/math400/Wesley-P1.pdf)
- [This is a lengthy one, but a very comprehensive collection of lecture notes](https://disi.unitn.it/~bernardi/RSISE11/Papers/curry-howard.pdf)
