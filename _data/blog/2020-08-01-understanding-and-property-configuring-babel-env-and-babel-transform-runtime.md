---
template: BlogPost
path: /understanding-babel-preset-env-and-transform-runtime
date: 2020-08-01T18:48:20.116Z
title: Understanding and property configuring @babel/env and @babel/transform-runtime
metaDescription: Understanding and properly configuring @babel/env and @babel/transform-runtime
thumbnail: /assets/1200px-Babel_Logo.svg.png
---
The field of frontend development has changed a lot in the recent years. Although it's a very chaotic field, with new frameworks emerging on a daily basis, I would say that today we have a more mature ecosystem, with established practices, frameworks and tooling that this field lacked, which wasn't the case with the backend development field.

Having this in mind, if we are serious about our frontend dev work, we should have a solid understanding of the tools we use on a daily basis in order to utilize the power they offer. In this post, we will focus on properly configuring `babel`, a transpilation tool that is used by nearly every project today.\
\
To be more specific, we will look how we can use `@babel/env` and `@babel/transform-runtime` together properly. There is a lot of misunderstanding on how to use these two together that I've come across, especially when answering issues on the babel repo, which is what made me to write this post in the first place. So let's start.

We will first take a look at `core-js` which is a polyfilling library. It exposes two packages that are relevant in this context and they are: `core-js` and `core-js-pure`. Now, `core-js` defines global polyfills, and `core-js-pure` provides polyfills that don't pollute the global environment. What this means in simpler words is that if you write this `import "core-js/stable/set";`, you import from the package that defines polluting polyfills, and as a result you have `global.Set`. On the other hand, if you write this `import Set from "core-js-pure/stable/set";`, you avoid polluting the global environment, you import `Set` and **bundle it** with your app.

Now, what does `useBuiltIns: 'usage'` together with the `corejs` option set on `@babel/preset-env` do? If you have `var p = new Promise()` in your app, `babel` will transform it to something like this:

```javascript
"use strict";

require("core-js/modules/es.object.to-string");

require("core-js/modules/es.promise");

var p = new Promise();
```

The `usage` word suggests what `babel` does: If you use code that browsers you targeted with your browser configuration do not support, `babel` will inject polyfills for that code. But the main question is: **from where**? If we inspect the transpiled code, we see that `babel` injects the `Promise` polyfill from `core-js`. Note that you should have `core-js` installed as a dependency in your app. `Babel` injects the polyfills that pollute the global environment with this setup. 

Ok, now let's see what `@babel/transform-runtime` does with the `corejs` option set. If you have the same code above, that is `var p = new Promise()`, `babel` transpiles that to somethings like this:

```javascript
"use strict";

var _interopRequireDefault = require("@babel/runtime-corejs3/helpers/interopRequireDefault");

var _promise = _interopRequireDefault(require("@babel/runtime-corejs3/core-js-stable/promise"));

var p = new _promise["default"]();
```

As we can see, the imports come from `@babel/runtime-corejs3`. This `babel` package doesn't contain any source code. Instead it only lists `core-js` and `regenerator-runtime` as dependencies. The `@babel/transform-runtime` plugin does some magic and inserts several folders with files in this package in `node_modules`. This is why `@babel/runtime-corejs3/helpers` or `@babel/runtime-corejs3/core-js-stable` folders exist in the first place. Now, the important part is that the files in these folders import things from **`core-js-pure`**. So when you use `@babel/transform-runtime` plugin, it includes polyfills from the `core-js-pure` folder, and doesn't pollute the global environment.

Answer to the question:

>  **Why when using `@babel/transform-runtime` my app gets large?**

As explained above, your app code is **bundled** together with the polyfills for features like `Promise`. Important to note here is that `@babel/transform-runtime` **doesn't care** about your `targets`. It doesn't even have an option, to specify the targets, it just includes the polyfill even if you want to target new environments.

Answer to the question: 

> **Why with using `useBuiltIns: "usage"` on `@babel/preset-env` the code is smaller than when using `@babel/transform-runtime`?**

As explained above, with this config, polyfills are included from the `core-js` package that pollutes global environment, and in the end you get something like `global.Promise`. Important to note here is that `@babel/preset-env` **respects your targets**, and doesn't include unnecessary pollyfills, where `@babel/transform-runtime` will include every polyfillable feature, which leads to many unnecessary polyfills.

