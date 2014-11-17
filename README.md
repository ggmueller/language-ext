C# Functional Language Extensions
=================================

Using and abusing the features of C# 6 to provide lots of helper functions and types, which, if you squint, can look like extensions to the language itself.

## Introduction

One of the great new features of C# 6 is that it allows us to treat static classes like namespaces.  This means that we can use static methods without qualifying them first.  This instantly gives us access to single term method names which look exactly like functions in functional languages.  I created this library to bring some of the functional world into C#.  It won't always sit well with the OO programmer, but I guess you can pick'n'choose what to work with.  There's still plenty here that will help day-to-day.

To use this library, simply include LanguageExt.Core.dll in your project.  And then stick this at the top of each cs file that needs it:
```C#
using LanguageExt;
using LanguageExt.Prelude;
```

`LanguageExt` contains the types, and `LanguageExt.Prelude` contains the helper functions.  There is also `LanguageExt.List`, more on that later.

What C# issues are we trying to fix?  Well, we can only paper over the cracks, but here's a glossary:

* Poor tuple support
* Null reference problem
* Lack of lambda and expression inference 
* Void isn't a real type
* List generators and processing
* The awful 'out' parameter

## Poor tuple support
I've been crying out for proper tuple support for ages.  It looks like we're no closer with C# 6.  The standard way of creating them is ugly `Tuple.Create(foo,bar)` compared to functional languages where the syntax is often `(foo,bar)` and to consume them you must work with the standard properties of `Item1`...`ItemN`.  No more...

```C#
    var ab = tuple("a","b");
```

I chose the lower-case `tuple` to avoid conflicts between other types and existing code.  I think also tuples should be considered fundamental like `int`, and therefore deserves a lower-case name.  I do this with a number of other functions, I realise this might be painful for you stalwart OO guys, but I think this is a better approach.  Happy to discuss it however :)

So consuming the tuple is now handled using `With`, which projects the `Item1`...`ItemN` onto a lambda function (or action):

```C#
    var name = tuple("Paul","Louth");
    var res = name.With( (first,last) => "Hello \{first} \{last}" );
```
Or, you can use a more functional approach:
```C#
    var name = tuple("Paul","Louth");
    var res = with( name, (first,last) => "Hello \{first} \{last}" );
```
This allows the tuple properties to have names, and it also allows for fluent handling of functions that return tuples.

## Null reference problem
`null` must be the biggest mistake in the whole of computer language history.  I realise the original designers of C# had to make pragmatic decisions, it's a shame this one slipped through though.  So, what to do about the 'null problem'?

`null` is often used to indicate 'no value'.  i.e. the method called can't produce a value of the type it said it was going to produce, and therefore it gives you 'no value'.  The thing is that when the 'no value' instruction is passed to the consuming code, it gets assigned to a variable of type T, the same type that the function said it was going to return, except this variable now has a timebomb in it.  You must continually check if the value is null, if it's passed around it must be checked too.  

As we all know it's only a matter of time before a null reference bug crops up because the variable wasn't checked.

Functional languages use what's known as an 'option type'.  In F# it's called `Option` in Haskell it's called `Maybe`...

## Optional
It works in a very similar way to `Nullable<T>` except it works with all types rather than just value types.  It's a `struct` and therefore can't be `null`.  An instance can be created by either calling `Some(value)`, which represents a positive 'I have a value' response;  Or `None`, which is the equivalent of returning `null`.

So why is it any better than returning `T` and using `null`?  It seems we can have a non-value response again right?  Yes, that's true, however you're forced to acknowledge that fact, and write code to handle both possible outcomes because you can't get to the underlying value without acknowledging the possibility of the two states that the value could be in.  This bulletproofs your code.  You're also explicitly telling any other programmers that use your method: "This method might not return a value, make sure you deal with that".  This explicit declaration is very powerful.

This is how you create an `Option<int>`:

```C#
var optional = Some(123);
```

To access the value you must check that it's valid first:

```C#
    optional.Match( 
        Some: v  => Assert.IsTrue(v == 123),
        None: () => Assert.Fail("Shouldn't get here")
        );
```
An alternative (functional) way of matching is this:

```C#
    match( optional, 
        Some: v  => Assert.IsTrue(v == 123),
        None: () => Assert.Fail("Shouldn't get here") 
        );
```

To smooth out the process of returning Option<T> types from methods there are some implicit conversion operators:

```C#
    // Automatically converts the integer to a Some of int
    Option<int> ImplicitSomeConversion() => 1000;

    // Automatically converts to a None of int
    Option<int> ImplicitNoneConversion() => None;
    
    // Will handle either a None or a Some returned
    Option<int> GetValue(bool select) =>
        select
            ? Some(1000)
            : None;
```

It's actually nearly impossible to get a `null` out of a function, even if the `T` in `Option<T>` is a reference type and you write `Some(null)`.  Firstly it won't compile, but you might think you can do this:

```C#
        private Option<string> GetStringNone()
        {
            string nullStr = null;
            return Some(nullStr);
        }
```

That will compile.  However the `Option<T>` will notice what you're up to and give you a `None` anyway.  This is pretty important to remember.  Even if you do this, you'll get a `None`.  

```C#
        private Option<string> GetStringNone()
        {
            string nullStr = null;
            return nullStr;
        }
```

So `null` mostly goes away if you use `Option<T>`.

