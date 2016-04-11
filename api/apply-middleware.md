# `applyMiddleware(...middlewares)`

Los middleware son la forma sugerida de extender Redux con funcionalidades personalizadas. Los middlewares te dajan envolver el método [`dispatch`](./Store.md#dispatch) del Store. La característica principal de los middlewares es que son combinables. Multiples middlewares se pueden combinarjunto, donde ninguno necesita saber cual vino antes o viene después.

El uso más común de los middlewares es soportar acciones asíncronas sin demasiado código o dependiendo de librerías como [Rx](https://github.com/Reactive-Extensions/RxJS). Logre eso gracias a que te permite despachar [acciones asíncronas](../glosario.md#acción-asíncrona) además de las normales.

Por ejemplo, [redux-thunk](https://github.com/gaearon/redux-thunk) permite a los creadores de acciones invertír el control despachando funciones. Van a recibir [`dispatch`](./Store.md#dispatch) como argumento y capaz llamarlo asíncronamente. Estas funciones son llamadas *thunks*. Otro ejemplo de middleware es [redux-promise](https://github.com/acdlite/redux-promise). Este te deja despachar una [Promesa](https://developer.mozilla.org/es/docs/Web/JavaScript/Referencia/Objetos_globales/Promesa) como una acción asíncrona, y despachar una acción normal cuando la Promesa se resuelve.

Los middlewares no vienen incluidos en [`createStore`](createStore.md) y no son una parte fundamental de la arquitectua de Redux, pero los consideramos suficientemente útiles para soportarlos directamente. De esta forma, hay una única forma estandarizada de extender [`dispatch`](./Store.md#dispatch) y diferentes middlewares probablemente compitan en expresividad y utilidad.

#### Argumentos

* `...middlewares` (*arguments*): Functions that conform to the Redux *middleware API*. Each middleware receives [`Store`](Store.md)’s [`dispatch`](Store.md#dispatch) and [`getState`](Store.md#getState) functions as named arguments, and returns a function. That function will be given the `next` middleware’s dispatch method, and is expected to return a function of `action` calling `next(action)` with a potentially different argument, or at a different time, or maybe not calling it at all. The last middleware in the chain will receive the real store’s [`dispatch`](Store.md#dispatch) method as the `next` parameter, thus ending the chain. So, the middleware signature is `({ getState, dispatch }) => next => action`.

#### Returns

(*Function*) A store enhancer that applies the given middleware. The store enhancer signature is `createStore => createStore'` but the easiest way to apply it is to pass it to [`createStore()`](./createStore.md) as the last `enhancer` argument.

#### Example: Custom Logger Middleware

```js
import { createStore, applyMiddleware } from 'redux'
import todos from './reducers'

function logger({ getState }) {
  return (next) => (action) => {
    console.log('will dispatch', action)

    // Call the next dispatch method in the middleware chain.
    let returnValue = next(action)

    console.log('state after dispatch', getState())

    // This will likely be the action itself, unless
    // a middleware further in chain changed it.
    return returnValue
  }
}

let store = createStore(
  todos,
  [ 'Use Redux' ],
  applyMiddleware(logger)
)

store.dispatch({
  type: 'ADD_TODO',
  text: 'Understand the middleware'
})
// (These lines will be logged by the middleware:)
// will dispatch: { type: 'ADD_TODO', text: 'Understand the middleware' }
// state after dispatch: [ 'Use Redux', 'Understand the middleware' ]
```

#### Example: Using Thunk Middleware for Async Actions

```js
import { createStore, combineReducers, applyMiddleware } from 'redux'
import thunk from 'redux-thunk'
import * as reducers from './reducers'

let reducer = combineReducers(reducers)
// applyMiddleware supercharges createStore with middleware:
let store = createStore(reducer, applyMiddleware(thunk))

function fetchSecretSauce() {
  return fetch('https://www.google.com/search?q=secret+sauce')
}

// These are the normal action creators you have seen so far.
// The actions they return can be dispatched without any middleware.
// However, they only express “facts” and not the “async flow”.

function makeASandwich(forPerson, secretSauce) {
  return {
    type: 'MAKE_SANDWICH',
    forPerson,
    secretSauce
  }
}

function apologize(fromPerson, toPerson, error) {
  return {
    type: 'APOLOGIZE',
    fromPerson,
    toPerson,
    error
  }
}

function withdrawMoney(amount) {
  return {
    type: 'WITHDRAW',
    amount
  }
}

// Even without middleware, you can dispatch an action:
store.dispatch(withdrawMoney(100))

// But what do you do when you need to start an asynchronous action,
// such as an API call, or a router transition?

// Meet thunks.
// A thunk is a function that returns a function.
// This is a thunk.

function makeASandwichWithSecretSauce(forPerson) {

  // Invert control!
  // Return a function that accepts `dispatch` so we can dispatch later.
  // Thunk middleware knows how to turn thunk async actions into actions.

  return function (dispatch) {
    return fetchSecretSauce().then(
      sauce => dispatch(makeASandwich(forPerson, sauce)),
      error => dispatch(apologize('The Sandwich Shop', forPerson, error))
    )
  }
}

// Thunk middleware lets me dispatch thunk async actions
// as if they were actions!

store.dispatch(
  makeASandwichWithSecretSauce('Me')
)

// It even takes care to return the thunk’s return value
// from the dispatch, so I can chain Promises as long as I return them.

store.dispatch(
  makeASandwichWithSecretSauce('My wife')
).then(() => {
  console.log('Done!')
})

// In fact I can write action creators that dispatch
// actions and async actions from other action creators,
// and I can build my control flow with Promises.

function makeSandwichesForEverybody() {
  return function (dispatch, getState) {
    if (!getState().sandwiches.isShopOpen) {

      // You don’t have to return Promises, but it’s a handy convention
      // so the caller can always call .then() on async dispatch result.

      return Promise.resolve()
    }

    // We can dispatch both plain object actions and other thunks,
    // which lets us compose the asynchronous actions in a single flow.

    return dispatch(
      makeASandwichWithSecretSauce('My Grandma')
    ).then(() =>
      Promise.all([
        dispatch(makeASandwichWithSecretSauce('Me')),
        dispatch(makeASandwichWithSecretSauce('My wife'))
      ])
    ).then(() =>
      dispatch(makeASandwichWithSecretSauce('Our kids'))
    ).then(() =>
      dispatch(getState().myMoney > 42 ?
        withdrawMoney(42) :
        apologize('Me', 'The Sandwich Shop')
      )
    )
  }
}

// This is very useful for server side rendering, because I can wait
// until data is available, then synchronously render the app.

import { renderToString } from 'react-dom/server'

store.dispatch(
  makeSandwichesForEverybody()
).then(() =>
  response.send(renderToString(<MyApp store={store} />))
)

// I can also dispatch a thunk async action from a component
// any time its props change to load the missing data.

import { connect } from 'react-redux'
import { Component } from 'react'

class SandwichShop extends Component {
  componentDidMount() {
    this.props.dispatch(
      makeASandwichWithSecretSauce(this.props.forPerson)
    )
  }

  componentWillReceiveProps(nextProps) {
    if (nextProps.forPerson !== this.props.forPerson) {
      this.props.dispatch(
        makeASandwichWithSecretSauce(nextProps.forPerson)
      )
    }
  }

  render() {
    return <p>{this.props.sandwiches.join('mustard')}</p>
  }
}

export default connect(
  state => ({
    sandwiches: state.sandwiches
  })
)(SandwichShop)
```

#### Tips

* Middleware only wraps the store’s [`dispatch`](Store.md#dispatch) function. Technically, anything a middleware can do, you can do manually by wrapping every `dispatch` call, but it’s easier to manage this in a single place and define action transformations on the scale of the whole project.

* If you use other store enhancers in addition to `applyMiddleware`, make sure to put `applyMiddleware` before them in the composition chain because the middleware is potentially asynchronous. For example, it should go before [redux-devtools](https://github.com/gaearon/redux-devtools) because otherwise the DevTools won’t see the raw actions emitted by the Promise middleware and such.

* If you want to conditionally apply a middleware, make sure to only import it when it’s needed:

  ```js
  let middleware = [ a, b ]
  if (process.env.NODE_ENV !== 'production') {
    let c = require('some-debug-middleware');
    let d = require('another-debug-middleware');
    middleware = [ ...middleware, c, d ];
  }

  const store = createStore(
    reducer,
    initialState,
    applyMiddleware(...middleware)
  )
  ```

  This makes it easier for bundling tools to cut out unneeded modules and reduces the size of your builds.

* Ever wondered what `applyMiddleware` itself is? It ought to be an extension mechanism more powerful than the middleware itself. Indeed, `applyMiddleware` is an example of the most powerful Redux extension mechanism called [store enhancers](../Glossary.md#store-enhancer). It is highly unlikely you’ll ever want to write a store enhancer yourself. Another example of a store enhancer is [redux-devtools](https://github.com/gaearon/redux-devtools). Middleware is less powerful than a store enhancer, but it is easier to write.

* Middleware sounds much more complicated than it really is. The only way to really understand middleware is to see how the existing middleware works, and try to write your own. The function nesting can be intimidating, but most of the middleware you’ll find are, in fact, 10-liners, and the nesting and composability is what makes the middleware system powerful.

* To apply multiple store enhancers, you may use [`compose()`](./compose.md).
