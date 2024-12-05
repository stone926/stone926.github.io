---
title: Nodejs中的模块化
date: 2024-12-04 15:23:56
tags:
  - 前端
  - Node.js
  - 模块化
---

**以下内容基于v22 LTS版的Node**

`Node`支持两套模块化系统`CommonJS(CJS)`和`ECMAScript modules(ESM)`

## CommonJS

`Node`加载`CJS`是完全同步（`synchronous`）的，因为需要执行模块文件。

`Node`将如下文件视作`CJS`模块：

- 扩展名为`.cjs`
- 扩展名为`.js`且`package.json`中配置了`"type": "commonjs"`
- 当扩展名为`.js`或无扩展名时，不存在`package.json`或最近的`package.json`没有配置`type`。除非按`CJS`解析有语法错误但按`ESM`解析没有语法错误
- 扩展名不为` .mjs`, `.cjs`, `.json`, `.node`,` .js`。当`package.json`配置了`"type": "module"`，此类文件按`CJS`解析当且仅当它们被`require()`引用

### 导出

```js
function foo() {}

module.exports = {
    foo
}

exports.foo = foo
```

上面两种方法都可以用来导出模块。注意，`module.exports`和`exports`是两个指针，`module.exports`指向的内容是最终导出的内容，`exports`初始指向与`module.exports`相同，如果对`exports`重新赋值，如：

```js
exports = {
    foo
}
```

会导致`exports`指向另一个对象，相当于改变一个指针的指向，而不会修改指针原先所指向的内容，所以不会导出任何内容。近似类比如下`C`代码：

```C
int a = 1, b = 2, *p = &a;
p = &b; // exports = { foo }
*p = b; // exports.foo = foo
```

因此不难理解如下代码最终导出的内容只有`name: "calculator"`：

```js
function plus(a, b) {
    return a + b;
}

exports.plus = plus;

module.exports = {
    name: "calculator"
}
```

因为我们让`module.exports`指向了一个新的对象。

### 导入

```js
const m = require("./lib.js")
```

`m`就是`module.exports`对象。`import()`可以用来导入`ESM`。

### 用require导入ESM

{% notel red Node命令行提示 %}

Support for loading ES Module in require() is an experimental feature and might change at any time

{% endnotel %}

`require`导入`ESM`须满足如下条件：

- `ESM`完全同步（`synchronous`），即顶层不包含`await`（在`CJS`中异步模块要以`import()`导入）
- 以下三条满足其一
	- 目标文件扩展名为`.mjs`
	- 目标文件扩展名为`.js`且最近的`package.json`配置了`"type": "module"`
	- 目标文件扩展名为`.js`且最近的`package.json`未配置`"type": "commonjs"`且目标文件包含`ESM`语法

