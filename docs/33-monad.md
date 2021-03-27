# Unit 33: Monad

## Learning Objectives

After this lecture, students should:

- Understand what are functors and monads
- Understand the laws that a functor and monad must obey and be able to verify them

## Generalizing `Logger<T>`

We have just created a class `Logger<T>` with a `flatMap` method that allows us to operate on the value encapsulated inside, along with some "side information".  `Logger<T>` follows a pattern that we have seen many times before.  We have seen this in `Maybe<T>` and `Lazy<T>`, and `InfiniteList<T>`.  Each of these classes has:

- an `of` method to initialize the value and side information.
- have a `flatMap` method to update the value and side information.

Different classes above have different side information that is initialized, stored, and updated when we use the `of` and `flatMap` operations.

- `Maybe<T>` stores the side information of whether the value is there or not there.
- `Lazy<T>` stores the side information of whether the value has been evaluated or not.
- `InfiniteList<T>` stores the side information that the values in the list may or may not be evaluated.
- `Logger<T>` stores the side information of a log describing the operations done on the value.

These classes that we wrote follow certain patterns that make them well behave when we create them with `of` and chain them with `flatMap`.  Such classes that are "well behave" are examples of a programming construct called _monads_.  A monad must follow three laws, to behave well.  Let's examine the laws below.

## Identity Laws

Before we list down the first and second laws formally, let's try to get some intuition over the desired behavior first.

The `of` method in a monad should behave like an identity.  It creates a new monad by initializing our monad with a value and its side information.   For instance, in our `Logger<T>`,
```Java
  public static Logger of(T value) {
  return new Logger(value, "");
  }
```
The logger is initialized with empty side information (e.g., empty string as a log message).

Now, let's consider the lambda that we wish to pass into `flatMap`  -- such a lambda takes in a value, compute it, and wrap it in a "new" monad, together with the correponding side information.  For instance,

```Java
Logger incrWithLog(int x) {
  return new Logger(incr(x), "incr " + x + "; ");
}
```

What should we expect when we take a fresh new monad `Logger.of(4)` and call `flatMap` with a function `incrWithLog`?  Since `Logger.of(4)` is new with no operation performed on it yet, calling 
```
Logger.of(4).flatMap(x -> incrWithLog(x)) 
```
should just result in the same value exactly as calling `incrWithLog(4)`.  So, we expect that, after calling the above, we have a `Logger` with a value 5 and a log message of `"incr 4"`.

Our `of` method should not do anything extra to the value and side information -- it should simply wrap the value 4 into the `Logger`.  Our `flatMap` method should not do anything extra to the value and the side information, it should simply apply the given lambda expression to the value.

Now, suppose we take an instance of `Logger`, called `logger`, that has already been operated on one or more times with `flatMap`, and contain some side information.  What should we expect when we call:
```
logger.flatMap(x -> Logger.of(x))
```

Since `of` should behave like an identity, it should not change the value or add extra side information.  The `flatMap` above should do nothing and the expression above should be the same as `logger`.

What we have just described above is called the _left identity law_ and the _right identity law_ of monads.  To be more general, let `Monad` be a type that is a monad and `monad` be an instance of it.

The left identity law says:
- `Monad.of(x).flatMap(x -> f(x))` must be the same as `f(x)`

The right identity law says:
- `monad.flatMap(x -> Monad.of(x))` must be the same as `monad`

## Associative Law

Let's now go back to the original `incr` and `abs` functions for a moment.  To compose the functions, we can write `abs(incr(x))`, explicitly one function after another.  Or we can compose them as into another function: 
```
int absIncr(int x) {
  return abs(incr(x));
}
```

and call `absIncr(x)`.  The effects should be exactly the same.  It does not matter if we group the functions together into another function before applying it to a value x.

Recall that after we build our `Logger` class, we were able to compose the functions `incr` and `abs` by chaining the `flatMap`:

```
Logger.of(4)
      .flatMap(x -> incrWithLog(x))
      .flatMap(x -> absWithlog(x))
```

We should get the resulting value as `abs(incr(4))`, along with the appropriate log messages.

Another way to call `incr` and then `abs` is to write something like this:
```
Logger<Integer> absIncrWithLog(int x) {
  return incrWithLog(x).flatMap(x -> absWithLog(x));
}
```

We have compose the methods `incrWithLog` and `absWithLog` and group them under another method.  Now, if we call:
```
Logger.of(4)
      .flatMap(x -> absIncrWithLog(x))
```

The two expressions must have exactly the same effect on the value and its log message.

This example leads us to the third law of monad: regardless of how we group that calls to `flatMap`, their behave must be the same.  This law is called the _associative law_.  More formally, it says:

- `monad.flatMap(x -> f(x)).flatMap(x -> g(x))` must be the same as `monad.flatMap(x -> f(x).flatMap(y -> g(y)))`

## A Counter Example

If our monads follow the laws above, we can safely write methods that receive a monad from others, operate on it, and return it to others.  We can also safely create a monad and pass it to the clients to operate on.  Our clients can then call our methods in any order and operates on the monads that we create, and the effect on its value and side information is as expected.

Let's try to make our `Logger` misbehave a litte.  Suppose we change our `Logger<T>` to be as follows:

```Java hl_lines="12 16"
// version 0.3 (NOT a monad)
class Logger<T> {
  private final T value;
  private final String log;

  private Logger(T value, String log) {
    this.value = value;
  this.log = log;
  }

  public static Logger of(T value) {
  return new Logger(value, "Logging starts: ");
  }

  public <R> Logger<R> flatMap(Transformer<? super T, ? extends Logger<? extends R>> transformer) {
    Logger<R> logger = transformer.transform(this.value)
  return new Logger(logger.value, logger.log + "\n" + this.log);
  }

  String toString() {
    return "value: " + this.value + ", log: " + this.log;
  }
}
```

Our `of` adds a little initialization message.  Our `flatMap` adds a little new line before appending with the given log message.  Now, our `Logger<T>` is not that well behave anymore.

Suppose we have two methods `foo` and `bar`, both take in an `x` and perform a series of operations on `x`.  Both returns us a `Logger` instance on the final value and its log.

```
Logger<Integer> foo(int x) {
  return Logger.of(x)
      .flatMap(...)
    .flatMap(...)
       :
  ;
}
Logger<Integer> bar(int x) {
  return Logger.of(x)
      .flatMap(...)
    .flatMap(...)
       :
  ;
}
```

Now, we want to perform the sequence of operations done in `foo`, followed by the sequence of operations done in `bar`.  So we called:
```
foo(4).flatMap(x -> bar(x))
```

We will find that the string `"Logging starts"` appears twice in our logs and there is now an extra blank line in the log file!

## Functor

We will end this unit with a brief discussion on _functor_, another common abstraction in functional-style programming.  A functor is a simpler construction than a monad in that it only ensures lambdas can be applied sequentially to the value, without worrying about side information.

Recall that when we build our `Logger<T>` abstraction, we add a `map` that only updates the value but change nothing to the side information.  One can think of a functor as an abstraction that supports `map`.  

A functor needs to adhere to two laws:

- preserving identity: `functor.map(x -> x)` is the same as `functor`
- preserving composition: `functor.map(x -> f(x)).map(x -> g(x))` is the same as `functor.map(x -> g(f(x))`. 

Our classes from `cs2030s.fp`, `Lazy<T>`, `Maybe<T>`, and `InfiniteList<T>` are functors as well.

## Monads and Functors in Other Languages

Such abstractions are common in other languages.  In Scala, for instance, the collections (list, set, map, etc.) are monads.  In pure functional languages like Haskell, monads are one of the fundamental building blocks.