# React (Redux) FSA with WebSocket integration

Integrating WebSockets into your project is always fun, but did you ever think that you can actually create an event emitter that would help you keep the same flow between Redux and your backend all in one. Well, during development of a FinTech product we were guided by an idea of achieving the architectural needs and create a smart solution.

> **Tip:** The code provided below will require for you to have good understanding of React, Redux and Sagas, so I would definitely suggest you getting some knowledge here:
> - [React](https://reactjs.org/)
> - [Redux](https://redux.js.org/)
> - [Sagas](https://redux-saga.js.org/)


## Why we built this?

Initially the idea was creating a singular function that would request a response from our backend through WebSockets and still fit the needs of the business logic in our app. We have used ReactJS as frontend library with Redux state management and Redux-Sagas as the middleware on frontend, while on backend we used Python and WebSockets. Since the project was dependent on live updates, but also static calls we have thought of an idea of integrating FSA as the standardized message for both the communication between backend and the frontend.

## What is FSA?

A basic Flux Standard Action can be defined through a following example:
```
{
  type: 'ADD_TODO',
  payload: {
    text: 'Do something.'
  }
}
```
It is a standardized form of an object that represent action, containing type and payload. In FSA the obligatory field for having a valid action would be type.

An action MUST

-   be a plain JavaScript object.
-   have a  `type`  property.

An action MAY

-   have an  `error`  property.
-   have a  `payload`  property.
-   have a  `meta`  property.

An action MUST NOT include properties other than  `type`,  `payload`,  `error`, and  `meta`.

## Idea: Higher Order Action Creator?

I was pretty sure that when learning React and Redux, the HoC (Higher order Component) was going to be the hardest concept to grasp, but those were the good old days. The main idea leading the development was a unification of WS message with our Redux action. What we envisioned was the flow where we would emit a message through WS which would be registered by our backend and upon receiving data, emit the same message back with the same action type, but different payload, later to be picked up by our higher order action creator, this should definetly be a thing "HoA". The message received would later on be dispatched into our Redux flow.

## Thought process

The thought process behind the following idea created the following logic. We were aware that we have a single idea of a dispatch/emit that actually has the same purpose, and that is to dispatch a message, either to a subscriber or to the middleware and reducers. So let's note that:

1. **Dispatch**
2. **Emit**

## Our Redux Architecture

As mentioned in the introduction we are using Sagas as middleware which handles all of the asynchronous requests and in our specific case also handles WebSocket emits.

![Handle Side-Effects With Redux-Saga - Frontend - Scalac.io](https://scalac.io/wp-content/uploads/2019/02/redux-middleware-diagram.png)

Sagas enable numerous approaches to tackling parallel execution, task concurrency, task racing, task cancellation, and more. It keeps total control over the flow of your code.

## Enough talk, show me the code.

We first defined an action type: `WS_SINGLE`, afterwards we started mapping out our action creator function, since we are doing one singular request we called it `getSingle`. So you can already envision what parameters this function needs to get in order to create another action creator that we would use in our redux system.

1. Event string, this will be used so we can track the event that is going to be **emitted** to WebSocket `eg. 'COMPANY_DATA'`.
2. Emit function, here is the action creator that we create afterwards, so it's pretty clear where we got the **HoA** name from, this is going to be our **dispatch** function when we pass it in argument.
3. Action: We need to pass the action that we are going to emit into our WebSocket, which is in FSA format.

```
const getSingle = (
  event: EventType,
  emit: EmitType,
  initialAction: InitialAction,
) => (...)
```

As this is an action creator itself, we are using our `getSingle` function as the action creator that will be called in order to dispatch the function with following values:

```
{
  type: types.WS_SINGLE,
  payload: {
    event,
    emit,
    initialAction,
  },
}
```

Now that we have our action creator let's go to our saga and to see how it actually works, by initializing it.

With function event argument:
```
getSingle(
  (evt) => evt.type === 'TICKER_DATA' && evt.meta.requestId = '123',
  (data) => ({ type: 'RECEIVE_TICKER_DATA', payload: data }),
  { type: 'SUBSCRIBE_TICKER_DATA', payload: ['AAPL'] }
)
```
To dispatch local redux action with same shape as WebSocket response,
pass in the identity function
```
getSingle(
  (evt) => evt.type === 'TICKER_DATA' && evt.meta.requestId = '123',
  (action) => action,
  { type: 'SUBSCRIBE_TICKER_DATA', payload: ['AAPL'] }
)
```

## How it actually works?

This is where the magic actually happens and where we execute our logic. First of all we assign the action type to our Saga.
```
...
takeEvery(types.WS_SINGLE, handleSingleRequest),
...
```
Now for the moment of truth:
```
function * handleSingleRequest(action) {
  const {
    event, emit, initialAction,
  } = action.payload;

  const id = uuid();
  let emitActions = [];
  if (Array.isArray(emit)) {
    emit.forEach((emitter) => emitActions.push(getEmitWithId(id, emitter)));
  } else {
    emitActions = getEmitWithId(id, emit);
  }

  yield put(subscribe(id, event, emitActions, initialAction));
  yield take((a) => (a.meta || {}).requestId === id);
  yield put(unsubscribe(id));
}
```
*Explanation of the function generator:* The emit will push the action into the stream and wait for the response by using the `take` method which is a "blocker" and when we get the response we dispatch the response we got from the publish method on our Python backend creating a flow of communication that is directly integrated in our Redux architecture and creating a reusable solution that will allow us to send one action which will hold the response from the WebSocket.

## TLDR

The idea of creating a WebSocket and Redux direct integration and in a way a unification between an **emit** and **dispatch** was that we created a structure on our backend and used **FSA** as the message format that matches the actions format in our software. The solution was creating an Higher order Action creator, which allowed us to pass an action creator that will create an action which later going to be dispatched with the payload containing the data from the backend and directly passed through our Redux system into our *Sagas*, *Reducers* and *Selectors*.

With this we created a specific action creator that will eventually create the second one. First action will be emitted to WebSocket, than we will `take`, by the type passed as an argument, it with some nice data from backend and dispatch the new or even the same action from the backend into middleware and reducers.



## Example dashboard using this architecture

![Yewno edge dashboard](https://ant-strapi-storage-bucket.fra1.digitaloceanspaces.com/antcolony-website/82a060985a338cc7df8d6cefafca91e6.png)