---
title: You don't have to use Redux
published: true
description: 
tags: React, Redux, Context API, Javascript
---

---

A React application is basically a tree of components that communicate data with each other. Passing data between components is often painless. However, as the app tree grows, it becomes harder to pass that data while maintaining a sound & readable codebase.

Say we have the following tree structure: 
![](https://thepracticaldev.s3.amazonaws.com/i/axnmkq4b19fxeso4lb7z.png)
Here we have a simple tree with 3 levels. In this tree, node D and node E both manipulate some similar data: **Say the user inputs some text in node D, which we wish to display in node E**. 
![](https://thepracticaldev.s3.amazonaws.com/i/f30lljepc1kzgozol7xt.gif)
### _How do we pass that data from node D to node E?_
The article presents 3 possible approaches to tackle this issue:
- Prop drilling
- Redux
- React's context API

The aim of the article is to compare these approaches and show that, when it comes to solving a common problem such as the one we just worded, it is possible to just stick with React's context API.

## Approach 1: Prop drilling
A way of doing it would be to naively pass the data from child to parent then from parent to child through props as such: D->B->A then A->C->E. 

The idea here is to use the `onUserInput` function triggered from child to parent to carry the input data from node D to the state at node A, then we pass that data from the state at node A to node E. 

We start with node D:

```javascript
class NodeD extends Component {
  render() {
    return (
      <div className="Child element">
        <center> D </center>
        <textarea
          type="text"
          value={this.props.inputValue}
          onChange={e => this.props.onUserInput(e.target.value)}
        />
      </div>
    );
  }
}
```

When the user types something, the `onChange` listener will trigger the `onUserInput` function from the prop and pass in the user input. That function in the node D prop will trigger another `onUserInput` function in the node B prop as such:

```javascript
class NodeB extends Component {
  render() {
    return (
      <div className="Tree element">
        <center> B</center>
        <NodeD onUserInput={inputValue => this.props.onUserInput(inputValue)} />
      </div>
    );
  }
}
```

Finally, when reaching the root node A, the `onUserInput` triggered in the node B prop will change the state in node A to the user input. 

```javascript
class NodeA extends Component {
  state = {
    inputValue: ""
  };

  render() {
    return (
      <div className="Root element">
        <center> A </center>
        <NodeB
          onUserInput={inputValue => this.setState({ inputValue: inputValue })}
        />
        <NodeC inputValue={this.state.inputValue} />
      </div>
    );
  }
}
```

That **inputValue** will then be through props from Node C to its child Node E:

```javascript
class NodeE extends Component {
  render() {
    return (
      <div className="Child element">
        <center> E </center>
        {this.props.inputValue}
      </div>
    );
  }
}
```

See it already added some complexity to our code even if it's just a small example. Can you image how it would become when the app grows? 🤔

This approach relies on the number of depth of the tree, so for a larger depth we would need to go through a larger layer of components. This can be too long to implement, too repetitive and increases code complexity.


---

## Approach 2: Using Redux
Another way would be to use a state management library like Redux.
> Redux is a predictable state container for JavaScript apps.
The state of our whole application is stored in an object tree within a single store, which your app components depend on. Every component is connected directly to the global store, and the global store life cycle is independent of the components' life cycle.

![](https://thepracticaldev.s3.amazonaws.com/i/5n276dibl7bxch5mls63.png)

We first define the state of our app: The data we interested in is what the user types in node D. We want to make that data available to node E. To do that, we can make that data available in our store. Node E can then subscribe to it in order to access the data. 
We will come back to the store in a bit.

### Step 1: Define Reducer
The next thing is to define our reducer. Our reducer specifies how the application's state changes in response to actions sent to the store. We define our reducer block as such:

```js
const initialState = {
  inputValue: ""
};

const reducer = (state = initialState, action) => {
  if (action.type === "USER_INPUT") {
    return {
      inputValue: action.inputValue
    };
  }
  return state;
};
```

Before the user has typed anything, we know that our state's data or **inputValue** will be an empty string. So we define a default initial state to our reducer with an empty string **inputValue**. 

> The logic here is once the user types something in node D, we "trigger" or rather **dispatch** an **action** and our reducer updates the state to whatever has been typed. By "update" here I do not mean "mutate" or change the current state, I do mean **return a new state**.

The if statement maps the dispatched action based on its type to the new state to be returned. So we already know that the dispatched action is an object containing a type key. How do we get the user input value for the new state ? We simply add another key called **inputValue** to our action object, and in our reducer block we make the new state's inputValue has that input value with `action.inputValue`. So the actions of our app will follow this architecture:
 
``{ type: "SOME_TYPE", inputValue: "some_value" } ``

Ultimately, our dispatch statement will look like this:

``dispatch({ type: "SOME_TYPE", inputValue: "some_value" })``

And when we call that dispatch statement from whatever component, we pass in the type of the action, and the user input value. 

Okay, now we have an idea of how the app works: In our input node D we dispatch an action of type `USER_INPUT` and pass in the value of whatever the user just typed, and in our display node E we pass in the value of the current state of the app aka the user input.  

### Step 2: Define the Store
In order to make our store available, we pass it in a`Provider` component we import from react-redux. We then wrap our App inside of it. Since we know that nodes D and E will use the data in that store, we want our Provider component to contain a common parent of those nodes, so either root node A or our entire App component. Let us chose our App component to be contained in our Provider as such:
```js
import reducer from "./store/reducer";
import { createStore } from "redux";
import { Provider } from "react-redux";

const store = createStore(reducer);
ReactDOM.render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById("root")
);
```
Now that we have set up our store and reducer, we can get our hands dirty with our nodes D and E ! 

### Step 3: Implement user input logic
Let's first take a look at node D. We are interested in what the user inputs in the `textarea` element. This means two things:

1- We need to implement the `onChange` event listener and make it store whatever the user types in the store.
2- We need the value attribute of the `textarea` to be the value stored in our store.

But before doing any of that, we need to set up a few things: 

We first need to connect our node D component to our store. To do so, we use the `connect()` function from react-redux. It provides its connected component with the pieces of the data it needs from the store, and the functions it can use to dispatch actions to the store. 

> This is why we use the two `mapStateToProps` and `mapDispatchToProps` which deal with the store's state and dispatch respectively. We want our node D component to be subscribed to our store updates, as in, our app's state updates. This means that any time the app's state is updated, `mapStateToProps` will be called. The results of `mapStateToProps` is an object which will be merged into our node D's component props. Our `mapDispatchToProps` function lets us create functions that dispatch when called, and pass those functions as props to our component. We will make use of this by returning new function that calls `dispatch()` which passes in an action.

In our case, for the `mapStateToProps` function, we are only interested in the *inputValue*, so we return an object `{ inputValue: state.inputValue }`. For the `mapDispatchToProps`, we return a function `onUserInput` that takes the input value as parameter and dispatches an action of type `USER_INPUT` with that value. The new state object returned by `mapStateToProps` and the `onUserInput` function are merged into our component's props. So we define our component as such:
```js
class NodeD extends Component {
  render() {
    return (
      <div className="Child element">
        <center> D </center>
        <textarea
          type="text"
          value={this.props.inputValue}
          onChange={e => this.props.onUserInput(e.target.value)}
        />
      </div>
    );
  }
}
const mapStateToProps = state => {
  return {
    inputValue: state.inputValue
  };
};

const mapDispatchToProps = dispatch => {
  return {
    onUserInput: inputValue =>
      dispatch({ type: "USER_INPUT", inputValue: inputValue })
  };
};
export default connect(
  mapStateToProps,
  mapDispatchToProps
)(NodeD);
```
We are done with our node D! Let's now move on to node E, where we want to display the user input. 

### Step 4: Implement user output logic
We wish to display the user input data on this node. We already know that this data is basically what is in the current state of our app, as in, our store. So ultimately, we wish to access that store and display its data. To do so, we first need to subscribe our node E component to the store's updates using the `connect()` function with the same `mapStateToProps` function we used before. After that, we simply need to access the data in the store from the props of the component using **this.props.val** as such: 

```js
class NodeE extends Component {
  render() {
    return (
      <div className="Child element">
        <center> E </center>
        {this.props.val}
      </div>
    );
  }
}
const mapStateToProps = state => {
  return {
    val: state.inputValue
  };
};

export default connect(mapStateToProps)(NodeE);
```
And we're _finally_ done with Redux! 🎉 You can take a look at what we just did [here](https://codesandbox.io/s/reduxtree-2n7ct?fontsize=14).

In the case of a more complex example, say with a tree with more components that share/manipulate the store, we would need those two `mapStateToProps` and `mapDispatchToProps` functions at each component. In this case, it might be wiser to separate our action types and reducers from our components by creating a separate folder for each. 
_…Who's got the time right?_ 

## Approach 3:Using React's context API
Now let's redo the same example using the context API.
The React [Context API](https://medium.com/r/?url=https%3A%2F%2Freactjs.org%2Fdocs%2Fcontext.html) has been around for a while but only now in React's version [16.3.0](https://reactjs.org/docs/context.html) did it become safe to use in production. The logic here is close to Redux's logic: we have a context object which contains some global data that we wish to access from other components.
First we create a context object containing the initial state of our app as default state. We then create a `Provider` and a `Consumer` component as such:
```js
const initialState = {
  inputValue: ""
};

const Context = React.createContext(initialState);

export const Provider = Context.Provider;
export const Consumer = Context.Consumer;
```
> Our `Provider` component has as children all the components from which we want to access the context data. Like the `Provider` from the Redux version above. To extract or manipulate the context, we use our Consumer component which represents the component.

We want our `Provider` component to wrap our entire App, just like in the Redux version above. However, this `Provider` is a little different than the previous one we've seen. In our App component, we initialise a default state with some data, which we can share via value prop our `Provider` component. 
In our example, we're sharing **this.state.inputValue** along with a function that manipulates the state, as in, our onUserInput function. 
```js
class App extends React.Component {
  state = {
    inputValue: ""
  };

  onUserInput = newVal => {
    this.setState({ inputValue: newVal });
  };

  render() {
    return (
      <Provider
        value={{ val: this.state.inputValue, onUserInput: this.onUserInput }}
      >
        <div className="App">
          <NodeA />
        </div>
      </Provider>
    );
  }
}
```
Now we can go ahead and access the data of our `Provider` component using our Consumer component :) 
For node D in which the user inputs data: 
```js
const NodeD = () => {
  return (
    <div className="Child element">
      <center> D </center>
      <Consumer>
        {({ val, onUserInput }) => (
          <textarea
            type="text"
            value={val}
            onChange={e => onUserInput(e.target.value)}
          />
        )}
      </Consumer>
    </div>
  );
};
```
For node E in which we display the user input:
```js
const NodeE = () => {
  return (
    <div className="Child element ">
      <center> E </center>
      <Consumer>{context => <p>{context.val}</p>}</Consumer>
    </div>
  );
};
```

And we're done with our context version of the example! 🎉 It wasn't that hard was it ? Check it out [here](https://codesandbox.io/s/contextapiexample-wwqoc)
What if we have more components that we wish to be able to access the context? We can just wrap them with the Provider component and use the Consumer component to access/manipulate the context! _Easy :)_ 


---

## Ok, but which one should I use
We can see that our Redux version of the example took a bit more time to do than our Context version. We can already see that Redux:

- Requires **more lines of code** and can be too **"boilerplate"** with a more complex example (more components to access the store). 
- **Increases complexity**: It might be wiser to separate your reducer and action types from the components into unique folders/files when dealing with many components.
- Introduces a **learning curve**: Some developers find themselves struggling to learn Redux as it requires you to learn some new concepts: reducer, dispatch, action, thunk, middleware…

If you are working on a more complex App and wish to view a history of all the dispatched action by your app, "click" on any one of them and jump to that point in time, then definitely consider using Redux's pretty dope [devTools extension](https://github.com/zalmoxisus/redux-devtools-extension)! 

However if you're only interested in making some data global to access it from a bunch of components, you can see from our example that Redux and React's context API both do roughly the same thing. So in a way, _you don't have to use Redux!_
