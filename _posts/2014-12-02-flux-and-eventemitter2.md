---
layout: post
title:  "An alternative render strategy with flux and React JS"
date:   2014-12-04 09:21:30
categories: javascript
tags: ["javascript", "flux"]
---
So, this new awesome FLUX architecture is starting to get some solid ground. There are several implementations of it, amongst those [Yahoo Dispatchrı](https://github.com/yahoo/dispatchr), [Fluxxor](http://fluxxor.com/), [McFly](https://github.com/kenwheeler/mcfly) and [jFlux](http://www.jflux.io), which I have personally been working on. The core concept of FLUX is **stores** that will emit a change whenever any of the state held within the store changes. Any components listening to changes to these stores will then grab state from the stores and rerender. In this article we are going to take a look at what actually happens in React JS when a store triggers this **change** event. Not all FLUX implementations use React JS, but they would also benefit from the concepts we go through in this article. We are going to use [flux-react](https://github.com/christianalfoni/flux-react) in the examples as it has an API to handle this alternative strategy we are going to talk about.

### The change event
So reading through documentation about the FLUX architecture you will surely meet the "change event". Basically whenever a store has done a change to some state it will emit this one type of event, *change*. By having one type of event your application will of course be more managable in terms of keeping all your components in sync with the stores when a state change occurs, but it also has a cost. Lets look at an example with a **flux-react** store:

{% highlight javascript %}
var store = flux.createStore({
  todos: [],
  addTodo: function (todo) {
    this.todos.push(todo);
    this.emit('change');
  },
  removeTodo: function (index) {
    this.todos.splice(index, 1);
    this.emit('change');
  },
  exports: {
    getTodos: function () {
      return this.todos;
    }
  }
});
{% endhighlight %}

Any components listening to a change event on this store would use the **getTodos** method when todos are added and removed to update themselves. That makes sense and it is absolutely nothing wrong with that as it is highly likely that any component interested in adding todos would probably also be interested in their removals. But let us add another state:

{% highlight javascript %}
var store = flux.createStore({
  todos: [],
  isSaving: false,
  addTodo: function (todo) {
    this.todos.push(todo);
    this.isSaving = true;
    this.emit('change');
    doSomethingAsync().then(function () {
      this.isSaving = false;
      this.emit('change');
    }.bind(this));
  },
  removeTodo: function (index) {
    this.todos.splice(index, 1);
    this.isSaving = true;
    this.emit('change');
    doSomethingAsync().then(function () {
      this.isSaving = false;
      this.emit('change');
    }.bind(this));
  },
  exports: {
    getTodos: function () {
      return this.todos;
    },
    isSaving: function () {
      return this.isSaving;
    }
  }
});
{% endhighlight %}

Now we are first triggering a change to notify our components that the store is in **isSaving** state and that we have a new todo. Later we notify our components again about the store not saving anymore. With this simple example we are starting to see where things are starting to go in the wrong direction. It is not because we are doing an async operation inside a store, but because we use a general 'change' event both for notfifying about our **isSaving** state and the update to our **todos** state. Lets visualize what I mean here. Imagine this HTML being components:

{% highlight html %}
<div>
  <AddTodoComponent/>
  <TodosListComponent/>
</div>
{% endhighlight %}

We want the input inside **AddTodoComponent** to disable itself while the store is in **isSaving** state. It does that by listening to a change event in the store. In addition to this we also want our **TodosListComponent** to update itself when there are changes to the todos array and we of course listen to the same change event to accomplish that. So what happens is the following:

1. We grab both **isSaving** and **todos** when components are created
2. We add a new todo causing a "change" event to occur
3. The **AddTodoComponent** grabs the new **isSaving** state and the **TodosListComponent** grabs the mutated **todos** state
4. When the async operation is done we trigger a new change event causing again our two components to grab the same states, though the **AddTodoComponent** did not really have to, since there were no mutation on the todos array

But this is not the only challenge with React JS rendering. Lets look at a couple of other things, but remember this when we look at EventEmitter2.

### React JS cascading renders
One important detail about React JS that is often overlooked is how **setState** on a component affects the nested components. When you use **setState** the nested components will run a check to verify if they need to update the DOM. That means if a change event is being listened to on your application root node, whenever a change event is triggered from the store, all your components will do a render and a diff to produce any needed DOM operations. Lets visualize this:

{% highlight javascript %}
[Cascading render]

               |---|
               | X | - Root component renders
               |---|
                 |
            |----|---|
            |        |
          |---|    |---|
          | X |    | X | - Nested components also renders
          |---|    |---|
               
{% endhighlight %}

But if a nested component does a **setState** it will not affect parent components.

{% highlight javascript %}
[Cascading render]

               |---|
               |   | - Root component does not render
               |---|
                 |
            |----|---|
            |        |
          |---|    |---|
          |   |    | X | - Nested component renders
          |---|    |---|
               
{% endhighlight %}

This actually means that if you had one store you could get away with only listening to changes on your root component, triggering a setState and then just grab state from the stores in the nested components. An example of this would be our todo example.

{% highlight html %}
<TodoApp>
  <AddTodoComponent/>
  <TodosListComponent/>
</TodoApp>
{% endhighlight %}

Lets look at our TodoApp component:

{% highlight javascript %}
var TodoApp = React.createClass({
  componentWillMount: function () {
    AppStore.on('change', this.update);
  },
  componentWillUnMount: function () {
    AppStore.off('change', this,update);
  },
  update: function () {
    this.setState({}); // Just trigger a rerender
  },
  render: function () {
    return (
      <div>
        <AddTodoComponent/>
        <TodosListComponent/>
      </div>
    );
  }
});
{% endhighlight %}

This component is just listening to a general change event and triggers a rerender. As stated above this will cascade down to the nested components, so if we f.ex. in our **AddTodoComponent** do this:

{% highlight javascript %}
var AddTodoComponent = React.createClass({
  render: function () {
    return (
      <form>
        <input type="text" disabled={AppStore.isSaving()}/>
      </form>
    );
  }
});
{% endhighlight %}

That is actually all we need to handle the disabled state. There is not need to listen for changes because our root component does this for us. Now I am not saying that this is best practice, I just want to clearify how React JS operates and make you think about how you would handle your rendering in React JS. The hidden message here though is that if you have a change listener to a store on your root component your whole application will rerender on every change from that store.

#### Repeated rendering
What is even more important to understand about **setState** is that using a general change event will not only cause cascaded rendering but will also cause repeated rendering in components. Let me explain:

{% highlight javascript %}
[Repeated rendering]

               |---|
               |   | - Root component listens to change
               |---|
                 |
            |----|---|
            |        |
          |---|    |---|
          |   |    |   | - Nested components listens to change
          |---|    |---|
               
{% endhighlight %}

When a change event now occurs the root component wil first trigger a render:

{% highlight javascript %}
[Repeated rendering]

               |---|
               | X | - Root component reacts to change event and rerenders
               |---|
                 |
            |----|---|
            |        |
          |---|    |---|
          | X |    | X | - Nested components render
          |---|    |---|
               
{% endhighlight %}

And after that, the nested components will actually rerender themselves again:

{% highlight javascript %}
[Repeated rendering]

               |---|
               |   |
               |---|
                 |
            |----|---|
            |        |
          |---|    |---|
          | X |    | X | - Nested components react to change event and rerenders
          |---|    |---|
               
{% endhighlight %}

Now if there were deeper nested components this would cause the same effect, but with even more repeated rendering due to each nested level causes an extra rerender.

### How to optimize
First I would like to take a minute to look at the **shouldComponentUpdate** method. This is used to control the behavior of cascading rendering. Lets look at our visualization again:

{% highlight javascript %}
[Cascading render]

               |---|
               | X | - Root component renders
               |---|
                 |
            |----|---|
            |        |
          |---|    |---|
          | X |    | X | - Nested components also renders
          |---|    |---|
               
{% endhighlight %}

If our nested components had the **shouldComponentUpdate** method and it returned false:

{% highlight javascript %}
var NestedComponent = React.createClass({
  shouldComponentUpdate: function () {
    return false;
  },
  render: function () {
    return (
      <div></div>
    );
  }
});
{% endhighlight %}

This would be the result:

{% highlight javascript %}
[Render cascading]

               |---|
               | X | - Root component renders
               |---|
                 |
            |----|---|
            |        |
          |---|    |---|
          |   |    |   | - Nested components do not render
          |---|    |---|
               
{% endhighlight %}

But it is a pain to add this to all your components though. I do wonder why React JS does not by default prevent rerender when there are no properties passed to the nested component, or even by default do a comparison of the new properties to verify that a rerender actually is needed. It would at least be nice to configure React JS to have that behavior. Anyway, there is probably a very good reason.

### EventEmitter2 and flux-react
Now lets have a look at how we can use a different rendering strategy to control all this. 

{% highlight javascript %}
var TodosListComponent = React.createClass({
  componentWillMount: function () {
    AppStore.on('todos.*', this.setState);
  },
  componentWillUnmount: function () {
    AppStore.off('todos.*', this.setState);
  },
  render: function () {
    return (
      <ul>
        {AppStore.getTodos().map(function (todo) {
          return <li>{todo.title}</li>
        })}
      </ul>
    );
  }
});
{% endhighlight %}

As you can see I used an asterix wildcard. This actually means that the store emits the following event on adding a todo, **this.emit('todos.add')**, and, **this.emit('todos.remove')**, when removing a todo. Our list will act upon both events. This gives us some freedom because we do not want our input in **AddTodoComponent** to be disabled when removing a todo, only adding one. Since we have an async operation in our store it will also emit a **this.emit('todos.added')** after the async operation is finished. Letting us do this:

{% highlight javascript %}
var AddTodoComponent = React.createClass({
  componentWillMount: function () {
    AppStore.on('todos.add', this.update);
    AppStore.on('todos.added', this.update);
  },
  componentWillUnmount: function () {
    AppStore.off('todos.add', this.update);
    AppStore.off('todos.added', this.update);
  },
  update: function () {
    this.setState({});
  },
  render: function () {
    return (
      <form>
        <input type="text" disabled={AppStore.isSaving()}/>
      </form>
    );
  }
});
{% endhighlight %}

So in this example you clearly see what events in the store the component acts upon. The important thing to notice here is that the store holds the actual state, **isSaving**. We just tell our component to rerender when these two events occur, but it is still the state of the store that defines what will actually be rendered. We would NEVER do something like this:

{% highlight javascript %}
var AddTodoComponent = React.createClass({
  componentWillMount: function () {
    AppStore.on('todos.add', this.setDisabled);
    AppStore.on('todos.added', this.setEnabled);
  },
  componentWillUnmount: function () {
    AppStore.off('todos.add', this.setDisabled);
    AppStore.off('todos.added', this.setEnabled);
  },
  setDisabled: function () {
    this.setState({disabled: true });
  },
  setEnabled: function () {
    this.setState({disabled: false});
  },
  render: function () {
    return (
      <form>
        <input type="text" disabled={this.state.disabled}/>
      </form>
    );
  }
});
{% endhighlight %}

As this would ignore the whole concept of FLUX, keeping the state in your stores to make it easier to scale your application. 

### Keeping control of the rendering
Although we have given more specific behavior to our components they will still do this cascading rendering which is exactly what we are trying to avoid with this strategy. You could create your own **shouldComponentUpdate** method for each component, but [flux-react](https://github.com/christianalfoni/flux-react) has a mixin that does this for you. It also includes an **update** method to make it easier for you to refresh the component.

{% highlight javascript %}
var TodosListComponent = React.createClass({
  mixins: [flux.RenderMixin],
  componentWillMount: function () {
    AppStore.on('todos.*', this.update);
  },
  componentWillUnmount: function () {
    AppStore.off('todos.*', this.update);
  },
  render: function () {
    return (
      <ul>
        {AppStore.getTodos().map(function (todo) {
          return <li>{todo.title}</li>
        })}
      </ul>
    );
  }
});
{% endhighlight %}

The mixin will prevent the component from rerendering if there are no props passed to it or the props are the same as current props. This means that if the parent component of **TodosListComponent** would run **setState** this component would not update itself.

### Summary
So what we have done now is basically going from very loosely defined rerendering of your application to a very strict concept you have full control of. We did this by:

1. Use EventEmitter2 to specify what state and what changes to that state the component should be affected by
2. Use RenderMixin to avoid unecessary cascaded rendering and repeated rendering in nested components

Depending on your app you might not want to introduce this control as it does require more thought and effort, but again you will have more control of how your application performs related to rendering. Check out [flux-react](https://github.com/christianalfoni/flux-react) or use this pattern on your library of choice. You can also check out the cascading and repeating rendering over at this [fiddle](http://www.jsfridge.com/fiddles/1417595062356).