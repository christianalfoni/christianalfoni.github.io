---
layout: post
title:  "My experiences building a FLUX application"
date:   2014-10-27 09:21:30
categories: javascript
tags: ["javascript", "FLUX"]
---

### Background
A few months ago I started a project, [www.jsfridge.com](http://www.jsfridge.com). In short it is a service that merges video education and fiddle services together in a new way to learn frontend web development. JSFridge has state up the wazoo. Actually it has exactly 35 application states, amongst those: "isPlaying", "isRecording", "isEditing", "currentScene", "duration", "isAudioReady" etc. In addition there is of course lots of view state. How can I say that my application has exactly 35 application states? Well, that is actually thanks to FLUX and we will get back to that.

I used Angular JS on the first iteration of building JSFridge, the ALPHA prototype. I had a pretty good idea of what I wanted and Angular JS makes you extremely productive. My solution to passing changes to application state was using events. I emitted events up the scope chain, and down the scope chain. My controllers had their own "event-controller" attached to them. There was a lot of state going all over the place. As I was working on the project every single day I managed to keep the complexity in my head, but my brain was constantly running on overdrive. Especially when I got to the bug-fixing part I had to lock myself in a room to not be disrupted, as I would totally loose may trail of thought. I could not continue using Angular JS going to BETA, at least not the way I was using it.

### My first look into React JS and FLUX
My first read on React JS and FLUX was on the Facebook site, [Facebook Flux](https://facebook.github.io/flux/docs/overview.html#content). I also watched the [Introduction to React JS](https://www.youtube.com/watch?v=XxVg_s8xAms). I thought; "This is what I need to keep me sane building the JSFridge BETA". I downloaded React JS and started to play around with it. I also had a look at the dispatcher and stores.

Its funny how we humans do not like things that are different and I am the first to admit that. Now having worked quite a bit with React JS I am comfortable with the API and I appreciate what it gives me. But there was a threshold. Initially I did not like the API. It felt like I was writing code even "below native JavaScript". What I mean is that "componentWillMount" and "componentDidUnmount" is nothing you are used to handling in other controller/view concepts in other frameworks. The fact that React JS and FLUX is not a complete toolset for building a webapplication was kind of a disappointment to me also. I was not used to choosing separate tools for routing, ajax etc. I wanted a complete framework. I wanted something like Backbone or Angular, only with FLUX.

#### Do it yourself sometimes work
So I decided to build my own FLUX framework inspired by the concepts of Facebook FLUX and React JS. I am not going to talk about that in this article, you can read more about it at [jflux.io](http://www.jflux.io), but the point is that it helped me figure out exactly what I wanted from FLUX and React JS to build a complex application like JSFridge. It also made me realise how brilliant React JS is, and why I just could not settle with the Dispatcher and Store concepts.

In this article we are going to go through the architecture of JSFridge and look at how jFlux concepts helped controlling the application state. Then, to finish off the article, we will look at how you can take these concepts I explain and use them with React JS.

### Application state in complex applications
So what FLUX initially helps us with is to introduce a strong concept of application state. Stores are what holds the state of your application, while a component holds the state of your view. A question quickly arises though and that is: "What is the scope of a store?". Do you want one store for your whole application or do you want to divide the responsibility of application state into multiple stores? Of course I could just answer that question with: "It depends", but there is actually a big problem with using multiple stores and that is the first thing we are going to look at.

#### Multiple or single stores?
This is pretty much the store setup of JSFridge:

{% highlight console %}
| FLUX - Stores |

  |-----------| |-----------| |-------------| |-------------------|
  | UserStore | | HomeStore | | CourseStore | | CreateCourseStore |
  |-----------| |-----------| |-------------| |-------------------|

{% endhighlight %}

But actually it did not start out like that. It was something like this:

{% highlight console %}
| FLUX - Stpres |

  |-----------| |-----------| |-------------| |-------------------|
  | UserStore | | HomeStore | | CourseStore | | CreateCourseStore |
  |-----------| |-----------| |-------------| |-------------------|
  
  |--------------| |---------------| |----------------| |------------|
  | ConsoleStore | | PlaybackStore | | RecordingStore | | SceneStore |
  |--------------| |---------------| |----------------| |------------|

{% endhighlight %}

What quickly became a problem with this setup was their circular dependencies. F.ex. the "isPlaying" state was stored in the "PlaybackStore" and the "isRecording" state was stored in the "RecordingStore", but both of them needed to know about each other in different scenarios. So what I realized is that you have two types of stores. You have application wide stores, like "UserStore" that is not dependant on any other stores. And then you have stores that represent a specific section in your application, like "CourseStore" and "HomeStore". It looks something like this:

{% highlight console %}
| FLUX - Stores |

  |-----------|
  | UserStore |
  |-----------|
        |
        |----------------------------------
        |              |                  |
  |-----------| |-------------| |-------------------|
  | HomeStore | | CourseStore | | CreateCourseStore |
  |-----------| |-------------| |-------------------|

{% endhighlight %}

So stores that are part of one section of your application should be contained within a single store. Think of them like pages of your webapp, a modal or some other contained section. Stores that has nothing to do with a specific section, like "UserStore", can also be contained within one store. What we want to avoid is stores being dependant on each other both ways. 

But now we run into a different problem. You can not put all your store code in one file, that will create quite a mess. The "CourseStore" of JSFridge is very big, but it is split into different files and handled much like "mixins" in React JS. We will look at a specific solution for this shortly.

#### Not only change events
A different issue I encountered was that not all state updates should trigger a change event. Let me use a simple example to explain this part:

In your store:
{% highlight javascript %}
  changeTitle: function (title) {
    this.title = title;
    this.emit('change');
  },
  getTitle: function () {
    return this.title;
  }
{% endhighlight %}

In your component:
{% highlight javascript %}
  componentWillMount: function () {
    store.addChangeListener(this.updateTitle);
  },
  updateTitle: function () {
    this.setState({
      title: store.getTitle()
    });
  },
  render: function () {
    return (<div>{this.state.title}</div>);
  }
{% endhighlight %}

Now, whenever the title changes it will be reflected in the component. That is all great, but what if we are not going to reflect a state in the UI, but react to the change of a state. An example of that would be toggling a duration.

In your store:
{% highlight javascript %}
  play: function () {
    this.isPlaying = true;
    this.emit('change');
    this.emit('play');
  },
  stop: function () {
    this.isPlaying = false;
    this.emit('change');
    this.emit('stop'); 
  },
  isPlaying: function () {
    return this.isPlaying;
  }
{% endhighlight %}

In your component:
{% highlight javascript %}
  componentWillMount: function () {
    store.addChangeListener(this.update);
    store.addListener('play', this.startDuration);
    store.addListener('stop', this.stopDuration);
  },
  startDuration: function () {
    this.interval = setInterval(function () {
      this.setState({
        duration: this.state.duration++
      });
    }.bind(this), 1000);
  },
  stopDuration: function () {
    clearInterval(this.interval);
  },
  update: function () {
    this.setState({
      buttonClass: store.isPlaying() ? 'red' : 'green'
    });
  },
  render: function () {
    return (<div><button className={this.state.buttonClass}>Toggle</button> {this.state.duration}</div>);
  }
{% endhighlight %}

It would not make sense to handle the starting and stopping of the duration with a change event, because then the component itself would have to know about the previous "isPlaying" state to know if it actually changed.

#### What does the dispatcher actually do?
What I also realized is that the dispatcher of the FLUX architecture does not really do much. Also the principle of calling all stores, no matter what action is dispatched did not quite make sense to me either. The important thing is the actions and since we are already using events, why not use events on them too? The concept is that you create a list of possible actions for your application and you trigger them by calling them, and passing optional arguments. The stores will then listen to actions directly and run methods when actions are triggered. It looks something like this in jFlux:

{% highlight javascript %}
  var actions = {
    'addTodo': createAction()
  };
  
  var myStore = createStore(function () {
    
    this.addTodo = function (title) {
      // Do something
    };
    
    this.listenTo(actions.addTodo, this.addTodo);
    
  });
  
  var myComponent = createComponent({
    buttonClick: function () {
      actions.addTodo(this.state.title);
    }
  });
{% endhighlight %}

As you can see we call the action from the component, we listen to the action from the store and that is how things connect. To me it just makes more sense.

### Bringing it together
So to build a React JS application with these principles I have deviced a little library called [flux-react](https://github.com/christianalfoni/flux-react). The initial version was based on an early article I wrote about FLUX and React JS, but as time passes and you get more experience, the tools change. So this is the new version.

#### Actions
So actions are what links your components (and anything else that wants to change state) with your stores. You define them just by naming them. When calling an action the arguments will be deepCloned to avoid later mutation of complex objects passed to the store.

{% highlight javascript %}
var flux = require('flux-react');
var actions = flux.createActions([
  'addTodo',
  'removeTodo',
  'toggleTodo',
  'updateTodo'
]);
module.exports = actions;
{% endhighlight %}


#### Stores
Stores are where you define your application state, update it and notify components about changes.

{% highlight javascript %}
var flux = require('flux-react');
var actions = require('./actions.js');
var TodosStore = flux.createStore({

  // We put the state directly on the store object
  todos: [],
  
  // Then we point to the actions we want to react to in this store
  actions: [
     actions.addTodo
  ],
  
  // The action maps directly to a method. So action addTodo maps to the
  // method addTodo()
  addTodo: function (title) {
    this.todos.push({title: title, completed: false});
    this.emitChange();
  },
  
  // The methods that components can use to get state information
  // from the store. The context of the methods is the store itself. 
  // The returned values are deepcloned, which means
  // that the state of the store is immutable
  exports: {
    getTodos: function () {
      return this.todos;
    }
  }
});
module.exports = TodosStore;
{% endhighlight %}

So lets just see how this is used in a component before going over the concept:

{% highlight javascript %}
var React = require('react');
var store = require('./TodosStore.js');

var MyComponent = React.createClass({
  getInitialState: function () {
    return {
      todos: store.getTodos()
    };
  },
  componentWillMount: function () {
    store.addChangeListener(this.update);
  },
  componentWillUnmount: function () {
    store.removeChangeListener(this.update);
  },
  update: function () {
    this.setState({
      todos: store.getTodos()
    });
  },
  render: function () {
    return (
      <div>Number of todos is: {this.state.todos.length}</div>
    );
  }
});
module.exports = MyComponent;
{% endhighlight %}

Okay. So what I noticed building JSFridge was that my actions always mapped directly to a method. If the action was called "addTodo", the method handling that action in my store was also called "addTodo". That is why actions in "flux-react" map directly to a method.

An other concept is the "exports". You only want to expose a set of getter methods that your components can use. Three things happens with exports:

1. The exports object is the object that is returned when creating a store
2. All methods in exports are bound to the store, letting you use "this" to point to the state in the store
3. Exported values are automatically deep cloned

Now that last part needs a bit more explenation. The state, in your store should only be mutated (changed) inside the store. F.ex. returning a list of todos to a component should not allow that component to do changes there that is reflected in the store. This is because of debugging. If lots of different components starts to change state directly in your store you will get into trouble. So instead the "flux-react" store makes sure that any value returned from exports is deep cloned, right out of the box. The same goes for complex objects passed as arguments to an action. We do not want them to be changed later outside of the store and by doing so change the state of the store.

What about performance? Well, the thing is that values you return from a store are not big, neither are values you pass as arguments. Yes, maybe you have a phonebook of 10.000 people in your store, but your interface will never show all 10.000 of them. You may grab the 50 first of them and the rest requires searching or pagination. In that case it is only the search result, or next page, that is deep cloned. Never all the 10.000 people.

##### Mixing it up
Another feature implemented in the "flux-react" stores is mixins. This points back to my experience of building stores that depend on each other. Instead of creating separate stores and risk circular reference you create mixins instead. Lets check out an example where we use a mixin:

{% highlight javascript %}
// FILE 1: My remove mixin
var actions = require('./actions.js');
var RemoveMixin = {
  actions: [
    actions.removeTodo
  ],
  removeTodo: function (index) {
    this.todos.splice(index, 1);
    this.emitChange();
  }
};
module.exports = RemoveMixin;

// FILE 2
var flux = require('flux-react');
var actions = require('./actions.js');
var RemoveMixin = require('./RemoveMixin.js');

var TodosStore = flux.createStore({
  todos: [],
  mixins: [RemoveMixin],
  actions: [
    actions.addTodo
  ],
  addTodo: function (title) {
    this.todos.push({title: title, completed: false});
    this.emitChange();
  },
  exports: {
    getTodos: function () {
      return this.todos;
    }
  }
});
module.exports = TodosStore;
{% endhighlight %}

As you can see you just create an object, just like a React JS mixin, and bring it into your main store file. The properties values that merge are: **actions**, **getInitialState**, **exports** and **mixins**. Any other methods defined will also be merged into the store. This will keep you sane and prevent you from hitting some kinda of dependency problem.

What I also found to be useful on JSFridge was putting all states in one mixin. That gives you a lookup of all states. The same for exports. Let me show you what I mean:

{% highlight javascript %}
// FILE 1: All states are listed here
var StatesMixin = {
  foo: 'bar',
  moreState: {},
  listState: [],
  isDoingSomething: true
};

// FILE 2: Example handler
var Handler1Mixin = {
  actions: [
    actions.addToList
  ],
  addToList: function (item) {
    this.listState.push(item);
    this.emitChange()
  }
};

// FILE 3: All exports are defined here, along with bringing in the mixins
var MyMainStore = flux.createStore({
  mixins: [StatesMixin, Handler1Mixin, Handler2Mixin, Handler3Mixin],
  exports: {
    getFoo: function () {
      return this.foo;
    },
    getList: function () {
      return this.listState;
    }
  }
});
{% endhighlight %}

So any of the handler mixins contains actual actions and handlers for those actions. That is a sweet setup and now you can see how easy it is to say: "My application has 35 application states".

### Summing it up
I was not trying to point out that Facebook Flux with its dispatcher and stores are the wrong way to go. I just wanted to share my experience using flux architecture and what I needed to keep me sane. If you want to check out [flux-react](https://github.com/christianalfoni/flux-react), please do. If not, I hope this article gave you a couple of valuable points. Thanks for reading!