+++
date = '2024-12-19T14:42:50+09:00'
title = 'Keep calculations out of your effectful code'
author = "James Haydon"
tags = ["Tech Blog", "haskell"]
+++

When discussing function purity (vs effects), we often hear that we should strive for purity because it makes for code which is easier to test and reason about. But this advice can feel impractical when the whole point of our code is to produce side-effects like printing output. We can't just _not_ do that.

What we should aim for is the "pure core, imperative shell" pattern, trying to seperate the effectful code from the pure code. To keep this split, the mantra I used to have in my mind while coding was _"try to keep effects out of the core"_, or even just _"try not to use effects"_. But looking back this didn't seem to help me much and was misguided: most of the time when you want to use effects, you do actually need them[^1]. So these days I've reframed this into: _"try to keep pure code out of the imperative shell"_. That is, your effectful functions should focus solely on orchestrating side effects, with all pure calculations handled in separate functions. Unpacking this produces very concrete guidance, that could even be made into a linter (but I'm not sure that would be a good idea).

## Effect categories

(You can skip this section, it's just some category theory background.)

It's worth taking a short detour through some theory, to understand what is essential to effectful programming, and what isn't. (For more on this, see my [categorical programming language](https://github.com/jameshaydon/lawvere "GitHub").)

Pure computation (what I like to call "calculation") is neatly modelled by a cartesian closed category `C`. Effectful computation is then modelled as a category `E` with a functor `lift : C -> E` which lifts pure computations into effectful ones, with certain properties. This is then called an _effect category_ or a [Freyd category](https://ncatlab.org/nlab/show/Freyd+category "nlab") over `C`. The category `E` is usually not cartesian closed, in fact it usually doesn't even have products. But it usually _does_ have coproducts that are compatible with the underlying category `C`. So when defining morphisms in `E` we can:

- compose morphisms, because it's a category,
- lift pure computations from `E`,
- do _branching_ (because it has coproducts),
- maybe other stuff like recursion (depending on the sort of effects).

In this style of programming, it's very obvious where the pure calculations are coming from, they all come from `lift`. Of course one could also replicate to some extent these calculations in `E`, but since `E` is lacks even products, this isn't practical.

In Haskell, the above manifests itself as the `Arrow` typeclass (see e.g. [Freyd is Kleisli, for Arrows](https://group-mmm.org/~ichiro/papers/arrow-algebras.pdf) (by _Bart Jacobs_ and our co-founder _Ichiro Hasuo_)):
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
- The categorical composition corresponds to the sequencing of effects with data dependency (with `.`, i.e. `>=>` for a Kleisli category)
- lifting pure computations (`arr`)
- carrying pure values in parallel to effectful computations (`first`, `second`, ..)
- branching (`|||`)
- possibly more (see e.g. `ArrowLoop`)

The principle I'm advocating for is simple: when lifting calculations via using `arr`, we don't define these inline, instead we refer to a defined function, which can therefore be independently tested and documented. 

But writing code with these methods directly is cumbersome, which is why Haskell has arrow-notation and `do`-notation. So let's see how this principle translates to `do`-notation next.

## In Practice

Let's look at what this means for typical monadic code using do-notation. Your effectful code should only consist of only the following:

1. Sequencing effects, i.e. calling other effectful function and using bind `<-`,
2. Branching (`if _ then _ else _` or `case _ of`),
3. Invoking pure functions in two places:
  - As the scrutinee of some branching code,
  - In the arguments to other effectful functions.

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
    putStrLn ("The sum is: " <> show (x + y))
```

According to the above rules, the illegal thing here is `putStrLn ("The sum is: " ++ show (x + y))`, because `calculator` is meant to only orchestrate effects, but it contains a pure calculation: `"The sum is: " ++ show (x + y)`. So, we must move this calculation to another function:

```haskell
calculate :: Int -> Int -> String
calculate x y = "The sum is: " <> show (x + y)

calculator :: IO ()
calculator = do
    putStrLn "Enter first number:"
    x <- getInt
    putStrLn "Enter second number:"
    y <- getInt
    putStrLn (calculate x y)
```

By moving all calculations out of the effectful functions, they become less noisy.

## More Complex Example: User Registration

Let's look at a more realistic example , some code to register a user to some web service. First, here's a version of the code which mixing calculations and effects:

```haskell
data User = UserInput
  { username :: String,
    email :: String,
    passwordHash :: String,
    createdAt :: UTCTime
  }

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
data User = UserInput
  { username :: String,
    email :: String,
    passwordHash :: String,
    createdAt :: UTCTime
  }

data UserInput = UserInput
  { username :: String,
    email :: String,
    password :: String,
    currentTime :: UTCTime
  }

validUsername :: String -> Bool
validUsername username = length username >= 3

validEmail :: String -> Bool
validEmail email = ('@' `elem` email) && ('.' `elem` dropWhile (/= '@') email)

validPassword :: String -> Bool
validPassword password =
  any isUpper password && any isLower password && length password >= 8

newUser :: UserInput -> User
newUser UserInput {..} =
  User {
    passwordHash = hashPassword password,
    createdAt = currentTime,
    ..
  }

welcomeMessage :: String -> String
welcomeMessage username =
  "Welcome " ++ username ++ "! Registration successful!"

passwordMatch :: String -> String -> Bool
passwordMatch = (==)

registerUserGood :: IO ()
registerUserGood = do
  username <- question usernamePrompt
  if validUsername username
    then do
      email <- question emailPrompt
      if validEmail email
        then do
          password <- question passwordPrompt
          if validPassword password
            then do
              passwordConfirm <- question confirmPasswordPrompt
              if passwordMatch password passwordConfirm
                then do
                  now <- getCurrentTime
                  saveUserToDB (newUser UserInput {currentTime = now, ..})
                  putStrLn (welcomeMessage username)
                else putStrLn passwordMatchErr
            else putStrLn passwordLengthErr
        else putStrLn emailFormatErr
    else putStrLn usernameFormatErr
  where
    usernamePrompt = "Enter username (minimum 3 characters):"
    emailPromp = "Enter email:"
    passwordPrompt = "Enter password (minimum 8 characters, must contain upper and lowercase):"
    confirmPasswordPrompt = "Confirm password:"
    passwordMatchErr = "Passwords don't match!"
    passwordLengthErr = "Password must be at least 8 characters and contain upper and lowercase letters! Please try again."
    emailFormatErr = "Invalid email format! Please try again."
    usernameFormatErr = "Username must be at least 3 characters! Please try again."
```

This example highlighted some things that crop up when trying to follow this style:
- _Literals:_ All the literal strings have been lifted out. For small literals, you might choose to keep them in the effectful code.
- _Does `UserInput {currentTime = now, ..}` beak the rules?_ Some functions you invoke (effectful or pure) take a product type as input. Often you have to assemble the input from various sources, this is part of the orchestration. When using `Arrow` and desuraging `proc`-notation, GHC will use a lot of tuples and tuple-manipulating functions. Similarly it's okay to deconstruct tuples/records in patterns.
- In this example the code has become quite nested, there are ways around this (e.g. [The trick to avoid deeply-nested error-handling code](https://www.haskellforall.com/2021/05/the-trick-to-avoid-deeply-nested-error.html "Haskell for all")), but I didn't want to detract from the main point of this blog post.
- You can break the rules in small places. For example sometimes you want to one branch of an `if _ then _ else _` to appear before the other, and so you need to use a `not` in the condition.
- Note also that following the principles guides us to creating a new type `UserInput`, for collecting all the data that needs to be passed to the `newUser` calculation.

## Conclusion

Separating effects from calculations isn't just theoretical advice - it's a practical approach to writing more maintainable code. By moving calculations out of the effectful functions, we've created code that follows many other principles for maintainable code: small, focused functions, menaingful intermediate types, independently testable units, etc.

[^1]: I'm not talking here about using a mutable reference cell when you don't need one, of course. I'm talking about majing network requests, printing to `stdout`, etc.
