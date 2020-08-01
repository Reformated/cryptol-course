# Introduction

`labs::CryptoProofs::CryptoProofs` introduced the concepts of 
_injectivity_ and _inversion_, among others.  In this lab, we build 
upon these concepts, introducing higher-order functions (functions 
that take functions as values) and specifying reusable property 
definitions that can be imported into other modules.

## Prerequisites

Before working through this lab, you'll need 
  * Cryptol to be installed,
  * this module to load successfully, and
  * an editor for completing the exercises in this file.

You'll also need experience with
  * loading modules and evaluating functions in the interpreter,
  * sequence and `Integer` types,
  * the `:prove` and `:sat` commands,
  * manipulating sequences using `#`, `take`, `split`, and `join`,
  * writing functions and properties,
  * lambda functions,
  * sequence comprehensions,
  * logical, comparison, and arithmetic operators,
  * lambda functions, and
  * `:prove` and `:sat` commands

## Skills You'll Learn

By the end of this lab you will be able to define higher-order 
functions (functions of functions) and predicates, and apply them to 
various cryptographic applications (e.g. decryption inverts 
encryption and most block ciphers are injective).
  
## Load This Module

This lab is a
[literate](https://en.wikipedia.org/wiki/Literate_programming) Cryptol
document --- that is, it can be loaded directly into the Cryptol
interpreter. Load this module from within the Cryptol interpreter
running in the `cryptol-course` directory with:

```sh
Cryptol> :m labs::Transposition::CommonPropertiesAnswers
```

The proofs in this lab require an array of different theorem provers
supported by Cryptol. In order to solve them, we recommend using the
Cryptol Docker container described in [INSTALL.md](/INSTALL.md) for 
this course.

The first line of a Cryptol module needs to be a module definition:

```cryptol
module labs::Transposition::CommonPropertiesAnswers where
```

In this lab, we'll redo exercises from 
`labs::CryptoProofs::CryptoProofs`, higher-order style, so we'll 
import the same dependency.  We'll give it an alias this time so 
Cryptol doesn't spew warnings about shadowing its definition of `f`:

```cryptol
import labs::CryptoProofs::CryptoProofsAnswers (known_key, known_ct, matched_pt, matched_ct, DESFixParity)
import specs::Primitive::Symmetric::Cipher::Block::DES (DES)
```

# Higher-Order Functions: Functions can be Values!

Let's count to 10...

```sh
labs::Transposition::CommonPropertiesAnswers> let _1 = 1 : Integer
labs::Transposition::CommonPropertiesAnswers> _1
1
labs::Transposition::CommonPropertiesAnswers> 1 + it
2
...
9
labs::Transposition::CommonPropertiesAnswers> 1 + it
10
```

That was repetitive.  Let's just ask Cryptol to count to 10:

```sh
labs::Transposition::CommonPropertiesAnswers> take`{10} (iterate (\x -> 1 + x) (1 : Integer))
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```sh

Adding 1 to something seems like a common task.  Let's name it...

```cryptol
S: Integer -> Integer
S x = 1 + x
```

Peano would be proud.  Wouldn't it be nice if Cryptol could just 
reuse S to repeatedly increment a counter?

```sh
labs::Transposition::CommonPropertiesAnswers> :t \(x : Integer) -> 1 + x
(\(x : Integer) -> 1 + x) : Integer -> Integer
labs::Transposition::CommonPropertiesAnswers> :t S
S : Integer -> Integer
```

Wait a minute...it can!

```sh
labs::Transposition::CommonPropertiesAnswers> take`{10} (iterate S 1)
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

So functions can be passed around as values.  Great!  A function that 
takes a function as a value and/or returns one as a result is a 
_higher-order_ function; a function that does not is _first-order_.
But why bother?

# Five Killer Apps: Silence of the Lambdas

From `labs::CryptoProofs::CryptoProofs`:

> | Proof | Invocation |
> |-|-|
> | Function reversal | `:sat \x -> f x == y` |
> | Proof of inversion | `:prove \x -> g (f x) == x` |
> | Proof of injectivity | `:prove \x y -> x != y ==> f x != f y` |
> | Collision detection | `:sat \x y -> f x == f y /\ x != y` |
> | Equivalence checking | `:prove \x -> f x == g x` |

Lambda functions are fun, but in order to reuse them, we would need 
to name them.  We could be silly and just assign directly to these 
lambda functions, filling in other parameters as necessary (and 
renaming some variables for some reason)...

| Proof | Invocation |
|-|-|
| Function reversal | `maps_to = \f y -> \x -> f x == y` |
| Proof of inversion | `inverts = \f' f -> \x -> f' (f x) == x` |
| Proof of injectivity | `injective = \f -> \x y -> x != y ==> f x != f y` |
| Collision detection | `collides = \f -> \x y -> f x == f y /\ x != y` |
| Equivalence checking | `equiv = \f f' -> \x -> f x == f' x` |

...but Cryptol supports a more familiar function definition syntax, 
so let's use that:

```cryptol
/**
 * `f` maps to `y` from `x`
 */
maps_to:
    {d, c} Cmp c => (d -> c) -> c -> d -> Bit
maps_to f y x =
    f x == y

/**
 * `f'` _inverts_ `f` iff when `f` maps `x` to `f x`, 
 * `f'` maps `f x` back to `x`.
 */
inverts:
    {d, c} Cmp d => (c -> d) -> (d -> c) -> d -> Bit
inverts g f x =
    g (f x) == x

/**
 * `f` is _injective_ iff it maps distinct elements of its domain `d`
 * to distinct elements of its codomain `d`
 */
injective:
    {d, c} (Eq d, Cmp c) => (d -> c) -> d -> d -> Bit
injective f x x' =
    x != x' ==> f x != f x'

/**
 * `x` and `x'` differ, but `f` maps them to the same value
 */
collides:
    {d, c} (Eq d, Cmp c) => (d -> c) -> d -> d -> Bit
collides f x x' =
    x != x' /\ f x == f x'
```

Cryptol went ahead and reserved infix operator `===` to denote that 
functions map the same value to the same value, so `:prove f === f'` 
is shorthand for `:prove \x -> f x == f' x`.  Functional equivalance 
must be pretty important!

# DES Exercises, Redux

Because the DES exercises were so much fun last time, let's revisit 
them with our newfound appreciation for higher-order functions!

**EXERCISE**: Use Boolector to reverse `DES.encrypt` for `known_key` 
and `known_ct` from `CryptoProofs`, using `maps_to`.  Avoid lambda.

```shell
labs::Transposition::CommonPropertiesAnswers> :s prover=boolector
labs::Transposition::CommonPropertiesAnswers> :sat maps_to (DES.encrypt known_key) known_ct
maps_to (DES.encrypt known_key) known_ct
  0x70617373776f7264 = True
(Total Elapsed Time: 0.773s, using Boolector)
```

(Hint: A curried function of two arguments can be viewed as a 
function of one argument to a function of one argument...)

**EXERCISE**: Use ABC to prove that `DES.encrypt` and `DES.decrypt` 
are mutual inverses for all possible `pt`, given the same `key`.
Do not mention `pt` in your proof commands.

```sh
labs::Transposition::CommonPropertiesAnswers> :s prover=abc
labs::Transposition::CommonPropertiesAnswers> :prove \key -> inverts (DES.decrypt key) (DES.encrypt key)
labs::Transposition::CommonPropertiesAnswers> :prove \key -> inverts (DES.encrypt key) (DES.decrypt key)
```

**EXERCISE**: Use Boolector to prove that `DES.encrypt` is injective 
for any given `key`.  Do not mention `pt`s in your proof command.  
Meditate on the nature of lambda and the (f)utility of names.

```sh
labs::Transposition::CommonPropertiesAnswers> :s prover=boolector
labs::Transposition::CommonPropertiesAnswers> :prove \key -> injective (DES.encrypt key)
```

**EXERCISE**: Use ABC to find two different keys and a single 
plaintext such that both keys encrypt that plaintext to the same 
ciphertext.  Mention `key`s to alias `DES.encrypt` in a `where` 
clause, but do not mention `key` in the lambda function you pass to 
your command.  Guess whether such a collision will be found before 
you observe a collision of black holes in nearby outer space.

```sh
labs::Transposition::CommonPropertiesAnswers> :s prover=abc
labs::Transposition::CommonPropertiesAnswers> :sat \pt -> collides (encrypt_ pt) where encrypt_ pt key = DES.encrypt key pt
```

**EXERCISE**: Use ABC to prove that the two keys you just found are 
equivalent keys; i.e., prove that keyed `DES.encrypt` and 
`DES.decrypt`, respectively, are equivalent for *all* `pt`/`ct`.  
Avoid lambda.

```sh
labs::Transposition::CommonPropertiesAnswers> :s prover=abc
labs::Transposition::CommonPropertiesAnswers> let { result = _, arg1 = pt, arg2 = key, arg3 = key_ } = it
labs::Transposition::CommonPropertiesAnswers> :prove (DES.encrypt key) == (DES.encrypt key_)
```

**EXERCISE**: Use ABC to prove that (DES.encrypt key) is equivalent 
to (DES.encrypt (DESFixParity)).  Do not mention `pt` in your 
proof command.

```sh
labs::Transposition::CommonPropertiesAnswers> :s prover=abc
labs::Transposition::CommonPropertiesAnswers> :prove \key -> DES.encrypt key === DES.encrypt (DESFixParity key)
```

**EXERCISE**: Finally, try to find a collision with keys of distinct 
parity; alternatively, try to prove that no such collision exists.  
(This wasn't in `CryptoProofs`, but let's be extra.)  Feel free to 
go crazy with lambda.  Ponder the abject hopelessness of classical 
crypto in a post-quantum world...

```sh
labs::Transposition::CommonPropertiesAnswers> :s prover=abc
labs::Transposition::CommonPropertiesAnswers> :sat \key key_ pt -> DESFixParity key != DESFixParity key_ /\ DES.encrypt key pt == DES.encrypt key_ pt
```

# Conclusion

This lab demonstrated higher-order functions and revisited the 
`CryptoProofs` with named higher-order properties.  Whether one 
chooses to use lambda notation or higher-order functions is mostly a 
matter of taste, though it seems useful to codify properties around 
which entire branches of mathematics have been studied...

# Solicitation

How was your experience with this lab? Suggestions are welcome in the
form of a ticket on the course GitHub page:
https://github.com/weaversa/cryptol-course/issues

# From here, you can go somewhere!

Up: [Course README](../../README.md)
Previous: [Crypto Proofs](/labs/CryptoProofs/CryptoProofsAnswers.md)
Next: [Transposition Ciphers, in the Abstract](TranspositionAnswers.md)