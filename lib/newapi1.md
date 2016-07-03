# jsdom New API Docs

We are currently attempting to build a new jsdom API which is simpler and more intuitive than the previous one. For now, you can acquire it via

```js
const jsdom = require("jsdom/lib/newapi1");
```

Like all parts of the public jsdom API, this will be stable for the duration of a major version. Eventually, when we feel the new API has reached parity with the previous one, we'll switch it over to being the default.

## Basic usage

To use jsdom, just pass it a string. You will get back a `JSDOM` object, which has a number of useful properties, notably `window`:

```js
const jsdom = require("jsdom/lib/newap1");

const dom = jsdom(`<!DOCTYPE html><p>Hello world</p>`);
console.log(dom.window.document.querySelector("p").textContent); // "Hello world"
```

(Note that jsdom will parse the HTML you pass it just like a browser does, including implied `<html>`, `<head>`, and `<body>` tags.)

The resulting object is an instance of the `JSDOM` class, which contains a number of useful properties and methods besides `window`. In general it can be used to act on the jsdom from the "outside," doing things that are not possible with the normal DOM APIs. For simple cases, where you don't need any of this functionality, we recommend a coding pattern like

```js
const window = jsdom(`...`).window;
// or even
const document = jsdom(`...`).window.document;
```

Full documentation on everything you can do with the `JSDOM` class is below, in the section "`JSDOM` Object API".

## Customizing jsdom

The `jsdom` function accepts a second parameter which can be used to customize your jsdom in the following ways.

### Simple Options

```js
const dom = jsdom(``, {
  url: "https://example.org/",
  referrer: "https://example.com/",
  contentType: "text/html",
  userAgent: "TODO: replace this with the weird browsernator example from the spec",
  includeNodeLocations: true
});
```

