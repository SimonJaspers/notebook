The previous post explained how a `map` method can help us use a *unary* function to convert one value in to another. In this post, we'll explore how we can compute values using functions that require multiple parameters.

I'll start with an example. Let's say we're coding a calendar app. When the user selects an event, we want to show a little indicator if the event is in progress. We'll have to calculate whether the page's event is ongoing.

A plain javascript example:

```js
// A binary function of (event, time) -> bool
const eventAndTimeOverlap = ({ start, end }, time) => 
  time >= start && time <= end;

// The event currently rendered in the UI
// Note: for brevity, we're using "hours past midnight" as a unit for
// our start and end time.
const selectedEvent = { 
  start: 8, 
  end: 10,
  title: "Sales meeting" 
};

// A time stamp
const currentTime = 9;

// The boolean value we can bind to the visibility of our "in-progress" icon:
const eventIsOngoing = eventAndTimeOverlap(selectedEvent, currentTime); // true
```

## If you have a hammer...
Let's try rewriting the example to reactive knockout code with just our favorite tool: `map`:

```js
const selectedEvent = ko.observable(
  { start: 8, end: 10, title: "Sales meeting" }
);
const currentTime = ko.observable(9);

const selectionAndTime = ko.observable(
  [ selectedEvent, currentTime ]
);

const eventIsOngoing = selectionAndTime.map(
  ([ event, time ]) => eventAndTimeOverlap(event(), time())
)
```

Let's break this up:

 - `selectionAndTime` collects the two arguments we require in to an observable array so we can `map` over them.
 - `eventIsOngoing` maps over the argument array with an inline function that does two things:
   - it *unwraps* the values from their observables
   - it *applies* `eventAndTimeOverlap` to the values

We could refactor to something like:

```js
// Utils
const map = xs => f => xs.map(f);
const apply = f => (...args) => f(...args);
const unwrapAll = map(ko.unwrap);

// App
// ...

const eventIsOngoing = ko.observable([ selectedEvent, currentTime ])
  .map(unwrapAll)
  .map(apply(eventAndTimeOverlap));

```

We've cleaned things up a bit, but it's not that much of an improvement... Knowing that we'll be using this pattern a lot in our apps, it might be worth it to create a helper.

## Argument Collection
Lacking the knowledge or inspiration to be good at naming abstract things... enter `ArgumentCollection`:

```js

const ArgumentCollection = (...args) => ({
  unwrap: f => ko.pureComputed(
    () => f(...args.map(ko.unwrap))
  )
});

// We can now write:
const AC = ArgumentCollection;

const eventIsOngoing = AC(selectedEvent, currentTime)
  .unwrap(eventAndTimeOverlap);

```

> [`ko.unwrap`](https://github.com/knockout/knockout/blob/c6e608ff360c57dcc7dffc5a0095ad7eab0fd3fb/src/utils.js#L433) is a knockout utility that gets the inner value of an observable (`obs => obs()`), or just returns its argument if its passed a non-observable.

