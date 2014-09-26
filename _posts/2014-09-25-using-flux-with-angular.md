---
layout: post
title:  "Is it possible to use the FLUX architecture with Angular JS?"
date:   2014-09-26 09:21:30
categories: javascript
tags: ["javascript", "FLUX"]
---
I had a presentation on the history of going from traditional web applications with MVC on the server, moving over to the client with stuffed script tags running Ajax and jQuery code and to the adoption of MVC in the browser environment. I ended the talk with FLUX concepts and how it helps us handle application state.

After almost shitting my pants during my first one hour talk on development I got a lot of good questions on MVC and FLUX. One of those questions was: "Can I use FLUX with Angular?". My answer to this was:

*"Well, Angular is a MV-whatever framework. By its very definition it does not handle the FLUX architecture. Angular favors mutating the state of the application both ways, with its two way databinding. FLUX has a "one way flow" of state."*

In the presentation I used a lot of examples of how you can try to handle scaling with Angular, as that is a problem with MV\* in my opinion. I showed examples of how you could use events, nesting, $rootScope and services to communicate application state and make that available to future implementations without affecting existing ones. 

Then I woke up the morning after thinking about this question. Would it be possible to create a FLUX plugin for Angular? Something that would help you put your application state where it should be, create a one way flow of state and still make it feel like Angular?

### What the hell are you trying to solve... really?
In short. MVC is a pattern traditionally used on the server to build web applications. With a database, a router and a templating engine you had everything you needed to produce the views that was passed to the browser. What we have to remember is that there was no state passed to the browser, only the UI. To get or change state of the application you would have to trigger a URL, either by a hyperlink or posting a form. What also to notice is that the state on the server was mostly entities in a database.

When MVC moved over to the browser it did not change much how we built applications, the focus was to give a better user experience, no page reloads. The state of the application was still mostly based on a database with entities we wanted to present in the UI. Backbone JS at its core is a very good example of this. It has a Model-concept to fetch database entities and a router (Controller) to produce its Views based on a Model.

#### Application state
As time passed we started creating more and more complex applications and the big thing that changed was what I call "application state". The majority of state was no longer our Model-layer, with database entities, it was application specific state that had nothing to do with the server. I am talking about the type of state we call: "isPlaying", "showChatWindow" and "currentDuration". Not only are we getting a shit load of new states, we also want to react to the state in a different way.

Traditionally a statechange means: "Update the UI". This is the only way we can react to a statechange when building web applications on the server with MVC and only passing HTML to the browser. Now as we handle our state in the browser, we are also able to trigger functions on statechange. This is handled very differently than updating the UI.

A simple example of this could be a "PLAY" button the should change its text to "PAUSE" when clicked. A different part of your application will also react to this statechange and trigger a function that sets off an interval that will update a duration state each second. I will show this example in a way the scales with Angular:

{% highlight javascript %}
angular.module('app', [])
  .service('AppState', function () {
    return {
      isPlaying: false,
      duration: 0
    };
  })
  .controller('PlayCtrl', function ($scope, AppState) {
    $scope.app = AppState;
    $scope.togglePlay = function () {
      AppState.isPlaying = !AppState.isPlaying;
    }
  })
  .controller('DurationCtrl', function ($scope, AppState, $interval) {
    var durationInterval = null;
    $scope.app = AppState;
    $scope.$watch('app.isPlaying', function (isPlaying) {
      if (isPlaying) {
        durationInterval = $interval(function () {
          AppState.duration++;
        });
      } else {
        $interval.cancel(durationInterval);
      }
    });
  });
{% endhighlight %}

{% highlight html %}
{% raw %}
<div>
  <div ng-controller="PlayCtrl">
    <button ng-click="togglePlay()">
      {{app.isPlaying ? 'Pause' : 'Play'}}
    </button>
  </div>
  <div ng-controller="DurationCtrl">
    {{app.duration}} seconds passed
  </div>
</div>
{% endraw %}
{% endhighlight %}

