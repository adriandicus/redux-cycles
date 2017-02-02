# Redux-cycles
Handle redux async actions using [Cycle.js](https://cycle.js.org/).

[![Build Status](https://travis-ci.org/cyclejs-community/redux-cycles.svg?branch=master)](https://travis-ci.org/cyclejs-community/redux-cycles)

## Install

`npm install --save redux-cycles`

Then use `createCycleMiddleware()` which takes as first argument your `main` Cycle.js function, and second argument the Cycle.js drivers you want to use:

```js
import { createCycleMiddleware } from 'redux-cycles';

function main(sources) {
  const pong$ = sources.ACTION
    .filter(action => action.type === 'PING')
    .mapTo({ type: 'PONG' });

  return {
    ACTION: pong$
  }
}

const cycleMiddleware = createCycleMiddleware(main);

const store = createStore(
  rootReducer,
  applyMiddleware(cycleMiddleware)
);
```

## Example

Try out this [JS Bin](https://jsbin.com/govola/10/edit?js,output).

See a real world example: [cycle autocomplete](https://github.com/cyclejs-community/redux-cycles/blob/master/example/cycle/index.js).

## Why?

There already are several side-effects solutions in the Redux ecosystem:

* [redux-thunk](https://github.com/gaearon/redux-thunk)
* [redux-saga](https://github.com/redux-saga/redux-saga)
* [redux-ship](https://clarus.github.io/redux-ship/)
* [redux-observable](http://redux-observable.js.org)

Why create yet another one?

The intention with redux-cycles was not to worsen the "JavaScript fatigue".
Rather it provides a solution that solves several problems attributable to the currently available libraries.

* **Respond to actions as they happen, from the side.**

  Redux-thunk forces you to put your logic directly into the action creator.
  This means that all the logic caused by a particular action is located in one place... which doesn't do the readability a favor.
  It also means cross-cutting concerns like analytics get spread out across many files and functions.

  Redux-cycles, instead, joins redux-saga and redux-observable in allowing you to respond to any action without embedding all your logic inside an action creator.

* **Declarative side-effects.**

  For several reasons: code clarity and testability.

  With redux-thunk and redux-observable you just smash everything together.

  Redux-saga does make testing easier to an extent, but side-effects are still ad-hoc.

  Redux-cycles, powered by Cycle.js, introduces an abstraction for reaching into the real world in an explicit manner.

* **Statically typable.**

  Because static typing helps you catch several types of mistakes early on.
  It also allows you to model data and relationships in your program upfront.

  Redux-saga falls short in the typing department... but it's not its fault entirely.
  The JS generator syntax is tricky to type, and even when you try to, you'll find that typing anything inside the `catch`/`finally` blocks will lead to unexpected behavior.

  Observables, on the other hand, are easier to type.

## What's this Cycle thing anyway?

[Cycle.js](https://cycle.js.org) is an interesting and unusual way of represting real-world programs.

The program is represented as a pure function, which takes in some *sources* about events in the real world (think a stream of Redux actions), does something with it, and returns *sinks*, aka streams with commands to be performed.

Redux-cycles provides an `ACTION` source, which is a stream of Redux actions, and listens to the `ACTION` sink.

Custom side-effects are handled similarly — by providing a different source and listening to a different sink.
An example with HTTP requests will be shown later in this readme.

Aside: while the Cycle.js website aims to sell you on Cycle.js for everything—including the view layer—you do *not* have to use Cycle like that.
With Redux-cycles, you are effectively using Cycle only for side-effect management, leaving the view to React, and the state to Redux.

## What does this look like?

Here's how Async is done using [redux-observable](https://github.com/redux-observable/redux-observable).
The problem is that we still have side-effects in our epics (`ajax.getJSON`).
This means that we're still writing imperative code:

```js
const fetchUserEpic = action$ =>
  action$.ofType(FETCH_USER)
    .mergeMap(action =>
      ajax.getJSON(`https://api.github.com/users/${action.payload}`)
        .map(fetchUserFulfilled)
    );
```

With Cycle.js we can push them even further outside our app using drivers, allowing us to write entirely declarative code:

```js
function main(sources) {
  const request$ = sources.ACTION
    .filter(action => action.type === FETCH_USER)
    .map(action => ({
      url: `https://api.github.com/users/${action.payload}`,
      category: 'users',
    }));

  const action$ = sources.HTTP
    .select('users')
    .flatten()
    .map(fetchUserFulfilled);

  return {
    ACTION: action$,
    HTTP: request$
  };
}
```

This middleware intercepts Redux actions and allows us to handle them using Cycle.js in a pure data-flow manner, without side effects. It was heavily inspired by [redux-observable](https://github.com/redux-observable/redux-observable), but instead of `epics` there's an `ACTION` driver observable with the same actions-in, actions-out concept. The main difference is that you can handle them inside the Cycle.js loop and therefore take advantage of the power of Cycle.js functional reactive programming paradigms.

## Drivers

Redux-cycles ships with two drivers:

* `ACTION`, which is a read-write driver, allowing to react to actions that have just happened, as well as to dispatch new actions.
* `STATE`, which is a read-only driver that streams the current redux state. It's a reactive counterpart of the `yield select(state => state)` effect in Redux-saga.

```javascript
import sampleCombine from 'xstream/extra/sampleCombine'

function main(sources) {
  const state$ = sources.STATE;
  const isOdd$ = state$.map(state => state.counter % 2 === 0);
  const increment$ = sources.ACTION
    .filter(action => action.type === INCREMENT_IF_ODD)
    .compose(sampleCombine(isOdd$))
    .map(([ action, isOdd ]) => isOdd ? increment() : null)
    .filter(action => action);

  return {
    ACTION: increment$
  };
}
```

Here's an example on [how the STATE driver works](https://jsbin.com/kijucaw/7/edit?js,output).

NOTE: If you want to use any other driver aside ACTION and STATE, make sure to have it installed and registered. You can do so at instantiation time, via the `createCycleMiddleware` API.
For example, for the HTTP driver:

```bash
npm install @cycle/http
```

```javascript
import { createCycleMiddleware } from 'redux-cycles';
import { makeHTTPDriver } from '@cycle/http';

const cycleMiddleware = createCycleMiddleware(main, { HTTP: makeHTTPDriver() });
```
## Utils

Redux-cycles ships with a `combineCycles` util. As the name suggests, it allows you to take multiple cycle apps (main functions) and combine them into a single one.

### Example

```javascript
import { combineCycles } from 'redux-cycles';

// import all your cycle apps (main functions) you intend to use with the middleware:
import fetchReposByUser from './fetchReposByUser';
import searchUsers from './searchUsers';
import clearSearchResults from './clearSearchResults';

export default combineCycles(
  fetchReposByUser,
  searchUsers,
  clearSearchResults
);

```

You can see it used in the provided [example](https://github.com/cyclejs-community/redux-cycles/blob/master/example/cycle/index.js).

## Testing

Since your main Cycle functions are pure dataflow, you can test them quite easily by giving streams as input and expecting specific streams as outputs. Checkout [these example tests](https://github.com/cyclejs-community/redux-cycles/blob/master/example/cycle/test/test.js). Also checkout the [cyclejs/time](https://github.com/cyclejs/time) project, which should work perfectly with redux-cycles.

## Why not just use Cycle.js?

Mainly because Cycle.js does not say anything about how to handle state, so Redux, which has specific rules for state management, is something that can be used along with Cycle.js. This middleware allows you to continue using your Redux/React stack, while allowing you to get your hands wet with FRP and Cycle.js.

### What's the difference between "adding Redux to Cycle.js" and "adding Cycle.js to Redux"?

This middleware doesn't mix Cycle.js with Redux/React at all (like other cycle-redux middlewares do). It behaves completely separately and it's meant to (i) intercept actions, (ii) react upon them functionally and purely, and (iii) dispatch new actions. So you can build your whole app without this middleware, then once you're ready to do async stuff, you can plug it in to handle your async stuff with Cycle.

You should think of this middleware as a different option to handle side-effects in React/Redux apps. Currently there's redux-observable and redux-saga (which uses generators). However, they're both imperative and non-reactive ways of doing async. This middleware is a way of handling your side effects in a pure and reactive way using Cycle.js.
