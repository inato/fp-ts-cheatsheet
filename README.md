# :memo: `fp-ts` cheatsheet

1. [Imports](#imports)
2. [Data Structures](#data-structures)
3. [Composing Functions](#chaining)
4. [From one data structure to another](#flip-data-struct)
5. [Reader](#reader)
6. [Some specific examples](#specific-examples)

## <a name="imports"></a>Imports

```ts
import { either, option } from 'fp-ts'; // import modules from root
import { flow, pipe } from 'fp-ts/functions'; // import functions from functions
import { Either } from 'fp-ts/Either'; // import types from their module
import { Option } from 'fp-ts/Option';

// rename common long module names
import { readerTaskEither as rte } from 'fp-ts';
```

## <a name="pipe-flow"></a>Pipe and Flow

```ts
const value = 'value';

// Imperative style
const value1 = addSthg(value);
const value2 = doSthgElse(value1);
const finalValue = doFinalSthg(value2);
// Or maybe inline
const finalValue = doFinalSthg(doSthgElse(addSthg(value)));

// With pipe
const finalValue = pipe(value, addSthg, doSthgElse, doFinalSthg);

// With flow
const transformations = flow(addSthg, doSthgElse, doFinalSthg);
const finalValue = transformations(value);
```

## <a name="option"></a>Option

### <a name="option-builders"></a>Create an `Option<T>`

```ts
// Build an Option
const noneValue = option.none;
const someValue = option.some('value');

// Build from a value
const noneValue = option.fromNullable(null); => // option.none
const optionValue = option.fromNullable('value'); => // option.some('value')

// Build from a predicate
const isEven = (number) => number % 2 === 0;

const noneValue = option.fromPredicate(isEven)(3); // option.none
const optionValue = option.fromPredicate(isEven)(4); // option.some(4)

// Convert from another type (eg. Either)
const leftEither = either.left('whatever');
const rightEither = either.right('value');

const noneValue = option.fromEither(leftEither); // option.none
const optionValue = option.fromEither(rightEither); // option.some('value')
```

#### <a name="option-getters"></a>Extract inner value

```ts
const noneValue = option.none;
const someValue = option.of('value');

// Convert Option<T> to T | undefined
option.toUndefined(noneValue); // undefined
option.toUndefined(someValue); // 'value'

// Convert Option<T> to T | null
option.toNullable(noneValue); // null
option.toNullable(someValue); // 'value'

// Convert Option<T> with a default value
option.getOrElse(() => 'default')(noneValue); // 'default'
option.getOrElse(() => 'default')(someValue); // 'value'

// Convert Option<T> to T | U with a default value of type U
option.getOrElseW(() => 3)(noneValue); // 3: string | number

// Apply a different function on None/Some(...)
const noneValue = option.none;
const someValue = option.of(10);

const doubleOrZero = option.match(
  () => 0, // none case
  (n: number) => 2 * n // some case
);

doubleOrZero(noneValue); // 0
doubleOrZero(someValue); // 20

// Pro-tip: option.match is short for the following:
const doubleOfZeroBis = flow(
  option.map((n: number) => 2 * n), // some case
  option.getOrElse(() => 0) // none case
);
```

## <a name="either"></a>Either

### <a name="either-builders"></a>Create an `Either<E, A>`

```ts
// Build an Either
const leftValue = either.left('value');
const rightValue = either.right('value');

// Build from a value
const leftValue = either.fromNullable('value was nullish')(null); // either.left('value was nullish')
const rightValue = either.fromNullable('value was nullish')('value'); // either.right('value')

// Build from a predicate
const isEven = (num: number) => num % 2 === 0;

const eitherBuilder = either.fromPredicate(
  isEven,
  number => `${number} is an odd number`
);

const leftValue = eitherBuilder(3); // either.left('3 is an odd number')
const rightValue = eitherBuilder(4); // either.right(4)

// Convert from another type (eg. Option)
const noneValue = option.none;
const someValue = option.some('value');

const left = either.fromOption('value was nullish')(noneValue); // either.left('value was nullish')
const right = either.fromOption('value was nullish')(someValue); // either.right('value')
```

### <a name="either-getters"></a>Extract inner value

```typescript
const leftValue = either.left("Division by Zero!");
const rightValue = either.right(10);

// Convert Either<E, A> to A with a default value
either.getOrElse(() => 0)(leftValue); // 0
either.getOrElse(() => 0)(rightValue); // 10

// Apply a different function on Left(...)/Right(...)
const doubleOrZero = either.match(
  (error: string) => {
    console.log(`The error was ${eitherError}`);
    return 0;
  },
  (n: number) => 2 * n,
);

doubleOrZero(leftValue); // logs "The error was Division by Zero!" and returns 0
doubleOrZero(rightValue); // 20

// Pro-tip: either.match is short for the following:
const doubleOrZeroBis = flow(
  either.map((n: number) => 2 * n),
  either.getOrElse((error: string) => {
    console.log(`The error was ${error}`);
    return 0;
  });
);
```

## <a name="taskeither"></a>TaskEither

### <a name="taskeither-builders"></a>Create a `TaskEither<E, A>`

```ts
// Build an Either
const leftValue = taskEither.left('value');
const rightValue = taskEither.right('value');

// Build from a value
const leftValue = taskEither.fromNullable('value was nullish')(null); // taskEither.left('value was nullish')
const rightValue = taskEither.fromNullable('value was nullish')('value'); // taskEither.right('value')

// Build from a predicate
const isEven = (num: number) => num % 2 === 0;

const taskEitherBuilder = taskEither.fromPredicate(
  isEven,
  number => `${number} is an odd number`
);

const leftValue = taskEitherBuilder(3); // taskEither.left('3 is an odd number')
const rightValue = taskEitherBuilder(4); // taskEither.right(4)

// Convert from Option
const noneValue = option.none;
const someValue = option.some('value');

const left = taskEither.fromOption('value was nullish')(noneValue); // taskEither.left('value was nullish')
const right = taskEither.fromOption('value was nullish')(someValue); // taskEither.right('value')

// Convert from Either
const eitherValue = either.right(42);
const taskEitherValue = taskEither.fromEither(eitherValue);

// Build from an async function
const asyncIsEven = async (n: number) => {
  await sleep(1000);

  if (!isEven(n)) {
    throw new Error(`${n} is odd`);
  }

  return n;
};

const buildTaskEither = (n: number) =>
  taskEither.tryCatch(
    () => asyncIsEven(n),
    (error: Error) => error.message
  );

const rightValue = buildTaskEither(4); // taskEither.right(4)
const leftValue = buildTaskEither(3); // taskEither.left('3 is odd')
```

#### <a name="taskeither-getters"></a>Extract inner value

```ts
// Simple case: an infaillible Task
const someTask = task.of(42);
const innerValue = await someTask(); // invoking a Task returns the underlying Promise

// TaskEither
const someTaskEither = taskEither.right(42);
const eitherValue = await someTaskEither(); // innerValue: Either<never, number>
const innerValue = either.getOrElse(() => 0)(eitherValue); // 42
```

## <a name="arrays"></a>Array and ReadonlyArray

### Basic manipulation

```ts
const someArray = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];

// filter and map
const result = pipe(someArray, array.filter(isEven), array.map(square)); // => [0, 4, 16, 36, 64]

// in one go with filterMap
const result = pipe(
  someArray,
  array.filterMap(flow(option.fromPredicate(isEven), option.map(square)))
);
```

### <a name="ord"></a>Sort elements with Ord

```ts
const strings = ['zyx', 'abc', 'klm'];

// basic sort
const sortedStrings = pipe(strings, array.sort(string.Ord)); // => ['abc', 'klm', 'zyx']

// reverse sort
const reverseSortedStrings = pipe(strings, array.sort(ord.reverse(string.Ord)));

// sort Option
const optionalNumbers = [option.some(1337), option.none, option.some(42)];

const sortedNums = pipe(nums, array.sort(option.getOrd(number.Ord))); // => [option.none, option.some(42), option.some(1337)]

// sort complex objects with different rules
type User = {
  name: string;
  age: Option<number>;
};

const byName = pipe(
  string.Ord,
  ord.contramap((user: User) => user.name)
);

const byAge = pipe(
  option.getOrd(number.Ord),
  ord.contramap((user: User) => user.age)
);

const sortUsers = array.sortBy([byAge, byName]); // will sort an array of users by age first, then by name
```
