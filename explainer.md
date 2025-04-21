# ShadowRealm Explainer

## What is this?

ShadowRealm is a mechanism that lets you **execute JavaScript code synchronously, in a fresh, isolated environment**.

"Fresh" means an execution environment with a pristine global object, without any globally visible modifications that might have been made — it's as if you were executing code in a completely new browser tab or interpreter process ("realm"). "Isolated" means that any modifications you make inside the ShadowRealm environment aren't visible to code outside of it. "Synchronous" means you don't have to `await` the result.

```javascript
globalThis.someValue = 2;

const realm = new ShadowRealm();

console.log(globalThis.someValue);
  // logs "2", as you might expect
console.log(realm.evaluate(`globalThis.someValue`));
  // logs "undefined": the global modification is not
  // visible inside the fresh environment

realm.evaluate(`globalThis.someValue = 3;`);

console.log(globalThis.someValue);
  // logs "2": the ShadowRealm's environment is isolated,
  // so the global modification does not affect the
  // environment outside of it
console.log(realm.evaluate(`globalThis.someValue`));
  // logs "3", as you might expect
```

To prevent information leakage, only **primitive values** and **callable objects** can pass in and out of a ShadowRealm. Non-callable objects are forbidden. Although callables are also objects, when they pass through the ShadowRealm boundary they are wrapped in another object that allows calling them but hides their properties.

```javascript
const realm = new ShadowRealm();

console.log(realm.evaluate(`42`));
  // logs "42", because 42 is a primitive
realm.evaluate(`{}`);
  // throws an exception; you can't access the ShadowRealm's
  // object from the original realm, even an empty object

const logValueFromOuterRealm = realm.evaluate(`
function logValueFromOuterRealm(value) {
  console.log(value);
}`);
// logValueFromOuterRealm is callable, so returning it as
// the result of `evaluate` to the outer realm gives us a
// callable object

logValueFromOuterRealm('I like cats');
  // logs "I like cats", because strings are primitives
logValueFromOuterRealm({});
  // throws an exception; you can't access the outer realm's
  // object from inside the ShadowRealm, even an empty object,
  // even as a function argument

realm.evaluate(`logValueFromOuterRealm.sideChannel = true;`);

realm.evaluate(`console.log(logValueFromOuterRealm.sideChannel);`);
  // logs "true", because logValueFromOuterRealm now has an
  // extra property.
console.log(logValueFromOuterRealm.sideChannel);
  // logs "undefined": the extra property can't be observed
  // or accessed in the outer realm, even though it's present
  // in the ShadowRealm
```

And that's the basic idea of ShadowRealm!

## Why would I want this?

Web applications grow bigger and more customizable all the time.

Applications that allow themselves to be customized via plugins always have to deal with the problem of badly-behaved plugins that reach into places where they're not supposed to and break other code. In JavaScript most built-in stuff is overwritable, so it's a common problem. When application writers have a way of segmenting off and isolating code they don't control, they can deliver a more stable experience to users.

Applications such as JavaScript programming and teaching environments have this same need, but more directly. Users' code should not be able to bring down the application, but also things defined in the application code should not be accessible from users' code.

