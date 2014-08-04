---
layout: post
title:  "Why javascript inheritance is so difficult to understand"
date:   2014-08-01 09:21:30
categories: javascript
tags: ["javascript"]
---

### Background
You have probably heard the term "prototypal inheritance", "constructor function", "prototype" and that JavaScript does not really have classes. I am going to try to give it my best to explain these concepts and end this article by showing a tiny library that not only helps you structure your code, but also makes sure that the objects that you produce, reflects the way JavaScript does it natively for its own objects. If you have deep understanding of JavaScript my explanation might be overkill, but I hope you take the time to look through and check out the small lib at the end.

### The concept
Now, as you have probably heard, "everything in JavaScript is an object". That basically being true, and also being quite easy to comprehend, we could also say: "All objects in JavaScript inherits from an instance of Object". Though not being that easy to comprehend, it has more truthyness to it and it says more about how JavaScript works. But what is this Object, and what is inheritance? Lets check out Object first:

{% highlight javascript %}
  console.log(Object); // => function Object() { [native code] } 
{% endhighlight %}

Object is a function? Actually, it is a **constructor**, it is a function that, with the help of the **new** keyword, creates a unique instance of an object. So you could say that **Object** instantiates an **object**. The same goes for everything in JavaScript. A **String** instantiates a **string object**, a **Function** instantiates a **function object** and an **Array** instantiates an **array object**.

{% highlight javascript %}
  Object // => function Object() { [native code] } 
  Array // => function Array() { [native code] }
  Function // => function Function() { [native code] } 
{% endhighlight %}

Putting these side by side, we get something like this:

{% highlight javascript %}
  Object // => function Object() { [native code] }
  new Object()); // => {} 
  
  Array // => function Array() { [native code] }
  new Array() // => []
  
  Function // => function Function() { [native code] }
  new Function() // => function anonymous () {}
{% endhighlight %}

As you can see, calling a **constructor** with the **new** keyword creates a result, an object. Now, lets investigate these objects, see if they leave some trace of where they came from.

{% highlight javascript %}
  var array = new Array();
  array.constructor // => function Array() { [native code] }
  array.__proto__ === Array.prototype // => true
  array.__proto__ === array.constructor.prototype // => true
  
  var object = new Object();
  object.constructor // => function Object() { [native code] }
  object.__proto__ === Object.prototype // => true
  object.__proto__ === object.constructor.prototype // => true
{% endhighlight %}

Okay, as we can see here, all objects has a **constructor** property that points back to what **constructor** created them. We also see that this strange **\_\_proto\_\_** property on the object is the same as the **prototype** property on its **constructor**. So we are starting to see a pattern here that we can resolve to:

1. new Object()
2. {}
3. {}.constructor === Object
4. {}.\_\_proto\_\_ === Object.prototype

So the object created knows about its **constructor**. The object created is also linked to a **prototype**, which the **constructor** supplied. To summarize, we are working on three levels:

1. Prototype: An instance of an object defined by a prototype constructor
2. Constructor: Creates an instance of an object and decorates it with unique property values. Then it connects the prototype object to it
3. Instantiated object: Inherits properties from the prototype and gets unique properties from the constructor

I think we need to look at an example:

{% highlight javascript %}
  // We create our own constructor
  function MyConstructor () { 
  
  }
  new MyConstructor(); // => MyConstructor {}
{% endhighlight %}

Now let us first start off by creating a unique property for each instance. As we see on the list above, that is a job for the constructor.

{% highlight javascript %}
  function MyConstructor () { 
    this.list = [];
  }
  var myConstr1 = new MyConstructor(); // => MyConstructor { list: [] }
  var myConstr2 = new MyConstructor(); // => MyConstructor { list: [] }
  myConstr1.list === myConstr2.list // => false
{% endhighlight %}

As we can see here, creating new instances gives the same object, though the property values are unique. Let us see what happens if we put this property on the prototype instead:

