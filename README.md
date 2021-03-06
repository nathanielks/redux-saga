# How does it work

An alternative Side Effect model for Redux applications. Instead of dispatching thunks which get handled by the redux-thunk
middleware. You create *Sagas* to gather all your Side Effects logic in a central place.

This means the logic of the application lives in 2 places

- Reducers are responsible of handling state transitions between actions

- Sagas are responsible of orchestrating complex/asynchronous operations (side effects or actions). This includes simple side effects
which react to one action (e.g. send a request on each button click), but also complex operations that span accross multiples
actions (e.g. User onBoarding, Wizard dialogs, aynchronous Game rules ...).


A Saga is a generator function that get user actions as inputs and may yield Side Effects (e.g. server updates, navigation ...)
as well as ther actions.

# Usage

## Getting started

Install
```
npm install redux-saga
```

Create the Saga (using the counter example from Redux)
```javascript
// sagas/index.js
function* incrementAsync(io) {

  while(true) {

    // wait for each INCREMENT_ASYNC action  
    const nextAction = yield io.take(INCREMENT_ASYNC)

    // call delay : Number -> Promise
    yield delay(1000)

    // dispatch INCREMENT_COUNTER
    yield io.put( increment() )
  }

}

export default [incrementAsync]
```

Plug redux-saga in the middleware pipeline
```javascript
// store/configureStore.js
import sagaMiddleware from 'redux-saga'
import sagas from '../sagas'

const createStoreWithSaga = applyMiddleware(
  // ...,
  sagaMiddleware(...sagas)
)(createStore)

export default function configureStore(initialState) {
  return createStoreWithSaga(reducer, initialState)
}
```

In the above example we created an `incrementAsync` Saga to handle all `INCREMENT_ASYNC` actions. The Generator function uses
`yield io.take` to wait asynchronously for the next action. The middleware handles action queries from the
Saga by pausing the Generator until an action matching the query happens. Then it resumes the Generator with the matching action.

After receiving the wanted action, the Saga triggers a call to `delay(1000)`, which in our example returns a Promise that will
be resolved after 1 second. Again, the middleware will pause the Generator until the yielded promise is resolved.

After the 1s second delay, the Saga dispatches an `INCREMENT_COUNTER` action using the `io.put(action)` method. And as for the 2 precedent statements, the Saga is resumed after resolving the result of the action dispatch (which will happen immediately if the redux dispatch function returns a normal value, but maybe later if the dispatch result is a Promise).

The Generator uses an infinite loop `while(true)` which means it will stay alive for all the application lifetime. But you can also create Sagas that last only for a limited amount of time. For example, the following Saga will wait for the first 3 `INCREMENT_COUNTER` actions, triggers a `showCongratulation()` action and then finishes.

```javascript
function* onBoarding(io) {

  for(let i = 0; i < 3; i++)
    yield io.take(INCREMENT_COUNTER)

  yield io.put( showCongratulation() )
}
```

## Declarative Effects

Sagas Generators can yield Effects in multiple forms. The simplest and most idiomatic is to yield a Promise result
from an asynchronous call ( e.g. `result = yield fetch(url)` ). However, this approach makes testing difficult, because we have
to mock the services that return the Promises (like `fetch` above). For this reason, the library provides some declarative ways
to yield Side Effects while still making it easy to test the Saga logic.

For example, suppose we have this Saga from the shopping cart example

```javascript
function* getAllProducts(io) {
  while(true) {
    // wait for the next GET_ALL_PRODUCTS action
    yield io.take(GET_ALL_PRODUCTS)

    // fetches the products from the server
    const products = yield fetch('/products')

    // dispqtch a RECEIVE_PRODUCTS action with the received results
    yield io.put( receiveProducts(products) )
  }
}
```

In order to test the above Generator, we have to mock the `fetch` call, and drive the Generator by successively calling its `next` method. An alternative way, is to use the `io.call(fn, ...args)` function. This function doesn't actually execute the call itself.
Instead it creates an object that describes the desired call. The middleware then handles automatically  the yielded result and run
the corresponding call.

