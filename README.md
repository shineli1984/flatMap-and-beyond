# flatMap-and-beyond

This is for you, if you
- have used `map` in some other lanaguage or in javascript.
- sometime wonder how different it is to call forEach or even have a for loop instead of map.
- realize there is some high order abstraction going on behind the scene and you are looking for it.

## Array as context
To understand why there is a method call `map`, we need to first treat `array` as a context.
Take `[1, 2, 3]` as an example, imagine `[` and `]` as a box wrapping around `1, 2, 3`:
```
+---------+
|         |
| 1, 2, 3 | .map fn => fn 1, fn 2, fn 3
|         |
+---------+
```
What `[1, 2, 3].map(element => fn)` does is unpack this box and take `1, 2, 3` out then calls the function `fn` on each number.

In other words, instead of saying `array` has a method `map`, we say:
A function (`fn`) outside the context of an array can operate on the value (`1, 2, 3`) inside an `array`, so that the fact the value (`1, 2, 3`) coming from an array is irrelevant to `fn`.

## Promise as context
From the seemingly trivial example above, I'm sure if you noticed the subtle difference between calling `array` a context and calling `array` an `array`. This change of sematics is a form of abstraction. It's OK if you don't get that. To understand abstraction, more practical example needs to be examed. Let's look at `promise` expressed below:
```
+---------+
|         |
|    1    | .then fn => fn 1
|         |
+---------+
```
Now the box represents a value in the future (`promise`). It provides a context around the value `1`. When `then` is called on the context, `1` will be unpacked and given to `fn`.

In other words, instread of saying `promise` has a method `then`, we say:
A function (`fn`) outside the context of a promise can operate on the value (`1`) inside a promise, so that the fact the value (`1`) coming in from a future point of time is irrelevant to `fn`. The benefit of `then` method is clearer to us than `map` on the array since now asynchronous behavior is abstracted away by treating it as a context.

## Context and map
After the two examples above, you may have realized that there is a common ground when it comes to deal with value in context. The abstraction goes something like this:
```
Context.map(deContextedValue => fn(deContextedValue))
```

## Long live the context
So how is this abstraction useful? Let's look at an example. 

### Maybe
Suppose you have a value, it can be null or a number. And you need to calculate the squared value of it. So here goes the way we don't know about context and map:
```
const a = aFuncation(); // this function returns a number or null

if (a !== null) {
  const b = Math.pow(a, 2);
}
```
What if you can do this:
```
import { unit } from './maybe.js';
const a = aFuncation(); 
const pow2 = value => Math.pow(value, 2);

const maybeB = unit(a)
  .map(pow2);
```
`unit` puts `a` in a context called maybe, which represents null or a value. `map` called on the context of maybe unpacks the value inside, if it has a value then call the function passed to map (`pow2`) then wrap the result in a maybe again, otherwise return the maybe. This way, `a` being null or a value is irrelevent to us. Once it is in the context of maybe, the logic of checking if `a` is `null` is a concern of `map` method.

The implementation of `maybe.js` goes like this:
```
const unit = value => ({
    value: () => value,
    map: fn => (value !== null) && unit(fn(value)) || {
        value: null
    }
});

export { unit };
```

## Here goes flatMap
Continue with the example above. What if function `pow2` for some reason produces a maybe as well?
```
const pow2 = value => unit(Math.pow(value, 2));
```
Then the example will become:
```
import { unit } from './maybe.js';
const a = aFuncation(); 
const pow2 = value => unit(Math.pow(value, 2));

const maybeB = unit(a)
  .map(pow2)
  .map(maybeValue => maybeValue.value());
```
Looking at the second map, the function `maybeValue => maybeValue.value()` is there purely to unpack the maybe returned from pow2. This seems like a job for `maybe`. We can change `maybe.js` like this:
```
const unit = value => ({
    value: () => value,
    map: fn => (value !== null) && unit(fn(value)) || unit(null),
    flatMap: fn => unit(value).map(fn).value()
});
```
So that 
```
const maybeB = unit(a)
  .flatMap(pow2);
```
By examing the example above, what `flatMap` does is to unpact again. 
Using `array` as an example, it is like transforming `[[1, 2], [3, 4]]` to `[1, 2, 3, 4]`. In the context of maybe, it becomes transforming `unit(unit (a))` to `unit(a)`. 

## and beyong
If you survived so far, congrats. You are half way into the functional world.
What we just went through is a notation called monad.

A monad must satisfy three laws:

- unit(a).flatMap(f) === f(a) 
    |        |            |
    left    right         cancels each other.  This is called left identity.

- unit(a).flatMap(unit) === unit(a)
             |      |       |
             left   right   cancels each other. This is called right identity.

- unit(a).flatMap(f).flatMap(g) === unit(a).flatMap(a => f(a).flatMap(g) )
                   |          |                             |            |
                   value1     value2                        value1       value2. This is called associativity.

given f and g both return a value inside some context that unit produce.

Head spinning? Let's take a break and find out why monads have these three laws.