引用[`Node`官方文档](https://nodejs.org/docs/latest-v22.x/api/modules.html)的例子：

```js
// distance.mjs
export function distance(a, b) { return (b.x - a.x) ** 2 + (b.y - a.y) ** 2; }

// index.cjs
const distance = require('./distance.mjs');
console.log(distance);
// [Module: null prototype] {
//   distance: [Function: distance]
// }
```

```js
// point.mjs
export default class Point {
  constructor(x, y) { this.x = x; this.y = y; }
}

// index.cjs
const point = require('./point.mjs');
console.log(point);
// [Module: null prototype] {
//   default: [class Point],
//   __esModule: true,
// } 
```

`require`在导入默认导出时，会添加`__esModule: true`，以区分`CJS`的`exports.default`和`ESM`的`exprot default`。如果在导出时已经有了`__esModule`属性则不会再添加。**不要**在任何时候使用这个属性，因为该特性尚不稳定可能改变。

当默认导出和具名导出同时存在时：

```js
// lib.js
export function f(a, b) {
  return a - b;
}

export default class T {
  constructor() {
    console.log("constructed!")
  }
}

// main.js
const lib = require("./lib.js")
console.log(lib)
// [Module: null prototype] {
//   __esModule: true,
//   default: [class T],
//   f: [Function: f]
// }
```

可以通过`export as`自定义哪些内容会导出给`CJS`，此时其余内容会被忽略。

```js
// lib.js
export function f(a, b) {
  return a - b;
}

export default class T {
  constructor() {
    console.log("constructed!")
  }
}

export { f as "module.exports" }

// main.js
const lib = require("./lib.js")
console.log(lib)
// [Function: f]
```

### 源码

`Node`的`CJS`模块运行在一个包裹函数`wrapper`中，这就是为什么我们可以访问到`module`, `require`, `__dirname`等。`wrapper`如下：

```js
// node/lib/internal/modules/cjs/loader.js

let wrap = function(script) { // eslint-disable-line func-style
  return Module.wrapper[0] + script + Module.wrapper[1];
};

const wrapper = [
  '(function (exports, require, module, __filename, __dirname) { ',
  '\n});',
];

let wrapperProxy = new Proxy(wrapper, {
  __proto__: null,

  set(target, property, value, receiver) {
    patched = true;
    return ReflectSet(target, property, value, receiver);
  },

  defineProperty(target, property, descriptor) {
    patched = true;
    return ObjectDefineProperty(target, property, descriptor);
  },
});

ObjectDefineProperty(Module, 'wrap', {
  __proto__: null,
  get() {
    return wrap;
  },

  set(value) {
    patched = true;
    wrap = value;
  },
});

ObjectDefineProperty(Module, 'wrapper', {
  __proto__: null,
  get() {
    return wrapperProxy;
  },

  set(value) {
    patched = true;
    wrapperProxy = value;
  },
});

```

源码中的`_compile`方法使模块在`wrapper`中运行并给出其运行结果，即导出的内容：

```js
// node/lib/internal/modules/cjs/loader.js

Module.prototype._compile = function (content, filename, format) {
  // 如果是typescript，擦除ts类型标注并切换模块类型
  if (format === 'commonjs-typescript' || format === 'module-typescript' || format === 'typescript') {
    content = stripTypeScriptModuleTypes(content, filename);
    switch (format) {
      case 'commonjs-typescript': {
        format = 'commonjs';
        break;
      }
      case 'module-typescript': {
        format = 'module';
        break;
      }
      // If the format is still unknown i.e. 'typescript', detect it in
      // wrapSafe using the type-stripped source.
      default:
        format = undefined;
        break;
    }
  }

  let redirects;

  let compiledWrapper;
  if (format !== 'module') { // 如果是CJS，将其包裹并运行
    const result = wrapSafe(filename, content, this, format);
    compiledWrapper = result.function;
    if (result.canParseAsESM) {
      format = 'module';
    }
  }

  if (format === 'module') { // 如果是ESM
    loadESMFromCJS(this, filename, format, content);
    return;
  }

  // 截至此，模块是CJS且无法转为ESM
  const dirname = path.dirname(filename);
  const require = makeRequireFunction(this, redirects);
  let result;
  const exports = this.exports;
  const thisValue = exports;
  const module = this;
  if (requireDepth === 0) { statCache = new SafeMap(); }
  setHasStartedUserCJSExecution();
  this[kIsExecuting] = true;
  if (this[kIsMainSymbol] && getOptionValue('--inspect-brk')) {
    const { callAndPauseOnStart } = internalBinding('inspector');
    result = callAndPauseOnStart(compiledWrapper, thisValue, exports,
      require, module, filename, dirname); // 运行包裹后的模块
  } else {
    result = ReflectApply(compiledWrapper, thisValue,
      [exports, require, module, filename, dirname]); // 运行包裹后的模块
  }
  this[kIsExecuting] = false;
  if (requireDepth === 0) { statCache = null; }
  return result;
};
```

```js
// node/lib/internal/modules/cjs/loader.js

function wrapSafe(filename, content, cjsModuleInstance, format) {
  assert(format !== 'module', 'ESM should be handled in loadESMFromCJS()');
  const hostDefinedOptionId = vm_dynamic_import_default_internal;
  const importModuleDynamically = vm_dynamic_import_default_internal;
  if (patched) {
    const wrapped = Module.wrap(content); // 这个方法在上面已经用Object.definerPorperty定义过了
    const script = makeContextifyScript(
      wrapped,                 // code
      filename,                // filename
      0,                       // lineOffset
      0,                       // columnOffset
      undefined,               // cachedData
      false,                   // produceCachedData
      undefined,               // parsingContext
      hostDefinedOptionId,     // hostDefinedOptionId
      importModuleDynamically, // importModuleDynamically
    );

    // Cache the source map for the module if present.
    const { sourceMapURL } = script;
    if (sourceMapURL) {
      maybeCacheSourceMap(filename, content, cjsModuleInstance, false, undefined, sourceMapURL);
    }

    return {
      __proto__: null,
      function: runScriptInThisContext(script, true, false), // 运行包裹后的模块
      sourceMapURL,
    };
  }

  let shouldDetectModule = false;
  if (format !== 'commonjs') {
    if (cjsModuleInstance?.[kIsMainSymbol]) {
      // For entry points, format detection is used unless explicitly disabled.
      shouldDetectModule = getOptionValue('--experimental-detect-module');
    } else {
      // For modules being loaded by `require()`, if require(esm) is disabled,
      // don't try to reparse to detect format and just throw for ESM syntax.
      shouldDetectModule = getOptionValue('--experimental-require-module');
    }
  }
  const result = compileFunctionForCJSLoader(content, filename, false /* is_sea_main */, shouldDetectModule); // 运行包裹后的模块

  // Cache the source map for the module if present.
  if (result.sourceMapURL) {
    maybeCacheSourceMap(filename, content, cjsModuleInstance, false, undefined, result.sourceMapURL);
  }

  return result;
}
```

通过调用`primordials.ReflectApply`执行包裹后的代码。`primordials`是一个内部（`internal`）模块，`ReflectApply`用`C++`实现，我没有看懂相关代码。

## ECMAScript Module

`Node`将如下代码视作`ESM`模块：

- 扩展名为`.mjs`
- 扩展名为`.js`且`package.json`中配置了`"type": "module"`
- 含有`ESM`语法，并且和显式模块类型声明（文件扩展名和`package.json`配置）不冲突。`ESM`语法不包括`import()`，因为它在两种模块系统中都能用；`ESM`语法包括：
  - `import`、`export`、`import.meta`
  - 顶层`await`
  - 重新声明包裹函数参数`require`、`module`、`exports`、`__dirname`、`__filename`

### 导出

```js
export default { foo: 1 }      // 默认导出 
export const obj = { bar: 2 }  // 具名导出
```

### 导入

```js
import Obj from "module1"              // 默认导入
import { Obj as Obj2 } from "module2"  // 具名导入
import * as Obj3 from "module3"        // 全部导入
import("module4").then(Obj4 => {})     // 异步导入
```

### 用import导入CJS

我们测试如下的代码：

```js
// index.js
import lib1a from "./lib1.cjs?query=1"
import * as lib1b from "./lib1.cjs?query=2"
console.log("import lib1a: ", lib1a, "\n")
console.log("import * as lib1b: ", lib1b)
```

```js
// lib1.cjs
module.exports.f = function f(a, b) {
  return a - b;
}

module.exports.U = class U {
  constructor() {
    console.log("constructed!")
  }
}
```


在配置了`"type": "module"`的语境下执行`node index.js`，得到输出：

```plainText
import lib1a:  { f: [Function: f], U: [class U] }

import * as lib1b:  [Module: null prototype] {
  U: [class U],
  default: { f: [Function: f], U: [class U] },
  f: [Function: f]
}
```

`module.exports`是默认导入会导入的内容。

```js
import { default as lib } from "./lib.cjs"
// equivalent to
import lib from "./lib.cjs" // syntax sugar
```

### CJS包裹内容的平替

`CJS`因为外层包裹可以使用`__dirname`、`__filename`、`require.resolve()`等，在`ESM`中，其平替存在于`import.meta`中，其包含如下内容：

- `import.meta.filename`：同`__filename`
- `import.meta.dirname`：同`__dirname`
- `import.meta.resolve`：同`require.resolve`，参数为想导入之模块的路径，返回值为其`file:///`开头的`URL`
- `import.meta.url`：以`file:///`开头的当前文件`URL`

但`require.cache`、`require.extensions`、`NODE_PATH`没有平替，无法访问到。如果想使用`require`函数，使用`node:module`中的`createRequire`函数创建`require`函数。