```javascript
function* getAllProducts(io) {
  while(true) {
    ...
    const products = yield io.call(fetch, '/products')
    ...  
  }
}
```

This allows us to easily test the Generator outside the Redux middleware environement.

```javascript
import test from 'tape'
import io from 'redux-saga/io' // exposed for testing purposes
/* import fetch, getAllProducts, ... */

test('getProducts Saga test', function (t) {

  const iterator = getProductsSaga(io)

  let next = iterator.next()
  t.deepEqual(next.value, io.take(GET_ALL_PRODUCTS),
    "must wait for next GET_ALL_PRODUCTS action"
  )

  next = iterator.next(getAllProducts())
  t.deepEqual(next.value, io.call(fetch, '/products'),
    "must yield api.getProducts"
  )

  next = iterator.next(products)
  t.deepEqual(next.value, io.put( receiveProducts(products) ),
    "must yield actions.receiveProducts(products)"
  )

  t.end()

})
```

All we had to do is to test the successive results returned from the Generator iteration process. When using the declarative forms
Sagas generator functions are effectively nothing more than functions which get a list of successive inputs (via `io.take(pattern)`) and yield a list of successive outputs (via `io.call(fn, ...args)`, `io.put(action)` and other forms we'll see in a moement).

The `io.call` method is well suited for functions which return Promise results. Another function `io.cps` can be used to handle
Node style functions (e.g. `fn(...args, callback)` where `callback` is of the form `(error, result) => ()`). For example

```javascript
const content = yield io.cps(readFile, '/path/to/file')
```

## Error handling

You can catch errors inside the Generator using the simple try/catch syntax. Un the following example, the Saga catch errors
from the `api.buyProducts` call (i.e. a rejected Promise)

```javascript
function* checkout(io, getState) {

  while( yield io.take(types.CHECKOUT_REQUEST) ) {
    try {
      const cart = getState().cart
      yield io.call(api.buyProducts, cart)
      yield io.put(actions.checkoutSuccess(cart))
    } catch(error) {
      yield io.put(actions.checkoutFailure(error))
    }
  }
}
```

## Effect Combinators

The `yield` statements are great for representing asynchronous control flow in a simple and linear style. But we also need to
do things in parallel. We can't simply write

```javascript
// Wrong, effects will be executed in sequence
const users  = yield io.call(fetch, '/users'),
      repose = yield io.call(fetch, '/repose')
```

Becaues the 2nd Effect will not get executed until the first call resolves. Instead we have to write

```javascript
// correct, effects will get executed in parallel
const [users, repose]  = yield [
  io.call(fetch, '/users'),
  io.call(fetch, '/repose')
]
```

When we yield an array of effects, the Generator is paused until all the effects are resolved (or rejected as we'll see later).

Sometimes we also need a behavior similar to `Promise.race`, i.e. gets the first resolved effect from multiples ones. The method `io.race` offers a declarative way of triggering a race between effects.

The following shows a Saga that triggers a `showCongratulation` message if the user triggers 3 `INCREMENT_COUNTER` actions with less
than 5 seconds between 2 actions.

```javascript
function* onBoarding(io) {
  let nbIncrements = 0
  while(nbIncrements < 3) {
    // wait for INCREMENT_COUNTER with a timeout of 5 seconds
    const winner = yield io.race({
      increment : io.take(INCREMENT_COUNTER),
      timeout   : io.call(delay, 5000)
    })

    if(winner.increment)
      nbIncrements++
    else
      nbIncrements = 0
  }

  yield io.put(showCongratulation())
}
```

Note the Saga has an internal state, but that has nothing to do with the state inside the Store, this is a local state to the
Saga function used to controle the flow of actions.

## Generator delegation via yield*

TBD

## Yielding Generators inside Generators

TBD

# Building from sources

```
git clone https://github.com/yelouafi/redux-saga.git
cd redux-saga
npm install
npm test
```

There are 2 examples ported from the Redux repos

Counter example
```
// build the example
npm run build-counter

// test sample for the generator
npm run test-counter
```

Shopping Cart example
```
// build the example
npm run build-shop

// test sample for the generator
npm run test-shop
```