What we see here is how the reaction to a statechange differs when we want to update the UI and when we want to trigger a function. A UI update happens automatically, but if we want to trigger a function we need to use the $watch method on our $scope. The issue here is that we start to handle state inside the controller. The $watch code actually belongs in our "AppState"-service.

#### The lines and the arrows
Something else that changed drastically moving MVC to the browser environment was the MVC pattern itself:

{% highlight console %}
| MVC - Implementation pattern |

|------|          |------------|          |-------|
| VIEW | < ---- > | CONTROLLER | < ---- > | MODEL |
|------|          |------------|          |-------|

{% endhighlight %}

It changed to some variation of this:

{% highlight console %}
| MVC - Implementation pattern |

|------|          |------------|          |-------|
| VIEW | < ---- > | CONTROLLER | < ---- > | MODEL |
|------|          |------------|          |-------|
                        |            
               |--------|---------|
               |                  |
         |------------|   |------------|          |-------|
         | CONTROLLER |   | CONTROLLER | < ---- > | MODEL |
         |------------|   |------------|          |-------|
         
{% endhighlight %}

The application state that does not fit in a Model-concept has a tendency to get spread between controllers. The problem is that new implementations might not have access to the application state it needs, because it is defined in an existing controller. To try to fix it we trigger events, we use $rootScope, we start nesting controllers and we use services. My opinion is that MVC fundamentally does not tackle application state, and with good reason, it did not have to. In traditional server side MVC applications there was no application state, or at least extremely limited compared to modern web applications.

So to summarize this. We need more than a "Model"-concept that reflects the state of database entries in a view. We need to handle a lot of application state that only lives in the browser. We also need a way to react to statechanges with function calls effectively. But the most important thing here is that we need a specific "application state"-concept that lets us easily reason about our application structure.

### FLUX does that
Facebooks FLUX architecture has a very strong concept of application state. That is great! What is not so great is the verbosity of its current tools. A lot of developers get scared, with good reason, seeing HTML inside JavaScript and the general verbosity of React JS compared to Angular JS. Personally I think it is a good thing to have HTML definition with the controller logic as they are connected, but it is a lot easier to adopt Angular JS because it honors something we already know very well, HTML templates. What I think most people do not consider here is that React JS (Components in flux) encourages building smaller pieces of UI. In Angular a controller can wrap ALOT of HTML which would look awful inside a JavaScript file, even in the template itself, but developing with React JS means building many small components instead.

#### But this is about FLUX, not Angular JS vs React JS
The message I am trying to pass here is that FLUX can help you build better apps, but currently it is a bit clouded by React JS. React JS is just a component tool that can be used with any MVC or FLUX framework.

