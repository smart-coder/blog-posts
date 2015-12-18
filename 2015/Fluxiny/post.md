# Dissection of Flux architecture or how to write your own flux 

I'm obsessed by making my code simpler. I didn't say *smaller* because having less code doesn't mean that is simple and easy to work with. I believe that big part of the problems in the software industry come from the unnecessary complexity. Complexity which is a result of our own abstractions. You know, we (the programmers) like to abstract. We like placing things in black boxes and hope that these boxes work together.

[Flux](http://facebook.github.io/flux/) is an architectural design pattern for building user interfaces. It was introduced by Facebook at their [F8](https://youtu.be/nYkdrAPrdcw?t=568) conference. Since then, lots of companies adopted the idea and it seems like a good pattern for building front-end apps. Flux is very often used with [React](http://facebook.github.io/react/). Another library released by Facebook. I myself use React+Flux in my [daily job](http://trialreach.com/) and I could say that the simplicity is one of the main benefits there. Flux as a pattern is simple enough to get your head around and React's API is really small.

## Flux architecture and its main characteristics

![Basic flux architecture](./fluxiny_basic_flux_architecture.jpg)

The main actor in this pattern is the **dispatcher**. It acts as a hub for all the events in the system. It's job is to receive notifications that we call **actions** and bypass them to all the **stores**. The store decides if it is interested or not and reacts by changing its internal state. The change is passed to the **views** which are (in my case) React components. If we have to compare Flux to the well known MVC we may say that the store is similar to the model.

The actions are coming to the dispatcher either from the views or from other part of the system like services. For example a module that performs a HTTP request. When it receives the result it may fire an action saying that the request was successful and attach the received data.

A wrong data flow is one of the biggest pitfalls in flux. For example, we may have an access to the store in our views but we should never call store methods that mutate its internal state. This should only happen via actions. Or we may end up with a store that receives and action and dispatches another one.

*(More or less the explanation above is based on my reading of the Flux's documentation and my day-to-day experience with the pattern. Have in mind that you may see other interpretations. And you better do :))*

## My two cents

As every other popular concept Flux also has some [variations](https://medium.com/social-tables-tech/we-compared-13-top-flux-implementations-you-won-t-believe-who-came-out-on-top-1063db32fe73). I decided to try and write my own. One of the main goals was to be as simple as possible. That's how [Fluxiny](https://github.com/krasimir/fluxiny) was born. 

<small>*You may be surprised but I invested a good amount of time in naming the repository. Do you know how difficult is to come with a short library name which at the same time is not registered in NPM? . For the record Fluxiny = Flux + tiny.*</small>

In the next few sections we'll see how I created [Fluxiny](https://github.com/krasimir/fluxiny). Why is that small and how I ended up having the code as it is.

### The dispatcher

In most of the cases we need a single dispatcher. Because it acts as a glue for the rest of the system's parts it makes sense that we have only one. There are two inputs to the dispatcher - actions and stores. The actions are simply forwarded to the stores so we don't necessary have to keep them. The stores however should be tracked inside the dispatcher:

![the dispatcher](./fluxiny_the_dispatcher.jpg)

That's what I started with:

```
var Dispatcher = function () {
  return {
    _stores: [],
    register: function (store) {  
      this._stores.push({ store: store });
    },
    dispatch: function (action) {
      if (this._stores.length > 0) {
        this._stores.forEach(function (entry) {
          entry.store.update(action);
        });
      }
    }
  }
};
```

The first thing that we notice is that we *expect* to see an `update` method in the passed stores. It will be nice to throw an error if such method is missing:

```
register: function (store) {
  if (!store || !store.update) {
    throw new Error('You should provide a store that has an `update` method.');
  } else {
    this._stores.push({ store: store });
  }
}
```

### Bounding the views and the stores

The next logical step is to connect our views to the stores so we rerender when the state in the stores is changed.

![Bounding the views and the stores](./fluxiny_store_view.jpg)


#### Using a helper

Some of the flux implementations available provide a helper function that does the job. For example:

```
var detach = Framework.attachToStore(view, store);
```

However, I don't quite like this approach. To make `attachStore` works we expect to see a specific API in the view and in the store. We kind of strictly define new public methods. Or in other words we say "Your views and store should have such APIs so we are able to connect them together". If we go down this road then we'll probably define our own base classes which could be extended so we don't bother the developer with Flux details. Then we say "All your classes should extend our classes". This doesn't sound good either because the developer may decide to switch to another Flux provider and he/she has to amend everything.

#### With a mixin

What if we use React's [mixins](https://facebook.github.io/react/docs/reusable-components.html#mixins).

```
var View = React.createClass({
  mixins: [Framework.attachToStore(store)]
  ...
});
```

That's a "nice" way to define behavior of existing React component. So, in theory we may create a mixin that does the bounding for us. To be honest, I don't think that this is a good idea. And [it looks](https://medium.com/@dan_abramov/mixins-are-dead-long-live-higher-order-components-94a0d2f9e750) like it's not only me. My reason of not liking mixins is that they modify the components in a non-predictable way. I have no idea what is going on behind the scenes. So I'm definitely crossing this option.

#### Using a context

Another technique that my answer to our question is React's [context](https://facebook.github.io/react/docs/context.html). It's a way to pass props to child components without the need to specify them in all the levels. Facebook suggests context in the cases where we have data that has to reach deeply nested compoments.

> Occasionally, you want to pass data through the component tree without having to pass the props down manually at every level. React's "context" feature lets you do this.

I see similarity with the mixins here. The context is defined somewhere at the top and magically serves props for all the children below. It's not immediately clear where the data comes from.

#### Higher-Order components concept

Higher-Order components pattern is [introduced](https://gist.github.com/sebmarkbage/ef0bf1f338a7182b6775) by Sebastian Markb&#229;ge and it's about creating a wrapper component that returns ours. However, while doing it it has the opportunity to send properties or apply additional logic. For example:

```
function attachToStore(Component, store, consumer) {
  const Wrapper = React.createClass({
    getInitialState() {
      return consumer(this.props, store);
    },
    componentDidMount() {
      store.onChangeEvent(this._handleStoreChange);
    },
    componentWillUnmount() {
      store.offChangeEvent(this._handleStoreChange);
    },
    _handleStoreChange() {
      if (this.isMounted()) {
        this.setState(consumer(this.props, store));
      }
    },
    render() {
      return <Component {...this.props} {...this.state} />;
    }
  });
  return Wrapper;
};
```

`Component` is our component. The view that we want attached to the `store`. The `consumer` function says what part of the store's state should be fetched and send to the view. A simple usage of the above function could be:

```
class MyView extends React.Component {
  ...
}

ProfilePage = connectToStores(MyView, store, (props, store) => ({
  data: store.get('key')
});

```

That's an interesting pattern because it shifts the responsibilities. It's the view fetching data from the store and not the store pushing something to the view. This of course has its own pros and cons. It is nice because it makes the store dummy. It only mutates the data and says "Oh, my state is changed". It is not responsible for sending anything. The downside of this approach is maybe the fact that we have one more component (the wrapper) and we need the three view, store and consumer in one place to fulfill the connection.

#### What I decided to do

The last option above, higher-order components, is really close to what I'm searching for. I like the fact that the view decides what it needs. That *knowledge* anyway exists in the component so it makes sense to keep it there. That's also why the functions that generate the higher-order components are usually kept in the same file as the view. What if we can use similar approach but not passing the store at all. Or in other words, a function that accepts only the consumer. And that function is called every time when there is a change in the store.

So far our implementation interacts with the store only in the `register` method. 

```
register: function (store) {
  if (!store || !store.update) {
    throw new Error('You should provide a store that has an `update` method.');
  } else {
    this._stores.push({ store: store });
  }
}
```

By using `register` we keep a reference to the store inside the dispatcher. However, `register` returns nothing. So, it's a nice candidate for a function tha accepts our consumer.

![Fluxiny - connect store and view](./fluxiny_store_view.jpg)

I decided to send the whole store to the consumer function and not the data that the store keeps. Like in the higher-order components pattern the view should say what it needs. This makes the store really simple and there is no trace of presentational logic.

Here is how the register method looks like after the changes:

```
register: function (store) {
  if (!store || !store.update) {
    throw new Error('You should provide a store that has an `update` method.');
  } else {
    var consumers = [];
    var subscribe = function (consumer) {
      consumers.push(consumer);
    };
    
    this._stores.push({ store: store });
    return subscribe;
  }
  return false;
}
```

The last bit in the story is how the store says that its internal state is changed. It's nice that we collect the consumer functions but right now there is no code that runs them. 

According to the basic principles of the flux architecture the stores change their state in response of actions. So, in the `update` method we send `action` but also a function `change`. Calling that function should trigger the consumers:

```
register: function (store) {
  if (!store || !store.update) {
    throw new Error('You should provide a store that has an `update` method.');
  } else {
    var consumers = [];
    var change = function () {
      consumers.forEach(function (l) { 
        l(store);
      });
    };
    var subscribe = function (consumer) {
      consumers.push(consumer);
    };
    
    this._stores.push({ store: store, change: change });
    return subscribe;
  }
  return false;
},
dispatch: function (action) {
  if (this._stores.length > 0) {
    this._stores.forEach(function (entry) {
      entry.store.update(action, entry.change);
    });
  }
}
```

<small>*Notice how we push `change` together with `store` inside the `_stores` array. Later in the `dispatch` method we call `update` by passing the `action` and the `change` function.*</small>

A common use case is to fetch an initial state of the data. This is required usually at the first render of the view. In the context of our implementation this means firing all the consumers at least once. This could be easily done in the `subscribe` method:

```
var subscribe = function (consumer, noInit) {
  consumers.push(consumer);
  !noInit ? consumer(store) : null;
};
```

Of course sometimes this is not needed so we add a flag which is by default falsy. Here is the final version of our dispatcher:

```
var Dispatcher = function () {
  return {
    _stores: [],
    register: function (store) {
      if (!store || !store.update) {
        throw new Error('You should provide a store that has an `update` method.');
      } else {
        var consumers = [];
        var change = function () {
          consumers.forEach(function (l) { 
            l(store);
          });
        };
        var subscribe = function (consumer, noInit) {
          consumers.push(consumer);
          !noInit ? consumer(store) : null;
        };
        
        this._stores.push({ store: store, change: change });
        return subscribe;
      }
      return false;
    },
    dispatch: function (action) {
      if (this._stores.length > 0) {
        this._stores.forEach(function (entry) {
          entry.store.update(action, entry.change);
        });
      }
    }
  }
};
```

## The actions

You probably noticed that we didn't talk about the actions. What are they? My opinion is that they should be simple objects having two properties - `type` and `payload`:

```
{
  type: 'USER_LOGIN_REQUEST',
  payload: {
    username: '...',
    password: '...'
  }
}
```

The `type` says what exactly the action is all about and the `payload` contains the information associated with the event. And in some cases we may leave the `payload` empty. 

It's interesting that the `type` is well known in the beginning. We know what type of actions should be floating in our app, who is dispatching them and which of the stores is interested. Thus, we can apply [partial application](http://krasimirtsonev.com/blog/article/a-story-about-currying-bind) and avoid passing the action object here and there. For example:

```
var createAction = function (type) {
  if (!type) {
    throw new Error('Please, provide action\'s type.');
  } else {
    return function (payload) {
      return dispatcher.dispatch({ type: type, payload: payload });
    }
  }
}
```

`createAction` leads to the following benefits:

* We no more need to remember the exact type of the action. We now have a function which we call passing only the payload.
* We no more need an access to the dispatcher which is a huge benefit. Otherwise, think about how we have to pass it to every single place where we need to dispatch an action. In the end we don't have to deal with objects but with functions which is much nicer. The objects are *static* while the functions describe a *process*.

![Fluxiny actions creators](./fluxiny_action_creator.jpg)

This approach for creating actions is actually really popular and functions like the one above are usually called **action creators**.

## The final code

In the section above we successfully hid the dispatcher while submitting actions. We may do it again for the store's registration: 

```
var createSubscriber = function (store) {
  return dispatcher.register(store);
}
```

And instead of exporting the dispatcher we may export only these two functions `createAction` and `createSubscriber`. Here is how the final code looks like:


```
var Dispatcher = function () {
  return {
    _stores: [],
    register: function (store) {
      if (!store || !store.update) {
        throw new Error('You should provide a store that has an `update` method.');
      } else {
        var consumers = [];
        var change = function () {
          consumers.forEach(function (l) { 
            l(store);
          });
        };
        var subscribe = function (consumer, noInit) {
          consumers.push(consumer);
          !noInit ? consumer(store) : null;
        };
        
        this._stores.push({ store: store, change: change });
        return subscribe;
      }
      return false;
    },
    dispatch: function (action) {
      if (this._stores.length > 0) {
        this._stores.forEach(function (entry) {
          entry.store.update(action, entry.change);
        });
      }
    }
  }
};

module.exports = {
  create: function () {
    var dispatcher = Dispatcher();

    return {
      createAction: function (type) {
        if (!type) {
          throw new Error('Please, provide action\'s type.');
        } else {
          return function (payload) {
            return dispatcher.dispatch({ type: type, payload: payload });
          }
        }
      },
      createSubscriber: function (store) {
        return dispatcher.register(store);
      }
    }
  }
};

```

53 lines of code, 1.45KB plain or 741 bytes after minification JavaScript.

## Wrapping up

Ok, we have a module that provides methods for using Flux. Let's write a simple example that not involves React. We'll need some UI to interact with it so:

```

```



























