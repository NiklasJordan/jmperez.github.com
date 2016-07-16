---
layout: post
title: Some things I learned while implementing data fetching for universal web apps
date: 2016-07-16 19:40:00+02:00
description: Tips when fetching data in an isomorphic/universal app.
---

Javascript is not just a language for the browser. Node.JS is becoming popular as a platform to run servers in JS, which combined with more modular JS applications makes it possible to write universal web apps. But what are universal apps? And why we should care about data fetching?

## Universal/Isomorphic apps

Developers haven't decided what to call them yet, but for the sake of consistency I'll be referring to them as _universal_ web apps. These projects have 2 main features:

- The server runs JS
- Most of the code of the project is shared between browser and server.

For a long time we tangled business logic with markup. One would generate some HTML in the server and JS would run in the client to make AJAX requests and changes in the DOM. This worked for most websites, but today lots of projects start as a SPA, where state is managed in a single place and the server acts merely to provide the initial page and a set of endpoints that provide data. But this approach has caused [long page load times](https://blog.twitter.com/2012/improving-performance-on-twittercom) and bad SEO forced rethinking the web.

The best way to provide fast-loading pages is to server-side render them. Though service workers might help eventually, the first request will still need to make all the way through to the server. In order to compose the page, the server needs to identify the user, fetch some data and generate the markup through some template system. After the initial load, JS comes in and replaces full page loads with some data fetching and markup composition.

Here we can see that some code will be duplicated, especially in SPAs. A way to improve this is by running JS in the server and trying to write code that is decoupled from the DOM. Server-side rendering can be seen these days as an enhancement for SPAs.

## Data fetching in universal apps

In universal apps, most of the code can be shared between browser and server. One of the pieces that differs is data fetching. Node.JS has its own way to make requests, and neither `XMLHttpRequest` nor `fetch` is supported.

There are many request libraries that will perform the request properly, using whatever is supported in the platform running the script. Some of them try to polyfill `fetch` in Node, others polyfill `http` in the browser. In our code we `require`/`import` them, and then the code is included in the bundle served to the browser using browserify or webpack.

### Choosing a universal request package

Regardless of which request library one you choose, **be aware of how they contribute to the bundle size**. Ideally, these libraries should represent a rather small chunk of the bundle, but I have been doing some tests and the results are pretty interesting.

I have used browserify to create a bundle just requiring each of there libraries separately, and then uglified the output. The size shown is minified, not gzipped:

- [iso-http 0.0.5](https://github.com/jedmao/iso-http): 4kB
- [isomorphic-fetch 2.2.1](https://github.com/matthew-andrews/isomorphic-fetch): 9kB
- [superagent 2.0.0](https://github.com/visionmedia/superagent): 9kB
- [axios 0.13.0](https://github.com/mzabriskie/axios): 20kB
- [isomorphic-request 1.0.0](https://registry.npmjs.org/isomorphic-request): 240kB
- [node-fetch 1.5.3](https://github.com/bitinn/node-fetch): 489kB

Note that these packages might not be equivalent in terms of features, and I haven't tried most of them myself. In some cases it might be worth it creating your custom request library that only supports the needed functionality to access the endpoints you are using. But all in all these libraries should generate code that wraps `XMLHttpRequest` and its response.

Notice the big difference in size between them, all the way from 4kB to 489kB. How can that be? For that we need to understand what is going on when our code is bundled.

### Bundling of Node's APIs

Something I learnt while creating those bundles is that it is very useful to double check the generated code. If the bundler sees that the code is using some feature only available in Node, it will include the JS code for it in the bundle.

An example is `Buffer`, which [I was using in a wrapper for the Spotify Web API](https://github.com/thelinmichael/spotify-web-api-node/blob/cf9b5834b828b38b659afd82fb85ae742d5ea0eb/src/spotify-web-api.js#L1241). Since `Buffer` is not supported in the browser environment, which accounts to 44kB of JS (minified).

Same thing can happen with these request libraries when they include code that is supposed to be only run in Node.JS and not on the browser.

### The `browser` field

When you are bundling some JS code that is going to be run only on the server, do not include it in the bundle that is going to be run on the browser. This applies both if you are writing a site and if you are writing a library that will be used by another project.

For this, use the [`browser` field](https://github.com/defunctzombie/package-browser-field-spec) in the `package.json` file. It is used, amongst others, to define what file needs to be replaced by other when bundling. When using webpack, make sure [`target`](https://webpack.github.io/docs/configuration.html#target) is not set or set to `'web'`.

You can see an example [in the superagent source code](https://github.com/visionmedia/superagent/blob/83892f35fe15676a4567a0eb51eecd096939ad36/package.json#L54):

{% highlight javascript %}
{
  "name": "superagent",
  ...
  "browser": {
    "./lib/node/index.js": "./lib/client.js",
    ...
  },
  ...
  "main": "./lib/node/index.js",
  ...
}
{% endhighlight %}

or [iso-http](https://github.com/jedmao/iso-http/blob/33870ab0ee79d93e73ad55d787c99c5f5e07f936/package.json#L6):

{% highlight javascript %}
{
  "name": "iso-http",
  ...
  "main": "js/node/Http.js",
  "browser": "js/browser/Http.js",
  ...
}
{% endhighlight %}

From the libraries I tested, 4 of them were using `browser` and 2 of them not. These 2 are precisely the ones that generate the largest bundle files.

### Getting stats about a bundle

It is not straightforward to know what modules are contributing the most size to the output bundle. One way you can try is by creating an entry JS file that only includes the module you are interested in, which is what I did when comparing request libraries. But if you are using webpack, a better way is to use [webpack-visualizer](https://github.com/chrisbateman/webpack-visualizer).

This tool generates a pie chart with all the required dependencies and makes it straightforward to find the culprit.

![iconv-lite and pako are the main contributors to node-fetch bundle size]({{ site.url }}/assets/images/posts/node-fetch-webpack-stats.png)
_^ iconv-lite and pako are the main contributors to node-fetch bundle size_

### One more thing: ES6 imports

A great way to reduce the footprint of a library is by exporting it in pieces. Thanks to ES6 and tree-shaking, one can import a subset of functions and symbols from a library, which can reduce largely the size of the output bundle.

This is really useful if you are implementing a wrapper for your API endpoints. It is convenient to support all the endpoints, but an app might only be interesting in using a few of those.

I have started experimenting with this building [a wrapper for the Spotify Web API](https://github.com/JMPerez/spotify-web-api-js-poc) that exposes each endpoint as an exported function, so that the unused endpoints are not part of the generated code.

## Conclusion

Universal apps are great, and so is sharing lots of code between server and client. As important as a fast rendering time is the time it takes the browser to download, parse and execute the JS that provides the client-side functionality. Keep an eye on your imports, and prevent including code that is not intended to be run on the browser.