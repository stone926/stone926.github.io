---
title: Babel插件开发小试：让Javascript可以运算符重载
date: 2024-11-24 20:32:57
tags:
 - 前端
 - Babel
---
## 一、插件的使用与配置

要使用此插件，在`Babel`的配置文件中需加入如下配置

```json
"plugins": [
    ["./plugin-operator/main.js", { "operatorObjectName": "$operator" }]
]
```

其中`./plugin-operator/main.js`为我在本地开发时插件的路径，在项目中使用时不能这么配置，至于怎么配置我还没研究完。

所有重载运算符的函数都必须定义在一个对象中，这个对象的名字就是`operatorObjectName`中配置的名字，如果不配置，默认是`$operator`。

## 二、重载的声明与使用

### 原位声明原位使用

```javascript
const $operator = {
  plus(left, right) {
    return parseFloat(left) + parseFloat(right);
  },
  minus(left, right) {
    return parseFloat(left) - parseFloat(right);
  },
  multiply(left, right) {
    return parseFloat(left) * parseFloat(right);
  },
  instanceof(x) {
    return false;  
  },
  typeof(x) {
    if (x instanceof String) return "string";
    if (x === null) return "undefined";
    return typeof x;
  }
}
```

这是一个例子，其中`$operator`就是在`Babel`配置文件中配置的`operatorObjectName`。所有对运算符的重载都必须在这个对象中声明，否则不会被解析；这个对象必须是常量，即声明为`const`，否则在编译阶段会报错。`$operator`内的一切运算不会重载。

该例子中声明了对加法`+`、减法`-`、乘法`*`、`instanceof`、取类型`typeof`的重载。对于二元运算符，函数的第一个参数是表达式的左语句（`Expression`），第二个参数是表达式的右语句，返回值就算运算的结果。也就是说算式`a+b`经`Babel`编译后变为`$operator.plus(a,b)`。每种运算都对应一个唯一的函数名，只有名称相应的函数才会重载相应的运算。运算与函数名的对应关系如下：

```javascript
const map = {
    plus: "+",
    minus: "-",
    multiply: "*",
    divide: "/",
    mod: "%",
    power: "**",
    incrementPrefix: "++",
    incrementSuffix: "++",
    decrementPrefix: "--",
    decrementSuffix: "--",
    plusAssignment: "+=",
    minusAssignment: "-=",
    multiplyAssignment: "*=",
    divideAssignment: "/=",
    modAssignment: "%=",
    powerAssignment: "**=",
    leftMoveAssignment: "<<=",
    rightMoveAssignment: ">>=",
    rightMoveUnsignedAssignment: ">>>=",
    bitAndAssignment: "&=",
    bitOrAssignment: "|=",
    andAssignment: "&&=",
    orAssignment: "||=",
    nullishCoalesceAssignment: "??=",
    equal: "==",
    equalStrict: "===",
    notEqual: "!=",
    notEqualStrict: "!==",
    greaterThanOrEqual: ">=",
    lessThanOrEqual: "<=",
    greaterThan: ">",
    lessThan: "<",
    and: "&&",
    or: "||",
    not: "!",
    bitAnd: "&",
    bitOr: "|",
    bitNot: "~",
    bitXor: "^",
    leftMove: "<<",
    rightMove: ">>",
    rightMoveUnsigned: ">>>",
    nullishCoalesce: "??",
    in: "in",
    instanceof: "instanceof",
    typeof: "typeof"
  }
```

特殊说明：`incrementPrefix`重载`++a`，`incrementSuffix`重载`a++`，`decrementPrefix`和`decrementSuffix`同理。自增、自减重载后依然遵循先运算后赋值或先赋值后运算。

注意到，赋值运算符`=`、取成员运算符`[]`和`.`无法被重载。

在声明了重载后，只需正常写代码，`Babel`会完成被重载之运算符的转换。例如：

```javascript
let a = 1, b = "1", c = new String("c");
console.log(a + b);
console.log(a - b);
console.log(typeof c);
```

