---
layout: post
title:  "React JS and a browserify workflow"
date:   2014-08-15 09:21:30
categories: javascript
tags: ["javascript"]
---

I have been working with **React JS** for a few days and I must say I am impressed. You can not really compare it to complete frameworks like **Angular JS** or **Ember JS**, but at the same time it is worth mentioning in the same context. A full framework helps you develop very fast and is a dream when it comes to prototyping, but building a high performance web application is easier when you control each part. More on that in a later post, lets dive into how I found an ideal way to work with React JS.

### The problem
I really do not appreciate the simplicity of examples shown on probably all library/framework sites. Nobody builds an application in the global scope, nobody builds an application in a single file and everybody wants to make their code production ready. The **React JS** site is no different and it took me quite some time creating a good workflow. This is what I needed:

- Write JSX and transform it to regular javascript on the fly
- Write my code in a module pattern
- Use the React JS chrome dev tool
- Bundle my javascript and use source maps for debugging
- Run other tasks on file change, like CSS concat etc.
- It has to be extremely fast

This post will go through each step I did to find a solution. If you just want the solution, [jump ahead](#solution).

### First try
So my first challenge was to solve the JSX transformation on the fly. Looking at the [React JS guide](http://facebook.github.io/react/docs/getting-started.html) I quickly figured out that the only thing I needed was to include an extra script tag in my HTML. It solves my problem, but it is not what I am looking for. The extra script tag is not something you want to put in production, so I would need to pre-transform my files containing JSX. 

### Second try
As the **React JS guide** states, you can install the **react-tools** globally and use a command line tool to watch files and convert them. It solves my problem, but it transforms each individual file and puts it in a different folder. This will create a messy project structure. At this point I thought maybe I was focusing on the wrong problem, I decided to look at some module pattern tools, maybe they have some plugins to convert the JSX for me.

### Third try
Having quite a bit of experience with **requirejs** I thought that would be a good bet, but it became more complex than I initially thought. A great job has been done on this plugin: [jsx-requirejs-plugin]('https://github.com/philix/jsx-requirejs-plugin'), but you get extra dependenices like the "text" plugin for requirejs and you have to use a modified version of the "JSXTransformer", that does not feel good. There was also an issue with bundling the whole project on each file change, it was too slow.

### Fourth try
Personally I have never tried **browserify** before, but I was running out of options. It turns out that browserify is quite awesome, it makes it possible to write node syntax in your javascript files. The result of this is that you get modules and the possibility to re-use your JSX files in Node. That is a very good thing because **React JS** allows for server side compiling of HTML and send the string out to the client for further handling by the client side version of the library.

What is also good about **browserify** is that you have a plugin called **watchify** that will cache your project and watch it for changes, only doing the necessary re-bundling on updates. It is blazingly fast! Since I am used to **Grunt** I tried using the **grunt-browserify** and **grunt-watchify** tasks. This worked quite well, though I could not get the sourcemapping to work. But an even bigger issue was that I did not only need to watch the javascript files for changes, I also needed to concatinate my css files.

After some research I found a **grunt-concurrent** task that would allow me to run two parallell watch tasks. This worked, but it was slow. And I still could not get my sourcemapping to work.

### Fifth try
So the way I used **Grunt** was too slow, what about **Gulp**? Now **Gulp** is a build tool like **Grunt**, only it streams the process so that you can manipulate the build process in memory instead of creating temporary files that are picked up by the next step in the build process.

Searching the web for a solution for **Gulp** I hit [this]('http://stackoverflow.com/questions/24190351/using-gulp-browserify-for-my-react-js-modules-im-getting-require-is-not-define') post on stackoverflow. It shows multiple solutions to the same problem and way down I found an example that also handled converting the files with **reactify**, but it had some steps I did not understand. Why all this require stuff? And why require specific react.js file? It feels a bit hacky. It also gave me an error when I ran it in the browser.

This led me to using **gulp-browserify** which worked beautifully. But what about the **watchify** part? Searching for **gulp-watchify** it states: "experimental". That does not feel right, so how could I get this stuff to work? Searching the web again I found the following statement: "gulp-browserify has been blacklisted". DOH!, what now?

### Sixth try
Sometimes you just have to do it from scratch. Using what I had experienced so far I built my own build process using the **browserify**, **watchify** and **reactify** npm modules. I was glad to see that **Gulp** also handled watching both my CSS for one task and that **watchify** task in parallell without any issues.

# <a name="solution"></a>The solution
So this is how my **gulpfile.js** file looks like:

{% highlight javascript %}
var gulp = require('gulp');
var source = require('vinyl-source-stream'); // Used to stream bundle for further concatination etc.
var browserify = require('browserify');
var watchify = require('watchify');
var reactify = require('reactify'); 
var concat = require('gulp-concat');
 
gulp.task('browserify', function() {
	var bundler = browserify({
		entries: ['./app/main.js'], // Only need initial file, browserify finds the deps
		transform: [reactify], // We want to convert JSX to normal javascript
		debug: true, // Gives us sourcemapping
		cache: {}, packageCache: {}, fullPaths: true // Requirement of watchify
	});
	var watcher  = watchify(bundler);

	return watcher
	.on('update', function () { // When any files update
		var updateStart = Date.now();
		console.log('Updating!');
		watcher.bundle() // Create new bundle that uses the cache for high performance
		.pipe(source('main.js'))
		.pipe(gulp.dest('./build/'));
		console.log('Updated!', (Date.now() - updateStart) + 'ms');
	})
	.bundle() // Create the initial bundle when starting the task
	.pipe(source('main.js'))
	.pipe(gulp.dest('./build/'));
});

// I added this so that you see how to run two watch tasks
gulp.task('css', function () {
	gulp.watch('styles/**/*.css', function () {
		return gulp.src('styles/**/*.css')
		.pipe(concat('main.css'))
		.pipe(gulp.dest('build/'));
	});
});

// Just running the two tasks
gulp.task('default', ['browserify', 'css']);
{% endhighlight %}

So to summarize: 

- Write JSX and transform it to regular javascript on the fly **(SOLVED)**
- Write my code in a module pattern **(SOLVED)**
- Use the React JS chrome dev tool
- Bundle my javascript and use source maps for debugging **(SOLVED)**
- Run other tasks on file change, like CSS concat etc. **(SOLVED)**
- It has to be extremely fast **(SOLVED)**

### What about the REACT DEV-TOOLS?
The **React DEV-TOOLS** needs an instance of the React JS lib on the global scope to trigger. To fix this you simply have to add it to your **main.js** file. 

{% highlight javascript %}
/** @jsx React.DOM */

var React = require('react');
// Here we put our React instance to the global scope. Make sure you do not put it 
// into production and make sure that you close and open your console if the 
// DEV-TOOLS does not display
window.React = React; 

var router = require('./router.js');
var externals = require('./utils/externals.js');

var App = require('./App.jsx');
React.renderComponent(<App/>, document.body);
{% endhighlight %}

So that was my journey. I truly hope that the people at Facebook will put an example of this on the React JS site. React JS is truly awesome and its very sad that people leave it out because it is such a pain setting up a good workflow. 

Thanks for listening to my story and have fun with [React JS]('http://facebook.github.io/react')!