- `url` sets the value returned by `window.location`, `document.URL`, and `document.documentURI`, and affects things like resolution of relative URLs within the document and the same-origin restrictions and referrer used while fetching external resources. It defaults to `"about:blank"`.
- `referrer` just affects the value read from `document.referrer`. It defaults to `"about:blank"`.
- `contentType` affects the value read from `document.contentType`, and how the document is parsed: as HTML or as XML. Values that are not `"text/html"` or an [XML mime type](https://html.spec.whatwg.org/multipage/infrastructure.html#xml-mime-type) will throw. It defaults to `"text/html"`.
- `userAgent` affects the value read from `navigator.userAgent`, as well as the `User-Agent` header sent while fetching external resources. It defaults to <code>\`Mozilla/5.0 (${process.platform}) AppleWebKit/537.36 (KHTML, like Gecko) jsdom/${jsdomVersion}\`</code>.
- `includeNodeLocations` preserves the location info produced by the HTML parser, allowing you to retrieve it with the `nodeLocation()` method (described below). It defaults to `false` to give the best performance, and cannot be used with an XML content type since our XML parser does not support location info.

Note that both `url` and `referrer` are canonicalized before they're used, so e.g. if you pass in `"https:example.com"`, jsdom will interpret that as if you had given `"https://example.com/"`. If you pass an unparseable URL, the call will throw. (URLs are parsed and serialized according to the [URL Standard](http://url.spec.whatwg.org/).)

### Executing Scripts

jsdom's most powerful ability is that it can execute scripts inside the jsdom. These scripts can modify the content of the page and access all the web platform APIs jsdom implements.

However, this is also highly dangerous when dealing with untrusted content. The jsdom sandbox is not foolproof, and code running inside the DOM's `<script>`s can, if it tries hard enough, get access to the Node environment, and thus to your machine. As such, the ability to execute scripts embedded in the HTML is disabled by default:

```js
const dom = jsdom(`<body>
  <script>document.body.appendChild(document.createElement("hr"));</script>
</body>`);

// The script will not be executed, by default:
dom.window.document.body.children.length === 1;
```

To enable executing scripts inside the page, you can use the `runScripts: "dangerously"` option:

```js
const dom = jsdom(`<body>
  <script>document.body.appendChild(document.createElement("hr"));</script>
</body>`, { runScripts: "dangerously" });

// The script will be executed and modify the DOM:
dom.window.document.body.children.length === 2;
```

Again we emphasize to only use this when feeding jsdom code you know is safe. If you use it on arbitrary user-supplied code, or code from the Internet, you are effectively running untrusted Node.js code, and your machine could be compromised.

If you are simply trying to execute script "from the outside", instead of letting `<script>` elements (and inline event handlers) run "from the inside", you can use the `runScripts: "outside-only"` option, which enables `window.eval`:

```js
const window = jsdom(``, { runScripts: "outside-only" }).window;

window.eval(`document.body.innerHTML = "<p>Hello, world!</p>";`);
window.document.body.children.length === 1;
```

This is turned off by default for performance reasons, but is safe to enable.

Note that we strongly advise against trying to "execute scripts" by mashing together the jsdom and Node global environments (e.g. by doing `global.window = dom.window`), and then executing scripts or test code inside the Node global environment. Instead, you should treat jsdom like you would a browser, and run all scripts and tests that need access to a DOM inside the jsdom environment, using `window.eval` or `{ runScripts: "dangerously" }`. This might require, for example, creating a browserify bundle to execute as a `<script>` element—just like you would in a browser.

### Virtual Consoles

Like web browsers, jsdom has the concept of a "console". This records both information directly sent from the page, via scripts executing inside the document, as well as information from the jsdom implementation itself. We call the user-controllable console a "virtual console", to distinguish it from the Node.js `console` API and from the inside-the-page `window.console` API.

By default, the `jsdom` function will return a `JSDOM` instance with a virtual console that forwards all its output to the Node.js console. To create your own virtual console and pass it to jsdom, you can override this default by doing

```js
const virtualConsole = new jsdom.VirtualConsole();
const dom = jsdom(``, { virtualConsole });
```

Code like this will create a virtual console with no behavior. You can give it behavior by adding event listeners for all the possible console methods:

```js
virtualConsole.on("error", () => { ... });
virtualConsole.on("warn", () => { ... });
virtualConsole.on("info", () => { ... });
virtualConsole.on("dir", () => { ... });
// ... etc. See https://console.spec.whatwg.org/#logging
```

(Note that it is probably best to set up these event listeners *before* calling `jsdom()`, since errors or console-invoking script might happen during parsing.)

If you simply want to redirect the virtual console output to another console, like the default Node.js one, you can do

```js
virtualConsole.sendTo(console);
```

There is also a special event, `"jsdomError"`, which will fire with error objects to report errors from jsdom itself. This is similar to how error messages often show up in web browser consoles, even if they are not initiated by `console.error`. So far, the following errors are output this way:

- Errors loading or parsing external resources (scripts, stylesheets, frames, and iframes)
- Script execution errors that are not handled by a window `onerror` event handler that returns `true` or calls `event.preventDefault()`
- Not-implemented errors resulting from calls to methods, like `window.alert`, which jsdom does not implement, but installs anyway for web compatibility

If you're using `sendTo(console)` to send errors to `console`, by default it will call `console.error` with information from `"jsdomError"` events. If you'd prefer to maintain a strict one-to-one mapping of events to method calls, and perhaps handle `"jsdomError"`s yourself, then you can do

```js
virtualConsole.sendTo(console, { omitJsdomErrors: true });
```

### Cookie Jars

Like web browsers, jsdom has the concept of a cookie jar, storing HTTP cookies. Cookies that have a URL on the same domain as the document, and are not marked HTTP-only, are accessible via the `document.cookie` API. Additionally, all cookies in the cookie jar will impact the fetching of external resources.

By default, the `jsdom` function will return a `JSDOM` instance with an empty cookie jar. To create your own cookie jar and pass it to jsdom, you can override this default by doing

```js
const cookieJar = new jsdom.CookieJar(store, options);
const dom = jsdom(``, { cookieJar });
```

This is mostly useful if you want to share the same cookie jar among multiple jsdoms, or prime the cookie jar with certain values ahead of time.

Cookie jars are provided by the [tough-cookie](https://www.npmjs.com/package/tough-cookie) package. The `jsdom.CookieJar` constructor is a subclass of the tough-cookie cookie jar which by default sets the `looseMode: true` option, since that [matches better how browsers behave](https://www.npmjs.com/package/tough-cookie). If you want to use tough-cookie's utilities and classes yourself, you can use the `jsdom.toughCookie` export to get access to the tough-cookie module instance packaged with jsdom.

### Intervening Before Parsing

jsdom allows you to intervene in the creation of a jsdom very early: after the `Window` and `Document` objects are created, but before any HTML is parsed to populate the document with nodes:

```js
const dom = jsdom(`<p>Hello</p>`, {
  beforeParse(window) {
    window.document.childNodes.length === 0;
    window.someCoolAPI = () => { /* ... */ };
  }
});
```

This is especially useful if you are wanting to modify the environment in some way, for example adding shims for web platform APIs jsdom does not support.

## `JSDOM` Object API

Once you have called the `jsdom()` function, you'll get back a `JSDOM` object with the following capabilities.

### Properties

The property `window` retrieves the `Window` object that you created.

The properties `virtualConsole` and `cookieJar` reflect the options you pass in, or the defaults created for you if nothing was passed in for those options.

### Serializing the document with `serialize()`

The `serialize()` method will return the [HTML serialization](https://html.spec.whatwg.org/#html-fragment-serialisation-algorithm) of the document, including the doctype:

```js
const dom = jsdom(`<!DOCTYPE html>hello`);

dom.serialize() === "<!DOCTYPE html><html><head></head><body>hello</body></html>";

// Contrast with:
dom.window.document.documentElement.outerHTML === "<html><head></head><body>hello</body></html>";
```

### Getting the source location of a node with `nodeLocation(node)`

The `nodeLocation` method will find where a DOM node is within the source document, returning the [parse5 location info](https://www.npmjs.com/package/parse5#options-locationinfo) for the node:

```js
const dom = jsdom(`<p>Hello
    <img src="foo.jpg">
  </p>`);

const document = dom.window.document;
const bodyEl = document.body; // implicitly created
const pEl = document.querySelector("p");
const textNode = pEl.firstChild;
const imgEl = document.querySelector("img");

console.log(dom.nodeLocation(bodyEl));   // null; it's not in the source
console.log(dom.nodeLocation(pEl));      // { start: 0, end: 39, startTag: ..., endTag: ... }
console.log(dom.nodeLocation(textNode)); // { start: 3, end: 13 }
console.log(dom.nodeLocation(imgEl));    // { start: 13, end: 32 }
```

### Reconfiguring the jsdom with `reconfigure(settings)`

The `top` property on `window` is marked `[Unforgeable]` in the spec, meaning it is a non-configurable own property and thus cannot be overridden or shadowed by normal code running inside the jsdom, even using `Object.defineProperty`.

Similarly, at present jsdom does not handle navigation (such as setting `window.location.href === "https://example.com/"`); doing so will cause the virtual console to emit a `"jsdomError"` explaining that this feature is not implemented, and nothing will change: there will be no new `Window` or `Document` object, and the existing `window`'s `location` object will still have all the same property values.

However, if you're acting from outside the window, e.g. in some test framework that creates jsdoms, you can override both of these using the special `reconfigure()` method:

```js
const dom = jsdom();

dom.window.top === dom.window;
dom.window.location.href === "about:blank";

dom.reconfigure({ top: myFakeTopForTesting, url: "https://example.com/" });

dom.window.top === myFakeTopForTesting;
dom.window.location.href === "https://example.com/";
```

Note that changing the jsdom's URL will impact all APIs that return the current document URL, such as `window.location`, `document.URL`, and `document.documentURI`, as well as resolution of relative URLs within the document, and the same-origin checks and referrer used while fetching external resources.

## Future New API Work

The New API is definitely not considered finished. In addition to responding to feedback based on your experience, we plan on adding the following functionality:

- A `dom.evalVMScript` API, similar to the one recently added to jsdom, but much better tested, with proper error handling
- A new custom resource loader loader infrastructure and the ability to enable external resource loads. Tenative plan:

  ```js
  const dom = jsdom(``, {
    resources: {
      allowed: ["script#foo", "iframe", `link[rel="stylesheet"]`] // selector-based filtering. But what about XHR??
      fetch({ url, cookie, referrer, defaultFetch }) {
        // return a promise
      }
    }
  });
  ```
- Promise-returning convenience methods, `jsdom.fromFile(filename, options)` and `jsdom.fromURL(url, options)` which will do the appropriate file-reading and fetching for you.
  - `jsdom.fromURL()` will likely not support much customization (such as the current `headers` option). Instead, you'll be urged to make your own requests and use `jsdom()`.
- Fetching configuration, for parity with the current `pool`, `agent`, `agentClass`, `agentOptions`, `strictSSL`, and `proxy` options.
- Miscellaneous options, such as `concurrentNodeIterators`.
- Accept `Buffer`s, typed arrays, and DataViews, along with an `encoding` option, to allow decoding binary data and setting the document's encoding. `encoding` is optional; if not present we default to scanning the bytes for a meta charset. (Strings-as-input stay utf-8 as the charset.)
- Speculative additional API ideas:
  - A `dom.loaded` promise that is fulfilled alongside the window's `"load"` event
  - `jsdom.fragment(html, options)` which returns a `DocumentFragment` resulting from parsing the HTML. (It is essentially equivalent to ``jsdom(`<template>${html}</template>`, options).window.document.body.firstChild.content``.)
  - `jsdom.jQuery(html, options)` which gives you back a `$` function for operating on the resulting DOM, similar to Cheerio.
  - `dom.insertScriptFromURL(url)` and `dom.insertScriptFromFile(filename)` (promise-returning)