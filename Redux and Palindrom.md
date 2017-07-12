# Using Starcounter With React and Redux

## Introduction
When it comes to Starcounter, what you see from the frontend's perspective is a library called Palindrom. This library is a library that hands you a server-synchronized view-model JavaScript object. This object is called `Palindrom.obj`, or Palindrom Object.

Meaning, if you modify this object the server will be instantly updated and you'll have the same data on both the server and the client sides at any given moment. Same for the Server side, if you modify anything on the server side, your update will be instantly synchronized with the Palindrom object without any effort. No REST API, no ajax calls. None of that.

Maybe Palindrom website has a nice demo on its homepage (https://palindrom.github.io),
![image](https://user-images.githubusercontent.com/17054134/28120912-bf109856-6719-11e7-932c-84fd50341034.png)

_you can modify the objects on the website for easy understanding_.

## Palindrom and React

Well, Palindrom is a lot more than a synced object library. It has callbacks for virtually all needed use-cases. It has `onLocalChange`, `onRemoteChange` and `onStateReset` etc. It's well documented on the website.

The point of mentioning these callbacks is to show how easy they make your life! Consider this React snippet:
```js
componentDidMount() {
 palindrom.onRemoteChange = () => this.setState({...palindrom.obj});
}
```
Assuming your app is a single component app, this tiny snippet will ensure all server changes are reflected instantly to your component's state! No ajax calls, no API, no nothing.

Regarding syncing client side changes with the server, you could use `onComponentUpdated` and alter `palindrom.obj` the way you like and it will be synced, too.

```js
onComponentUpdated() {
Object.assign(palindrom.obj, this.state);
}
// done
```
## Redux

Of course, life is more complex than that, there are no single component apps in real-life. We need smarter ways to manage our states. Hence Redux was created.

### Redux and Palindrom can play very well together

There are virtually infinite solutions to connecting the two. I'll propose one of those ways:

Obviously, we have two data directions, upward and downward.

Upward: data originating from the client that should go to the server, we could use a Redux middleware:

```
const palindromMiddleware = store => next => action => {
  if (action.type === 'CHANGE_NAME') {
    palindrom.obj.name = action.payload.
  }
  next(action);
};
```
Your assumed original reducer:
```
const nameReducer = (state = {}, action) => {
  switch (action.type) {
    case 'CHANGE_NAME':
      return Object.assign({}, state, { name: action.payload } );
  }
  return state;
};
// Palindrom comes in here:
const middleware = redux.applyMiddleware(palindromMiddleware);
const store = redux.createStore(nameReducer, middleware);
// now you can use this store as a normal store throughout your app!
```

Now, we're done with the upward data, let's talk downward (coming from the server):

### There are two ways:
#### The easy one:

We could simply create a `STATE_RESET` action that has the new Palindrom object as payload. Such as:

```
const reducer = (state = {}, action) => {
  switch (action.type) {
    case 'STATE_RESET':
      return Object.assign({}, state, palindrom.obj );
  }
  return state;
};
```
And `dispatch` this action inside `onRemoteChange` callback.

```js
palindrom.onRemoteChange = () => {
store.dispatch(stateReset(palindrom.obj))
}
```

Now, this callback is only called after the Palindrom state is updated, and this means we can invoke `STATE_RESET` action synchronously, sticking to React's guidelines.

####  The more specific one:
You see, `onRemoteChange` passes to you the patches that were applied causing the change. And these patches include a path to each changed variable, example:
```js
//from the server
palindrom.name = 'Omar';
```
Now `onRemoteChange` will be called with such argument:
```js
onRemoteChange(data)
// => data is [{op: 'replace', path: '/name', value: 'Omar'}]
```
You can use this information to alter certain parts of the state instead of copying the whole thing.

In this example, `/name` was changed. So you could dispatch `NAME_CHANGED` action. You could create an action for every path of your state tree.
