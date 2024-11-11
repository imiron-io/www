+++
date = '2024-11-11T14:42:50+09:00'
title = 'Pure Calculation vs Effects in Haskell: A Path to Clearer Code'
author = "James Haydon"
+++

When writing Haskell applications, one of our greatest tools is the type system's ability to cleanly separate pure calculations from effectful operations. Let's explore how to leverage this separation to create more maintainable and testable code.

## The Core Principle

The key idea is to push as much calculation as possible into pure functions, leaving our monadic code to focus solely on sequencing effects. This makes both parts easier to reason about and test.

## A Common Anti-Pattern

Consider this typical first attempt at writing a function that processes user data:

```haskell
processUserData :: String -> IO String
processUserData input = do
    let cleaned = filter isAlpha input
    let normalized = map toLower cleaned
    putStrLn "Processing user data..."
    writeFile "audit.log" $ "Processed: " ++ input
    return $ reverse normalized
```

While this works, it mixes pure calculations (cleaning, normalizing, reversing) with effects (printing, writing to files). This makes it harder to test the logic and understand the flow of effects.

## A Better Approach

Let's separate this into pure calculation and effect handling:

```haskell
-- Pure calculation
processString :: String -> String
processString = reverse . map toLower . filter isAlpha

-- Effect handling
processUserData :: String -> IO String
processUserData input = do
    putStrLn "Processing user data..."
    writeFile "audit.log" $ "Processed: " ++ input
    return $ processString input
```

## Benefits in Complex Scenarios

This pattern becomes even more valuable in complex business logic. Here's a more substantial example:

```haskell
-- Pure Types
data UserInput = UserInput 
    { name :: String
    , age :: Int 
    }

data ProcessedData = ProcessedData 
    { normalizedName :: String
    , ageCategory :: String
    , isEligible :: Bool
    }

-- Pure business logic
processUserInput :: UserInput -> ProcessedData
processUserInput UserInput{..} = ProcessedData
    { normalizedName = map toLower $ filter isAlpha name
    , ageCategory = categorizeAge age
    , isEligible = age >= 18 && length name > 0
    }
  where
    categorizeAge a
        | a < 18 = "Minor"
        | a < 65 = "Adult"
        | otherwise = "Senior"

-- Effect handling
handleUserRegistration :: UserInput -> IO Bool
handleUserRegistration input = do
    let processed = processUserInput input
    
    logAction $ "Processing registration for: " ++ name input
    
    if isEligible processed
        then do
            saveToDatabase processed
            notifyAdmin processed
            return True
        else do
            logAction "Registration rejected"
            return False
```

## Testing Benefits

The pure calculation functions are trivial to test:

```haskell
testProcessUserInput :: Test
testProcessUserInput = TestList
    [ TestCase $ assertEqual "Normal case"
        (ProcessedData "john" "Adult" True)
        (processUserInput $ UserInput "John!" 25)
    , TestCase $ assertEqual "Edge case"
        (ProcessedData "" "Minor" False)
        (processUserInput $ UserInput "" 15)
    ]
```

## Handling Complex Effects

When dealing with multiple effects, this separation becomes even more valuable. Consider a function that might need to interact with multiple services:

```haskell
data UserProfile = UserProfile { ... }
data EmailContent = EmailContent { ... }

-- Pure calculations
generateEmailContent :: UserProfile -> EmailContent
generateEmailContent = ...

-- Effect handling
handleUserUpdate :: UserProfile -> IO ()
handleUserUpdate profile = do
    let emailContent = generateEmailContent profile
    
    -- Effects are clearly separated and sequential
    saveToDatabase profile
    sendEmail emailContent
    updateCache profile
    notifyDownstreamServices profile
```

## Conclusion

By maintaining this separation, we get:
- Easily testable pure functions
- Clear effect sequences
- Better type safety
- More maintainable code

The extra type definitions and function splits are a small price to pay for these benefits. When you encounter complex business logic mixed with effects, consider whether you can separate them using this pattern.

This approach aligns well with functional programming principles and helps create code that's both robust and easy to understand.
