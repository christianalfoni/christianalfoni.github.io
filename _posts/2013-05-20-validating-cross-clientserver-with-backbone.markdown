---
layout: post
title:  "Validating cross client/server with Backbone"
date:   2013-05-20 09:21:30
categories: javascript
tags: ["backbone","node"]
---

### Background
I have been working on a project using Backbone and I must say I really like it. The project is a small CRUD application and for that
Backbone is a perfect fit. Bigger projects would probably require the use of Marionette JS, which I’m only at a theoretical level for the
time being. As I understand it, it has better handling of the views, especially considering memory leaks, and overall structure of the project.
On that note, have a look at: [BACKBONE RAILS EPISODES](http://www.backbonerails.com/series/engineering_single_page_apps). Episode 1-4 is something you should definitely check out!

That said, there was one detail that got me thinking… validation. There are multiple ways to go about this and I thought I would share some of my thoughts and how I ended up doing this.

### Backbones validation
Validation in Backbone is part of the Model. The model has a validate method. It can be used in conjunction with the isValid method, which returns true or false
based on what the validate method returns. If the validate method returns something else than the default undefined, it will set the models validationError
property to the error returned by the validate method and isValid will return false.

You can manually trigger validation by either calling the validate method or isValid method. When using set and save, you can send extra options which will
trigger the validator, stop the setting or saving of the model and trigger an “invalid” event.

Too much theory, lets see it in code:

{% highlight javascript %}

    var myModel = Backbone.Model.extend({
        defaults: {
            someProperty: 'someValue'
        },
        validate: function (attrs, options) {
            if (typeof attrs.someProperty !== 'string' || attrs.someProperty.length === 0) {
                return 'There should be some text here!';
            }
        }
    });

    myModel.isValid(); // => Returns true

    myModel.set('someProperty', '');
    myModel.isValid(); // -> Returns false
    console.log(myModel.validationError) // => 'There should be some text here'

    myModel.on('invalid', function () {
        console.log('Could not save or set model attributes');
    });
    myModel.set('someProperty', 'someText', { validate: true } ); // Will be set
    myModel.set('someProperty', '', { validate: true }); // Will not be set and triggers 'invalid' event

{% endhighlight %}

### So I got to a point…
The user can fill out a typical form and for this I needed some validation. But what was I actually validating here? Was I validating the form or the
model that the form would eventually produce? In other words, should I validate the form and then build a model or should I build a “blank” model,
update it with data from the form and then validate the model? The same form should both create new models and edit models too, how would that affect the validation?

Diving into the “validate the form”-concept I quickly found myself putting a lot of validation code inside the view presenting the form. I also had a
lot of code clearing out the form, triggering different events based on if it was a new model or update of a model. It did not feel right. So I did the following changes:

- **Moved all validation to the model**, cleaning up the view code.

{% highlight javascript %}

    var Message = Backbone.Model.extend({
        validate: validate.backboneModel('message'), // Getting back to how this works
    ...

{% endhighlight %}

- **Triggered the view with a new or existing model.** A new model, with default attributes, or an existing model would cause the
form to automatically clear itself, as it was based on the model passed in.

{% highlight javascript %}

    ...
    createMessage: function () {
        Backbone.trigger('show:modal', new MessageModel());
    },
    editMessage: function (message) {
        Backbone.trigger('show:modal', message);
    },
    ...

    // In the view, setting the value in the input
    this.$pointOfContactInput.val(this.message.get('pointOfContact'));
    ...

{% endhighlight %}

- **Setting form data on a cloned model.** Instead of collecting the form data and validating it, I set the data on a cloned model when the inputs change.

{% highlight javascript %}

    ...
    // When initializing the form view
    this.$commentInput = this.$('#comment').on('change', function () {
        self.message.set('comment', $(this).val());
    });
    ...
    // When showing the form view
    showModal: function (message) {
        this.originalMessage = message;
        this.message = message.clone();
    ...

{% endhighlight %}

- **Verify clone.** I check if the clone is valid, and if so I move the data over to the original model.

{% highlight javascript %}

    if (this.message.isValid()) {
        this.originalMessage.set(this.message.toJSON());
        Backbone.trigger('save:message', this.originalMessage);
    } else {
        this.invalidate(this.message.validationError);
    }

{% endhighlight %}

I chose to NOT go with the “invalid” event. It does not always make sense to wait for an event and in this situation
I just wanted to validate it and do one or the other based on if it was valid or not. So I run isValid and if it returns
true I check if the model isNew. If it is new I wait for the sync event to add it to my collection, causing it to appear
in the list of messages. Then I save the message.

{% highlight javascript %}

    ...
    saveMessage: function (message) {
        if (message.isNew()) {
            message.once('sync', function () {
                messagesCollection.add(message);
            });
        }
        message.save();
        // Should of course be some error handling here
    },
    ...

{% endhighlight %}

### Shared validation
Backbone has a really nice validation plugin, [BACKBONE.VALIDATION](https://github.com/thedersen/backbone.validation), but it has of course a dependency to the
Backbone library and for me it is not very straight forward. I want straight forward. This is the short version of everything I have written so far:

- **Initialise a view with a form.** The change event of each input changes an attribute on the referenced model in the view

- **Show the view passing a new or existing model.** The model is cloned in the view and the original one is kept.

- **Validate the cloned model, save the original.**

To me this seems more natural and it gives me more control, but I still have to do the actual validating. So I though I could have a try myself,
writing a small cross client/server validation library/plugin… whatever. This is what I wanted:

- **Validate on the fly.** If I for some reason just wanted to validate some data, that should be possible without pre-configuration.

- **Pre-configure validation.** If I am going to reuse the validation in different parts of the code and/or cross client/server I want to configure it.

- **I want to easily set a validate function to a Backbone model.** Setting the validation function would require a method call which returns the function that validates the model.

- **I want flexibility in the actual validators.** It should be easy to set validation by length, type of data and cross validate on properties like the property “startDate”-date should be before det “endDate”-date.

### In action
The **validator** lets you do four different things. Configure error messages, pre-configure validation, use a pre-configured validation in different scenarios and of course validate.

{% highlight javascript %}

    // Configure error messages, where the % is a set placeholder for type of validation
    validate.config({
        errorMessages: {
            length: 'You have the wrong length, it should be % characters long',
            email: 'The email %, is not valid, please insert a valid email'
        }
    });

    // Pre-configure a validation which lets you easily validate any object
    validate.extend('message', {
        comment: ['required'],
        pointOfContact: ['required'],
        startTime: ['required', 'before:endTime'],
        endTime: ['required', 'after:startTime'],
        priority: ['required']
    });
    validate('message', myMessage.toJSON());

    // Set a backbone validation, by using a preconfigured validation
    var Message = Backbone.Model.extend({
        validate: validate.backboneModel('message'),
    ...

    // Validate on the fly
    validate({
        myProperty: 'yeeah'
    }, {
        myProperty: ['required']
    });

{% endhighlight %}

The validator returns an object with two properties, isValid: true/false and an errors object. It could look something like this:

{% highlight javascript %}

    var validation = validate({
        myProperty: ''
    }, {
        myProperty: ['required']
    });
    console.log(validation.isValid) // -> Logs out: false
    console.log(validation.errors) // -> Logs out default required error { myProperty: ['You have to type something!'] }

{% endhighlight %}

So to make this validation work cross client and server you should have a validation module which depends on the validate plugin. This is my validation module:

{% highlight javascript %}

    define(['validate'], function (validate) {
        'use strict';
        validate.extend('message', {
            comment: ['required'],
            pointOfContact: ['required'],
            startTime: ['required', 'before:endTime'],
            endTime: ['required', 'after:startTime'],
            priority: ['required']
        });
        return validate; // I return the plugin to access the validation methods
    });

{% endhighlight %}

So two different modules, my model and my router, depends on the validation module:

{% highlight javascript %}

    // MessageModel.js
    define(['backbone', 'timeHelper', 'validation'], function (Backbone, time, validate) {
        'use strict';
        var Message = Backbone.Model.extend({
            validate: validate.backboneModel('message'),
    ...

    // routes.js in the middleend
    ...
    createMessage: function (req, res) {
        var validation = validate('message', req.body);
        if (!validation.isValid) {
            return res.send(400, validation.errors);
        }
    ...

{% endhighlight %}

So that is just some thoughts on validation. The plugin is up on github, if it suits your needs I will be very happy if you want to contribute with more validators. More info on that:
[VALIDATE ON GITHUB](https://github.com/christianalfoni/validate)