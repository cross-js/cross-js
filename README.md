<div align="center">
  <img src="./cross-js.svg" alt="logo"  width="300">
  
# CrossJs
</div>
# Why should I use CrossJS style guide?

Adopting the CrossJS style means your JavaScript can work in any environment without being dependent on any core browser/node JS API and can work in any context, without unnecessarily increasing the bundle size. This might not make sense for all projects and development cultures. These rules certainly don't apply if you are only targeting one platform that has the API built in, but it makes sense for single modules.

# For References

By only including [readable-stream](https://www.npmjs.com/package/readable-stream) in browser without anything else you have included buffer, events, string_decoder and inherits modules among many more smaller modules and already broke 4 of these rules and increased your bundle to:

|                | gzip    | uncompressed |
| -------------- | ------- | ------------ |
| **unminified** | 43.73KB | 174.77KB     |
| **minified**   | 19.79KB | 67.93KB      |

**Some Read Up:**

- [JavaScript Bootup Time Is Too High](https://developers.google.com/web/tools/lighthouse/audits/bootup)
- [JavaScript Start-up Optimization](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/)

Summary: Javascript is now the highest performance hit on many websites... One large image is able to load faster than JS. I also recommend that you try out [lighthouse](https://chrome.google.com/webstore/detail/lighthouse/blipmdconlkpinefehnmjammfjpmpbjk) and perform a performance test.
If you follow this rule you might be able to write code with just a fraction of what you otherwise would need.

# CrossJS — The Rules

- [Don't use fs](https://github.com/cross-js/cross-js#dont-use-fs)
- [Don't use Buffer](https://github.com/cross-js/cross-js#dont-use-buffer)
- [Don't use EventEmitter or EventTarget](https://github.com/cross-js/cross-js#dont-use-eventemitter-or-eventtarget)
- [Don't create Node or Web readable Stream yourself](https://github.com/cross-js/cross-js#dont-create-node-or-web-readable-stream-yourself)
- [Don't use any ajax/request library](https://github.com/cross-js/cross-js#dont-use-any-ajaxrequest-library)
- [Don't use node's Url or querystring](https://github.com/cross-js/cross-js#dont-use-nodes-url-or-querystring)
- [Don't use node's string_decoder](https://github.com/cross-js/cross-js#dont-use-nodes-string_decoder)
- [Don't use inherits](https://github.com/cross-js/cross-js#dont-use-inherits)
- [Don't use if-else platform specific code inside functions](https://github.com/cross-js/cross-js#dont-use-if-else-platform-specific-inside-functions)
- [Don't depend of things that would make your application crash in another context](https://github.com/cross-js/cross-js#dont-depend-of-things-that-would-make-your-application-crash-in-another-context)
- [Don't use extensionless import](https://github.com/cross-js/cross-js#dont-use-extensionless-import)
- [Don't use anything else then javascript](https://github.com/cross-js/cross-js#dont-use-anything-else-then-javascript)
- [Don't use cancelable promises](https://github.com/cross-js/cross-js#dont-use-cancelable-promises)

## Don't use fs

... or the [FileReader](https://developer.mozilla.org/en-US/docs/Web/API/FileReader)

#### Why?
To understand the concept of writing cross platform application then you must understanding the [onion architecture](https://codeguru.com/csharp/csharp/cs_misc/designtechniques/understanding-onion-architecture.html).
Try to develop your package with as little dependencies or knowledge of the platform you are running your code on, try to think as if your module was running on a sandboxed enviorment (or web worker) with no filesystem or network access, or no access to node or browser API's.
The core layer of your application should be like a stdin and stdout. How the consumer reads and saves data should be entirely up to the the developers using your package.
[Deno](https://deno.land/) requires modules to ask for permission to use fs/net. It feels safer to provide data to a third party package that does the transformation for you and gives you data back, rather than giving it read/write permissions to an entire folder.

Here is a senario: Say you have developed a tool that can encode/decode csv data to/from json. For it to work in node, deno and browser, it shouldn't be responsible for reading and saving files. The input data can come from many sources such as network, blob, fs, web socket and the output can have many destinations as well:

```js
// A node developer would use it like this
asyncReadIterator = fs.createReadStream(file)
asyncJsonIterator = csv_to_json(asyncReadIterator)

for await (let row of asyncJsonIterator) http.request(row.url)

// A browser developer would use it like this
asyncReadIterator = blob.stream()
asyncJsonIterator = csv_to_json(asyncReadIterator)

for await (let row of asyncJsonIterator) await fetch(row.url)
```

Only the end developer that is on the top of the onion structure and sitting in just one enviroment is allowed to use fs, but as soon as you allow someone else to use your package or it starts to be cross platform compatible, then it should be forbidden.

## Don't use Buffer

#### Why?

`Uint8Array` and `Buffer` have a lot in common and are very similar to each other.
Adding `buffer` will increase your bundle size a lot. And the fact that buffer inherits from
`Uint8Array` have made recent Node.js core API's accept typed arrays. For example: `fs.writeFile()` used to only accept a Buffer but now works with both typed arrays and buffers.

Following this rule doesn't mean you have to convert all buffers you receive from node's core API and other modules from buffer to UInt8Array, just treat the buffer as a Uint8Array instead since buffer inherits from it.

#### How then?

Use [Uint8Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array) and [DataView](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView)

```js
// ✗ avoid
const chunk = Buffer.from(source)
const chunk = Buffer.alloc(n)
const chunk = Buffer.allocUnsafe(n)
const chunk = new Buffer(source)

// ✓ ok
// Buffer allocation
const chunk = new Uint8Array(n)

// Buffer from An array-like or iterable object to convert to a typed array.
const chunk = Uint8Array.from(source[, mapFn[, thisArg]])

// Buffer from string
const chunk = new TextEncoder().encode('abc')

// Buffer from base64 (minimalist, there are synchronous way to, try avoiding base64 in the first place)
const arrayBuffer = await fetch(`data:;base64,${string}`).then(r => r.arrayBuffer())
const chunk = new Uint8Array(arrayBuffer)
```

## Don't use [EventEmitter](https://nodejs.org/api/events.html)

#### Why?

**UPDATE:** EventTarget have been added to NodeJS and that is now considered best, it has `once`, and `signal` support also and works in deno, node and browsers, `addEventListener(name, fn, {signal, once: true})`

Don't take this seriously, sometimes it can be good to have more then one listener of one type registered. Also if you need something that can bubble up & down. IMHO I think that using events can increase the complexity of some application. It could certainly be avoided by other means without depending on other modules. In the end it will just increase the bundle size. Use them if it makes sense.

All I'm saying is:<br>
Think twice before you decide to use them and if you really need it.<br>
There is more than one way to skin a cat<br>

- Including [EventEmitter](https://nodejs.org/api/events.html) comes with a bundle cost.
- Also, how often do you need to subscribe to some event more than twice?

Often you know all the event you want to subscribe to beforehand, so why not just pass those down in the constructor or the function instead

**Update** NodeJS have introduced EventTarget and if you choose to use some kind of event handeling then I would suggest that you use EventTarget instead of EventEmitter, one new cool thing about EventTarget is that you can use AbortSignal (now also introduced to NodeJS core) as a way to also stop listening to multiple events as you call `abortController.abort()`

#### How then?

```js
// ✗ avoid
import EventEmitter from 'node:event'
class Foo extends EventEmitter {}

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

// ✓ ok (extending EventTarget)
class Bar extends EventTarget {}

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
const fr = new FileReader()
fr.addEventListener('load', function (evt) {...})
fr.addEventListener('error', function (evt) {...})
fr.readAsArrayBuffer(blob)

// I usually solve this by doing something like:
const arrayBuffer = await new Response(blob).arrayBuffer()
const json = await new Response(blob).json()
const iterable = new Response(blob).body
const text = await new Response(blob).text()

// Worth mentioning that blob now has new methods blob.text(), blob.arrayBuffer() & blob.stream()
// First two returns a promise
```

When a application knows what callback functions you have registered then there is no need for the application to compute and dispatch all events that nobody has subscribed to. The FileReader dispatches a progress and a loadend event that I'm not even subscribed to.

## Don't create Node or Web readable Stream yourself.

#### Why?

- Node streams are not available in browser and Web streams are not available in Node.
- Browserify node-stream will drag in the Buffer module as well and increase the size even more.
- Using them will create a lot of overhead stuff you don't even need.

#### How then?

**Update:** node v18 now have whatwg streams built in, but they are relatively slow,  browser still lacks some functionallity.

Use iterator and/or asyncIterator.<br>
Those are the minimum you will need to be able to create a producer and a consumer that can be both read and write.

```js
// ✗ avoid
import stream from 'readable-stream'
new ReadableStream({...})
new stream.Readable({...})

// ✓ ok (create a async iterator that reads a large file/blob in chunks)
async function* blobToIterator(blob, chunkSize = 8 << 16) { // 0.5 MiB
  let position = 0
  
  while (true) {
    const chunk = blob.slice(position, (position = position + chunkSize))
    if (!chunk.size) return
    yield new Uint8Array(await chunk.arrayBuffer())
  }
}

const iterable = blobToIterator(new Blob(['123']))

// ✓ ok (convert a blob to a stream and read its iterator)
const stream = blob.stream()
const iterable = stream[Symbol.asyncIterator]()
```

There is no problem returning or consuming a stream you get from example `Response.body` or `fs.createReadStream()` you can pass this stream around how much you like. Node streams have a `Symbol.asyncIterator` in the prototype and (web streams will eventually have them as well) you can add the symbol to your class also to make it a bit nicer

So you could just do this hack to transform a blob into a stream:

```js
// ✓ ok
const iterable = new Response(blob).body
// ✓ ok
const iterable = blob.stream() // (chrome v76)
// you don't create a stream yourself, you merely just transform a blob into an iterable stream ;)
// a stream that has Symbol.asyncIterator
```

Then you can consume, pause, resume and pipe the iterator & streams (thanks due to Symbol.asyncIterator)
All you need to do now is:

```js
async function* transform(iterable) {
  for await (chunk of iterable) {
    yield do_transformation(chunk)
  }
}

// piping one iterator to another iterator
iterator = transform(get_iterable())
```

To make web streams iterable, you could use something like this:

```js
import 'fast-readable-async-iterator'

// or

if (typeof ReadableStream !== 'undefined' && !ReadableStream.prototype[Symbol.asyncIterator]) {
  ReadableStream.prototype[Symbol.asyncIterator] = function () {
    const reader = this.getReader()
    let last = reader.read()
    return {
      next () {
        const temp = last
        last = reader.read()
        return temp
      },
      return () {
        return reader.releaseLock()
      },
      throw (err) {
        this.return()
        throw err
      },
      [Symbol.asyncIterator] () {
        return this
      }
    }
  }
}
```

I think that your lower level api should be an (async)Iterator and provided to the user as is. If the user then wants to pipe it and do stuff with it then they could use either [nodes](https://nodejs.org/api/stream.html#stream_stream_readable_from_iterable_options) or [WHATWG upcomming](https://github.com/whatwg/streams/issues/1018) stream.from(iterable). It's also a good way to convert whatwg & node streams to one or the other (since both are @@asyncIterable) but noed stream should be avoided in a browser context and vise versa in the first place. 
```js
// should be left out of a lib and be used by the developer themself.
ReadableStream.from(iterable || node_stream || whatwg_stream)
  .pipeThrough(new TextEncoderStream()) // convert text to uint8arrays
  .pipeThrough(new CompressionStream('gzip')) // compress bytes to gzip
  .pipeTo(destination)
  .then(done, fail)

// or in node
import { Readable } from 'streamx'

const stream = Readable.from(iterable || node_stream || whatwg_stream)
```

## Don't use any ajax/request library

**Update** NodeJS v18 has fetch built in, use it instead.

Use [Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API), [Node-Fetch](https://www.npmjs.com/package/node-fetch), [isomorphic-fetch](https://www.npmjs.com/package/isomorphic-fetch), or [fetch-ponyfill](https://github.com/qubyte/fetch-ponyfill)

#### Why?

The idea here is to keep the bundle size small, and use less JIT compilation. Node servers only have to download and compile `node-fetch` once. [Deno](https://deno.land) also has fetch built right in, so no extra dependency is needed there, web workers don't have XMLHttpRequest, only fetch is supported so axios won't even work in web workers

#### How then?

```js
// ✗ avoid
import http from 'node:http'
import https from 'node:https'
import request from 'request'
import axios from 'axios'
import superagent from 'superagent'
const ajax = jQuery.ajax

// ✓ ok
import fetch, { Headers, Response, Request } from 'node-fetch' // browser exclude
globalThis.fetch // requires node v18
```

Something even better if you apply the _onion architecture_ from the "Don't use the fs" rule section. Don't make the actual request. Instead construct a Request like object and pass it back to the developer so he/she can modify/make the request itself so i can use whatever http library they want. Think of it as not having any access to network request.

## Don't use node's [Url](https://nodejs.org/api/url.html#url_legacy_url_api) or [querystring](https://nodejs.org/api/querystring.html)

#### Why?

- The [WHATWG URL](https://developer.mozilla.org/en-US/docs/Web/API/URL) Standard uses a more selective and fine grained approach to selecting encoded characters than that used by the Legacy API.
- WHATWG URL and URLSearchParams is available in all contexts
- querystring will mix the value between string and arrays giving you an inconsistent api (see parse example below)

#### How then?

```js
// ✗ avoid
import URL from 'whatwg-url'
import URLSearchParams from 'url-search-params'
import querystring from 'node:querystring'
import Url from 'node:url'
const parsed = Url.parse(source)
const obj = querystring.parse('a=a&abc=x&abc=y') // { a: 'a', abc: ['x', 'y'] }

// ✓ ok
const parsed = new URL(source)
const params = new URLSearchParams(source)
```

## Don't use node's [string_decoder](https://nodejs.org/api/string_decoder.html)

#### Why?

[TextDecoder](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder/TextDecoder) & [TextEncoder](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder/TextEncoder) can accomplish the same task but are available in both context

#### How then?

```js
// ✗ avoid
import stringDecoder from 'string_decoder'

// ✓ ok
const { TextDecoder, TextEncoder } = globalThis
```

## Don't use [inherits](https://nodejs.org/docs/latest/api/util.html#util_util_inherits_constructor_superconstructor)

#### Why?

You can accomplish the same thing with class extends

#### How then?

```js
// ✗ avoid
import { inherits } from 'node:util'
import inherits from 'inherits'

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

So it works everywhere.

Others are going to want to use your module but they may also use code testing/coverage inside NodeJS.

A good example of this is the ReadableStream asyncIterator polyfill shown above.

Or for some reason import your script to a web worker without using it.

Your script doesn't have to be fully functional, you can still write code that depends on the DOM or using the location to redirect to another page or use any node specific code like `process.nextTick`. Even if you write a module that is targeted for only the browser.

#### How then?

If your file can be executed with this three method without crashing then you are golden.

```bash
node utils.js
```

```html
<script type="module" src="utils.js"></script>
<script>new Worker('utils.js', { type: 'module' })</script>
```

```js
// utils.js

// ✗ avoid

// Use the same canvas and context for everything
const canvas = document.createElement('canvas')
const ctx = canvas.getContext('2d')

// ✓ ok
let canvas
let ctx
if (typeof document === 'object') {
  canvas = document.createElement('canvas')
  ctx = canvas.getContext('2d')
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

### Don't use extensionless import

...or importing a index file with `./`

#### Why?

Browser can't just guess that it should fetch a resource that ends with `.js` or `index.js`
it creates more burden on the compiler and it don't work with ESM

```js
// ✗ avoid
import foo from './foo'
import index from './'

// ✓ ok
import foo from './foo.js'
import index from './index.js'
```

### Don't use anything else then javascript

...such as PureScript, TypeScript, LiveScript or CoffeeScript that isn't able to run on any platform without beeing transpiled to javascript first

#### Why?

Read my other article on why [You might not need TypeScript](https://jimmywarting.github.io/you-might-not-need-typescript/)

The point is that:

- It should just work without any transpilation
- Transpilers just adds an additional building step, building time and the output is sometimes much larger
- Your project will be more complex.
- You are always going stay in the shadow of JavaScript and always have to wait for other transpilers to start supporting new syntax before you can use it.
- People should be able to include a part of your module without having to require the entire bundle or having to compile it themselves
- You need to educate developers to properly use other language other then what was built for the platform, while at the same time you need to know a bit of javascript to know what is going on

```js
import x from 'module/foo.js'
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

There is a standard available and it's called [AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) it was initially built for aborting a fetch request but they can be used for other things as well. There is a polyfill available and you can remove it once it becomes available later on (easier to refactor)

#### How then?

```js
// ✗ avoid
import * from 'p-cancelable'
import * from 'p-timeout'
import * from 'promise-cancelable'

// ✓ ok
const controller = new AbortController();
const signal = controller.signal;
fetch(url, { signal })
// Later
controller.abort();
```

# Is there a readme badge?

Yes! but I have not made one myself, since so many love to use https://shields.io in their readme's you could include this to let people know that your code is using CrossJS style.

```Markdown
[![Cross js compatible](https://img.shields.io/badge/Cross--js-Compatible-brightgreen.svg)](https://github.com/cross-js/cross-js)
```

[![CrossJS compatible](https://img.shields.io/badge/Cross--js-Compatible-brightgreen.svg)](https://github.com/cross-js/cross-js)
