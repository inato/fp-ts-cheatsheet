# FP-TS 101

This aims to present how we use [fp-ts](https://gcanti.github.io/fp-ts/) at Inato.
I have mainly compiled the work of my wonderful colleagues [@LaureRC](https://github.com/LaureRC) and [@punie](https://github.com/punie)

1. [Data Structures](#data-structures)
2. [Composing Functions](#chaining)
3. [From one data structure to another](#flip-data-struct)
4. [Reader](#reader)
5. [Some specific examples](#specific-examples)

## <a name="data-structures"></a>Data Structures

Functional programming is all about data-structures that encapsulate values to give them more context.

Think about an `Array`. It is a data structure encapsulating a group of values of the same type. The `Array` gives a context of "there are multiple values". It's the same with the data structures we use in FP.

Here are the ones we use:

- [Option](#option)
  - [Build Options](#build-options)
  - [Get Value from an Option](#get-options)
- [Either](#either)
  - [Build Either](#build-either)
  - [Get Value from an Either](#get-either)
- [TaskEither](#taskeither)
  - [Build TaskEither](#build-taskeither)
  - [Get Value from a TaskEither](#get-teither)
- [Array and ReadonlyArray](#arrays)
  - [Filter and map in one go](#filter-map)
  - [Sorting with `Ord` instances](#sort)

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
const isEven = (number) => number % 2 === 0;

const noneValue = Option.fromPredicate(isEven)(3); // Option.None
const optionValue = Option.fromPredicate(isEven)(4); // Option.Some(4)
```

##### From another data structure

You can build an `Option` from an `Either`. If the Either is in its left state (~ error) you get an `Option.None`, otherwise you get an `Option.Some` of the value in the Either

```typescript
const leftEither = Either.left("whatever");
const rightEither = Either.right("value");

const noneValue = Option.fromEither(leftEither); // Option.None
const optionValue = Option.fromNullable(rightEither); // Option.Some("value")
```

#### <a name="get-options"></a>Get Value from an Option

When your value is encapsulated into an Option, you sometimes want to retrieve it and opt-out of the functional paradigm. For example if you want to give it to the outside world, in a graphql resolver.

##### Get the value or <null | undefined>

Easiest way is to use the `toNullable` or `toUndefined` helpers. They are pretty straightforward, if your Option contains a value, you get it, otherwise you get null or undefined:

```typescript
const noneValue = Option.none;
const someValue = Option.of("value");

Option.toUndefined(noneValue); // undefined
Option.toUndefined(someValue); // "value"
Option.toNullable(noneValue); // null
Option.toNullable(someValue); // "value"
```

##### Get the value with a default

You can use one of the following helper to retrieve your value or have a default if your `Option` is in `none` state.

`getOrElse` takes a function as parameter that returns the default value for `none`. Please note the default value must have the same type as your initial Option. Eg: `getOrElse` on an `Option<number>` must return a `number`. If you want to return another type, you can use `getOrElseW`.

```typescript
const noneValue = Option.none;
const someValue = Option.of("value");

Option.getOrElse(() => "default")(noneValue); // "default"
Option.getOrElse(() => "default")(someValue); // "value"

Option.getOrElseW(() => 3)(noneValue); // 3
```

##### Compute and get the value

The last way to get your value is `fold` and it allows you to compute before returning it.
it takes a two functions, the first one is executed if your Option is `none`, the second one if your `Option` contains some value.

```typescript
const noneValue = Option.none;
const someValue = Option.of(10);

const doubleOrZero = Option.fold(
  () => 0, // this is called when your Option is none
  (n: number) => 2 * n // called when your Option has some value
);

doubleOrZero(noneValue); // 0
doubleOrZero(someValue); // 20
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
const leftValue = Either.fromNullable("value was nullish")(null); // Either.left('value was nullish')
const optionValue = Either.fromNullable("value was nullish")("value"); // Either.right("value")
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

const leftValue = eitherBuilder(3); // Either.left('3 is an odd number')
const rightValue = eitherBuilder(4); // Either.right(4)
```

##### From another data structure

You can build an `Either` from an `Option`. It works exactly as the `fromNullable` as we've seen an Option could represent a nullable value.

```typescript
const noneValue = Option.none;
const someValue = Option.some("value");

const left = Either.fromOption("value was nullish")(noneValue); // Either.left('value was nullish')
const right = Either.fromOption("value was nullish")(someValue); // Either.right('value')
```

#### <a name="get-either"></a>Get Value from an Either

This part looks a lot like the `Option` part. There are two `Either` "destructors", which are really similar to the ones we've seen above.

##### Get value or default

The `getOrElse` destructor and its `getOrElseW` version work exactly as the one from `Option`. You have to pass it what to return if you are in the left branch.

```typescript
const leftValue = Either.left("Division by Zero!");
const rightValue = Either.right(10);

Either.getOrElse(() => 0)(leftValue); // 0
Either.getOrElse(() => 0)(rightValue); // 10
```

##### Compute left and right branches

The `fold` destructor also takes two functions, the first one representing what to do on left branch, the second one what to do on right branch.

```typescript
const leftValue = Either.left("Division by Zero!");
const rightValue = Either.right(10);

const doubleOrZero = Either.fold(
  (eitherError: string) => {
    console.log(`The error was ${eitherError}`);
    return 0;
  }, // on left branch
  (value: number) => 2 * value // on right branch
);

doubleOrZero(leftValue); // logs "The error was Division by Zero!" and returns 0
doubleOrZero(rightValue); // 20
```

### <a name="taskeither"></a>TaskEither

A `Task` represents an asynchronous computation. You can view it as a `Promise`.
So quite simply a `TaskEither` is an asychronous computation that can have two results (~ that can fail).

A TaskEither is typed `TaskEither<A, B>` where `A` is the type of the left track and `B` the right track.

**Note:** The type `TaskEither<E, A>` is **strictly** equivalent to `Task<Either<E, A>>`, and so there are no conversion functions between the two. All the functions from the `Task` module can thus be used on a `TaskEither` with the caveat that combinators such as `map` or `chain` give you access to the whole inner `Either`.

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
    throw new Error("ODD");
  }
  return a;
};
const buildTaskEither = (number: number) =>
  TaskEither.tryCatch(
    () => asyncIsEven(number),
    () => `${number} is odd`
  );

const rightValue = buildTaskEither(4); // TaskEither.right(4)
const leftValue = buildTaskEither(3); // TaskEither.left('3 is odd')
```

#### <a name="get-teither"></a>Get Value from a TaskEither

In fp-ts, if you invoke a `TaskEither`, you get a `Promise<Either>` so most of the time, what we do is we invoke our TaskEither and then use the aforementioned methods on the `Either` to retrieve its value.

```typescript
const ten = TaskEither.right(10); // this is a right branch of a taskEither
const rightValue = await ten(); // we invoke ten, making it a Promise<Either> and then await the promise, so rightValut is an Either.right

const returnedValue = Either.getOrElse(() => 0)(rightValue); // 10
```

### <a name="arrays"></a>Array and ReadonlyArray

The `Array` and `ReadonlyArray` types from `fp-ts` extend their counterparts from native TypeScript so they can be used without having to do any conversion.

Those modules provide equivalent functions as `Array.prototype.map` or `Array.prototype.filter` but with a more `fp-ts`, `pipe`-friendly API:

```typescript
const array = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];

pipe(array, Array.filter(isEven), Array.map(square));
// => [0, 4, 16, 36, 64]
```

#### <a name="filter-map"></a>Filter and map in one go

Sometimes, you want to map an array with a function that may fail to produce a value and then get rid of those failures. That is precisely what `Array.filterMap` does:

```typescript
// getById(id: Id): Option<Item>;
const ids = [id1, id2, id3];
const foundItems = pipe(ids, Array.filterMap(getById));
// => Array<Item>
```

#### <a name="sort"></a>Sorting with `Ord` instances

Sorting strings:

```typescript
import * as Ord from "fp-ts/lib/Ord";

const strings = ["zyx", "abc", "klm"];

const sortedStrings = pipe(strings, Array.sort(Ord.ordString));
// => ['abc', 'klm', 'zyx']
```

There are various instances of `Ord` for primitive types, available in `fp-ts`:

- Strings: `ordString: Ord<string>`
- Numbers: `ordNumber: Ord<number>`
- Booleans: `ordBoolean: Ord<boolean>`
- Dates: `ordDate: Ord<Date>`

The `Ord` module also provides us with ways to manipulate and combine them to create richer sorting algorithms!

For example, to get an instance of `Ord` that will sort strings in reverse order, I can simply create and use it as such:

```typescript
const strings = ["zyx", "abc", "klm"];

const reversedOrdString = Ord.getDualOrd(Ord.ordString);

const sortedStrings = pipe(strings, Array.sort(reversedOrdString));
// => ['zyx', 'klm', 'abc']
```

Some higher level types like `Option<A>` also implement `Ord`! In this particular case, `none` is considered lower than `some` for example. So if you want to sort an array of `Option<number>`:

```typescript
const nums = [Option.some(1337), Option.none, Option.some(42)];

const ordOptionalNumbers = Option.getOrd(Ord.ordNumber);

const sortedNums = pipe(nums, Array.sort(ordOptionalNumbers));
// => [Option.none, Option.some(42), Option.some(1337)]
```

Likewise, if you have a more complex construct, like a `User`:

```typescript
interface User {
  name: string;
  age: Option<number>;
}
```

You can build various instance of `Ord<User>` based on your need with `Ord.contramap`. This function is just a way of defining how to access the field that will be used for sorting.

```typescript
const byName = pipe(
  Ord.ordString,
  Ord.contramap((user: User) => user.name)
);

const byAge = pipe(
  Option.getOrd(Ord.ordNumber),
  Ord.contramap((user: User) => user.age)
);
```

You can then tell `Array.sort` to use either one of these instances depending on your needs.

Finally, if you need to combine those sorting conditions (first, sort by age, and then by name), you can use `Array.sortBy` which accepts an array of `Ord` instances, to be used in order:

```typescript
const sortedUsers = pipe(users, Array.sortBy([byAge, byName]));
```

## <a name="chaining"></a>Composing Functions

In a functional paradigm, you want to have small functions, and to call them one after another to transform data easily.

- [Basics](#chaining-basics)
  - [Pipe](#pipe)
  - [Flow](#flow)
- [Working with data structures: Combinators](#combinators)
  - [Mapping](#map)
  - [Chaining](#chain)

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
const finalValue = pipe(value, addSthg, doSthgElse, doFinalSthg);
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
const transformData = flow(addSthg, doSthgElse, doFinalSthg); // please note it's exactly as the original pipe, without the first value

const value = "value";
const finalValue = transformData(value);
```

Here `transformData` could be reused somewhere else, be simple and easy to pipe with other stuff.

### <a name="combinators"></a>Working with data structures: Combinators

Let's say you have a datastructure containing a value you wanna work with. You can have an `Option.some(3)` or an `Either.right(currentUser)` or even `["some", "strings"]`.
Your value is encapsulated in a context, for example you computed your `currentUser` and maybe the computation could fail. To keep working with the value without removing its context (what we've seen above), we'll use combinators.

#### <a name="map"></a> Mapping

- [Mapping Options](#mapoption)
- [Mapping Either](#mapeither)
- [Mapping TaskEither](#maptaskeither)

So you have a context containing your value, `Either<number>` but you want to transform it with a function that takes an input of your value type. Eg: `double: (number) => (number)`. Unfortunately, you cannot do `double(myEitherNumber)`.

You will use `map`. Map will unwrap the value, apply the function and then wrap the value again in its context. That's it.
`map` takes a function as parameter (the function transforming the value) and can then be applied to the value.

```typescript
const doubleIfEven = (n: number) => n % 2 === 0 ? 2 * n : n

const optionEven = Option.some(2);
const optionOdd = Option.some(3);

Option.map(doubleIfEven)(optionEven) // Option.some(4)

pipe(
  optionOdd
  Option.map(doubleIfEven)
) // Option.some(3)
```

According to your datastructure, maps behave differently. We all know the JS `Array.prototype.map` which actually unwraps values from an array, transform those values with a function, and then repacks the values in an Array.

```typescript
[1, 2, 3].map(doubleIfEven); // [1, 4, 3]
FPArray.map(doubleIfEven)([1, 2, 3]); // same result but using fp-ts
```

`map` allows you to go from `DataStructure<A>` to `DataStructure<B>` as you can apply any function going from `A`.

```typescript
pipe(
  facility, // Option<Facility>
  Option.map((facility: Facility) => facility.country.code) // => returns an Option<CountryCode>
);
```

##### <a name="mapoption"></a> Mapping Options

`Map` behavior on Options is pretty simple. If there is `some` value, it will apply the function to the value. If the Option is `none`, it will just return `none`.

```typescript
const someOption = Option.some(2);
const noneOption = Option.none;

Option.map(doubleIfEven)(someOption); // Option.some(4)
Option.map(doubleIfEven)(noneOption); // Option.none
```

##### <a name="mapeither"></a> Mapping Either

Remember `Either` is generally considered as "Everything went well" (right branch) vs "Something happened" (left branch).
So `map` will only transform your data if you are in the right branch.

If you want to map on the left branch, you can use `mapLeft`!

```typescript
const rightValue = Either.right(2);
const leftValue = Either.left(2);

Either.map(doubleIfEven)(rightValue); // Either.right(4)
Either.map(doubleIfEven)(leftValue); // Either.left(2)

Either.mapLeft(doubleIfEven)(leftValue); // Either.left(4)
```

Eventually, there is also a bimap helper mapping the first function to the left branch, and the second function to the right branch

```typescript
const evenErrorMessage = (n: number) => {
  if (n % 2 === 0) return `${n} is even but in Error State`;

  return `${n} is odd and in Error State`;
};

const doubleOrError = Either.bimap(evenErrorMessage, doubleIfEven);

const rightValue = Either.right(2);
const leftValue = Either.left(3);

doubleOrError(rightValue); // Either.right(4)
doubleOrError(leftValue); // Either.left("3 is odd and in Error State")
```

##### <a name="maptaskeither"></a> Mapping TaskEither

It's really exactly the same as mapping `Either` same functions available, same api.

#### <a name="chain"></a> Chaining

- [Chaining Options](#chainoption)
- [Chaining Either](#chaineither)
- [Chaining TaskEither](#chaintaskeither)

So you have a context containing your value, `Option<number>`. But you want to apply to your value another function that can maybe return null, Eg: `maybeDouble: (number) => Option<number>`. If you apply `map` as seen above, you'll end up with an `Option<Option<number>>`, which is not easy to handle.

You could use `flatten` to transform an `Option<Option>` (or an `Array<Array>` etc) but we will more likely use `chain`

Chain will unwrap the value, apply the function and then combine the initial context with the new context. That's it. What's implied is in order to combine, you can only chain functions that returns "roughly" the same context as the original one. IE, you `Either.chain` on functions returning Either.

`chain` takes a function as parameter (the function transforming the value) and can then be applied to the value.

```typescript
const doubleIfEvenElseNone = (n: number) => n % 2 === 0
  ? Option.some(2 * n)
  : Option.none

const optionEven = Option.some(2);
const optionOdd = Option.some(3);

Option.chain(doubleIfEvenElseNone)(optionEven) // Option.some(4)

pipe(
  optionOdd
  Option.chain(doubleIfEvenElseNone)
) // Option.none
```

Please note as mentioned it is the same as

```typescript
pipe(
  optionEven,
  Option.map(doubleIfEvenElseNone), // Option.some(Option.some(4))
  Option.flatten // Option.some(4)
);

Option.chain === flow(Option.map, Option.flatten);
```

According to your datastructure, chains behave differently.

##### <a name="chainoption"></a> Chaining Options

`Chain` behavior on Options is pretty simple. It applies the function to your value if the original is `some` else it return `none`. And that's about it!

```typescript
const someOption = Option.some(2);
const someOddOption = Option.some(3);
const noneOption = Option.none;

Option.chain(doubleIfEvenElseNone)(someOption); // Option.some(4)
Option.chain(doubleIfEvenElseNone)(someOddOption); // Option.none
Option.chain(doubleIfEvenElseNone)(noneOption); // Option.none
```

##### <a name="chaineither"></a> Chaining Either

Remeber `Either` is generally considered as "Everything went well" (right branch) vs "Something happened" (left branch).

Let's say you have an either `Either<E, number>`, where `E` is the left type, and `number` the right brancch type
So `chain` will only transform your data if you are in the right branch. Otherwise, it will return your Either as it was.
To "flatten" your Either, there are two cases:

1. you chain a function with the same left type (returning `Either<E, *>`). You can use the `chain` method and it will return an `Either<E, *>` (what is returned by your chained function)
2. you chain a function with a different left type (returning `Either<D, *>`). To flatten your Either, you will have to combine both error types. You have to use the `chainW` method, where `W` means widen, as you extend the error type. It will return an `Either<E | D, *>`

```typescript
const initialError = (): InitialError => "This is an initial Error";
const notEvenError = (num: number): NotEvenError => `${num} is not Even`;

// Here we instanciate three Either<InitialError, number>
const rightEvenValue = Either.right(2);
const rightOddValue = Either.right(3);
const leftValue = Either.left(InitialError);

const doubleIfEven = (n: number): Either<NotEvenError, number> =>
  n % 2 === 0 ? Either.right(2 * n) : Either.left(notEvenError(n));

Either.chainW(doubleIfEven)(rightEvenValue); // Either.right(4)
Either.chainW(doubleIfEven)(rightOddValue); // Either.left(NotEvenError)
Either.chainW(doubleIfEven)(leftValue); // Either.left(InitialError)
```

We could not use `chain` in the above example as the returned Either type of the function was different from the input one.

If you want to use your data but return it unaltered to the rest of your pipeline (like some kind of side effect), but still fail if something wrong happened, you can use the `chainFirst` (or `chainFirstW`) combinator!

```typescript
const doubleAndTellIfEven = (n: number): Either<NotEvenError, string> =>
  n % 2 === 0
    ? Either.right(`${2 * n} is an even number!`)
    : Either.left(notEvenError(n));

const rightEvenValue = Either.right(2);
const rightOddValue = Either.right(3);

pipe(
  rightEvenValue,
  Either.chainFirstW(doubleAndTellIfEven), // this goes right branch, but we drop the string returned and pass the initial value
  Either.getOrElse(() => 0) // here we get the original 2
);

pipe(
  rightOddValue,
  Either.chainFirstW(doubleAndTellIfEven), // this goes left branch due to failed validation
  Either.getOrElse(() => 0) // here we get default 0
);
```

##### <a name="chaintaskeither"></a> Chaining TaskEither

As mentioned, `TaskEither` works a lot like Either, so you can apply the same stuff as described above.

One interesting behaviour, is you have a `TaskEither<value>` and you want to apply to your value a function returning an Either. You would have a `TaskEither<Either<value>>`.
instead of then doing `fromEither` and `flatten`, you can use the `chainEitherK` (or `chainEitherKW` if error types are not equivalent) combinator, that does the same!

```typescript

const rightEvenValue = TaskEither.right(2);
const rightOddValue = TaskEither.right(3);

// note this function returns an Either!
const doubleIfEven = (n: number): Either<NotEvenError, number> =>
  n % 2 === 0 ? Either.right(2 * n) : Either.left(notEvenError(n));

pipe(
  rightEvenValue
  TaskEither.chainEitherKW(doubleIfEven) // this returns TaskEither(4)
)

pipe(
  rightOddValue
  TaskEither.chainEitherKW(doubleIfEven) // this returns TaskEither.left(NotEvenError)
)

```

## <a name="flip-data-struct"></a>From one data structure to another

- [Flipping data structures](#flip)
- [Applying functions returning another data type](#apply-return)

### <a name="flip"></a>Flipping data structures

One thing we really do often is "flip" two contexts we have. For example you make multiple queries and you end up with an `Array<Either<E, A>>` but you'd rather have an `Either<E, Array<A>>`.

Or you map a function that may fail on an `Option` and end up having an `Option<Either>` but you want it the other way around.

We can do it by using the `sequence` function of our data structures. When combining, please note an `Array<Either>` where some Either are right and other left, will turn into an `Either.left`

```typescript
import * as FPArray from "fp-ts/lib/Array";
import * as Either from "fp-ts/lib/Either";
import * as Option from "fp-ts/lib/Option";
import * as TaskEither from "fp-ts/lib/TaskEither";

const arrayOfEither = [
  Either.right(42),
  Either.left(SomeError),
  Either.right(1337),
];
const eitherOfArray = FPArray.array.sequence(Either.either)(arrayOfEither);
// eitherOfArray: Either<SomeError, Array<number>> == Either.left(SomeError)

const arrayOrRight = [Either.right(42), Either.right(1337)];

const eitherOfArray = FPArray.array.sequence(Either.either)(arrayOrRight);
// eitherOfArray: Either<SomeError, Array<number>> == Either.right([42, 1337])

const optionOfTaskEither = Option.some(TaskEither.right(42));
const taskEitherOfOption = Option.option.sequence(TaskEither.taskEither)(
  optionOfTaskEither
);
// taskEitherOfOption: TaskEither<SomeError, Option<number>> ==
//   TaskEither.right(Option.some(42))
```

### <a name="apply-return"></a>Applying functions returning another data type

Another useful use case (overlapping a bit the above one), is when you apply a function returning another datatype.
You can simply use `map` of course, like you have an `Option` and wanna fetch something if the option is some, and you'll end up with an `Option<TaskEither>`

```typescript
const getUserPreferences = (userId: UserID) => TaskEither<UserNotFound, UserPreferences>
const optionUserId = Option.some(userId)

pipe(
  optionUser,
  Option.map(getUserPreferences) // this will give you an Option<TaskEither.right(userPreferences)>
)
```

but maybe, you want it the other way around directly, because maybe you want to chain it with other TaskEither, or whatever reasons.
You can use the `traverse` function for that, allows you to "traverse" your context with a function.

traverse takes a first argument which is the datastructure that will be returned by the transformation function. And then two arguments, the first one being the data that will be transformed, and then the transformation function.

```typescript
const getUserPreferences = (userId: UserID) => TaskEither<UserNotFound, UserPreferences>
const optionUserId = Option.some(userId)

const result = Option.option.traverse(TaskEither.taskEither)(optionUserId, getUserPreferences)
// 👆 this returns a TaskEither<UserNotFound, Option<UserPreferences>>


pipe(
  (optionUser, getUserPreferences),
  Option.option.traverse(TaskEither.taskEither) // this is the same as above.
)
```

## <a name="reader"></a>Reader

If you already had to carry dependencies throught out multiple functions, you will quickly understand the value of `Reader`.
You can see the definition given by Giulio Canti [here](https://dev.to/gcanti/getting-started-with-fp-ts-reader-1ie5), and basically `Reader<R,A>` represents a function `(r: R) => A` where `R` will be your dependencies, and A the result.

Let's take an example:

```typescript
const foo = ({
  firstId,
  secondId,
  eventData,
  firstDependency,
  secondDependency,
}): TaskEither<Error, Value> =>
  pipe(
    firstDependency.getById(firstId),
    TaskEither.chain(() => secondDependency.sendEvent({ secondId, eventData }))
  );
```

Using `Reader`, you could write it this way:

```typescript
interface Deps1 {
  firstDependency: D1;
}
interface Deps2 {
  secondDependency: D2;
}
interface Deps extends Deps1, Deps2 {}

const foo = ({
  firstId,
  secondId,
  eventData,
}): ReaderTaskEither<Deps, Error, Value> =>
  pipe(
    getById(firstId),
    ReaderTaskEither.chain(() => sendEvent({ secondId, eventData }))
  );

// with:
const getById = (firstId) =>
  pipe(
    Reader.ask<Deps1>(),
    Reader.map(({ firstDependency }) => firstDependency.getById(firstId))
  );
const sendEvent = ({ secondId, eventData }) =>
  pipe(
    Reader.ask<Deps2>(),
    Reader.map(({ secondDependency }) =>
      secondDependency.sendEvent({ secondId, eventData })
    )
  );
```

You need to adapt some methods to be able to properly use the `Reader` pattern, but once it's done, you won't need to carry your dependencies from a method to another as those will only be seen where they are needed.

## <a name="specific-examples"></a> Some specific examples

### <a name="call-in-parallel"></a> Call in parallel

At some point, you might need to call methods in parallel, for instance because you need to send data to multiple external services and you don't want to decrease the performance of your use case. Here is an example that shows you how to call two `ReaderTaskEither` in parallel :

```typescript
import * as RTE from "fp-ts/ReaderTaskEither";
import { pipe } from "fp-ts/function";

declare const foo: RTE.ReaderTaskEither<R1, E1, A1>;
declare const bar: RTE.ReaderTaskEither<R2, E2, A2>;

pipe(
  RTE.Do,
  RTE.apS("foo", foo),
  RTE.apSW("bar", bar)
  // this returns RTE.ReaderTaskEither<R1 & R2, E1 | E2, {readonly foo: A1; readonly bar: A2}>
);
```