运行编译后的如上代码，会得到输出：

```javascript
2
0
string
```

如果未经过重载，会得到输出：

```javascript
"11"
0
object
```

### （尚未实现）模块化：导入与导出重载

可以导入一个重载对象，使当前文件也会编译。

```javascript
// operator.js
export const $operator = {
    equal(left, right) {
        return left === right;
    }
}

// index.js
import { $operator } from "./operator";
console.log(1 == "1");
```

运行编译后的如上代码，会得到输出：

```javascript
false
```

如果未经过重载，会得到输出：

```javascript
true
```

导出必须为具名导出，且名称必须为`operatorObjectName`中声明的名称。

## 三、原理

如果`$operator`中声明了某种运算的重载，那么插件就会编译当前文件下所有的此运算，将其变为函数调用。

- 二元赋值运算符（`AssignmentExpression`）的编译结果一律为：`(left = $operator.${name}(left, right))`，其中`${name}`表示该运算对应的重载函数名，`left`表示左表达式，`right`表示右表达式，下同。

- 二元非赋值运算符（`BinaryExpression`,`LogicalExpression`）的编译结果一律为：`$operator.${name}(left, right)`。

- 一元运算符（`UnaryExpression`）的编译结果一律为：`$operator.${name}(x)`。

- 前置自增减运算符（`UpdateExpression`）的编译结果为：`(x=$operator.${name}(x))`。

- 后置自增减运算符（`UpdateExpression`）的编译结果为：`x;x=$operator.${name}(x)`或`_tmp=x,x=$operator.${name}(x),_tmp`

## 四、操作抽象语法树

我们在[AST Explorer](astexplorer.net)上可以看到`Babel`解析出的抽象语法树。

想要创建一个`Babel`插件（`Plugin`），首先要在`Babel`配置文件中声明该插件（见第一部分）。该配置指向的文件需要导出一个函数，如下给出了这个函数的基本结构。

```javascript
// code 4-1
module.exports = function ({ types: t }) {
	return {
        pre(state) {
            // ...
        },
        visitors: {
            BinaryExpression(path, state) {
                // ...
            }
        },
        post(state) {
            // ...
        },
        inherits: require("...")
    }
}
```

在`Babel`官方教程中，模块化方式为`ESModule`，即`import export`，但亲测会报错，解决方法未知。在此使用`module.exports`没有任何影响。`Babel`源码中的模块化代码明显是经过`Babel`编译后的`import export`。

### pre和post

分别在插件开始运行和运行完成时执行，可以用于定义运行时拥有、运行后销毁的变量。在`pre`中定义的变量不需要在`post`中释放或重新初始化，因为编译每个文件时都是新的。

### visitors

`Babel`会递归遍历抽象语法树的每一个节点，当进入该节点和离开该节点时都会调用相应的`visitor`。上面给出的`BinaryExpression`是如下代码的简写。

```js
BinaryExpression: {
    enter(path, state) {
        // ...
    }
} 
```

离开节点时调用的函数写法如下：

```js
BinaryExpression: {
    exit(path, state) {
        // ...
    }
} 
```

`BinaryExpression`和`LogicalExpression`编译产物也相同，可以用如下方法合并两个`visitor`。

```js
"BinaryExpression|LogicalExpression"(path, state) {
	// ...
}
```

各类节点都有统称，例如`BinaryExpression`和`LogicalExpression`都是`Expression`，`Expression`也是一个`visitor`，在任何`Expression`被访问时都会调用。

```js
Expression(path, state) {
    // ...
}
```

### 操作节点

我们在`code 4-1`中导出的函数具有参数`t`，这是`babel-types`，提供了修改、删除、创建、校验抽象语法树节点的函数，在一个`visitor`中，我们使用如下方式将`BinaryExpression`替换为函数调用。

```js
const operator = this.registeredOperators.get(path.node.operator); // 找到重载函数的函数名

path.replaceWith(
	t.callExpression(
		t.memberExpression(
			t.identifier(this.operatorObjectName),
			t.identifier(operator)
		), [path.node.left, path.node.right]
	)
);
```

