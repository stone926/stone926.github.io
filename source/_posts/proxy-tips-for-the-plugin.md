---
title: Babel插件开发小试：用Proxy让代码更易读与其他优化
date: 2024-12-09 18:34:47
tags:
  - Proxy
  - Babel	
  - 前端
---

## 一、遇到的问题

最近我在给运算符重载`Babel`插件增加`Typescript`支持，我们可以以如下方式声明同一个运算对多个类型的重载：

```typescript
const $operator = {
    plus: [
        (l: number, r: number): number => l + r * 2,
        (l: string, r: string): number => l.length + r.length
    ]
}
```

左值和右值类型匹配的加法会编译，不匹配则不编译：

```js
$operator.plus[0](1, 2);
$operator.plus[1]("str1", "str2");
true + false;
```

用于构建其抽象语法树的代码如下：

```js
t.callExpression(
	t.memberExpression(
		t.memberExpression(
			t.identifier(operatorObjName), t.identifier(operator.toString())
		), t.numericLiteral(index), true
	), [path.node.left, path.node.right]
);
```

多个函数嵌套可读性很差，而且冗长，并且要从内往外读，不符合阅读习惯。于是，我们能否写一个工具，让所见即所得？如果我进行一次函数调用，就会套一层`callExpression`，访问一个元素，就会套一层`memberExpression`。

## 二、解决方案

我们不难想到用`Proxy`拦截对一个对象的访问，然后在`get`中返回生成好的抽象语法树节点`memberExpression`。

```js
export const memberExpression = (obj, prop, computed = false) => {
  return new Proxy(t.memberExpression(obj, prop, computed), {
    get(target, prop) {
      return t.memberExpression(target, t.numericLiteral(Number(prop)), true)
    }
  });
}
```

使用实例：

```js
memberExpression(t.identifier("$operator"), t.identifier("plus"))[0]
// 所得内容：
t.memberExpression(
	t.memberExpression(t.identifier("$operator"), t.identifier("plus")),
    0, true
)
```

那我们不禁要问，能不能链式调用？按现在这样如想再访一次元素，就要再套一层`memberExpression`，我们希望嵌套变成链式。此外，最好也不需要自己声明许多`identifier`，只需要传入一些字符串。

在此基础上在增加函数调用的功能即可。如果用`Proxy`代理一个函数`function`，我们对代理对象进行函数调用，相当于调用被代理的函数。我们期望这个函数的参数就是`callExpression`的参数。

```js
function f(...args) {}
let p = new Proxy(f, {/* ... */})
p(something) /* equivalent to */ f(something)
```

我们最终可以得到：

```js
/** 
 * 传入的参数应为抽象语法树节点Node或字符串string
 */
export const build = (_obj) => {
  let obj = _obj;
  // 如果参数是字符串，自动变为identifier
  if (typeof obj === "string" || obj instanceof String) {
    obj = t.identifier(String(obj));
  }
  return new Proxy(
    // args应为原始类型number、string、boolean等或表达式类型的抽象语法树节点Expression
  	(...args) => build(t.callExpression(obj, args.map(item =>
        // fromLiteral就是把原始类型转为相应literal节点的函数，在此省略
    	t.isExpression(item) ? item : fromLiteral(item) 
  	))), {
    get(target, prop) { // prop的值可取string或symbol。如果不是这两个，会自动toString()
      if (prop === "raw") { // 类似vue的unref，我们最终需要的是节点而不是Proxy
        return obj;
      } else if (isAssignmentOperator(prop)) { // 这个后面再说
        return buildAssignment(obj, prop);
      }
        // 如果传入了symbol，说明使用者可能想让Babel生成形如obj[someSymbol]的代码
        // 此时他应该传入symbol变量的identifier或创建一个对Symbol的callExpression
        else if (typeof prop === "symbol") { 
        throw new TypeError("please build Symbol by function call");
      } else {
        // 再套一层build实现链式调用，因为prop形式可能多种多样，如`a..b`、`??`等
        // 在此一概令computed参数为true
        // 数组下标[0]其实是["0"]，所有数组下标最终都是字符串
        return build(t.memberExpression(obj, t.stringLiteral(prop), true));
      }
    }
  });
}
```

