---
layout: post
title:  "Think twice about ES6 classes"
date:   2015-01-01 09:21:30
categories: javascript
tags: ["javascript"]
---
I got into JavaScript about 5 years ago and I have no to very little experience with other languages. That can sometimes be a bad thing, but it can also be a very good thing. In this article I am going to talk about classes, which is coming to the next version of JavaScript. I am going to talk about why I do not understand developers simulating classes in JavaScript and why I think it is not a very good idea to bring the concept into JavaScript at all. I will do a step by step comparison of how you build with classes in ES6 and how you would use vanilla JavaScript to make a more powerful construct. At the end of the article we will look at how you can use the power of composition and delegation to bring a lot more power than what classes could ever do. Hopefully this will explain why I do not think classes is a good idea. I will be using the new ES6 syntax throughout the examples.

> *"When asked what he might do differently if he had to rewrite Java from scratch, James Gosling suggested that he might [do away with class inheritance](http://www.javaworld.com/article/2073649/core-java/why-extends-is-evil.html) and write a [delegation only](http://www.artima.com/intv/gosling34.html) language."* [delegation vs inheritance](http://javascriptweblog.wordpress.com/2010/12/22/delegation-vs-inheritance-in-javascript/)

If you want to know more on this topic please head over to Kyle Simpsons article on [http://davidwalsh.name/javascript-objects](http://davidwalsh.name/javascript-objects) and Eric Elliots talk on Fluent 2014 [https://www.youtube.com/watch?v=lKCCZTUx0sI](https://www.youtube.com/watch?v=lKCCZTUx0sI). The first part of this article will bring up different concepts related to objects in JavaScript, the second part will show how you can use a more powerful construct to build objects.

### The freedom of JavaScript
So the one thing I love about JavaScript is its freedom. You do not have to tell your program what everything is, either by types or classes, you just write the code. Examples of this would be:

{% highlight javascript %}
var obj = {
  foo: 'bar'
}; // I just created an object
var num = 13; // I just created a number
var string = 'foo'; // I just created a string
{% endhighlight %}

I did not have to tell my application what my object should look like before I create it, or that I was going to create a number or a string. I just do it. At this point it might not make much sense why I bring it up, but let us start looking into object creation and keep this in mind.

### Creating an object
So in the previous example we just create an object, but in any application you would probably like to have more than one instance of that object. So you need some kind of construct to create instances of objects that looks and behaves the same. In traditional JavaScript you do that with either a constructor or you can do it with a normal function. With ES6 you get the ability to define a class:

{% highlight javascript %}
// With a constructor
function MyObjectA () {};
new MyObjectA(); // {}

// With a function
function MyObjectB () {
  return {};
};
MyObjectB(); // {}

// ES6 class
class MyObjectC {}
new MyObjectC(); // {}
{% endhighlight %}

Now all these snippets of code gave the same result, but they work a bit differently. First we will look at traditional JavaScript and then we will start comparing that with the new ES6 class syntax.

### The constructor
So why do we use a constructor? As stated above it is a construct to instantiate multiple instances of objects that looks and behaves the same. Note here that there are no restrictions to the instantiated object, you can manipulate it however you want after it has been created. An object instantiated by a constructor or a normal function does not differ in that sense. Where it differs though is a property on the constructor function called "prototype". Now, you have probably heard about "prototypal inheritance" and we will take a look into that now. Let us compare again by adding a method to the object:

{% highlight javascript %}
// With a constructor
function MyObjectA () {
  this.myMethod = function () {};
};
new MyObjectA(); // {myMethod: function () {}}

// With a function
function MyObjectB () {
  return {
    myMethod: function () {}
  };
};
MyObjectB(); // {myMethod: function () {}}
{% endhighlight %}

Now, these instantiators will create a new myMethod method for each object instance. Unless you have to create hundreds of thousands of objects, adding a new version of the method for each instance will **not** get you into performance issues. I believe you should stick with describing your object as it will look like when instantiated, not create less comprehensable code for performance sake. That said, there are situations where you want to use **delegation**.

When I say delegation I specifically mean that we link one object to an other. The object linked to the object you are instantiating can act on behalf of it. You do this specifically by adding some object on the prototype property of a constructor function. When using the normal function pattern we use the `Object.create` method. It has the same effect:

{% highlight javascript %}
// With a constructor
function MyObjectA () {}
MyObjectA.prototype = {
  myMethod: function () {}
};

var obj = new MyObjectA(); // {}
obj.myMethod(); // Prototype object acts on behalf of obj

// With a function
var proto = {
  myMethod: function () {}
};
function MyObjectB () {
  return Object.create(proto);
}

var obj = MyObjectB(); // {}
obj.myMethod(); // Prototype object acts on behalf of obj

// ES6 class
class MyObjectC {
  myMethod () {
  
  }
}
var obj = new MyObjectC(); // {}
obj.myMethod(); // Prototype object acts on behalf of obj
{% endhighlight %}

The term **prototypal inheritance** is a bit misleading. Inheritance suggests that it is the instantiated object the somehow receives the look and behavior of parent objects on the prototype chain, but actually it is the other way around. A prototype object acts as a delegate, an object that can act on behalf of the instantiated object. 

Try to think of this like event delegation in the DOM. Lets say you have a list with items. Instead of registering a click on each item in the list, you register the click on the list itself. When a click occurs on an item it will try to trigger click listeners, but there are none, so the click moves up to the list element. Now it tries to trigger click listeners on the list element and there it is.

{% highlight console %}
DOM delegation

|----------------|
| |------------| |
| |   Item 1   | | 1. Clicking on item
| |------------| |
|                | 2. The list acts upon the click on behalf of the item (delegation)
| |------------| |
| |   Item 2   | |  
| |------------| |
|----------------|

{% endhighlight %}

Now think about an instance of our MyObjectA here. When you run `new MyObjectA().myMethod()` it will try to find the method on the instance, but it will not find it. Then it checks the object linked to the instance and there it finds it. There you go, delegation, not inheritance.
 
{% highlight javascript %}
// OBJECT delegation
var objectA = Object.create(); // Object.create() creates a plain object
objectA // {}

var proto = {doSomething: function () {}};
var objectB = Object.create(proto);
var objectC = Object.create(proto);

objectB // {}
objectC // {}
objectB === objectC // false

// The proto object acts on the method call on behalf of objectB (delegation)
objectB.doSomething();

// The same proto object acts on the method call on behalf of objectC (delegation)
objectC.doSomething();

{% endhighlight %}

So this is quite powerful. Lets say we wanted to use the methods from EventEmitter in combination with our own. We could do this:

{% highlight javascript %}
// With a constructor
function MyObjectA () {
  this.myMethod = function () {};
}
MyObjectA.prototype = EventEmitter.prototype;

// With a function
function MyObjectB () {
  var object = Object.create(EventEmitter.prototype);
  object.myMethod = function () {};
  return object;
}

// ES6 class
class MyObjectC extends EventEmitter {
  myMehtod () {}
}
{% endhighlight %}

Every instance is linked to the EventEmitter.prototype object. It is a delegate, an object acting on the behalf of the instantiated object. At the same time we create a new myMethod method for each instance, except ES6 which forces all methods on a linked prototype object. We could actually have made everything unique to each instance by doing this:

{% highlight javascript %}
// With a constructor
function MyObjectA () {
   Object.assign(this, EventEmitter.prototype, {myMethod: function () {}}); 
}

// With a function
function MyObjectB () {
  return Object.assign({}, EventEmitter.prototype, {myMethod: function () {}});
}

// With ES6 class
class MyObjectC {
  constructor () {
    Object.assign(this, EventEmitter.prototype, {myMethod: function () {}});
  }
}
{% endhighlight %}

In this case we are not using delegation, because we actually do create new properties on each instance. Though this increases memory usage and takes a performance hit in its instantiation (copying all properties) it is not as bad as you would think. Every function, object and array are copied by reference. So even though each instance will have their own "on" and "off" property, the actual function they reference is the same on all instances. What is very powerful here though is that we are starting to see composition in action. You could keep adding objects as arguments to Object.assign. In ES6 classes, using extend, you can only extend one delegate.

So now we have been playing around with the concept of delegation, and pointed out that it is not really inheritance. You should also have a good idea of why we use the constructor and the power of delegation and that we can achieve the same effect with just a simple function.

### ES6 classes vs object factory
Lets us now go through the class syntax in ES6 and see how it compares to this simple function that creates objects. I will call the simple function pattern an "Object factory".

#### Instantiate an object
{% highlight javascript %}
// ES6 Class
class MyObjectA {}
new MyObjectA(); // {}

// Object factory
function MyObjectB (obj = {}) {
  return obj; // Return passed object or a new object
}
MyObjectB() // {}
MyObjectB({foo: 'bar'}) // {foo: "bar"}
{% endhighlight %}

So we already see the advantages of the object factory. First of all we do not have to worry about the "new" keyword. We also see that we can pass an existing object if we want to, or just let it create a clean instance.

#### Instantiation function (constructor)
{% highlight javascript %}
// ES6 Class
class MyObjectA {
  constructor () {
    this.list = [];
  }
}
new MyObjectA().list === new MyObjectA().list; // false

// Object factory
function MyObjectB (obj = {}) {
  obj.list = [];
  return obj;
}
MyObjectB().list === MyObjectB().list // false
{% endhighlight %}

As you can see in the object factory pattern the function itself is the instantiating function, there is no constructor function that implicitely gets called whenever you use the new keyword in front, and call the class as a function.

#### Privates
A fantastic feature in JavaScript is closures and you can use those to hide implementation details in your objects. Using the ES6 class syntax this is actually not possible, so this is what you would have to do:

{% highlight javascript %}
// Class
let MyObjectA = (function () {

  let myPrivate = 'foo';

  class MyObjectA {
    getPrivate () {
      return myPrivate;
    }
  }
  
  return MyObjectA;

}());
new MyObjectA().getPrivate(); // "foo"

// Object factory
function MyObjectB (obj = {}) {
  let myPrivate = 'foo';
  obj.getPrivate = function () {
    return myPrivate;
  };
  return obj;
}
MyObjectB().getPrivate(); // "foo"
{% endhighlight %}

Sorry about the bad example, but I hope you see that I am just making a point :-) As you can see you actually need to create the class with its own outer scope, in this case an IFFI (Immediately Invoked Function Expression). In the object factory pattern you can just define and use it.

#### Adding methods
{% highlight javascript %}
// ES6 Class
class MyObjectA () {
  constructor () {
    this.list = ['foo'];
  }
  getFirstInList () {
    return this.list[0];
  }
}
new MyObjectA().getFirstInList() // "foo"

// Object factory
function MyObjectB (obj = {}) {
  obj.list = ['foo'];
  obj.getFirstInList = function () {
    return this.list[0];
  };
  return obj;
}
MyObjectB().getFirstInList() // "foo"
{% endhighlight %}

There is not much difference in syntax, but the methods added in ES6 classes will not be attached to the instance, but a prototype object created "under the hood", causing delegation. There is a hidden feature of the object factory pattern though. When using `this` you actually make the method dynamic. What I mean is that you can change the context of the method when it runs:

{% highlight javascript %}
function MyObjectB (obj = {}) {
  obj.list = ['foo'];
  obj.getFirstInList = function () {
    return this.list[0];
  };
  return obj;
}
var someObj = {list: ['bar']};
MyObjectB().getFirstInList.call(someObj); // "bar"
{% endhighlight %}

Extending a function call with `call` or `apply` will set the `this` of the function to whatever you pass as the first argument, in this case `someObj`. But we can actually lock the context of the methods using the object factory, making them more predictable:

{% highlight javascript %}
function MyObjectB (obj = {}) {
  obj.list = ['foo'];
  obj.getFirstInList = function () {
    return obj.list[0];
  };
  return obj;
}
var someObj = {list: ['bar']};
MyObjectB().getFirstInList.call(someObj); // "foo"
{% endhighlight %}

Since we point to the actual object being built it is not possible to change the context of which it is run.

#### Inheritance
So a class in ES6 has the possibility to extend from an other class, but not only a specific "class" definition, but actually any constructor and its prototype. It also gets something brand new in JavaScript. A function, called **super**, that knows what method it is being called in and calls that same method on the extended "class" with the current context. It is not easy to learn JavaScript with such a function, I must say!

The challenge with thinking inheritance is that you have to compose in your head how an object actually looks like by looking into different classes AND follow the usage of super, it can get very complex. The classes also become completely dependant on each other because you do not compose smaller parts, only build on top of existing functionality. [inheritance is evil and must be destroyed](http://berniesumption.com/software/inheritance-is-evil-and-must-be-destroyed/).

With an object factory it is not encouraged to think inheritance at all, but composing. Think that you already have an object and you add behavior to it, rather than describing behavior that produces an object. Let us create an example with a Backbone Model.

{% highlight javascript %}
// Class
class MyModel extend Backbone.Model {
  constructor (options) {
    super(options.attributes);
  }
}

// Object factory
function MyModel (model = {}) {
  model = Object.assign(Object.create(Backbone.Model.prototype), model);
  Backbone.Model.call(model, model.attributes);
  return model;
}
{% endhighlight %}

So in this example the ES6 syntax of course wins the prize. What worries me a great deal as that with such simple, but restricted syntax, you risk loosing some of the best features of JavaScript. So I created a tiny lib that will give you some sugar on top to get the full benefit of composing and delegation. Syntax should really not be the winning argument nevertheless.

### Objectory (Object + factory)
So objectory allows you to compose objects while still taking advantage of delegation. Please let me show you with some examples:

#### Instantiating
{% highlight javascript %}
var MyObject = objectory(function (obj) {
  obj.foo = 'bar';
  obj.getFoo = function () {
    return obj.foo; // Forced usage of obj
  };
});

MyObject(); // Extending default empty object: {foo: 'bar'}
MyObject({name: 'Roger'}); // Extending passed object: {name: "Roger", foo: "bar"}
{% endhighlight %}

So as you might expect this looks much like a normal constructor, but it has some benefits:

1. You can pass your own object to extend, instead of the default
2. You can use forced context on the methods
3. You do not have to use the "new" keyword or return the object created form the constructor

#### Composing objects
So lets dive in to the power of objectory and JavaScript. In the example above everything added to "obj" would become instance properties. But lets see what happens when we start to compose.

{% highlight javascript %}
var DEFAULTS = {
  name: 'no name',
  age: 18
};
var Person = objectory(function (person) {
  person.compose(DEFAULTS);
});

var personA = Person(); // {}
personA.name; // "no name"
personA.age; // 18

var personB = Person({name: 'Roger', age: 30}); // {name: "Roger", age: 30}
{% endhighlight %}

As you can see, by composing a normal object with a person object we created a delegate. We linked the DEFAULTS object to any instantiated person object.

#### Composing constructors and their prototypes
Most libraries are based on constructors with prototypes. An example of that would be Backbone.Model or EventEmitter. Let us turn our Person into a Backbone Model.

{% highlight javascript %}
var Person = objectory(function (person) {
  person.compose(Backbone.Model, {
    name: 'no name',
    age: 18
  });
});

var personA = Person(); // {_events: {...}, attributes: {...}...}
personA.get('name'); // "no name"
{% endhighlight %}

In this example the prototype property from "Backbone.Model" (Backbone.Model.prototype) will be linked to the "Person" objects created. When the Person constructor runs it will run the "Backbone.Model" constructor and pass in an object representing attributes of the model. 

This example also shows an advantage over Backbones own "extend" method. If any attributes passed was objects or arrays they would all be unique for each instance of Person. That would not be the case and is often confusing with Backbone.

But we can also take advantage of our defaults and do this:

{% highlight javascript %}
var DEFAULTS = {
  attributes: {
    name: 'no name'
  }
};
var Person = objectory(function (person) {
  person.compose(DEFAULTS);
  person.compose(Backbone.Model, person.attributes);
});

var personA = Person(); // {attributes: {name: "no name"}...}
personA.get('name'); // "no name"

var personB = Person({attributes: {name: 'Roger'}}); // {attributes: {name: "Roger"}...}
personB.get('name'); // "Roger"
{% endhighlight %}

In this example the delegate for Person will consist of the "DEFAULTS" object AND the prototype of "Backbone.Model" (Backbone.Model.prototype). When the "Person" constructor runs it will call the "Backbone.Model" constructor and pass in the attributes. Either by delegation from DEFAULTS or any passed attributes.

And now lets take a look at something really awesome. Lets say we wanted to use EventEmitter instead of the default event system in Backbone.

{% highlight javascript %}
var Person = objectory(function (person) {
  person.compose(Backbone.Model, person.attributes);
  person.compose(EventEmitter);
});

var personA = Person(); // {attributes: {name: "no name"}...}
personA.on('change', function () {}); // works
{% endhighlight %}

This of course requires that the API of EventEmitter works the same way as Backbone, but you start to see what is available to you.

#### Composing other factories
You can of course also compose other factories.

{% highlight javascript %}
var Person = objectory(function (person) {
  person.age = 18;
});
var Student = objectory(function (student) {
  student.compose(Person);
  student.grade = 'A';
});

var student = Student(); // {grade: "A", age: 18}
{% endhighlight %}

In this case Person does not have any compositions, which means there is no delegation either. Lets check that:

{% highlight javascript %}
var DEFAULTS = {
  name: 'no name',
  age: 18
};
var Person = objectory(function (person) {
  person.compose(DEFAULTS);
});
var Student = objectory(function (student) {
  student.compose(Person);
  student.grade = 'A';
});

var student = Student(); // {grade: "A"}
student.name; // "no name"
student.age; // 18
{% endhighlight %}

### Summary
If you go to [objectory repo](https://github.com/christianalfoni/objectory) there are more examples, some perfomance and compatability info. I hope this gave you enough input to see how it is possible to use vanilla JavaScript with delegation and composition to create a construct more powerful than a class could ever hope to be!