For those of you who know what a monad is, then the `Option<T>` type implements `Select` and `SelectMany` and is therefore a monad and can be use in LINQ expressions:

```C#
    var two = Some(2);
    var four = Some(4);
    var six = Some(6);

    // This exprssion succeeds because all items to the right of 'in' are Some of int.
    (from x in two
     from y in four
     from z in six
     select x + y + z)
    .Match(
        Some: v => Assert.IsTrue(v == 12),
        None: () => Assert.Fail("Shouldn't get here")
    );

    // This expression bails out once it gets to the None, and therefore doesn't calculate x+y+z
    (from x in two
     from y in four
     from _ in Option<int>.None
     from z in six
     select x + y + z)
    .Match(
        Some: v => Assert.Fail("Shouldn't get here"),
        None: () => Assert.IsTrue(true)
    );
```

## Lack of lambda and expression inference 

One really annoying thing about the `var` type inference in C# is that it can't handle inline lambdas.  For example this won't compile, even though it's obvious it's a `Func<int>`.
```C#
    var fn = () => 123;
```
There are some good reasons for this, so best not to bitch too much.  Instead use the `fun` function from this library:
```C#
    var fn = fun( () => 123 );
```
This will work for `Func<..>` and `Action<..>` types of up to seven generic arguments.  To do the same for `Expression<..>`, use the `expr` function:

```C#
    var e = expr( () => 123 );
```

## Void isn't a real type

Functional languages have a concept of a type that has one possible value, itself, called Unit.  As an example `bool` has two values: `true` and `false`.  `Unit` has one value, usually represents in functional languages as `()`.  You can imagine that methods that take no arguments actually take one argument of `()`.  Anyway, we can't use the `()` representation in C#, so `LanguageExt` now provides `unit`.

```C#

    public Unit Empty()
    {
        return unit;
    }
```

`Unit` is the type and `unit` is the value.  It is used throughout the `LanguageExt` library instead of `void`.  The primary reason is that if you want to program functionally then all functions should return a value and `void` isn't a first-class value.  This can help a lot with LINQ expressions for example.

## List generators and processing

Support for `cons`, which is the functional way of constructing lists:
```C#
    var test = cons(1, cons(2, cons(3, cons(4, cons(5, empty<int>())))));

    var array = test.ToArray();

    Assert.IsTrue(array[0] == 1);
    Assert.IsTrue(array[1] == 2);
    Assert.IsTrue(array[2] == 3);
    Assert.IsTrue(array[3] == 4);
    Assert.IsTrue(array[4] == 5);
```

Functional languages usually have additional list constructor syntax which makes the `cons` approach easier.  It usually looks something like this:

```F#
    let list = [1;2;3;4;5]
```

In C# it looks like this (or worse):

```C#
    var list = new int[] { 1, 2, 3, 4, 5 };
```
So now there's an additional `list(...)` function which takes any number of parameters and turns them into a list:

```C#
    // Creates a list of five items
     var test = list(1, 2, 3, 4, 5);
```

This is much closer to the 'functional way'.

(from this point on all functions require `using LanguageExt.List`)

Also `range`:

```C#
    // Creates a list of 1001 integers lazily.
    var list = range(1000,2000);
```

Some of the standard list functions are available.  These are obviously duplicates of what's in LINQ, therefore they've been put into their own namespace:

```C#
    // Generates 10,20,30,40,50
    var input = list(1, 2, 3, 4, 5);
    var output1 = map(input, x => x * 10);

    // Generates 30,40,50
    var output2 = filter(output1, x => x > 20);

    // Generates 120
    var output3 = fold(output2, 0, (x, s) => s + x);

    Assert.IsTrue(output3 == 120);
```

The above can be written in a fluent style also:

```C#
    var res = list(1, 2, 3, 4, 5)
                .map(x => x * 10)
                .filter(x => x > 20)
                .fold(0, (x, s) => s + x);

    Assert.IsTrue(res == 120);
```

Other list functions:
`head`
`headSafe` - returns `Option<T>`
`tail`
`foldr`
`sum`

## The awful 'out' parameter
This has to be one of the most awful patterns in C#:

```C#
    int result;
    if( Int32.TryParse(value, out result) )
    {
        ...
    }
    else
    {
        ...
    }
```
There's all kinds of wrong there.  Essentially the function needs to return two pieces of information:
* Whether the parse was a success or not
* The successfully parsed value
This is a common theme throughout the .NET framework.  For example on `IDictionary.TryGetValue`

```C#
    int value;
    if( dict.TryGetValue("thing", out value) )
    {
       ...
    }
    else
    {
       ...
    }
```       

So to solve it we now have methods that instead of returning `bool`, return `Option<T>`.  If the operation fails it returns `None`.  If it succeeds it returns `Some(the value)` which can then be matched.  Here's some usage examples:

```C#
    int value1 = parseInt("123").Failure(() => 0);

    int value2 = failure(parseInt("123"), () => 0);

    int value3 = parseInt("123").Failure(0);

    int value4 = failure(parseInt("123"), 0);

    parseInt("123").Match(
        Some: v  => UseTheInteger(v),
        None: () => failwith("Not an integer")
        );

    match( parseInt("123"),
        Some: v => UseTheInteger(v),
        None: () => failwith("Not an integer")
        );
```
       
### Future
There's more to come with this library.  Feel free to get in touch with any suggestions.