其中`t.callExpression`创建了一个函数调用，第一个参数为函数，第二个参数为参数列表；`t.memberExpression`创建了一个成员访问，例如：

```js
t.memberExpression(t.identifier("obj"), t.identifier("member"), false);
t.memberExpression(t.identifier("obj"), t.identifier("member"), false);
t.memberExpression(t.identifier("obj"), t.stringLiteral("member"), true);
t.memberExpression(t.identifier("arr"), t.numericalLiteral(1), true);
```

抽象语法树节点对应的代码分别是

```js
obj.member
obj[member]
obj["member"]
arr[1]
```

更多抽象语法树节点详见[Babel的github](https://github.com/babel/babel/tree/master/packages/babel-types/src/definitions)。

我们也可以将一个节点换为多个节点。代码如下：

```js
path.replaceWithMultiple([
	t.parenthesizedExpression(
		t.assignmentExpression(
			"=", path.node.left, t.callExpression(
				t.memberExpression(t.identifier(this.operatorObjectName), t.identifier(operator)),
                [path.node.left, path.node.right]
			)
		), path.node.left
	),
    t.expressionStatement(t.stringLiteral("str1")),
    t.expressionStatement(t.stringLiteral("str2")),
]);
```

这段代码插入了一个赋值语句和两个字符串，赋值语句的左侧是原来的左值，右侧是一个对重载函数的调用。

用以下代码在当前节点的前面和后面插入新的节点。

```js
path.insertBefore(t.expressionStatement(t.stringLiteral("This will be inserted BEFORE current node")));
path.insertAfter(t.expressionStatement(t.stringLiteral("This will be inserted AFTER current node")));
```

在本插件中也用到了查找父节点函数。须知`path`（路径）不等于`node`（抽象语法树节点），`path`同时存储了父节点指针、作用域等信息，最有用的是`scope`（作用域），其他的大多我也不知道是干什么的。

```js
const operatorObjectParent = path.findParent((parentPath) =>
	t.isVariableDeclaration(parentPath) &&
    that.operatorObjectName == parentPath.node.declarations?.[0].id.name
);
if(operatorObjectParent) return;
```

这段代码用于判断当前节点是否是`$operator`中的，如果是，那就退出遍历以免将自己的运算重载造成递归死循环，例如：

```js
plus(l, r) {
    return plus(l, r);
}
```

### 完整代码见github仓库

[babel-plugin-operator](https://github.com/stone926/babel-plugin-operator)

## 五、遇到的问题

### （已解决）重载范围过大

如果仅有第四部分的代码，编译后的代码是有问题的，在此给出解释。

`Babel`处理代码时，执行顺序为`presets`$\rightarrow$`plugins`，其中，根据配置顺序，`presets`逆序执行，`plugins`顺序执行。但`Babel`会将所有对抽象语法树（`AST`）的遍历（`traverse`）合并，以插件为例，也就是说顺序并不是插件一$\rightarrow$插件二，而是在同一个`visitor`内是顺序执行的。以代码为例：

```javascript
// plugin1.js
Program(path) {
    doSomething1
},
BinaryExpression(path) {
    doSomething2
}

// plugin2.js
BinaryExpression(path) {
    doSomething3
}
```

```json
"plugins": ["./plugin2.js", "./plugin1.js"]
```

执行顺序并不是：`doSomething3`$\rightarrow$`doSomething1`$\rightarrow$`doSomething2`

而是：`doSomething1`$\rightarrow$`doSomething3`$\rightarrow$`doSomething2`

因为`Babel`为了提高性能而减少遍历，将其合并，示意结果如下：

```javascript
Program(path) {
    doSomething1
},
BinaryExpression(path) {
    doSomething3
    doSomething2
}
```

显然`Program`会最先`visit`。

我们称被编译的原样的代码称作原始代码，`Babel`编译后写入的新代码或改变后的代码称作产物代码。由于上述原因，产物代码也会被重载，这是我们不希望的。而且，如果原始代码中的重载函数也被编译为产物代码，产物代码又被重载，就会导致递归死循环。

#### 解决方案一

因此，我们要给原始代码打上标记，标明它是原始的，可以修改。产物代码没有这种标记，我们就不修改。我们通过加特定的注释打标记，因为每个抽象语法树节点`Node`都拥有属性`TrailingComments`，即尾注释。我们可以通过配置修改标记的内容，引入新配置：

```json
"plugins": [
    [
     "./plugin-operator/main.js",
     { "operatorObjectName": "$operator" },
     { "originalMark": "__original__" }
    ]
]
```

如果不配置，注释内容默认为`__original__`。在`visitor`对象中加入如下代码：

```javascript
pre(state) {
	this.originalMark = state.opts.originalMark ?? "__original__";
},
Program(path, state) {
	const that = this;
	path.traverse({
		"BinaryExpression|LogicalExpression|AssignmentExpression|UpdateExpression|UnaryExpression"(path, state) {
			t.addComment(path.node, "trailing", that.originalMark);
		}
    });
},
```

在每个`visitor`内加入如下代码判断是否拥有标记：

```javascript
if (!isOriginal(path.node, this.originalMark)) return;
```

其中`isOriginal`的实现为：

```js
const isOriginal = (node, originalMark) => {
    let b = false;
    node.trailingComments?.forEach(comment => {
      b = (comment.value === originalMark);
      if (b) comment.value = ""; // 移除注释，减小编译产物体积
    });
    return b;
};
```

#### 优化：解决方案二

但上述方法添加注释会增加代码体积，注释也增加了许多无用的内容，并且会破坏编译产物可读性（虽然可能没什么人会读），例如如下编译结果十分丑陋：

```js
arr[++i /**/]; 
$operator.plus(a, b);/**/
```

为了避免`Babel`的自动合并导致bug，我们就不要让`Babel`来遍历节点，我们自己遍历节点。须知，`path`上存在方法`traverse`让我们可以自己遍历`path`，其参数与插件导出之函数的返回值差别不大，只是没有`pre`、`post`，每个方法也没有`state`形参，因为他不会读取插件配置。`traverse`方法的第二个参数会绑定到一个参数的`this`上。于是我们可以写出如下代码：

```js
Program(path, state) {
    const that = this;
    path.traverse(
    	UnaryExpression(path){
            // replace the nodes
        },
        // other visitors
    )
}
```

这样就不需要打标记，直接操作节点就可以了。

## 六、未来展望

### （一）与Typescript结合，实现精确到对某种类型的重载

`Javascript`是弱类型的，无法进行类型校验，所有在运算符重载时，你在心里必须清楚参与运算的是什么类型，并且要在重载函数中判断参数的类型。如果引入`Typescript`，可以做到在编译阶段就确定哪些运算需要重载，哪些不需要。例如：定义`Typescript`类型`Matrix`，只编译`Matrix`的矩阵乘法，不编译其他乘法。

### （二）兼容Vue Template和jsx

暂未测试改插件能否在各种框架自定义的语法下使用。希望今后实现如下效果：

```vue
<template>
{{ person.name }}完成了{{ person.count }}次跳绳
<button @click="++person">click to jump</button>
</template>
<script>
import { reactive } from "vue";
const $operator = {
    incrementPrefix(x) {
        return { ...x, count: x.count + 1 };
    }
};
const person = reactive({ name: "小明", count: 1 });
</script>
```

```jsx
import { useState } from "react";
const $operator = {
    equal(l, r) { return l === r; },
    plus(l, r) {
       	if(typeof l != "number" || typeof r != "number") return parseInt(l) + parseInt(r); 
        return l + r;
    }
}
export default function Test() {
    const [a, setA] = useState(1);
    const [b, setB] = useState("1");
    function handleClick() {
        setA(a + 1);
        setB(b + 1);
    }
    return <div onClick={handleClick}>
		{ a == b } {/* Always true */}
    </div>
}
```



