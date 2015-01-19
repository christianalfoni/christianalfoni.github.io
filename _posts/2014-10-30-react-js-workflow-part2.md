---
layout: post
title:  "React JS and a browserify workflow, PART 2"
date:   2014-10-30 09:21:30
categories: javascript
tags: ["javascript"]
---

I wrote an article on working with React JS in a browserify workflow. Well, I got some more experience with it and here is PART2. You can grab this boilerplate at [react-app-boilerplate](https://github.com/christianalfoni/react-app-boilerplate).

There were specifically two things we did not handle very well on the initial workflow:

1. External dependencies
2. Testing

### External dependencies
When browserify rebundles your project it goes through all the require statements in your application code and figures out what to put into your bundle. Luckily we have watchify that makes sure that it does not rebundle what it already has bundled, BUT browserify still goes through the files and figures out if something should be done. You might not notice this when only React JS is part of the bundle, but as you load up more dependencies it can take several seconds on each save and rebundle of your project.

What we want to do is only touch these external dependencies once and never touch them again during the workflow session. You will not do any changes to React JS or underscore I suppose?

### Testing
The initial version of this workflow used Karma for testing. Now Karma is great, but by including browserify as part of the testing it got very slow. By letting our current workflow bundle our tests with external dependencies and watchify we can make this a lot faster! It would also be nice that our tests run automatically when we do changes to our project.

### Our Gulpfile.js
{% highlight javascript %}
// We need a bunch of dependencies, but they all do an important
// part of this workflow
var gulp = require('gulp');
var source = require('vinyl-source-stream');
var browserify = require('browserify');
var watchify = require('watchify');
var reactify = require('reactify'); 
var gulpif = require('gulp-if');
var uglify = require('gulp-uglify');
var streamify = require('gulp-streamify');
var notify = require('gulp-notify');
var concat = require('gulp-concat');
var cssmin = require('gulp-cssmin');
var gutil = require('gulp-util');
var shell = require('gulp-shell');
var glob = require('glob');
var livereload = require('gulp-livereload');
var jasminePhantomJs = require('gulp-jasmine2-phantomjs');

// We create an array of dependencies. These are NPM modules you have
// installed in node_modules. Think: "require('react')" or "require('underscore')"
var dependencies = [
	'react' // react is part of this boilerplate
];

// Now this task both runs your workflow and deploys the code,
// so you will see "options.development" being used to differenciate
// what to do
var browserifyTask = function (options) {

  /* First we define our application bundler. This bundle is the
     files you create in the "app" folder */
	var appBundler = browserify({
		entries: [options.src], // The entry file, normally "main.js"
        transform: [reactify], // Convert JSX style
		debug: options.development, // Sourcemapping
		cache: {}, packageCache: {}, fullPaths: true // Requirement of watchify
	});

	/* We set our dependencies as externals of our app bundler.
     For some reason it does not work to set these in the options above */
  appBundler.external(options.development ? dependencies : []);
  
  /* This is the actual rebundle process of our application bundle. It produces
    a "main.js" file in our "build" folder. */
  var rebundle = function () {
    var start = Date.now();
    console.log('Building APP bundle');
    appBundler.bundle()
      .on('error', gutil.log)
      .pipe(source('main.js'))
      .pipe(gulpif(!options.development, streamify(uglify())))
      .pipe(gulp.dest(options.dest))
      .pipe(gulpif(options.development, livereload())) // It notifies livereload about a change if you use it
      .pipe(notify(function () {
        console.log('APP bundle built in ' + (Date.now() - start) + 'ms');
      }));
  };

  /* When we are developing we want to watch for changes and
    trigger a rebundle */
  if (options.development) {
    appBundler = watchify(appBundler);
    appBundler.on('update', rebundle);
  }
  
  // And trigger the initial bundling
  rebundle();

  if (options.development) {

    // We need to find all our test files to pass to our test bundler
  	var testFiles = glob.sync('./specs/**/*-spec.js');
    
    /* This bundle will include all the test files and whatever modules
       they require from the application */
    var testBundler = browserify({
      entries: testFiles,
      debug: true,
      transform: [reactify],
      cache: {}, packageCache: {}, fullPaths: true // Requirement of watchify
    });

    // Again we tell this bundle about our external dependencies
    testBundler.external(dependencies);

    /* Now this is the actual bundle process that ends up in a "specs.js" file
      in our "build" folder */
    var rebundleTests = function () {
      var start = Date.now();
      console.log('Building TEST bundle');
      testBundler.bundle()
        .on('error', gutil.log)
        .pipe(source('specs.js'))
        .pipe(gulp.dest(options.dest))
        .pipe(livereload()) // Every time it rebundles it triggers livereload
        .pipe(notify(function () {
          console.log('TEST bundle built in ' + (Date.now() - start) + 'ms');
        }));
    };
    
    // We watch our test bundle
    testBundler = watchify(testBundler);
    
    // We make sure it rebundles on file change
    testBundler.on('update', rebundleTests);
    
    // Then we create the first bundle
    rebundleTests();

    /* And now we have to create our third bundle, which are our external dependencies,
      or vendors. This is React JS, underscore, jQuery etc. We only do this when developing
      as our deployed code will be one file with all application files and vendors */
    var vendorsBundler = browserify({
      debug: true, // It is nice to have sourcemapping when developing
      require: dependencies
    });
    
    /* We only run the vendor bundler once, as we do not care about changes here,
      as there are none */
    var start = new Date();
    console.log('Building VENDORS bundle');
    vendorsBundler.bundle()
      .on('error', gutil.log)
      .pipe(source('vendors.js'))
      .pipe(gulpif(!options.development, streamify(uglify())))
      .pipe(gulp.dest(options.dest))
      .pipe(notify(function () {
        console.log('VENDORS bundle built in ' + (Date.now() - start) + 'ms');
      }));
    
  }
  
}

// We also have a simple css task here that you can replace with
// SaSS, Less or whatever
var cssTask = function (options) {
    if (options.development) {
      var run = function () {
        gulp.src(options.src)
          .pipe(concat('main.css'))
          .pipe(gulp.dest(options.dest));
      };
      run();
      gulp.watch(options.src, run);
    } else {
      gulp.src(options.src)
        .pipe(concat('main.css'))
        .pipe(cssmin())
        .pipe(gulp.dest(options.dest));   
    }
}

// Starts our development workflow
gulp.task('default', function () {

  browserifyTask({
    development: true,
    src: './app/main.js',
    dest: './build'
  });
  
  cssTask({
    development: true,
    src: './styles/**/*.css',
    dest: './build'
  });

});

// Deploys code to our "dist" folder
gulp.task('deploy', function () {

  browserifyTask({
    development: false,
    src: './app/main.js',
    dest: './dist'
  });
  
  cssTask({
    development: false,
    src: './styles/**/*.css',
    dest: './dist'
  });

});

// Runs the test with phantomJS and produces XML files
// that can be used with f.ex. jenkins
gulp.task('test', function () {
    return gulp.src('./build/testrunner-phantomjs.html').pipe(jasminePhantomJs());
});
{% endhighlight %}

### Getting in the flow

1. Run `gulp`
2. Start up a webservice in the "build" folder, f.ex. `python -m SimpleHTTPServer 3000`
3. Go to "localhost:3000", here you will find your app
4. Go to "localhost:3000/testrunner.html", here you will find your tests

To make **LiveReload** work you need to install an extension to Chrome, [LiveReload](https://chrome.google.com/webstore/detail/livereload/jnihajbhpnppcggbcgedagnkighmdlei). This puts a small button up in the right corner of your browser. When on you app and/or your tests, hit that button and you will see the little circle in the center will become a black dot. Now these pages will refresh on file changes either in your tests or application files... blazingly fast!
