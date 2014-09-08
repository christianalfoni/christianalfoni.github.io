---
layout: post
title:  "jQuery as a framework, could that work?"
date:   2014-09-08 09:21:30
categories: javascript
tags: ["javascript"]
---

We have all come to know and love our JS frameworks. Ember JS, Angular JS, React JS, Backbone, Knockout etc. All of them has given us the power of productivity and they make it even more fun to write code.

What I have realised though is that I never use all the features the framework has to offer, mostly a small percentage of it. There are so many concepts, so many methods, sometimes it does not look like JavaScript and I am still having problems keeping my code sane and scalable. 

It is difficult to identify the exact point where you start feeling bad about your code, and what you did to get to that point. Though frameworks helps you feel good about your code in general, the mental image of your application, implementation of new code and introduction of new team members is a big challenge nevertheless. So I have been pondering on this idea. What if I take the really awesome concepts from the big frameworks and create a very small and strict one? Would it make things better?

You can take a look at the API documentation and download examples over at the [repo](https://github.com/christianalfoni/jframework), but there will be lots of code examples here, so please read on.

### Getting to the core of it
You basically want to control two things in your application. **State** and **UI**. If some state changes you want the UI to update. If the user interacts with the UI, you probably want some state to update.

{% highlight console %}
|-------|     |----|
| STATE | <-> | UI |
|-------|     |----|
{% endhighlight %}

In MVC the state is spread between your domain models which holds the state of your database entities, and the controllers which holds the state of your application. Your controllers mutate the models directly and/or their internal controller state. 

{% highlight console %}
|---------------|     |-----------------------|     |-----------|
| MODEL (STATE) | <-> | CONTROLLER (STATE/UI) | <-> | VIEW (UI) |
|---------------|     |-----------------------|     |-----------|
{% endhighlight %}

Looking at this overview it looks very simple, but as your application grows, this is what happens:

{% highlight console %}
|-------|       |------------|       |------|
| MODEL | <---> | CONTROLLER | <---> | VIEW |
|       | <--   |------------|       |------|
|-------|   |         |
            |         |
|-------|   |   |------------|       |------|
| MODEL |   --> | CONTROLLER | <---> | VIEW |
|       | <---> |            |       |      |
|-------|       |------------|       |------|
                      |
                |------------|       |------|
                | CONTROLLER | <---> | VIEW |
                |------------|       |------|
{% endhighlight %}
The point of this diagram is to show you how things start to depend on each other and that data flows in both directions. This makes it very hard to scale the application. It is a challenge to make new implementations aware of the state changes on your existing models, and more challenging is making the implementations aware of state changes in other controllers.

### FLUX
Facebooks FLUX architecture aims to solve this problem by creating a unidirectonal flow. You have your states and the only way those states can change is by actions going through a dispatcher, down to your state handlers. These state handlers, called stores in FLUX, will notify your UI layer, called components. It looks something like this:

{% highlight console %}
    |---------------|     |---------------|     |----------------|
--> | DISPATCHER    | --> | STORE (state) | --> | COMPONENT (UI) | --
|   |---------------|     |---------------|     |----------------|  |
|                                                                   |
-------------------------- ACTIONS ----------------------------------
{% endhighlight %}

It is not possible for your component to manipulate the state of your application directly. That said, you will hold state in your UI too, but that is related only to that specific component. F.ex. "Has all required inputs a value?". `hasAllRequiredInputsValue` is a state that could be used to enable/disable a save button in your component. That is typically only a concern for the component itself. But lets say you do implement an other component that needs to be aware of the `hasAllRequiredInputsValue` state. In MVC this state would probably be defined in a controller and it would be very tempting to create a relationship between the existing controller and your new controller, but that is exactly where things start to go wrong. It would look something like this:

{% highlight console %}
|------------|       |------|
| CONTROLLER | <---> | VIEW |
|------------|       |------|
      |
      |
|------------|       |------|
| CONTROLLER | <---> | VIEW |
|------------|       |------|
{% endhighlight %}

In FLUX though, you would solve the problem like this:

{% highlight console %}
    |---------------|      |---------------|      |----------------|
--> | DISPATCHER    | ---> | STORE (state) | ---> | COMPONENT (UI) | --
|   |---------------|      |---------------|  |   |----------------|  |
|                                             |                       |
|                                             |   |----------------|  |
|                                             --> | COMPONENT (UI) | -|
|                                                 |----------------|  |
|                                                                     |
-----------------------------------------------------------------------
{% endhighlight %}

As you can see it is a lot easier to reason about what is going on in this diagram. Our `hasAllRequiredInputsValue`-state is moved to a store and in the process making it available to any other current and future components as well.

And this is how FLUX scales. You keep adding stores and components and it does not get more complex, only bigger. And most importantly, stateupdates only flow in one direction.

{% highlight console %}
    |---------------|      |---------------|      |----------------|
--> | DISPATCHER    | ---> | STORE (state) | ---> | COMPONENT (UI) | --
|   |---------------|  |   |---------------|  |   |----------------|  |
|                      |                      |                       |
|                      |                      |   |----------------|  |
|                      |                      --> | COMPONENT (UI) | -|
|                      |                          |----------------|  |
|                      |                                              |
|                      |   |---------------|      |----------------|  |
|                      --> | STORE (state) |  --> | COMPONENT (UI) | -|
|                          |---------------|      |----------------|  |
|                                                                     |
-----------------------------------------------------------------------
{% endhighlight %}

So FLUX is great and with React JS as your component tool you have a very good setup. My issues with React JS though is the philosphy of rerendering the whole application on state changes, instead of notifying the specific components (UI) that rely on those changes. React JS is also complex. There are many tools, addons and method names to keep track of. It is neither a complete framework like Ember JS and Angular JS, at least not yet. This is not to say React JS is bad, it is awesome! It just does not solve everything I want it to solve.

### jFramework in short
jFramework takes the principles of FLUX, forcing a unidirectonal flow, but does not have a philosophy of rerendering the whole application on state changes. Only components that depend on certain states will do a smart rerender, removing/adding/replacing DOM content required to keep in sync. Any nested components will only be affected if their parent component passes properties that have changed. jFramework also introduces all the tools needed to build an application, like a router, actions and state stores. It is also focused on being a very restricted and simple API. It should be easy to understand, remember and be productive in. And of course have fun with :-)

