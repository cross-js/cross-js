# Why should I use CrossJS style guide?

Adopting CrossJS style means your javascript can work in any environment without being dependent on any core browser/node js api and can work in any context just as it's without increasing the bundled size to much. This might not make sense for 100% of projects and development cultures. This rules certainly don't apply if you are only targeting one platform that has the api built in.

# For References

By only including [readable-stream](https://www.npmjs.com/package/readable-stream) for browser without anything else you have included the buffer, events, string_decoder and inherits module among many more smaller modules and already broken 4 of this rules and increased your bundle to:

|                | gzip    | uncompressed |
| -------------- | ------- | ------------ |
| **unminified** | 43.73KB | 174.77KB     |
| **minified**   | 19.79KB | 67.93KB      |

**Some Read Up:**

- [JavaScript Bootup Time Is Too High](https://developers.google.com/web/tools/lighthouse/audits/bootup)
- [JavaScript Start-up Optimization](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/)

Summary: Javascript is now the highest performance hit on many websites... One large image is able to load faster than js. I also recommend that you try out [lighthouse](https://chrome.google.com/webstore/detail/lighthouse/blipmdconlkpinefehnmjammfjpmpbjk) and perform a performance test.
If you follow this rule you might be able to write code with just a fraction of what you otherwise would need.

# CrossJS — The Rules

- [Don't use Buffer](https://github.com/cross-js/cross-js#dont-use-buffer)
- [Don't use EventEmitter or EventTarget](https://github.com/cross-js/cross-js#dont-use-eventemitter-or-eventtarget)
- [Don't create Node or Web readable Stream yourself](https://github.com/cross-js/cross-js#dont-create-node-or-web-readable-stream-yourself)
- [Don't use any ajax/request library](https://github.com/cross-js/cross-js#dont-use-any-ajaxrequest-library)
- [Don't use node's Url or querystring](https://github.com/cross-js/cross-js#dont-use-nodes-url-or-querystring)
- [Don't use node's string_decoder](https://github.com/cross-js/cross-js#dont-use-nodes-string_decoder)
- [Don't use inherits](https://github.com/cross-js/cross-js#dont-use-inherits)
- [Don't use if-else platform specific code inside functions](https://github.com/cross-js/cross-js#dont-use-if-else-platform-specific-inside-functions)
- [Don't depend of things that would make your application crash in another context](https://github.com/cross-js/cross-js#dont-depend-of-things-that-would-make-your-application-crash-in-another-context)
- [Don't use anything else then javascript](https://github.com/cross-js/cross-js#dont-use-anything-else-then-javascript)
- [Don't use cancelable promises](https://github.com/cross-js/cross-js#dont-use-cancelable-promises)

## Don't use Buffer

#### Why?

`Uint8Array` and `Buffer` have very much in common and are very similar to each other.
Adding `buffer` will increase your bundle size a lot. And the fact that buffer inherits from
`Uint8Array` have made recent Node.js core api's acceptable to typed arrays. For example: `fs.writeFile()` used to only accept a Buffer but now works with both typed arrays and buffers.

Following this rule doesn't mean you have to convert all buffers you receive from node's core api and other modules from buffer to UInt8Array, just treat the buffer as a Uint8array instead since buffer inherits from it.

#### How then?

Use [Uint8Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array) and [DataView](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView)

```js
// ✗ avoid
var chunk = Buffer.from(source)
var chunk = Buffer.alloc(n)
var chunk = Buffer.allocUnsafe(n)
var chunk = new Buffer(source)

// ✓ ok
// Buffer allocation
var chunk = new Uint8Array(n)

// Buffer from An array-like or iterable object to convert to a typed array.
var chunk = Uint8Array.from(source[, mapFn[, thisArg]])

// Buffer from string
var chunk = new TextEncoder().encode('abc')

// Buffer from base64 (minimalist, there are synchronous way to, try avoiding base64 in the first place)
var arrayBuffer = await fetch(`data:;base64,${string}`).then(r => r.arrayBuffer())
var chunk = new Uint8Array(arrayBuffer)
```

## Don't use [EventEmitter](https://nodejs.org/api/events.html) or [EventTarget](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget)

#### Why?

Don't take this seriously, sometimes it can be good to have more then one listener of one type registered. Also if you need something that can bubble up & down. IMHO I think that using events will increase the complexity of some application. It could certainly be avoided by other means without depending on other modules. In the end it will just increase the bundle size. Use them if it makes sense.

All I'm saying is:<br>
Think twice before you decide to use them and if you really need it.<br>
There is more than one way to skin a cat<br>

- Including [event](https://nodejs.org/api/events.html) comes with a bundle cost.
- Having EventEmitter and EventTarget gives a mixed api and have no uniformed api across Node and Browsers.
- Extending EventTarget is not so cross browser compatible either yet.
  - So you have to include a polyfill for this also.
- Also, how often do you need to subscribe to some event more than twice?

Often you know all the event you want to subscribe to beforehand, so why not just pass those down in the constructor or the function instead

#### How then?

```js
// ✗ avoid
const EventEmitter = require('event')
class Foo extends EventEmitter {}
class Bar extends EventTarget {}

source.on(event, fn)
source.once(event, fn)
source.addEventListener(event)
source.addEventListener(event, { once: true })

source.dispatchEvent(event)
source.emit(event)

// Some suggestion:

// ✓ ok (using setter/getter)
class Foo {
  get onmessage() {...}
  set onmessage(x) {...}
}

// ✓ ok (passing in the events you want to subscribe to)
class Bar {
  constructor(opts) {
    this.opts = opts
  }
  somethingHappens() {
    // Bail early, no need to continue
    if (!this.opts.progress) return

    const totalDownload = havyCalculation(all_requests)
    this.opts.progress(totalDownload)
  }
}

const bar = new Bar({
  onData (...args) {...},
  onError (...args) {...},
  onEnd (...args) {...},
})

// stop listening
bar.opts.onData = null

// Another example when reading a file / blob
// How often do you need to listen to this events more then twice?
var fr = new FileReader()
fr.addEventListener('load', function (evt) {...})
fr.addEventListener('error', function (evt) {...})
fr.readAsArrayBuffer(blob)

// I usually solve this by doing something like:
var arrayBuffer = await new Response(blob).arrayBuffer()
var json = await new Response(blob).json()
var iterable = new Response(blob).body
var text = await new Response(blob).text()

// Worth mentioning that blob now has new methods blob.text(), blob.arrayBuffer() & blob.stream()
// First two returns a promise
```

When a application knows what callback functions you have registered then there is no need for the application to compute and dispatch all events that nobody have subscribed to. The FileReader dispatch a progress and a loadend event that I'm not even subscribed onto.

## Don't create Node or Web readable Stream yourself.

#### Why?

- Node streams are not available in browser and Web streams are not available in Node.
- Browserify node-stream will drag in the Buffer module as well and increase the size even more.
- Using them will create a lot of overhead stuff you don't even need.

#### How then?

Use iterator and/or asyncIterator.<br>
That are the minimum you will need to be able to create a producer and a consumer that can be both readable and writable

```js
// ✗ avoid
const stream = require('readable-stream')
new ReadableStream({...})
new stream.Readable({...})

// ✓ ok (create a async iterator that reads a large file/blob in chunks)
async function* blobToIterator(blob, chunkSize = 8 << 16) { // 0.5 MiB
  const fr = new FileReader()
  let position = 0

  function read(chunk) {
    return new Promise(rs => {
      fr.onload = () => rs(new Uint8Array(fr.result))
      fr.readAsArrayBuffer(chunk)
    })
  }
  
  while (true) {
    const chunk = blob.slice(position, (position = position + chunkSize))
    if (!chunk.size) return
    yield await read(chunk)
  }
}

const iterable = blobToIterator(new Blob(['123']))
```

There is no problem returning or consumeing a stream you get from example `Response.body` or `fs.createReadStream()` you can pass this stream around how much you like. Node streams have a `Symbol.asyncIterator` in the prototype and (web streams will eventually have them as well) you can add the symbol to your class also to make it a bit nicer

So you could just do this hack to transform a blob into a stream:

```js
const iterable = new Response(blob).body
const iterable = blob.stream() // (chrome v76)
// you don't create a stream yourself, you merely just transform a blob into an iterable stream ;)
// a stream that has Symbol.asyncIterator
```

Then you can consume, pause, resume and pipe the iterator & streams (thanks due to Symbol.asyncIterator)
All you need to do now is

```js
async function* transform() {
  for await (chunk of iterable) {
    yield do_transformation(chunk)
  }
}

// piping one iterator to another iterator
iterator = transform(get_iterable())
```

To make web streams iterable, you could use something like this:

```js
if (!ReadableStream.prototype[Symbol.asyncIterator]) {
  ReadableStream.prototype[Symbol.asyncIterator] = async function* () {
    const reader = this.getReader()
    while (1) {
      const chunk = await reader.read()
      if (chunk.done) return chunk.value
      yield chunk.value
    }
  }
}
```

## Don't use any ajax/request library

Use [Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API), [Node-Fetch](https://www.npmjs.com/package/node-fetch), [isomorphic-fetch](https://www.npmjs.com/package/isomorphic-fetch), or [fetch-ponyfill](https://github.com/qubyte/fetch-ponyfill)

#### Why?

The idea here is to keep the bundle size small, and make less JIT compilation. The server only has to download and compile `node-fetch` once

#### How then?

```js
// ✗ avoid
const http = require('http')
const https = require('https')
const request = require('request')
const axios = require('axios')
const superagent = require('superagent')
const ajax = jQuery.ajax

// ✓ ok
const fetch = require('node-fetch') // browser exclude
const { Headers, Response, Request } = fetch // browser exclude
```

## Don't use node's [Url](https://nodejs.org/api/url.html#url_legacy_url_api) or [querystring](https://nodejs.org/api/querystring.html)

#### Why?

- The [WHATWG URL](https://developer.mozilla.org/en-US/docs/Web/API/URL) Standard uses a more selective and fine grained approach to selecting encoded characters than that used by the Legacy API.
- WHATWG URL and URLSearchParams is available in both context
- querystring will mix the value between string and arrays giving you an inconsistent api (see parse example below)

#### How then?

```js
// ✗ avoid
const URL = require('whatwg-url')
const URLSearchParams = require('url-search-params')
const querystring = require('querystring')
const url = require('url').Url
const parsed = url.parse(source)
const obj = querystring.parse('a=a&abc=x&abc=y') // { a: 'a', abc: ['x', 'y'] }

// ✓ ok
const { URL, URLSearchParams } = require('url') // browser exclude

// in Node v10.0.0 this is available on the global scope
const parsed = new URL(source)
const params = new URLSearchParams(source)
```

## Don't use node's [string_decoder](https://nodejs.org/api/string_decoder.html)

#### Why?

[TextDecoder](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder/TextDecoder) & [TextEncoder](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder/TextEncoder) can accomplish the same task but are available in both context

#### How then?

```js
// ✗ avoid
const stringDecoder = require('string_decoder')

// ✓ ok
const { TextDecoder, TextEncoder } = require('util') // browser exclude
```

## Don't use [inherits](https://nodejs.org/docs/latest/api/util.html#util_util_inherits_constructor_superconstructor)

#### Why?

You can accomplish the same thing with class extends

#### How then?

```js
// ✗ avoid
const inherits = require('util').inherits
const inherits = require('inherits')

// ✓ ok
class Foo extends Something {}
```

## Don't use if-else platform specific inside functions

#### Why?

Doing the same check over and over again makes the compiler unable to garbage collect the unnecessary part

#### How then?

```js
// ✗ avoid
Foo.prototype.update = function update (state) {
  if (typeof requestIdleCallback === 'function') {
    requestIdleCallback(() => { this.updateState(state) })
  } else {
    this.updateState(state)
  }
}

// ✓ ok
if (typeof requestIdleCallback === 'function') {
  Foo.prototype.update = function update (state) {
    requestIdleCallback(() => { this.updateState(state) })
  }
} else {
  Foo.prototype.update = Foo.prototype.updateState
}
```

## Don't depend of things that would make your application crash in another context

#### Why?

So it works everywhere.<br>
Others are going to want to use your module but they may also use code testing/coverage inside NodeJS.<br>
Or for some reason import your script to a web worker without using it<br>
Your script doesn't have to be fully functional, you can still write code that depends on the DOM or using the location to redirect to another page or use any node specific code like `process.nextTick`. Even if you write a module that is targeted for only the browser.

#### How then?

If your file can be executed with this three method without crashing then you are golden.

```bash
node utils.js
```

```html
<script src="utils.js"></script>
<script>new Worker('utils.js')</script>
```

```js
// utils.js

// ✗ avoid

// Use the same canvas and context for everything
const canvas = document.createElement('canvas')
const ctx = canvas.getContext('2d')

// ✓ ok
if (typeof document === 'object') {
  var canvas = document.createElement('canvas')
  var ctx = canvas.getContext('2d')
}

/**
 * @param  {HTMLVideoElement} video the video element
 * @param  {Function} cb function to call when done
 */
function captureFrame (video, cb) {...}

/**
 * @param  {Image} the image to rotate
 * @param  {Function} cb function to call when done
 */
function rotateFrame (img, cb) {...}

module.exports = { captureFrame, rotateFrame }
```

### Don't use anything else then javascript

...such as PureScript, TypeScript, LiveScript or CoffeeScript that isn't able to run on any platform without beeing transpiled to javascript first

#### Why?

The point is that:

- It should just work without any transpilation
- Transpilers just adds an additional building step, building time and the output is sometimes much larger
- Your project will be more complex.
- You are always going stay in the shadow of JavaScript and always have to wait for other transpilers to start supporting new syntax before you can use it.
- People should be able to include a part of your module without having to require the entire bundle or having to compile it themselves
- You need to educate developers to properly use other language other then what was built for the platform, while at the same time you need to know a bit of javascript to know what is going on

```js
import x from 'module/foo'
```

It should feel less like a [jungle](https://hackernoon.com/how-it-feels-to-learn-javascript-in-2016-d3a717dd577f)
Its like trying to fit a square into a circle hole, and invading our platform. Have you looked at the output of a compiled file? It's much larger then if you had written it yourself.

[you might not need typescript](https://medium.com/javascript-scene/you-might-not-need-typescript-or-static-types-aa7cb670a77b) - by Eric Elliott

#### How then?

It's still fine to use `d.ts` files to help with IDE.<br>
You can also use jsDoc comment annotation, default params like so:

```js
/**
 * @param  {HTMLVideoElement} video the video element
 * @return {Promise<blob>}
 */
function captureFrame (video) {...}
```

(Look github parse this so well with syntax color ☺️)<br>
using jsDoc you would be able to take full advantage of closure-compiler advanced optimizations also
The doc annotations works well with visual studio code also

Hold your horses and wait until Static Typing becomes a real thing: https://github.com/sirisian/ecmascript-types
It's not fun to transpile your existing ts/flow back to js when it lands. So use jsDoc & `d.ts` for now.

## Don't use cancelable promises

#### Why?

There is a standard available and it's called [AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) it was initially built for aborting a fetch request but they can be used for other things as well. There is polyfill available and you can remove it once it becomes available later on (easier to refactor)

#### How then?

```js
// ✗ avoid
import * from 'p-cancelable'
import * from 'p-timeout'
import * from 'promise-cancelable'

// ✓ ok
const controller = new AbortController();
const signal = controller.signal;
fetch(url, {signal})
// Later
controller.abort();
```

# Is there a readme badge?

Yes! but I have not made one myself, since so many love to use https://shields.io in their readme's you could include this to let people know that your code is using CrossJS style.

```Markdown
[![Cross js compatible](https://img.shields.io/badge/Cross--js-Compatible-brightgreen.svg)](https://github.com/cross-js/cross-js)
```

[![CrossJS compatible](https://img.shields.io/badge/Cross--js-Compatible-brightgreen.svg)](https://github.com/cross-js/cross-js)