{% highlight javascript %}
  function MyConstructor () { 
 
  }
  
  // Using new Object() to Illustrate that we are actually instantiating a new object,
  // {} is just a "shortcut"
  MyConstructor.prototype = new Object({ 
    list: []
  });
  var myConstr1 = new MyConstructor(); // => MyConstructor {  }
  var myConstr2 = new MyConstructor(); // => MyConstructor {  }
  myConstr1.list === myConstr2.list // => true
  myConstr1.__proto__ // => Object { list: [] }
  myConstr1.__proto__ // => Object { list: [] }
{% endhighlight %}

We instantiated an Object object on the prototype of our constructor and gave it a list property with an array. This is what happened:

1. Prototype: An instance of Object that contains a property named "list" that has an array as its value
2. Constructor: Instantiates a MyConstructor object and connects the prototype object to the new instantiated object
3. Instantiated object: Inherits the list property value from the MyConstructor prototype object

Now that we have gotten a look at prototype and inheritance we can check the statement: "All objects in JavaScript inherits from an instance of Object". Lets take a look some native constructors:

{% highlight javascript %}
  var array = [];
  array.__proto__ // => Instance of an Array
  array.__proto__.__proto__ // => Object {}

  var function = function () {};
  function.__proto__ // => Instance of a Function
  function.__proto__.__proto__ // => Object {}
  
  var string = "test string";
  function.__proto__ // => Instance of a String
  function.__proto__.__proto__ // => Object {}
  
{% endhighlight %}

As we can see everything inherits from an instance of an Object. You might ask the question, why does an instance of an Array inherit from an instance of an Array? Or an instance of a Function from an instance of a Function? Actually it is not. F.ex. the instance of an Array is inhereting from an instance of an ArrayPrototype, not an other instance of an Array. The Array constructor is not the constructor that created the array on the prototype. The ArrayPrototype constructor is what actually defines a complete array, with all its methods etc. The Array constructor is just a constructor that decorates a new Array object and attaches the instance of the ArrayPrototype object to it. Actually, if your application only needed one array, you could have used Array.prototype everywhere in your app :-)

{% highlight javascript %}

var array = new Array();
array instanceof Array // => true
Array.prototype instanceof Array // => false

{% endhighlight %}

As you can see, the prototype instance is not from the same constructor as the array instance you create. But lets put this in the back of our heads for a few minutes, lets move on to a common pattern:

{% highlight javascript %}
  function Parent () { 
    this.sharedList = [];
  }
  Parent.prototype = new Object();
  
  function Child () {
    this.ownList = [];
  }
  Child.prototype = new Parent();
  
  var child1 = new Child(); // => Child { ownList: [] }
  var child2 = new Child(); // => Child { ownList: [] }
  
  child1.ownList === child2.ownList // => false
  child1.sharedList === child2.sharedList // => true
  child1.__proto__ // => Parent { sharedList: true }
  child1.__proto__.sharedList === child2.__proto__.sharedList // => true
  
  child1.constructor // => Object {} ... wut?
{% endhighlight %}

Okay, step by step, what happens here:

1. We define a Parent constructor that will on instantiating create a unique "sharedList"" property with an array for the object created
2. We define an instance of an Object as its prototype (empty object)

3. We define a Child constructor that will on instantiating create a unique "ownList" property with an array for the object created
4. We define an instance of a Parent as its prototype

5. We instantiate two Child objects

As we see each child will get their own "ownList" and they point to the same instantiated Parent object on their prototype. Now this works...

### But is this right?
If we would do this the JavaScript native objects way, the prototype of **Child** should not be an instance of **Parent**, the prototype of **Child** should be an instance of a **ChildPrototype** and the prototype of **ChildPrototype** should be an instance of **ParentPrototype**. Like this:

{% highlight javascript %}
/*
  Child -
    ownList: []
    __proto__ - ChildPrototype
      constructor: Child
      ownList: []
      __proto__ - ParentPrototype
        constructor: Parent
        sharedList: []
*/
{% endhighlight %}