#### So you thought jQuery was out?
React JS uses a string representation of the UI before and after a state change to calculate DOM operations that is required to make the necessary changes to the UI. jFramework does something similar. It builds up its own internal structure of the UI using jQuery and checks differences on an object level, not a string level. The nice thing about jQuery is the amazing amounts of plugins, when put in a composable component concept becomes even more powerful.

I think it is time to check some code. All the examples below are shown in the CommonJS pattern so that you can see how the code is written with modules. You are free to use it globally though, just make sure you load jQuery first.

### Components
{% highlight javascript %}
var $$ = require('jframework');
module.exports = $$.component(function (template) {

    this.render(function () {
        return template(
            '<h1>',
                'Hello world!',
            '</h1>'
        );
    });
    
});
{% endhighlight %}
This code defines a render-callback that runs whenever the component needs to render. The render-callback needs to return a template. A template can be created by using the one argument passed to the component itself. The function takes unlimited arguments that builds up and returns a DOM structure. This syntax makes it very easy to write HTML in javascript and it gives some key advantages. Lets have a look at these advantages to get a better understanding of our first example.

#### Scope in components
{% highlight javascript %}
var $$ = require('jframework');
module.exports = $$.component(function (template) {

  var myStaticValue = 'foo';
  
  this.render(function () {
    return template(
      '<h1>',
        myStaticValue,
      '</h1>'
    );
  });
});
{% endhighlight %}
In this example we have a string value of `foo`, but it could have been a number or an array of templates. String and number values will be converted to a text node.
#### Passing properties to a component
{% highlight javascript %}
/* MyComponent.js */
var $$ = require('jframework');
module.exports = $$.component(function (template) {

  this.render(function () {
    return template(
      '<h1>',
        this.props.title,
      '</h1>'
    );
  });
  
});