Finally, some applications have use for a simulated environment: for example, the [JSDOM](https://github.com/jsdom/jsdom) library builds an environment with a fake DOM, for doing HTML document manipulation on the server. JSDOM is specific to Node.js. With ShadowRealm you can build the same thing in a cross-platform way. This allows more secure scraping or processing such as that done in [Firefox Reader View](https://github.com/mozilla/readability).

There are currently other ways to accomplish approximately the same thing as ShadowRealm, but they are either not synchronous, or not isolated, or not cross-platform. ShadowRealm allows developers to implement this kind of thing with just less hassle and potential for mistakes. That's good for users, who then face fewer security bugs.

See the example "Isolating dependencies from one another" in the use cases section, which compares ShadowRealm with other means of accomplishing the same thing.

## What _isn't_ ShadowRealm for?

ShadowRealm is sometimes called a "sandbox", but people might have conflicting expectations of the term "sandbox". ShadowRealm gives you **integrity protection**, in that you have complete control over what objects exist in the ShadowRealm, and you have complete control over how code in the ShadowRealm is allowed to affect the outer realm.

However, it does not give you **availability protection**. ShadowRealms share a process and a thread with the outer realm. That's why synchronous communication is possible. But that also means it's possible to freeze up the outer realm or use all of its memory. It's as simple as this:
```javascript
realm.evaluate(`while (true) {}`);
```

ShadowRealm also does not give you **confidentiality protection**. In other words, it won't protect you from timing attacks such as Spectre, and it's possible to deduce fingerprinting information from inside a ShadowRealm. For example, `Intl.supportedValuesOf('timeZone')` gives you fingerprinting information about what version of the time zone database is used by the JavaScript engine.

You _can_ use ShadowRealm as a building block for an environment that _does_ protect confidentiality, by deleting APIs such as `Intl.supportedValuesOf` immediately after creating the ShadowRealm, but that's at your own risk.

Here's a table showing which tool offers which security protections. "🌓" means it can be done, but is not by default.

| **Tool** 👇 Protects 👉 | **Integrity** | **Availability** | **Confidentiality** |
| --- | --- | --- | --- |
| `node:vm` module | ✅ | ❌ | 🌓 |
| iframe | ❌ | ❌ | ❌ |
| cross-origin iframe | ✅ | 🌓 | 🌓 |
| Worker | ✅ | 🌓 | 🌓 |
| **ShadowRealm** | ✅ | ❌ | 🌓 |

## Proposed API

The API surface is tiny. Just three functions! The API is a low-level one, intended to be used as a building block for more complex functionality.

### Constructor
```javascript
realm = new ShadowRealm();
```
Creates a fresh ShadowRealm object. It's always separate from any other realms, even other ShadowRealms that have already been created.

### evaluate
```javascript
result = realm.evaluate(code);
```
Executes `code`, a string containing a JavaScript expression, inside the ShadowRealm, and returns the result if it is a primitive. If the result is a callable object, returns a wrapper object that allows calling, but hides any properties.

If the result is a non-callable object, throws a TypeError. If executing `code` causes an exception, it throws a fresh TypeError so as not to expose the ShadowRealm's exception object. See the [errors explainer](./errors.md) for more information.

### importValue
```javascript
exportValue = await realm.importValue(moduleSpecifier, exportName);
```

Imports `moduleSpecifier` into the ShadowRealm environment (as if `await import(moduleSpecifier)` were executed) and gets an export named `exportName` from that module. Just as in `evaluate()`, if the export is a primitive or callable value, it is returned or wrapped, respectively, and otherwise a TypeError is thrown.

Each ShadowRealm has its own module graph, meaning that imports are not shared between realms.

## Use cases

### Example: Isolating dependencies from one another

Say you have a large codebase with dependencies that conflict with each other. Maybe they require incompatible versions of a common dependency, or maybe one dependency [modifies a built-in prototype in a way that breaks another dependency](https://developer.chrome.com/blog/smooshgate). Maybe your product is an app that may be customized by each customer with plugins that they write; of course you can't guarantee the code quality of these plugins, and the main codebase needs to be robust against that. Or maybe you're just part of a large organization and you want to limit the damage that miscommunications between departments can wreak in production.

As a developer of such a codebase, you can use ShadowRealm to segment off the potentially badly-behaved dependencies. The end user of this codebase benefits by having a more stable experience while still being able to load whatever weird plugins they want.

As a contrived example, let's consider a fictitious dependency, `sketchy-product.js`, that calculates the result of multiplying numbers together:

```javascript
// My version is leeter than built-in reduce???
Array.prototype.reduce = function (callback, accumulator = 1) {
  for (let index = 0; index < this.length; index++) {
    accumulator = callback(accumulator, this[index]);
  }
  return accumulator;
}

// Calculate the product of the numbers passed as arguments
function product(...multiplicands) {
  return multiplicands.reduce((a, b) => a * b);
}
export default product;
```

(What's sketchy about it? `product()` gives perfectly fine results, but it overwrites the global `Array.prototype.reduce` with a broken version! This will break most other code that calls `reduce` without the second argument, including the common idiom `reduce((a, b) => a + b)` to calculate a sum. Bonus if you spotted that it also doesn't call the callback with the index and array as 3rd and 4th argument.)

You need to use this dependency in your codebase, but it breaks large portions of the rest of your code. Here's how to isolate it with ShadowRealm:

```javascript
const realm = new ShadowRealm();
const safeProduct = await realm.importValue('./sketchy-product.js', 'default');

// Does it still work?
console.assert(safeProduct(1, 2, 3, 4) === 24);

// Is it really safe?
console.assert([1, 2, 3, 4].reduce((a, b) => a + b) === 10);
// phew!
```

Let's compare some of the alternatives that exist today, without ShadowRealm. First off, in Node.js you could use the VM module.

```javascript
// Run with --experimental-vm-modules
import assert from 'node:assert';
import fs from 'node:fs/promises';
import vm from 'node:vm';

const sourceText = await fs.readFile('sketchy-product.js', { encoding: 'utf-8' });

const context = vm.createContext({});
const isolatedModule = new vm.SourceTextModule(sourceText, { context });
await isolatedModule.link(() => {
  // This callback can be empty because sketchy-product doesn't
  // have any other dependencies that need to be linked
});
await isolatedModule.evaluate();

const safeProduct = isolatedModule.namespace.default;

// Does it still work?
assert.equal(safeProduct(1, 2, 3, 4), 24);

// Is it really safe?
assert.equal([1, 2, 3, 4].reduce((a, b) => a + b), 10);
// phew!
```

Node.js's VM module is just as good as ShadowRealm for this use case, and in some ways is a more powerful API that allows influencing more parts of the module import process. In other ways, it is less safe because you have to avoid passing objects into the realm that might leak references to the main realm's global object.

Unfortunately, `node:vm` is specific to Node.js (although Deno provides it as well), and still experimental. Blink provides an Isolate API which is similar but runs in another thread, but it is not available in JS userspace.

In a browser, you could use an iframe. Here's one way to do that:

```javascript
const iframe = document.createElement('iframe');

// We must attach the iframe to the active browsing context,
// or import() will be blocked
const body = document.getElementsByTagName('body')[0];
body.append(iframe);

const realm = iframe.contentWindow;
const imported = await realm.eval(`import('./sketchy-product.js')`);
const safeProduct = imported.default;
iframe.remove();
// It's OK to detach the iframe now that we have imported
// the function, because there are no further imports.

// Does it still work?
console.assert(safeProduct(1, 2, 3, 4) === 24);

// Is it really safe?
console.assert([1, 2, 3, 4].reduce((a, b) => a + b) === 10);
// phew!
```

This example imports the module into a separate realm just like ShadowRealm. But that realm isn't isolated like a ShadowRealm is. Unlike ShadowRealm where nothing is accessible between realms by default, with iframes _everything_ is accessible by default. In the above example we can still just reach into the iframe realm and manipulate its objects: in fact that's what we're doing with `realm.eval`, we're literally grabbing the iframe realm's global `eval` function and executing it.

Suppose `sketchy-product.js` shipped a new version that popped up a message:
```javascript
alert('If you liked this multiplication, please follow my SoundCloud');
```
This would just pop up an intrusive message in the application! We'd have to guard against this by clearing all of the potentially dangerous stuff out of the iframe realm, _before_ loading the module, with things like `realm.alert = () => {}`. But some dangerous properties like [`window.top`](https://developer.mozilla.org/en-US/docs/Web/API/Window/top) can't be overwritten or deleted.

To solve this problem, you can lock down an iframe using the `sandbox` attribute. This allows you to remove certain permissions from the iframe realm. You can treat it as cross-origin (i.e., originating from a different website and therefore severely limited in permissions.)

Here's an example using a cross-origin iframe. It's built on the previous example, but is much more complicated.

```javascript
const iframe = document.createElement('iframe');
iframe.setAttribute('sandbox', 'allow-scripts');
iframe.setAttribute('style', 'visibility: hidden;');
const permissions = [
  'camera', 'display-capture', 'fullscreen', 'gamepad', 'geolocation',
  // ...etc. (All permissions, even those not yet supported, should be
  // listed here, as the default is to allow all.)
];
iframe.setAttribute('allow',
  permissions.map((feature) => `${feature} 'none'`).join('; '));

const iframeHandshake = new Promise((resolve, reject) => {
  window.addEventListener('message', ({ origin, source, data }) => {
    try {
      if (origin !== 'null') throw new Error(`unexpected origin ${origin}`);
      if (source !== iframe.contentWindow) throw new Error('wrong source');
      if (data !== 'handshake') throw new Error('unexpected handshake message');
      resolve();
    } catch (error) {
      reject(error);
    }
  }, { once: true });
});

// We must attach the iframe to the active browsing context, or it
// won't load
const body = document.getElementsByTagName('body')[0];
body.append(iframe);

// Load the isolated realm in the iframe and import the module we
// want to isolate
iframe.srcdoc = `
  <!doctype html>
  <html>
  <head>
    <script type="module">
      import product from './sketchy-product.js';

      const targetOrigin = 'http://localhost:8000';
      const mainRealm = window.parent;

      window.addEventListener('message', ({ origin, source, data }) => {
        try {
          if (origin !== targetOrigin)
            throw new Error(\`unexpected origin ${origin}\`);
          if (source !== mainRealm) throw new Error('wrong source');
          const { operation, operands } = data;
          console.assert(operation === 'product');
          const result = product(...operands);
          mainRealm.postMessage({ operation, result }, { targetOrigin });
        } catch (error) {
          mainRealm.postMessage({
            operation: 'product',
            error,
          }, { targetOrigin });
        }
      });

      window.addEventListener('load', () => {
        mainRealm.postMessage('handshake', { targetOrigin });
      });
    </script>
  </head>
  <body></body>
  </html>
`;
await iframeHandshake;
const realm = iframe.contentWindow;

function safeProduct(...multiplicands) {
  return new Promise((resolve, reject) => {
    globalThis.addEventListener('message', ({ origin, source, data }) => {
      try {
        if (origin !== 'null') throw new Error(`unexpected origin ${origin}`);
        if (source !== realm) throw new Error('wrong source');
        const { operation, result, error } = data;
        console.assert(operation === 'product');
        resolve(result);
      } catch (error) {
        reject(error);
      }
    }, { once: true });

    realm.postMessage({
      operation: 'product',
      operands: multiplicands,
    }, { targetOrigin: '*' });
  });
}

// Does it still work?
console.assert(await safeProduct(1, 2, 3, 4) === 24);

// Is it really safe?
console.assert([1, 2, 3, 4].reduce((a, b) => a + b) === 10);
// phew!
```

Synchronously executing the iframe realm's `eval` function, as we did in the original iframe example, is no longer allowed. A cross-origin iframe realm can only communicate asynchronously, via `postMessage`. So now a simple synchronous operation has become asynchronous, which is a strong drawback! You have to use `await` to call the function, and it means that everything using it has to become asynchronous as well.

It is also, as you can see, much more involved to set up asynchronous communication between the main realm and an iframe realm. The above example can't be run from a local file, it requires an HTTP server that serves `sketchy-product.js` with the `Access-Control-Allow-Origin: null` header. Additionally, it's easy to make a mistake that results in the setup being insecure, such as forgetting to check the message source.

None of the above solutions are available both in server runtimes and in the browser. ShadowRealm makes it possible to do this in a portable way. However, for an almost-cross-platform solution without ShadowRealm, you could use a Worker. Here's an example.

**main.js:**
```javascript
const worker = new Worker('adaptor.js', {
  type: 'module',
  name: 'sketchy dependency adaptor',
});

function safeProduct(...multiplicands) {
  return new Promise((resolve, reject) => {
    worker.addEventListener('message', (event) => {
      const { operation, result, error } = event.data;
      console.assert(operation === 'product');
      error ? reject(error) : resolve(result);
    }, { once: true });

    worker.postMessage({
      operation: 'product',
      operands: multiplicands,
    });
  });
}

// Does it still work?
console.assert(await safeProduct(1, 2, 3, 4) === 24);

// Is it really safe?
console.assert([1, 2, 3, 4].reduce((a, b) => a + b) === 10);
// phew!

worker.terminate();
```

**adaptor.js:**
```javascript
import product from './sketchy-product.js';

addEventListener('message', (event) => {
  try {
    const { operation, operands } = event.data;
    console.assert(operation === 'product');
    const result = product(...operands);
    postMessage({ operation, result });
  } catch (error) {
    postMessage({ operation: 'product', error });
  }
});
```

Some server runtimes, such as Deno and Bun, also support this. Node.js [does not](https://github.com/nodejs/node/issues/43583). This simple example could be changed to use Node.js's `worker_threads` module with minimal adaptation (replacing `addEventListener` with `on`/`once`, etc.) but that can [quickly get more complicated](https://github.com/developit/web-worker).

However, Workers still have an obvious drawback: like cross-origin iframes, they only allow asynchronous communication. It's not possible to communicate synchronously with a Worker, because the Worker is running in another thread. Also like cross-origin iframes, Workers bring a lot more overhead with them.

Here's a table summarizing the advantages and disadvantages of each tool for isolating your dependencies:

| **Tool** 👇 | **Cross-platform** | **Synchronous** | **Isolated** | **Convenient** |
| --- | --- | --- | --- | --- |
| `node:vm` module | ❌ | ✅ | ✅ | ✅ |
| iframe | ❌ | ✅ | ❌ | ✅ |
| cross-origin iframe | ❌ | ❌ | ✅ | ❌ |
| Worker | 🌓 almost | ❌ | ✅ | ✅ |
| **ShadowRealm** | ✅ | ✅ | ✅ | ✅ |

In a real-world example, things would be more complicated. Probably the badly-behaved dependency would need to deal with objects and not just numbers, so you'd need to design an interface between it and the rest of your code. You'd pass the interface's capabilities in to the ShadowRealm as callable functions.

If the dependency needed to pass objects back and forth, you'd need to use a [membrane](https://github.com/ajvincent/es-membrane) [library](https://github.com/salesforce/near-membrane) to set up communication proxy objects on either side of the boundary. When we talk about using ShadowRealm as a building block for higher-level functionality, a membrane library is an example of higher-level functionality that can be built with ShadowRealm.

### Example: Online code editor

Here's another example: an online code editor, built using React, that is powered by ShadowRealm.

In this code editor, you can't do `alert()` or access `window.top` or overwrite builtins or anything else nefarious, the way you could if using `eval()` instead of ShadowRealm, and that's by default!

```javascript
import React, { useState } from "react";

export function App() {
  const [code, setCode] = useState("");
  const [result, setResult] = useState("");
  const [logs, setLogs] = useState([]);

  const runCode = () => {
    // Create a new realm each time. Otherwise, the results
    // from previous runs could influence this run. (That
    // may be what you want, in which case you could reuse
    // the same realm.)
    const realm = new ShadowRealm();

    // Setup a fake console.log() in the ShadowRealm that
    // sends its results back to the main realm. Objects
    // can't pass the boundary, so we convert the result to
    // a string. This produces results like [object Object],
    // so in a more sophisticated code editor you'd probably
    // want to send some kind of serialized JSON object that
    // the console could use to do rich formatting on the
    // value.
    const pendingLogs = [];
    realm.evaluate(`
      (appendLog) => {
        globalThis.console = {
          log(...args) {
            args.forEach((arg) => appendLog(String(arg)));
          },
        };
      }
    `)((message) => pendingLogs.push(message));

    // Execute the user's code inside the ShadowRealm. Same
    // as the console window, we convert the result into a
    // string.
    try {
      const val = realm.evaluate(`
        (code) => {
          try {
            return String(eval(code));
          } catch (error) {
            return '⚠️ ' + String(error);
          }
        }
      `)(code);
      setResult(val);
    } catch (error) {
      // Errors from the user's code are handled above. If
      // we catch an error here, that's probably something
      // wrong with our realm setup, so show it differently.
      setResult('🔴 ' + error.toString());
    }
    setLogs(logs.concat(pendingLogs));
  };

  return (
    <div className="container">
      <textarea
        placeholder="Enter JavaScript code"
        value={code}
        onChange={(e) => setCode(e.target.value)}
      />
      <button onClick={runCode}>Run Code</button>
      <div className="box result">
        <h3>Result</h3>
        <pre>{result}</pre>
      </div>
      <div className="box console">
        <h3>Console</h3>
        <ul>
          {logs.map((message, ix) => (
              <li key={ix}>{message}</li>
          ))}
        </ul>
      </div>
    </div>
  );
}
```

### Example: Testing environment with controlled clock

Here's an example of ShadowRealm being used to create a controlled environment within which to run code. It maintains a [mock](https://en.wikipedia.org/wiki/Mock_object) system clock outside of the ShadowRealm, and ensures that APIs that access the system clock from inside the ShadowRealm (like `Date.now()`) use the mock clock instead.

(In reality, this example doesn't fix _everything_ to use the mock clock; you'd have to replace the `Date` constructor as well, `setInterval`/`clearInterval`, `AbortSignal.timeout()`, and `performance`, and you'd have to handle the same edge cases as the real APIs. This is all omitted for brevity.)

ShadowRealm is ideal for testing. Besides allowing you to set up the environment to work exactly as you want, it's also helpful that each ShadowRealm instance has its own module graph. That allows you to make sure that stateful dependencies are reset in between tests, for example.

```javascript
class ClockControlledRealm extends ShadowRealm {
  // Temporal.Instant representing the system time under our control
  #systemTime;
  // Monotonic clock consisting of elapsed Temporal.Duration
  #elapsedMonotonic = new Temporal.Duration();
  #pendingTimeouts = [];
  #nextID = 0;

  constructor(initialSystemTime = new Temporal.Instant(0n)) {
    super();
    this.#systemTime = Temporal.Instant.from(initialSystemTime);

    // Set up the means of querying the system time. (Note, there are others
    // as well not covered here, such as the Date constructor. This is just
    // to illustrate how it would work.)
    this.evaluate(`
      (systemEpochNs) => {
        Temporal.Now.instant = () => new Temporal.Instant(systemEpochNs());
        Date.now = () => Number(systemEpochNs() / 1_000_000n);
      }
    `)(() => this.#systemTime.epochNanoseconds);

    // Set up the timers. (Likewise, this is an illustration of how it would
    // work and doesn't cover setInterval, AbortSignal.timeout, etc.)
    this.evaluate(`
      (set, clear) => {
        globalThis.setTimeout = (f, delay = 0, ...args) => {
          const callable = f.bind(globalThis, ...args);
          return set(delay, callable);
        };
        globalThis.clearTimeout = clear;
      }
    `)((delayMs, callable) => {
      // enqueue a timeout
      const triggerTime = this.#elapsedMonotonic.add({ milliseconds: delayMs });
      const id = this.#nextID++;
      const index = this.#pendingTimeouts.findLastIndex((entry) =>
        Temporal.Duration.compare(entry.triggerTime, triggerTime) <= 0);
      this.#pendingTimeouts.splice(index + 1, 0, { id, triggerTime, callable });
      return id;
    }, (id) => {
      // clear timeout with given ID
      const index = this.#pendingTimeouts.findIndex((entry) => entry.id === id);
      this.#pendingTimeouts.splice(index, 1);
    });
  }

  setClock(newSystemTime) {
    // Set the system clock to the new time. The monotonic clock doesn't change;
    // timeouts will continue to execute after the requested time has elapsed.
    this.#systemTime = Temporal.Instant.from(newSystemTime);
  }

  timePasses(duration) {
    duration = Temporal.Duration.from(duration);

    // Advance the system clock by the requested amout
    this.#systemTime = this.#systemTime.add(duration);

    // Also advance the monotonic clock by the requested amount. Balance the
    // result up to hours
    this.#elapsedMonotonic = this.#elapsedMonotonic
      .add(duration)
      .round({ largestUnit: 'hours' });

    // Execute any pending timeouts that would have happened in the meantime
    while (this.#pendingTimeouts[0] &&
      Temporal.Duration.compare(
        this.#pendingTimeouts[0].triggerTime,
        this.#elapsedMonotonic
      ) <= 0) {
      const { callable } = this.#pendingTimeouts.shift();
      callable();
    }
  }
}
```

You can run code, such as a test suite, inside this realm, and advance the time as needed. This is a feature in some test harnesses such as [Jasmine](https://jasmine.github.io/tutorials/async#using-the-mock-clock-to-avoid-writing-asynchronous-tests). It would otherwise need to be implemented by overwriting the main realm's clock.

## Security implications

Any code evaluation mechanism in this API is subject to the existing [content security policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) (CSP). If the CSP directive from a page disallows `unsafe-eval`, it prevents synchronous evaluation in the ShadowRealm. That is, `evaluate()` won't work.

On the other hand, `importValue()` doesn't require `unsafe-eval` — it's equivalent to a dynamic `import()` call. However, the CSP of a page can set directives like `default-src` to prevent a ShadowRealm from loading resources using `importValue()`.

Without ShadowRealm, if you wanted to provide the same functionality as ShadowRealm in a cross-platform way, it'd require using `eval` and therefore you'd have to allow `unsafe-eval` in your website's CSP. With ShadowRealm, the main use case of isolating dependencies from each other becomes possible using only `importValue()` and therefore does not require a CSP with `unsafe-eval`.

## Decision Record

- ShadowRealm (then called [Realm](https://gist.github.com/dherman/7568885)) was part of the original ES6 spec, but didn't make the cut.
- ShadowRealms execute code in the same thread and process, because there are already mechanisms to do this across threads and processes (e.g., Workers); and not all use cases require asynchronous communication.
- ShadowRealm environments include all the built-ins defined in the ECMAScript specification, but hosts are allowed to add additional built-ins. Browser hosts will add built-ins with [an `[Exposed=*]` annotation in WebIDL](https://www.w3.org/TR/design-principles/#expose-everywhere). This is because developers shouldn't need to be aware that common "JavaScript-y" APIs like `TextEncoder` are technically not part of JavaScript from a standards perspective. (History: [#284](https://github.com/tc39/proposal-shadowrealm/issues/284), [#288](https://github.com/tc39/proposal-shadowrealm/issues/288), [#393](https://github.com/tc39/proposal-shadowrealm/issues/393).)
- More past discussions can be found in the [proposal-shadowrealm issue tracker](https://github.com/tc39/proposal-shadowrealm/issues) and in TC39 plenary meeting notes:
  - [February 2025](https://github.com/tc39/notes/blob/main/meetings/2025-02/february-18.md#shadowrealm-status-update)
  - [December 2024](https://github.com/tc39/notes/blob/main/meetings/2024-12/december-02.md#shadowrealm-for-stage-3)
  - [June 2024](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2024-06/june-12.md#shadowrealm-update)
  - [February 2024](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2024-02/feb-7.md#shadowrealms-update)
  - [November 2023](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2023-11/november-27.md#shadowrealm-stage-2-update)
  - [September 2023](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2023-09/september-27.md#shadowrealm-implementer-feedback-and-demotion-to-stage-2)
  - [November 2022](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2022-11/dec-01.md#shadowrealm)
  - [September 2022](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2022-09/sep-13.md#shadowrealm-update)
  - [June 2022](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2022-06/jun-06.md#shadowrealm-implementation-status-and-normate-updates)
  - [March 2022](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2022-03/mar-29.md#shadowrealms-updates)
  - [December 2021](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2021-12/dec-14.md#shadowrealms-updates-and-potential-normative-changes)
  - [August 2021](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2021-08/aug-31.md#realms-renaming-bikeshedding-thread)
  - [July 2021](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2021-07/july-13.md#realms-for-stage-3) ([+ continuation](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2021-07/july-15.md#realms-for-stage-3-continued))
  - [May 2021](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2021-05/may-26.md#realms)
  - [April 2021](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2021-04/apr-21.md#isolated-realms-update)
  - [January 2021](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2021-01/jan-26.md#realms-update)
  - [November 2020](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2020-11/nov-17.md#realms-for-stage-3)
  - [June 2020](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2020-06/june-4.md#realms-stage-2-update)
  - [February 2020](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2020-02/february-5.md#update-on-realms)
  - [July 2018](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2018-07/july-24.md#report-on-realms-shim-security-review)
  - [May 2018](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2018-05/may-23.md#realms)
  - [March 2018](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2018-03/mar-20.md#10ia-update-on-frozen-realms-in-light-of-meltdown-and-spectre)
  - [March 2017](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2017-03/mar-23.md#10iic-realms-update)
  - [January 2017](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2017-01/jan-26.md#13iid-seeking-stage-1-for-realms)
  - [March 2016](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2016-03/march-30.md#draft-proposed-frozen-realm-api)
  - [May 2015](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2015-05/may-29.md#fresh-realms-breakout)
  - [June 2014](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2014-06/jun-4.md#47-removal-of-realms-api-from-es6-postponement-to-es7)
  - [January 2014](https://github.com/tc39/notes/blob/21ff7b482a627bf86ea0981eac60ceb5924ed1f1/meetings/2014-01/jan-29.md#security-review-for-loadersrealms)
