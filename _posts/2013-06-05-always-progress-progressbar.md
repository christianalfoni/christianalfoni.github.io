---
layout: post
title:  "Always progress progressbar"
date:   2013-06-05 09:21:30
categories: javascript
tags: ["javascript", "css"]
---

I got thinking about the progressbar we use on a project. To prevent the user from feeling that something is wrong we display the progress of fetching resources.
This has to do with user experience. If we only used a spinning loader, or even worse, nothing at all, the user would not get any progress feedback and in a
worst case scenario might think that something is wrong. The progressbar on the other hand uses 5 steps (5 different resources) to indicate that something is actually happening.

But I felt this was not good enough. Even though we have 5 steps, each step stops, waiting for the next step to complete. We even have an animation on
the actual progressbar, making it feel that something is happening. Still a worst case scenario is the feeling of something not working correctly, it is
not the optimal user experience. This might seem a bit extreme, it does not really take much time to load the application, but it’s a principle!

So my thought was; “What if the progressbar progressed continuously, but adapted its speed to the finished steps?”. There might be libraries for
this out there already, but I wanted to solve this challenge on my own.

First of all I wanted it to have as many steps as you would need, synchronously or asynchronously and it had to “feel right”!

To have a quick look before reading more, check out this jsfiddle: [HTTP://JSFIDDLE.NET/XEWRF/2/](HTTP://JSFIDDLE.NET/XEWRF/2/).

### What I wanted to write

When I work on small solutions like this one, I usually write the code that should make my solution work first. In this case, I wanted this:

{% highlight javascript %}

    alwaysProgress(document.getElementById('bar'), true, 30).
        step(function (done) {
            setTimeout(function () {
                console.log('done with step 1');
                done();
            }, 2000);
        }).
        step(function (done) {
            setTimeout(function () {
                console.log('done with step 2');
                done();
            }, 4000);
        }).
        step(function (done) {
            setTimeout(function () {
                console.log('done with step 2');
                done();
            }, 6000);
        }).
        step(function (done) {
            setTimeout(function () {
                console.log('done with step 2');
                done();
            }, 16000);
        }).
        finished(function (finishedSteps) {
            console.log('done with ' + finishedSteps);
        });

{% endhighlight %}

I wanted to call a function with three arguments. The first argument is a block (display: block) element which is my actual progressbar.
The second argument is an optional bool argument. If it is true it will handle the steps synchronously. The last argument
(which requires to set the second bool argument) is a maximum expected loading time. This last argument helps you tune the progression,
preventing if from slowing down. By default it is based on the number of steps you set up.

I also wanted this to work with jQuery, which lets me write it like this:

{% highlight javascript %}

    $('#bar').alwaysProgress(true, 30).
    ...

{% endhighlight %}

I also wanted to handle loading it with requirejs, with jQuery or not.

The solution
How it handles being triggered in combination with other libraries is not very interesting, but how it actually works hopefully is.

First of all, without any steps, the progressbar goes from 0-100% with an interval of 30 milliseconds. It will never move from one percent to
the next faster than this. So during the speed-calculation, if the calculation drops below 30 it will still be 30 milliseconds.
This actually proved to be my last “feel right” fix.

So when the progressbar fires up it moves to the next percent with an interval of 30 milliseconds, but it quickly figures out that it should slow down:

{% highlight javascript %}

    ...
    progress = function () {
        var newInterval = minInterval * (progressed - (100 / steps * finishedSteps));
        progressed++;
    ...

{% endhighlight %}

I am no mathematician, you would be amazed what kind of simple mathematical conundrums I can entangle myself into and make far worse than they really are…
but hopefully what I lack in math I make up in more practical problem solving and intuition.

So first I needed to start somewhere, so why not start with the minimum interval? I wanted the minimum interval to increase when waiting for steps
to finish, causing longer time between each percent jump. The closer the progress got to the next/last step without that step being finished,
the slower it would move. So by taking the current progress and subtract with the progress of the finished steps, I get a number.

If the progressbar is at 24% and I am still missing all of my 4 steps, it will actually multiply the interval with 24, setting it to
720 milliseconds ( 30 * (24 – (100 / 4 * 0)) ). But if it was at 15% and the first step was complete, the number would be 30 * -10 = -300,
causing the minimum interval to kick in and set it to 30 milliseconds again until it tips the next steps progress and will slow down again.
This is kinda nice, but 720 milliseconds is not really that much. It has to go slower or steps using lots of seconds might pass the allowed
percentages of that step. I initially thought I could just multiply with the number of steps, but ended up giving some tuning configuration.

By setting f.ex. 30 seconds as maximum load time (3 seconds more than actual load time in this example) and then divide that by 10, which is 3.
That works very well, also if I remove the 20 second step and set maximum load time to 10 (3 seconds more than actual load time)
it also works very well. So this is how the calculation ended up:

{% highlight javascript %}

    ...
    progress = function () {
        var newInterval = minInterval * (progressed - (100 / steps * finishedSteps)) * (expectedMaxSeconds ? expectedMaxSeconds / 10 : steps);
        progressed++;
    ...

{% endhighlight %}

Giving a expectedMaxSeconds on each step for synchronous steps would improve this even more, but that is not implemented.
It would probably also be a good idea to have a “breaker”. If it is about to pass a progress step it should not pass, it should
temporarily stop until the step is finished and it can continue.

We are going to use this in our next project so it will be updated with at least the “breaker” by then.

Again, this is no hardcore mathematical brainiac stuff, it just feels right!