---
title: Babel插件开发小试：基础Typescript支持
date: 2024-12-10 20:34:27
tags:
 - 前端
 - Babel
---

github仓库地址：[stone926/babel-plugin-operator: A babel plugin to enable operator overloading in javascript](https://github.com/stone926/babel-plugin-operator)

在最新版中，无论`TS`还是`JS`，重载函数有三种写法：`object method`、箭头函数、普通函数，即：

```
plus(l, r) {}
plus: (l, r) => {}
plus: function(l, r) {}
```

声明带某种运算对特定类型的重载，也支持自定义类型：

```typescript
const $operator: OperatorObject = {
  plus: [
    (l: number, r: number): number => l + r,
    (l: Matrix, r: Matrix): Matrix => l,
    function (l: string, r: string): number {
      return l.length + r.length;
    },
  ],
  minus: (l: string, r: number) => l.slice(r),
};
```

但是声明对联合类型（如：`string|number|boolean`）的重载会抛出`TypeError`，因为会导致混乱。

对插件来说只有重载函数参数的类型才有意义，返回值类型是可有可无的。如果重载函数没有注明类型，则视为`any`。只有当类型严格相同时才会重载，也就是说参数类型为`(any, any)`的函数不会重载`(number, number)`。重载当且仅当类型严格相等。

`Babel`编译`Typescript`的方式是移除一切类型检查，`Babel`不做任何类型分析。所以在插件中只能判断两类节点的类型：`Literal`和`Identifier`。`Identifier`有两种：`undefined`和变量名。`undefined`的类型是`undefined`；对于变量名，可以在`scope`中的`binding`中找到其声明时的`identifier`，其中可以找到`typeAnnotation`节点；对于`Literal`，只需要判断是哪种`Literal`就可以确定类型。因此我们可以写出如下函数：

```js
export const getType = (node, scope) => {
  if (t.isIdentifier(node)) {
    if (node.name === "undefined") return t.tsUndefinedKeyword();
    const binding = scope.getBinding(node.name);
    return binding?.identifier.typeAnnotation?.typeAnnotation;
  } else if (t.isStringLiteral(node) || t.isTemplateLiteral(node)) {
    return t.tsStringKeyword();
  } else if (t.isNumericLiteral(node)) {
    return t.tsNumberKeyword();
  } else if (t.isNullLiteral(node)) {
    return t.tsNullKeyword();
  } else if (t.isBooleanLiteral(node)) {
    return t.tsBooleanKeyword();
  } else if (t.isRegExpLiteral(node)) {
    return t.tsTypeReference(t.identifier("RegExp"));
  } else if (t.isBigIntLiteral(node)) {
    return t.tsBigIntKeyword();
  } else if (t.isDecimalLiteral(node)) { // !! what's this?
    return t.tsNumberKeyword();
  } else if (t.isUnaryExpression(node)) {
    return node.operator === '-' && t.isNumericLiteral(node.argument) ? t.tsNumberKeyword() : undefined;
  }
}
```

其中`decimalLiteral`是什么我并不清楚也没有查到，暂时当作`number`类型处理。需要注意，`-1`其实是一个`UnaryExpression`，其运算符`operator`为`-`，`argument`为`NumericLiteral`的`1`。需要特殊判断负号表达式将其视为`number`类型。

得到类型后，我们判断运算符操作对象的类型和重载函数参数的类型就可以判定是否要重载此处的运算。我们将`visitorFactory`改造成这样：

```js
const visitorFactory = (replacement, typeKeys, tail = () => "") => (path) => {
  const operatorObjectParent = path.findParent((parentPath) =>
    t.isVariableDeclaration(parentPath) && operatorObjName == parentPath.node.declarations?.[0].id.name
  );
  if (operatorObjectParent) return;
  let key = path.node.operator;
  key += tail(path);
  const operator = outer.registeredOperators.get(key);
  if (operator) {
    let replacer;
      if (outer.isTs) {
        const types = operator.types;
        types.forEach((type, index) => {
          let allSameType = true;
          typeKeys.forEach((typeKey) => {
            allSameType = allSameType && isSameType(getType(path.node[typeKey], path.scope), 
                                                            type[typeKey]);
          })
          if (allSameType) {
            if (type.index == -1) {
              replacer = replacement(build(operatorObjName)[operator], path);
            } else {
              replacer = replacement(build(operatorObjName)[operator][index], path);
            }
          }
        })
      } else {
        replacer = replacement(build(operatorObjName)[operator], path);
       }
    if (replacer) path.replaceWith(replacer[build.raw] ?? replacer);
  }
};
```

其中，`operator.types`是我们在`VariableDeclaration`这个`visitor`中添加的：

```js
export const tsVariableDeclarationVisitor = (outer) => (path) => {
  const name = path.node.declarations?.[0].id.name;
  if (name === outer.operatorObjectName) {
    path.node.declarations?.[0].init.properties.forEach(item => {
      const name = new String(item.key.name);
      name.types = [];
      if (isFunctionOverloader(item)) {
        const functionNode = t.isObjectMethod(item) ? item : item.value;
        name.types.push(buildType(functionNode, path.buildCodeFrameError));
      } else if (isArrayOverloader(item)) {
        item.value.elements.forEach((functionNode, index) => {
          t.assertFunction(functionNode);
          name.types.push(buildType(functionNode, path.buildCodeFrameError, index));
        });
      }
      registerOperator(outer.registeredOperators, name);
    });
  }
};

const buildType = (functionNode, err, index = -1) => {
  const typeAnnotated = {}, anyTypeAnnotation = t.tsAnyKeyword();
  const paramLength = functionNode.params.length;
  if (paramLength == 2) {
    typeAnnotated.left = functionNode.params[0].typeAnnotation?.typeAnnotation ?? anyTypeAnnotation;
    typeAnnotated.right = functionNode.params[1].typeAnnotation?.typeAnnotation ?? anyTypeAnnotation;
  } else if (paramLength == 1) {
    typeAnnotated.argument = functionNode.params[0].typeAnnotation.typeAnnotation;
  } else {
    throw err(`Invalid Params Count. Expected 1 or 2, but got ${paramLength}`);
  }
  typeAnnotated.return = functionNode.returnType?.typeAnnotation ?? anyTypeAnnotation;
  typeAnnotated.index = index;
  return typeAnnotated;
}
```

新的`visitor`如下，以`UpdateExpression`为例，原先其写法过于冗长：

```js
{
  UpdateExpression: visitorFactory((builded, path) =>
   path.node.prefix ?
     build(path.node.argument)['='](builded(path.node.argument))[build.raw] :
     void (
       path.replaceWith(path.node.argument),
       path.insertAfter(build(path.node)['='](builded(path.node))[build.raw])
     ), ["argument"], (path) => path.node.prefix
  ),
}
```

这样，我们在访问`Expression`的节点时不需要关心是不是`TypeScript`、类型匹配与否等问题，只关心创建的新节点。