Answer to the question: 

> **Should I use `useBuiltIns: 'usage'` and `corejs` option on `@babel/preset-env` together with `@babel/transform-runtime` with `core-js` option set to `false`?**

The answer is **NO**. This is not obvious at first. The only case where you will get away with this usage is when you include `@babel/runtime-corejs3` or `@babel/runtime-corejs2` in your app, and that is when `corejs: *`! And not when you include `@babel/runtime`, which happens when `corejs: false`. Why? To explore this, we need a different example. Let's say you have this code in your app: 

```javascript
async function f() {}
```

The transpiled code is this:

```javascript
"use strict";

var _interopRequireDefault = require("@babel/runtime/helpers/interopRequireDefault");

var _regenerator = _interopRequireDefault(require("@babel/runtime/regenerator"));

require("regenerator-runtime/runtime");

var _asyncToGenerator2 = _interopRequireDefault(require("@babel/runtime/helpers/asyncToGenerator"));

function f() {
  return _f.apply(this, arguments);
}

function _f() {
  _f = (0, _asyncToGenerator2["default"])( /*#__PURE__*/_regenerator["default"].mark(function _callee() {
    return _regenerator["default"].wrap(function _callee$(_context) {
      while (1) {
        switch (_context.prev = _context.next) {
          case 0:
          case "end":
            return _context.stop();
        }
      }
    }, _callee);
  }));
  return _f.apply(this, arguments);
}
```

Look at this line here:

```javascript
var _asyncToGenerator2 = _interopRequireDefault(require("@babel/runtime/helpers/asyncToGenerator"));
```

`Babel` includes helpers from `@babel/runtime`! These helpers can depend on some global features to be available. In this case, that feature is `Promise`. The code in `@babel/runtime/helpers/asyncToGenerator` uses `Promise`s!. Now you may think: **But with `useBuiltIns: 'usage'` I included polyfills for my targeted browsers?**. Yes, that's true, but with that config `babel` includes polyfills when you use that feature in your code! And you haven't used `Promise` anywhere! Now you have a problem. You need a way to transpile the babel helpers, and that's not good. This is the case that @zloirock refers to, when he says that you need to transpile the helpers. You will get away here if you use `Promise` in your app, because then globally pollutable polyfill for `Promise` will be injected like this: `require("core-js/modules/es6.promise");` As you can see, this is not predictable, and very difficult to configure. Note about the cases where I said that you will get away if you have `@babel/runtime-corejs3` or `@babel/runtime-corejs2` as dependencies instead of `@babel/runtime`. In this case polyfills from `core-js-pure` will be injected, like this: `var _asyncToGenerator2 = _interopRequireDefault(require("@babel/runtime-corejs3/helpers/asyncToGenerator"));` (`corejs` option should be set on `@babel/transform-runtime` as well)

Answer to the question: 

> **What is a good config then?**

I'll go like this: 

**App**: If you are authoring an app, use `import 'core-js` at the top of your app with `useBuiltIns` set to `entry` and `@babel/transform-runtime` only for helpers (`@babel/runtime` as dependency). This way you pollute the global environment but you don't care, its your app. You will have the benefit of helpers aliased to `@babel/runtime` and polyfills included at the top of your app. This way you also don't need to process `node_modules` (except when a dependency uses a syntax that has to be transpiled) because if some dependency used a feature that needs a polyfill, you already included that polyfill at the top of your app.

**Library**: If you are authoring a library, use only `@babel/transform-runtime` with `corejs` option plus `@babel/runtime-corejs3` as dependency, and `@babel/preset-env` for syntax transpilation with `useBuiltIns: false`. Also I would transpile packages I would use from `node_modules`. For this you will need to set the `absoluteRuntime` option (<https://babeljs.io/docs/en/babel-plugin-transform-runtime#absoluteruntime>) to resolve the runtime dependency from a single place, because `@babel/transform-runtime` imports from `@babel/runtime-corejs3` directly, but that only works if `@babel/runtime-corejs3` is in the `node_modules` of the file that is being compiled.

Thanks for reading!