/* main.js */
var $ = require('jquery');
var $$ = require('jframework');
$(function () {
  $$.render(MyComponent({title: 'Hello world!'}), 'body');
});
{% endhighlight %}
You can pass properties to an object and use them with `this.props` in the scope of the component.
#### Composing components
{% highlight javascript %}
/* Title.js */
var $$ = require('jframework');
module.exports = $$.component(function (template) {

  this.render(function () {
    return template(
      '<small>',
        this.props.title,
      '</small>'
    );
  });
  
});

/* MyComponent.js */
var $$ = require('jframework');
var Title = require('./Title.js');
module.exports = $$.component(function (template) {

  this.render(function () {
    return template(
      '<h1>',
        Title({title: 'Wazup?'}),
      '</h1>'
    );
  });
  
});
{% endhighlight %}
You can use a component inside an other component. If the title value passed by `MyComponent` would change, `Title` would rerender itself with the updated value.
#### Create lists
{% highlight javascript %}
var $$ = require('jframework');
module.exports = $$.component(function (template) {

  this.render(function () {
  
    var items = ['foo', 'bar'].map(function (title) {
      return template(
        '<li>',
          title,
        '</li>'
      );
    });
    
    return template(
      '<ul>',
        items,
      '</ul>'
    );
    
  });
  
});
{% endhighlight %}
In the render callback you can create an array of templates. If you are going to mutate the list itself or its contents, the main node of the list item will need an ID. The ID helps jFramework to keep track of the items in the list when updating it. Usually you will have an ID related to the items in a list, but it can be anything, just as long as it is unique for the item.

**Pro tip** Templates returned in a list could be components. Be sure to pass an ID property to keep track of the components in the list, `Item({id: 'foo'})`.
#### Listening to UI events
{% highlight javascript %}
var $$ = require('jframework');
module.exports = $$.component(function (template) {

  this.log = function () {
    console.log('I was clicked!');
  };

  this.listenTo('click', this.log);
  this.render(function () {
    return template(
      '<button>',
        'Click me!',
      '</button>'
    );
  });
});
{% endhighlight %}
You can listen to UI events with normal jQuery code, using `listenTo`. The callback passed will be bound to the component context, again making `this` point to the component. In this case the top node was the button to listen for. If the button was further down in the template tree you would do this:
{% highlight javascript %}
var $$ = require('jframework');
module.exports = $$.component(function (template) {

  this.log = function () {
    console.log('I was clicked!');
  };

  this.listenTo('click', 'button', this.log);
  this.render(function () {
    return template(
      '<div>',
        '<button>',
          'Click me!',
        '</button>',
      '</div>'
    );
  });
});
{% endhighlight %}
As you can see you use jQuery delegated events. You can use any jQuery selector to handle UI events in your template.
#### Using jQuery plugins
There are lots of jQuery plugins available and you can use them with jFramework. Normally when triggering plugins you do this:
{% highlight javascript %}
$('#myDiv').somePlugin();
{% endhighlight %}
In jFramework you use a `plugin` method to achieve the same thing:
{% highlight javascript %}
var $$ = require('jframework');
module.exports = $$.component(function (template) {

  this.plugin('somePlugin');
  this.render(function () {
    return template(
      '<div></div>'
    );
  });

});
{% endhighlight %}