Not only is the chain wrong, we also have the wrong constructor property. As you see in the previous code example it is pointing to Object, not **Child** as it should.

### Resolving the confusion
The way JavaScript inherits on its native objects is kinda weird at first, but after looking deeper into this it is actually quite interesting:

{% highlight javascript %}
  var MyArray = (function () {
    var Constr = function MyArray () {};
    var Proto = function MyArrayPrototype () {
      Object.defineProperty(this, 'constructor', {
        enumerable: false,
        configurable: false,
        writable: false,
        value: Proto
      });
      this.push = function () {};
    }
    Proto.prototype = Object.prototype;
    Constr.prototype = new Proto();
    return Constr;
  }());
  
  MyArray.constructor instanceof Function // => true
  Array.constructor instanceof Function // => true
  
  new MyArray().__proto__ // => MyArrayPrototype { push: function () {} }
  new Array().__proto__ // => []
  
  new MyArray() instanceof MyArray // => true
  MyArray.prototype instanceof MyArray // => false
  new Array() instanceof Array // => true
  Array.prototype instanceof Array // => false
  
  new MyArray().__proto__.__proto__ // => Object {}
  new Array().__proto__.__proto__ // => Object {}
   
{% endhighlight %}

So basically, after my research and understanding, defining a new object the native way requires two constructors. One constructor which is just a handler for instantiating, and one constructor to instantiate a prototype. So the conclusion here is that we have two types of inheritance:

1. You define two constructors, an "instantiation constructor" and a "prototype constructor". The "instantiation constructor" has a prototype property where an instance of the "prototype constructor" is located. This means that the prototype of your instantiated object by defenition is a **prototype**. The prototype is "all that object is" and your instantiated object is the bare minimum, everything else is inherited from the prototype. 

2. You define a single constructor with a prototype that holds the methods for your constructor. This is a very typical pattern, though it breaks with the way JavaScript does it natively because here the constructor that instantiates new objects and the prototype is "all that object is". Natively only the prototype should be "all that object is" and the constructor is just there to produce the "bare minimum" object that inherits from its prototype

Doing a lot of testing and research to write this article it has actually occured to me that it makes perfect sense that the prototype of your object is the complete object with all methods and default values... like, actually a prototype of it... not just an object-container for methods you want to share across instances. The actual job of a constructor is to instantiate a new object, manipulate its properties and attach the prototype. Not splitting the definition of your object across a constructor and a plain prototype object, like this:

{% highlight javascript %}
  
  function MyConstr () {
 
     this.myProp = [];
 
  }
  
  MyConstr.prototype = {
    doThis: function () {}
  };
   
{% endhighlight %}

Going native we should first define our prototype with a constructor and build a complete object with methods and other properties. Then we create an instantiation constructor for that prototype and set that constructors prototype to be an instance of the prototype constructor defined.

A code example:

{% highlight javascript %}
  
  var MyObject = (function () {
  
    var Constructor = function MyObject (options) {
      options = options || {};
      if (options.name) this.name = options.name;
    };
  
    var Prototype = function MyObjectPrototype () {
      var somePrivateMethod = function () {};
      this.constructor = Constructor;
      this.name = 'Unknown';
      this.method1 = function () {};
      this.method2 = function () {};
      this.method3 = function () {};
    };
    
    Prototype.prototype = Object.prototype;
    Constructor.prototype = new Prototype();
    
    return Constructor;
    
  }());
   
{% endhighlight %}

I dont know about you, but this kinda makes total sense to me, but of course it gives some extra code. Maybe we could do it a bit simpler?

### Making it look a bit nicer
There are many libraries out there and they tackle these issues differently, including me :-) This is the syntax of my small lib to make prototype and inheritance easier to do in JavaScript.

{% highlight javascript %}

  var MyBase = Protos('MyBase', function () {
    var list = [];
    this.push = function (item) {
      list.push(item);
    }
  });
  
  var myBase = new MyBase();
  myBase // => MyBase { }
  myBase.__proto__ // => MyBasePrototype { push: function () {} }
  
