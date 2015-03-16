# Shed

> A collection of tools for [service workers](https://slightlyoff.github.io/ServiceWorker/spec/service_worker/)

## Service Worker helpers

Shed provides some simple helpers for use in creating your own service workers. If you're not sure what service workers are or what they are for, start with [the explainer doc](https://github.com/slightlyoff/ServiceWorker/blob/master/explainer.md).

### Installing Shed

Shed is available through Bower, npm or direct from github:

`bower install --save shed`

`npm install --save shed`

`git clone https://github.com/wibblymat/shed.git`

### Registering your service worker

From your registering page, register your service worker in the normal way. For example:

```javascript
navigator.serviceWorker.register('my-service-worker.js', {scope: '/'});
```

For even lower friction, if you don't intend to doing anything more fancy than just registering with a default scope, you can instead include the Shed companion script in your HTML:

```html
<script src="/path/to/shed/companion.js" data-service-worker="my-service-worker.js"></script>
```

As currently implemented in Chrome 40, a service worker must exist at the root of the scope that you intend it to control, or higher. So if you want all of the pages under `/myapp/` to be controlled by the worker, the worker script itself must be served from either `/` or `/myapp/`. The default scope is the containing path of the service worker script.

### Using Shed in your worker script

In your service worker you just need to use `importScripts` to load Shed

```javascript
importScripts('bower_components/shed/shed.js'); // Update path to match your own setup
```

## Basic usage
Within your service worker file
```javascript
// Set up routes from URL patterns to request handlers
shed.router.get('/myapp/index.html', someHandler);

// For some common cases Shed provides a built-in handler
shed.router.get('/', shed.networkFirst);

// URL patterns are the same syntax as ExpressJS routes
// (http://expressjs.com/guide/routing.html)
shed.router.get(':foo/index.html', function(request, values) {
  return new Response('Handled a request for ' + request.url +
      ', where foo is "' + values.foo + '");
});

// For requests to other origins, specify the origin as an option
shed.router.post('/(.*)', apiHandler, {origin: 'https://api.example.com'});

// Provide a default handler
shed.router.default = myDefaultRequestHandler;

// You can provide a list of resources which will be cached at service worker install time
shed.precache(['/index.html', '/site.css', '/images/logo.png']);
```

## Request handlers
A request handler receives three arguments

```javascript
var myHandler = function(request, values, options) {
  // ...
}
```

- `request` - [Request](https://fetch.spec.whatwg.org/#request) object that triggered the `fetch` event
- `values` - Object whose keys are the placeholder names in the URL pattern, with the values being the corresponding part of the request URL. For example, with a URL pattern of `'/images/:size/:name.jpg'` and an actual URL of `'/images/large/unicorns.jpg'`, `values` would be `{size: 'large', name: 'unicorns'}`
- `options` - the options object that was used when [creating the route](#api)

The return value should be a [Response](https://fetch.spec.whatwg.org/#response), or a [Promise](http://www.html5rocks.com/en/tutorials/es6/promises/) that resolves with a Response. If another value is returned, or if the returned Promise is rejected, the Request will fail which will appear to be a [NetworkError](https://developer.mozilla.org/en-US/docs/Web/API/DOMException#exception-NetworkError) to the page that made the request.

### Built-in handlers

There are 5 built-in handlers to cover the most common network strategies. For more information about offline strategies see the [Offline Cookbook](http://jakearchibald.com/2014/offline-cookbook/).

#### `shed.networkFirst`
Try to handle the request by fetching from the network. If it succeeds, store the response in the cache. Otherwise, try to fulfill the request from the cache. This is the strategy to use for basic read-through caching. Also good for API requests where you always want the freshest data when it is available but would rather have stale data than no data.

#### `shed.cacheFirst`
If the request matches a cache entry, respond with that. Otherwise try to fetch the resource from the network. If the network request succeeds, update the cache. Good for resources that don't change, or for which you have some other update mechanism.

#### `shed.fastest`
Request the resource from both the cache and the network in parallel. Respond with whichever returns first. Usually this will be the cached version, if there is one. On the one hand this strategy will always make a network request, even if the resource is cached. On the other hand, if/when the network request completes the cache is updated, so that future cache reads will be more up-to-date.

#### `shed.cacheOnly`
Resolve the request from the cache, or fail. Good for when you need to guarantee that no network request will be made - to save battery on mobile, for example.

#### `shed.networkOnly`
Handle the request by trying to fetch the URL from the network. If the fetch fails, fail the request. Essentially the same as not creating a route for the URL at all.

## API

### Global Options
Any method that accepts an `options` object will accept a boolean option of `debug`. When true this causes Shed to output verbose log messages to the worker's console.

Most methods that involve a cache (`shed.cache`, `shed.uncache`, `shed.fastest`, `shed.cacheFirst`, `shed.cacheOnly`, `shed.networkFirst`) accept an option called `cache`, which is the **name** of the [Cache](https://slightlyoff.github.io/ServiceWorker/spec/service_worker/#cache) that should be used. If not specifed Shed will use a default cache.

### `shed.router.get(urlPattern, handler, options)`
### `shed.router.post(urlPattern, handler, options)`
### `shed.router.put(urlPattern, handler, options)`
### `shed.router.delete(urlPattern, handler, options)`
### `shed.router.head(urlPattern, handler, options)`
Create a route that causes requests for URLs matching `urlPattern` to be resolved by calling `handler`. Matches requests using the GET, POST, PUT, DELETE or HEAD HTTP methods respectively.

- `urlPattern` - an Express style route. See the docs for the [path-to-regexp](https://github.com/pillarjs/path-to-regexp) module for the full syntax
- `handler` - a request handler, as [described above](#request-handlers)
- `options` - an object containing options for the route. This options object will be available to the request handler. The `origin` option is specific to the route methods, and is an exact string or a Regexp against which the origin of the Request must match for the route to be used.

### `shed.router.any(urlPattern, handler, options)`
Like `shed.router.get`, etc., but matches any HTTP method.

### `shed.router.default`
If you set this property to a function it will be used as the request handler for any request that does not match a route.

### `shed.precache(arrayOfURLs)`
Add each URL in arrayOfURLs to the list of resources that should be cached during the service worker install step. Note that this needs to be called before the install event is triggered, so you should do it on the first run of your script.

### `shed.cache(url, options)`
Causes the resource at `url` to be added to the cache. Returns a Promise.

### `shed.uncache(url, options)`
Causes the resource at `url` to be removed from the cache. Returns a Promise.
