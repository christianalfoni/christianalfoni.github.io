---
layout: post
title:  "React JS and FLUX"
date:   2014-08-20 09:21:30
categories: javascript
tags: ["javascript"]
---

I am not going to take up your time explaining FLUX in details, that is already very well done on the [Facebook flux site](http://facebook.github.io/flux/). What I want to tell you about is why you would want to consider the flux architecture with React JS
as your tool to build an interface and handle interaction and updates to that interface.

### Its all about state
Just to be sure that we are on the same page, state can be explained as simple as a checkbox. When the checkbox is checked, the state is different than if it is unchecked. What you want to call these states does not matter. Maybe you call it `checked:true` or `state:'on'`, it does not matter, what matters is the way this state is communicated to and from the interface.

### A little story
I was working on a huge project, building a switchboard application on the web. It was early on in my career at [Marcello](http://www.marcello.no), and things like build tools and Backbone was at its very early stage, Angular did not exist. An implementation of this project was to display the calls of the switchboard user in the browser. The calls were event based, meaning that we did not poll for the "call-state", but we got individual events from the server stating changes in the calls. 

A long story short, the implementation basically translated those events into DOM operations. Each event type had some logic trying to figure out the correct DOM operation. Now, this might not seem like such a bad idea, but just the call interface itself had like 20-30 states. Everything from alerting call, hanging up, idle, transferring, hold, call 1 hold and call 2 active etc. This got totally out of control. There was a lot of bugs.

At a later point I got the opportunity to re-implement this solution. By that time Backbone was popular and we gave it a try. Instead of passing single events from the backend, we changed the architecture to pass the complete state of the calls. Those calls were translated to models in a collection and then passed to the view. Every time the collection of calls changed, interface was completely rerendered. No more bugs.

There are two points to this story:

1. Having the complete state before rendering is a lot easier than trying to build up the correct state based on bits of data

2. The concept of rerender is a very good thing, it makes it a lot easier to handle application logic. That said, it has some problems, but maybe there is a way to get the best of both worlds

### Backbone
{% highlight javascript %}
var model = Backbone.Model.extend({
  notify: false
});
var View = Backbone.View.extend({
  events: {
    'click input': 'updateUser',
    template: handlebars.compile('<input type="checkbox" {#if notify}checked{/if}/> Notify'),
    model: model,
    initialize: function () {
      this.listenTo(this.model, 'change', this.render);
    },
    render: function () {
      this.$el.html(this.template(this.model.toJSON()));
      return this;
    },
    updateUser: function () {
      this.model.set('notify', this.$el.find('input').is(':checked'));
    }
  }
});
{% endhighlight %}
In this scenario we pass a model to the view defined. The view has a template that in this case uses handlebars. When the view is initialized it listens for changes on the model. If any changes occur it will rerender its contents. If the user clicks the input the view will run an **updateUser** method that updates the model. When the model is updated the 'change' event is triggered and the view rerenders.

#### The good
Backbone makes it easy to keep your views up to date compared to vanilla javascript. Instead of updating specific elements in the DOM, you just rerender a section of the page related to a model. Any interaction coming back from the view goes straight to the model, updates it and in turn updates the view. Specifically you do not need to update both the view and the model. The model is the "master".

#### The not so good
Backbone rerenders the complete view. Yeah, it is a bad thing too. The reason is that DOM updates is the slowest part in the browser and it should be at a minimal. The other thing is that a full rerender of the view might break interaction. Maybe the user was typing something in an input etc. 

There is also a problem of scaling. Giving the view that much direct control of your model might become a problem. Maybe changing *notify* would affect some other part of the application. You would end up with multiple listeners to the model, each doing something to the state of your application. This could potentially be very hard to manage.

### In Angular
{% highlight javascript %}
.factory('UserService', function () {
  var user = {};
  return {
    getUser: function () {
      return user;
    }
  }
})
.controller('MyCtrl', function ($scope, UserService) {
  $scope.user = UserService.getUser();
});
{% endhighlight %}

{% highlight html %}
<div ng-controller="MyCtrl">
  <input type="checkbox" ng-model="user.notify"/>
</div>
{% endhighlight %}
Looking at Angular we instantly see how powerful two way databinding is. We define a service, which could be looked at as our model for the user. This user object is attached to a scope property called user. To reflect the state of the notify property on the checkbox we just have to "bind" it. Whenever the checkbox changes, the notify property changes. It is really quite beautiful. 

#### The good
Angulars two way databinding reduces the amount of code you need to write. You are very productive and personally I love Angular for prototyping. There is also a very loose concept of a model. A model can be any object with any kind of behaviour. This makes it a lot easier to build different types of models with their own specific behaviour, instead of a general Model concept.

#### The not so good
Even though we write less code we get the same problem as with Backbone, actually it gets worse. In bigger applications multiple components might want to know about changes to a property. Though two way databinding is powerful, there is no handler for setting the property, you instantly mutate your model. You would have to trigger events or manually watch the property to know about changes. That gives the same problem as Backbone, handling different parts of your application reacting on specific state changes.

Angulars two way databinding is based on "dirty checking". There is a whole lot going on in Angulars digest cycle and it is difficult to identify when this digest cycle runs. This can potentially result in slower applications and errors when you need to manually trigger the digest cycle with $apply.

### FLUX
Now, FLUX is not a framework, it is an architecture that uses React JS as its "controller - view" layer. The good thing about this architecture, comparing it to MV\*, is that it is a very basic concept that scales endlessly. There are no MVVC, MVC, MVMCMVMCM... yeah, the last one was a pun... I love the Angular team calling it MV-whatever. It does not matter what you call it, what matters is that you have a simple to understand and scalable architecture. The FLUX architecture, in my opinion, beats MV* at that. So lets looks more deeply into that statement. These are the three main concepts that builds up FLUX:

* Dispatcher
* Stores
* Components

Though these concepts does not translate directly to the examples above, there are some similarities. A model represents what we call a Store in FLUX. What seperates the Store in FLUX from a traditional model though is that it is part of a one way flow. In Backbone and Angular the interface updates are based on the model, and the interface updates right back. That is not the case with FLUX.

In FLUX the interface is not able to update the stores. They can pass an "intent", also called action, to the dispatcher and the dispatcher notifies all the stores about this intent. So the point here is that the stores holds all the application state and when a user interaction occurs, a server request is responded etc., they are all passed through the dispatcher as an "intent" that notifies all the stores.

When a store has updated their state, a "change" event is emitted which any component can listen to. A component receiving this 'change' event will rerender. This is the same principle as Backbone, BUT there is one huge difference. A component does not rerender all its content, it will only update the parts of the DOM that it needs to. This is done by React JS own virtual DOM and a difference alghoritm that results in DOM operations. It is really quite brilliant.

So the flow is: **DISPATCHER -> STORES -> COMPONENTS**. If a component wants to change state they have to send an intent to the dispatcher. In MVC, you often have: **MODEL <-> CONTROLLER <-> VIEW**, state is changed in both directions. So think of double digits of models, controllers and views, all going in both directions, crossing each other. It gets problematic! In FLUX though, it does not matter how complex your application gets, the flow is the same all over. And that is exactly what makes FLUX easy to work with.

### Looking at the code
There is no official dispatcher and store library for flux, they are just concepts. So in this example I will use a FLUX library I built that has its own dispatcher and store concept, extending the React library itself. You can get more information on that here: [flux-react](https://github.com/christianalfoni/flux-react).
{% highlight javascript %}
/* ==== main.js ==== */
/** @jsx React.DOM */
var React = require('flux-react');
React.renderComponent(<Checkbox/>, document.body);

/* ==== UserStore.js ==== */
var React = require('flux-react');
var user = {
  notify: false
};

// We return an object that we will pass when
// building the store
module.exports = React.createStore({
  getNotify: function () {
    return user.notify;
  },
  
  // dispatch runs whenever a new action is received
  // from the dispatcher
  dispatch: function (payload) {
    switch (payload.type) {
      case 'CHANGE_NOTIFY':
        user.notify = payload.notify;
        this.flush(); // Give notice that changes has been done
        break;
    }
  }
});

/* ==== Checkbox.js ==== */
/** @jsx React.DOM */
var React = require('flux-react');
var UserStore = require('./UserStore.js');
module.exports = React.createClass({

  // What stores the component is dependant of
  stores: [UserStore], 
  
  getInitialState: function () {
    return {
      notify: UserStore.getNotify() 
    };
  },
  
  // A react-flux callback that triggers when an action
  // is dispatched and the stores have flushed
  storesDidUpdate: function () {
    this.setState({
      notify: UserStore.getNotify()
    });
  },
  notify: function () {
    React.dispatch({
      type: 'CHANGE_NOTIFY',
      notify: this.refs.checkbox.getDOMNode().checked
    });
  },
  render: function () {
    return (
      <input ref="checkbox" type="checkbox" checked={this.state.notify} onChange={this.notify}/>
    )
  }
});
{% endhighlight %}
Wow, that was a bit more code! Well, try to think of it like this. In the above examples, if we were to do any changes to the application we would probably have to move things around. In the FLUX example we have considered that from the start. Any changes to the application is adding, not moving things around. If you need a new store, just add it and make components dependant of it. If you need more views, create a component and use it inside any other component without affecting their current "parent controller or models".

React JS and FLUX is still very new and I still need to try this out on a large application, but the concepts make sense. Hopefully this will help you get going with React JS and its FLUX architecture. Thanks for hearing me out!