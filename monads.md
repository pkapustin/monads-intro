# Disclaimer

Disclaimer: my first attempt at this, super informal, intended to be a quick intro to provide some initial intuition.

# Introduction

I would say that monads allow to compose a part of your program from smaller steps of similar type by chaining them together. When using a monadic interface, you can decide what the next steps will be based on the results of a previous step.

In Haskell, you define what the types of steps and results are, as well as how exactly the chaining works by implementing the `Monad` type class (Haskell's take on interfaces) for a specific type.

By extracting this "glue" (what happens in between the steps) into the monad definition, you can make your code look more concise and also more linear, as otherwise you would often need to do some branching after each step (for example, to check whether the step has succeeded or failed before proceeding to the next step).

# Pattern

To understand the pattern, it might be interesting to take a look at some of the functions at the top of the Elm search, with andThen function defining this "glue".
https://klaftertief.github.io/elm-search/?q=andThen

While Elm follows the same pattern without explicitly defining it, Haskell does with the `Monad` type class. In Haskell, the `andThen` function is called `bind`, denoted `>>=`, meaning that the function "binds" the result of the step to the first argument of the `computationRest` function, making it available to the rest of the computation.

First, some notes on syntax. In Haskell, functions are called without parentheses, like `f a`. All type names are lowercase. Type parameters are written without parentheses: `m a`.
In function signatures, parameters are separated by `->`, what comes after the last `->` is the return value. 

Here is the signature for the `>>=` function (as long as it is an operator, it is taken in parentheses):

```haskell
(>>=) :: m a -> (a -> m b) -> m b
(>>=) step computationRest = ...
```

In words, this means that if you have:
- a computation step (of type `m`) that yields a result of type `a` (first argument to `>>=`)
- a function that can use this result in the rest of the step sequence (second argument to `>>=`)
To write the bind function for a given monad, you need to define how to glue these together (producing the return value of `>>=`).

Let's take a look at some examples.

# Chaining optional values

Optional values are often modeled in Haskell using Maybe type that is defined like this:

```haskell
Maybe a = Just a | Nothing
```

This means that either you have a value of type `a`, or you don’t have any value.

Here is the definition of `>>=` for `Maybe` type:

```haskell
step >>= computationRest = case step of
 Nothing -> Nothing
 Just value -> computationRest value
```

In words, if we don't have a value, we stop and return `Nothing`, if we have a value, we use it by passing it to the `computationRest` function.

Suppose we have two functions that return optional values (`parseUserId` and `lookupUser`).

```haskell
parseUserId :: String -> Maybe Int
lookupUser :: Map Int User -> Int -> Maybe User
```

We can now write `getUser` as chaining of steps:

```haskell
getUser strUserId userList =
parseUserId strUserId >>= lookupUser userList
```

Note that here we don't need to explicitly check whether `parseUserId` succeeds, because this code is now moved to the `>>=` function.

Also, remember the signature of `>>=`:
```haskell
(>>=) :: m a -> (a -> m b) -> m b
```
In this example, `m` instantiates to `Maybe`, `a` to `Int` and `b` to `User`, so this becomes:

```haskell
(>>=) :: Maybe Int -> (Int -> Maybe User) -> Maybe User
```

Using do-notation syntax sugar, this can also be written the following way:

```haskell
getUser strUserId userList = do
 userId <- parseUserId strUserId
 lookupUser userList userId
```

Here the result of parsing the user id is bound to the name `userId`, so it can be used in the rest of the step sequence.

# IO

As Haskell is a pure language, functions cannot perform side effects, and IO operations (files, databases, network, etc.) are modeled as values describing actions that should be performed by the runtime. Each IO action may perform side effects, some of them also produce results. This can be naturally represented with monads, and explains the fact why monads are so common in Haskell, as you need to use them to be able to perform any kind of IO.

```haskell
getLine :: IO String
putStrLn :: String -> IO ()
```

This says that `getLine` is an IO action that, when executed, yields a `String` (step type is `IO`, result type is `String`), while `putStrLn` is a function that takes a `String` and returns an IO action that doesn’t produce any result (step type is `IO`, result type is `()`, which is void in Haskell).

One can naturally chain them together, binding the result of `getLine` in the rest of the computation ::

```haskell
readAndPrint :: IO ()
readAndPrint = getLine >>= putStrLn
```

Remembering the signature of `>>=`:
```haskell
(>>=) :: m a -> (a -> m b) -> m b
```
Here `m` is `IO`, `a` is `String` and `b` is `()`, so this becomes:

```haskell
(>>=) :: IO String -> (String -> IO ()) -> IO ()
```

With do-notation:
```haskell
readAndPrint = do
  userInput <- getLine
  putStrLn userInput
```

# Parser combinators

There are many resources on parser combinators in Haskell (parsing libraries with many different parsers that can be composed to create more complex parsers). Let’s take a look at how monads can help with that.

A bit similar to IO, where different actions are modeled as values that do something, when run, we could model our parsers as values that parse something, when run.

```haskell
newtype Parser a = Parser {runner :: String -> Either String (a, String)}
```

Here `Parser` is a record with a single field: function `runner` that takes string input and returns either an error of type `String`, or a tuple consisting of the parsed value and the remainder of the input.
Type parameter `a` of the `Parser` is the type of the value this parser produces on successful parsing.

Type `Either` is defined like this in Haskell (normally `Left` signifies error and `Right` signifies success):

```haskell
data Either a b = Left a | Right b
```
This means that you either have an error value of type `a` or a success value of type `b`.


Here is the definition of the bind function:

```haskell
parser >>= computationRest = Parser $ \input ->
   case parser.runner input of
     Left err -> Left err
     Right (parsedResult, rest) ->
       let restParser = computationRest parsedResult
        in restParser.runner rest
```

This says that we can chain a parser with the rest of the computation, returning the combined parser that will, when run:
- Run the provided parser with the initial input
- If that fails, the whole parsing fails
- If that succeeds, we pass the result of the first parser to the function `computationRest`, that returns the parser to parse the rest of the input with.
- We need to run that parser by calling its `runner` function, passing it the input unconsumed by the first parser. The value it returns will become the result of the combined parser.

Let’s see how this can be used. Suppose we have some string input with person names. For adults the input will also contain their phone numbers.

```haskell
data PersonKind = Child | Adult

data Person = Person {name :: String, kind :: PersonKind, phone :: Maybe String}

phoneNumberParser :: Parser PhoneNumber
personKindParser :: Parser PersonKind
```
We can now write a more complex parser like this:

```haskell
parsePerson :: Parser Person
parsePerson = do
  name <- stringParser
  spaceParser
  personKind <- personKindParser
  case personKind of
    Child -> return $ Person {name = name, kind = Child, phone = Nothing}
    Adult -> do
      phone <- phoneParser
      return $ Person {name = name, kind = Adult, phone = Just phone}
```

Note that all the details related to what input needs to be given to what parser, and what to do if a parsing step fails, don’t need to appear here, as they are now extracted into the `>>=` function.

The same thing can be written with `>>=` (and do-notation is indeed desugared to something similar).		

```haskell
parsePerson :: Parser Person
parsePerson =
  stringParser
    >>= ( \name ->
            ( spaceParser
                >>= ( \_ ->
                        personKindParser
                          >>= ( \kind ->
                                  case kind of
                                    Child -> return $ Person {name = name, kind = Child, phone = Nothing}
                                    Adult ->
                                      phoneParser
                                        >>= ( \phone ->
                                                return $
                                                  Person {name = name, kind = Adult, phone = Just phone}
                                            )
                              )
                    )
            )
        )
```

See how `>>=` binds the result of each step in the rest of the computation, for example how the result of the first `stringParser` becomes available in the rest of the code as `name`.

# Return function

The function `return` appearing above is the second function in the monad interface, and is often used at the end of the computation chain, allowing to construct a step that just "returns" a given value, rather than actually "doing" anything. Remember, the function passed as a second argument to `>>=` needs to return the step type. Imagine you are combining optional values, and you arrived at the result `5`. You still need to wrap it in a `Maybe`, what if you had a `Nothing`?

```haskell
return :: a -> m a
```

For optional values:

```haskell
return :: a -> Maybe a
return value = Just value
```

In words, if we have a value, we just wrap it in `Just`.

For parsers:

```haskell
return value = Parser $ \input -> Right (value, input)
```

In words, just return the provided value without consuming any input.

It is worth noting that while `return` allows you to go from a result type to the step type, there is no general way of going from the step type to the result type that works for all monads.
For example, there is no function that will take `IO a` as input and return `a` as output (except for `unsafePerformIO` that should be avoided and is very rarely used). This also means that once you have some IO code, `IO` type will appear in the signature of your function and all the functions that use it, meaning that it is very easy to distinguish between pure functions and functions that perform some IO.

# Summary

So, monads provide a way of composing a computation from smaller steps by chaining them together, useful when you need to be able to make decisions regarding the next steps based on the results of a previous step. Some other techniques, like `Applicative` might be more appropriate if you don't need this kind of branching logic.

In a pure and lazy language like Haskell, where expressions evaluate on-demand, where there are no such thing as "statements", and functions are not allowed to perform side effects, monads provide a natural way to model imperative parts of the program where one has to execute actions that have side effects in specific order.

Unlike in many other languages where monads exist as a pattern but are not necessarily made into something explicit, extracting bind and return functions into a type class in Haskell allows to create libraries that work for any monads, support things like do-notation, etc.
