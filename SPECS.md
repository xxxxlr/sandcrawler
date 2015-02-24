Sandcrawler
===========

Sandcrawler is (or rather will be) a scraping framework meant to be used within a [Node.js](http://nodejs.org/) environment.

Its aims is to offer a straightforward API to inject scraping scripts that will be executed client-side (leveraged with jQuery and artoo.js) on target pages thanks to [phantomjs](http://phantomjs.org/).

It should therefore provide an API simple enough to be quick to code for simple use cases but should also provide a full array of options enabling us to perform complex and production-viable tasks if needed.

Finally, even if sandcrawler's primary goal is to target phantomjs, it should remain possible to use the framework through non-dynamic engines such as [request](https://www.npmjs.org/package/request) in a homoiconic fashion.


## Introduction

### Name

Sandcrawler is the Jawas' droid transport on Tatooine.

<p align="center">
  <img width="300" src="http://img4.wikia.nocookie.net/__cb20130812001443/starwars/images/f/ff/Sandcrawler.png">
</p>

It refers to [artoo.js](http://medialab.github.io/artoo/) through the Star Wars universe as well as being a pun on the *crawler* word.


### Concepts

#### JawaScript

I will use the expression *JawaScript* anytime I have to refer to a function or piece of code written in a node.js script but that will in fact be executed in a phantomjs' page context.

<p align="center">
  <img width="300" src="http://www.harlan-workshop.com/wp-content/uploads/ImagesHW/Costume_Jawa/Jawa_002.jpg">
</p>

#### Crawler

**Note** - should this be named a crawler?

A crawler, in sandcrawler, is an object attached to a phantomjs child and that can be used to perform scraping tasks and subscribe to its phantomjs child's events for debugging purposes.

#### Task

**Note** - should this be named a task?

A task, in sandcrawler, is created by a crawler instance and defines the urls to scrape and how they should be scraped.


### Scraping lifecycle

To provide for an easily hookable API, sandcrawler uses an event-based scraping lifecycle.

Here is a concise list of events that can be subscribed to:

**Crawler level**

* `phantom:log` (the phantom child logged a message)
* `phantom:error` (the phantom child threw a JavaScript error)
* `phantom:exit` (the phantom child exited unexpectedly)

**Task level**

* `task:before` (before task is started - exposes middleware system - *Example: online account logging*)
* `task:start` (the task is starting)
* `task:fail` (the task failed globally)
* `task:success` (the task succeeded globally)
* `task:end` (the task is over)

**Page level**

*Events*

* `page:log` (the scraped page logged a message)
* `page:error` (the scraped page threw a JavaScript error)
* `page:alert` (the scraped page triggered an alert)
* other phantomjs page hooks to be determined

*Page cycle*

* `page:before` (before page is ordered to be scraped - exposes middleware system - *Example: throttling*)
* `page:scrape` (a page will be scraped)
* `page:after` (after page has been scraped - exposes middleware system* - *Example: data validation*)
* `page:retry`
* `page:add`
* `page:delay` over or under the stack?

*Page done*

* `page:process` (a page has to be processed)
* `page:success` (a page has successfully been scraped)
* `page:fail` (a page hasn't successfully been scraped)


## API Proposition

Overall, I try here to stick to Node.js patterns as firmly as I can so the library can interact without issues with the Node.js environment.

### Installation

Through npm (note that phantomjs will be automatically installed along sandcrawler without further ado).

```bash
npm install sandcrawler
```

Then require it.

```js
var sandcrawler = require('sandcrawler');
```

### Instantiation

```js
sandcrawler.create(params, function(err, crawler) {
  if (err) {
    // Something went badly when instantiating the crawler
  }

  // Otherwise do something with the crawler
  crawler.blabla();
});
```

You can of course indicate a lot of parameters here such as the phantomjs child command line arguments etc. and whether you want your crawler to be dynamic or not.

**Note** - both crawler and task objects are event emitters.

### Creating a task

**Note** - Naming alert!!!

```js
// To permit easy chaining, a task can be created likewise
var myTask = crawler.task(feed);
```

Possible feeds being:

* a single url
* an array of url
* an iterator function

User can also provide an expressive object rather than a string url through the different feeds if he wants finer and different settings for each of the cases he needs to treat. Also, any additional data given should be kept for an eventual later use in the scraping process.

**Note** - Everything being totally asynchronous, user is free to launch several tasks in parallel if he wants to.

Once your task is created, you can chain a wide array of method to register callbacks and tune finely the behaviour of your task.

### Example - Scraping a single url

*Most simple case*

```js
crawler
  .task('http://myfancyurl.com')
  .inject(function(done) {
    var data = $('.title > a').scrape('href');
    done(data);
  })
  .then(function(err, data) {
    console.log(data);
  });
```

*Explanation*

```js
crawler

  // Creating a single url task
  .task('http://myfancyurl.com')

  // JawaScript to inject into the page we need to scrape
  // We need to call the done callback to asynchronously indicate
  // to phantomjs that the job is done.
  .inject(function(done) {
    var data = $('.title > a').scrape('href');
    done(data);
  })

  // Final callback firing when the task completed successfully
  // and taking the scraped data as a single argument.
  .then(function(err, data) {
    console.log(data);
  });
```

*More complex case*

```js
crawler
  .task('http://myfancyurl.com')

  // Injecting scraping script from a file
  .injectScript('./scrapers/my-file.js')

  // We want to subscribe to the page's javascript log
  .on('page:log', function(url, msg) {
    console.log('page ' + url + ' logged:', msg);
  })

  .then(function(err, data) {

    // The task could fail
    if (err) {
      // Deal with errors, life is hard...
    }

    console.log(data);
  });
```

### Example - Scraping several urls

Simply passing an array to the `task` method will work.

Here, we might want to process each scraped page separately.

```js
crawler
  .task(['url1', 'url2'])
  .inject(function(done) {
    var data = $('.title > a').scrape('href');
    done(data);
  })

  // Here we register a callback fired whenever a page has been scraped
  .process(function(err, page) {
    if (err) {
      // Something went badly when scraping the page
    }

    // Else we can do something with the retrieved data
    console.log(page.url, 'scraped data:', page.data);

    // For instance now we save data into a mongo store
    mongo.save(page);
  })

  // Final hook
  .then(function(err, report) {
    console.log('here are the page failing:', report.remains);

    console.log(report);
  });
```

### Non-dynamic engine

One could easily create a non-dynamic scraper while using cheerio and artoo.js' node version to achieve his/her goals.

**Note** - Should we still use *inject* or switch to *parse*? On a more wide subject: how can we ensure homoiconicity? or should we even ensure homoiconicity?

```js
sandcrawler.create({dynamic: false}, function(err, crawler) {
  crawler
    .task('http://myfancyurl.com')
    .parse(function($) {

      // Here, $ would refer to the cheerio selector
      return $('.title > a').scrape('href');
    })
    .then(function(err, data) {
      console.log(data);
    });
});
```

### Retries & Expansion

You might want to be able to tell the crawler that you want to retry scraping a page on failure.

```js
crawler
  .task(['url1', 'url2'])
  .inject(function(done) {
    var data = $('a').scrape({
      title: 'text',
      link: 'href'
    });

    done(data);
  })
  .process(function(err, page) {
    if (err) {
      // An error occured, we want to retry
      return this.retry(page, /* params */);
    }

    // We want to add found links to the current task
    page.data.forEach(function(item) {
      this.add(item.link);
    }, this);
  })
  .then(function() {
    // Finished...
  });
```

### Middlewares and Plugins

As both the crawler and the task class are event emitters, it should be quite easy to hook on their lifecycle to provide reusable logic to them.

Here is my proposition on how reusable logic might be implemented:

#### Proposition

**Note** - `plug` is rather ugly. Naming alert!!!

```js
// Let's say we want to be able to validate data before considering
// scraped data is correct.
function validate() {

  // Hooking a middleware on the after scrape event
  // Note that `this` here is the task on which the function is plugged
  this.afterScraping(function(page, next) {
    if (page.data instanceof Array)
      next();
    else
      next(new Error('invalid-data'));
  });
}

// Now let's plug it
crawler
  .task(['url1', 'url2'])
  .plug(validate())
  .process(function(err, page) {
    if (err.message === 'invalid-data')
      return console.log('Aha, I told you it looked fishy!')

    // Processing...
  });
```

#### Examples

##### Fancy logger

One could want to integrate a helpful and colorful console logger right out of the box.

```js
// Minimalist implementation
function logger() {

  // Hooking on page log
  this.on('page:log', function(url, msg) {
    console.log('page logged:', msg);
  });

  // Hooking on the task failure
  this.on('task:fail', function() {
    console.error('Boom! Explosion!');
  });
}

crawler
  .task(['url1', 'url2'])
  .plug(logger());
```

##### Miscellaneous

* auto-trottling
* lazy logging (connecting if needed to a facebook account before attempting to scrape)
* mongo or other dbs pipelines (scrapy style in a sense...)
* etc.

### Task methods index

* `inject` (inject some JawaScript closure)
* `injectScript` (inject some JawaScript from a file)
* `injectSync` (inject some synchronous JawaScript)
* `injectScriptSync` (inject some synchronous JawaScript from a file)
* `parse` (inject equivalent for non-dynamic crawler)
* `process` (process the result of scraping a single page)
* `waitFor` register a global procedure for something to wait to happen before trying to scrape
* `config` (change the task's settings)
* `add` add url to the task
* `plug` plug external behaviour
* `before` register a middleware before the tasks starts
* `beforeScraping` register a middleware before the scraping in the lifecycle
* `afterScraping` register a middleware after the scraping in the lifecycle
* `on` adding a listener to the task's event emitter
* `then` (callback for global task succes)

### Page object handle

The various page-level callbacks should provide a useful object to the user. This one should contain the following:

* meta of the page (url, status etc. index in the iteration)
* passed request (basically just an url but, why not, the provided object)
* scraped data

**Note** - should we try to switch to standard node.js pattern [err, results] concerning the global outcome of a task?