我们就可以实现如下效果：

```js
import generator from "@babel/generator"

const node = build("obj")[t.identifier("prop")][0].foo.bar(t.identifier("id"), "id", 1, null).baz
console.log(generator.default(node.raw).code)
// console output:
// obj[prop]["0"]["foo"]["bar"](id, "id", 1, null).baz
```

在构建抽象语法树时，所见即所得。

但我们在编写插件时还需要生成赋值语句，如果赋值语句也能所见即所得呢？我们引入`buildAssignment`函数。

```js
export const buildAssignment = (obj, operator) => {
  return (_right) => {
    let right = _right;
    try {
      right = fromLiteral(right); // 如果right是literal，将其变为literal节点
    } catch {
      right = right.raw ?? right; // 如果right是一个被build过的节点，将其"unref"
    }
    // 返回一个可以链式调用的节点
    return build(t.assignmentExpression(
      operator, obj, right
    ))
  };
}
```

因为不明原因，如果使用`let right = t.isExpression(_right) ? _right : fromLiteral(_right)`，就会抛出我上面`throw`的`TypeError: please build Symbol by function call`，我不知道这个`symbol`是从哪来的，只能暂时使用`try-catch`，我也不知道为什么这样就能修复这个问题并且目前没有发现编译出的代码有误。

当我们调用访问`build`后之节点的属性，且该属性是赋值运算符，就会返回相应的`buildAssignment`函数，我们调用这个函数并传入赋值运算符的右值，就可以得到相应的`AssignmentExpression`。我们可以实现如下效果：

```js
import generator from "@babel/generator"

const node = build("obj").prop['='](build("p").k()).func()
console.log(generator.default(node.raw).code)
// console output:
// (obj["prop"]=p["k"]())["func"]()
```

虽然等号两边不对称，但大体还是所见即所得。

## 三、实施优化

我们提取出相同的逻辑，改造`visitorFactory`。

```js
const visitorFactory = (replacement, tail = () => "") => (path) => {
	const operatorObjectParent = path.findParent((parentPath) =>
		t.isVariableDeclaration(parentPath) && operatorObjName == parentPath.node.declarations?.[0].id.name
	);
	if (operatorObjectParent) return;
	let key = path.node.operator;
	key += tail(path); // 运算符后缀，让自增自减的特判放到外面
	const operator = outer.registeredOperators.get(key);
	if (operator) {
        // 调用build，构建$operator.plus等共有的节点
		const replacer = replacement(build(operatorObjName)[operator], path);
        // replacement的返回值可以是build过的节点也可以是纯节点，纯节点上没有raw属性，因此可以空值合并
		path.replaceWith(replacer.raw ?? replacer);
	}
}
```

`visitor`可以改造为如下样子：

```js
path.traverse({
	"BinaryExpression|LogicalExpression": visitorFactory((builded, { node: { left, right } }) =>
		builded(left, right)
	),
	AssignmentExpression: visitorFactory((builded, { node: { left, right } }) => t.parenthesizedExpression(
		build(left)['='](builded(left, right)).raw // 构造赋值语句，不要丢掉.raw
	)),
	UpdateExpression: visitorFactory((builded, path) => {
		if (path.node.prefix) {
			return t.parenthesizedExpression(
 				build(path.node.argument)['='](builded(path.node.argument)).raw
			)
		} else {
			path.replaceWith(path.node.argument);
			path.insertAfter(build(path.node)['='](builded(path.node)).raw);
            return path.node;
		}
	}, (path) => path.node.prefix), // 为自增减添加后缀
	UnaryExpression: visitorFactory(
		(builded, { node: { argument } }) => builded(argument), // 构造callExpression，传入argument做参数
		(path) => path.node.operator === '-' ? "negative" : "" // 区分减法与负号
	)
 });
```

可以看到这部分代码简洁了很多，整体逻辑也更清晰了。
