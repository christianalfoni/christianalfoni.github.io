---
layout: post
title:  "Using FLUX with Angular JS, revisited"
date:   2014-11-02 09:21:30
categories: javascript
tags: ["javascript", "angularjs", "flux"]
---

After writing [my initial article about Angular JS and FLUX](http://www.christianalfoni.com/javascript/2014/09/25/using-flux-with-angular.html) I worked on bringing the experiences I had building [www.jsfridge.com](http://www.jsfridge.com) with [jFlux](http://www.jflux.io) to React JS. This also ended up in an article, [My experience building a FLUX application](http://christianalfoni.github.io/javascript/2014/10/27/my-experiences-building-a-flux-application.html).

Shortly after I watched the [ng-conf Misko talk](https://www.youtube.com/watch?v=lGdnh8QSPPk) and I was both surprised and happy to see that Google will embrace FLUX in Angular 2.0. How exactly is yet to see, but it left me with some thoughts on how to use FLUX with Angular JS. So I wanted to write an article that brings up the concepts, the challenges with Angular and how you can start using FLUX in Angular JS today using [flux-angular](https://github.com/christianalfoni/flux-angular).

You can have a look at a [fiddle](http://www.jsfridge.com/fiddles/1414929331028) over at JSFridge if you want to. It is a small TODO-app with API implementation. Keep reading to explore how we make Angular JS work with FLUX.

### Immutability
The first thing we have to fix, that goes totally against one of the coolest things about Angular JS is the two-way-databinding. Two-way-databinding is awesome, but there are situations it does more harm than good. It works very well when you define a state on the $scope of your controller and you use that in a template. It is a controlled environment because a controller and a template is a "one to one" relationship. If you were to use a service to handle the state though, you suddenly get a "one to many" relationship. The reason is that a service can be used by many controllers. If all those controllers are able to just mutate state in the service you risk loosing control. So two-way-databinding is a good thing between your controllers and templates, but not between your controllers and services.

This is where FLUX fits in nicely. It does not go against how Angular JS works, it just fixes the parts that does not work too well, which is state handling with services.

First lets look at the code that creates the state in a service and binds it to a $scope:

{% highlight javascript %}
angular.module('app', ['flux'])
  .factory('MyStore', function (flux) {
    return flux.store({
      todos: [],
      exports: {
        getTodos: function () {
          return this.todos;
        }
      }
    });
  })
  .controller('MyCtrl', function (MyStore, $scope) {
    MyStore.bindTo($scope, function () {
      $scope.todos = MyStore.getTodos();
    });
  });
{% endhighlight %}

So what is happening here? When you bind a flux store to a $scope  you have to define its behaviour. So in the callback of the binding process you collect state from the store and attach it to your $scope. What also happens in the background is that the store keeps a reference to the callback. Lets see a simple FLUX in action:

{% highlight javascript %}
angular.module('app', ['flux'])
  .factory('actions', function (flux) {
    return flux.actions([
      'addTodo' // We add an action
    ]);
  })
  .factory('MyStore', function (flux, actions) {
    return flux.store({
      todos: [],
      actions: [
        actions.addTodo // We listen to the action
      ],
      addTodo: function (title) {
        this.todos.push({title: title}); // We update the state
        this.emitChange(); // We trigger a change
      },
      exports: {
        getTodos: function () {
          return this.todos;
        }
      }
    });
  })
  .controller('MyCtrl', function (MyStore, $scope, actions) {
    MyStore.bindTo($scope, function () {
      $scope.todos = MyStore.getTodos();
    });
    
    // This is a controller specific $scope property. Here it is
    // perfectly fine to have two-way-databinding
    $scope.newTitle = ''; 
    $scope.addTodo = function () {
      actions.addTodo($scope.newTitle);
    };
  });
{% endhighlight %}

So when the controller triggers an action and the store adds a new todo to the todos-array it triggers "emitChange". Under the hood the store will go through the bound callbacks and run them. You might think this is alot of extra effort and not very performant. If so, you are partly right. The extra effort is necessary because we do not want our controllers to change stuff in our stores directly, they should do it by using actions. This is really important because this is how you are kept sane, this is how you manage to scale your application and still understand how everything is connected. When it comes to perfomance it is of course extra calculations, but again, the dirty checking by Angular is neither very performant compared to other solutions, but it makes things easier for you. So dirty checking means less code, immutability in stores keeps you sane.

### Building stores
So a store in flux has a very strong concept of state handling. Angular does not really have an application wide concept of holding state, it is often locked in controllers. Though you are able to share state in services, that is not their "selling point". What I means is that services are more often used as a link between your backend and controller or they do some heavy logic, not to just hold and share state between your controllers. 

So a store definition looks something like this:

{% highlight javascript %}
angular.module('app', ['flux'])
  .factory('MyStore', function (flux, actions, MyStoreMixin) {
    return flux.store({
      todos: [],
      mixins: [MyStoreMixin]
      actions: [
        actions.addTodo
      ],
      addTodo: function (title) {
        this.todos.push({title: title});
        this.emitChange();
        this.emit('todos:add');
      },
      exports: {
        getTodos: function () {
          return this.todos;
        }
      }
    });
  })
{% endhighlight %}

#### State
You put your state properties directly on the store, like the **todos** in the example above. These states are available in the handlers. Your export methods are bound to the store, making the state properties available there too.

#### Mixins
Your stores will hold quite a bit of logic and can grow pretty large. To handle scalability you could create multiple stores, but you have to be careful about that. The reason is that splitting up stores might make them dependant of each other and cause circular dependencies. In my experience it is better to identify a store as part of a section of your application, maybe a specific page or an isolated section of a page. Then you use mixins to divide the logic of the store. Again, looking at todos, you might have the logic for handling actions and update the todos array in the store, then you add a mixin to handle the server communication. Lets see a simple example:

{% highlight javascript %}
angular.module('app', ['flux'])
  .factory('HTTPMixin', function ($http) {
    return {
      isSaving: false,
      postTodo: function (todo) {
        this.isSaving = true;
        this.emitChange();
        $http.post('/API/todos', todo)
          .success(function (newTodo) {
            todo.id = newTodo.id;
            this.isSaving = false;
            this.emitChange();
          }.bind(this));
      }
    };
  })
  .factory('MyStore', function (flux, actions, HTTPMixin) {
    return flux.store({
      todos: [],
      mixins: [HTTPMixin]
      actions: [
        actions.addTodo
      ],
      addTodo: function (title) {
        var todo = {title: title, checked: false};
        this.todos.push(todo);
        this.postTodo(todo);
        this.emitChange();
      },
      exports: {
        getTodos: function () {
          return this.todos;
        }
      }
    });
  })
{% endhighlight %}

So a mixin is just an object that will be merged with your store. All properties will merge as expected. As you can see in this example we have split the concern of the store into handling actions and handling HTTP communication. You can do this however you want, but the advantage is that all mixins has access to all state and all handlers.

#### Actions
Actions are a very simple concept. It is just an array with names. By looking at your actions service you can see what state changes can be done in your application. It is a good reference. You can call an action from anywhere in your application, it does not really matter. What matters is that an intent to change the state in a store should always start with an action. Even if your store needs to talk to the backend to get its initial state, that should be triggered with an action.

#### Handlers
When a store is configured with an action that action will map directly to a handler with the same name. So the action: "actions.addTodo" maps directly to a method in the store called: "addTodo". The handler can point to the store with **this** and mutate the state with whatever logic you need. When they are done mutating the state they can update the bound $scopes and/or broadcast an event to all the active $scopes in the application.

#### Emit change
The **emitChange** method will run all the binding callbacks registered to the store and update the bound $scopes. This will also guarantee that a digest loop is run. This means that whenever you run **emitChange** you can feel certain that the UI will update.

#### Emitting events
Very often you just need to notify your controllers about something. **emitChange** makes it easy for you to reflect updates in the UI, but sometimes you just need to notify about a state change. This could be when a model is loaded from the server and you want to trigger a transition. It could be an animation you want to run when a todo is added to the array etc.

#### Exports
Exports are necessary to let controllers grab state from the stores. The returned value from an export method will always be a copy of the value it returns. This is what ensures that the store is immutable. When using **this** in an export method it points to the store itself.

### Summary
Using FLUX with Angular JS is absolutely possible and it makes it easier to scale your applications. It is different than what you are used to from MVC and you do a tiny bit more "wiring", but that is exactly what makes it easier to work with. You can safely move controllers around, change the UI and implement new controllers without ever worrying about how to grab "that state over there".

A quick summary:

  1. Go bananas with two-way-databinding between your templates and your controllers, "ng-model it up!"
  2. The only way to change states in a store is through an action, even loading initial data from the server. Remember, you can use actions anywhere
  3. Use **emitChange** to notify about changes in your store and extend with **emit** when you need to notify controllers about a specific change in a store
  4. **emitChange** ensures that the digest loop runs

A good mental image is that your state is now flowing on top of your controllers, not "in between". Enjoy building FLUX applications with [flux-angular](https://github.com/christianalfoni/flux-angular)!
