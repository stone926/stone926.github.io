---
title: Babel插件开发小事：支持模块化、代码复用化
date: 2024-11-17 18:38:15
tags:
 - 前端
 - Babel
---

## 一、解决上期问题

上期提到，如果用如下方式导出插件会报错：

```js
export default function ({ types: t }) {
    // ...
}
```

解决方式是在`package.json`根对象下加入

```json
"type": "module"
```

这样，所有`.js`文件都会当作`ES module`处理。如果想按`CommonJS`处理，将文件扩展名改为`.cjs`。同理，如果`type`是`commonjs`，所有`ES module`文件的扩展名应为`.mjs`。

注意，用`ESM`引入`CJS`模块时：

```js 
// lib.js
exports.default = function traverse() { 
	console.log("successfully imported")
}
traverse.foo = foo
traverse.bar = bar

// index.js
import traverse from "./lib.js"
traverse()
```

如果这样引用模块就会报错`traverse`不是函数。因为这里引入的是整个`exports`对象，所以需要如下写：

```js
import traverse from "./lib.js"
traverse.default(); // output on console: successfully imported
```

这点我们后面会用到。

## 二、支持导入重载

```js
import { $operator as $ } from "./operator.js"

console.log(1 + "1")
```

现在会被编译为如下代码（如果`$operator`重载了加法）：

```js
import { $operator as $ } from "./operator.js"

console.log($.plus(1, "1"))
```

编译后重载对象的名称都是`as`后面的名称，`import`必须为具名导入，`as`不是必需的，不能为：

```js
import $operator from "./operator.js"
import * as $operator from "./operator.js"
```

这与插件的工作原理有关。

## 三、导入的工作原理

```js
Program(path, state) {
	path.traverse({
		ImportDeclaration(path) {
			for (let i = 0; i < (path.node.specifiers.length ?? 0); i++) {
				let specifier = path.node.specifiers[i];
				if (specifier.imported.name === outer.operatorObjectName) {
                	let x = path.node.source.value;
                	if (!x.endsWith(".js")) x += ".js"
                	operatorFileName = nodePath.join(state.filename, "../", x);
                	operatorObjName = specifier.local.name;
                	return;
				}
			}
		}
	});       
}
```

插件会先遍历`import`语句，并判断`import`进来的名字是不是`babel`配置文件中配置的重载对象名`operatorObjectName`。然后将`$operator`所在的文件解析为绝对路径，用于后续读取。`specifier.local.name`在有`as`的情况下为`as`后的值，没有则是原来的值，即`specifier.imported.name`。

```js
import parser from "@babel/parser";
import traverse from "@babel/traverse";

const VariableDeclaration = (path) => { /* ... */ };

if (operatorFileName) {
	let operatorFile = fs.readFileSync(operatorFileName, { encoding: outer.encoding });
	const ast = parser.parse(operatorFile, { sourceType: "module" });
	traverse.default(ast, { VariableDeclaration });
} else {
	path.traverse({ VariableDeclaration });
}
```

接下来读取`$operator`所在的文件为字符串，并讲其解析为抽象语法树，然后遍历其抽象语法树，找到重载对象的声明并注册重载。

我们需要在`Babel`配置文件中配置被读取的文件的编码，默认值为`utf8`。

```json
"plugins": [["./plugin-operator/main.js", { "encoding": "utf8" }]]
```

此处我们用到了两个`Babel API`。`parser.parse`读取字符串并将其解析为`AST`，只有在第二个参数中传入`sourceType: "module"`才能解析`ESM`，否则会报错。

## 四、复用代码

注意到，本插件中所有的`visitor`都哦有类似的如下结构：

```js
const operatorObjectParent = path.findParent((parentPath) => t.isVariableDeclaration(parentPath) && 	operatorObjName == parentPath.node.declarations?.[0].id.name);
if (operatorObjectParent) return;
const operator = outer.registeredOperators.get(path.node.operator /* + 特殊处理自增减 */);
if (operator) { path.replaceWithMultiple(/* replacement */); }
```

我们可以将这些代码提取出来以优化代码结构。每个`visitor`都是一个函数，因此我们可以创建一个高阶函数，他接受一个函数`replacement`，这是所有`visitor`唯一不同的地方，并返回一个拥有相应`replacement`的函数。

```js
const visitorFactory = (replacement) => (path) => {
	const operatorObjectParent = path.findParent((parentPath) => t.isVariableDeclaration(parentPath) && operatorObjName == parentPath.node.declarations?.[0].id.name);
	if (operatorObjectParent) return;
	const operator = outer.registeredOperators.get(path.node.operator + path.node.prefix ?? "");
	if (operator) {
		path.replaceWithMultiple(replacement);
	}
}
```

但只有这样是不够的，因为`replacement`需要访问`path`和`operator`，所以我们需要将`replacement`改成一个函数，将`path`和`operator`传递给他。

```js
const visitorFactory = (replacement) => (path) => {
	const operatorObjectParent = path.findParent((parentPath) => t.isVariableDeclaration(parentPath) && operatorObjName == parentPath.node.declarations?.[0].id.name);
	if (operatorObjectParent) return;
	const operator = outer.registeredOperators.get(path.node.operator + path.node.prefix ?? "");
	if (operator) {
		path.replaceWithMultiple(replacement(operator, path));
	}
}
```

于是`visitor`可以改造成这样：

```js
AssignmentExpression: visitorFactory((operator, path) => t.parenthesizedExpression(
	t.assignmentExpression(
		"=", path.node.left, t.callExpression(
			t.memberExpression(t.identifier(operatorObjName), t.identifier(operator)),
			[path.node.left, path.node.right]
		)
	), path.node.left
)),
```

对于自增减，我们进行如下的特殊处理：

```js
if (path.node.prefix) {
	// ...
} else {
	path.replaceWith(path.node.argument);
	path.insertAfter(
		t.expressionStatement(
			t.assignmentExpression(
				"=", path.node, t.callExpression(
					t.memberExpression(t.identifier(operatorObjName), t.identifier(operator)),
					[path.node]
				)
			)
		)
	);
	return path.node;
}
```

我们直接在这里操作抽象语法树节点，并返回节点自身，这样在`visitorFacotry`中就不会再修改了。

