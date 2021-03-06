---
layout: default
title: Examples
permalink: /cardano/plutus/examples/
group: cardano-plutus
---
Here we'll take a look at some of the common examples of programs to give you a
better feel for how the Plutus language works. We'll implement Peano numerals,
cons lists, and binary trees, together with some common functions relating them.

To start, let's define Peano numerals:

    data Nat = { Zero | Suc Nat }

The naturals support a variety of functions, of course, such as addition,
multiplication, factorial, and fibonacci, which are typical example programs.

    add : Nat -> Nat -> Nat {
      add Zero n = n ;
      add (Suc m) n = Suc (add m n)
    }

    mul : Nat -> Nat -> Nat {
      mul Zero _ = Zero ;
      mul (Suc m) n = add (mul m n) n
    }

    fac : Nat -> Nat {
      fac Zero = Suc Zero ;
      fac (Suc n) = mul (Suc n) (fac n)
    }

    fib : Nat -> Nat {
      fib Zero = Suc Zero ;
      fib (Suc Zero) = Suc Zero ;
      fib (Suc (Suc n)) = add (fib n) (fib (Suc n))
    }

Cons lists are also a familiar type:

    data List a = { Nil | Cons a (List a) }

This demonstrates the use of parametric types, where he, `List a` has a type
parameter `a` for the type of elements. So, for example, `List Nat` is the type
of lists of Peano numerals.

Lists support a variety of functions, such as `length`, `append`, and `map`:

    length : forall a. List a -> Nat {
      length Nil = Zero ;
      length (Cons _ xs) = Suc (length xs)
    }

    append : forall a. List a -> List a -> List a {
      append Nil ys = ys ;
      append (Cons x xs) ys = Cons x (append xs ys)
    }

    map : forall a b. (a -> b) -> List a -> List b {
      map _ Nil = Nil ;
      map f (Cons x xs) = Cons (f x) (map f xs)
    }

Here we can see the use of polymorphism in Plutus. These functions work for any
list, regardless of the element type, so we can abstract over the element type
by using `forall`. So for instance, the type of `length` says that for any
choice of `a`, we have a function of type `List a -> Nat`.

It's important to note that in Plutus, this polymorphism exists only for the
declaration of values. Any time you use a polymorphically-declared value, the
choice of the type variable must be fixed by the use site. You can't treat these
declarations as giving polymorphic values in general, as in System-F. Rather,
a polymorphic type in a declaration is an abbreviation for an infinite family of
identical definitions that differ only in the choice of that type variable. For
example, we could define multiple `length` functions like so:

    lengthNat : List Nat -> Nat {
      lengthNat Nil = Zero ;
      lengthNat (Cons _ xs) = Suc (lengthNat xs)
    }

    lengthBool : List Bool -> Nat {
      lengtBool Nil = Zero ;
      lengthBool (Cons _ xs) = Suc (lengthBool xs)
    }

    lengthListNat : List (List Nat) -> Nat {
      lengthListNat Nil = Zero ;
      lengthListNat (Cons _ xs) = Suc (lengthListNat xs)
    }

And they're all identical except the name and the choice for `a`. This is of
course redundant, so we can use the polymorphic declaration given above. But,
this declaration does not give us a value `length` with the type
`forall a. List a -> Nat`. Instead, it gives us that entire infinite family of
declarations, but with a convenient abbreviation syntax. This is why the use of
such polymorphic declarations requires the choice of the type variables to be
fixed at the use site.

Another common type is the type of binary trees with data in the branches:

    data Tree a = { Leaf | Branch a (Tree a) (Tree a) }

Such trees support functions such as `count`, `traversal`, and `reverse`:

    count : forall a. Tree a -> Nat {
      count Leaf = Zero ;
      count (Branch _ l r) = Suc (add (count l) (count r))
    }

    traversal : forall a. Tree a -> List a {
      traversal Leaf = Nil ;
      traversal (Branch x l r) = Cons x (append (traversal l) (traversal r))
    }

    reverse : forall a. Tree a -> Tree a {
      reverse Leaf = Leaf ;
      reverse (Branch x l r) = Branch x (reverse r) (reverse l)
    }
