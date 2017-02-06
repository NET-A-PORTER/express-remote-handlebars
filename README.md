# Express Remote Handlebars

[![Version](https://img.shields.io/npm/v/express-remote-handlebars.svg)](https://www.npmjs.com/package/express-remote-handlebars)
[![License](https://img.shields.io/npm/l/express-remote-handlebars.svg)](https://www.npmjs.com/package/express-remote-handlebars)
[![Build Status](https://travis-ci.org/NET-A-PORTER/express-remote-handlebars.svg?branch=master)](https://travis-ci.org/NET-A-PORTER/express-remote-handlebars)

[Handlebars][] view engine for [Express][] which transparently integrates remotely stored templates
into your rendering flow.

This is useful in an SOA environment where separate applications are responsible for rendering
different parts of a site while still requiring one shared layout.

[Express]: https://github.com/strongloop/express
[Handlebars]: https://github.com/wycats/handlebars.js
[request]: https://github.com/request/request


## Installation

Install using npm:

```shell
$ npm install --save express-remote-handlebars
```


## Tests

Run tests using npm:

```shell
$ npm test
```


## Usage

### Simple App

A simple application will look as follows:

```javascript
var express = require('express');
var remoteHandlebars = require('express-remote-handlebars');
var app = express();

app.engine('handlebars', remoteHandlebars({ layout: 'http://localhost/template.handlebars', cacheControl: 'max-age=600, stale-while-revalidate=86400' }));
app.set('view engine', 'handlebars');

app.get('/', function (req, res) {
    res.render('index');
});
```

In the example above, when `res.render()` is called the view engine will:

* Get view and layout from cache if available - Stale items are revalidated in the background (configured using `stale-while-revalidate`)
* Otherwise read `views/index.handlebars` from disk and download `http://localhost/template.handlebars`
* Cache local view forever and remote layout for 10 minutes (configured using `max-age`)
* Render view and layout (view will be inserted in `{{{content}}}` placeholder which is configurable using `options.placeholder`)
* Send response


### Overriding Defaults

You can override instance specific defaults using the options param of `res.render()`:

```javascript
var express = require('express');
var remoteHandlebars = require('express-remote-handlebars');
var app = express();

app.engine('handlebars', remoteHandlebars({ layout: 'http://localhost/template.handlebars' }));
app.set('view engine', 'handlebars');

app.get('/', function (req, res) {
    res.render('index'); // Will be rendered with default layout
});

app.get('/users/:id', function (req, res) {
    res.render('post', {
        layout: false // Will be rendered without a layout
    });
});
```

See [documentation](#render-filepath-options-callback) for all overridable options. 


### Request in Parallel

In most applications you will get data from a database or REST API before rendering your views. 

In order to minimize response times you can fetch layouts together with any other async request
using `getLayout()`:

```javascript
var express = require('express');
var remoteHandlebars = require('express-remote-handlebars').create({ cacheControl: 'max-age=60, stale-while-revalidate=600' });
var app = express();

app.engine('handlebars', remoteHandlebars.engine);
app.set('view engine', 'handlebars');

app.get('/users/:id', function (req, res, next) {
    async.parallel([
        function (callback) {
            // Get user
            User.findOne({ id: req.params.id }, callback);
        },
        function (callback) {
            // Fetch layout
            remoteHandlebars.getLayout('http://localhost/template.handlebars', callback);
        }
    ], function (error, results) {
        if (error) return next(error);
        res.render('index', {
            user: results[0], // User data from database
            layout: results[1] // Compiled Handlebars template from getLayout()
        });
    });
});
```


## Documentation

* [RemoteHandlebars](#remotehandlebars-options)
* [render](#render-filepath-options-callback)
* [getLayout](#getlayout-url-callback)
* [getView](#getview-filepath-callback)
* [getPartials](#getpartials-partialsdir-callback)


### RemoteHandlebars (options)

Constructor to instantiate a new view engine.

##### Arguments

* `options.cacheControl` - Cache control string (default: `max-age=60, stale-while-revalidate=600`)

    Use `max-age=<seconds>` to define when the item will expire.
    
    Combine with `stale-while-revalidate=<seconds>` to set the time window after `max-age` has expired in which the item is
    marked as stale but still usable. After both `max-age` and `stale-while-revalidate` have expired the item will be
    deleted from cache.

    Examples:

    * `no-cache, no-store, must-revalidate` - Will be dropped out of cache immediately
    * `max-age=600, must-revalidate` - Will be cached for 10 minutes and then dropped out of cache
    * `max-age=600, stale-while-revalidate=86400` - Will be cached for 10 minutes and then revalidated in the background if
      the item is accessed again within a time window of 1 day

* `options.helpers` - Object with custom helper functions
* `options.layout` - URL or [request object][request] of layout template (default: false)
* `options.partialsDir` - Path(s) to partials (default: views/partials/)
* `options.placeholder` - Name of content placeholder in layout (default: content)
* `options.request (options, callback)` - Function used to request templates *(Default: [request](https://github.com/request/request))*
* `options.size` - Maximum number of layouts to cache (default: Infinity)

##### Examples

Simple:

```javascript
var remoteHandlebars = require('express-remote-handlebars');
app.engine('handlebars', remoteHandlebars(options));
```

Factory:

```javascript
var remoteHandlebars = require('express-remote-handlebars').create(options);
app.engine('handlebars', remoteHandlebars.engine);
```

Constructor:

```javascript
var RemoteHandlebars = require('express-remote-handlebars').RemoteHandlebars;
app.engine('handlebars', new RemoteHandlebars(options).engine);
```

_Note: Simple syntax only exposes the view engine. Factory and constructor pattern give access to public methods for more advanced use cases._

---


### render (filePath, options, callback)

Called by [Express][] when running `res.render()`. 

##### Arguments

* `filePath` - Path to template
* `options` - Context for template (Merged with `app.locals` and `res.locals`)
* `options.cache` - Toggle caching (This is set by [Express][] via `app.enable('view cache')` but can also be overridden manually)
* `options.helpers` - Object with custom helper functions
* `options.layout` - URL, [request object][request] or template function
* `options.partialsDir` - Path(s) to partials
* `callback (error, rendered)` - Called once view with layout has been fully rendered

---


### getLayout ([url, options], callback)

Fetches and compiles a template from remote. 

Template is *temporarily* cached (see `max-age`, `stale-while-revalidate` and `size`) unless disabled. 

##### Arguments

* `url` - URL or [request object][request] of layout template (optional, default: `this.layout`)
* `options.cache` - Toggle caching (optional, default: true)
* `callback (error, template)` - Called once template has been fetched and compiled

---


### getView (filePath, [options], callback)

Reads and compiles a template from disk. 

Template is cached *forever* unless disabled. 

##### Arguments

* `filePath` - Path to template
* `options.cache` - Toggle caching (optional, default: true)
* `callback (error, template)` - Called once template has been read and compiled

---


### getPartials ([partialsDir, options], callback)

Recursively finds and compiles all partials in a directory. 

Partials are cached *forever* unless disabled. 

##### Arguments

* `partialsDir` - Path(s) to partials (optional, default: `this.partialsDir`)
* `options.cache` - Toggle caching (optional, default: true)
* `callback (error, partials)` - Called once partials have been read and compiled