{% endhighlight %}

What you are doing here is actually building the prototype constructor for "MyBase". Since there is no constructor passed for instantiation you will get an empty one. So lets see what happens if we add our own constructor:

{% highlight javascript %}

  var MyBase = Protos('MyBase', function () { 
    this.myList = [];
  }, function () {
    this.myList = [];
    this.push = function (item) {
      this.myList.push(item);
    }
  });
  
  var myBase = new MyBase();
  myBase // => MyBase { myList: [] }
  myBase.__proto__ // => MyBasePrototype { myList: [], push: function () {} }
  
{% endhighlight %}

Passing in two functions tells Protos that the first one is your instantiation constructor and the second one is your prototype constructor. So how would we handle deeper inheritance? Well, if you just want to create a base prototype that will never get instantiated, only inherited, just pass in one function and build it:

{% highlight javascript %}

  var MyBase = Protos('MyBase', function () {
    var list = [];
    this.pushToAll = function (item) {
      list.push(item);
      return list;
    }
  });
  
  var Child = MyBase.extend('Child', function () {
    this.list = [];
  }, function () {
    this.list = [];
    this.push = function (item) {
      this.list.push(item);
      return list;
    }
  });
  
  var child1 = new Child();
  var child2 = new Child();
  child1.push('child1'); // => ['child1']
  child2.push('child2'); // => ['child2'];
  child1.pushToAll('something'); // => ['something']
  child2.pushToAll('somethingElse'); // => ['something', 'somethingElse']
  
  child1 // => Child { list: [] }
  child1.__proto__ // => ChildPrototype () { list: [], push: function... }
  child1.__proto__.__proto__ // => MyBasePrototype () { pushToAll: function... }
  
{% endhighlight %}

As you can see on the last logs we get a simple and logical prototype chain, as we would expect.  

Take a look at the repo for more info and download: [https://github.com/christianalfoni/Protos](https://github.com/christianalfoni/Protos)

### What about Object.create()?
Object.create is a semi-constructor. It will create an object for you and attach the prototype you pass, if any. This is great for creating new empty objects that needs some functionality from an other object. The problem though is that it does not fix this prototype chain differences. It will neither state where it came from when you debug. I have seen examples of this:

{% highlight javascript %}

  function SomeOther () {}
  SomeOther.prototype = { test: 'test' };
  function MyConstr () {}
  MyConstr.prototype = Object.create(SomeOther.prototype);

  new MyConstr();
  /*
    MyConstr {
      __proto__
        __proto__
        test: 'test'
    }
  */

{% endhighlight %}

This does not make sense because at its core a prototype is a single instance object. There is no reason to create a new object. It will just create an extra empty prototype. But if you are going to build up a new prototype based on an other:

{% highlight javascript %}

  function SomeOther () {}
  SomeOther.prototype = { test: 'test' };
  function MyConstr () {}
  MyConstr.prototype = Object.create(SomeOther.prototype);
  MyConstr.prototype.newProp = 'newProp';

{% endhighlight %}

It is of course a different story as you do not want to manipulate the existing prototype. Now, again, the problem here is that by debugging you will not know that you are chained with SomeOther... only some object: 

{% highlight javascript %}

  function SomeOther () {}
  SomeOther.prototype = { test: 'test' };
  function MyConstr () {}
  MyConstr.prototype = Object.create(SomeOther.prototype);
  MyConstr.prototype.newProp = 'newProp';
  
  new MyConstr();
  /*
    MyConstr {}
      newProp: 'newProp'
      __proto__ - Object
        test: 'test'
  */

{% endhighlight %}

Last, but not least, you will always need a constructor. You will need a constructor to produce multiple instances of the same type of object where you want unique complex objects for each instance.

### Conclusion
Okay, this was probably a lot to take in. Hopefully I managed to point out some interesting things about how JavaScript works and that the concepts are very loosely defined and extremely flexible. This is kind of a good thing, but it makes it very hard to comprehend certain parts of the language. Thanks for reading!