# FP-TS Cheat Sheet

1. [Data Structures](#data-structures)
2. [Composing Functions](#chaining)

## <a name="data-structures"></a>Data Structures

Functional programming is all about data-structures that encapsulate values to give them more context.

Think about an `Array`. It is a data structure encapsulating a group of values of the same type. The `Array` gives a context of "there are multiple values". It's the same with the data structures we use in FP. 

Here are the ones we use:
  - [Option](#option)
    - [Build Options](#build-options)
  - [Either](#either)
    - [Build Either](#build-either)
  - [TaskEither](#taskeither)
    - [Build TaskEither](#build-taskeither)
 
### <a name="option"></a>Option
 
An Option represents a value which might be there, or not. If it's there then it is `Option.Some(value)` and if not it is `Option.none`
You can think of an option as something that can be null or undefined.

#### <a name="build-options"></a>Building an Option

##### From a value
The easiest way to build an `Option` is to use the some or none constructor that returns a value encapsulated in an `Option`.
```typescript
  const noneValue = Option.none;
  const someValue = Option.some("value");
```

If you have a value and want to check it you can use `fromNullable`. If the value is `null | undefined` you get a `Option.None` otherwise you get the value wrapped in an `Option` data structure.
```typescript
  const noneValue = Option.fromNullable(null); // Option.None
  const optionValue = Option.fromNullable("value"); // Option.Some("value")
```

You can also pass you own validation function to build an `Option` with the fromPredicate helper: 
```typescript
  const isEven = number => number % 2 === 0
  
  const noneValue = Option.fromPredicate(isEven)(3) // Option.None
  const optionValue = Option.fromPredicate(isEven)(4) // Option.Some(4)
```

##### From another data structure
You can build an `Option` from an `Either`. If the Either is in its left state (~ error) you get an `Option.None`, otherwise you get an `Option.Some` of the value in the Either
```typescript
  const leftEither = Either.left("whatever");
  const rightEither = Either.right("value");
  
  const noneValue = Option.fromEither(leftEither) // Option.None
  const optionValue = Option.fromNullable(rightEither) // Option.Some("value")
```

### <a name="either"></a>Either
 
An Either represents a computation that can have two results, called branches or tracks (left and right).

Most of the time, we use it to represent a **computation that can fail**.
So the left branch represents the failure, and the right branch the success. So when we manipulate an Either, if we enter the left branch, we most of the time won't carry out further manipulations, and just return the Error context that is encapsulated in the left branch (it is still accessible tho, through different helpers). If we are in the right branch, we'll keep on manipulating the value and passing it through the right branch.

An Either is typed `Either<E, A>` where `E` is the type of the left track (E for `Error` most of the time) and `A` the right track.


#### <a name="build-either"></a>Building an Either

##### From a value

The easiest way to build an `Either` is to use the right or left constructor that returns a value encapsulated as a right or left `Either`.
```typescript
  const leftValue = Either.left("value"); // -> you are on the left branch
  const rightValue = Either.right("value"); // -> you are on the right branch
```

You have also a `fromNullable` helper. If the value is `null` or `undefined` you need to provide your Either what to put in the left track, otherwise it will put your value in the right track.
```typescript
  const leftValue = Either.fromNullable('value was nullish')(null); // Either.left('value was nullish')
  const optionValue = Either.fromNullable('value was nullish')("value"); // Either.right("value")
```


You can also pass you own validation function to build an `Either` with the fromPredicate helper. Here it is a bit more complicated.
You have to first pass a function checking your value is correct (should I go right or left track). This function can be a simple function returning a boolean, or a type guard. And then pass a value for the left track.
```typescript
 type EvenNumber = number;
 const isEven = (num: number) => num % 2 === 0;
 const isEvenTypeGuard = (num: number): num is EvenNumber => num % 2 === 0;
 const eitherBuilder = Either.fromPredicate(
   isEven, // here we could use isEvenTypeGuard to infer the number type and have an Either<E, EvenNumber>
   (number) => `${number} is an odd number`
 );
 // Here when you use eitherBuilder you get something with the type Either<string, number>
  
 const leftValue = eitherBuilder(3) // Either.left('3 is an odd number')
 const rightValue = eitherBuilder(4) // Either.right(4)
```

##### From another data structure
You can build an `Either` from an `Option`. It works exactly as the `fromNullable` as we've seen an Option could represent a nullable value.
```typescript
  const noneValue = Option.none;
  const someValue = Option.some("value");
  
  const left = Either.fromOption('value was nullish')(noneValue) // Either.left('value was nullish')
  const right = Either.fromOption('value was nullish')(someValue) // Either.right('value')
```

### <a name="taskeither"></a>TaskEither

A `Task` represents an asynchronous computation. You can view it as a `Promise`.
So quite simply a `TaskEither` is an asychronous computation that can have two results (~ that can fail)

A TaskEither is typed `TaskEither<A, B>` where `A` is the type of the left track and `B` the right track.


#### <a name="build-taskeither"></a>Building a TaskEither

##### From a value

You can build a TaskEither as you would build an Either: `left`, `right`, `fromNullable`, `fromPredicate`.
```typescript
  const leftValue = TaskEither.left("value");
  const rightValue = TaskEither.right("value");

  ...
```

You can also build a TaskEither from a Promise! You have the `tryCatch` helper, that takes two functions as arguments. The first one returns a promise, and the second one returns the value to put in left if the promise rejects.

```typescript
  const asyncIsEven = async (a: number) => {
    await asyncExternalCall();

    if (a % 2 !== 0) {
      throw new Error('ODD');
    }
    return a;
  }
  const buildTaskEither = (number: number) => TaskEither.tryCatch(
    () => asyncIsEven(number),
    () => `${number} is odd`
  );
  
  const rightValue = buildTaskEither(4) // TaskEither.right(4)
  const leftValue = buildTaskEither(3) // TaskEither.left('3 is odd')
```


## <a name="chaining"></a>Composing Functions

In a functional paradigm, you want to have small functions, and to call them one after another to transform data easily.

- [Basics](#chaining-basics)
  - [Pipe](#pipe)
  - [Flow](#flow)

### <a name="chaining-basics"></a>Basics

#### <a name="pipe"></a>Pipe

The `pipe` function creates a "pipeline" to transform data. It takes one value as a starting point, and several functions, and then passes the value into the first function, then the return value of the first function into the second function etc...

In classic imperative style we would do:
```typescript
const value = "value";
const value1 = addSthg(value);
const value2 = doSthgElse(value1);
const finalValue = doFinalSthg(value2);
```

It's the same as: 
```typescript
const value = "value";
const finalValue = pipe(
  value,
  addSthg,
  doSthgElse,
  doFinalSthg
);
```
Remark: as you pass **one** value through your pipe, that's why `fp-ts` functions tend to accept only one value, it's easier to pipe them and so on, as we can see here:
```typescript
const myOption = pipe(
  "value",
  Either.fromNullable('Given value was null),
  Option.fromEither
); // myOption = Option.Some("value")
```

#### <a name="flow"></a>Flow

The `flow` function works exactly as the `pipe`, without the first value. It creates the pipeline of data but does not transform anything, it returns a function that can be used somewhere else (in a pipe possibly!)

We could rewrite the above example as: 
```typescript
const transformData = flow(
  addSthg,
  doSthgElse,
  doFinalSthg
); // please note it's exactly as the original pipe, without the first value

const value = "value";
const finalValue = transformData(value)
```
Here `transformData` could be reused somewhere else, be simple and easy to pipe with other stuff.
