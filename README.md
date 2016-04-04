rewire
======
**Easy monkey-patching for node.js unit tests**

[![](https://img.shields.io/npm/v/rewire.svg)](https://www.npmjs.com/package/rewire)
[![](https://img.shields.io/npm/dm/rewire.svg)](https://www.npmjs.com/package/rewire)
[![Dependency Status](https://david-dm.org/jhnns/rewire.svg)](https://david-dm.org/jhnns/rewire)
[![Build Status](https://travis-ci.org/jhnns/rewire.svg?branch=master)](https://travis-ci.org/rewire/jhnns)
[![Coverage Status](https://img.shields.io/coveralls/jhnns/rewire.svg)](https://coveralls.io/r/jhnns/rewire?branch=master)

rewire adds a special setter and getter to modules so you can modify their behaviour for better unit testing. You may

- inject mocks for other modules or globals like `process`
- inspect private variables
- override variables within the module.

rewire does **not** load the file and eval the contents to emulate node's require mechanism. In fact it uses node's own require to load the module. Thus your module behaves exactly the same in your test environment as under regular circumstances (except your modifications).

**Please note:** The current version of rewire is not compatible with `const` or [babel](http://babeljs.io/). See [Limitations](https://github.com/jhnns/rewire#limitations).

<br>

Installation
------------

`npm install rewire`

<br />

Introduction
------------

Imagine you want to test this module:

```javascript
// lib/myModules.js
// With rewire you can change all these variables
var fs = require("fs"),
    path = "/somewhere/on/the/disk";

function readSomethingFromFileSystem(cb) {
    console.log("Reading from file system ...");
    fs.readFile(path, "utf8", cb);
}

exports.readSomethingFromFileSystem = readSomethingFromFileSystem;
```

Now within your test module:

```javascript
// test/myModule.test.js
var rewire = require("rewire");

var myModule = rewire("../lib/myModule.js");
```

rewire acts exactly like require. With just one difference: Your module will now export a special setter and getter for private variables.

```javascript
myModule.__set__("path", "/dev/null");
myModule.__get__("path"); // = '/dev/null'
```

This allows you to mock everything in the top-level scope of the module, like the fs module for example. Just pass the variable name as first parameter and your mock as second.

```javascript
var fsMock = {
    readFile: function (path, encoding, cb) {
        expect(path).to.equal("/somewhere/on/the/disk");
        cb(null, "Success!");
    }
};
myModule.__set__("fs", fsMock);

myModule.readSomethingFromFileSystem(function (err, data) {
    console.log(data); // = Success!
});
```

You can also set multiple variables with one call.

```javascript
myModule.__set__({
    fs: fsMock,
    path: "/dev/null"
});
```

You may also override globals. These changes are only within the module, so you don't have to be concerned that other modules are influenced by your mock.

```javascript
myModule.__set__({
    console: {
        log: function () { /* be quiet */ }
    },
    process: {
        argv: ["testArg1", "testArg2"]
    }
});
```

`__set__` returns a function which reverts the changes introduced by this particular `__set__` call

```javascript
var revert = myModule.__set__("port", 3000);

// port is now 3000
revert();
// port is now the previous value
```

For your convenience you can also use the `__with__` method which reverts the given changes after it finished.

```javascript
myModule.__with__({
    port: 3000
})(function () {
    // within this function port is 3000
});
// now port is the previous value again
```

The `__with__` method is also aware of promises. If a thenable is returned all changes stay until the promise has either been resolved or rejected.

```javascript
myModule.__with__({
    port: 3000
})(function () {
    return new Promise(...);
}).then(function () {
    // now port is the previous value again
});
// port is still 3000 here because the promise hasn't been resolved yet
```

<br />

Limitations
-----------

**Using `const`**<br>
It's not possible to rewire `const` (see [#79](https://github.com/jhnns/rewire/issues/79)). This can probably be solved with [proxies](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Proxy) someday but requires further research. 

**Transpilers**<br>
Some transpilers, like babel, rename variables in order to emulate certain language features. Rewire will not work in these cases (see [#62](https://github.com/jhnns/rewire/issues/62)). A possible solution might be switching to [babel-plugin-rewire](https://github.com/speedskater/babel-plugin-rewire).

**Variables inside functions**<br>
Variables inside functions can not be changed by rewire. This is constrained by the language.

```javascript
// myModule.js
(function () {
    // Can't be changed by rewire
    var someVariable;
})()
```

**Modules that export primitives**<br>
rewire is not able to attach the `__set__`- and `__get__`-method if your module is just exporting a primitive. Rewiring does not work in this case.

```javascript
// Will throw an error if it's loaded with rewire()
module.exports = 2;
```

**Globals with invalid variable names**<br>
rewire imports global variables into the local scope by prepending a list of `var` declarations:

```javascript
var someGlobalVar = global.someGlobalVar;
```

If `someGlobalVar` is not a valid variable name, rewire just ignores it. **In this case you're not able to override the global variable locally**.

**Special globals**<br>
Please be aware that you can't rewire `eval()` or the global object itself.


<br />

API
---

### rewire(filename: String): rewiredModule

Returns a rewired version of the module found at `filename`. Use `rewire()` exactly like `require()`.

### rewiredModule.&#95;&#95;set&#95;&#95;(name: String, value: *): Function

Sets the internal variable `name` to the given `value`. Returns a function which can be called to revert the change.

### rewiredModule.&#95;&#95;set&#95;&#95;(obj: Object): Function

Takes all enumerable keys of `obj` as variable names and sets the values respectively. Returns a function which can be called to revert the change.

### rewiredModule.&#95;&#95;get&#95;&#95;(name: String): *

Returns the private variable with the given `name`.

### rewiredModule.&#95;&#95;with&#95;&#95;(obj: Object): Function&lt;callback: Function>

Returns a function which - when being called - sets `obj`, executes the given `callback` and reverts `obj`. If `callback` returns a promise, `obj` is only reverted after the promise has been resolved or rejected. For your convenience the returned function passes the received promise through.

<br />


### Intercepting require calls
You can intercept calls to `require` by giving `rewire` a second argument.

```javascript
var rewire = require("rewire");

var myModule = rewire("../lib/myModule.js", {
    'fs' : fsMock,
    './otherModule.js' : { otherFunction: function(){} }
});
```

Calls to `require` using paths found in the given object will be intercepted, and the relevant
mock object will be returned.

You can also use a function to resolve the intercepted call.


```javascript
var rewire = require("rewire");

var myModule = rewire("../lib/myModule.js", function (path){
    if(/otherModule/.test(path)){
        return { otherFunction: function(){} };
    } else {
        return undefined;
    }
});
```


If the function's return value will be used as the mock object, if it is not `undefined`. Otherwise the original `require` function is called.


Caveats
-------

**Difference to require()**<br>
Every call of rewire() executes the module again and returns a fresh instance.

```javascript
rewire("./myModule.js") === rewire("./myModule.js"); // = false
```

This can especially be a problem if the module is not idempotent [like mongoose models](https://github.com/jhnns/rewire/issues/27).

**Globals are imported into the module's scope at the time of rewiring**<br>
Since rewire imports all gobals into the module's scope at the time of rewiring, property changes on the `global` object after that are not recognized anymore. This is a [problem when using sinon's fake timers *after* you've called `rewire()`](http://stackoverflow.com/questions/34885024/when-using-rewire-and-sinon-faketimer-order-matters/36025128).

**Dot notation**<br>
Although it is possible to use dot notation when calling `__set__`, it is strongly discouraged in most cases. For instance, writing `myModule.__set__("console.log", fn)` is effectively the same as just writing `console.log = fn`. It would be better to write:

```javascript
myModule.__set__("console", {
    log: function () {}
});
```

This replaces `console` just inside `myModule`. That is, because rewire is using `eval()` to turn the key expression into an assignment. Hence, calling `myModule.__set__("console.log", fn)` modifies the `log` function on the *global* `console` object.

<br />

webpack
-------
See [rewire-webpack](https://github.com/jhnns/rewire-webpack)

<br />

CoffeeScript
------------

Good news to all caffeine-addicts: rewire works also with [Coffee-Script](http://coffeescript.org/). Note that in this case CoffeeScript needs to be listed in your devDependencies.

<br />

## License

MIT
