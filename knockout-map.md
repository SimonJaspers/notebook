# Mapping over observable values

Knockout has `observable` and `computed` values. Both these value types are `subscribable`, meaning they can automatically notify other pieces of code whenever they change.

```js
// Initialize userName as an observable empty string
const userName = ko.observable("");

// Define a userHandle based on the user name
const userHandle = ko.pureComputed(
  () => `@${userName().toLowerCase().replace(/ /g, "-")}`
);

// Make sure that changes to the handle are logged
userHandle.subscribe(console.log);

// Change the userName to a real name
userName("Sarah Jane Doe"); // Logs "@sarah-jane-doe"
```

> In knockout, observable values are instantiated using `ko.observable(initialValue)` . To **set** an observable value, you call it with an argument. To **get** its value, you call it without arguments.  

In the example above, `userHandle` is a computed variable. It takes a `userName`, prepends an `@`, transforms it to lower case and replaces spaces by dashes.

Rather than just wrapping a static value in an observable container, a `pureComputed` defines *how its value is computed*. You can think of it as a machine that takes one or more *inputs* and a certain *calculation* to output a new value. Whenever one of the required inputs for the computation changes, it reevaluates its own outcome. 

One of the beauties of `pureComputed` values is that they are lazily evaluated. *When the computed value isn't used or needed elsewhere, it will not run its computation*. This explains the `pure` part of its name: since we can't be sure when or how often the computation runs, we cannot allow it to have side effects!

## Room for improvement

The example above is pretty useful but has some downsides:

 - It’s not immediately clear that there’s a dependency to only `userName`
 - There’s quite some noise surrounding the core logic we’re implementing:
    - `ko.pureComputed`
    - The function wrapper `() => { /* … */ }`
 - It’s hard to test `userHandle` because it accesses `userName` that has to be in its closure

Let’s define some helpers and use some of javascript’s more functional features to streamline the process of creating computed values!

## Extending knockout with `map`
The pattern of having an `initial value + transform function = new value` is used so often, that it could do with a method on every observable: `map`.

Just like [`Array.prototype.map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map) enables us to take a function of `a → b` to transform an `Array of a` to an `Array of b`, we'll be able to transform a `Subscribable of a` to a `Subscribable of b`!

```js
ko.subscribable.fn.map = function(mapper) {
  return ko.pureComputed(() => mapper(this());
};	
```
> Knockout doesn't use prototypes or `new` internally. Defining a method on the `fn` object is the way to extend the `subscribable` instance.

This extension ensures all our newly created observable and computed values have a `map` method. This method helps us fix the first two issues we described earlier:

```js
const userHandle = userName.map(
  (name) => `@${name.toLowerCase().replace(/ /g, "-")}`
);
```

We’ve managed to remove the inner call to `userName()`, and there’s no annoying syntax noise surrounding our core logic.

## Ensuring testability
This leaves us with our last issue: how do we test `userHandle`? By using easily testable utility functions!

```js
// Utils
export const createUserHandle = name => `@${name.toLowerCase().replace(/ /g, "-")}`;

// App
import { createUserHandle } from "./utils";

const userHandle = userName.map(createUserHandle);
```

We can write unit tests for `createUserHandle` to describe its behavior and limitations. We can write generic tests for our `map` method. 

## Use as you see fit
We can easily change how much of `createHandle`'s logic we want to expose in our main `App` code:

```js
// Utils
export const prepend = prefix => str => `${prefix}{str}`;
export const replace = (regex, repl) => str => str.replace(regex, repl);
export const toLower = str => str.toLowerCase();

export const pipe = (...fns) => x => fns.reduce((acc, f) => f(acc), x);

// App
import { prepend, replace, toLower, pipe } from "./utils";

// Two other ways of defining `userHandle`
const userHandleAlt1 = userName
  .map(toLower)
  .map(replace(/ /g, "-"))
  .map(prepend("@"));
  
const userHandleAlt2 = userName.map(
  pipe(
    toLower,
    replace(/ /g, "-"),
    prepend("@")
  )
);
  
// userHandle ≈≈ userHandleAlt1 ≈≈ userHandleAlt2
```

## ~~Computing~~ Concluding
Knockout has some really cool reactive data types for us to use, but the syntax leaves room for improvement. Adding a `map` method is a great first step towards writing more declerative, easier-to-test knockout code.

---

P.S. Notice from the last example that `x.map(f).map(g) ≈≈ x.map(pipe(f, g))`? This happens to be one of the laws required for a [Functor](https://github.com/fantasyland/fantasy-land#functor)! The second law, the identity law, (`x ≈≈ x.map(x => x)` also holds.