But lets get into the good stuff. I am going to go through implementation specific details of refactoring the Angular JS Todo app at: [todomvc.com](http://www.todomvc.com) and explain how it makes it easier to scale the application at a later point. Please use the todomvc.com Angular JS project as a reference.

### The FLUX service
To create an Angular JS application with the FLUX architecture we need some tools. Specifically this [angular-flux](https://github.com/christianalfoni/angular-flux) plugin. It exposes a "flux" service that you can use to create your actions and your stores. The plugin does not have a dispatcher concept as the dispatcher only passes an action to a store.

So let us build an "actions"-service first:

{% highlight javascript %}
angular.module('todomvc', ['flux'])
  .service('actions', function (flux) {
  
    return flux.createActions([
      'addTodo',
      'removeTodo',
      'updateTodo',
      'clearCompletedTodos',
      'markAll',
      'toggleCompleted'        
    ]);
    
  });
{% endhighlight %}

So these actions are the statechanges possible to run in our application. The actions-service becomes a manifest of functionality that very nicely describes our application and what it can do. If you are not familiar with FLUX the actions are the only way to change states in your application.

What we need next is our store. Lets just call that "store" and add some state to it:

{% highlight javascript %}
angular.module('todomvc')
  .service('store', function (flux, todoStorage) {
  
    return flux.createStore(function () {
    
      this.addState({
        todos: todoStorage.get(),
        stats: '',
        statusFilter: null,
        remainingCount: 0,
        completedCount: 0,
        allChecked: false
      });
    
    });
    
  });
{% endhighlight %}

And then we add the state in our store to the $scope of the controller:

{% highlight javascript %}
angular.module('todomvc')
  .controller('TodoCtrl', function ($scope, store) {
    store.addStateTo($scope);
  });
{% endhighlight %}

Okay, so what did we do here? First of all we defined the actions that we will use shortly. Then we created a store that is responsible for changing all the state in our application. To populate our scope with the state from the store we use the **store.addStateTo($scope)** method. So in our HTML we can now grab state like normal:

{% highlight html %}
{% raw %}
<div>
  <div ng-controller="TodoCtrl">
    {{remainingCount}}
  </div>
</div>
{% endraw %}
{% endhighlight %}

#### Adding todos
So lets have a look at how a specific state is changed. Lets add a todo:

{% highlight html %}
<form id="todo-form" ng-submit="addTodo()">
  <input id="new-todo" placeholder="What needs to be done?" ng-model="titles.newTodo" autofocus>
</form>
{% endhighlight %}

By convention we point to a $scope object called "titles" and its "newTodo" property. This property is used to later reference the new title of the todo. You might argue that this is application state and should be put into our store, but that would create a "two way flow" of data between the controller and the store that we do not want. Two way databinding is great between the template and the controller, to f.ex. in this case grab the value of the input, but it ends there. The only reason for putting this "titles.newTodo" state in the store would be to allow other controllers, also in the future, to access the state. My decision was that "titles.newtodo" is specific to that controller.

{% highlight javascript %}
angular.module('todomvc')
  .controller('TodoCtrl', function ($scope, store, actions) {
  
    store.addStateTo($scope);
    $scope.titles = {
      newTodo: ''
    };
    
    $scope.addTodo = function () {
      actions.addTodo({
        title: $scope.titles.newTodo,
        completed: false
      });
      $scope.titles.newTodo = '';
    }
    
  });
{% endhighlight %}

So now we start to see FLUX in action. We have no logic for handling the title at all, we just pass it with an action and reset the newTodo title. In our Store we can now do:

{% highlight javascript %}
angular.module('todomvc')
  .service('AppStore', function (flux, todoStorage, actions) {
  
    return flux.createStore(function () {
    
      this.addState({
        todos: todoStorage.get(),
        stats: '',
        statusFilter: null,
        remainingCount: 0,
        completedCount: 0,
        allChecked: false
      });
    
      this.addTodo = function (todo) {
        var todos = this.getState('todos');
        todo.title = todo.title.trim();
        if (todo.title) {
          todos.push(todo);
        }
      };
      
      this.listenTo(actions.addTodo, this.addTodo);
    
    });
    
  });
{% endhighlight %}

So first of all we create a method for handling the state change. All it does is trim the text, check if it has a valid value and then pushes a new object into the todos array. We grab the todos array by using a **getState** method.

In traditional Facebook FLUX we would now have to trigger an event to notify the components about an update. In Angular JS that is not necessary because of the databinding. The state of this store has been attached to the scope of our controller so everything just works. We are still honoring the principles of FLUX, the state is only flowing down.

### Lets look at some improvements
Now you have actually seen the whole thing in action. Everything from now on will work exactly like this. We will only trigger actions and our store handles the state update. It is a "one way flow", meaning that all statechanges start at the "top" of your application and are then being acted upon in your stores first, then further down into your controllers and finally reflecting any state in the templates. We allow for two-way databinding between the template and the controller specific properties, as they are tightly connected and that is where two way databinding should occurr. But to change state in a store you must trigger an action. That is the important part.

#### A very short controller
This is actually the whole controller now:

{% highlight javascript %}
angular.module('todomvc')
  .controller('TodoCtrl', function TodoCtrl ($scope, store, actions) {
    'use strict';

    // We attach the state to the scope
    store.addStateTo($scope);

    // Controller specific scope properties
    $scope.titles = {
      newTodo: '',
      editedTodo: ''
    };
    $scope.editedTodo = null;

    // Our methods
    
    $scope.addTodo = function () {
      actions.addTodo({
        title: $scope.titles.newTodo,
        completed: false
      });
      $scope.titles.newTodo = '';
    };

    $scope.editTodo = function (todo) {
      $scope.editedTodo = todo;
      $scope.titles.editedTodo = todo.title;
    };

    $scope.doneEditing = function (todo) {
      actions.updateTodo($scope.titles.editedTodo, todo);
      $scope.editedTodo = null;
    };

    $scope.revertEditing = function () {
      $scope.editedTodo = null;
    };

    $scope.removeTodo = function (todo) {
      actions.removeTodo(todo);
    };

    $scope.toggleCompleted = function (todo) {
      actions.toggleCompleted(todo);
    };

    $scope.clearCompletedTodos = function () {
      actions.clearCompletedTodos();
    };

    $scope.markAll = function (completed) {
      actions.markAll(completed);
    };

  });
{% endhighlight %}

As we can see there is not much going on in our controller. We are mostly passing actions to the "top" of our application, awaiting application wide state changes, and some controller specific behaviour.

#### Reacting to statechange without $watch
As all statechanges now happens with actions there is no need to use the expensive $watch any longer. This section of code in the todomvc.com **controller**...

{% highlight javascript %}
$scope.$watch('todos', function (newValue, oldValue) {
  $scope.remainingCount = $filter('filter')(todos, { completed: false }).length;
  $scope.completedCount = todos.length - $scope.remainingCount;
  $scope.allChecked = !$scope.remainingCount;
  if (newValue !== oldValue) { // This prevents unneeded calls to the local storage
    todoStorage.put(todos);
  }
}, true);
{% endhighlight %}

...has now become this section of code in our **store**:

{% highlight javascript %}
this.updateStats = function () {
  var todos = this.getState('todos');
  var remainingCount = todos.filter(function (todo) {
    return !todo.completed;
  }).length;
  this.setState({
    remainingCount: remainingCount,
    completedCount: todos.length - remainingCount,
    allChecked: !remainingCount
  });
};
{% endhighlight %}

This **updateStats** method gets called whenever a todo is added, removed or a todo has changed its completed state.

#### Actions are awesome
Lets imagine that we wanted to implement WebSockets and update our list with new todos as they were added by other users. We could reuse our action "addTodo". This is a websocket jibberish example, but the point here is the reuse of the action:

{% highlight javascript %}
angular.module('app')
  .service('WebSocket', function (actions) {
    var connection = new WebSocket();
    connection.onmessage = function (todo) {
      actions.addTodo(JSON.parse(todo));
    };
  });
{% endhighlight %}

It does not matter who triggers the action, it can come from anywhere, not just the UI.

#### Implementing new controllers
Lets say we wanted a controller to display a message for 2 seconds when a new todo was added to the list AND we wanted the new todo to indicate that it was a new todo when arriving in the UI. With our current setup that is no problem, just adding to our current code:

{% highlight javascript %}
angular.module('todomvc')
  .service('AppStore', function (flux, todoStorage, actions) {
  
    return flux.createStore(function () {
    
      this.addState({
        todos: todoStorage.get(),
        stats: '',
        statusFilter: null,
        remainingCount: 0,
        completedCount: 0,
        allChecked: false,
        newTodo: null // New state
      });
    
      this.addTodo = function (todo) {
        var todos = this.getState('todos');
        todo.title = todo.title.trim();
        if (todo.title) {
          todos.push(todo);
          this.setState('newTodo', todo); // We set the new todo as newTodo
          this.saveTodos();
          
          // We run a timeout to reset the newTodo state after two seconds
          setTimeout(function () {
            this.setState('newTodo', null);
          }.bind(this), 2000);
        }
      };
      
      this.listenTo(actions.addTodo, this.addTodo);
    
    });
    
  });
{% endhighlight %}

And our todos HTML:

{% highlight html %}
{% raw %}
<!-- We have added a new class "newTodo" if the todo is the newTodo -->
<li 
  ng-repeat="todo in todos | filter:statusFilter track by $index"
  ng-class="{newTodo: todo == newTodo, completed: todo.completed, editing: todo == editedTodo}"
>
  <div class="view">
    <input class="toggle" type="checkbox" ng-model="todo.completed">
    <label ng-dblclick="editTodo(todo)">{{todo.title}}</label>
    <button class="destroy" ng-click="removeTodo(todo)"></button>
  </div>
  <form ng-submit="doneEditing(todo)">
    <input 
      class="edit" 
      ng-trim="false" 
      ng-value="todo.title" 
      todo-escape="revertEditing(todo)" 
      ng-blur="doneEditing(todo)" 
      todo-focus="todo == editedTodo"
    >
  </form>
</li>
{% endraw %}
{% endhighlight %}

And our new controller, just to show how easy it is to extend:

{% highlight javascript %}
angular.module('todomvc')
  .controller('NotifyCtrl', function ($scope, store) {
    'use strict';

    // We attach only the newTodo state to the scope
    store.addStateTo($scope, ['newTodo']);

  });
{% endhighlight %}

And the small piece of HTML showing it:
{% highlight html %}
{% raw %}
  <div ng-controller="NotifyCtrl" ng-show="newTodo">
    {{newTodo.title}} was just added to the list!
  </div>
{% endraw %}
{% endhighlight %}

#### Triggering events
Angular events are attached to the $scope chain and that can be a problem because you might need that event both up the $scope chain and down, and maybe even to a sibling controller which is not reachable with an event at all. The good thing is that events are usually triggered when states are changed, so that is a perfect job for our store. It uses the $rootScope and emits the event down to all your controllers.

So lets say we had a fox next to the todoslist and we had a range of sounds we wanted to randomly play... like: "What does the fox say?". Showing the change to the **addTodo** method:

{% highlight javascript %}
angular.module('todomvc')
  .service('AppStore', function (flux, todoStorage, actions) {
  
    return flux.createStore(function () {
        
      this.addTodo = function (todo) {
        var todos = this.getState('todos');
        todo.title = todo.title.trim();
        if (todo.title) {
        
          todos.push(todo);
          this.setState('newTodo', todo);
          this.saveTodos();
          
          setTimeout(function () {
            this.setState('newTodo', null);
          }.bind(this), 2000);
          
          this.emit('what:does:the:fox:say'); // Emitting an event to all controllers
        }
      };
      
      this.listenTo(actions.addTodo, this.addTodo);
    
    });
    
  });
{% endhighlight %}

And our controller:

{% highlight javascript %}
angular.module('todomvc')
  .controller('FoxCtrl', function ($scope, soundService) {
    'use strict';
    
    var sounds = ['hatihatihatiho', 'dingdingdingdingdingedingeding', 'powpowpowpowpow'];
    
    $scope.animateFox = false;
    
    $scope.$on('what:does:the:fox:say', function () {
      var index = Math.round(Math.random() * 2);
      $scope.animateFox = true;
      soundService.play(sounds[index]).then(function () { $scope.animateFox = false; });
    });

  });
{% endhighlight %}

### Summary
I think FLUX is a very exciting way to go. It is an architecture based on our experiences so far building very complex web applications. MVC was, and still is, a very good fit for traditional web applications, but as we put more and more application state into our code we need something that keeps us sane.

If you want to try out this plugin you can download it from this repo [angular-flux](https://github.com/christianalfoni/angular-flux). There you can also find the complete refactored TodoMVC application. If this was not your cup of tea I hope it at least gave you some "food for thought". Thanks for listening!