In this example the plugin is activated on the top node of the template. If you want to activate a plugin in a nested element, you would do this:
{% highlight javascript %}
module.exports = $$.component(function (template) {

  this.plugin('somePlugin', 'button');
  this.render(function () {
    return template(
      '<div>',
        '<button>My plugin button</button>',
      '</div>'
    );
  });

});
{% endhighlight %}
You can use any jQuery selector as the second argument. If the last argument is an object, that will be passed to the plugin itself. So:
{% highlight javascript %}
/* jQuery way */
$('#myPlugin').somePlugin({option1: 'value1'});

/* jFramework way */
this.plugin('somePlugin', '#myPlugin', {options1: 'value1'});

{% endhighlight %}
So f.ex. a dropdown from the Bootstrap library could become a configurable component by doing something like this:
{% highlight javascript %}
/* BootstrapDropdown.js */
var $$ = require('jframework');
module.exports = $$.component(function (template) {

  this.plugin('dropdown');
  this.render(function () {
  
    var items = this.props.items.map(function (item) {
        return template(
          '<li role="presentation">',
            '<a role="menuitem" tabindex="-1" href="#">', 
              item,
            '</a>',
          '</li>'
        );
    });
    
    return template(
      '<div class="dropdown">',
        '<a data-toggle="dropdown" href="#">',
          this.props.title,
        '</a>',
        '<ul class="dropdown-menu" role="menu" aria-labelledby="dLabel">',
          items,
        '</ul>',
      '</div>'
    );
    
  });
  
});

/* main.js */
var $ = require('jquery');
var BootstrapDropdown = require('./BootstrapDropdown.js');
$(function () {
  $$.render(BootstrapDropdown({title: 'MyDropDown', items: ['foo', 'bar']}), 'body');
});
{% endhighlight %}
Now you are starting to see how powerful these components are becoming. Any jQuery plugin available can be used. Just pass the name of the plugin, an optional selector if the element is nested in the template and an optional options object that will be passed to the plugin.
### Actions
To update any state in your application you will need an action. Actions can be defined as easily as:
{% highlight javascript %}
/* myAction.js */
var $$ = require('jframework');
module.exports = $$.action();

/* someOtherFile.js */
var myAction = require('./myAction.js');
myAction('foo', 'bar');
{% endhighlight %}
As you can see in the example you can call that action with any number of arguments. These arguments will be passed to anyone who is listening to the action. You will probably have quite a few actions, so for conveniance you can do:
{% highlight javascript %}
/* actions.js */
var $$ = require('jframework');
var actions = $$.action([
  'addTodo',
  'removeTodo'
]);

/* someOtherFile.js */
var actions = require('./actions.js');
actions.addTodo();
actions.removeTodo();
{% endhighlight %}
This gives you a manifest of what actions are possible in your application. These actions defines what kind of state changes can be done. Just by reading your list of actions you see what your application is able to do.
### States
A state in jFramework can listen to actions, change their internal state and notify components about state changes.
{% highlight javascript %}
var $$ = require('jframework');
var actions = require('actions');
module.exports = $$.state(function (flush) {
  var todos = [];
  
  this.addTodo = function (todo) {
    todo.id = $$.generateId();
    todos.push(todo);
    flush();
  };
  
  this.listenTo(actions.addTodo, this.addTodo);
  this.exports({
    getTodos: function () {
      return todos;
    }
  });
});
{% endhighlight %}
So there are a few things happening here. First of all there is a flush function passed to the state callback. When called it will notify all listening components about a change in that specific state. We also see a variable `todos` declared. It is a good convention to declare variables for state values and point to the state itself with `this` to create methods mutating those values, like our `addTodo` method. We also see that we listen to the `addTodo` action, triggering the `addTodo` method in the state context. We also call an `exports` method. This method takes an object with methods that will be available when pointing to the state from your components. You might notice that the syntax here looks a lot like a component. That is no coinsidence. Lets look at some more code:
{% highlight javascript %}
var $$ = require('jframework');
var actions = require('actions');
var AppState = require('./AppState.js');
module.exports = $$.component(function (template) {

  /* METHODS */
  this.addTodo = function (todo) {
    actions.addTodo({
      title: 'foo',
      completed: false
    });
  };
  
  /* LISTENERS */
  this.listenTo(AppState, this.update);
  this.listenTo('click', 'button', this.addTodo);
  
  /* RENDER */
  this.render(function () {
    var todos = AppState.getTodos().map(function (todo) {
      return template(
        '<li id="' + todo.id + '">',
          todo.title,
        '</li>'
      );
    });
    return template(
      '<div>',
        '<button>Add todo</button>',
        '<ul>',
          todos,
        '</ul>',
      '</div>'
    );
  });
});
{% endhighlight %}
In this example our component listens to the `AppState`. If it triggers it will run the update method on itself. The update method rerenders the component. You can call `this.update()` at any time, though you will mostly use it with state updates. We are also listening to clicks on the button in our template. When triggered the component will create a new todo and call the `addTodo` action, passing the new todo. Lets sum this up.

