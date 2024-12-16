+++
date = '2024-11-21T14:42:50+09:00'
title = 'Pure Calculation vs Effects in Haskell: A Path to Clearer Code'
author = "James Haydon"
tags = ["Tech Blog", "haskell"]
+++

We often hear that we should "strive to keep our functions pure" because they're easier to test, referentially transparent, and easier to reason about. Indeed sometimes pure functional programming can acheive the same thing as imperative code with lots of mutation. But this advice can sometimes feel non-actionable, if the whole point of my code is to run some effects, like print some output for the user, then I can't _not_ do that. But there is still something to do, that is more subtle, which is to try to keep effectful and pure code kept separate to some extent. When talking about high-level code organisation, this is sometimes referred to as _"pure core, imperative shell"_. Probably because of the fact that sometimes it _is_ possible to completely banish effects, for quite a while I thought of this as _"keep effects out of pure functions"_. But this isn't actually very actionable, and (in Haskell at least) easily checked by the compiler. What I've found much more helpful, especially when coding those "imperative shell" parts of a codebase, is to instead _"keep pure code out of effectful functions"_.

Your effectful functions should focus solely on managing effects, with all _calculation_ (a term I use for "pure computation") handled separately. Unpacking this produces very concrete guidance, that could even be made into a linter (but I'm not sure that would be a good idea).

## Effect categories

(If you don't want to hear about category theory, feel free to skip this section.)

It's worth taking a short detour through some theory, to understand what is essential to effectful programming, and what isn't. (For more on this, see [categorical programming](https://github.com/jameshaydon/lawvere "GitHub").)

Pure computation ("calculation") is neatly modelled by a cartesian closed category `C`. Effectful compuation is then modelled as a category `E` such that there is a function `lift : C -> E` which lifts pure computations into effectful ones, with certain properties. This is then called an "effect category" or a [Freyd category](https://ncatlab.org/nlab/show/Freyd+category "nlab"). The category `E` is usually not cartesian closed, in fact it usually doesn't even have products. But it usually _does_ have coproducts that are compatible with the underlying category `C`. So when defining morphisms in `E` we can:

- compose morphisms, because it's a category,
- do _branching_ (because it has coproducts),
- lift pure computations from `E`,
- maybe other stuff like recursion (depending on the sort of effects).

In this style of programming, it's very obvious where calculations are coming from, they all come from `lift`. Of course one could also replicate to some extent these calculations in `E` itself, but since `E` is even lacking products, this isn't practical.

In Haskell, the above manifests itself as the `Arrow` class ([Freyd is Kleisli, for Arrows](https://group-mmm.org/~ichiro/papers/arrow-algebras.pdf) (Bart Jacobs and Ichiro Hasuo)):
- `arr`: lifting pure functions
- `first` and `second`: "wiring" functions for carrying pure values
- `(|||)`: handling coproducts (branching) via (in `ArrowChoice`)

```haskell
class Category cat where
    id :: cat a a
    (.) :: cat b c -> cat a b -> cat a c

class Category arr => Arrow arr where
    arr :: (a -> b) -> arr a b
    first :: arr a b -> arr (a, c) (b, c)
    -- ... other methods

class Arrow a => ArrowChoice a where
    (|||) :: a b d -> a c d -> a (Either b c) d
    -- ... other methods
```

Writing code using these abstractions directly gives us insight into the essential constructs of effectful code:
- sequencing of effects with data dependency (with `.`, i.e. `>=>` for a Kleisly category)
- lifting pure computations (`arr`)
- carrying pure values in parallel to effectful computations (`first`, `second`, ..)
- branching (`|||`)
- possibly more (see e.g. `ArrowLoop`)

The principle I'm advocating for is simple: when lifting calculations via using `arr`, we don't define these inline, instead we refer to a defined function, which can therefore be independently tested and documented. 

But writing code with these methods directly is cumbersome, which is why Haskell has arrow-notation and `do`-notation. So let's see how this principle translates to `do`-notation next.

## In Practice

Let's look at what this means for typical monadic code using do-notation. Your effectful code should only consist of only the following:

1. Sequencing effects, i.e. using the bind `<-`,
2. Branching (`if _ then _ else _` or `case _ of`),
3. Invoking pure functions in two places:
  - As the scrutinee of some branching code,
  - In the arguments to effectful functions.

This will make your effectful code look a certain way which is very easy to read.

## A Concrete Example: Calculator

Here's a simple calculator that reads two numbers and prints their sum. The first version does it all at once:

```haskell
calculator :: IO ()
calculator = do
    putStrLn "Enter first number:"
    x <- getInt
    putStrLn "Enter second number:"
    y <- getInt
    putStrLn ("The sum is: " ++ show (x + y))
```

According to the above rules, the illegal thing here is `putStrLn ("The sum is: " ++ show (x + y))`, because `calculator` is meant to be "purely effectful", but it contains a calculation `"The sum is: " ++ show (x + y)`. This calculation must be moved to a pure function: 

```haskell
calculate :: Int -> Int -> String
calculate x y = show (x + y)

calculator :: IO ()
calculator = do
    putStrLn "Enter first number:"
    x <- getInt
    putStrLn "Enter second number:"
    y <- getInt
    putStrLn (calculate x y)
```

So code written in this style (and `do`-notation) looks like:
- a sequence of binds
- possibly branching (though see [The trick to avoid deeply-nested error-handling code](https://www.haskellforall.com/2021/05/the-trick-to-avoid-deeply-nested-error.html "Haskell for all") for tips on keeping effectful code linear)
- invocations of other effectful functions _whose arguments are invocation of pure functions_, `launchMissiles` like `putStrLn (calculate x y)`

## More Complex Example: User Registration

Let's look at a more realistic example involving user registration. First, here's a version of the code, mixing calculations and effects:

```haskell
registerUserBad :: IO ()
registerUserBad = do
  username <- question "Enter username (minimum 3 characters):"
  if length username < 3
    then putStrLn "Username must be at least 3 characters! Please try again."
    else do
      email <- question "Enter email:"
      if not (('@' `elem` email) && ('.' `elem` dropWhile (/= '@') email))
        then putStrLn "Invalid email format! Please try again."
        else do
          password <- question "Enter password (minimum 8 characters, must contain upper and lowercase):"
          if not (any isUpper password && any isLower password && length password >= 8)
            then putStrLn "Password must be at least 8 characters and contain upper and lowercase letters! Please try again."
            else do
              passwordConfirm <- question "Confirm password:"
              if password /= passwordConfirm
                then putStrLn "Passwords don't match! Please try again."
                else do
                  now <- getCurrentTime
                  let user =
                        User
                          { username = username,
                            email = email,
                            passwordHash = hashPassword password,
                            createdAt = now
                          }
                  saveUserToDB user
                  putStrLn $ "Welcome " ++ username ++ "! Registration successful!"
```

And here is the improved version:

```haskell
validUsername :: String -> Bool
validUsername username = length username >= 3

validEmail :: String -> Bool
validEmail email = ('@' `elem` email) && ('.' `elem` dropWhile (/= '@') email)

validPassword :: String -> Bool
validPassword password =
  any isUpper password && any isLower password && length password >= 8

data UserInput = UserInput
  { username :: String,
    email :: String,
    password :: String,
    currentTime :: UTCTime
  }

newUser :: UserInput -> User
newUser UserInput {..} = User {passwordHash = hashPassword password, createdAt = currentTime, ..}

welcomeMessage :: String -> String
welcomeMessage username =
  "Welcome " ++ username ++ "! Registration successful!"

passwordMatch :: String -> String -> Bool
passwordMatch = (==)

registerUserGood' :: IO ()
registerUserGood' = do
  username <- question "Enter username (minimum 3 characters):"
  if validUsername username
    then do
      email <- question "Enter email:"
      if validEmail email
        then do
          password <- question "Enter password (minimum 8 characters, must contain upper and lowercase):"
          if validPassword password
            then do
              passwordConfirm <- question "Confirm password:"
              if passwordMatch password passwordConfirm
                then do
                  now <- getCurrentTime
                  saveUserToDB (newUser UserInput {currentTime = now, ..})
                  putStrLn (welcomeMessage username)
                else putStrLn "Passwords don't match! Please try again."
            else putStrLn "Password must be at least 8 characters and contain upper and lowercase letters! Please try again."
        else putStrLn "Invalid email format! Please try again."
    else putStrLn "Username must be at least 3 characters! Please try again."
```

## Remarks

This example highlighted some things that crop up when trying to follow this style:
- _What about literals?_ In the example I left these in the effectful code, e.g. the string literal `"Enter username (minimum 3 characters):"`. Technically these should also be lifted out, e.g. defined as `usernamePrompt`. I think it depends on the situation.
- _Doesn't `UserInput {currentTime = now, ..}` beak the rules?_ Creating product types is IMO part of the shuttling of values to the correct place during the effectful code. When desuraging arrow-notation, a lot of tuples and tuple-manipulating functions will be created. Similarly it's okay to deconstruct tuples/records in patterns.
- 

## Conclusion

Separating effects from calculations isn't just theoretical advice - it's a practical approach to writing more maintainable code. By keeping your effectful code focused solely on managing effects and moving all calculations to pure functions, you'll find your code easier to test, reason about, and modify.

Next time you're writing effectful code, ask yourself: "Is this really about managing effects, or am I mixing in calculations that could be pure?" The answer will guide you toward better code organization.


