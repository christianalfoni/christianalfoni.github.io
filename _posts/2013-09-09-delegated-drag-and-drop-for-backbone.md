---
layout: post
title:  "Delegated drag and drop for Backbone"
date:   2013-06-23 09:21:30
categories: javascript
tags: ["backbone"]
---

Download from: [BACKBONE-DRAGANDDROP-DELEGATION](https://github.com/christianalfoni/backbone-dragandrop-delegation)

I’m working on a project now using Backbone. The interface has a lot of elements, multiple lists and the items in the lists should be draggable. Earlier I built a small plugin, it did not use delegation and it had some weird code due to IE8 support. This project does not have to support IE8 and it should support delegation as that is how Backbone structures its event listening. So I thought I could have a look at the native drag and drop implementation.

### Long story short…

Long story short, native drag and drop does not work very well. I like the concepts of just having the events and not a lot of crazy extra options like jQuery UI, but it is not possible to execute it consistently across browsers. Since the project uses Backbone I also wanted drag and drop to use delegation and searching the web I did not really find any good fit. As I understand jQuery UI you will have to find all draggable elements and run the draggable() method, which probably adds multiple events to each draggable element… I did not want that.

So a colleague of mine started implementing the native drag and drop with Backbone. The structure was really nice, but as stated, it did not work consistently across browsers. So I started with the structure and built a small jQuery plugin, extending elements with a new detectDrag() method that made the experience consistent.

### The changes

I did one big change though. Instead of having the attribute: draggable=”true”, I used: dropable=”true” instead. I have no idea why the native drag and drop handles draggable=”true” as both drop and drag containers, it is really weird. I thought about using both, but since I wanted delegation it would go against the concept if I added a “mousedown”-event to all items in the list with draggable=”true” to detect the drag.

To handle passing of data there was no reason to limit it to just strings, like native drag and drop does. So when the drag starts you get access to a data object. You can add properties to it directly or use the set() method to set the data, it either being a new object, a string etc.

### How you use it

So this is how a View with draggable list items ended up. It looks exactly like native drag and drop, except that you have to detect drag on mousedown.

{% highlight javascript linenos %}

    Backbone.View.extend({
        tagName: 'div',
        className: 'dragdrop',
        events: {
            //Handle drag
            'mousedown .draggable': 'detectDrag',
            'dragstart .draggable': 'dragstart',
            'dragend .draggable': 'dragend',
            // Handle drop
            'dragenter .dropable': 'dragenter',
            'dragleave .dropable': 'dragleave',
            'drop .dropable': 'drop'
        },
        detectDrag: function (event) {
            // Trigger detectDrag on the mousedowned element
            $(event.currentTarget).detectDrag();
        },
        dragstart: function (event, data, clone, element) {
            // Add any properties to the data object
            // to pass it to the drop. You can also
            // manipulate the clone created that follows the
            // mouse pointer and the element that started
            // the drag
            data.someProperty = 'someValue';
            // Or set the value of data
            data.set('someValue');
        },
        dragenter: function (event, clone, element) {
            // When you drag something and hover the drop container.
            // Do something to the dropcontainer, the element where the
            // drag started or the clone following the cursor
            var dropContainer = $(event.currentTarget);
            dropContainer.animate({opacity: 1}, 'fast');
        },
        dragleave: function (event, clone, element) {
            // Typically revert changes on dragenter
            var dropContainer = $(event.currentTarget);
            dropContainer.animate({opacity: 0.5}, 'fast');
        },
        drop: function (event, data, clone, element) {
            // Trigger code on drop
            console.log(data.someProperty); // => someValue
            // Or if data was set()
            console.log(data); // => someValue
        },
        dragend: function (event, clone, element) {
            // Triggered when NOT dropping into a drop container.
            // Either the drop or the dragend event is triggered
        }
    });

{% endhighlight %}

Note that dragging and dropping events does not have to be in the same View. In our project we listen to drag events in one view and drop events in an other.

### How it works

The code is actually quite small, around 60 lines. It works pretty much like this:

- When detectDrag() method is called it listens to selectstart on the element and mouseup and mousemove on window
- Selectstart listening prevents any text selection to occur. Mouseup triggers “drop” or “dragend” event, and unregisters all events. Mousemove detects a starting drag
- If a drag is detected the element is cloned and positioned in the body about 5 pixels away from the cursor. The clone can not be underneath the cursor as the cursor has to detect drop containers
- When you move the mouse it will check if the current target has a “dropable” attribute, triggering dragenter or dragleave on the current target
- Any event triggered will bubble up the DOM and thus supporting event delegation in Backbone