This is what happens when the component is made available to the user:

1. Component DOM node is created with its children
2. Component listens to delegated button clicks on its main node
2. Component listens to updates on AppState
3. Component main node is added to the DOM

The flow:

1. User clicks button
2. The **component** runs `addTodo`
3. `addTodo` triggers the `addTodo` **action**, passing a new todo
4. Our **state** called AppState is notified and adds an id to the new todo before adding it to the list
5. The **Component** is notified about the update and runs its update method. That will in turn run the render callback and do a calculated DOM operation which adds a new li element to the ul

As highlighted in bold, this is the flow:
{% highlight console %}
    |---------|     |-------|     |-----------|
--> | ACTIONS | --> | STATE | --> | COMPONENT | --
|   |---------|     |-------|     |-----------|  |
|                                                |
--------------------------------------------------
{% endhighlight %}

What is really nice about this is that if any other component needs to know about updates to the todos, they just have to listen to `AppState` and they will be notified and update themselves accordingly. 

### What is a framework without a router?
jFramework has a router that supports both hash urls and modern popstate. It supports dynamic urls and can redirect to other urls. This is a typical setup:

{% highlight javascript %}
var $$ = require('jframework');
var App = require('./App.js');
var Posts = require('./Posts.js');
var Post = require('./Post.js');

$$.route('/', function () {
  $$.render(App(), 'body');
});

$$.route('/posts', function () {
  $$.render(Posts(), 'body');
});

$$.route('/posts/{id}', function (params) {
  $$.render(Post(params), 'body');
});

$$.route('*', '/');

{% endhighlight %}
jFramework will use hash tags by default. The framework automatically unloads existing components when you target a node with a new component. If you target a node with the same kind of component it will not render a new one, but update the existing one if the properties has changed.

#### Nested routes
jFramework takes a different strategy in nested routes. Since it is aware of what components that are in the DOM it is every effective at handling only needed changes, much like components themselves. Lets look at an example:

{% highlight javascript %}
var $$ = require('jframework');
var App = require('./App.js');
var Posts = require('./Posts.js');
var Post = require('./Post.js');

$$.route('/', function () {
  $$.render(App(), 'body');
});

$$.route('/posts', function () {
  $$.render(App(), 'body');
  $$.render(Posts(), '#app-content');
});

$$.route('/posts/{id}', function (params) {
  $$.render(App(), 'body');
  $$.render(Posts(), '#app-content');
  $$.render(Post(), '#posts-content');
});

{% endhighlight %}

This gives a clear definition of what happens when hitting the route. You could even make the route do different things based on some state. Maybe if there is a user logged in you render more components to the page. You can do whatever you want. Do not worry about overriding components as they will either be updated or removed automatically by jFramework.

### Summary
You can read the full API documentation and the current state of the project at [jFramework](https://github.com/christianalfoni/jframework). Hopefully I get my ideas through with this post and I would sure appreciate you giving it a try. If you do, please provide some feedback. Thanks for reading!